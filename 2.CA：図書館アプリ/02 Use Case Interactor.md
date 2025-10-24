# 02 Use Case Interactor

# 👨‍🏫 Use Case : `application/use_cases/check_out_book.py`

Entityという部品を指揮して具体的なシナリオを実現する、**UseCase**を解説します。今回は「本を貸し出す」という、このシステムのメインとなるユースケースを見ていきましょう。

## 🎯 このクラスの役割

**`CheckOutBookUseCase`は、図書館の「司書」のように、「本を貸し出す」という一つの業務シナリオ（ユースケース）の実行に責任を持つ**クラスです。

`Entity`が持つ個別のルール（例：「貸出中の本は貸し出せない」）を尊重しつつ、複数の`Entity`（`Book`、`Member`、そして`Loan`）や外部の機能（データベース）を協調させて、業務全体の流れを制御（オーケストレーション）します。

ここには、**個々のEntityだけでは完結しない、アプリケーション固有のビジネスルール**が実装されます。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

## ✅ このクラスにあるべき処理

⭕️ **含めるべき処理の例:**

- **複数のEntityの指揮（オーケストレーション）**:
    - `MemberRepository`から会員を探し、`BookRepository`から本を探し、それらの情報から新しい`Loan` Entityを作成する、といった複数のデータソースやEntityをまたいだ処理。
- **Entityをまたがるビジネスルール**:
    - 「会員資格が有効で、かつ、本の在庫がある場合にのみ貸出処理を行う」といった、複数のEntityの状態に依存する判断。
- **外部への要求（インターフェースを通じた呼び出し）**:
    - `boundaries.py`で定義されたインターフェースを通じて、`Repository`にデータの永続化を依頼したり、`Presenter`に結果の表示準備を依頼したりすること。

❌ **含めてはいけない処理の例:**

- **Entityが持つべきルールの重複実装**:
    - 「本のステータスが貸出可能か？」というチェックは、`book.check_out()`メソッドを呼び出すことで`Book` Entity自身に任せるべきです。`UseCase`が`if book.status == ...`のようなチェックを直接行うのは、責務の侵害になります。
- **UIやデータベース、フレームワークに関する知識やコード。**

## 💻 ソースコードの詳細解説

（※このコードは、後ほど作成する`boundaries.py`と`data_structures.py`に依存していることを前提とします。）

```python
# application/use_cases/check_out_book.py

from datetime import date
from boundaries import (
    CheckOutBookInputBoundary, CheckOutBookOutputBoundary,
    BookDataAccessInterface, MemberDataAccessInterface, LoanDataAccessInterface
)
from data_structures import CheckOutBookInputData, CheckOutBookOutputData
from domain.entities import Book, Member, Loan # 内側のEntityに依存

# -----------------------------------------------------------------------------
# Use Case
# - クラス図の位置: UseCase
# - 同心円図の位置: Use Cases (内側から2番目の円)
# -----------------------------------------------------------------------------
class CheckOutBookUseCase(CheckOutBookInputBoundary):
    """「本を貸し出す」というユースケースの実装"""
    def __init__(self,
                 presenter: CheckOutBookOutputBoundary,
                 book_repository: BookDataAccessInterface,
                 member_repository: MemberDataAccessInterface,
                 loan_repository: LoanDataAccessInterface):
        """
        [依存性の注入 (Dependency Injection)]
        Presenterと、今回は3種類のRepositoryを、すべて抽象的なインターフェースとして受け取る。
        このクラスは、具体的なDB実装やUI実装を一切知らない。
        """
        self._presenter = presenter
        self._book_repository = book_repository
        self._member_repository = member_repository
        self._loan_repository = loan_repository

    def handle(self, input_data: CheckOutBookInputData):
        """
        [ビジネスロジックの実行]
        本を貸し出すための具体的な処理手順（レシピ）を記述する。
        """
        # 手順1: 関連するEntityをリポジトリ経由で取得する
        book = self._book_repository.find_by_id(input_data.book_id)
        member = self._member_repository.find_by_id(input_data.member_id)

        # 手順2: 事前条件のチェック
        if not book:
            raise ValueError("指定された本が見つかりません。") # 本来はエラー用Presenterを呼ぶ
        if not member:
            raise ValueError("指定された会員が見つかりません。")

        # 手順3: Entityにビジネスルールを実行させる（委任）
        # 「貸出可能か？」の判断はBook Entity自身が行う
        try:
            book.check_out()
        except ValueError as e:
            # Entityがルール違反（例：貸出中の本を貸し出そうとした）でエラーを投げたら、それを捕捉して処理する
            raise e

        # 手順4: このユースケース固有のロジックを実行 -> 新しいLoan Entityを作成
        # 「貸出」というイベントが発生した記録(Loan)を生成する
        new_loan = Loan(id=None, # IDは永続化層で採番される想定
                        book_id=book.id,
                        member_id=member.id,
                        loan_date=date.today())

        # 手順5: 変更を永続化する (オーケストレーション)
        self._book_repository.save(book)       # 本の状態を更新
        self._loan_repository.save(new_loan) # 新しい貸出記録を作成

        # 手順6: 結果を出力データ（OutputData）に変換し、Presenterに渡す
        output_data = CheckOutBookOutputData(
            book_title=book.title,
            member_name=member.name,
            due_date=new_loan.due_date # 返却日はLoan Entityが計算してくれる
        )
        self._presenter.present(output_data)

```

