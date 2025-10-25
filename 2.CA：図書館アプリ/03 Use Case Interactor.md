# 02 Use Case Interactor

# 👨‍🏫 Use Case
### `core/usecase/interactor/check_out_book.py`

## 🎯 このクラスの役割

`CheckOutBookUseCase` は、図書館アプリにおける「本を貸し出す」という具体的な業務シナリオ（ユースケース）を実現するクラスです。

* `Book` / `Member` / `Loan` といった複数の Entity をまとめて扱い、
* 永続化（保存）を Repository に依頼し、
* 結果を Presenter に伝える

という「図書館の司書」のような役割を担います。

Entity 個別のルール（例：「貸出中の本は貸し出せない」）を尊重しつつ、それらを正しい順番で組み合わせて処理を成立させるのが Use Case の責務です。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## 📁 ファイルの配置

本ユースケースと、そのユースケースが依存する「契約（Boundary）」やDTOの置き場所は次のとおりです。

```text
├─ core/
│   ├─ domain/
│   │   ├─ book.py                   # <E> Book Entity
│   │   ├─ member.py                 # <E> Member Entity
│   │   ├─ loan.py                   # <E> Loan Entity
│   │   └─ repository.py             # <I> Repositoryインターフェース群
│   │
│   └─ usecase/
│       ├─ boundary/
│       │   ├─ input_boundary.py     # <I> InputBoundary（Controllerが呼ぶ）
│       │   ├─ output_boundary.py    # <I> OutputBoundary（Presenterが実装）
│       │   └─ dto.py                # <DS> Data Structures（Input/Output）
│       │
│       └─ interactor/
│           └─ check_out_book.py     # <UC> 本を貸し出すユースケース本体
│
├─ interface_adapters/
│   ├─ controllers/
│   │   └─ checkout_controller.py    # Controller（Viewが呼ぶ）
│   ├─ presenters/
│   │   └─ checkout_presenter.py     # Presenter（OutputBoundary実装）
│   └─ views/
│       └─ ...                       # CLI / GUI / Web など複数実装OK
│
└─ infrastructure/
    └─ repositories/
        ├─ in_memory_book_repository.py
        ├─ in_memory_member_repository.py
        └─ in_memory_loan_repository.py
        # ← Repositoryインターフェース群の具象実装（DB/メモリなど）
```

ポイント：

* `core/` は内側（ビジネスルール側）。

  * `domain/` は Entity（ビジネスのルールと状態）と、それらを取得・保存するための抽象リポジトリ（Repositoryインターフェース）。
  * `usecase/` はアプリ固有のシナリオ。「貸し出し」などの手順（Interactor）。
* `interface_adapters/` は外側の境界。Controller / Presenter / View はここ。
* `infrastructure/` は一番外。実際の保存方法（インメモリでもDBでも）が入る。

この構成は、TODOアプリと同じレイヤー分け・命名規則に揃えています。

---

## ✅ このクラスにあるべき処理

⭕️ **含めるべき処理の例:**

* **複数の Entity のオーケストレーション**

  * Member を探す
  * Book を探す
  * Book を「貸出中」に変更する
  * Loan（貸出記録）を作る
  * それぞれ保存する
* **ユースケース固有のビジネスルール**

  * 「会員が有効であること」
  * 「本が貸出可能であること」
  * 「貸出記録を残すこと」
* **外部コンポーネントへの依頼**

  * Repository に保存を依頼
  * Presenter に結果を渡す

❌ **含めてはいけない処理の例:**

* UI処理（print / input / HTML生成など）
* DB詳細（SQLやORMの具体操作など）
* Entityが本来持つべきルールの複製
  例：`if book.status == ...` と直接判定せず、`book.check_out()` を呼んで Book に判断させる

---

## 🔍 ソースコード（正式版）

