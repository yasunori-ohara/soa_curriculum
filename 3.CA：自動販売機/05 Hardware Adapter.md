# 05 Hardware Adapter

# 🤖 Hardware Adapter
### `vending_machine/interface_adapters/hardware_adapter.py`

`UseCase` は「商品を出して」「お釣り返して」と命令はしますが、
実際にどうやってモーターを回すか・どうコインを返すかは知りません。

その“現実の世界のやり方”を肩代わりするのが **Hardware Adapter** です。

このファイルでは、`usecase/boundaries.py` で宣言した
`HardwareInterface` を、実際に動くクラスとして実装します。

* UseCase視点：
  「HardwareInterfaceっていう契約どおりに動いてくれるやつがいればいい」

* Adapter視点：
  「その契約どおりに、実機・シミュレーション・テスト用など好きな形で振る舞います」

この分離によって、
**ビジネスロジック（買うという流れ）とハードウェア（どう動くか）が完全に独立**します。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## 🎯 このファイルの役割

* `HardwareInterface`（= UseCaseが期待しているインターフェース）を**実装する**
* UseCaseから受け取った「抽象的な指示」を、**具体的な動きに変換する**

今回の教材では、実機の代わりに `print()` を使って動作をわかりやすく可視化します。
実務であれば、ここに本物のモーター制御や、コインメカとの通信処理などが入ります。

---

## ✅ このクラスが“やっていい”こと／“やってはいけない”こと

⭕ やっていいこと

* モーターを回す／GPIOピンを叩く／シリアル通信するなど、物理的な制御の詳細
* 今回の教材ではその代わりに `print()` で「排出しました」「お釣りを返しました」と表示する

❌ やってはいけないこと

* 「この商品、売っていいか？」の判断

  * それは `Item` エンティティや `PaymentManager` が決めている
* 「お金足りる？お釣りは？」の計算

  * それは `PaymentManager` の仕事
* 「在庫を減らす」などビジネスルール的な更新

  * それは `Item.dispense()` の仕事
* 画面表示用の文言生成（Presenterの仕事）

> 🧠 合言葉：
> **Adapterは「手足」。判断はしない。ただ実行する。**

## 💻 フォルダ構成

```text
vending_machine/
├─ domain/
│   ├─ entities.py          # Item, PaymentManager など
│   ├─ errors.py            # ドメインルール違反例外
│
├─ usecase/
│   ├─ dto.py               # InputData / OutputData / ViewModel
│   ├─ boundaries.py        # UseCaseが外部とやり取りする契約
│   ├─ insert_coin_usecase.py
│   └─ select_item_usecase.py
│
├─ interface_adapters/
│   ├─ controller.py        # Controller（入力受付）
│   ├─ presenter.py         # Presenter（出力整形）
│   ├─ view_console.py      # CLIでの動作確認
│   ├─ data_access.py       # 在庫リポジトリのインメモリ実装
│   └─ hardware_adapter.py  # ハードウェア操作のコンソール版実装    <- いまここ
│
└─ main.py                  # Composition Root（全配線）

```

## 💻 コード（`hardware_adapter.py`）

```python
# vending_machine/interface_adapters/hardware_adapter.py

# UseCaseが定義した契約（HardwareInterface）をインポートする
from vending_machine.usecase.boundaries import HardwareInterface


# -----------------------------------------------------------------------------
# ConsoleHardwareAdapter
# - 役割:
#     UseCase からの「商品を排出して」「お釣りを返して」という
#     抽象的な指示を、具体的な処理に変換する。
#
# - この教材では、実機ハードウェアの代わりに print() でシミュレートする。
#   実務であればここに、GPIO制御やシリアル通信などの本物の処理が入る。
#
# - クリーンアーキテクチャ上の位置づけ:
#     interface_adapters 層（最も外側の円の1つ）
# -----------------------------------------------------------------------------
class ConsoleHardwareAdapter(HardwareInterface):
    """
    HardwareInterface の具体的な実装（コンソール版）

    「UseCaseの依頼を現実世界の動作に変える」のが責務。
    ここでは print() によるメッセージ出力でそれを模倣する。
    """

    def __init__(self):
        # もし将来、ハードウェア制御用のドライバやポート番号など
        # 具体的な依存があれば、ここで受け取って保持するようにする。
        # 例: self.motor_driver = motor_driver
        pass

    def dispense_item(self, slot_id: str):
        """
        [インターフェースの実装]
        指定スロットの商品を物理的に排出する処理。

        UseCaseからは「A1の商品を出して」といった抽象命令しか来ない。
        それをここで具現化する。

        slot_id:
            どのスロットの商品を落とすかを示す識別子 (例: "A1")
        """
        # ここでは実物のモーターの代わりにコンソール出力で表現している
        print(f"（ガコンッ！スロット {slot_id} の商品を排出しました）")

    def return_change(self, amount: int):
        """
        [インターフェースの実装]
        指定された金額ぶんのお釣りを返却する処理。

        UseCaseからは「お釣り 30 円返して」のような依頼が来るだけであって、
        どのコインを何枚返すか、何msソレノイドを開けるか、などは関知しない。

        amount:
            返すべきお釣りの総額（円）
        """
        # ここでは実物のコインメカの代わりにコンソール出力で表現している
        print(f"（チャリン！お釣り {amount} 円を返却しました）")
```

