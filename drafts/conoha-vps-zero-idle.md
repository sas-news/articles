<!--
status: draft（未公開）
title: ConoHa APIでVPSを生やしては消す話
target: Qiita / ConoHa Advent Calendar 2026（https://qiita.com/advent-calendar/2026/conoha）
tags: ConoHa, VPS, API, OpenStack, nftables
source: sas-news/conoha-port-gate（実装に基づく）。
  根拠: conoha.py（エンドポイント・リクエストボディ・削除確認）、
  cloudinit.py（nft/ufw分岐）、sessions.py（ネームタグ・掃除）。
公開時のやること:
  - 冒頭に「ConoHa Advent Calendar 2026 n日目です」を追記
  - 本文のGitHubリンクは public 前提で貼ってある。非公開のまま出すなら外す
  - 公開後、このファイルを qiita/<記事ID>.md に移動しURLを記録、STOCK.md を published に更新
-->

# ConoHa APIでVPSを生やしては消す話

# はじめに
自宅サーバーでMinecraftを動かしているんですが、24時間公開するほどでもない。かといって固定でVPSを借りると、使わない時間も課金され続けます。
ConoHa VPSは時間課金です。ただし **停止しても課金は止まらず、削除でしか止まりません**（[公式FAQ](https://support.conoha.jp/common/faq/payment-q/billing-q/)）。なので「使うときだけAPIでVPSを作って、終わったら即消す」形にすれば、待機中はずっと0円になります。
ConoHa VPS 3.0にはOpenStackベースのAPIが付いているので、作成から削除まで全部APIでやります。この記事はそのときの記録です。APIの話が主体なので、自動化ツールの話は最後にちょっとだけ出てきます。

# 準備（APIユーザー）
コントロールパネルの「VPS → API」からAPIユーザーを作ります。このメニュー、VPSを1台も契約していないと表示されません。初回だけ手動で1台契約してください（APIユーザーを作った後に消してOK）。
コントロールパネルは左下の「バージョン切替」で **Ver.3.0** にしておいてください（VPS 3.0のAPIです）。
APIユーザーを作ったら「API情報」からテナントIDとエンドポイントを控えます。エンドポイントはサービスごとに分かれていて、ホスト名にリージョンが入ります（例は`c3j1`）。

- `identity.c3j1.conoha.io` … 認証
- `compute.c3j1.conoha.io` … サーバー
- `block-storage.c3j1.conoha.io` … ボリューム
- `networking.c3j1.conoha.io` … セキュリティグループ
- `image-service.c3j1.conoha.io` … イメージ

:::note warn
APIユーザーは1アカウントに1つだけ作れて、権限は変更できません（全サービスの操作権を持つ）。パスワードはConoHaアカウントのものではなく、APIユーザーに設定した専用のものです。トークンが漏れると何でもできるので取り扱い注意です。
:::

# トークンをもらう
OpenStack流に、まずトークンを発行してもらいます。

```shell
$ curl -s -D - -o /dev/null https://identity.c3j1.conoha.io/v3/auth/tokens \
  -H 'content-type: application/json' \
  -d '{
    "auth": {
      "identity": {
        "methods": ["password"],
        "password": {
          "user": {
            "name": "APIユーザー名",
            "domain": {"name": "default"},
            "password": "APIユーザーのパスワード"
          }
        }
      },
      "scope": {"project": {"id": "テナントID"}}
    }
  }' | grep -i x-subject-token
```

レスポンスヘッダーの`X-Subject-Token`がトークンです。以降はこれを`X-Auth-Token`ヘッダーに付けて叩きます。

# VPSを作る
作成は「ボリューム → セキュリティグループ → サーバー」の順です。

## ボリューム
まずイメージからブートボリュームを作ります。512MBプランだとOSはDebian 13で、イメージ名は`debian-13.5`でした（Ubuntuは1GBからのようです）。フレーバーは`g2l-t-c1m512`。イメージとフレーバーの一覧はそれぞれ`GET /v2/images`（image-service）と`GET /v2.1/flavors/detail`（compute）で取れます。

```shell
$ curl -s https://block-storage.c3j1.conoha.io/v3/テナントID/volumes \
  -H "X-Auth-Token: $TOKEN" -H 'content-type: application/json' \
  -d '{
    "volume": {
      "size": 30,
      "name": "myserver-vol",
      "volume_type": "ボリュームタイプ（/v3/テナントID/typesで一覧）",
      "imageRef": "イメージID"
    }
  }'
```

作成直後は`creating`なので、`GET /v3/テナントID/volumes/{id}`を叩いて`available`になるまで待ちます。

## セキュリティグループ
空のSGを作って、開けたいポートのルールを足します。

```shell
$ curl -s https://networking.c3j1.conoha.io/v2.0/security-groups \
  -H "X-Auth-Token: $TOKEN" -H 'content-type: application/json' \
  -d '{
    "security_group": {
      "name": "myserver-sg",
      "description": "Minecraft用"
    }
  }'
```

```shell
$ curl -s https://networking.c3j1.conoha.io/v2.0/security-group-rules \
  -H "X-Auth-Token: $TOKEN" -H 'content-type: application/json' \
  -d '{
    "security_group_rule": {
      "security_group_id": "SGのID",
      "direction": "ingress",
      "ethertype": "IPv4",
      "protocol": "tcp",
      "port_range_min": 25565,
      "port_range_max": 25565,
      "remote_ip_prefix": "0.0.0.0/0"
    }
  }'
```

## サーバー
サーバーは`block_device_mapping_v2`でボリュームを指定して作ります。`user_data`にはcloud-init（`#cloud-config`）をbase64したものを渡せるので、ソフトのインストールやファイアウォールの設定はここに押し込みます。

```shell
$ curl -s https://compute.c3j1.conoha.io/v2.1/servers \
  -H "X-Auth-Token: $TOKEN" -H 'content-type: application/json' \
  -d '{
    "server": {
      "flavorRef": "フレーバーID",
      "block_device_mapping_v2": [{"uuid": "ボリュームID"}],
      "security_groups": [{"name": "myserver-sg"}],
      "metadata": {"instance_name_tag": "myserver"},
      "user_data": "cloud-initをbase64したもの"
    }
  }'
```

`instance_name_tag`はコントロールパネルの「ネームタグ」に反映されます。便利なので、あとで使います。

# VPSを待つ
`POST /v2.1/servers`が返ってきた時点ではまだ使えません。`GET /v2.1/servers/{id}`を叩くと`status`が見えるので、`ACTIVE`になるまでポーリングします（だいたい60〜120秒）。`addresses`にグローバルIPv4が入ってくるので、接続先はここから取れます。

# VPSを消す
課金が止まるのはここだけです（停止では止まりません）。

```shell
$ curl -X DELETE https://compute.c3j1.conoha.io/v2.1/servers/サーバーID \
  -H "X-Auth-Token: $TOKEN"
```

DELETEが返ってきた瞬間はまだ消えていないことがあるので、GETが404を返すまで確認します。ボリュームとSGも同様に消して、404になったら完了です。
私は消し忘れが怖いので、ネームタグに`pg-<セッションID>-<期限のunixtime>`のような形式で期限を刻んで、期限切れのものを定期的にAPIで掃除するようにしています。ConoHa側に「いつまでの存在か」が残っているので、スクリプトが死んでいてもコントロールパネルを見れば判断できます。
あとConoHaは時間課金でも **月額上限がある**（月額を超える請求は発生しない）ので、自動化が全部死んでしまっても損失は頭打ちです。これは安心して使い捨てる上で結構効いています。

# ハマったところ
## SGを開けても繋がらない
セキュリティグループで25565/tcpを`0.0.0.0/0`に許可したのに、外部からtimeoutしました。
VPSの中で`nft list ruleset`を見ると、ConoHaのDebian 13テンプレートは **OS内のnftablesがinputチェーン`policy drop`・22番だけ許可** で起動していました。SG（インフラ層）とOS（テンプレート層）は別レイヤーなので、SGで開けてもOSがdropしていたら届きません。
この罠の詳細は先人が書いてくれているので、こちらを見てください。
[ConoHaのVPSのポートに穴をあけて外部通信できるようにした](https://blog.lycolia.info/0745)

対処は、SGで開けたのと同じポートをcloud-initでOS側にも開けることです。nftablesがあればnft、なければufwに分岐させます。

```bash
if command -v nft >/dev/null 2>&1 && nft list table inet filter >/dev/null 2>&1; then
  nft add rule inet filter input tcp dport 25565 accept
  nft add rule inet filter input ip saddr 203.0.113.10/32 tcp dport 2333 accept
  nft list table inet filter > /etc/nftables.conf
  systemctl enable nftables 2>/dev/null || true
else
  ufw --force reset
  ufw default deny incoming
  ufw default allow outgoing
  ufw logging on
  ufw allow 25565/tcp
  ufw allow from 203.0.113.10/32 to any port 2333 proto tcp
  ufw --force enable
fi
```

`203.0.113.10`は自宅のグローバルIPです。トンネルの制御ポート（2333/tcp）を自宅IPだけに限定しています。

## その他の注意
- 同一フレーバーの契約数に上限があるようで、消し忘れたVPSが残っていると次の作成が400になりました
- 1時間未満の利用は1時間分の料金なので、5分だけのために作ると損です
- イメージのAPIホストはリージョンごとに変わるので、エンドポイント一覧をちゃんと見る

# まとめたものを置いておく
この「作って → 使って → 消す」の流れ（TTLの管理、期限切れの掃除、cloud-initでのセットアップ、トンネルの用意、DNSの切り替え）を1つにまとめたものをGitHubに置いておきます。

https://github.com/sas-news/conoha-port-gate

使うときは1コマンドです。

```console
$ cpg up mc --ttl 120 --wait
id       : 5f3c1a2b
state    : active (done 100%)
host     : mc.example.com
ip       : 203.0.113.10
cost     : 約 2.6 円

$ cpg down
closing: 5f3c1a2b (mc)
```

VPSの中ではcloud-initがratholeというトンネルを起動して、VPSの公開ポートを自宅のMinecraftサーバーへ転送しています。自宅からはoutboundだけで済むので、ポート開放は不要です。

# おわりに
時間課金って「常時動かす人向けの従量制」だと思っていましたが、APIで生やしては消すと「待機費0円の公開環境」になりました。必要なエンドポイントは数個だけなので、やってみると意外と簡単です。月額上限のおかげで消し忘れの恐怖もだいぶ薄いです。

# 参考サイト
https://doc.conoha.jp/reference/api-vps3/
https://support.conoha.jp/common/faq/payment-q/billing-q/
https://blog.lycolia.info/0745
https://github.com/rathole-org/rathole
