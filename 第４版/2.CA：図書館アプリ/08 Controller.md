# 08 Controller

# 🎨 Controller
### `interface_adapters/controllers/checkout_controller.py` - `CheckOutBookController`

---

## 🎯 このクラスの役割

`CheckOutBookController` は、**View から受け取ったユーザー入力を、Use Case が正式に受け取れる形に変換して渡す「交通整理員」**です。

* View は「ユーザーがボタンを押した」「入力欄に 12 と書いた」などの生データを持っています。
* Use Case は「会員ID=12 の利用者に、書籍ID=34 の本を貸し出してください」という業務リクエストを受けたい。

このギャップを埋めるのが Controller の仕事です。

Controller は以下だけをします：

1. View から渡された値を受け取る
2. それを公式な入力DTO（`CheckOutBookInputData`）にまとめる
3. Use Case の入力用インターフェース（`CheckOutBookInputBoundary`）を呼ぶ

それ以上のことはしません。
ビジネスルールの判断はしないし、画面も更新しません。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## 📁 フォルダ構成で見る Controller の位置

Controller は **Interface Adapters 層**に属します。
Use Case 層の抽象（InputBoundary）に依存しますが、逆方向の依存（Use Case → Controller）はありません。

```text
├─ core/
│   ├─ domain/
│   │   ├─ book.py
│   │   ├─ member.py
│   │   ├─ loan.py
│   │   └─ repository.py
│   │
│   └─ usecase/
│       ├─ boundary/
│       │   ├─ dto.py                # <DS> InputData / OutputData / ViewModel
│       │   ├─ input_boundary.py     # <I> InputBoundary (Controllerが呼ぶ)
│       │   └─ output_boundary.py    # <I> OutputBoundary (Presenterが実装)
│       │
│       └─ interactor/
│           └─ check_out_book.py     # <UC> 「本を貸し出す」ユースケース本体
│
└─ interface_adapters/
    ├─ controllers/
    │   └─ checkout_controller.py    # ← この章の主役
    ├─ presenters/
    │   └─ checkout_presenter.py
    └─ views/
        ├─ view_console.py
        ├─ view_gui.py
        └─ view_web.py
```

ポイント：

* Controller は `core` には入れません（内側はビジネスルールの世界なので UI 事情を持ち込まない）。
* Controller は Use Case のインターフェースに依存することで、具体的な Use Case 実装（クラス名など）を知らずに動けるようになります。

---

## 💻 ソースコード（正式版）

```python
# --------------------------------------------------------------------
# File: interface_adapters/controllers/checkout_controller.py
# Layer: Interface Adapters（Controller）
#
# 責務:
#   - View から渡された入力値を公式な入力DTOに変換し、
#     Use Case(InputBoundary) に処理を依頼する。
#
# 依存:
#   - <I> CheckOutBookInputBoundary（core.usecase.boundary.input_boundary）
#   - <DS> CheckOutBookInputData（core.usecase.boundary.dto）
#
# 同心円図での位置:
#   - Interface Adapters 層
#   - 内側(core.usecase.boundary.*)には依存するが、
#     内側はこの Controller を知らない一方向依存
# --------------------------------------------------------------------

from core.usecase.boundary.input_boundary import CheckOutBookInputBoundary
from core.usecase.boundary.dto import CheckOutBookInputData


class CheckOutBookController:
    """
    View と Use Case のあいだに立つ「交通整理員」。

    - ユーザーからの入力を受ける（View から呼ばれる）
    - その値を CheckOutBookInputData に詰める
    - use_case.handle(...) を呼び出す

    それ以外は何もしない。
    """

    def __init__(self, use_case: CheckOutBookInputBoundary):
        """
        use_case:
            CheckOutBookInputBoundary を実装したオブジェクト。
            （典型的には core.usecase.interactor.check_out_book.CheckOutBookUseCase ）
        """
        self._use_case = use_case

    def check_out(self, book_id: int, member_id: int) -> None:
        """
        View 層から「この本をこの人に貸し出したい」というリクエストを受ける想定のメソッド。

        - book_id / member_id は View 側が集めた生の入力。
        - ここで公式DTO CheckOutBookInputData にまとめる。
        - あとは Use Case に丸投げする。
        """

        # 1. Viewの素の入力値を、Use Case が理解できる正式な入力DTOに変換
        input_data = CheckOutBookInputData(
            book_id=book_id,
            member_id=member_id,
        )

        # 2. Use Case(InputBoundary) に処理を依頼
        self._use_case.handle(input_data)
```

