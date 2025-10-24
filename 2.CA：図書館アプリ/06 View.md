# 06 View

# 📺 View : `adapters/view.py`

UI層の最後にして、ユーザーが直接触れる部分、`ConsoleView`を解説します。

## 🎯 このクラスの役割

**`ConsoleView`は、ユーザーが直接目にし、操作するユーザーインターフェース（UI）そのもの**です。

このクラスの責務は、大きく分けて2つです。

1. **表示（Output）**: `Presenter`によって状態が更新された`ViewModel`のデータを、画面（今回はコンソール）に具体的に描画する。
2. **入力（Input）**: ユーザーからのキーボード入力などを受け付け、その操作を`Controller`に伝える。

`View`は、アプリケーションの他の部分がどうなっているかを知る必要はありません。ただ、与えられた設計図（`ViewModel`）通りに家を建て、住人（ユーザー）からの要望を管理人（`Controller`）に伝えるだけの、純粋な「窓口」です。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

## ✅ このクラスにあるべき処理

⭕️ **含めるべき処理の例:**

- `print()`や`input()`など、具体的なUI技術（コンソール入出力）を直接扱うコード。
- `ViewModel`のデータを読み取って、画面に表示するロジック。
- ユーザーの操作に応じて、`Controller`のメソッドを呼び出すこと。

❌ **含めてはいけない処理の例:**

- ビジネスロジック（`UseCase`の責務）。
- 表示用データの整形ロジック（`Presenter`の責務）。
- データベースや外部APIに関する知識。

## 💻 ソースコードの詳細解説

```python
# adapters/view.py

# ControllerとViewModelは同じAdapters層にいる仲間
from .controller import CheckOutBookController
from application.data_structures import BookViewModel

# -----------------------------------------------------------------------------
# View
# - クラス図の位置: View
# - 同心円図の位置: Adapters (外側の円)
# -----------------------------------------------------------------------------
class ConsoleView:
    """
    ユーザーとの直接的なやり取り（入力受付・画面表示）を担当する。
    このクラスは具体的なUI技術（今回はコンソール）に強く依存する。
    """
    def __init__(self, controller: CheckOutBookController, view_model: BookViewModel):
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
        アプリケーションのメインループ。
        ユーザーからの入力を受け付け、Controllerに伝達する。
        """
        try:
            book_id_str = input("貸し出す本のIDを入力してください: ")
            member_id_str = input("貸し出す会員のIDを入力してください: ")

            # 入力された文字列を数値に変換。ここで失敗すればValueErrorが発生する。
            book_id = int(book_id_str)
            member_id = int(member_id_str)

            # Controllerに処理を依頼する。Viewはここから先のビジネスを知らない。
            self._controller.check_out(book_id=book_id, member_id=member_id)

        except ValueError:
            # 入力が不正だった場合のエラーメッセージはView自身が直接ViewModelを更新する。
            # これはビジネスロジックではない、純粋なUIレベルのエラーだから。
            self._view_model.display_text = "エラー: IDは数字で入力してください。"
        except Exception as e:
            # UseCaseなど、より内側の層から例外がスローされた場合のエラーメッセージ。
            self._view_model.display_text = f"エラー: {e}"

        # 処理結果（成功またはエラー）を画面に表示する
        self.render()

    def render(self):
        """
        [表示処理]
        自身の持つViewModelの現在の状態を元に画面を描画する。
        """
        print("\\n--- 処理結果 ---")
        print(self._view_model.display_text)
        print("----------------")

```

- `__init__`メソッドは、命令を伝える相手である`_controller`と、表示内容の情報源である`_view_model`を保持します。
- `run`メソッドが、この`View`の具体的な**入力**処理です。Pythonの`input()`関数を使ってユーザーからの入力を受け取り、それを`_controller`に渡しています。また、入力が数字でなかった場合などの簡単なエラー処理も担当しています。
- `render`メソッドが、具体的な**表示**処理です。`_view_model`の`display_text`プロパティを`print()`関数でコンソールに出力する、という単純な作業だけを行っています。

## 💡 ユニットテストでViewの正しさを証明する

`View`のテストは少し工夫が必要です。実際の`input()`や`print()`を動かすのではなく、\*\*「ユーザーが特定の入力を行ったら、`Controller`が正しく呼ばれるか」\*\*を検証します。

```python
# tests/adapters/test_view.py の例
from unittest.mock import patch, MagicMock

def test_viewはユーザー入力をcontrollerに正しく伝える():
    # 1. Arrange (準備): ControllerとViewModelをモック（偽物）にする
    mock_controller = MagicMock()
    view_model = BookViewModel()

    # 2. Act (実行): テスト対象のViewを作成し、'input'関数を偽の入力で置き換えて実行
    #    'builtins.input'をpatchすることで、実際のキーボード入力を待たずに済む
    #    side_effectで、input()が呼ばれるたびに返す値をリストで指定できる
    view = ConsoleView(mock_controller, view_model)
    with patch('builtins.input', side_effect=['100', '200']):
        view.run()

    # 3. Assert (検証): モックのControllerが期待通りに呼び出されたか確認
    # 意図: 「ユーザーが '100', '200' と入力したら、controller.check_out(100, 200)が
    #       一回だけ呼ばれること」をテスト
    mock_controller.check_out.assert_called_once_with(book_id=100, member_id=200)

```

- **テストの意図**: このテストは、`View`の**入力受付と伝達**という責務に焦点を当てています。`patch`を使ってUIの具体的な動作（`input`）をシミュレートし、`View`がそのシミュレーションに応じて`Controller`を正しく呼び出すことだけを検証します。

## 🐍 PythonとC言語の比較（初心者の方へ）

- **Python (オブジェクト指向)**: `ConsoleView`という**クラス**が、UIの状態（`_view_model`）と振る舞い（`run`, `render`）をまとめて管理します。
- **C言語 (手続き型)**: 通常、`main`関数内の`while`ループがUIの主役になります。ループの中で`scanf`で入力を受け付け、`if`文で入力を判定し、対応するビジネスロジック関数を呼び出し、`printf`で結果を表示する、という処理がすべて一箇所に集まりがちです。`ncurses`のようなライブラリを使えば画面をリッチにできますが、責務の分離という点ではプログラマ自身が強く意識する必要があります。

## 🛡️ このクラスの鉄則

このクラスは、ロジックを持たず、単純な作業に徹します。

> 愚直であれ (Be Dumb)
> 
- `View`はビジネス上の判断を一切行いません。ただ、「ユーザーの操作をControllerに伝え、ViewModelを画面に表示する」という2つの単純な責務だけを果たします。
- このクラスを「おバカさん」に保つことで、UIのデザイン変更（例：コンソールからGUIへ）があったとしても、影響範囲をこの`View`と`Presenter`だけに限定することができます。ビジネスロジックである`UseCase`や`Entity`は、一切影響を受けません。