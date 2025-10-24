# 06 Presenter

# 🎨 Presenter : `interface_adapters/presenter.py` - `TodoPresenter`

---

## 🧭 このクラスの役割

`TodoPresenter` は、**ビジネスロジックの結果（`OutputData`）を、画面表示に最適な形式（`ViewModel`）へと変換する**「翻訳家」または「アーティスト」です。

![クリーンアーキテクチャ](https://www.notion.so../%E3%82%AF%E3%83%AA%E3%83%BC%E3%83%B3%E3%82%A2%E3%83%BC%E3%82%AD%E3%83%86%E3%82%AF%E3%83%81%E3%83%A3.png)

`Use Case`が返すデータは、あくまでビジネス中心の純粋なデータです。

それを、ユーザーにとって分かりやすいメッセージに整形したり、特定の色や書式を加えたりといった、**プレゼンテーション（表示）に関わるロジック**に責任を持ちます。

これにより、`Use Case`は「どう表示されるか」を一切気にすることなく、ビジネスロジックに集中できます。

---

## ✅ このクラスにあるべき処理

⭕️ **含めるべき処理の例**

- `TodoOutputBoundary`インターフェースの実装
- `OutputData`から`ViewModel`へのデータ変換
- 表示用メッセージの組み立て（例：絵文字、フォーマット）

❌ **含めてはいけない処理の例**

- ビジネスルールの判断（Use Caseの責務）
- `print()`などの画面描画処理（Viewの責務）
- ControllerやRepositoryに関する知識

---

## 🔍 ソースコード（コメント充実）

```python
# --------------------------------------------------------------------
# File: interface_adapters/presenter.py
# Layer: Interface Adapters（Presenter）
#
# 目的:
#   - Use Caseの出力（OutputData）を、Viewが扱いやすい形式（ViewModel）に整形する。
#   - 表示ロジックはここに集約し、Viewは描画に専念できるようにする。
#
# C言語の感覚に近い説明:
#   - クラスは「構造体＋関数」のようなもの。
#   - self は「thisポインタ」に相当し、現在のインスタンス（自分自身）を指す。
# --------------------------------------------------------------------

from boundaries import TodoOutputBoundary
from data_structures import TodoOutputData, TodoViewModel

class TodoPresenter(TodoOutputBoundary):
    """
    <I> Output Boundary（契約）を実装するクラス。
    Use Caseからの結果をViewModelに変換する責務を持つ。
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
        'present'という抽象的な要求を、OutputDataからViewModelを
        更新するという具体的な処理に変換して実行する。
        """
        # ビジネスの結果（OutputData）から、表示用のメッセージを生成する
        display_text = f"✅ タスク'{output_data.title}' (ID: {output_data.id}) を追加しました。"

        # Viewと共有しているViewModelの状態を更新する
        self._view_model.display_text = display_text

```

---

## 🧪 ユニットテスト例（Presenterの責務確認）

```python
# --------------------------------------------------------------------
# File: tests/test_presenter.py
# Layer: Test（Presenterの単体テスト）
# --------------------------------------------------------------------

import unittest
from data_structures import TodoOutputData, TodoViewModel
from interface_adapters.presenter import TodoPresenter

class TestPresenter(unittest.TestCase):
    def test_presenter_updates_view_model(self):
        """PresenterがOutputDataをViewModelに正しく反映することを確認"""
        vm = TodoViewModel()
        presenter = TodoPresenter(vm)

        # Use Caseから渡されたOutputDataをPresenterに渡す
        output_data = TodoOutputData(id=1, title="テストタスク")
        presenter.present(output_data)

        # ViewModelが更新されていることを確認
        self.assertIn("テストタスク", vm.display_text)
        self.assertTrue(vm.display_text.startswith("✅"))

if __name__ == "__main__":
    unittest.main()

```

💡 **テストの意図**

- Presenterは **OutputData → ViewModel** の変換責務を果たしているか
- ビジネスロジックやUI描画の責務を持っていないことを確認

---

## 🛡 鉄則

> 表示のために翻訳せよ、ただしビジネスを語るな。
> 
- Presenterはビジネス判断を行わない
- 表示メッセージの変更はPresenterだけで完結する（Use Caseに影響しない）

---

## 🔍 補足：依存性逆転の役割

Presenterの最も重要な役割は、Use CaseがViewに依存するという、あってはならない依存関係を**逆転**させることです。

- **もしPresenterがなければ？**
    
    Use Caseが直接Viewを操作 → ビジネスロジックがUI技術に汚染される
    
- **Presenterがあると？**
    
    Use Caseは抽象的な `OutputBoundary` に結果を通知するだけ → UIの詳細から絶縁
    

つまり、Presenterは**「絶縁体」**として、UIとビジネスロジックを切り離す極めて重要な役割を果たします。

---

次は Controller の分離版に進めましょうか？それとも View（CLI）を分離した `view_console.py` に進めますか？