# 02 Use Case Interactor

# Use Case : `application/todo_UseCase.py`

## 🧭 このクラスの役割

`TodoUseCase` は、このアプリケーションにおける**具体的なシナリオ（ユースケース）を一つ担当するクラスです。`Entity`がビジネスの「材料」だとしたら、Use Caseはそれらの材料を使って特定の料理（目的）を完成させるための「レシピ」であり、「シェフ」の役割**を担います。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

このクラスは、「TODOを追加する」というユーザーの目的を達成するための一連の手順を定義し、`Entity`や外部の機能（DBなど）を指揮（オーケストレーション）します。

`Entity`が持つ普遍的なルールとは異なり、この**アプリケーション固有のビジネスルール**がここに実装されます。

---

## 🧑‍🍳 Use Casesとは何か？

Use Casesは、アプリケーション固有のビジネスルールを実装する層です。

Entitiesがビジネスの**「名詞」**と**「物理法則」**だとしたら、Use Casesはその法則を使って**具体的な目的を達成するための「動詞」や「レシピ」**にあたります。

ユーザーがこのアプリケーションで「何をしたいか」（例：「TODOを追加したい」「商品を注文したい」）を、一つ一つのクラスとして表現します。

このクラスは、**Interactor（インタラクター）**とも呼ばれます。

---

## ⚙️ このクラスにあるべき処理

✅ **含めるべき処理の例**

- **Entityの指揮（オーケストレーション）**
- **アプリケーション固有のルール**
- **外部への要求（インターフェースを通じた呼び出し）**

❌ **含めてはいけない処理の例**

- UIに関する知識（`print`文など）
- DBに関する知識（SQLなど）
- フレームワーク依存コード（Django/Flaskなど）
- 具体クラスへの依存（PresenterやRepositoryの実装を直接使う）

---

## 🔍 ソースコード

### 【補足：コードで使う Repository とは？】

この後出てくるコードでは、データ永続化（保存・取得）を担当するインターフェースを、クラス図にある「Data Access Interface」という名前ではなく、Repository (リポジトリ) という名前で扱います。
Repositoryとは、ドメイン駆動設計（DDD）で使われるパターンで、**「Entity（Todo）の集合を管理する抽象的な倉庫」**を意味します。この倉庫に保存を依頼することで、私たちはデータベースの種類を一切知らずに済みます。

```python
# --------------------------------------------------------------------
# File: application/todo_UseCase.py
# Layer: Use Case（アプリケーション固有のビジネスルール）
#
# 目的:
#   - TODO管理に関するユースケースを実現するクラス。
#   - Entity（材料）を指揮し、外部システム（保存・表示）はインターフェース越しに依頼する。
#
# C言語の感覚に近い説明:
#   - クラスは「構造体＋関数」のようなもの。
#   - 依存性注入（DI）は「関数ポインタ（抽象インターフェース）を受け取って差し替え可能にする」イメージ。
#   - 例外は「失敗を呼び出し側へ通知する仕組み」。必要に応じてPresenterが人向けに整形する。
# --------------------------------------------------------------------

from domain.todo import Todo
from boundaries import (
    TodoInputBoundary,
    TodoOutputBoundary,
    TodoDataAccessInterface,
)
from data_structures import (
    TodoInputData,
    TodoOutputData,
)

class TodoUseCase(TodoInputBoundary):
    """
    「TODOを追加する」というユースケースを実装するクラス。

    - TodoInputBoundary（入力契約）を実装することで、
      外部の層はこのクラスを「契約書どおりに呼び出せる」。
    """

    def __init__(self, presenter: TodoOutputBoundary, repository: TodoDataAccessInterface):
        """
        [依存性の注入（Dependency Injection）]
        このクラスが動作するために必要な「部品」（PresenterとRepository）を、
        具体的な実装クラスではなく、抽象的なインターフェース（Boundary）として受け取る。
        これにより、UIやDBの実装からこのクラスは完全に独立する。
        """
        self._presenter = presenter
        self._repository = repository

    def execute(self, input_data: TodoInputData) -> None:
        """
        [ビジネスロジックの実行]
        このユースケースにおける具体的な処理手順（レシピ）を記述する。
        """
        # --- 前処理: タイトルの空白を除去（UX向上・品質担保） ---
        normalized_title = input_data.title.strip()

        # Entityが詳細な検証（空文字禁止・ID正）を担当するため、
        # ここでは「人間が入力しがちな不要な空白」の除去のみに留める。

        # --- 保存依頼: 具体実装は知らずに「契約」経由で依頼 ---
        # Entityを生成し、Repositoryに保存を依頼する
        # IDはEntityの必須属性だが、具体的な採番はRepositoryの責務であるため、
        # ここでは「未採番」の印としてID=0でTodo Entityを生成する。
        new_todo = Todo(id=0, title=normalized_title)

        # --- 保存依頼: 具体実装は知らずに「契約」経由で依頼 ---
        # Repository（データ永続化層）に保存を依頼。
        # Repositoryは保存時にIDを割り当て、その結果（IDがセットされた新しい状態のTodo Entity）を返す。
        saved_todo = self._repository.save(new_todo)

        # --- 出力データ構築: Presenterに渡すための箱に詰める ---
        # 結果を出力データ構造（OutputData）に変換する
        # saved_todoからIDとタイトルを取り出し、Presenterに渡すデータを作成する。
        output_data = TodoOutputData(id=saved_todo.id, title=saved_todo.title)

        # --- 結果通知: 出力境界に従ってPresenterへ渡す ---
        # 出力境界（OutputBoundary）を通じて、Presenterに結果を渡す
        self._presenter.present(output_data)

```

