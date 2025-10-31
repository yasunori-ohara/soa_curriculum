# 08 Controller

# 🚦 Controller
### `interface_adapters/controllers/todo_controller.py` - `TodoController`

## 🧭 このクラスの役割

`TodoController` は、**`View`からのユーザー入力を受け取り、それを`Use Case`が理解できる形式（`TodoInputData`）に整えて「伝達」する交通整理員**です。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

`View`はユーザーが何をしたか（例：テキスト入力・ボタン押下）を知っていますが、それがビジネス的に**何を意味するのか**は知りません。

`Controller`は、その「クリック」「送信」といったUIイベントを、「TODOを追加せよ」というビジネス上のリクエストに変換し、適切な`Use Case`の窓口に渡します。

つまり `View` という「受付」と、`Use Case` という「専門部署」の間に立つ、薄いながらも重要な**仲介役**です。

---

## ✅ このクラスにあるべき処理

⭕️ **含めるべき処理の例**

* `View`からのイベント（メソッド呼び出し）を受け取る
* `View`から渡された生データ（例：ユーザー入力の文字列）を、`TodoInputData` オブジェクトに変換する
* `TodoInputBoundary` インターフェースを通じて、適切な `Use Case` を呼び出す

❌ **含めてはいけない処理の例**

* ビジネスロジックそのもの（それは UseCase の責務）
* 画面表示用メッセージの組み立て（それは Presenter の責務）
* 画面を直接描画する処理（それは View の責務）
* データベースに関する知識（Repository / Gateway の責務）

---

## 📁 ファイルの配置

Controller は「インターフェースアダプタ層」に属します。
UseCase の入口契約 (`InputBoundary`) は core 側にあります。

```
├─ core/
│   └─ usecase/
│       └─ boundary/
│           ├─ input_boundary.py      # <I> InputBoundary（Controllerが呼ぶ入口契約）
│           ├─ output_boundary.py     # <I> OutputBoundary（Presenterが実装する出口契約）
│           └─ dto.py                 # <DS> TodoInputData / TodoOutputData / TodoViewModel
│
└─ interface_adapters/
    └─ controllers/
        └─ todo_controller.py         # Controller本体（Viewから呼ばれる）
```

---

## 🔍 ソースコード（コメント完全保持ベース）

```python
# --------------------------------------------------------------------
# File: interface_adapters/controllers/todo_controller.py
# Layer: Interface Adapters（Controller）
#
# 責務：
#   Viewからのユーザー入力を受け取り、
#   Use Caseが理解できる形式（TodoInputData）に変換して、
#   Use Case（のInputBoundary）を呼び出す。
#
# 依存：
#   - <I> InputBoundary（ユースケースの入口となる契約）
#   - <DS> TodoInputData（ユースケースに渡す公式な入力データ構造）
#
# 位置づけ（クリーンアーキテクチャの図）：
#   - クラス図：Interface Adapters（Controller）
#   - 同心円図：Interface Adapters 層
#                （UIイベントをビジネスリクエストに変換する層）
#
# C言語の感覚に近い説明:
#   - Controllerは「UIイベントハンドラ」のようなもの。
#   - UseCaseを直接触るのではなく、事前に決められた関数プロトタイプ
#     （InputBoundaryインターフェース）を呼び出す役。
# --------------------------------------------------------------------

from core.usecase.boundary.input_boundary import TodoInputBoundary
from core.usecase.boundary.dto import TodoInputData


class TodoController:
    """
    ControllerはUIイベントとビジネスロジックを繋ぐ薄い層。

    Viewから呼ばれ、InputBoundaryを通じてUseCaseを起動する。
    """

    def __init__(self, use_case: TodoInputBoundary):
        """
        [依存の持ち方]

        Controllerは「どのUseCaseのどの実装なのか」を知らない。
        代わりに、UseCaseが満たすべき契約（TodoInputBoundary）だけを受け取る。

        これにより、UseCaseの実際のクラス名や実装が変わっても、
        Controller側は変更不要になる。
        """
        self._use_case = use_case

    def add_todo(self, title: str) -> None:
        """
        [伝達処理の本体]

        Viewからの生の入力（titleの文字列）を、
        UseCaseに渡す公式な入力データ（TodoInputData）に変換し、
        UseCaseを実行する。

        ここではビジネスロジックを一切行わない。
        """
        # 1. Viewからの生データを、UseCaseが期待するInputDataに変換
        input_data = TodoInputData(title=title)

        # 2. InputBoundaryを通じて、UseCaseの実行を依頼
        self._use_case.execute(input_data)
```

---

## 🧪 ユニットテスト例（Controllerの責務確認）

```python
# --------------------------------------------------------------------
# File: tests/unit/test_todo_controller.py
# Layer: Test（Controllerの単体テスト）
#
# 目的:
#   - Controllerが生の入力文字列をTodoInputDataに変換しているか
#   - ControllerがUseCase(InputBoundary)を正しく呼び出しているか
# --------------------------------------------------------------------

import unittest
from core.usecase.boundary.input_boundary import TodoInputBoundary
from core.usecase.boundary.dto import TodoInputData
from interface_adapters.controllers.todo_controller import TodoController


# フェイクUseCase: Controllerが正しく呼び出すか確認するためのテストダブル
class FakeTodoUseCase(TodoInputBoundary):
    def __init__(self):
        self.last_input: TodoInputData | None = None

    def execute(self, input_data: TodoInputData) -> None:
        self.last_input = input_data


class TestTodoController(unittest.TestCase):
    def test_controller_calls_use_case_with_input_data(self):
        """
        ControllerがInputDataを生成し、UseCaseに渡すことを確認する。
        """
        fake_use_case = FakeTodoUseCase()
        controller = TodoController(fake_use_case)

        controller.add_todo("テストタスク")

        # UseCaseに正しいデータが渡されたか確認
        self.assertIsNotNone(fake_use_case.last_input)
        self.assertEqual(fake_use_case.last_input.title, "テストタスク")


if __name__ == "__main__":
    unittest.main()
```

💡 **テストの意図**

* Controllerは **TodoInputData を適切に組み立てているか？**
* Controllerは UseCase（＝TodoInputBoundaryの実装）を呼び出したか？
* Controller自身がビジネスロジックを勝手にやっていないか？（やってたらNG）

---

## 🛡 このクラスの鉄則

> 入力を整理し、然るべき場所に渡せ。

* Controllerはビジネスロジックを持たない
* ControllerはUIとUseCaseの間に立つ「緩衝材」として薄く保つ
* Controllerがあることで、UIとUseCaseが直接依存しなくなる（＝疎結合）

---

## 🔍 補足：Controllerの重要性

* **依存性逆転を守る役割の一部**

  * ControllerはUseCaseを呼ぶが、UseCase側はControllerを知らない
  * つまり依存は一方向（外→内）になっている
  * UseCaseがUIフレームワークの事情を一切知らない状態を保てる

* **テスト容易性**

  * ControllerはフェイクUseCaseを渡して簡単に振る舞いを検証できる
  * WebフレームワークやCLI I/Oが絡む前にロジックをテストできる

* **UI差し替えの柔軟さ**

  * CLI用のControllerとWeb用のControllerを分けることもできる
  * それでも中のUseCaseは同じ（依存先はInputBoundaryで一定だから）

