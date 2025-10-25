# 10 Data Access

# 💾 Data Access
### `interface_adapters/data_access/in_memory_repositories.py`

`Use Case` が要求した「契約書」（= Repositoryインターフェース）を、具体的な技術で満たすのが **Data Access（リポジトリ実装）** です。

この章では、データベースとのやり取りを担当するこの層を解説します。
今回はまだ本物のDBは使わず、インメモリ（Pythonの辞書）で実装します。

扱うデータは3種類：📘本（Book）、🧑会員（Member）、📄貸出記録（Loan）。
それぞれに対応するリポジトリ実装を用意します。

---

## 🎯 このファイルの役割

このファイルは、`core/domain/repository.py` で定義された抽象インターフェース（`BookRepository` など）を、**具体的な技術（ここではインメモリ辞書）で実装するアダプター**をまとめたものです。

`Use Case` からくるリクエストはいつも抽象的です。

* 「このIDの本を取ってきてください」
* 「この本の状態を保存してください」
* 「この貸出記録を新しく登録してください」

Data Access は、その抽象的な要求を、実際の保存技術（ここでは辞書操作）に変換して実行します。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## 📁 フォルダ構成における位置づけ

Data Access は **Interface Adapters 層** に属します。
内側（Use Case, Entity）には依存しますが、その逆はありません。

```text
├─ core/
│   ├─ domain/
│   │   ├─ book.py
│   │   ├─ member.py
│   │   ├─ loan.py
│   │   └─ repository.py           # ← 抽象: BookRepository / MemberRepository / LoanRepository
│   │
│   └─ usecase/
│       ├─ boundary/
│       │   ├─ dto.py
│       │   ├─ input_boundary.py
│       │   └─ output_boundary.py
│       │
│       └─ interactor/
│           └─ check_out_book.py   # ← Use Case (CheckOutBookUseCase)
│
└─ interface_adapters/
    ├─ data_access/
    │   └─ in_memory_repositories.py   # ← この章の主役（技術実装）
    ├─ presenters/
    │   └─ checkout_presenter.py
    ├─ controllers/
    │   └─ checkout_controller.py
    └─ views/
        └─ view_console.py
```

ポイント💡

* `core/domain/repository.py` が「契約書（抽象インターフェース）」
* `interface_adapters/data_access/in_memory_repositories.py` が「その契約を満たす具体的な実装」

内側（core）は外側（interface_adapters）を知らないまま生きていけます。
この“片思い状態”がクリーンアーキテクチャの安全性です。

---

## ✅ このファイルにあるべき処理

⭕️ **含めるべき処理の例:**

* Repositoryインターフェースで定義されたメソッド（`find_by_id` や `save`）の具体的な実装
* Entityを保存・更新・検索するためのストレージ操作（辞書、ファイル、DBなど）

❌ **含めてはいけない処理の例:**

* ビジネスルール（それは Use Case や Entity の責務）
* 出力用の文面整形（それは Presenter の責務）
* ユーザー入力のハンドリング（それは View / Controller の責務）

---

## 💻 ソースコードの詳細解説

```python
# --------------------------------------------------------------------
# File: interface_adapters/data_access/in_memory_repositories.py
# Layer: Interface Adapters（Gateway / Repository実装）
#
# 目的:
#   - core/domain/repository.py で定義した抽象Repositoryを、
#     「インメモリ辞書」という具体技術で実装する。
#
# 依存:
#   - 内側の Entity (Book, Member, Loan)
#   - 内側の Repositoryインターフェース
#
# 同心円図との対応:
#   - クラス図では「Data Access」
#   - 同心円図では「Gateway」
#   → どちらも、Use Case と外部（DBなど）をつなぐ役
# --------------------------------------------------------------------

from typing import Dict, Optional
from datetime import date

from core.domain.book import Book, BookStatus
from core.domain.member import Member
from core.domain.loan import Loan
from core.domain.repository import (
    BookRepository,
    MemberRepository,
    LoanRepository,
)


class InMemoryBookRepository(BookRepository):
    """BookRepository の「インメモリDB」版実装"""
    def __init__(self):
        # 簡易DBとして辞書を使用
        self._database: Dict[int, Book] = {
            1: Book(id=1, title="クリーンアーキテクチャ", author="Robert C. Martin"),
            2: Book(id=2, title="達人プログラマー", author="Andrew Hunt"),
            3: Book(id=3, title="リファクタリング", author="Martin Fowler", status=BookStatus.CHECKED_OUT),
        }

    def find_by_id(self, book_id: int) -> Optional[Book]:
        """[インターフェースの実装] 'find_by_id' を辞書検索で実現"""
        return self._database.get(book_id)

    def save(self, book: Book) -> Book:
        """[インターフェースの実装] 'save' を辞書への上書き保存で実現"""
        self._database[book.id] = book
        return book


class InMemoryMemberRepository(MemberRepository):
    """MemberRepository の「インメモリDB」版実装"""
    def __init__(self):
        self._database: Dict[int, Member] = {
            1: Member(id=1, name="Alice"),
            2: Member(id=2, name="Bob"),
        }

    def find_by_id(self, member_id: int) -> Optional[Member]:
        """[インターフェースの実装] 'find_by_id' を辞書検索で実現"""
        return self._database.get(member_id)


class InMemoryLoanRepository(LoanRepository):
    """LoanRepository の「インメモリDB」版実装"""
    def __init__(self):
        self._database: Dict[int, Loan] = {}
        self._next_id: int = 1  # IDを自動採番するためのカウンター

    def save(self, loan: Loan) -> Loan:
        """
        [インターフェースの実装] 'save' を辞書への新規保存で実現
        - Loan.id が未設定の場合は、自動採番してから保存する
        """
        if loan.id is None:
            loan.id = self._next_id
            self._next_id += 1

        self._database[loan.id] = loan
        return loan

    def find_overdue(self, today: date) -> list[Loan]:
        """
        追加例（後で拡張する場合のイメージ）:
        期限切れになっている貸出を一覧で取得する。
        ※ こういったクエリ系の処理も Repository の責務になりうる。
        """
        return [
            record
            for record in self._database.values()
            if record.is_overdue(today)
        ]
```

