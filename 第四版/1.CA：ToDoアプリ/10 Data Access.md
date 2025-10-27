# 10 Data Access

# 🗄 Data Access（Repository実装）:
### `infrastructure/repositories/in_memory_todo_repository.py` - `InMemoryTodoRepository`

---

## 🧭 このクラスの役割

`InMemoryTodoRepository` は、`core/domain/repository.py` に定義された `TodoRepository` インターフェース（＝契約）を、実際に動く形で実装したものです。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

* `TodoRepository` は「Todoを保存できる / 取り出せるべき」という**ビジネス上の要求**を表していました。
* しかし、それが「SQLiteに保存されるのか？」「メモリに保存されるのか？」「クラウドに保存されるのか？」はドメインの関心事ではありません。

そこで、**インフラ層（アプリの一番外側）**に、実際の保存方法（実装）を用意します。
今回の `InMemoryTodoRepository` は、名前の通り「プロセスのメモリ上（リスト）にTodoを保存する」バージョンです。

---

## 🧩 どの層に属するのか？

`InMemoryTodoRepository` は「インフラストラクチャ層（Infrastructure）」に属します。
層ごとの関係はこうなります：

* **Domain層**

  * `Todo`（Entity）
  * `TodoRepository`（Repositoryインターフェース / 抽象）
* **UseCase層**

  * `create_todo.py`（Interactor / ユースケース本体）
  * こいつは `TodoRepository` に依存する（実装は知らない）
* **Infrastructure層** ← 今回ここ！

  * `InMemoryTodoRepository`（Repositoryの実装 / 具体クラス）

ここで大事なのは依存方向です：

* UseCaseは「抽象である `TodoRepository` を知っている」
* 具体実装である `InMemoryTodoRepository` は、UseCaseの外側にあり、UseCaseからは知られない

つまり、**内側は外側を知らないが、外側は内側を知っている**という形を守っているわけです（依存性逆転の原則）。

---

## 📁 ファイルの配置

ここまでの全体の中で、Repository実装はこの位置に置きます。

```
├─ core/
│   ├─ domain/
│   │   ├─ todo.py                 # <E> Entity
│   │   ├─ repository.py           # <I> TodoRepository 抽象インターフェース
│   │   └─ errors.py
│   │
│   └─ usecase/
│       ├─ interactor/
│       │   └─ create_todo.py      # UseCase（Todoを追加する）
│       └─ boundary/
│           ├─ input_boundary.py
│           ├─ output_boundary.py
│           └─ dto.py
│
├─ interface_adapters/
│   ├─ presenters/
│   │   └─ todo_presenter.py
│   ├─ controllers/
│   │   └─ todo_controller.py
│   └─ views/
│       └─ view_console.py
│
└─ infrastructure/
    └─ repositories/
        └─ in_memory_todo_repository.py   # Repositoryの具体実装（今回の主役）
```

将来的にDBをSQLiteやPostgreSQLにする場合は、同じ場所に `sqlite_todo_repository.py` のような別実装を追加できます。
つまり「保存先が違うだけで、UseCaseは一切変更しなくてよい」というのがこの層分離の強みです。

---

## 🔍 ソースコード（InMemory版 Repository 実装）

