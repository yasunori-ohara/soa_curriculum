# 01 Entities

# 🧩 Entities : `domain/entities.py`

システムの心臓部である **Entities** から始めます。このシステムには`Book`（本）、`Member`（会員）、そして両者を結びつける`Loan`（貸出）という3つの主要なEntityが存在します。

---

## このファイル（クラス群）の役割

このファイルに定義されるクラス群は、アーキテクチャの最も内側に位置する**Entities**です。アプリケーション全体の**核となるビジネスルール**とデータをカプセル化（内包）します。

これらは特定のアプリケーションのためではなく、このシステムのビジネスドメイン、つまり「図書館の貸出管理」そのものにとって**普遍的なルールとデータ**を表現します。

平たく言えば、「**このビジネスにおける『本』『会員』『貸出』とは何か、そして彼らが従うべき『ルール』は何か**」を定義する、最も重要な場所です。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## Entity 1：`Book`

### このクラスにあるべき処理

`Book` Entityに含めるべきなのは、**アプリケーションの都合に依存しない、純粋な「本」に関するルール**です。

⭕️ **含めるべき処理の例:**

- **状態遷移のロジック**:
    - `check_out()`メソッド。本の状態を「貸出可能」から「貸出中」へ変更します。このとき、「貸出中の本は貸し出せない」というルールを内部に持ちます。
    - `return_book()`メソッド。本の状態を「貸出中」から「貸出可能」へ戻します。
- **不変条件の維持（自己バリデーション）**:
    - 本のタイトルは空文字列であってはならない、といったチェック。

❌ **含めてはいけない処理の例:**

- **データベースへの保存処理**: `save_to_database()`のようなメソッドは**絶対に含めません**。Entityは自分がどう保存されるかを知るべきではありません。
- **UIに関する処理**: `display_on_screen()`のような画面表示ロジックは含めません。
- **他のEntityに関する知識**: `Book` Entityは、どの`Member`が自分を借りているか、といった情報を直接知るべきではありません（それは**Use Case**と\*\*`Loan` Entity\*\*が管理します）。

### ソースコードの詳細解説

```python
# domain/entities.py

from enum import Enum
from datetime import date, timedelta

# -----------------------------------------------------------------------------
# Book Entity
# - クラス図の位置: Entities
# - 同心円図の位置: Entities (最も内側)
# -----------------------------------------------------------------------------
class BookStatus(Enum):
    """本の状態を定義する列挙型。文字列よりも安全で明確。"""
    AVAILABLE = "貸出可能"
    CHECKED_OUT = "貸出中"

class Book:
    """
    Bookエンティティ。
    このシステムのビジネスにおける「本」の概念とルールを定義します。
    """
    def __init__(self, id: int, title: str, author: str, status: BookStatus = BookStatus.AVAILABLE):
        """
        コンストラクタ: Bookオブジェクトを初期化します。
        [データ] 本が本質的に持つべきデータ（属性）を定義します。
        """
        # 不変条件のチェック（自己バリデーション）
        if not title:
            raise ValueError("本のタイトルは必須です。")

        self.id = id
        self.title = title
        self.author = author
        self.status = status

    def check_out(self):
        """
        [ビジネスルール] 本を貸し出す際の、本自身の状態遷移ルール。
        """
        if self.status != BookStatus.AVAILABLE:
            raise ValueError(f"この本「{self.title}」は現在貸出中です。")
        self.status = BookStatus.CHECKED_OUT

    def return_book(self):
        """
        [ビジネスルール] 本を返却する際の、本自身の状態遷移ルール。
        """
        if self.status != BookStatus.CHECKED_OUT:
            raise ValueError(f"この本「{self.title}」は貸し出されていません。")
        self.status = BookStatus.AVAILABLE

    def __repr__(self):
        # オブジェクトを分かりやすく表示するための特殊メソッド
        return f"Book(id={self.id}, title='{self.title}', status={self.status.value})"

```

- `BookStatus(Enum)`: `Enum`（列挙型）を使い、本の状態を「貸出可能」「貸出中」という2つの明確な選択肢に限定しています。これにより、`"貸出中"`と`"貸し出し中"`のようなタイプミスによるバグを防ぎます。
- `check_out()`メソッド：このメソッドが、このEntityの核となるビジネスルールです。まず、現在の状態が「貸出可能」であるかをチェックし、そうでなければエラー（`ValueError`）を発生させます。ルールを満たしている場合のみ、自身の状態を「貸出中」に変更します。**`Book`自身が自分の状態を管理する責任**を持っています。

---

## Entity 2 : `Member`

`Member` Entityも同様に、**純粋な「会員」に関するルール**のみを持ちます。

```python
# domain/entities.py (続き)

# -----------------------------------------------------------------------------
# Member Entity
# - クラス図の位置: Entities
# - 同心円図の位置: Entities (最も内側)
# -----------------------------------------------------------------------------
class Member:
    """
    Memberエンティティ。
    このシステムのビジネスにおける「会員」の概念とルールを定義します。
    """
    def __init__(self, id: int, name: str):
        """
        コンストラクタ: Memberオブジェクトを初期化します。
        [データ] 会員が本質的に持つべきデータ（属性）を定義します。
        """
        if not name:
            raise ValueError("会員名は必須です。")

        self.id = id
        self.name = name

    def __repr__(self):
        return f"Member(id={self.id}, name='{self.name}')"

```