---

## 🧪 この段階でユニットテストをすることができます

Use Caseは外部の具体実装（DB・UI）を**インターフェース越しに扱う**ため、**フェイク（簡易実装）**を注入すれば、この段階でユニットテストが可能です。

以下に、**最小限のフェイクRepository・フェイクPresenter**を使ったテスト例を示します。

### 📁 `tests/test_todo_usecase.py`

```python
# --------------------------------------------------------------------
# ユニットテスト例: TodoUseCase の振る舞い確認
# - フェイクRepository/Presenterで、外部依存を排除
# - Python標準の unittest を使用（初心者にも扱いやすい）
# --------------------------------------------------------------------

import unittest
from application.todo_UseCase import TodoUseCase
from domain.todo import Todo
from boundaries import TodoOutputBoundary, TodoDataAccessInterface
from data_structures import TodoInputData, TodoOutputData

# --- フェイク実装: Presenter（結果受け取り側） ---
class FakePresenter(TodoOutputBoundary):
    """結果の受け取りを記録するだけのフェイクPresenter"""
    def __init__(self):
        self.last_output = None

    def present(self, output_data: TodoOutputData) -> None:
        # Presenterは通常、ViewModelを更新するが、
        # このフェイクは渡されたデータをそのまま保持するだけ
        self.last_output = output_data

# --- フェイク実装: Repository（保存・取得側） ---
class InMemoryTodoRepository(TodoDataAccessInterface):
    """メモリ上にTodoを保持するだけの簡易Repository"""
    def __init__(self):
        self._store = []
        self._next_id = 1  # ID採番用のカウンター

    def find_all(self):
        # 現在の一覧を返す（コピーで返すとより安全）
        return list(self._store)

    def save(self, todo: Todo) -> Todo:
        # Entityが仮ID（0）で渡されるので、ここで正式なIDを付与する
        todo.id = self._next_id
        self._next_id += 1
        self._store.append(todo)
        return todo

class TestTodoUseCase(unittest.TestCase):
    def setUp(self):
        # PresenterとRepositoryをフェイクで用意し、UseCaseに注入
        self.presenter = FakePresenter()
        self.repository = InMemoryTodoRepository()
        self.use_case = TodoUseCase(self.presenter, self.repository)

    def test_add_first_todo(self):
        # 入力（前後空白はUseCaseで除去される）
        input_data = TodoInputData(title="  最初のタスク  ")
        self.use_case.execute(input_data)

        # Presenterに結果が渡っていること
        self.assertIsNotNone(self.presenter.last_output)
        self.assertEqual(self.presenter.last_output.id, 1)
        self.assertEqual(self.presenter.last_output.title, "最初のタスク")

        # Repositoryに保存されていること
        all_todos = self.repository.find_all()
        self.assertEqual(len(all_todos), 1)
        self.assertEqual(all_todos[0].id, 1)
        self.assertEqual(all_todos[0].title, "最初のタスク")
        self.assertFalse(all_todos[0].completed)

    def test_add_second_todo_increments_id(self):
        # 先に1件保存（ID=1）
        self.repository.save(Todo(id=0, title="既存タスク"))
        # 次を追加（ID=2になるはず）
        input_data = TodoInputData(title="次のタスク")
        self.use_case.execute(input_data)

        self.assertEqual(self.presenter.last_output.id, 2)

    def test_empty_title_raises_value_error_from_entity(self):
        # 空白のみ → UseCaseがstripし、Entityが空文字エラーを投げる
        input_data = TodoInputData(title="   ")
        with self.assertRaises(ValueError):
            self.use_case.execute(input_data)

if __name__ == "__main__":
    unittest.main()

```

> 💡 テストの着眼点
> 
> - Presenterへの通知（`present()`に正しい値が渡る）
> - Repositoryに保存される（追加後の状態確認）
> - IDの採番ルール（自動採番）
> - エラー処理（Entityが投げた例外をそのまま伝える）

---

## 🛡 このクラスの鉄則

> ビジネスロジックを指揮せよ、ただし詳細には関わるな。
> 
- 外側のレイヤー（Adapters, Frameworks）の変更が、このレイヤーに影響を与えてはいけません。
- この層はアプリケーションが**何をするか（What）**を定義し、外側は**どうやって（How）**を定義します。

---

## 📝 補足：説明の順番について

このアーキテクチャでは、**内側の層が外側へ「何が必要か」を宣言**します。

1. **Use Case が「この仕事を頼むなら、この入力データ（`InputData`）で」と要求**
2. それに応じて、**`InputData`／`OutputData` とインターフェース（`Boundary`）**が設計される

料理の比喩で言えば、**シェフ（Use Case）**が目的から逆算して、必要な**道具（Data Structures／Interfaces）**を定義する流れです。

---

## 🔧 実務への改善提案（参考）

- 🛑 **エラー提示用のOutputBoundary**（`present_error()`など）
- 🔁 **トランザクション境界の明示**（複数操作の一体性）