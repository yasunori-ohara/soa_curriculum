# 02 Repository Interface（Data Access Interface）

# 🗄 Repository Interface
### `core/domain/repository.py`

## 🎯 このファイルの役割

`Repository` は、**Entity（ドメイン）を永続化するための抽象的な倉庫**です。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

UseCase はデータベースの種類（SQLite / PostgreSQL / ファイルなど）を知るべきではありません。
そのため、「どんなデータ操作ができるか」だけを宣言した抽象インターフェースとして Repository を定義します。

💡 **「どこに保存するか」ではなく、「どんな操作が可能か」を宣言する場所**です。

---

## 📁 ファイルの配置

```
├─ core/
│   ├─ domain/
│   │   ├─ todo.py           # <E> Entity（ビジネスルール）
│   │   ├─ repository.py     # <I> Repository Interface（データアクセス契約）
│   │   └─ errors.py
│   │
│   └─ usecase/
│       └─ boundary/
│           ├─ input_boundary.py
│           ├─ output_boundary.py
│           └─ dto.py
```

---

## 🔍 ソースコード

```python
# --------------------------------------------------------------------
# File: core/domain/repository.py
# Layer: Domain（Entity 層に属する Data Access Interface）
#
# 目的:
#   - Entity（Todo）を永続化・取得するための抽象インターフェースを定義する。
#   - UseCase（Interactor）はこの抽象を利用し、DB技術の詳細を知らない。
#
# なぜ Domain 層に含まれるのか:
#   - Repository は「どんなデータ操作が必要か」というビジネス上の要件を表すため、
#     ビジネスルールの一部と見なされる。
#   - どのように保存するか（技術）は外側（Infrastructure）の責務。
#   - つまり「データが存在すること」自体がドメインの関心事であり、
#     その抽象を Domain 層が握ることで内側の安定性を保つ。
# --------------------------------------------------------------------

from abc import ABC, abstractmethod
from typing import List
from core.domain.todo import Todo


class TodoRepository(ABC):
    """
    Repository Interface（データアクセス契約）

    - Todo エンティティを保存・取得するための抽象。
    - 実際の保存先（DB・メモリ・API）は知らない。
    """

    @abstractmethod
    def save(self, todo: Todo) -> Todo:
        """新しい Todo を保存し、ID を付与して返す"""
        pass

    @abstractmethod
    def find_all(self) -> List[Todo]:
        """すべての Todo を取得する"""
        pass
```

---

## 💡 Repository が Domain に属する理由

| 項目                          | 説明                                                              |
| --------------------------- | --------------------------------------------------------------- |
| **表しているのは「技術」ではなく「要件」**     | 「Todoを保存・取得できること」はビジネス要件であり、DBの種類とは関係がない。                       |
| **安定性の中心に置くため**             | Domain 層は最も変更されにくい層。Repository 抽象をここに置くことで、DBを変更しても内側のコードが壊れない。 |
| **「保存できる」という行為はビジネスルールの一部** | Todoが永続的に管理されることは、業務ドメインにとって重要な性質。                              |

---

## 🧪 概念テスト（フェイク実装）

```python
from core.domain.repository import TodoRepository
from core.domain.todo import Todo

class InMemoryTodoRepository(TodoRepository):
    """メモリ上で動作する簡易リポジトリ（テスト用）"""
    def __init__(self):
        self._store = []
        self._next_id = 1

    def save(self, todo: Todo) -> Todo:
        todo.id = self._next_id
        self._next_id += 1
        self._store.append(todo)
        return todo

    def find_all(self):
        return list(self._store)


repo = InMemoryTodoRepository()
repo.save(Todo(id=0, title="書類作成"))
print(repo.find_all())
```

---

## 🛡 鉄則

> Repository は「技術のためのクラス」ではない。
> 「ビジネス上、どんなデータ操作が必要か」を表すための抽象である。

* DB技術を隠蔽することで、UseCaseは「保存できる」という事実だけを前提に動作できる。
* 永続化技術（SQLAlchemy, Firestore, JSONファイルなど）は外側の実装層で自由に差し替え可能。
* これにより、**ビジネスロジックは技術的変更から完全に独立**できる。

---

# 🧪 Repository の深堀

## 💡 その１：Repository は「技術」ではなく「ビジネス要件」

たとえば「Todoを保存できる」「Todoを一覧取得できる」という行為は、
アプリケーションの技術的な都合ではなく、**ドメインの“ルール”や“要件”の一部**です。

| 層                   | 何を表すか         | 例                              |
| ------------------- | ------------- | ------------------------------ |
| **Domain層**         | ビジネスそのものの関心事  | 「Todoを保存・取得できること」              |
| **UseCase層**        | ビジネスルールの組み合わせ | 「Todoを追加する」「完了にする」             |
| **Infrastructure層** | 技術の詳細         | 「SQLiteで保存する」「HTTP API経由で取得する」 |

つまり、**Repositoryは「どんな操作が必要か」を表す契約であって、どう保存するかは関心外**。
だから、**Domainの内側に置く方が論理的に正しい**のです。

---

## 💡 その２：「永続化できること」がドメインの一部だから

例えば「Todoを登録できる」「Todoを一覧で見る」といった行為は、
このアプリケーションのビジネス上の基本機能であり、**ドメインモデルの期待値**です。

そのため、

* Domainは「保存できる」という抽象を知っておく必要がある
* しかし、「どのように保存するか（SQL？JSON？）」は知らなくていい

この線引きのため、Repository InterfaceはDomain層に含めます。

---

## 💡 その３：DDD（ドメイン駆動設計）との整合性

DDD（Domain-Driven Design）でも、
Repositoryは「ドメイン層の一部として定義し、実装はインフラ層に置く」ことが原則です。

> Eric Evans『Domain-Driven Design』では、
> 「Repositoryはドメインオブジェクトの集合を表すインターフェースであり、
> ドメイン層に属する。ただし実装はインフラストラクチャ層に置く」と定義されています。

---

## 💡 その４：UseCaseが外部技術に依存しないため

UseCase（Interactor）が Repository Interface を使うとき、
その実体がDBでもファイルでもHTTPでも関係なく動作できる必要があります。

Domain配下に Repository を置くことで、
UseCaseは「Domainにある契約（抽象）」だけを見て動けるようになります。

---

## 🧩 図で見る依存方向

```
Frameworks/UI
   ↓
InterfaceAdapter（Controller / Presenter）
   ↓
UseCase（Interactor）
   ↓
Domain（Entity, Repository Interface）
   ↓
Infrastructure（DB / 外部API 実装）
```

矢印の向き（依存方向）はすべて「内側へ」。
Repository Interface が Domain にあることで、この構造が成立します。

---

## ✅ 結論まとめ

| 配置案                          | 評価        | 理由                               |
| ---------------------------- | --------- | -------------------------------- |
| `core/usecase/repository.py` | ❌ 不適切     | UseCase層が技術境界を持つと依存方向が逆転する       |
| `core/domain/repository.py`  | ✅ 適切      | Repositoryは「技術」ではなく「ビジネス上の抽象」だから |
| `infra/repository_impl.py`   | ✅ 適切（実装側） | DBやAPIの具象はインフラ層で自由に実装すべき         |

---

## 🧠 一言でまとめると

> Repository は「ビジネスのための契約」だから Domain に置く。
> それを「技術的にどう実現するか」は、もっと外側（Infrastructure）の責務。
