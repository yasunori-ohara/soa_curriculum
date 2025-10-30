# 第1章：Entity（エンティティ）【第2巡：図書貸出】

## 🎯 この章の目的

この章では、クリーンアーキテクチャの最も内側に位置する「**Entity（エンティティ）**」を、図書貸出ドメイン（Library Lending）で実装します。

Entityはアプリケーションの**ビジネスルールそのもの**を表す層です。
外部の技術（Webフレームワーク、データベース、UI）に一切依存せず、
システムの**核となる意味と不変条件（不変条件・整合性）**を定義します。

---

## 🧩 Entityとは

クリーンアーキテクチャの同心円で言うと、最も中心の円に位置します。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

この層は「ビジネスルール（Business Rules）」です。
**技術的な詳細を排除した純粋なロジック**だけを持ちます。

第2巡の題材「図書貸出」における主な概念は次の3つです。

* **Book** … 書籍（タイトル、著者、ISBN など）
* **Member** … 会員（名前、メール）
* **Loan** … 貸出履歴（どの本を誰がいつ借りて、いつまでに返すか）

> 重要：
> 「借りられている最中に同じ本をもう一度借りることはできない」等の**アプリケーションの振る舞い**は UseCase で扱います。
> 一方、**“空のタイトルはありえない”** 等の**概念の不変条件**は Entity が責務を持ちます。

---

## 🧱 Entity層のファイル構成

```
project_root/
└── domain/
    ├── book.py    ← Bookエンティティ
    ├── member.py  ← Memberエンティティ
    └── loan.py    ← Loanエンティティ
```

---

## 🧭 設計方針（第1巡と同じ基本原則）

* **何物にも依存しない（Depend on Nothing）**
  import は標準ライブラリのみ。DB、FastAPI、SQLは登場しません。
* **概念の不変条件（Invariant）を担保**
  例：空文字を許さない、メール書式の最低限チェック など。
* **外部の技術が変わっても、この層は変わらない**
  Web/DB/認証の都合は一切持ち込まない。

---

## 🧠 例題：図書貸出の Entity

### ✳️ `domain/book.py`

```python
# --------------------------------------------------------------------
# Entity 層に相当（最も内側の円）
# 図書貸出ドメインの「書籍」の本質的なルールを表す。
# 外部技術（Web/DB）には一切依存しない。
# --------------------------------------------------------------------
from dataclasses import dataclass
from datetime import datetime
from typing import Optional


@dataclass
class Book:
    id: Optional[int]
    title: str
    author: str
    isbn: str
    created_at: datetime

    def __post_init__(self):
        if not self.title or not self.title.strip():
            raise ValueError("Book.title は必須です")
        if not self.author or not self.author.strip():
            raise ValueError("Book.author は必須です")
        if not self.isbn or not self.isbn.strip():
            raise ValueError("Book.isbn は必須です")

    def rename(self, new_title: str):
        """書籍タイトルの変更（概念的に許容する場合のみ使用）"""
        if not new_title or not new_title.strip():
            raise ValueError("Book.title は空にできません")
        self.title = new_title
```

### ✳️ `domain/member.py`

```python
# --------------------------------------------------------------------
# 図書貸出ドメインの「会員」の本質的なルールを表す Entity
# --------------------------------------------------------------------
from dataclasses import dataclass
from datetime import datetime
from typing import Optional
import re


EMAIL_RE = re.compile(r"^[^@\s]+@[^@\s]+\.[^@\s]+$")


@dataclass
class Member:
    id: Optional[int]
    name: str
    email: str
    created_at: datetime

    def __post_init__(self):
        if not self.name or not self.name.strip():
            raise ValueError("Member.name は必須です")
        if not self.email or not self.email.strip():
            raise ValueError("Member.email は必須です")
        # 簡易チェック（詳細な検証はInterface層で追加可能）
        if not EMAIL_RE.match(self.email):
            raise ValueError("Member.email が不正な形式です")

    def rename(self, new_name: str):
        if not new_name or not new_name.strip():
            raise ValueError("Member.name は空にできません")
        self.name = new_name
```

