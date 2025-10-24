# 06 Presenter

# 🎨 Presenter : adapters/presenter.py

UIを構成するアダプター群の中から、まずは`Presenter`を解説します。これは`UseCase`と`View`の間に立つ、重要な「ディスプレイメッセージ設計者」です。

## 🎯 このクラスの役割

`VendingMachinePresenter`は、`UseCase`からのビジネスロジックの実行結果（`OutputData`）を受け取り、それを画面表示に最適な形式（`ViewModel`）へと変換する役割を担います。

`UseCase`が返すのは、あくまでビジネス中心の純粋なデータです（例：商品名=`お茶`, お釣り=`30`）。それを、ユーザーにとって分かりやすい「『お茶』が出てきました。お釣りは30円です。」といった最終的なメッセージに整形するのが`Presenter`の責務です。

これにより、`UseCase`は「結果がどのように表示されるか」を一切気にすることなく、ビジネスロジックに集中できます。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

## ✅ このクラスにあるべき処理

⭕️ 含めるべき処理の例:

- `SelectItemOutputBoundary`インターフェースの実装。
- `OutputData`から`ViewModel`へのデータ変換。
- 表示用メッセージの組み立てロジック（例：お釣りが0円の場合はメッセージに含めない、といった条件分岐）。

❌ 含めてはいけない処理の例:

- ビジネスルールの判断（`UseCase`の責務）。
- `print()`文など、具体的な画面描画処理（`View`の責務）。
- `Controller`や`DataAccess`、`Hardware`に関する知識。

## 💻 ソースコードの詳細解説

```python
# adapters/presenter.py

# 内側の世界の「境界」と「データ構造」にのみ依存する
from application.boundaries import SelectItemOutputBoundary
from application.data_structures import SelectItemOutputData, VendingMachineViewModel

# -----------------------------------------------------------------------------
# Presenter
# - クラス図の位置: Presenter
# - 同心円図の位置: Adapters (外側の円)
# -----------------------------------------------------------------------------
class VendingMachinePresenter(SelectItemOutputBoundary):
    """
    <I> Output Boundary（契約書）を実装するクラス。
    UseCaseからの出力を、UI表示用の形式に変換する責務を持つ。
    """
    def __init__(self, view_model: VendingMachineViewModel):
        """
        [状態の共有]
        Viewと共有するViewModelを受け取り、自身のプロパティとして保持する。
        このViewModelを更新することが、このPresenterの唯一の仕事。
        """
        self._view_model = view_model

    def present(self, output_data: SelectItemOutputData):
        """
        [インターフェースの実装]
        'present'という抽象的な要求を、OutputDataからViewModelを
        更新するという具体的な処理に変換して実行する。
        """
        # --- ここがPresenterの主な仕事 ---
        # 1. 表示用メッセージの組み立て
        message = f"「{output_data.item_name}」が出てきました。"

        # 2. 条件に応じた表示ロジック（プレゼンテーションロジック）
        if output_data.change > 0:
            message += f" お釣りは{output_data.change}円です。"
        # --- ここまで ---

        # 3. Viewと共有しているViewModelの状態を更新する
        self._view_model.display_text = message

```

- `__init__`: `View`と共有する`ViewModel`を受け取ります。
- `present`: このクラスの核となる処理です。
    1. `UseCase`から渡された`output_data`を元に、基本となるメッセージを生成します。
    2. `if output_data.change > 0:` というプレゼンテーションロジックに基づき、お釣りが存在する場合のみ、お釣りに関するメッセージを追記しています。
    3. 最後に、完成したメッセージを共有`ViewModel`にセットすることで、`View`に「表示すべき内容が変わった」ことを間接的に伝えています。

## 💡 ユニットテストでPresenterの正しさを証明する

Presenterのテストでは、「翻訳」ロジックが正しいかを検証します。お釣りの有無など、条件によってメッセージが正しく変化するかを確認します。

```python
# tests/adapters/test_presenter.py の例

def test_お釣りがある場合_メッセージにお釣りが含まれる():
    # 1. Arrange (準備)
    view_model = VendingMachineViewModel()
    output_data = SelectItemOutputData(item_name="お茶", change=30)

    # 2. Act (実行)
    presenter = VendingMachinePresenter(view_model)
    presenter.present(output_data)

    # 3. Assert (検証)
    expected_text = "「お茶」が出てきました。 お釣りは30円です。"
    assert view_model.display_text == expected_text

def test_お釣りがゼロの場合_メッセージにお釣りは含まれない():
    # 1. Arrange (準備)
    view_model = VendingMachineViewModel()
    output_data = SelectItemOutputData(item_name="コーヒー", change=0)

    # 2. Act (実行)
    presenter = VendingMachinePresenter(view_model)
    presenter.present(output_data)

    # 3. Assert (検証)
    expected_text = "「コーヒー」が出てきました。"
    assert view_model.display_text == expected_text

```

## 🐍 PythonとC言語の比較（初心者の方へ）

- Python (オブジェクト指向): `VendingMachinePresenter`というクラスに、データ変換とメッセージ組み立ての責務をカプセル化します。
- C言語 (手続き型): `format_vending_machine_message(output_data)`のような、データ構造体を受け取って整形済みの文字列（`char*`）を返す関数として実装するでしょう。状態管理は、グローバル変数や、関数の呼び出し元が持つ変数へのポインタを渡すことで行うことになり、依存関係が複雑になりがちです。

## 🛡️ このクラスの鉄則

このクラスは、忠実な「翻訳家」に徹します。

> 表示のために翻訳せよ、ただしビジネスを語るな。 (Translate for presentation, but don't speak business.)
> 
- `Presenter`はビジネスの判断を行いません。`UseCase`から渡された結果を、ただ表示用に美しく整えるだけです。
- このクラスのおかげで、表示メッセージの文言変更（例：「出てきました」を「どうぞ！」に変える）やデザインの調整が、ビジネスロジックに一切影響を与えずに行えます。