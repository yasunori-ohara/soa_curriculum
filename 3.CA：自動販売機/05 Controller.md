# 05 Controller

### `interface_adapters/controllers/checkout_controller.py` — `CheckOutBookController`

---

## 🎯 役割

`CheckOutBookController` は、**ユーザー入力（View側で取得した値）を、ユースケース（業務処理）に正しく渡すための中継点**です。

より具体的には、次の責務だけを持ちます。

1. View から受け取った入力値（例：`book_id`, `member_id`）を引数として受け取る
2. それらをユースケースの公式な入力形式である `CheckOutBookInputData`（DTO）にまとめる
3. ユースケースの入り口インターフェース `CheckOutBookInputBoundary` を呼び出す

これだけです。

☑ Controller はビジネスロジックを判断しません
☑ Controller は画面を更新しません
☑ Controller はDBに触れません

Controller はあくまで「橋渡し役」「交通整理役」です。業務ルールそのものは扱いません。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

## 🧭 全体の中での位置づけ

プロジェクト構成の中で見ると、Controller は `interface_adapters/` に所属します。
これは「UI層・外部インターフェース層」にあたります。

```text
├─ core/
│   ├─ domain/
│   │   ├─ book.py
│   │   ├─ member.py
│   │   ├─ loan.py
│   │   └─ errors.py
│   │
│   └─ usecase/
│       ├─ dto.py                 # <DS> DTO / ViewModel
│       ├─ boundaries.py          # <I> 入出力インターフェース群
│       ├─ check_out_book_usecase.py
│       └─ (他のユースケース…)
│
└─ interface_adapters/
    ├─ controller.py              # ← Controller
    ├─ presenter.py               # Presenter
    ├─ view_console.py            # View（CLI実装）
    ├─ data_access.py             # データ取得・保存の具体実装（インメモリなど）
    └─ hardware_adapter.py        # ハードウェア操作の具体実装
```

ここで押さえたい依存方向は次のとおりです。

* Controller（interface_adapters 層）は、UseCase 層が定義する抽象インターフェース（Boundary）を呼び出します
* UseCase 層は、Controller の存在も名前も知りません

つまり依存は「外側 → 内側」だけに流れます。
内側（ビジネスロジック）は、外側のUI・フレームワーク・入出力手段に縛られません。

---

## 💻 コード

以下は Controller の具体例です。
ユーザーの「本を借りたい」という操作をユースケースに受け渡す責務に特化しています。

```python
# --------------------------------------------------------------------
# File: interface_adapters/controller.py
# Layer: Interface Adapters（Controller）
#
# 目的:
#   - View（画面/UI）から渡された入力値を公式なDTOにまとめ、
#     ユースケースの入り口（InputBoundary）を呼び出す。
#
# 重要:
#   - Controller はビジネスロジックを実行しない。
#   - Controller は画面表示を担当しない。
#   - Controller はDBやハードウェアに触れない。
#
# 依存関係:
#   - CheckOutBookInputBoundary（ユースケース側が定義する「入口インターフェース」）
#   - CheckOutBookInputData（ユースケース側で定義された入力DTO）
#
# 逆方向の依存は存在しない:
#   - ユースケース層はこの Controller のことを知りません。
#   - これにより、ビジネスロジックはUI技術から分離されます。
# --------------------------------------------------------------------

from vending_machine.usecase.boundaries import (
    CheckOutBookInputBoundary,
    CheckOutBookInputData,
)


class CheckOutBookController:
    """
    View と UseCase のあいだの橋渡しを行う Controller。

    View（例: ConsoleView）はユーザーから受け取った入力値を
    このコントローラのメソッドに渡すだけでよい。

    Controller は、その値をユースケースが理解できる
    公式なDTO(CheckOutBookInputData)に変換し、
    ユースケースの入り口(CheckOutBookInputBoundary.handle)を呼び出す。
    """

    def __init__(self, use_case: CheckOutBookInputBoundary):
        """
        Parameters
        ----------
        use_case : CheckOutBookInputBoundary
            - 「本を貸し出す」ユースケースを表すインターフェース。
            - 実際の実装（CheckOutBookUseCaseなど）は
              main.py から依存注入（DI）される。
        """
        self._use_case = use_case

    def check_out(self, *, book_id: int, member_id: int) -> None:
        """
        View から呼ばれる公開メソッド。

        「この会員がこの本を借りたい」というリクエストを受け取り、
        ユースケースに処理を委ねる。

        ここではビジネス判断は行わない。
        あくまで値をまとめて渡すだけ。
        """

        # 1️⃣ View から渡された生の入力値を、
        #     ユースケースが期待するDTO(入力データ構造)に詰め直す。
        request_dto = CheckOutBookInputData(
            book_id=book_id,
            member_id=member_id,
        )

        # 2️⃣ ユースケースの "公式な入口" を呼び出す。
        #     Controller はユースケースの中身を知らない。
        self._use_case.handle(request_dto)
```

---

## 🔎 コードで押さえるべきポイント

### 1. `CheckOutBookInputBoundary` に依存している

```python
def __init__(self, use_case: CheckOutBookInputBoundary):
    self._use_case = use_case
```

* Controller はユースケースの「具体クラス名」を知りません。
  たとえば `CheckOutBookUseCase` という実装クラスのことは知らない想定です。
* 知っているのは、ユースケース層が提示している「契約（インターフェース）」だけです。

→ これにより、ユースケースの中身を差し替えても Controller のコードは変わりません。
　テストでもモック（偽物）を渡すだけで動きを確認できます。

---

### 2. 入力を DTO にまとめてから渡す

