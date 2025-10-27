# 09 View

# 🖥 View
### `interface_adapters/views/view_console.py` - `ConsoleView`


## 🧭 このクラスの役割

`ConsoleView` は、**ユーザーが直接目にし、操作するユーザーインターフェース（UI）そのもの**です。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

このクラスの責務は大きく2つです：

1. **表示（Output）**
   `Presenter`によって準備された`TodoViewModel`のデータを、コンソール画面に具体的に描画する。
2. **入力（Input）**
   ユーザーからのキーボード入力を受け付け、その操作を`Controller`に伝える。

`View`は、アプリケーションの他の層（UseCase・Entity・Repositoryなど）を一切知らず、
**ただ与えられた`ViewModel`を表示し、ユーザー操作を`Controller`へ渡すだけ**の、純粋な「窓口」です。

---

## ✅ このクラスにあるべき処理

⭕️ **含めるべき処理の例**

* `print()`や`input()`など、具体的なUI技術（コンソール入出力）
* `ViewModel`の内容を描画する
* ユーザーの操作を受けて`Controller`を呼び出す

❌ **含めてはいけない処理の例**

* ビジネスロジック（UseCaseの責務）
* 表示用の整形ロジック（Presenterの責務）
* データベースアクセス（Repositoryの責務）

---

## 📁 ファイルの配置

```
├─ core/
│   └─ usecase/
│       └─ boundary/
│           ├─ dto.py                 # <DS> TodoViewModel 定義
│           ├─ input_boundary.py      # <I> Controllerが呼ぶ契約
│           ├─ output_boundary.py     # <I> Presenterが実装する契約
│
└─ interface_adapters/
    ├─ controllers/
    │   └─ todo_controller.py         # Controller（Viewが呼ぶ）
    ├─ presenters/
    │   └─ todo_presenter.py          # Presenter（OutputBoundary実装）
    └─ views/
        └─ view_console.py            # View本体（ConsoleView）
```

---

## 🔍 ソースコード（正式版）

```python
# --------------------------------------------------------------------
# File: interface_adapters/views/view_console.py
# Layer: Interface Adapters（View）
#
# 責務：
#   - ユーザーからの入力を受け取り、Controllerに渡す。
#   - Presenterによって更新されたViewModelをもとに画面を描画する。
#
# 依存：
#   - <I> Controller（入力の伝達先）
#   - <DS> TodoViewModel（表示内容のデータ構造）
#
# 同心円図での位置：
#   - Frameworks & Drivers（最外層）
#   - UIの実装層として、他レイヤーに依存するが逆依存はされない。
# --------------------------------------------------------------------

from interface_adapters.controllers.todo_controller import TodoController
from core.usecase.boundary.dto import TodoViewModel


class ConsoleView:
    """
    コンソール上でのユーザーとの対話（入力受付・画面表示）を担当する。
    """

    def __init__(self, controller: TodoController, view_model: TodoViewModel):
        """
        [依存関係]
        - Controller: 指示を伝える相手
        - ViewModel: 表示内容の元となるデータ
        """
        self._controller = controller
        self._view_model = view_model

    def run(self):
        """
        [入力処理]
        ユーザーからの入力を受け取り、Controllerに伝える。
        """
        title = input("追加するTODOのタイトルを入力してください: ")

        # Controllerに入力を渡してUse Caseを起動
        self._controller.add_todo(title)

        # 処理が終わったら、最新の状態で画面を再描画
        self.render()

    def render(self):
        """
        [表示処理]
        現在のViewModelの状態をもとに画面を描画する。
        """
        print("\n--- 画面表示 ---")
        print(self._view_model.display_text)
        print("----------------")
```

---

## 🧪 ユニットテスト例（Viewの責務確認）

```python
# --------------------------------------------------------------------
# File: tests/unit/test_view_console.py
# Layer: Test（Viewの単体テスト）
# --------------------------------------------------------------------

import unittest
from unittest.mock import patch
from core.usecase.boundary.dto import TodoViewModel
from interface_adapters.views.view_console import ConsoleView
from interface_adapters.controllers.todo_controller import TodoController


# フェイクController：呼び出し確認専用
class FakeController(TodoController):
    def __init__(self):
        self.called_with: str | None = None

    def add_todo(self, title: str):
        self.called_with = title


class TestConsoleView(unittest.TestCase):
    @patch("builtins.input", return_value="テスト入力")
    @patch("builtins.print")
    def test_view_calls_controller_and_renders(self, mock_print, mock_input):
        """ViewがControllerを呼び出し、ViewModelを表示することを確認"""
        vm = TodoViewModel(display_text="初期表示")
        controller = FakeController()
        view = ConsoleView(controller, vm)

        view.run()

        # Controllerが呼び出されたことを確認
        self.assertEqual(controller.called_with, "テスト入力")

        # ViewModelの内容が画面に表示されたことを確認
        printed_texts = [args[0] for args in mock_print.call_args_list]
        self.assertTrue(any("初期表示" in text for text in printed_texts))


if __name__ == "__main__":
    unittest.main()
```

---

## 🛡 このクラスの鉄則

> 愚直であれ（Be Dumb）

* Viewは「入力をControllerに渡す」「ViewModelを描画する」だけ
* ビジネスロジックや整形処理は一切行わない
* 単純であるほど、UI変更時の影響が少ない

---

## 🔍 補足：UI変更に強い理由

* **Viewは技術依存の最外層**
  → CLIをGUIやWebに差し替えても、Controller・Presenter・UseCaseはそのまま再利用可能

* **責務が薄い**
  → UI変更の影響範囲が限定され、保守性が高い

