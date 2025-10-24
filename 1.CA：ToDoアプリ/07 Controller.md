# 07 Controller

# 🚦 Controller : `interface_adapters/controller.py` - `AddTodoController`

## 🧭 このクラスの役割

`AddTodoController` は、**`View`からのユーザー入力を受け取り、それを`Use Case`が理解できる形式（`InputData`）に整えて「伝達」する交通整理員**です。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

`View`はユーザーが何をしたか（例：ボタンがクリックされた）を知っていますが、それがビジネス的に**何を意味するのか**は知りません。

`Controller`は、その「クリック」というUIイベントを、「TODOを追加せよ」というビジネス上のリクエストに変換し、適切な`Use Case`の窓口に渡すことが唯一の責務です。

`View`という「受付」と、`Use Case`という「専門部署」の間に立つ、薄いながらも重要な**仲介役**です。

---

## ✅ このクラスにあるべき処理

⭕️ **含めるべき処理の例**

- `View`からのイベント（メソッド呼び出し）を受け取る
- `View`から渡された生データ（例：文字列）を、`InputData`オブジェクトに変換する
- `InputBoundary`インターフェースを通じて、適切な`Use Case`を呼び出す

❌ **含めてはいけない処理の例**

- ビジネスロジックそのもの（`Use Case`の責務）
- 画面表示の準備（`Presenter`の責務）
- 画面を直接描画する処理（`View`の責務）
- データベースに関する知識

---

## 🔍 ソースコード（コメント完全保持）

```python
# --------------------------------------------------------------------
# Controller に相当
#
# 責務：Viewからのユーザー入力を受け取り、Use Caseが理解できる形式（InputData）に変換して、
#       Use Case（のInputBoundary）を呼び出す。
# 依存：<I> Input Boundary と <DS> InputData を知っている。
#
# 位置づけ（クラス図／同心円図）：
# - クラス図：Interface Adapters（Controller）
# - 同心円図：Interface Adapters 層（UIイベントをビジネスリクエストに変換）
# --------------------------------------------------------------------

from boundaries import AddTodoInputBoundary
from data_structures import AddTodoInputData

class AddTodoController:
    """
    ControllerはUIイベントとビジネスロジックを繋ぐ薄い層。
    """

    def __init__(self, use_case: AddTodoInputBoundary):
        """
        [依存先の定義]
        呼び出し先であるUse Caseの具体的な実装クラスではなく、
        その「窓口」であるAddTodoInputBoundaryインターフェースにのみ依存する。
        """
        self._use_case = use_case

    def add_todo(self, title: str) -> None:
        """
        [伝達処理]
        Viewからの生の入力(title)を、InputDataという公式な形式に変換し、
        Use Caseの実行を依頼する。
        """
        # 1. Viewからの生データを、InputDataオブジェクトに変換
        input_data = AddTodoInputData(title=title)

        # 2. 変換したデータを添えて、インターフェースを通じてUse Caseの実行を依頼
        self._use_case.execute(input_data)

```

---

## 🧪 ユニットテスト例（Controllerの責務確認）

```python
# --------------------------------------------------------------------
# File: tests/test_controller.py
# Layer: Test（Controllerの単体テスト）
# --------------------------------------------------------------------

import unittest
from data_structures import AddTodoInputData
from boundaries import AddTodoInputBoundary
from interface_adapters.controller import AddTodoController

# フェイクUseCase: Controllerが正しく呼び出すか確認するため
class FakeAddTodoUseCase(AddTodoInputBoundary):
    def __init__(self):
        self.last_input: AddTodoInputData | None = None

    def execute(self, input_data: AddTodoInputData) -> None:
        self.last_input = input_data

class TestController(unittest.TestCase):
    def test_controller_calls_use_case_with_input_data(self):
        """ControllerがInputDataを生成し、UseCaseに渡すことを確認"""
        fake_use_case = FakeAddTodoUseCase()
        controller = AddTodoController(fake_use_case)

        controller.add_todo("テストタスク")

        # UseCaseに正しいデータが渡されたか確認
        self.assertIsNotNone(fake_use_case.last_input)
        self.assertEqual(fake_use_case.last_input.title, "テストタスク")

if __name__ == "__main__":
    unittest.main()

```

💡 **テストの意図**

- Controllerは **InputDataを生成してUse Caseに渡す**責務を果たしているか
- ビジネスロジックやUI描画の責務を持っていないことを確認

---

## 🛡 このクラスの鉄則

> 入力を整理し、然るべき場所に渡せ。
> 
- Controllerはビジネスロジックを持たない
- UIとUse Caseの間に立つ「緩衝材」として薄く保つ
- この層があることで、UIとビジネスロジックが直接依存しない

---

## 🔍 補足：Controllerの重要性

- **依存性逆転の一部**
    
    ControllerはUIイベントをUse Caseに渡すが、Use CaseはControllerを知らない → 双方向依存を防ぐ
    
- **テスト容易性**
    
    Controllerは単体テストで簡単に検証可能（フェイクUseCaseを注入）
    
- **UI差し替えの柔軟性**
    
    Web、CLI、GUIなど、Viewを変えてもControllerの責務は変わらない