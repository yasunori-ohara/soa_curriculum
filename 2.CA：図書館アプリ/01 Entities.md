# 01 Entities

# 🧩 Entities
### `core/domain/*.py`

システムの心臓部である **Entities** から始めます。
このシステムには `Book`（本）、`Member`（会員）、そして両者を結びつける `Loan`（貸出記録）という3つの主要なEntityが存在します。


## このレイヤーの役割

ここで定義するクラスは、アーキテクチャの最も内側に位置する **Entities（ドメイン層 / core/domain）** です。
アプリケーション全体の **核となるビジネスルール** とそれに必要なデータを内包します。

これらは「図書館アプリ」という特定のUIやDBのためではなく、もっと抽象的な「図書館というビジネスそのもの」にとって普遍的なルールを表現します。

つまりここは：

> **このビジネスにおける『本』『会員』『貸出』とは何か、そしてそれぞれが従うべきルールは何か**
> を定義する場所です。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## ファイル配置

Entitiesは1ファイルにまとめず、役割ごとに分けて `core/domain/` に置きます。
この構成は後続の UseCase / Repository / Presenter と独立しており、他の層を import しません。

```text
clean_architecture_library/
├─ core/
│   ├─ domain/
│   │   ├─ book.py        # Book, BookStatus
│   │   ├─ member.py      # Member
│   │   ├─ loan.py        # Loan
│   │   ├─ repository.py  # Repositoryインターフェース群（次章で扱う）
│   │   └─ errors.py      # ドメイン固有の例外（必要なら）
│   └─ usecase/
│       └─ ...
└─ ...
```

以下では、それぞれのEntityの役割とコード例を順に見ていきます。

---

## Entity 1：`Book` (`core/domain/book.py`)

### このクラスにあるべき処理

`Book` は「本」そのものを表すEntityです。
ここに含めるのは、**アプリケーション固有ではない、本という概念に本質的なルール**だけです。

⭕️ 含めるべき処理の例

* **状態遷移ルール**

  * `check_out()` で「貸出可能 → 貸出中」への状態変更
  * `return_book()` で「貸出中 → 貸出可能」への状態変更
  * 状態が不正な場合はエラーを発生させる（例：「すでに貸出中なのに、さらに貸し出そうとした」）
* **不変条件の維持**

  * タイトルは空ではいけない、といった自己バリデーション

❌ 含めてはいけない処理の例

* データベースへの保存（`save_to_database()` のようなメソッドは禁止）
* 画面表示（`display_on_screen()`は禁止）
* 誰が借りたかなどの外部管理ロジック（それは `Loan` と UseCaseの責務）

### コード例

```python
# core/domain/book.py

from enum import Enum

class BookStatus(Enum):
    """本の状態を定義する列挙型。文字列よりも安全で明確。"""
    AVAILABLE = "貸出可能"
    CHECKED_OUT = "貸出中"

class Book:
    """
    Bookエンティティ。
    図書館ビジネスにおける「本」という概念と、そのルールを表す。
    """
    def __init__(self, id: int, title: str, author: str, status: BookStatus = BookStatus.AVAILABLE):
        # 不変条件（Invariant）のチェック
        if not title:
            raise ValueError("本のタイトルは必須です。")

        self.id = id
        self.title = title
        self.author = author
        self.status = status

    def check_out(self):
        """
        本を貸し出すときの状態遷移ルール。
        - すでに貸出中なら貸し出せない。
        """
        if self.status != BookStatus.AVAILABLE:
            raise ValueError(f"この本「{self.title}」は現在貸出中です。")
        self.status = BookStatus.CHECKED_OUT

    def return_book(self):
        """
        本を返却するときの状態遷移ルール。
        - 貸し出されていない本を返却しようとするとエラー。
        """
        if self.status != BookStatus.CHECKED_OUT:
            raise ValueError(f"この本「{self.title}」は貸し出されていません。")
        self.status = BookStatus.AVAILABLE

    def __repr__(self):
        # 開発者向けのデバッグ表示
        return f"Book(id={self.id}, title='{self.title}', status={self.status.value})"
```

ポイント：

* `Book` 自身が自分の状態を守る
* 「貸出中の本はさらに貸せない」といった禁止ルールが `Book` クラスに閉じている
* DBは一切知らない

---

## Entity 2：`Member` (`core/domain/member.py`)

`Member` は会員を表すEntityです。
ここでも「会員とは何か」「どの状態が有効か」といった普遍的なルールのみを扱います。

