# 02 Repository

# 🗄 Repository（DataAccessInterface）
### `core/domain/repository.py`


## 🧭 このページの目的

この章では、**ドメイン層に置く Repository（リポジトリ）インターフェース**を定義します。
Repository は、これまで `<I> DataAccessInterface` と呼んでいたものと同じ役割です。

* `Book` をどう保存するのか
* `Member` をどう取得するのか
* 新しい `Loan`（貸出記録）をどう記録するのか

…といった「永続化・検索の手段」を、**ユースケースから隠蔽するための窓口**（＝抽象）です。

ここでは抽象クラスだけ定義します。
実際に「どのDBに保存するか」「どのようにIDを採番するか」は、この章では決めません。
それはインフラ層（`infrastructure/`）に後で生やします。

---

## 🌍 なぜ Repository は `core/domain/` に置くのか？

「データを保存・読み込めること」はビジネスとして必要な能力そのものだからです。

* 図書館という業務の観点では

  * 本は検索できなければ困る
  * 会員情報は確認できなければ困る
  * 貸出記録（誰がどの本を借りたか）は残っていなければ困る
* これは UI の都合ではなくて、ビジネスそのものの要件です。

だからこそ、その「必要条件」は**ドメイン側から宣言**されるべきです。
それが Repository です。

一方で、「どうやって保存するか？」は純粋に技術的な詳細です。

* メモリ上に保存するのか？
* SQLiteやPostgreSQLに保存するのか？
* 外部APIにPOSTするのか？

こういった具体手段はドメインの関心ではないので、外側（`infrastructure/`）に閉じ込めます。

> まとめると：
>
> * **何が必要か**（＝Repositoryインターフェース）は内側（ドメイン）で決める
> * **どうやるか**（＝具体実装）は外側（インフラ）で決める
>
> これが依存性逆転（Dependency Inversion）です。

---

## 📂 フォルダ構成（この章に関係する部分）

まずは、ドメイン層とインフラ層の関係がわかるように、今回必要になるディレクトリだけを抜粋します。

```text
clean_architecture_library/
├─ core/
│   └─ domain/
│       ├─ book.py              # Entity: 本
│       ├─ member.py            # Entity: 会員
│       ├─ loan.py              # Entity: 貸出記録
│       ├─ repository.py        # ← いま定義するRepositoryインターフェース
│       └─ errors.py            # ドメイン固有の例外など（必要なら）
│
├─ core/
│   └─ usecase/
│       ├─ interactor/
│       │   └─ checkout_book.py # "本を貸し出す"ユースケース（後の章）
│       └─ boundary/
│           ├─ dto.py           # DTO（InputData/OutputData/ViewModel）
│           ├─ input_boundary.py
│           └─ output_boundary.py
│
└─ infrastructure/
    └─ repositories/
        ├─ in_memory_book_repository.py     # 後で用意する具体実装（メモリ版）
        ├─ in_memory_member_repository.py
        └─ in_memory_loan_repository.py
        # さらに将来は postgres_book_repository.py などが増える
```

ポイント：

* `core/domain/repository.py` は **抽象（インターフェース）だけ**。
* `infrastructure/repositories/...` は **具体実装**。
* UseCase は抽象だけ知っており、具体実装は知らない。

---

## 💻 ソースコード（`core/domain/repository.py`）

