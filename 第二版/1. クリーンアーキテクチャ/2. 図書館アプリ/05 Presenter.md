# 🎨 Presenter : `adapters/presenter.py`

UIを構成するアダプター群の中から、まずは**Presenter**を解説します。これは`UseCase`と`View`の間に立つ、重要な「翻訳家」です。

## 🎯 このクラスの役割

`CheckOutBookPresenter`**は、`UseCase`からのビジネスロジックの実行結果（`OutputData`）を受け取り、それを画面表示に最適な形式（`ViewModel`）へと変換する**「翻訳家」**または**「プレゼンテーション専門家」です。

`UseCase`が返すのは、あくまでビジネス中心の純粋なデータです（例：`datetime.date`オブジェクト）。それを、ユーザーにとって分かりやすいメッセージ（例：「返却期限は2025年10月27日です」）に整形するのが`Presenter`の責務です。

これにより、`UseCase`は「結果がどのように表示されるか」を一切気にすることなく、ビジネスロジックに集中できます。

## ✅ このクラスにあるべき処理

⭕️ **含めるべき処理の例:**

  * `CheckOutBookOutputBoundary`インターフェースの実装。
  * `OutputData`から`ViewModel`へのデータ変換。
  * データのフォーマット（例：日付オブジェクトを人間が読みやすい文字列に変換する）。
  * ユーザーに表示するためのメッセージ全体の組み立て。

❌ **含めてはいけない処理の例:**

  * ビジネスルールの判断（`UseCase`の責務）。
  * `print()`文など、具体的な画面描画処理（`View`の責務）。
  * `Controller`や`DataAccess`に関する知識。

## 💻 ソースコードの詳細解説

```python
# adapters/presenter.py

# 内側の世界の「境界」と「データ構造」にのみ依存する
from application.boundaries import CheckOutBookOutputBoundary
from application.data_structures import CheckOutBookOutputData, BookViewModel

# -----------------------------------------------------------------------------
# Presenter
# - クラス図の位置: Presenter
# - 同心円図の位置: Adapters (外側の円)
# -----------------------------------------------------------------------------
class CheckOutBookPresenter(CheckOutBookOutputBoundary):
    """
    <I> Output Boundary（契約書）を実装するクラス。
    UseCaseからの出力を、UI表示用の形式に変換する責務を持つ。
    """
    def __init__(self, view_model: BookViewModel):
        """
        [状態の共有]
        Viewと共有するViewModelを受け取り、自身のプロパティとして保持する。
        このViewModelを更新することが、このPresenterの唯一の仕事。
        """
        self._view_model = view_model

    def present(self, output_data: CheckOutBookOutputData):
        """
        [インターフェースの実装]
        UseCaseからの'present'という抽象的な要求を、OutputDataからViewModelを
        更新するという具体的な処理に変換して実行する。
        """
        # --- ここがPresenterの主な仕事 ---
        # 1. データのフォーマット
        #    ビジネスの世界のデータ（dateオブジェクト）を、
        #    プレゼンテーションの世界のデータ（フォーマット済み文字列）に変換する。
        due_date_str = output_data.due_date.strftime('%Y年%m月%d日')

        # 2. 表示用メッセージの組み立て
        display_text = (
            f"貸出処理が完了しました。\n"
            f"  書籍: 『{output_data.book_title}』\n"
            f"  会員: {output_data.member_name}様\n"
            f"  返却期限: {due_date_str}"
        )
        # --- ここまで ---
        
        # 3. Viewと共有しているViewModelの状態を更新する
        #    これにより、Viewは新しい表示内容を知ることができる。
        self._view_model.display_text = display_text
```

  * `__init__`メソッドで`View`と共有する`ViewModel`を受け取ります。
  * `present`メソッドがこのクラスの核となる処理です。
    1.  `UseCase`から渡された`output_data`に含まれる`datetime.date`オブジェクトを、`strftime`を使って`YYYY年MM月DD日`という人間が読みやすい文字列にフォーマットしています。
    2.  本のタイトルや会員名、フォーマットした日付を組み合わせて、最終的に画面に表示されるリッチなメッセージを組み立てています。
    3.  最後に、共有されている`_view_model`のプロパティを更新することで、`View`に「表示すべき内容が変わった」ことを間接的に伝えています。

## 💡 ユニットテストでPresenterの正しさを証明する

Presenterのテストでは、**「翻訳」ロジックが正しいか**を検証します。`UseCase`から特定の`OutputData`を受け取ったときに、`ViewModel`が期待通りに更新されるかを確認します。

```python
# tests/adapters/test_presenter.py の例
from datetime import date

def test_presenterはOutputDataを正しくViewModelに変換する():
    # 1. Arrange (準備): Viewと共有されるViewModelと、UseCaseから渡される想定のOutputDataを用意
    view_model = BookViewModel()
    output_data = CheckOutBookOutputData(
        book_title="クリーンアーキテクチャ",
        member_name="テストユーザー",
        due_date=date(2025, 10, 27)
    )
    
    # 2. Act (実行): テスト対象のPresenterを作成し、実行する
    presenter = CheckOutBookPresenter(view_model)
    presenter.present(output_data)

    # 3. Assert (検証): ViewModelが期待通りの表示用テキストに更新されているか確認
    # 意図: 「日付が正しくフォーマットされ、メッセージ全体が期待通りに組み立てられているか？」をテスト
    expected_text = (
        "貸出処理が完了しました。\n"
        "  書籍: 『クリーンアーキテクチャ』\n"
        "  会員: テストユーザー様\n"
        "  返却期限: 2025年10月27日"
    )
    assert view_model.display_text == expected_text
```

  * **テストの意図**: このテストは、`Presenter`が持つ唯一のロジック（データフォーマットとメッセージ組み立て）に焦点を当てています。`UseCase`や`View`の具体的な実装は一切不要で、`Presenter`の責務だけを独立して検証できます。

## 🐍 PythonとC言語の比較（初心者の方へ）

  * **Python (オブジェクト指向)**: `CheckOutBookPresenter`という**クラス**に、データ変換とメッセージ組み立ての責務をカプセル化します。`View`との連携は、`ViewModel`という状態を共有するオブジェクトを介して行います。
  * **C言語 (手続き型)**: もしC言語で書くなら、`format_checkout_message(output_data)`のような、データ構造体を受け取って整形済みの文字列（`char*`）を返す**関数**として実装するでしょう。状態管理は、グローバル変数や、関数の呼び出し元が持つ変数へのポインタを渡すことで行うことになり、依存関係が複雑になりがちです。

## 🛡️ このクラスの鉄則

このクラスは、忠実な「翻訳家」に徹します。

> **表示のために翻訳せよ、ただしビジネスを語るな。 (Translate for presentation, but don't speak business.)**

  * `Presenter`はビジネスの判断を行いません。`UseCase`から渡された結果を、ただ表示用に美しく整えるだけです。
  * このクラスのおかげで、表示メッセージの文言変更やデザインの調整が、ビジネスロジックに一切影響を与えずに行えます。例えば、日付のフォーマットを`YYYY/MM/DD`に変えたくなった場合、変更するのはこのファイルの`strftime`の部分だけで済みます。