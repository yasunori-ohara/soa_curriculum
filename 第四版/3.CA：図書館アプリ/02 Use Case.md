# 第2章：UseCase（ユースケース）


## 🎯 この章の目的

この章では、クリーンアーキテクチャの**2層目（内側から2番目）**である
**UseCase（ユースケース層）**を実装します。

ここは、アプリケーションが**「何をできるか」**を定義する層です。
Entity（ビジネスルール）を操作して「業務の流れ」を表現します。

---

## 🧩 UseCaseの位置づけ

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

| クラス図の箱                       | 役割                         | 層         |
| ---------------------------- | -------------------------- | --------- |
| **UseCaseInteractor**        | アプリ固有の手続きを実装する中心           | UseCases層 |
| **InputBoundary（I）**         | Controller → UseCase の入口契約 | UseCases層 |
| **OutputBoundary（I）**        | UseCase → Presenter の出口契約  | UseCases層 |
| **InputData／OutputData（DS）** | 境界を渡るデータ構造                 | UseCases層 |
| **Data Access Interface（I）** | Repositoryの契約（下層との橋）       | UseCases層 |
| **Entity**                   | ビジネスルールの本体                 | Entities層 |

> 依存の方向：外（Controller, Presenter, Gateway）→ 内（UseCase）→ 最内（Entity）

---

## 🧱 フォルダ構成（クラス図に対応）

```
project_root/
├── domain/
│   ├── book.py
│   └── member.py
└── usecase/
    ├── dto.py                        # InputData / OutputData（DS）
    ├── boundaries/                   # Controller／Presenterとの契約（Input/OutputBoundary）
    │   ├── register_book_boundary.py
    │   └── register_member_boundary.py
    ├── contracts/                    # Repository（Data Access Interface）
    │   ├── book_repository.py
    │   └── member_repository.py
    ├── register_book.py              # UseCaseInteractor本体（Book）
    └── register_member.py            # UseCaseInteractor本体（Member）
```

---

## 💡 設計方針

* UseCaseは**具体的な外部技術（WebやDB）を知らない**。
* Controller／Presenterとは **Boundary（境界インターフェース）** を通してやり取りする。
* Repository（データアクセス）は **契約（Interface）** のみを知る。
* InputData／OutputData は **境界を渡る純粋なデータ構造（DS）**。
* Entityは**ビジネスルールの本体**であり、UseCaseがそれを操作する。

---

## 🧠 例題：本と会員の登録

### ✳️ `usecase/dto.py`（InputData／OutputData：DS）

```python
# --------------------------------------------------------------------
# [クラス図] InputData / OutputData（DS）
# [同心円] UseCase層（外部形式から独立したデータ構造）
# --------------------------------------------------------------------
from dataclasses import dataclass
from datetime import datetime

# --- Book登録 ---
@dataclass
class RegisterBookInput:   # InputData
    title: str
    author: str
    isbn: str

@dataclass
class RegisterBookOutput:  # OutputData
    id: int
    title: str
    author: str
    isbn: str
    created_at: datetime


# --- Member登録 ---
@dataclass
class RegisterMemberInput:  # InputData
    name: str
    email: str

@dataclass
class RegisterMemberOutput:  # OutputData
    id: int
    name: str
    email: str
    created_at: datetime
```

---

### ✳️ `usecase/contracts/book_repository.py`（Data Access Interface）

```python
# --------------------------------------------------------------------
# [クラス図] Data Access Interface（Repository契約）
# [同心円] UseCase層
# --------------------------------------------------------------------
from abc import ABC, abstractmethod
from typing import List, Optional
from domain.book import Book

class BookRepository(ABC):
    """Bookデータの取得・保存の契約（UseCase→Gatewayに依存逆転）"""

    @abstractmethod
    def next_id(self) -> int:
        """次のIDを発行"""
        pass

    @abstractmethod
    def save(self, book: Book) -> Book:
        """Bookを保存"""
        pass

    @abstractmethod
    def find_by_id(self, book_id: int) -> Optional[Book]:
        """ID検索"""
        pass

    @abstractmethod
    def find_by_isbn(self, isbn: str) -> Optional[Book]:
        """ISBN検索"""
        pass

    @abstractmethod
    def search(self, keyword: str, limit: int = 50) -> List[Book]:
        """部分一致検索"""
        pass
```

---

### ✳️ `usecase/boundaries/register_book_boundary.py`（Input／OutputBoundary）

```python
# --------------------------------------------------------------------
# [クラス図] InputBoundary / OutputBoundary
# [同心円] UseCase層（Controller／Presenterとの契約）
# --------------------------------------------------------------------
from typing import Protocol
from usecase.dto import RegisterBookInput, RegisterBookOutput

class RegisterBookInputBoundary(Protocol):
    """Controller → UseCase の契約"""
    def execute(self, inp: RegisterBookInput) -> None: ...

class RegisterBookOutputBoundary(Protocol):
    """UseCase → Presenter の契約"""
    def present(self, out: RegisterBookOutput) -> None: ...
```

