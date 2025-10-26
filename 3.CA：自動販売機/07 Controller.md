# 07 Controller

# 🕹 Controller
### `vending_machine/interface_adapters/controller.py`

`Controller` は、ユーザーの操作（ボタン押下などの入力）を受け取り、
それを「ユースケースが理解できる正式なリクエスト」に変換して、
ユースケースを呼び出す役目です。

たとえば、誰かが「A1 の商品ください」というボタンを押したとします。

* Controller はそれを受け取って
* `SelectItemInputData(slot_id="A1")` というDTOを組み立てて
* `SelectItemUseCase.handle(...)` を呼び出します

このように、Controllerは「外の世界のイベント」を「アプリケーションの操作」に翻訳するゲートです。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## 🎯 Controller の立ち位置（ざっくりいうとこう）

* Controller は、UIや物理ボタンなど“ユーザーの入力側”と UseCase のあいだに立つ。
* Controller は、自分でビジネスルールを判断しない。
* Controller は、「正しい形式でユースケースを呼び出したら、その仕事は終わり」です。

比喩でいえば：

> Controller は受付係、UseCase は現場責任者。

---

## ✅ Controllerがやるべきこと / やってはいけないこと

⭕ Controllerがやるべきこと

* ユーザーの操作を受け取る

  * 例：「A1 ボタンが押された」「100円コインが投入された」
* その情報から、UseCase用のDTO（`SelectItemInputData`など）を組み立てる
* 適切なユースケース（`SelectItemInputBoundary`）のメソッドを呼び出す

❌ Controllerがやってはいけないこと

* 「お金が足りるか？」などの判断（それは `PaymentManager` や UseCaseに任せる）
* 「お釣り30円です」といった表示用メッセージをつくる（それは Presenter）
* モーターを回す・お釣りを出すなどハードウェア制御（それは HardwareAdapter）
* ViewModelを更新したり print() したりする（それは View / Presenter）

> Controller の合言葉は
> **「正しいユースケースを、正しく呼ぶ。それ以上のことはしない。」**

---

## 💼 フォルダ構成の位置づけ

今回扱うのは `interface_adapters/controller.py` です。
フォルダ全体ではこういう位置づけになります👇

```text
vending_machine/
├─ domain/
│   ├─ entities.py             # Item, PaymentManager など (ビジネスルールの中心)
│   └─ errors.py               # ドメインルール違反例外
│
├─ usecase/
│   ├─ dto.py                  # InputData / OutputData / ViewModel
│   ├─ boundaries.py           # UseCaseが外部に求める契約
│   ├─ select_item_usecase.py  # ← 商品購入ユースケース本体
│   └─ insert_coin_usecase.py  # ← コイン投入ユースケース（後で解説）
│
├─ interface_adapters/
│   ├─ controller.py           # ← 今ここ。入力を受けてUseCaseを呼ぶ
│   ├─ presenter.py            # UseCaseの結果→ViewModel
│   ├─ view_console.py         # 画面表示（CLI）
│   ├─ data_access.py          # 在庫のリポジトリ実装（インメモリ）
│   └─ hardware_adapter.py     # ハードの実装（printでシミュレーション）
│
└─ main.py                     # ぜんぶ配線してアプリを起動する場所
```

---

## 💻 コード（`controller.py`）

このControllerは、コンソールUIを想定した「とても素直な」バージョンです。

* ユーザーが購入したいスロットIDを渡すと
* UseCaseに処理を依頼する
* 「投入金額の状態（PaymentManager）」も、UseCaseに引き渡す

```python
# vending_machine/interface_adapters/controller.py

# UseCaseを呼び出すための契約（InputBoundary）と、
# UseCaseに渡すためのデータ型（DTO）をインポート
from vending_machine.usecase.boundaries import SelectItemInputBoundary
from vending_machine.usecase.dto import SelectItemInputData
from vending_machine.domain.entities import PaymentManager


# -----------------------------------------------------------------------------
# VendingMachineController
# - クラス図の位置: Controller
# - 同心円図の位置: interface_adapters（外側の円）
#
# 役割:
#   ユーザーの操作(=どのスロットを買いたいか 等)を受け取り、
#   ユースケースを正しい形で呼び出す。
# -----------------------------------------------------------------------------
class VendingMachineController:
    """
    自動販売機の操作をまとめて受け付ける「受付係」。

    - 「このスロットの商品が欲しい」という要求を受け取り、
      SelectItemUseCase に処理を依頼する。

    - 「100円硬貨が投入された」といったイベントを受け取り、
      InsertCoinUseCase に処理を依頼する（後で追加予定）。

    Controller 自身はビジネスルールを判断しないのがポイント。
    """

    def __init__(
        self,
        select_item_usecase: SelectItemInputBoundary,
        payment_manager: PaymentManager,
    ):
        """
        [依存性の注入]

        select_item_usecase:
            商品購入のユースケース本体。
            これは SelectItemUseCase のインスタンスが来る想定だが、
            コード上はインターフェース(SelectItemInputBoundary)として扱うことで
            具体的な実装を知らずに済むようにしている。

        payment_manager:
            現在の投入金額など、金銭状態を保持するエンティティ。
            自販機にお金を入れてから買う、という一連の流れで共有される。
        """
        self._select_item_usecase = select_item_usecase
        self._payment_manager = payment_manager

    def select_item(self, slot_id: str):
        """
        [Controllerのメインメソッド例]
        ユーザーが「このスロットの商品を買いたい」と要求したときに呼ばれる。

        ここでやることは3つだけ：
          1. DTO(SelectItemInputData)を組み立てる
          2. UseCaseのhandle()を呼ぶ
          3. PaymentManagerも一緒に渡す

        Controllerは成功/失敗そのものをprintしない。
        → 実際の表示は Presenter & View が担当する。
        → ハードウェアの動作もここでは触らない。
        """
        # 1. ユースケースに渡す正式なリクエストDTOを作る
        input_data = SelectItemInputData(slot_id=slot_id)

        # 2. ユースケースを実行する
        #    - UseCase内で在庫チェック／支払い処理／お釣り計算／ハード指示など
        #      一連の購入フローが走る
        self._select_item_usecase.handle(
            input_data=input_data,
            payment_manager=self._payment_manager,
        )
```