```python
# core/domain/member.py

class Member:
    """
    Memberエンティティ。
    図書館ビジネスにおける「会員」という概念と、その基本的なルール。
    """
    def __init__(self, id: int, name: str, active: bool = True):
        if not name:
            raise ValueError("会員名は必須です。")

        self.id = id
        self.name = name
        self.active = active  # 例: 無効会員は貸出不可、などの将来拡張に使える

    def is_active(self) -> bool:
        """
        会員として有効かどうかを返す。
        ここでは単純なフラグだが、将来的に「会費未払い→停止」などの
        複雑な条件が入っても、このメソッドの中に閉じ込められる。
        """
        return self.active

    def __repr__(self):
        return f"Member(id={self.id}, name='{self.name}', active={self.active})"
```

ポイント：

* `Member` が「有効会員かどうか？」の判断ロジックを持つ
* まだルールが単純でも、将来の拡張（利用停止、会費未納など）を受け止める居場所がここ

---

## Entity 3：`Loan` (`core/domain/loan.py`)

`Loan` は **「誰が、どの本を、いつ借りて、いつまでに返すのか」** を表す貸出記録そのものです。
業務の中核となるEntityです。

```python
# core/domain/loan.py

from datetime import date, timedelta

class Loan:
    """
    Loanエンティティ。
    ある会員が、ある本を、いつ借りて、いつまでに返すべきかを表現する。
    """
    LOAN_PERIOD_DAYS = 14  # 「貸出は2週間まで」などビジネス上の基本ルール

    def __init__(
        self,
        id: int,
        book_id: int,
        member_id: int,
        loan_date: date,
        due_date: date | None = None,
    ):
        self.id = id
        self.book_id = book_id
        self.member_id = member_id
        self.loan_date = loan_date
        # 返却期限は渡されなければ自動計算
        self.due_date = due_date or (loan_date + timedelta(days=self.LOAN_PERIOD_DAYS))

    def is_overdue(self, current_date: date) -> bool:
        """
        延滞しているかどうかを判定する純粋なビジネスロジック。
        """
        return current_date > self.due_date

    def __repr__(self):
        return (
            f"Loan(id={self.id}, book_id={self.book_id}, member_id={self.member_id}, "
            f"loan_date='{self.loan_date}', due_date='{self.due_date}')"
        )
```

ポイント：

* 「貸出期間は14日」などの業務ルールは `Loan` に集約される
* 「延滞かどうか」は `Loan.is_overdue()` が判断する
* `Loan` は `Book` や `Member` の中に書かれません
  → 責務が分離されているので保守しやすい

---

## 💡 ユニットテストでEntityの正しさを証明する

Entitiesは**何にも依存しない（DBもUIもimportしない）**ため、pytestなどで単体テストしやすいのが強みです。

以下は例として `Book` のテストです。

```python
# tests/unit/test_domain_book.py

import pytest
from core.domain.book import Book, BookStatus

def test_貸出可能な本は貸し出せる():
    # Arrange
    book = Book(id=1, title="クリーンアーキテクチャ", author="ボブおじさん")

    # Act
    book.check_out()

    # Assert
    assert book.status == BookStatus.CHECKED_OUT

def test_貸出中の本はさらに貸し出せない():
    # Arrange
    book = Book(
        id=1,
        title="テスト駆動開発",
        author="ケント・ベック",
        status=BookStatus.CHECKED_OUT,
    )

    # Act & Assert
    with pytest.raises(ValueError, match="現在貸出中"):
        book.check_out()
```

ここにDB接続処理・フレームワーク依存・HTTPリクエストなどの「外側の都合」は一切登場しません。
**これが内側（core/domain）の強さです。**

---

## 🐍 PythonとC言語の比較（初心者向け）

* Python（オブジェクト指向）

  * データとそれを操作するルールを同じクラスにまとめる
  * 例: `book.check_out()`
  * 「本」というオブジェクト自身が、自分の正しい状態遷移を保証する

* C言語（手続き型）

  * `struct Book { ... };` のようにデータだけを定義し
  * `check_out_book(struct Book* b)` のような外部関数で状態変更する
  * データとロジックは分離する

* クリーンアーキテクチャ

  * Pythonの前者のスタイルを最大限に活かして、
    **Entityを「自律する小さなルールの塊」として中心に置く**ことで、
    UIやDBが変わってもビジネスロジックが壊れないようにする。

---

## 🛡️ Entities層の鉄則

> 何物にも依存するな (Depend on Nothing)

* UIやDBといった外側の詳細を一切 import しない
* 最も安定しているべきビジネスルールを、最も内側に閉じ込める
* 他の層はこの層を呼び出すが、この層は誰も呼びにいかない（一方通行）

この鉄則を守ることで、ビジネスの中心となるルールが
**技術的な変更から完全に隔離され、変化に強く、テストしやすい**状態になります。