```python
# --------------------------------------------------------------------
# File: core/usecase/interactor/check_out_book.py
# Layer: Use Case（アプリケーション固有のビジネスルール）
#
# 目的:
#   - 「本を貸し出す」という業務シナリオを実行する。
#   - Entity同士とRepositoryを正しい順序で調整し、
#     最終的な結果をPresenterに渡す。
#
# C言語の感覚に近い説明:
#   - このクラスは「貸出処理」という大きな関数の呼び出し役。
#   - Repositoryインターフェースは「関数ポインタの束」に相当し、
#     実際にどこからデータを読むか（DB？メモリ？）は知らなくてよい。
# --------------------------------------------------------------------

from datetime import date

from core.usecase.boundary.input_boundary import CheckOutBookInputBoundary
from core.usecase.boundary.output_boundary import CheckOutBookOutputBoundary
from core.usecase.boundary.dto import (
    CheckOutBookInputData,
    CheckOutBookOutputData,
)

from core.domain.repository import (
    BookRepository,
    MemberRepository,
    LoanRepository,
)

from core.domain.book import Book
from core.domain.member import Member
from core.domain.loan import Loan


class CheckOutBookUseCase(CheckOutBookInputBoundary):
    """
    「本を貸し出す」というユースケース（Interactor）

    - Controller はこのクラスを「入力契約（InputBoundary）」として呼び出す。
    - Presenter（OutputBoundary実装）と各Repository（抽象）を注入して使う。
    """

    def __init__(
        self,
        presenter: CheckOutBookOutputBoundary,
        book_repository: BookRepository,
        member_repository: MemberRepository,
        loan_repository: LoanRepository,
    ):
        """
        [依存性の注入 (Dependency Injection)]

        - presenter:
            画面表示用の情報を組み立てる担当（OutputBoundaryの実装）
        - book_repository / member_repository / loan_repository:
            データ取得・保存担当（Repositoryインターフェース）
        """
        self._presenter = presenter
        self._book_repository = book_repository
        self._member_repository = member_repository
        self._loan_repository = loan_repository

    def handle(self, input_data: CheckOutBookInputData) -> None:
        """
        [貸出シナリオの本体]
        1. 会員と本を取得
        2. 状態チェック
        3. 貸出実施（BookやLoanに委譲）
        4. 永続化
        5. Presenterに結果を渡す
        """

        # 1. 関連するEntityをRepository経由で取得
        book: Book | None = self._book_repository.find_by_id(input_data.book_id)
        member: Member | None = self._member_repository.find_by_id(input_data.member_id)

        # 2. 事前条件のチェック（存在チェックなど）
        if not book:
            raise ValueError("指定された本が見つかりません。")
        if not member:
            raise ValueError("指定された会員が見つかりません。")

        # ここで会員資格チェックなどを行うなら member 側に責務を持たせる
        # 例: if not member.is_active(): raise ValueError("無効な会員です。")

        # 3. Book自身のルールを使って貸出処理を適用
        #    （「貸出可能？」などのビジネスルールはEntityに委譲）
        book.check_out()  # Book側がNGなら ValueError を投げる

        # 4. このユースケース固有の処理：
        #    新しい Loan を作成する（貸出記録そのもの）
        new_loan = Loan(
            id=None,  # 採番はRepository側の責務
            book_id=book.id,
            member_id=member.id,
            loan_date=date.today(),
        )

        # 5. 永続化（Bookの状態更新＋Loan記録の保存）
        self._book_repository.save(book)
        self._loan_repository.save(new_loan)

        # 6. Presenterに渡すための出力データを組み立てる
        output_data = CheckOutBookOutputData(
            book_title=book.title,
            member_name=member.name,
            due_date=new_loan.due_date,  # Loanが期限計算してくれる
        )

        # 7. Presenter（=出力境界）に渡す
        self._presenter.present(output_data)
```

---

## 🧪 この段階でユニットテストができます

Use Case は、外部の具体実装（DBやUI）には依存せず、**抽象インターフェース（Boundary / Repository）を注入するだけ**で動きます。
つまり、テストではフェイクの Presenter / Repository を渡すだけで、完全にローカルで検証できます。

```python
# tests/unit/usecase/test_check_out_book.py など

def test_貸出シナリオが正しく実行される():
    # 1. Arrange: フェイク依存を注入
    presenter = FakePresenter()                # OutputBoundaryのフェイク
    book_repo = FakeBookRepository()           # BookRepositoryのフェイク
    member_repo = FakeMemberRepository()       # MemberRepositoryのフェイク
    loan_repo = FakeLoanRepository()           # LoanRepositoryのフェイク

    # ここで、フェイクRepoが返す初期データとして
    # - 貸出可能なBook
    # - 有効なMember
    # を事前に登録しておくイメージ

    use_case = CheckOutBookUseCase(
        presenter,
        book_repo,
        member_repo,
        loan_repo,
    )

    # 2. Act: ユースケース実行
    input_data = CheckOutBookInputData(book_id=1, member_id=42)
    use_case.handle(input_data)

    # 3. Assert: 正しい一連の流れが呼ばれたか
    # - Bookが「貸出中」に変わった
    # - Loanが保存された
    # - Presenterが成功結果を受け取った
    assert book_repo.saved_book.status == "貸出中"  # 実装に合わせて書き換え
    assert loan_repo.saved_loan is not None
    assert presenter.present_was_called is True
```

テストの目的は、

* 正しい順序で処理しているか
* 依存先（Repository や Presenter）にちゃんと依頼しているか
  を確認することです。DB接続もUI表示も不要です。

---

## 🐍 PythonとC言語の比較（初心者向け）

* **Python / クリーンアーキテクチャの場合**
  `CheckOutBookUseCase` というクラスに「貸し出し」という責務を閉じ込め、
  `use_case.handle(input_data)` の1呼び出しでビジネスフロー全体が進む。

* **C言語の場合**
  1つの巨大な関数 `check_out_book(book_id, member_id)` を書き、
  その中で DBアクセス、状態チェック、在庫更新、表示用メッセージ作成など
  すべてを直列に書きがち。
  → テストしづらい・差し替えづらい・影響範囲が広い。

* **クリーンアーキテクチャの狙い**
  「貸し出しフロー」は Use Case に集約し、
  「保存方法」は Repository の実装に任せ、
  「表示の整形」は Presenter に任せ、
  役割を分割することで変更に強くする。

---

## 🛡️ このクラスの鉄則

> ビジネスの流れを指揮せよ、ただし詳細には関わるな。
> (Orchestrate the business flow, but delegate the details.)

* Use Case は「何をするか（フローと条件）」を定義する。
* 具体的な「どうやって保存するか」「どうやって見せるか」は他の層に丸投げする。
* これにより、UIをCLI→Web→GUIに置き換えても、DBをメモリ→Postgresに置き換えても、このユースケースクラスを変更せずに済む。これがアーキテクチャの耐久性そのもの。
