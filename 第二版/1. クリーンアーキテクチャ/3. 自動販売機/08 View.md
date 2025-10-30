# 📺 View : adapters/view.py

UI層の最後にして、ユーザーとの直接の接点となる`ConsoleView`について解説します。

## 🎯 このクラスの役割

`ConsoleView`は、ユーザーが直接目にし、操作するユーザーインターフェース（UI）そのものです。

このクラスの責務は、大きく分けて2つです。

1.  *表示（Output）*: `Presenter`によって準備された`ViewModel`のデータを、画面（今回はコンソール）に具体的に描画する。
2.  *入力（Input）*: ユーザーからのキーボード入力などを受け付け、その操作を`Controller`に伝える。

`View`は、アプリケーションの他の部分がどうなっているかを知る必要はありません。ただ、与えられた設計図（`ViewModel`）通りに家を建て、住人（ユーザー）からの要望を管理人（`Controller`）に伝えるだけの、純粋な「窓口」です。

## ✅ このクラスにあるべき処理

⭕️ 含めるべき処理の例:

  * `print()`や`input()`など、具体的なUI技術を直接扱うコード。
  * `ViewModel`のデータを読み取って、画面に表示するロジック。
  * ユーザーの操作に応じて、`Controller`のメソッドを呼び出すこと。

❌ 含めてはいけない処理の例:

  * ビジネスロジック（`UseCase`の責務）。
  * 表示用データの整形ロジック（`Presenter`の責務）。
  * データベースやハードウェアに関する知識。

## 💻 ソースコードの詳細解説

```python
# adapters/view.py

# ControllerとViewModelは同じAdapters層にいる仲間
from .controller import VendingMachineController
from application.data_structures import VendingMachineViewModel
from domain.entities import PaymentManager

# -----------------------------------------------------------------------------
# View
# - クラス図の位置: View
# - 同心円図の位置: Adapters (外側の円)
# -----------------------------------------------------------------------------
class ConsoleView:
    """ユーザーとの直接的なやり取り（入力受付・画面表示）を担当する。"""
    def __init__(self, controller: VendingMachineController, view_model: VendingMachineViewModel):
        """
        [依存先の定義]
        指示を出す相手であるControllerと、表示内容の元となるViewModelを保持する。
        """
        self._controller = controller
        self._view_model = view_model

    def run(self):
        """
        [入力処理]
        自動販売機のメインの操作ループを実行する。
        """
        # 取引の状態を管理するPaymentManagerを生成してループを開始
        payment_manager = PaymentManager()
        
        while True:
            print("\n--------------------")
            print(f"現在の投入金額: {payment_manager.current_amount}円")
            print("操作を選んでください: [c: コイン投入, s: 商品選択, r: お釣り銭返却, q: 終了]")
            action = input("> ").lower()

            try:
                if action == 'c':
                    coin_str = input("投入する硬貨（10, 50, 100, 500）: ")
                    # Viewは「コインが投入された」というイベントをControllerに伝える
                    self._controller.insert_coin(int(coin_str), payment_manager)

                elif action == 's':
                    slot_id = input("購入する商品のスロットID: ")
                    # Viewは「商品が選択された」というイベントをControllerに伝える
                    self._controller.select_item(slot_id, payment_manager)

                elif action == 'r':
                    # Viewは「お釣り銭返却が要求された」というイベントをControllerに伝える
                    self._controller.return_change(payment_manager)

                elif action == 'q':
                    print("ご利用ありがとうございました。")
                    break
                else:
                    self._view_model.display_text = "無効な操作です。"

            except ValueError:
                self._view_model.display_text = "エラー: 不正な入力です。"
            except Exception as e:
                self._view_model.display_text = f"予期せぬエラー: {e}"

            # 処理結果を表示
            self.render()

    def render(self):
        """
        [表示処理]
        ViewModelの現在の状態を元に画面を描画する。
        """
        print(f"ディスプレイ: {self._view_model.display_text}")
```

  * `run()`: 自動販売機のメインループです。ユーザーの入力に応じて、対応する`Controller`のメソッドを呼び出します。`View`は、`Controller`の先に何があるか（`UseCase`や`Entity`）を一切知りません。
  * `render()`: `ViewModel`の`display_text`プロパティをコンソールに出力するだけの単純な作業です。`Presenter`が成功メッセージをセットした場合も、`View`自身が入力エラーメッセージをセットした場合も、このメソッドが最終的な表示を担当します。

