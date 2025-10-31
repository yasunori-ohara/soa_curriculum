# 07 Presenter

# 🎨 Presenter
### `interface_adapters/presenters/todo_presenter.py` - `TodoPresenter`


## 🧭 このクラスの役割

`TodoPresenter` は、**ビジネスロジックの結果（`TodoOutputData`）を、画面表示に最適な形式（`TodoViewModel`）へと変換する**「翻訳家」または「アーティスト」です。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

`UseCase` が返すデータは、あくまでビジネス中心の「中立な結果」です。
それをユーザーにとってわかりやすいメッセージに整形したり、絵文字を入れたりといった**プレゼンテーション（表示）のロジック**はPresenterの仕事です。

これにより、UseCaseは「どう表示されるか」を一切気にすることなく、ビジネスロジックに集中できます。

---

## ✅ このクラスにあるべき処理

⭕️ 含めるべき処理の例

* `TodoOutputBoundary` インターフェースの実装
* `TodoOutputData` から `TodoViewModel` への変換
* 表示用テキストの組み立て（絵文字、文言、表現、言い回しなど）

❌ 含めてはいけない処理の例

* ビジネスルールの判断（それはUseCase側）
* `print()` など実際の描画処理（それはView側）
* RepositoryやControllerやDBの扱い（Presenterは知らない）

---

## 📁 ファイルの配置

Presenterは「インターフェースアダプタ層」に属します。
OutputBoundaryは「ユースケース層（core/usecase/boundary）」に属します。
PresenterはこのOutputBoundaryを実装します。

```
├─ core/
│   └─ usecase/
│       └─ boundary/
│           ├─ input_boundary.py      # Controllerが呼ぶ入口の契約
│           ├─ output_boundary.py     # Presenterが実装する出口の契約
│           └─ dto.py                 # TodoOutputData / TodoViewModel など
│
└─ interface_adapters/
    └─ presenters/
        └─ todo_presenter.py          # OutputBoundaryを実装する具体的Presenter
```

---

## 🔍 ソースコード

```python
# --------------------------------------------------------------------
# File: interface_adapters/presenters/todo_presenter.py
# Layer: Interface Adapters（Presenter）
#
# 目的:
#   - UseCaseの出力（TodoOutputData）を、Viewが扱いやすい形式（TodoViewModel）に整形する。
#   - 表示ロジックはここに集約し、Viewは「描画するだけ」にする。
#
# C言語の感覚に近い説明:
#   - クラスは「構造体＋関数」のようなもの。
#   - self は「thisポインタ」に相当し、現在のインスタンス（自分自身）を指す。
# --------------------------------------------------------------------

from core.usecase.boundary.output_boundary import TodoOutputBoundary
from core.usecase.boundary.dto import TodoOutputData, TodoViewModel


class TodoPresenter(TodoOutputBoundary):
    """
    OutputBoundary（出力境界インターフェース）を実装するクラス。
    UseCaseからの結果をViewModelに変換する責務を持つ。
    """

    def __init__(self, view_model: TodoViewModel):
        """
        [状態の共有]
        Viewと共有するViewModelを受け取り、自身のプロパティとして保持する。
        このViewModelを更新することが、このPresenterの仕事。
        """
        self._view_model = view_model

    def present(self, output_data: TodoOutputData) -> None:
        """
        [インターフェースの実装]
        'present' という抽象的な要求を、
        OutputDataからViewModelを更新するという具体的な処理に変換して実行する。
        """

        # ビジネスの結果（OutputData）から、表示用のメッセージを生成する
        display_text = (
            f"✅ タスク『{output_data.title}』 (ID: {output_data.id}) を追加しました。"
        )

        # Viewと共有しているViewModelの状態を更新する
        self._view_model.display_text = display_text
```

---

## 🧪 ユニットテスト例（Presenterの責務確認）

```python
# --------------------------------------------------------------------
# File: tests/unit/test_todo_presenter.py
# Layer: Test（Presenterの単体テスト）
# --------------------------------------------------------------------

import unittest
from core.usecase.boundary.dto import TodoOutputData, TodoViewModel
from interface_adapters.presenters.todo_presenter import TodoPresenter


class TestPresenter(unittest.TestCase):
    def test_presenter_updates_view_model(self):
        """PresenterがOutputDataをViewModelに正しく反映することを確認"""
        vm = TodoViewModel()
        presenter = TodoPresenter(vm)

        # UseCaseから渡されたOutputDataをPresenterに渡す
        output_data = TodoOutputData(id=1, title="テストタスク")
        presenter.present(output_data)

        # ViewModelが更新されていることを確認
        self.assertIn("テストタスク", vm.display_text)
        self.assertTrue(vm.display_text.startswith("✅"))

if __name__ == "__main__":
    unittest.main()
```

💡 テストの意図

* Presenterは **TodoOutputData → TodoViewModel** の変換責務を果たしているか
* ビジネスルールや入出力制御の責務を持っていないことを確認できるか

---

## 🛡 鉄則

> 表示のために翻訳せよ、ただしビジネスを語るな。

* Presenterはビジネス判断をしない
* 表示メッセージの変更はPresenterだけで完結する（UseCaseに影響しない）
* Viewの実装（CLIかWebかGUIか）を知らずに仕事をする

---

## 🔍 依存性逆転との関係

Presenterは、クリーンアーキテクチャの「依存は内側へ」のルールを守るためのキーマンです。

* 悪い例：UseCaseが直接printする

  * → UseCaseが「CLI表示」という具体UIに縛られる
  * → Webに変えたいときにUseCaseを書き換えなきゃいけない

* 良い例：UseCaseは `TodoOutputBoundary.present(…)` を呼ぶだけ

  * → 実体はPresenterに差し替えるだけ
  * → CLIでもWebでもGUIでも、UseCaseは無変更で再利用できる

Presenterは、**ビジネスロジックをUIの都合から守る絶縁体**です。