---

## 🔍 ここで強調したいポイント

### 1. `ConsoleHardwareAdapter(HardwareInterface)` という継承

この一行がとても重要です。

* 「私は `HardwareInterface` という契約を守る実装クラスです」と宣言している
* だから UseCase は「HardwareInterface 型」としてこのクラスを受け取れる
* UseCase は中身が本物のハードだろうとダミーだろうと気にしなくていい

→ この形が「依存性逆転」です。
→ 内側（UseCase）が決めた契約に、外側（Adapter）が合わせに来る。

---

### 2. UseCaseはハードウェアの詳細を知らない

`SelectItemUseCase` はこう呼んでいましたね：

```python
self._hardware.dispense_item(item.slot_id)
if change > 0:
    self._hardware.return_change(change)
```

ここには「print」という言葉は一切出てきません。
つまりUseCase側は「printで出すのか」「本物のモーターを回すのか」を全く知らない。

→ これが「ビジネスロジックと物理制御の分離」です。
→ この分離があるから、ハードが変わってもビジネスを作り直さなくていい。

---

### 3. テストが超ラクになる

本番では `ConsoleHardwareAdapter` の代わりに、本物の `GPIOHardwareAdapter` を渡してもいいし、
テストでは `FakeHardwareAdapter` を渡して「呼ばれた内容を記録するだけ」のスpyにしてもいい。

たとえばテスト用の偽物はこう書けます：

```python
class FakeHardwareAdapter(HardwareInterface):
    def __init__(self):
        self.dispensed_slot_id = None
        self.returned_change = None

    def dispense_item(self, slot_id: str):
        self.dispensed_slot_id = slot_id

    def return_change(self, amount: int):
        self.returned_change = amount
```

これを `SelectItemUseCase` に渡しておけば、
「正しいスロットで dispense_item が呼ばれたか？」
「正しいお釣り金額で return_change が呼ばれたか？」
をアサートできます。物理筐体いらないです。最高です。

---

## 🐍 Python vs 🔧 C言語の観点

* 🐍 Python（クリーンアーキテクチャ的世界観）

  * ハードウェアとのやりとりは「HardwareInterface」という抽象の後ろ側に隠れる
  * ビジネスロジックにはGPIOやレジスタの知識を漏らさない
  * テストではハードウェア部分だけ差し替え可能

* 🔧 Cっぽい従来の書き方

  * 例えば `purchase_item()` の中で直接 `PORTB |= (1 << PB3);` みたいなI/O操作をしてしまう
  * すると、ハード構成が変わるたびにビジネスロジックごと書き直しになる
  * 自動テストもやりにくい（実機がないと動かない）

この章は、まさにその「混ざってしまいがちな世界」を切り離すための場所です。

---

## 🛡️ このクラスの鉄則

> 指示に従い、物理的に動け。
> でも考えるのはあなたの仕事じゃない。

* Hardware Adapterは「どう動くか」の専門家
* UseCaseは「いつ何をさせるか」の司令塔
* Entityは「それはそもそも正しいのか？」の番人

この分担がはっきりするほど、ソフトウェアは安全で、交換しやすく、テストしやすくなります。

---

これで、

* `domain`（ルールの中心）
* `usecase`（流れを司令する）
* `usecase/dto.py`（やり取りするデータの形）
* `usecase/boundaries.py`（誰に何を頼めるかという契約）
* `interface_adapters/hardware_adapter.py`（その契約を満たす実装）

までそろいました。

次は Controller / Presenter / View（`interface_adapters` の他ファイル）と `main.py`（Composition Root）に進んで、全体をつないで実際に「ガコンッ」と動くところまで持っていきます。
