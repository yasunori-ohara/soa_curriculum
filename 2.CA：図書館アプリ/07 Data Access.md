# 07 Data Access

# 💾 Data Access : `adapters/data_access.py`

`UseCase`が要求した「契約書」（`boundaries.py`のインターフェース群）を、具体的に実装する**Adapters**層を見ていきましょう。

まずは、データベースとのやり取りを担当する**Data Access**について解説します。今回は本、会員、そして貸出記録という3種類のデータを扱うため、それぞれのリポジトリ（Data Accessクラス）を実装します。

## 🎯 このファイルの役割

このファイルは、`boundaries.py`で定義されたデータアクセスインターフェース（`BookDataAccessInterface`など）を、**具体的な技術（今回はインメモリ辞書）で実装するアダプター**を集めたものです。

`UseCase`からの「ID:1の本を探して」「この貸出記録を保存して」といった抽象的な要求を、Pythonの辞書を操作するという具体的なストレージへの読み書き処理に**変換**して実行します。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

## ✅ このファイルにあるべき処理

⭕️ **含めるべき処理の例:**

- `DataAccessInterface`で定義されたメソッドの具体的な実装。
- `Entity`を永続化するためのデータソース（辞書、ファイル、DBなど）とのやり取り。

❌ **含めてはいけない処理の例:**

- ビジネスロジック（`UseCase`の責務）。

## 💻 ソースコードの詳細解説

```python
# adapters/data_access.py

from typing import Dict, Optional

# 内側の世界のEntityと「境界」にのみ依存する
from domain.entities import Book, Member, Loan, BookStatus
from application.boundaries import (
    BookDataAccessInterface, MemberDataAccessInterface, LoanDataAccessInterface
)

# -----------------------------------------------------------------------------
# DataAccess (Repository)
# - クラス図の位置: DataAccess
# - 同心円図の位置: Adapters (Gateways)
# -----------------------------------------------------------------------------

class InMemoryBookDataAccess(BookDataAccessInterface):
    """BookDataAccessInterfaceの「インメモリDB」版実装"""
    _database: Dict[int, Book] = {}

    def __init__(self):
        # サンプルとして初期データをいくつか用意しておく
        self._database[1] = Book(id=1, title="クリーンアーキテクチャ", author="Robert C. Martin")
        self._database[2] = Book(id=2, title="達人プログラマー", author="Andrew Hunt")
        self._database[3] = Book(id=3, title="リファクタリング", author="Martin Fowler", status=BookStatus.CHECKED_OUT)

    def find_by_id(self, book_id: int) -> Optional[Book]:
        """[インターフェースの実装] 'find_by_id'を辞書検索で実現"""
        return self._database.get(book_id)

    def save(self, book: Book) -> Book:
        """[インターフェースの実装] 'save'を辞書への上書き保存で実現"""
        self._database[book.id] = book
        return book

class InMemoryMemberDataAccess(MemberDataAccessInterface):
    """MemberDataAccessInterfaceの「インメモリDB」版実装"""
    _database: Dict[int, Member] = {}

    def __init__(self):
        self._database[1] = Member(id=1, name="Alice")
        self._database[2] = Member(id=2, name="Bob")

    def find_by_id(self, member_id: int) -> Optional[Member]:
        """[インターフェースの実装] 'find_by_id'を辞書検索で実現"""
        return self._database.get(member_id)

class InMemoryLoanDataAccess(LoanDataAccessInterface):
    """LoanDataAccessInterfaceの「インメモリDB」版実装"""
    _database: Dict[int, Loan] = {}
    _next_id: int = 1 # IDを自動採番するためのカウンター

    def save(self, loan: Loan) -> Loan:
        """
        [インターフェースの実装] 'save'を辞書への新規保存で実現.
        IDが未設定の場合は自動で採番する。
        """
        if loan.id is None:
            loan.id = self._next_id
            self._next_id += 1
        self._database[loan.id] = loan
        return loan

```

- **3つのクラス**: `Book`, `Member`, `Loan`という3つのEntityそれぞれに対応する`DataAccess`クラスを用意しています。これは**単一責任の原則**に従い、責務を明確に分離するためです。
- **`InMemoryLoanDataAccess`**: `Loan`は`UseCase`によって動的に作成されるため、`save`メソッド内でIDを**自動採番**する簡単な仕組みを持っています。

