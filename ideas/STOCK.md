# ネタ置き場（STOCK）

新記事のネタをここに貯める。1行追加するだけ。書きたいものができたら `drafts/` で執筆する。

## ルール

- 1記事 = 1技術。作ったものの紹介は書かない。日本語記事が少ないニッチ技術だけ。
- `希少性` 欄は日本語検索で確認した根拠を書く。未確認のまま書かない。
- 状態: `idea` → `draft` → `published`（公開後はURLを記録）。

## 一覧

| # | リポジトリ | 技術（記事の核） | 候補先 | 希少性 | 状態 | 公開URL |
|---|---|---|---|---|---|---|
| 1 | SAS-Watch | ESP32-S3 + ESP-IDF 5.5 + LVGL 9.5の自作スマートウォッチ。`core/`をC++17でハード分離しPCでctest、`sim/`でLVGL画面をPNG出力、自作BLE protocol-v1 | Qiita | LVGL 9系の日本語記事はほぼ皆無（2026-11確認） | idea | - |
| 2 | space-controller | RustでSpaceMouseを仮想ゲームパッド化。hidraw/evdev二本立て（旧SpaceNavigatorはhidraw必須）、uinput出力、integralモードのスロットル | Qiita | 日本語記事ゼロ級（2026-11確認） | idea | - |
| 3 | rathole-tool | Rust製リバースプロキシratholeの単一YAML→toml自動生成＋systemd＋UFW | Qiita | 日本はfrp/ngrok一色でrathole日本語記事は希少（2026-11確認） | idea | - |
| 4 | rope-mode-lab | ブラウザでXPBDロープ物理シミュレーション。多重ループを演出でなく実測、Frequency Sweep＋粗→精2段階グリッドサーチ | Zenn | XPBD日本語記事は希少（2026-11確認） | idea | - |
| 5 | d-op | WXT+TSモノレポでChrome/Firefox両対応MV3（service_workerとscripts+gecko設定の吸収）、Astro+Workers/D1共有サイト、Zod共有スキーマ | Zenn | WXT両対応の日本語記事は希少（2026-11確認） | idea | - |