### ✳️ `domain/loan.py`

```python
# --------------------------------------------------------------------
# 図書貸出ドメインの「貸出」の本質的なルールを表す Entity
# --------------------------------------------------------------------
from dataclasses import dataclass
from datetime import datetime
from typing import Optional


@dataclass
class Loan:
    id: Optional[int]
    book_id: int
    member_id: int
    borrowed_at: datetime
    due_at: datetime
    returned_at: Optional[datetime]

    def __post_init__(self):
        if self.due_at <= self.borrowed_at:
            raise ValueError("Loan.due_at は borrowed_at より後である必要があります")

    # Entity は“概念に内在する判断”のみ提供。
    # 実際の「借りる」「返す」の振る舞い（検査や永続化を伴う処理）は UseCase の責務。
    def is_active(self) -> bool:
        """返却されていない貸出か？"""
        return self.returned_at is None

    def is_overdue(self, now: datetime) -> bool:
        """延滞か？（返却されておらず、かつ期限を過ぎている）"""
        return self.is_active() and (now > self.due_at)

    def mark_returned(self, when: datetime):
        """返却日時を記録（不変条件の更新はEntityで担保して良い）"""
        if self.returned_at is not None:
            raise ValueError("すでに返却済みです")
        if when < self.borrowed_at:
            raise ValueError("返却日時は貸出日時より前にはできません")
        self.returned_at = when
```

---

## ✅ 解説（どこまでを Entity に書くか）

* **概念の不変条件**（空文字禁止、時系列が前後しない 等）は **Entity** で担保します。
  → 「Book.title は必須」「Loan.due_at は borrowed_at より後」など。

* **アプリケーションの手続き**（“この会員は何冊まで借りられるか？”“同じ本は同時に重複貸出不可”など、複数 Entity/外部状態を横断して判断する処理）は **UseCase** で扱います。
  → 外部状態（既存の貸出有無、会員の制限）を参照するため **抽象リポジトリ**経由で行うのが筋。

* **データ保存やHTTP入出力**は **Entity の責務ではない**
  → DBやWebの都合を入れないことで、**最も安定**した層になります。

---

## 💬 Entity層で“やってはいけないこと”

| NG 例                           | なぜダメ？                                     |
| ------------------------------ | ----------------------------------------- |
| `save_to_db()` のような保存メソッド      | DBは外側の詳細。永続化は Infrastructure 層の責務。        |
| `from_request(req)` のようなHTTP依存 | Webは外側の詳細。入力整形は Controller/PRESENTER の責務。 |
| SQL文字列や ORM モデルを import        | 技術詳細が混ざると内側が外側に縛られてしまう。                   |
| 外部API呼び出し                      | 外界との通信は上位層で扱う。                            |

---

## 📘 この章の理解ポイント

* **Entity は最内層の“ルールの心臓”**。外側の都合（Web/DB）を一切持ち込まない。
* **Invariant（不変条件）**は Entity の責務。
  一方、**外部状態を参照する業務手続き**は UseCase の責務。
* ここを堅牢にすると、**外側が変わっても壊れなくなる**。

---

## 🔄 次のステップ：UseCase層へ

次は、**Entity を使ってアプリケーションの振る舞いを定義する**
**UseCase（ユースケース）層** を実装します。

この章で作った `Book` / `Member` / `Loan` を組み合わせ、

* 本を登録する
* 会員を登録する
* 本を借りる
* 本を返す
* 延滞を一覧する
* 検索する（本/会員）

といった操作を、**抽象リポジトリ**に依存して実装します。
依存の向きと制御の流れを、再度ここで確認しておきましょう。

![クリーンアーキテクチャ・依存と制御](../クリーンアーキテクチャ・依存と制御.png)


