# 08 View

# 🖥 View : `ui/view_console.py` - `ConsoleView`

---

## 🧭 このクラスの役割

`ConsoleView` は、**ユーザーが直接目にし、操作するユーザーインターフェース（UI）そのもの**です。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

このクラスの責務は、大きく分けて2つです：

1. **表示（Output）**`Presenter`によって準備された`ViewModel`のデータを、画面（今回はコンソール）に具体的に描画する。
2. **入力（Input）**
ユーザーからのキーボード入力などを受け付け、その操作を`Controller`に伝える。

`View`は、アプリケーションの他の部分がどうなっているかを知る必要はありません。

ただ、与えられた設計図（`ViewModel`）通りに画面を描画し、ユーザーの操作を`Controller`に伝えるだけの、純粋な「窓口」です。

---

## ✅ このクラスにあるべき処理

⭕️ **含めるべき処理の例**

- `print()`や`input()`など、具体的なUI技術（コンソール入出力）を直接扱うコード
- `ViewModel`のデータを読み取って、画面に表示するロジック
- ユーザーの操作に応じて、`Controller`のメソッドを呼び出すこと

❌ **含めてはいけない処理の例**

- ビジネスロジック（`Use Case`の責務）
- 表示用データの整形ロジック（`Presenter`の責務）
- データベースや外部APIに関する知識

---

## 🔍 ソースコード（コメント完全保持）

```python
# --------------------------------------------------------------------
# View に相当
#
# 責務：ユーザーとの直接的なやり取り（入力受付・画面表示）を担当する。
# 依存：<DS> ViewModel を知っており、Controller を呼び出す。
#
# 位置づけ（クラス図／同心円図）：
# - クラス図：Interface Adapters（View）
# - 同心円図：Framework & Drivers（実際の入出力を扱う最外層）
# --------------------------------------------------------------------

from interface_adapters.controller import AddTodoController
from data_structures import TodoViewModel

class ConsoleView:
    """
    ユーザーとの直接的なやり取り（入力受付・画面表示）を担当する。
    """

    def __init__(self, controller: AddTodoController, view_model: TodoViewModel):
        """
        [依存先の定義]
        指示を出す相手であるControllerと、
        表示内容の元となるViewModelを保持する。
        """
        self._controller = controller
        self._view_model = view_model

    def run(self):
        """
        [入力処理]
        ユーザーからの入力を受け付け、Controllerに伝達する。
        """
        title = input("追加するTODOのタイトルを入力してください: ")

        # Controllerに入力を渡し、Use Caseの実行を間接的に依頼
        self._controller.add_todo(title)

        # 処理が終わったら、最新の状態で画面を再描画する
        self.render()

    def render(self):
        """
        [表示処理]
        自身の持つViewModelの現在の状態を元に画面を描画する。
        """
        print("\\n--- 画面表示 ---")
        print(self._view_model.display_text)
        print("----------------")

```

---

## 🧪 ユニットテスト例（Viewの責務確認）

```python
# --------------------------------------------------------------------
# File: tests/test_view_console.py
# Layer: Test（Viewの単体テスト）
# --------------------------------------------------------------------

import unittest
from unittest.mock import patch
from data_structures import TodoViewModel
from ui.view_console import ConsoleView
from interface_adapters.controller import AddTodoController

# フェイクController: 呼び出し確認用
class FakeController(AddTodoController):
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

        # Controllerが呼ばれたか
        self.assertEqual(controller.called_with, "テスト入力")

        # ViewModelの内容が画面に表示されたか
        printed_texts = [args[0] for args in mock_print.call_args_list]
        self.assertTrue(any("初期表示" in text for text in printed_texts))

```

💡 **テストの意図**

- Viewは **Controllerを呼び出す責務** を果たしているか
- Viewは **ViewModelの内容を表示する責務** を果たしているか
- Viewは **ビジネスロジックやデータ整形を行っていない**ことを確認する

---

## 🛡 このクラスの鉄則

> 愚直であれ（Be Dumb）
> 
- Viewは「入力をControllerに渡す」「ViewModelを表示する」だけ
- ビジネスロジックやデータ整形は一切行わない
- この単純さがUI差し替えの柔軟性を生む

---

## 🔍 補足：UI変更に強い理由

- **Viewは技術依存の最外層**
    
    → コンソールからGUIやWebに変えても、ControllerとPresenterは再利用可能
    
- **責務が薄い**
    
    → UI変更時の影響範囲を最小化