---

## ✅ このファイルにあるべき処理

⭕️ **含めるべき処理の例:**

* 受け取った入力値を型安全な DTO（`CheckOutBookInputData`）にまとめる
* Use Case の「入り口」（InputBoundary）を呼ぶ
* 必要なら、簡単な型変換・前処理（例：文字列の "42" を int(42) にするなど）

❌ **含めてはいけない処理の例:**

* ビジネス条件の判断（「この会員は貸出停止か？」など）→ Use Case の責務
* 結果メッセージの整形（敬称付けや日付フォーマットなど）→ Presenter の責務
* 画面の更新（print / ラベル更新 / HTML返却など）→ View の責務
* DBに保存・検索する処理 → Repository の責務（＝さらに内側の層がやること）

Controller は「橋」や「ゲートウェイ」だと思ってください。太らせないのが正解です。

---

## 🧪 ユニットテスト例

Controller のユニットテストはめちゃくちゃシンプルです。
目的は「正しい DTO を渡して、Use Case の handle() を呼んでる？」だけ確認すること。

```python
# tests/interface_adapters/controllers/test_checkout_controller.py

from core.usecase.boundary.input_boundary import CheckOutBookInputBoundary
from core.usecase.boundary.dto import CheckOutBookInputData
from interface_adapters.controllers.checkout_controller import CheckOutBookController


class SpyCheckOutBookUseCase(CheckOutBookInputBoundary):
    """
    テスト用スパイ:
    - handle(...) が呼ばれたか
    - どんな引数で呼ばれたか
    を記録する。
    """
    def __init__(self):
        self.called = False
        self.received_input = None

    def handle(self, input_data: CheckOutBookInputData) -> None:
        self.called = True
        self.received_input = input_data


def test_controllerは_Viewからの入力をDTOにまとめて_use_case_handleを呼び出す():
    # Arrange: スパイUseCaseを注入
    spy_use_case = SpyCheckOutBookUseCase()
    controller = CheckOutBookController(use_case=spy_use_case)

    # Act: Viewから呼ばれる想定のメソッドを実行
    controller.check_out(book_id=10, member_id=99)

    # Assert: Use Case が正しく呼ばれたことを確認
    assert spy_use_case.called is True

    # DTOの中身が正しいか？
    assert isinstance(spy_use_case.received_input, CheckOutBookInputData)
    assert spy_use_case.received_input.book_id == 10
    assert spy_use_case.received_input.member_id == 99
```

このテストでは、

* Use Case の本物は使わない（ビジネスロジックは検証対象じゃないから）
* Presenter や View も使わない（表示やメッセージ整形も対象外だから）

つまり Controller はそれ単体でテスト可能です。
ちゃんと「薄い層」になっている証拠です。

---

## 🐍 Python と C のイメージ差し

* **Python / クリーンアーキテクチャ的な考え方**

  * Controller はクラス。
  * View はこのクラスのメソッドを呼ぶだけでよい。
  * Controller は Use Case の抽象（InputBoundary）だけを知り、Use Case の具体クラス名やDBの実装は知らない。

* **C 言語的な書き方だと？**

  * ボタン押下コールバック関数内で、直接ビジネス処理関数を呼び、さらにその場でエラーメッセージを `printf` したりしがち。
  * そうすると「UIイベント処理」「ビジネスロジック」「表示メッセージの作り方」が全部１つの関数に混ざり、分割もテストもしづらくなる。

Controller を分離するメリットはまさにここにあります。

---

## 🛡 鉄則

> 入力を整理し、然るべき窓口に渡せ。
> それ以上のことはしない。

* Controller は「ユーザーがこう操作した」という事実を、ビジネス世界に正しく届ける翻訳ゲート。
* Controller 自身はビジネス判断もUI表示も担当しない。
* その結果、View と Use Case はお互いを知らずに済む。依存も一方向になる。

これが、クリーンアーキテクチャ流の「交通整理員」です 🚦