- この`Member` Entityは、現在のシンプルな要件では、主にデータを保持する役割を担います。`Book`のように複雑な状態遷移はありませんが、`name`が必須であるというバリデーションルールは持っています。
- 今後「会員資格の有効期限」といった概念が追加されれば、`is_active()`のようなビジネスルールを持つメソッドがここに追加されるでしょう。

---

## Entity 3 : `Loan` (貸出)

`Loan`は\*\*「誰が、どの本を、いつ借りて、いつまでに返すか」\*\*という貸出記録そのものを表す、**ビジネスの核心となるEntity**です。`Book`と`Member`の関係性を定義します。

```python
# domain/entities.py (続き)

# -----------------------------------------------------------------------------
# Loan Entity
# - クラス図の位置: Entities
# - 同心円図の位置: Entities (最も内側)
# -----------------------------------------------------------------------------
class Loan:
    """
    Loanエンティティ。
    貸出というビジネスイベントそのものを表し、関連するルールを定義します。
    """
    LOAN_PERIOD_DAYS = 14  # 貸出期間を14日とするビジネスルール

    def __init__(self, id: int, book_id: int, member_id: int, loan_date: date, due_date: date = None):
        """
        コンストラクタ: Loanオブジェクトを初期化します。
        """
        self.id = id
        self.book_id = book_id
        self.member_id = member_id
        self.loan_date = loan_date
        # due_dateが指定されなければ、貸出日から自動計算する
        self.due_date = due_date or self.loan_date + timedelta(days=self.LOAN_PERIOD_DAYS)

    def is_overdue(self, current_date: date) -> bool:
        """
        [ビジネスルール] この貸出が現在延滞しているかどうかを判断する。
        """
        return current_date > self.due_date

    def __repr__(self):
        return (f"Loan(id={self.id}, book_id={self.id}, member_id={self.member_id}, "
                f"due_date='{self.due_date}')")

```

- `LOAN_PERIOD_DAYS = 14`: 「貸出期間は2週間」という**普遍的なビジネスルール**を定数として定義しています。このルールが変わった場合、変更箇所はこの一箇所だけで済みます。
- `__init__`内の`due_date`の計算：返却期限日(`due_date`)が与えられなければ、貸出日(`loan_date`)を基に**自動で計算**します。これも`Loan`が持つべき重要なビジネスロジックです。
- `is_overdue()`: 「延滞しているか？」を判断するロジックです。これも純粋な`Loan`に関するビジネスルールです。

---

## 💡 ユニットテストでEntityの正しさを証明する

Entityは**何にも依存しない**ため、非常に簡単に単体でテストできます。これはクリーンアーキテクチャの大きな利点です。

```python
# tests/domain/test_entities.py の例

import pytest
from domain.entities import Book, BookStatus

def test_貸出可能な本は貸し出せる():
    # 1. Arrange (準備): テスト対象のオブジェクトを作成
    book = Book(id=1, title="クリーンアーキテクチャ", author="ボブおじさん")

    # 2. Act (実行): メソッドを呼び出して状態を変化させる
    book.check_out()

    # 3. Assert (検証): 期待通りの状態になっているか確認
    assert book.status == BookStatus.CHECKED_OUT

def test_貸出中の本は貸し出せない():
    # 1. Arrange (準備): 最初から「貸出中」の本を用意
    book = Book(id=1, title="テスト駆動開発", author="ケント・ベック", status=BookStatus.CHECKED_OUT)

    # 2. Act & Assert (実行と検証): 特定のエラーが発生することを期待する
    # 意図: 「貸出中の本を再度貸し出そうとしたら、ちゃんとエラーになるか？」をテスト
    with pytest.raises(ValueError, match="この本「テスト駆動開発」は現在貸出中です。"):
        book.check_out()

```

- **テストの意図**: これらのテストは、DBやUIが一切ない状態で、`Book` Entityが持つ**ビジネスルール（状態遷移）だけが正しく動作するか**を検証しています。これが「責務に集中する」ということです。

---

## 🐍 PythonとC言語の比較（初心者の方へ）

- **Python (オブジェクト指向)**: このカリキュラムのように、データ（`title`や`status`）と、そのデータを操作するルール（`check_out()`メソッド）を`Book`という**一つのクラス（モノ）にまとめます**。`book.check_out()`と書くことで、`book`オブジェクト自身がルールに従って振る舞います。
- **C言語 (手続き型)**: もしC言語で書くなら、本のデータは`struct BookData`のような構造体で定義し、貸出処理は`check_out_book(struct BookData* book)`のような**外部の関数**として実装するでしょう。データと処理が分離しています。
- **クリーンアーキテクチャ**: Pythonのオブジェクト指向を活かし、Entityを\*\*「自律的なデータとロジックの塊」\*\*として設計することで、関心事を綺麗に分離し、変更に強い構造を作ります。

---

## 🛡️ このクラス群の鉄則

Entity層を守るべき最も重要なルールは、TODOアプリの時と全く同じです。

> 何物にも依存するな (Depend on Nothing)
> 
- **最も安定した、変更の少ないコードであるべきです。** 図書館の基本的なルール（「貸出中の本は貸し出せない」など）は、システムのUIやDB技術が変わっても、簡単には変わりません。
- **外側のレイヤー（Use Cases, Adaptersなど）について一切知ってはいけません。** `import`文で外側のレイヤーのファイルを読み込むことは固く禁じられています。
- Entityは、アプリケーションの他の部分から**利用される**だけの存在です。

この鉄則を守ることで、システムの基盤であるビジネスルールが技術的な変更から完全に隔離され、**変化に強く、テストしやすいソフトウェア**を構築することができるのです。