- **`CheckOutBookUseCase`**: クラス名を`Interactor`から`UseCase`に変更し、その責務が「ユースケースの実現」であることを明確にしました。
- **`__init__`**: `Loan`という貸出記録を永続化する必要があるため、新たに`LoanDataAccessInterface`を受け取ります。部品が増えても、同じ原則（**抽象に依存する**）で綺麗に管理できます。
- **`handle`メソッド**:
    - `book.check_out()`を呼び出し、`Book` Entityに状態変更の仕事を**委任**しています。
    - **このユースケースの核心**として、`Loan` Entityを**新規作成**しています。これは`Book`や`Member`単体では完結しない、この「貸出」という**シナリオ固有のルール**です。
    - 最後に、変更された`Book`と新しい`Loan`の両方を、それぞれの`Repository`に**永続化**するよう依頼しています。このように複数の部品を指揮するのがUseCaseの重要な役割です。

## 💡 ユニットテストでUseCaseの正しさを証明する

UseCaseのテストでは、**ビジネスの「流れ」が正しいか**を検証します。DBやPresenterはモック（偽物）に差し替え、純粋なロジックだけをテストします。

```python
# tests/application/use_cases/test_check_out_book.py の例

def test_貸出シナリオが正しく実行される():
    # 1. Arrange (準備): すべての依存先をモック（偽物）で用意する
    mock_presenter = MockPresenter()
    mock_book_repo = MockBookRepository(貸出可能な本を返す設定)
    mock_member_repo = MockMemberRepository(有効な会員を返す設定)
    mock_loan_repo = MockLoanRepository()

    # 2. Act (実行): テスト対象のUseCaseを、モックを注入して作成し、実行する
    use_case = CheckOutBookUseCase(
        mock_presenter, mock_book_repo, mock_member_repo, mock_loan_repo
    )
    input_data = CheckOutBookInputData(book_id=1, member_id=1)
    use_case.handle(input_data)

    # 3. Assert (検証): UseCaseが外部と正しく連携したか（メソッドを呼んだか）を確認
    # 意図: 「貸出処理の最後に、ちゃんと本と貸出記録をDBに保存しようとしたか？」をテスト
    assert mock_book_repo.save_was_called_with_updated_book is True
    assert mock_loan_repo.save_was_called_with_new_loan is True
    # 意図: 「処理の最後に、ちゃんとPresenterに成功結果を渡そうとしたか？」をテスト
    assert mock_presenter.present_was_called_with_success_data is True

```

- **テストの意図**: このテストは、`CheckOutBookUseCase`が、(1)本と会員を見つけ、(2)`Book`の状態を変更し、(3)新しい`Loan`を作り、(4)両方を保存し、(5)最後にPresenterを呼ぶ、という正しい手順（レシピ）を踏んでいるかを検証します。DB接続や画面表示のコードは一切不要です。

## 🐍 PythonとC言語の比較（初心者の方へ）

- **Python (オブジェクト指向)**: `CheckOutBookUseCase`というクラス（設計図）として、「本の貸出」という責務をカプセ-ル化します。`use_case.handle()`のように、オブジェクト自身が責任を持ってビジネスフローを実行します。
- **C言語 (手続き型)**: もしC言語で書くなら、`check_out_book(book_id, member_id)`のような**巨大な関数**を一つ作ることになるでしょう。その関数の中に、DBからデータを読む処理、ルールをチェックする処理、DBに書き込む処理がすべて順番に書かれます。これではテストが難しく、変更にも弱くなりがちです。
- **クリーンアーキテクチャ**: Pythonのクラスを使い、**一つのクラスには一つの責務**（この場合は「本の貸出シナリオの実行」）を持たせることで、コードを整理し、テストしやすく、変更しやすい状態に保ちます。

## 🛡️ このクラスの鉄則

`Use Case`層の鉄則は、システムの防波堤となることです。

> ビジネスの流れを指揮せよ、ただし詳細には関わるな。 (Orchestrate the business flow, but delegate the details.)
> 
- このクラスは、`Entity`に任せるべきこと（本質的なルール）と、自身が担当すべきこと（アプリケーションのシナリオ）を明確に区別します。
- このクラスのおかげで、将来「貸出期間を会員ランクによって変える」といったルール変更があっても、修正箇所はこのファイルに限定され、`Entity`やUI層に影響が及ぶのを防ぐことができます。