## 💡 ユニットテストでDataAccessの正しさを証明する

DataAccessのテストでは、**永続化ロジックが正しく機能するか**を検証します。`save`した後に`find_by_id`で正しく取得できるか、などを確認します。

```python
# tests/adapters/test_data_access.py の例

def test_InMemoryBookDataAccessは本の保存と検索ができる():
    # 1. Arrange (準備): テスト対象のDataAccessオブジェクトを作成
    book_repo = InMemoryBookDataAccess()
    new_book = Book(id=99, title="テスト駆動開発", author="ケント・ベック")

    # 2. Act (実行): 新しい本を保存し、その後IDで検索する
    book_repo.save(new_book)
    found_book = book_repo.find_by_id(99)

    # 3. Assert (検証): 保存した本と、検索結果が一致するか確認
    # 意図: 「saveしたものが、ちゃんとfind_by_idで見つかるか？」をテスト
    assert found_book is not None
    assert found_book.title == "テスト駆動開発"

```

- **テストの意図**: このテストは、`InMemoryBookDataAccess`が持つ永続化ロジック（今回は辞書操作）に焦点を当てています。`UseCase`やUIは一切関係なく、このクラスが契約通りにデータを保存・取得できることだけを独立して検証できます。

## 🐍 PythonとC言語の比較（初心者の方へ）

- **Python (オブジェクト指向)**: `InMemoryBookDataAccess`のような**クラス**が、`BookDataAccessInterface`という契約（インターフェース）を実装します。将来データベースをPostgreSQLに変更したくなったら、同じインターフェースを実装した`PostgresBookDataAccess`クラスを新たに作り、**差し替える**だけで済みます。
- **C言語 (手続き型)**: `save_book_to_file(book)`や`find_book_from_file(book_id)`のような、特定のデータソース（ファイルなど）に密結合した**関数群**として実装されることが多いでしょう。データソースを変更するには、これらの関数をすべて書き直す必要があり、変更の影響が大きくなりがちです。

## 🛡️ このファイル群の鉄則

このファイル内のすべてのクラスは、同じ鉄則に従います。

> データを変換し、黙って実行せよ。 (Translate and execute, but do not think.)
> 
- これらのクラスはビジネスルールについて**思考しません**。`UseCase`という管理者からの指示を、技術的な作業として忠実に実行するだけの「作業員」です。
- このファイルのおかげで、`UseCase`はデータベースがメモリなのか、ファイルなのか、あるいはPostgreSQLなのかを一切気にせずに済みます。将来データベースを変更したくなった場合、影響範囲はこの`data_access.py`ファイルだけに限定されます。

## ❓ Q\&A

### **Q. `DataAccess`と`Gateway`は何が違うのですか？図によって場所が違うように見えます。**

**A.** 素晴らしい質問です。これらは密接に関連していますが、**概念のレベル**が異なります。

- **`Gateway` (同心円図の用語)**
    - **分類**: アーキテクチャ上の**役割**や**パターン** 🚪
    - **意味**: `UseCase`とデータベースのような**外部の何か**との通信を仲介する門（ゲートウェイ）の役割を指す、**抽象的な概念**です。同心円図では、`Presenter`などと同じ`Adapters`層に属する「役割」として描かれます。`DataAccessInterface`は、この`Gateway`の仕様を定義する契約書です。
- **`Data Access` (クラス図の用語)**
    - **分類**: `Gateway`の具体的な**実装** 🛠️
    - **意味**: `Gateway`という役割を、**具体的なコードで実装したもの**です。私たちの`InMemoryDataAccess`クラスがこれにあたります。クラス図では、アプリケーションのロジックとは切り離された、一番外側の「詳細」として、`Database`の近くに描かれることが多いです。

**結論として、「`Gateway`という設計上の役割を、`DataAccess`という具体的なクラスで実装している」と理解してください。** 同心円図は役割（`Gateway`）を示し、クラス図は具体的な実装部品（`DataAccess`）とその接続先を示しているため、見え方が異なるのです。