---

## 🔍 コードの見どころ

### 1. `VendingMachineController` はビジネスを知らない

在庫チェックも、お金が足りるかの判断もしません。
「それはUseCaseに聞いてください」というスタンスです。

これはめちゃくちゃ重要で、
**Controllerが太り始めると一気に“全部ここでやればよくない？”状態になる**ので要注意です。
太ったControllerはフレームワーク依存になり、再利用できなくなります。

### 2. DTOを組み立ててからUseCaseに渡す

```python
input_data = SelectItemInputData(slot_id=slot_id)
self._select_item_usecase.handle(input_data, self._payment_manager)
```

ポイントは、素の値（ただの文字列 `"A1"`）をそのまま渡さないこと。
「ユースケースに渡す公式な入力フォーマット（SelectItemInputData）」に包んでから渡します。

これで、境界をまたいだ時点で「データの形が保証される」ようになります。
バラバラの辞書や生の引数を渡し始めると、急にカオスになります。

### 3. `payment_manager` を Controller が握っている

なぜ `PaymentManager` を Controller が持つかというと、
自販機の利用中、「投入中の金額」という状態は現在のユーザーセッションに紐づいて生き続けるからです。

* 100円入れる
* さらに100円入れる
* それから商品を買う

という一連の流れの間、`PaymentManager.current_amount` はずっと更新され続けます。
これをどこに保持するか？という答えのひとつが「Controllerが握る」なんです。

（別の設計もあり得ます。例えばPaymentManager用の専用Adapterを作って永続管理する、など。そこは後で拡張できる部分です）

---

## 🧪 Controllerのユニットテスト

Controllerのテストは「正しいユースケースを、正しい形で呼んでいるか？」を確認します。
ビジネスの結果はPresenter/Viewで確認すればOKなので、Controllerテストではそこまで踏み込みません。

```python
# tests/interface_adapters/test_controller.py

from vending_machine.interface_adapters.controller import VendingMachineController
from vending_machine.usecase.dto import SelectItemInputData
from vending_machine.domain.entities import PaymentManager


class FakeSelectItemUseCase:
    """
    SelectItemInputBoundary を満たすテスト用の偽物。
    呼ばれたときの引数を覚えておくスパイ役。
    """
    def __init__(self):
        self.called_with_input = None
        self.called_with_pm = None

    def handle(self, input_data, payment_manager):
        self.called_with_input = input_data
        self.called_with_pm = payment_manager


def test_Controllerは_usecaseを正しく呼び出す():
    # 1. Arrange
    fake_usecase = FakeSelectItemUseCase()
    pm = PaymentManager(current_amount=200)
    controller = VendingMachineController(
        select_item_usecase=fake_usecase,
        payment_manager=pm,
    )

    # 2. Act
    controller.select_item(slot_id="A1")

    # 3. Assert
    # ユースケースが正しく呼ばれたか？（= Controllerは受付係として仕事したか？）
    assert isinstance(fake_usecase.called_with_input, SelectItemInputData)
    assert fake_usecase.called_with_input.slot_id == "A1"
    assert fake_usecase.called_with_pm is pm
```

チェックしてるのはたったこれだけです：

* ControllerはDTOをちゃんと作って渡した？
* Controllerは正しいPaymentManagerを渡した？
* その結果、UseCaseが呼ばれた？

これだけでもう、Controllerの責務は満たせているといえます。

---

## 🐍 Python と 🔧 C言語のイメージ比較

* 🐍 Python / クリーンアーキテクチャ的なController

  * 「外部入力（ボタン・HTTP・CLI）を受けて、UseCaseに橋渡しする役」
  * フレームワークにべったりしない。
  * 将来、CLIからWeb APIに変えても、Controllerを差し替えるだけで済む。

* 🔧 C言語ベースの組み込みあるある

  * メインループの中で「ボタン押されたら直接在庫変数いじって、直接モーター回して、直接printfする」
  * つまり受付・ビジネス・ハード・表示が1カ所にベタッと固まる
  * → 部分的なテストができない
  * → 仕様変更すると一気に全部直す羽目になる

Controllerを分ける狙いは、まさにこの「ごちゃまぜロジック」を分解することにあります。

---

## 🛡️ Controller層の鉄則

> 正しいユースケースを、正しい形で呼べ。
> それ以外のことは他の層に任せろ。

* Controllerは“受付窓口”。
* UseCaseに「これお願いします」と依頼するのが仕事。
* それ以上はやらない（やらせない）。

こうして、

* UseCaseは「ビジネスの流れ」
* Presenterは「見せ方」
* HardwareAdapterは「物理的な動き」
* Controllerは「入力の取りまとめ」
  という役割分担が、きれいに四分割で揃います。


