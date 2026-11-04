<!--
title: Pybricks何deg/s出せる？
qiita_id: e8afb3e5adb848f2b33b
url: https://qiita.com/SASNEWS/items/e8afb3e5adb848f2b33b
tags: Lego, SPIKEPrime, Pybricks
created_at: 2024-07-01T08:23:59+09:00
updated_at: 2024-07-01T08:23:59+09:00
likes: 0
stocks: 0
バックアップ: Qiita APIから取得した原文。正本はQiita側。修正したら手動でQiitaに反映すること。
-->

Pybricksでは`dc(モーターのパワー)`には100までの制限がありますが、`run(モーターのスピード)`には特に制限が設けられていません。
とりあえず、何deg/sまでスピードが出せるのか検証するだけです。

# 検証
環境はSPIKE Prime、電力供給あり(充電中)です。
## 速度を見る
```py
from pybricks.parameters import Port
from pybricks.pupdevices import Motor
from pybricks.tools import wait

motor = Motor(Port.A)
cnt = 0
total = 0
motor.dc(100)
wait(10000)

while True:
    cnt += 1
    total += motor.speed()
    print(total/cnt)
```
こちらの速度測定プログラムで`dc(100)`の場合の速度を測ります。
```
1263.028
1263.024
1263.02
1263.016
1263.011
1263.003
1262.995
~~省略~~
1240.872
1240.875
1240.878
1240.881
1240.884
1240.887
```
平均値を計算してみると、速度は一定ではなく1263deg/sから1240deg/sあたりを彷徨っている事がわかります。Lモーターに置いても同様の計測を行ったところ、1150deg/sから1170deg/sあたりになりました。

## それぞれ計測
次に最高速と最低速度を計測します。
```py
from pybricks.hubs import PrimeHub
from pybricks.parameters import Port
from pybricks.pupdevices import Motor
from pybricks.tools import wait, StopWatch

hub = PrimeHub()
motor = Motor(Port.A)
time = StopWatch()
cnt = 0
total = 0
min = 10000
max = 0
motor.dc(100)
wait(10000)
time.reset()

while True:
    cnt += 1
    total += motor.speed()
    if motor.speed() < min:
        min = motor.speed()
    if motor.speed() > max:
        max = motor.speed()
    if time.time() > 60000:
        break
print("Average: ", total/cnt)
print("Min: ", min)
print("Max: ", max)
```

### パワー100
こちらのコードで(`dc(100)'で)60秒間計測してみたところ、それぞれ以下のような結果になりました。
| `dc(100)` | Mモーター | Lモーター |
| --- | --- | --- |
| 平均 (ave) | 1240.857 | 1160.183 |
| 最低 (min) | 1124 | 1040 |
| 最高 (max) | 1354 | 1303 |

### スピード1000
ついでに`dc(100)`を`run(1000)`に置き換えて、実行してみてどのくらい正しく実行されているかも計測しました。

| `run(1000)` | Mモーター | Lモーター |
| --- | --- | --- |
| 平均 (ave) | 999.9902 | 999.9998 |
| 最低 (min) | 889 | 920 |
| 最高 (max) | 1110 | 1098 |

### スピード9999(オーバー)
`run(9999)`に設定した際に最高速度が出てくれるのかも検証します。

| `run(9999)` | Mモーター | Lモーター |
| --- | --- | --- |
| 平均 (ave) | 999.9515 | 999.9941 |
| 最低 (min) | 887 | 913 |
| 最高 (max) | 1108 | 1122 |

残念ながら1000が最大値として設定されていて、それ以上は`run()`では出せないようです。

# まとめ
速度は割と安定している印象があったので、序盤はスピード指定でも大丈夫そうですね。

Lモーターの方が安定して目標値が出るので、精度が必要な移動にはLモーターを選ぶのが正解だと思います。

制度は必要なく、速度があればいい場合はMモーターがいいですね。俊敏な動作が期待できます。