### 補足

* 3つのクラス (`InMemoryBookRepository`, `InMemoryMemberRepository`, `InMemoryLoanRepository`) を分けているのは、**単一責任の原則**のためです。
  “本のことは本の倉庫（BookRepository）に聞く”という形にすることで、変更の影響範囲をピンポイントにできます。

* `InMemoryLoanRepository` は、`Loan` が新規に作られる特性上、`id` を自動採番する責務も持っています。これは「技術的な都合」でありビジネスルールではないので、Use Case ではなく Repository 側にあります。

---

## 💡 ユニットテストでData Accessの正しさを証明する

Data Access のテストでは、**永続化ロジックが契約どおりに動くか**だけを見ればOKです。
UIもUse Caseも関係ありません。

```python
# tests/interface_adapters/data_access/test_in_memory_repositories.py

from core.domain.book import Book
from interface_adapters.data_access.in_memory_repositories import InMemoryBookRepository


def test_InMemoryBookRepositoryは本の保存と検索ができる():
    # 1. Arrange (準備)
    repo = InMemoryBookRepository()
    new_book = Book(id=99, title="テスト駆動開発", author="ケント・ベック")

    # 2. Act (実行)
    repo.save(new_book)
    found_book = repo.find_by_id(99)

    # 3. Assert (検証)
    # 意図: 「saveしたものが、ちゃんとfind_by_idで見つかるか？」をテスト
    assert found_book is not None
    assert found_book.title == "テスト駆動開発"
```

* このテストは「このリポジトリ実装が約束通りに動いているか？」だけを確認しています。
* Use Case のテストは別で行います。責務を分離しているので、テストも分離できます。

---

## 🐍 PythonとC言語の比較（初心者の方へ）

* **Python (オブジェクト指向)**

  * `BookRepository` のようなインターフェース（抽象）をまず宣言します。
  * その契約を満たす具体クラス（`InMemoryBookRepository`や、将来的には`PostgresBookRepository`など）を差し替えるだけで、システム全体を壊さずにストレージ技術を変えられます。

* **C言語 (手続き型)**

  * `save_book_to_file(book)` や `find_book_from_file(id)` のような関数が直接ファイルI/Oを呼びにいく形になりがちです。
  * この場合、保存先を「ファイル→データベース」に変えるとき、呼び出し元のコードをすべて書き換える必要が出ます。依存がベタ結合になりやすいのが難点です。

---

## 🛡️ このファイル群の鉄則

この層のクラスはすべて、同じ掟に従います。

> データを変換し、黙って実行せよ。 (Translate and execute, but do not think.)

* Data Access（= Repository実装）は、ビジネスルールについて**考えません**。
  「Use Case から言われたとおりに取り出し／保存するだけ」です。
* どのDB技術を使うか（インメモリ / SQLite / PostgreSQL / クラウドAPI...）は、**この層だけで閉じます**。
  つまり、将来データベースを差し替えることになっても、修正範囲は `interface_adapters/data_access/` 以下だけで済みます。

---

## ❓ Q&A

### **Q. `DataAccess` と `Gateway` は何が違うのですか？図によって名前が違います。**

とても良い質問です。これは「役割の名前」と「具体的な実装」の違いです。

* **`Gateway`（ゲートウェイ）**

  * 分類: アーキテクチャ上の**役割 / パターン** 🚪
  * 意味: Use Case と「外の世界」（DB・外部API・ファイルなど）をつなぐ門のこと。
    同心円図では、Use Case の外側にある “外界とのインターフェース” として描かれます。
  * インターフェース（例: `BookRepository`）はこの Gateway の契約書です。

* **`Data Access`**

  * 分類: 上の役割を満たす**具体的なコードの実装** 🛠
  * 意味: その契約書どおりに、実際にデータを保存・検索する担当。
    今回の `InMemoryBookRepository` などがこれにあたります。

**まとめると、「Gateway という役割を、Data Access が実装している」と考えてください。**
図ごとに呼び名が違って見えても、実は同じ位置の話をしています。
