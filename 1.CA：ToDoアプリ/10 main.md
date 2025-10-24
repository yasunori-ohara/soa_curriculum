# 10 main

# 🚀 Composition Root : `main.py`

## 🧭 このファイルの役割

`main.py` は、これまで作ってきた全ての部品（クラス）を組み立てて一つのアプリケーションとして完成させ、起動スイッチを入れる「組立工場」の役割を果たします。

![クリーンアーキテクチャ](https://www.notion.so../%E3%82%AF%E3%83%AA%E3%83%BC%E3%83%B3%E3%82%A2%E3%83%BC%E3%82%AD%E3%83%86%E3%82%AF%E3%83%81%E3%83%A3.png)

クリーンアーキテクチャでは、この場所を **Composition Root** と呼びます。ここは、アプリケーション全体で **唯一**、具体的な実装クラス（`AddTodoPresenter`, `InMemoryTodoDataAccess`など）が互いを直接知ることを許された場所です。

ここで全ての部品を配線（**依存性の注入 / Dependency Injection**）することで、他の全てのクラス（`UseCase`や`Controller`など）は、お互いの具体的な実装を知らない「クリーン」な状態を保つことができます。

---

## ✅ このファイルにあるべき処理

⭕️ **含めるべき処理の例**

- 全ての具体的な実装クラスのインスタンス化（オブジェクト生成）
- あるクラスのインスタンスを、別のクラスのコンストラクタ（`__init__`）に渡すことによる、依存性の注入（DI）
- アプリケーションの最初の処理（今回は `view.run()`）を呼び出すこと

❌ **含めてはいけない処理の例**

- ビジネスロジック、データ変換ロジック、UIの描画ロジックなど
→ このファイルの責務は、あくまで **組み立て** に専念します

---

## 🔍 ソースコード（コメント充実）

```python
# --------------------------------------------------------------------
# File: main.py
# Layer: Composition Root（依存関係の組み立てと起動）
#
# 目的:
#   - アプリケーションのすべての部品を組み立てて起動する。
#   - 依存性注入（DI）を一手に引き受ける唯一の場所。
#
# C言語の感覚に近い説明:
#   - main関数は「初期化処理＋最初の関数呼び出し」に相当。
#   - ここで構造体や関数ポインタを組み合わせて、全体の流れを構築する。
# --------------------------------------------------------------------

# --- UI層のクラス ---
from interface_adapters.presenter import AddTodoPresenter
from interface_adapters.controller import AddTodoController
from ui.view_console import ConsoleView

# --- アダプター層（UI以外）のクラス ---
from adapters.data_access import InMemoryTodoDataAccess

# --- ユースケース ---
from application.todo_UseCase import TodoUseCase

# --- データ構造 ---
from data_structures import TodoViewModel

def main():
    """アプリケーションのすべての部品を組み立て、起動する"""
    print("--- 1. アプリケーションの部品を組み立てます ---")

    # [ViewModel] を生成
    # ViewとPresenterの間で共有される、画面の状態を持つオブジェクト
    view_model = TodoViewModel()

    # [Presenter] を生成（ViewModelを注入）
    # 依存するViewModelを注入（DI）
    presenter = AddTodoPresenter(view_model)

    # [Data Access] を生成（インメモリDB）
    data_access = InMemoryTodoDataAccess()

    # [Use Case] を生成（PresenterとDataAccessを注入）
    # 依存するPresenterとDataAccessを注入（DI）
    use_case = TodoUseCase(presenter, data_access)

    # [Controller] を生成（Use Caseを注入）
    # 依存するUse Caseを注入（DI）
    controller = AddTodoController(use_case)

    # [View] を生成（ControllerとViewModelを注入）
    # 依存するControllerと、共有するViewModelを注入（DI）
    view = ConsoleView(controller, view_model)

    print("--- 2. 組み立て完了、アプリケーションを実行します ---\\n")

    # アプリケーションの実行開始
    view.run()

if __name__ == "__main__":
    main()

```

---

## 🧪 ユニットテスト例（起動構成の確認）

> Composition Rootは通常テスト対象ではありませんが、依存関係の組み立てが正しく行われているかを確認することで、構成ミスを防ぐことができます。
> 

```python
# --------------------------------------------------------------------
# File: tests/test_main_composition.py
# Layer: Test（Composition Rootの構成確認）
# --------------------------------------------------------------------

import unittest
from interface_adapters.presenter import AddTodoPresenter
from interface_adapters.controller import AddTodoController
from ui.view_console import ConsoleView
from adapters.data_access import InMemoryTodoDataAccess
from application.todo_UseCase import TodoUseCase
from data_structures import TodoViewModel

class TestMainComposition(unittest.TestCase):
    def test_component_wiring(self):
        """各コンポーネントが正しく接続されることを確認"""
        vm = TodoViewModel()
        presenter = AddTodoPresenter(vm)
        repo = InMemoryTodoDataAccess()
        use_case = TodoUseCase(presenter, repo)
        controller = AddTodoController(use_case)
        view = ConsoleView(controller, vm)

        # ViewがControllerとViewModelを保持しているか
        self.assertIs(view._controller, controller)
        self.assertIs(view._view_model, vm)

        # ControllerがUseCaseを保持しているか
        self.assertIs(controller._use_case, use_case)

        # UseCaseがPresenterとRepositoryを保持しているか
        self.assertIs(use_case._presenter, presenter)
        self.assertIs(use_case._repository, repo)

if __name__ == "__main__":
    unittest.main()

```

💡 **テストの意図**

- 各コンポーネントが正しく接続されているか（DIが成立しているか）
- main.pyの構成が壊れていないかを早期に検知するための安全確認

---

## 🛡 このファイルの鉄則

> すべてを知り、すべてを組み立て、そして仕事は最初に任せよ。
> 
- この `main.py` は、アプリケーションの具体的な実装クラスをすべて知っている、唯一の「汚れる」場所です。
- この場所が **依存関係を一手に引き受ける** ことで、他のすべてのクラスはインターフェースにのみ依存する「クリーン」な状態を保つことができます。
- 組み立てが終わったら、最初のきっかけ（`view.run()`）を与えるだけで、あとは他のクラスに仕事を任せます。