---

### ✳️ `usecase/register_book.py`（UseCaseInteractor）

```python
# --------------------------------------------------------------------
# [クラス図] UseCaseInteractor
# [同心円] UseCase層
# --------------------------------------------------------------------
from datetime import datetime
from typing import Optional
from domain.book import Book
from .dto import RegisterBookInput, RegisterBookOutput
from .contracts.book_repository import BookRepository
from .boundaries.register_book_boundary import (
    RegisterBookInputBoundary,
    RegisterBookOutputBoundary,
)

class RegisterBook(RegisterBookInputBoundary):
    """
    UseCaseInteractor
    - InputBoundaryを実装
    - Repository契約（Data Access Interface）に依存
    - OutputBoundaryに結果を通知
    """

    def __init__(self, books: BookRepository):
        self.books = books
        self._presenter: Optional[RegisterBookOutputBoundary] = None

    def set_output_boundary(self, presenter: RegisterBookOutputBoundary) -> None:
        """PresenterをDIで注入"""
        self._presenter = presenter

    def execute(self, inp: RegisterBookInput) -> None:
        if self.books.find_by_isbn(inp.isbn):
            raise ValueError("このISBNはすでに登録されています。")

        now = datetime.utcnow()
        new_id = self.books.next_id()
        book = Book(id=new_id, title=inp.title, author=inp.author, isbn=inp.isbn, created_at=now)
        saved = self.books.save(book)

        out = RegisterBookOutput(
            id=saved.id, title=saved.title, author=saved.author, isbn=saved.isbn, created_at=saved.created_at
        )
        assert self._presenter is not None, "Presenter（OutputBoundary）が設定されていません"
        self._presenter.present(out)
```

---

### ✳️ `usecase/boundaries/register_member_boundary.py`

```python
# --------------------------------------------------------------------
# [クラス図] InputBoundary / OutputBoundary（Member版）
# --------------------------------------------------------------------
from typing import Protocol
from usecase.dto import RegisterMemberInput, RegisterMemberOutput

class RegisterMemberInputBoundary(Protocol):
    def execute(self, inp: RegisterMemberInput) -> None: ...

class RegisterMemberOutputBoundary(Protocol):
    def present(self, out: RegisterMemberOutput) -> None: ...
```

---

### ✳️ `usecase/register_member.py`

```python
# --------------------------------------------------------------------
# [クラス図] UseCaseInteractor（Member版）
# --------------------------------------------------------------------
from datetime import datetime
from typing import Optional
from domain.member import Member
from .dto import RegisterMemberInput, RegisterMemberOutput
from .contracts.member_repository import MemberRepository
from .boundaries.register_member_boundary import (
    RegisterMemberInputBoundary,
    RegisterMemberOutputBoundary,
)

class RegisterMember(RegisterMemberInputBoundary):
    def __init__(self, members: MemberRepository):
        self.members = members
        self._presenter: Optional[RegisterMemberOutputBoundary] = None

    def set_output_boundary(self, presenter: RegisterMemberOutputBoundary) -> None:
        self._presenter = presenter

    def execute(self, inp: RegisterMemberInput) -> None:
        if self.members.find_by_email(inp.email):
            raise ValueError("このメールアドレスは既に登録されています。")

        now = datetime.utcnow()
        new_id = self.members.next_id()
        member = Member(id=new_id, name=inp.name, email=inp.email, created_at=now)
        saved = self.members.save(member)

        out = RegisterMemberOutput(
            id=saved.id, name=saved.name, email=saved.email, created_at=saved.created_at
        )
        assert self._presenter is not None, "Presenter（OutputBoundary）が設定されていません"
        self._presenter.present(out)
```

---

## 🧠 ここまでの依存関係

| 層                   | 依存方向                                 | 制御方向                 |
| ------------------- | ------------------------------------ | -------------------- |
| Controller          | → InputBoundary                      | ← 外から呼び出す            |
| UseCaseInteractor   | → Entity, Repository, OutputBoundary | ← Controllerから制御を受ける |
| Presenter           | → OutputData                         | ← UseCaseから制御を受ける    |
| Repository(Gateway) | → DB                                 | ← UseCaseから制御を受ける    |

→ **依存は内へ、制御は外へ**

---

## 💬 UseCase層で“やってはいけないこと”

| 禁止事項                  | 理由                        |
| --------------------- | ------------------------- |
| FastAPIやPydanticなどの使用 | フレームワークに依存するため            |
| SQL/ORMの記述            | Data Access Interfaceに委ねる |
| HTTPステータスの判断          | Presenterの責務              |
| ViewModelやテンプレート整形    | Presenterの責務              |

---

## 🧪 テストのヒント（この章の範囲）

**Interactor単体テスト例**

