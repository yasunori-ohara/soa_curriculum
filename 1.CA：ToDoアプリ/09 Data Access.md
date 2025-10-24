# 09 Data Access

# 🗄 Data Access : `adapters/data_access.py` - `InMemoryTodoDataAccess`

## 🧭 このクラスの役割

`InMemoryTodoDataAccess` は、**`TodoDataAccessInterface`という「契約書」を、具体的な技術（今回はインメモリ辞書）で実装するアダプター**です。

![クリーンアーキテクチャ](https://www.notion.so../%E3%82%AF%E3%83%AA%E3%83%BC%E3%83%B3%E3%82%A2%E3%83%BC%E3%82%AD%E3%83%86%E3%82%AF%E3%83%81%E3%83%A3.png)

`Use Case`からの「この`Todo`を保存してほしい」といった抽象的なデータ永続化の要求を、具体的なストレージ（今回はPythonの辞書）への読み書き処理に**変換**して実行します。

`Use Case`という「設計部門」と、`Database`という「製造部門」の間に立つ、「技術部門」のような役割を果たします。

---

## ✅ このクラスにあるべき処理

⭕️ **含めるべき処理の例**

- `TodoDataAccessInterface`で定義されたメソッド（`save`, `find_all`など）の具体的な実装
- 特定のデータソースとのやり取り（例：辞書の操作、ファイルへの書き込み、SQLクエリの実行）
- `Entity`オブジェクトと、永続化形式（DBの行データ、JSONなど）との相互変換

❌ **含めてはいけない処理の例**

- `Use Case`が担当すべきビジネスロジック
    - 例：「このユーザーはTODOを追加する権限があるか？」といった判断は**行いません**。ただ「保存して」という指示を忠実に実行するだけです。

---

## 🔍 ソースコード（コメント充実）

```python
# --------------------------------------------------------------------
# Data Access に相当
#
# <I> Data Access Interface（設計図）を、具体的な技術（今回はメモリ）で
# 実装するクラス。Interactorからのデータ永続化の要求を、実際に
# データベース（今回はPythonの辞書）に保存・読み込みすることで満たす。
#
# C言語の感覚に近い説明:
#   - このクラスは「関数ポインタ付き構造体」のようなもの。
#   - Interface（契約）に従って、関数を実装することで差し替え可能になる。
# --------------------------------------------------------------------

from typing import Dict, List
from domain.todo import Todo
from boundaries import TodoDataAccessInterface

class InMemoryTodoDataAccess(TodoDataAccessInterface):
    """
    TodoDataAccessInterfaceの「インメモリDB」版実装。
    これが変換アダプターの実体。
    """

    # Databaseの代わりとなるインメモリ（メモリ上）の辞書
    _database: Dict[int, Todo] = {}

    def save(self, todo: Todo) -> Todo:
        """
        [インターフェースの実装]
        'save'という抽象的な要求を、「辞書にデータを追加/更新する」という
        具体的な処理に変換して実行する。

        - IDが0の場合は新規追加とみなし、IDを自動採番する。
        - IDが既に存在する場合は上書き保存。
        """
        if todo.id == 0:
            # ID採番: 現在の最大ID + 1
            next_id = max(self._database.keys(), default=0) + 1
            todo.id = next_id

        self._database[todo.id] = todo
        return todo

    def find_all(self) -> List[Todo]:
        """
        [インターフェースの実装]
        'find_all'という抽象的な要求を、「辞書の全ての値をリストで返す」という
        具体的な処理に変換して実行する。
        """
        return list(self._database.values())

```

---

## 🧪 ユニットテスト例（保存処理の責務確認）

```python
# --------------------------------------------------------------------
# File: tests/test_data_access.py
# Layer: Test（Data Accessの単体テスト）
# --------------------------------------------------------------------

import unittest
from domain.todo import Todo
from adapters.data_access import InMemoryTodoDataAccess

class TestInMemoryTodoDataAccess(unittest.TestCase):
    def setUp(self):
        # テストごとに新しいインスタンスを使う（状態をリセット）
        self.repo = InMemoryTodoDataAccess()
        self.repo._database.clear()

    def test_save_assigns_id_if_zero(self):
        """ID=0のTodoを保存すると、自動採番されることを確認"""
        todo = Todo(id=0, title="新規タスク")
        saved = self.repo.save(todo)

        self.assertGreater(saved.id, 0)
        self.assertEqual(saved.title, "新規タスク")

    def test_save_preserves_existing_id(self):
        """IDが指定されている場合はそのIDで保存されることを確認"""
        todo = Todo(id=42, title="指定IDタスク")
        saved = self.repo.save(todo)

        self.assertEqual(saved.id, 42)
        self.assertEqual(self.repo._database[42].title, "指定IDタスク")

    def test_find_all_returns_all_saved_todos(self):
        """保存されたTodoがすべて取得できることを確認"""
        self.repo.save(Todo(id=0, title="A"))
        self.repo.save(Todo(id=0, title="B"))

        all_todos = self.repo.find_all()
        titles = [t.title for t in all_todos]

        self.assertIn("A", titles)
        self.assertIn("B", titles)
        self.assertEqual(len(all_todos), 2)

if __name__ == "__main__":
    unittest.main()

```

💡 **テストの意図**

- `save()` は ID=0 の場合に自動採番するか
- `save()` は 指定されたIDで保存できるか
- `find_all()` は 保存されたすべてのTodoを返すか
- 内部状態（辞書）を正しく操作しているか

---

## 🛡 このクラスの鉄則

> データを変換し、黙って実行せよ。 (Translate and execute, but do not think.)
> 
- このクラスはビジネスルールについて**思考しません**。Use Caseという管理者からの指示を、技術的な作業として忠実に実行するだけです。
- このファイルの素晴らしい点は、**差し替え可能**であることです。
この `InMemoryTodoDataAccess` を、PostgreSQLとやり取りする `PostgresTodoDataAccess` に差し替えるだけで、アプリケーションの永続化先を変更できます。
その際、**Use Case層のコードには一切変更が必要ありません**。