```python
request_dto = CheckOutBookInputData(
    book_id=book_id,
    member_id=member_id,
)
self._use_case.handle(request_dto)
```

* Controller は「生の引数」をそのままユースケースに渡しません。
* 必ずユースケースで定義された正式な“箱”（`CheckOutBookInputData`）に詰めます。

これは2つの意味で重要です。

* ✅ ユースケースが「どの情報をもらえるか」を自分でコントロールできる
  （UIに依存せず、自分のほしいフォーマットを要求できる）

* ✅ UI 側（View 側）は「とにかく book_id と member_id を渡せばいい」という単純な責務になる
  （ビューはビジネスロジックを考えなくてよい）

---

### 3. Controller は「判断」しない

Controller は、貸し出し可否などのビジネスルールを決めません。
その判断はユースケースに一任します。

ダメな例：

```python
# ❌ Controller が勝手にビジネス判断している悪い例
if member_id == 0:
    raise Exception("会員登録されていないので貸出不可です")

# ❌ Controller が勝手に状態を変更している悪い例
db_session.update("books", book_id, status="CHECKED_OUT")
```

こういった処理は Controller の責務ではありません。
「何が正しいか」を決めるのはビジネスルール側（UseCase / Entities）です。

---

## ✅ Controller に含めてよい処理 / 含めてはいけない処理

### ⭕ 含めてよい

* View から渡された引数を受け取る
* DTO（`CheckOutBookInputData`）を生成する
* ユースケースの `handle()` を呼ぶ
* （必要であれば）UIレベルの軽い調整
  例：文字列 `"42"` を `int(42)` に変換するなど

### ❌ 含めてはいけない

* ビジネスロジックの判断（権限チェックや貸出上限など）
* 見せ方の整形（メッセージ文、日付のフォーマット、敬称づけ）
* 画面出力（`print` や HTML レンダリングなど）
* DB やハードウェアへの直接アクセス

これらを Controller に入れてしまうと、後で UI を差し替えるときや、ビジネスルールを拡張するときにすぐ壊れてしまいます。

---

## 🧪 ユニットテスト例（テストしやすさが売りです）

Controller は非常にテストしやすい層です。
理由は「ユースケースの抽象を呼ぶだけ」だからです。

下はテスト例です。
狙いは「Controller が正しい DTO を組み立てて、ユースケースの `handle()` を呼んでいるか」を確認することです。

```python
# tests/interface_adapters/test_controller.py

from vending_machine.usecase.boundaries import (
    CheckOutBookInputBoundary,
    CheckOutBookInputData,
)
from interface_adapters.controller import CheckOutBookController


class SpyUseCase(CheckOutBookInputBoundary):
    """
    テスト用のダミー実装。
    Controller が正しく呼んでくれたかどうかを記録する。
    """
    def __init__(self):
        self.called = False
        self.received_input = None

    def handle(self, input_data: CheckOutBookInputData) -> None:
        self.called = True
        self.received_input = input_data


def test_controllerは_DTOを生成して_use_case_handleを呼び出す():
    # Arrange: ダミーのユースケースを注入
    spy_use_case = SpyUseCase()
    controller = CheckOutBookController(use_case=spy_use_case)

    # Act: View から呼ばれる想定のメソッドを直接叩く
    controller.check_out(book_id=10, member_id=99)

    # Assert: ユースケースが呼ばれたこと
    assert spy_use_case.called is True

    # Assert: 渡されたDTOの中身が正しいこと
    assert isinstance(spy_use_case.received_input, CheckOutBookInputData)
    assert spy_use_case.received_input.book_id == 10
    assert spy_use_case.received_input.member_id == 99
```

このテストでは、

* 実際のユースケース実装（ビジネスロジック）は一切動かしていません
* Repository も Presenter も View も不要です
* Controller 単体をピンポイントで検証できます

→ Controller を「痩せた責務」に保っているからこそ、こうしたテストが可能になります。

---

## 🧠 C言語との比較イメージ

C言語スタイルでは、ユーザー入力の処理、ビジネスロジックの呼び出し、そして画面出力がひとつの関数（例えば `main()` や GUI のコールバック関数）に混ざりやすくなります。

```c
// C的によくあるパターン（全部まとめてしまう）
void on_button_click() {
    int book_id = read_input();
    int member_id = read_input();

    if (!is_member_active(member_id)) {
        printf("利用できません\n");
        return;
    }

    checkout_book(book_id, member_id); // ドメイン直叩き
    printf("貸出しました\n");
}
```

このやり方だと、表示方法や取得方法を変えるたびに、この関数を直接編集することになります。

一方、クリーンアーキテクチャではこう分けます。

* View … 入力を受け、Controller を呼ぶ
* Controller … DTO を作ってユースケースに渡す
* UseCase … 実際の業務ロジックを実行する
* Presenter … 結果をユーザー向けメッセージに整形する
* View … 整形済みメッセージを表示する

→ 役割を分離することで、UIを差し替えても業務ロジックを壊さずに済みます。

---

## 🛡 まとめ（Controller の鉄則）

> 入力を正規化し、ユースケースに引き渡せ。
> それ以上は手を出さない。

* Controller は「ユーザーがこう操作した」という事実を、ユースケースが扱いやすい形式に翻訳する係です
* Controller はビジネスルールを決定しません
* Controller は表示もしません
* Controller の責務が痩せているほど、UIとビジネスロジックを安全に切り離せます

これにより、UI（コンソール→GUI→Web）を差し替えても、ユースケースやエンティティなど内側のロジックは影響を受けなくなります。これがクリーンアーキテクチャの狙いです。