```python
class DummyBookRepository:
    def __init__(self): self.data = []
    def next_id(self): return len(self.data) + 1
    def find_by_isbn(self, isbn): return None
    def save(self, book): self.data.append(book); return book

class DummyPresenter:
    def __init__(self): self.result = None
    def present(self, out): self.result = out

repo = DummyBookRepository()
uc = RegisterBook(repo)
presenter = DummyPresenter()
uc.set_output_boundary(presenter)

uc.execute(RegisterBookInput("DDD", "Evans", "1234"))
assert presenter.result.title == "DDD"
```

---

## 📘 この章の理解ポイント

| 観点        | 内容                                    |
| --------- | ------------------------------------- |
| **役割**    | Entityを使ってアプリの動作を定義する                 |
| **依存関係**  | Entity・Repository・Boundaryにのみ依存       |
| **安定性**   | Entityに次いで安定。UIやDB変更に影響されない           |
| **制御の流れ** | 外（Controller）→内（UseCase）→外（Presenter） |


---
## 📎 補足１：UseCase層における契約ファイルの正体とは？

UseCase層には、2種類の契約があります。

| 契約の種類                                   | フォルダ                  | 対象                   | 内容               |
| --------------------------------------- | --------------------- | -------------------- | ---------------- |
| **Data Access Interface（Repository契約）** | `usecase/contracts/`  | Gateway／DB           | データ保存・検索の取り決め    |
| **Input／Output Boundary（入出力契約）**        | `usecase/boundaries/` | Controller／Presenter | リクエスト・レスポンスの取り決め |

UseCase層はこれらの契約を**中心に持つ層**であり、
**上下左右の外界と“抽象インターフェース”を通じて通信**します。

```text
Controller → InputBoundary → UseCaseInteractor → OutputBoundary → Presenter
                         ↓
                 Data Access Interface → Gateway → DB
```

> 💡 つまり UseCase層は、アプリ全体の「交通整理係」。
> 外側のどんな技術（Web, DB, API）が変わっても、
> **この層のコードは変わらない**のが理想です。

---

### 🧭 補足２：`boundaries` と `contracts` に分けた理由と一般的な命名

ここでは、**クラス図をそのままプログラム構造に写す**ことを目的として、
インターフェースを2種類に分けています。

| フォルダ名         | 含まれるインターフェース                 | 主な関係先                | 依存性逆転の有無      |
| ------------- | ---------------------------- | -------------------- | ------------- |
| `boundaries/` | InputBoundary／OutputBoundary | Controller・Presenter | ⚙️ 弱い（境界の明示）  |
| `contracts/`  | Repository Interface         | Gateway・DB           | ✅ 強い（DIPそのもの） |

この分割は「依存の方向」を明確にし、
学習者が**“どの方向で逆転が起きているか”を体感できる**ように設計されています。

---

### 💬 一般的なクリーンアーキテクチャとの違い

実務では、これらをまとめて **`interfaces/`** や **`ports/`** に置くことが多いです。

| 命名例           | 含まれる範囲                      | 主な文脈            |
| ------------- | --------------------------- | --------------- |
| `interfaces/` | Boundary＋Repository契約をまとめる  | 一般的なクリーンアーキ構成   |
| `ports/`      | Boundary（Input／Output）をまとめる | ヘキサゴナルアーキテクチャ   |
| `gateways/`   | Repository契約を表す             | Uncle BobやDDD文脈 |

---

### 🎓 教材での意図（教育上の整理）

> 🧭 一般的なプロジェクトでは `interfaces` にまとめられることが多いですが、
> この教材では「**クラス図と同心円図の対応を体感する**」ことを優先しています。
>
> そのため、Controller／Presenterとの契約を `boundaries/` に、
> Repositoryとの契約を `contracts/` に分けています。
>
> これは「現場の命名ルール」ではなく、
> **クリーンアーキテクチャを図から理解するための整理**です。

---

これを追記すると：

* 「boundaries／contracts の意味」→「一般的な命名との比較」→「教育的意図」
  という流れが自然につながり、
  「なぜこの構成なのか？」を読んだ学習者がすっきり理解できます。

必要であれば、直後に「🔄 次のステップ：Interface Adapter層へ」を再掲して章を閉じる構成にできます。



## ✅ **この章のゴール**

* クラス図どおりのBoundary／Interfaceを理解する
* 依存の向きが「内向き」である理由を体感する
* 第3章でのController／Presenter実装の準備が整った状態になる



## 🔄 次のステップ：Interface Adapter層へ

次章では、これらの契約を使って外部と橋渡しを行う層、
**Interface Adapter（Controller／Presenter／Gateway）** を実装します。

ここでは：

* Controller → InputBoundary（呼び出し側）
* Presenter → OutputBoundary（受け取り側）
* Gateway → Repository Interface（契約実装）

という**図の矢印どおり**の流れを、
実際のFastAPIコードで体験します。