```python
# --------------------------------------------------------------------
# File: core/domain/repository.py
# Layer: Domain
#
# 目的:
#   - ドメインオブジェクト(Book / Member / Loan)を永続化・取得するための
#     抽象インターフェース（Repository）を定義する。
#
# このファイルでやること:
#   - 具体的な保存方法は一切書かない。
#   - 「どんなメソッドを提供すべきか」だけを約束する。
#
# 同心円図での位置:
#   - Domain（最も内側）
#   - Infrastructure（最も外側）がこの抽象を"実装"する。
#
# C言語に例えるなら:
#   - ここは .h の宣言。
#   - 実際の .c（中身）は infrastructure 側に置くイメージ。
# --------------------------------------------------------------------

from __future__ import annotations
from abc import ABC, abstractmethod
from typing import Optional, List

from core.domain.book import Book
from core.domain.member import Member
from core.domain.loan import Loan


class BookRepository(ABC):
    """
    本に関する永続化の窓口（抽象）。
    UseCase層はこのインターフェースにだけ依存し、
    DBや保存形式の詳細は知らないままでよい。
    """

    @abstractmethod
    def find_by_id(self, book_id: int) -> Optional[Book]:
        """
        IDで本を検索して返す。
        見つからなければ None を返す。
        """
        raise NotImplementedError

    @abstractmethod
    def save(self, book: Book) -> Book:
        """
        本の状態を保存する。

        例:
        - 新規の本ならIDを採番して付与する
        - 既存の本なら状態(statusなど)を更新する

        採番やストレージ構造といった詳細は、
        具体実装側（infrastructure側）の責務。
        """
        raise NotImplementedError


class MemberRepository(ABC):
    """
    会員に関する永続化の窓口（抽象）。
    """

    @abstractmethod
    def find_by_id(self, member_id: int) -> Optional[Member]:
        """
        IDで会員を検索する。
        見つからなければ None を返す。
        """
        raise NotImplementedError

    @abstractmethod
    def save(self, member: Member) -> Member:
        """
        会員情報を保存する。
        将来、会員ステータスや会員資格の更新などを扱う可能性がある。
        """
        raise NotImplementedError


class LoanRepository(ABC):
    """
    貸出記録(Loan)に関する永続化の窓口（抽象）。
    """

    @abstractmethod
    def save(self, loan: Loan) -> Loan:
        """
        新しい貸出記録を保存し、必要であればIDを割り当てて返す。
        """
        raise NotImplementedError

    @abstractmethod
    def find_all_by_member(self, member_id: int) -> List[Loan]:
        """
        特定の会員が現在借りている(あるいは過去に借りた)Loan一覧を返す。

        例:
        - 延滞チェックのユースケースで使える
        - 「この会員はいま何冊借りているか？」といったビジネスルール検証にも使える
        """
        raise NotImplementedError
```

---

## 🔌 Repositoryをどう使うのか（ユースケース視点）

このあと定義するユースケース（例：`checkout_book.py`）は、こういう感じでリポジトリを受け取ります。

* `CheckOutBookInteractor` は

  * `BookRepository`
  * `MemberRepository`
  * `LoanRepository`
    に依存します。

でもそれはすべて「抽象（インターフェース）として」の依存です。
だからユースケースは「本がどこに保存されるのか」を気にしなくていい。

> ユースケースはこう言います：
> 「本と会員と貸出記録を、探したり保存したりできる"誰か"が必要です。
> それはあなた（インフラ層）で実装してください。」

---

## 🧪 Repositoryインターフェース自体はテストするべき？

原則として、ここ（`core/domain/repository.py`）にあるのは**抽象定義だけ**なので、
ここ自体にユニットテストはほぼ不要です。

代わりにテストするべきは：

* ユースケース（Interactor）がこれらのRepositoryを正しく呼ぶか？
  → UseCaseの単体テスト（フェイクRepositoryを注入して検証する）

* Repositoryの「実装」（例：`InMemoryBookRepository`）が期待どおり動くか？
  → infrastructure層のテスト

これはちょうど、「.h の宣言そのものはテストしないけど、その宣言を使うロジックと、その宣言を実装した .c はテストする」というC言語的な発想と同じです。

---

## 🐍 PythonとC言語の比較（初心者向けのおさらい）

* Python：

  * `class BookRepository(ABC): ...` のように抽象クラスを定義できる
  * これは「このメソッドは必ず実装してください」という**契約 (contract)** をコードで表現するもの
* C言語：

  * インターフェースという言語機能はない
  * かわりに、`.h` ファイルで関数のプロトタイプを宣言し、`.c` 側でその関数を実装する
  * 呼び出し側は「その関数がある」と信じてヘッダをインクルードできる
    → これとほぼ同じ役割を、Pythonでは抽象クラスでやっているイメージです

---

## 🛡 この章の鉄則

> 具象に依存するな。抽象に依存せよ。
> Depend on abstractions, not on concretions.

* ユースケースは `InMemoryBookRepository` や `PostgresBookRepository` といった**具体クラス**を知らない
* ユースケースが知っていいのは `BookRepository` / `MemberRepository` / `LoanRepository` という**抽象インターフェースだけ**
* だから保存先がメモリでも、SQLiteでも、PostgreSQLでも、REST APIでも、
  **ビジネスロジック（ユースケース）は一切変えなくていい**