```python
# --------------------------------------------------------------------
# File: infrastructure/repositories/in_memory_todo_repository.py
# Layer: Infrastructure（具体的なデータ保存方法）
#
# 目的:
#   - core/domain/repository.py の TodoRepository 抽象インターフェースを実装する。
#   - Todoエンティティをメモリ上のリストとして保持する。
#
# 特徴:
#   - 実際のDBを使わず、アプリを動かしながら動作確認・学習・テストできる。
#   - IDの採番もここで行う（UseCaseやEntityには採番ロジックを持たせない）。
#
# C言語の感覚に近い説明:
#   - ここは「静的なグローバル配列のようなもの」を使って
#     レコードを握っておく簡易ストレージ。
#   - UseCaseからはポインタ経由(=インターフェース経由)で呼ばれているイメージ。
# --------------------------------------------------------------------

from typing import List
from core.domain.todo import Todo
from core.domain.repository import TodoRepository


class InMemoryTodoRepository(TodoRepository):
    """
    メモリ上にTodoを保持するだけの簡易的なRepository実装。

    実運用ではDBやファイル保存に差し替えることになるが、
    この実装は「まずアプリを動かす」「ユースケースのテストを書く」ための最初の一歩として使える。
    """

    def __init__(self):
        # _store は Todo の一覧、_next_id は採番カウンタ
        self._store: List[Todo] = []
        self._next_id: int = 1

    def save(self, todo: Todo) -> Todo:
        """
        新しい Todo を保存し、採番したIDを付与して返す。

        UseCase側はIDを0など仮の値で渡してくる想定。
        永続化の層（=ここ）が正式なIDを振る責務を負う。
        """
        todo.id = self._next_id
        self._next_id += 1
        self._store.append(todo)
        return todo

    def find_all(self) -> List[Todo]:
        """
        すべての Todo を取得する。
        現状は単純にリスト全体のコピーを返す。
        """
        return list(self._store)
```

ポイント：

* `save()` の中で ID 採番をやっています。
  → **ID採番はインフラ責務**。UseCaseやEntityに入れない。
* `find_all()` はUseCaseから利用される想定（たとえば「一覧表示ユースケース」などで）。

---

## 🧪 ユニットテスト例（Repository実装の確認）

```python
# --------------------------------------------------------------------
# File: tests/unit/test_in_memory_todo_repository.py
# Layer: Test（Infrastructure層の単体テスト）
# --------------------------------------------------------------------

import unittest
from core.domain.todo import Todo
from infrastructure.repositories.in_memory_todo_repository import InMemoryTodoRepository


class TestInMemoryTodoRepository(unittest.TestCase):
    def test_save_assigns_incrementing_ids(self):
        """save() が Todo に連番IDを付与することを確認"""
        repo = InMemoryTodoRepository()

        t1 = repo.save(Todo(id=0, title="一つ目"))
        t2 = repo.save(Todo(id=0, title="二つ目"))

        self.assertEqual(t1.id, 1)
        self.assertEqual(t2.id, 2)

    def test_find_all_returns_all_saved_items(self):
        """保存したTodoが find_all() で取得できることを確認"""
        repo = InMemoryTodoRepository()

        repo.save(Todo(id=0, title="タスクA"))
        repo.save(Todo(id=0, title="タスクB"))

        todos = repo.find_all()
        titles = [t.title for t in todos]

        self.assertIn("タスクA", titles)
        self.assertIn("タスクB", titles)
        self.assertEqual(len(todos), 2)


if __name__ == "__main__":
    unittest.main()
```

このテストでは、「インフラ層の実装が期待通りの振る舞いをしているか？」だけを確認します。
他レイヤー（ControllerやView）には一切触れません。これがレイヤー分離の嬉しさです。

---

## 🛡 この層の鉄則

> 具体的な保存方法はここで完結しろ。内側には一切漏らすな。

* UseCaseは**抽象インターフェース（`TodoRepository`）**だけを知り、保存先の詳細は知らない
* Repository実装（ここ）は、DB・ネットワーク・ファイル・メモリなど「どう保存するか」をすべて引き受ける
* この分離によって、保存戦略の変更（メモリ→SQLite→クラウドDBなど）が**アプリ全体ではなくこの層だけの変更で済む**

---

## 🧠 一歩先の話（差し替えの未来）

この `InMemoryTodoRepository` は「まず動くもの」を支える役ですが、現実のアプリでは、たとえばこんな実装に置き換えることができます：

* `sqlite_todo_repository.py`（SQLiteでテーブルにINSERT/SELECTする実装）
* `file_todo_repository.py`（JSONやCSVファイルに書き込む実装）
* `api_todo_repository.py`（外部サービスのREST APIにPOSTする実装）

**どれに置き換えても、UseCaseやControllerやPresenterには一切変更が発生しない**のがクリーンアーキテクチャの威力です。