## 💡 ユニットテストでViewの正しさを証明する

`View`のテストでは、実際の`input()`や`print()`を動かすのではなく、「ユーザーが特定の入力を行ったら、`Controller`が正しく呼ばれるか」を検証します。

```python
# tests/adapters/test_view.py の例
from unittest.mock import patch, MagicMock

def test_viewはユーザーのコイン投入入力をcontrollerに正しく伝える():
    # 1. Arrange (準備): ControllerとViewModelをモック（偽物）にする
    mock_controller = MagicMock()
    view_model = VendingMachineViewModel()

    # 2. Act (実行): 'input'関数を偽の入力で置き換えて実行
    view = ConsoleView(mock_controller, view_model)
    # ユーザーが'c', '100', 'q'と入力したと仮定する
    with patch('builtins.input', side_effect=['c', '100', 'q']):
        view.run()

    # 3. Assert (検証)
    # 意図: 「Controllerのinsert_coinが、100という値で呼ばれていること」をテスト
    mock_controller.insert_coin.assert_called_once()
    # 呼び出された際の2番目の引数（coin）が100であることを確認
    assert mock_controller.insert_coin.call_args[0][0] == 100
```

## 🐍 PythonとC言語の比較（初心者の方へ）

  * Python (オブジェクト指向): `ConsoleView`というクラスが、UIの状態（`_view_model`）と振る舞い（`run`, `render`）をまとめて管理します。
  * C言語 (手続き型): 通常、`main`関数内の`while`ループがUIの主役になります。ループの中で`scanf`で入力を受け付け、`if`文で入力を判定し、対応するビジネスロジック関数を呼び出し、`printf`で結果を表示する、という処理がすべて一箇所に集まりがちです。

## 🛡️ このクラスの鉄則

このクラスは、ロジックを持たず、単純な作業に徹します。

> *愚直であれ (Be Dumb)*

  * `View`はビジネス上の判断を一切行いません。ただ、「ユーザーの操作をControllerに伝え、ViewModelを画面に表示する」という2つの単純な責務だけを果たします。
  * このクラスを「おバカさん」に保つことで、UIのデザイン変更（例：コンソールからタッチパネルディスプレイへ）があったとしても、影響範囲をこの`View`と`Presenter`だけに限定することができます。

## ❓ Q\&A

### **Q. `View`にある `if action == 'c':` という分岐はロジックではないのですか？**

**A.** 素晴らしい質問です。これはロジックですが、`View`が持つべき**プレゼンテーションロジック**であり、`View`が持ってはいけない**ビジネスロジック**とは区別されます。

  * *ビジネスロジック (Business Logic)*

      * *役割*: アプリケーションの`本質的なルール`や`ドメイン知識`を扱うロジック。（例：「投入金額は商品の価格以上か？」）
      * *担当者*: `Entity` と `UseCase`

  * *プレゼンテーションロジック (Presentation Logic)*

      * *役割*: ユーザーとの`やり取り`や`表示`を制御するためのロジック。（例：「ユーザーが'c'と入力したら、コイン投入の処理に進む」）
      * *担当者*: `View` と `Presenter`

`View`にある`if`文は、「ユーザーの『c』というキー入力を、『コイン投入』というUI上のアクションに変換する」という、まさに`View`が担当すべきプレゼンテーションロジックなのです。テレビのリモコンが「音量＋」ボタンが押されたら「音量を上げる信号を送る」と判断するのと同じで、テレビ本体がどうやって音量を上げるか（ビジネスロジック）は知りません。この責務の分離こそが、クリーンアーキテクチャの核心です。