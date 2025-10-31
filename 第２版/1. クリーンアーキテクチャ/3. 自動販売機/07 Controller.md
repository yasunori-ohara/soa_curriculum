# 🛂 Controller : adapters/controller.py

UI層の「交通整理員」、`VendingMachineController`について解説します。

## 🎯 このクラスの役割

`VendingMachineController`は、`View`からのユーザー入力を受け取り、それを`UseCase`が理解できる形式（`InputData`）に整えて「伝達」する交通整理員です。

`View`はユーザーが何をしたか（例：「A1」ボタンが押された）を知っていますが、それがビジネス的に何を意味するのかは知りません。`Controller`は、そのUIイベントを、「スロットID:A1の商品を選択せよ」というビジネス上のリクエストに変換し、適切な`UseCase`の窓口に渡すことが唯一の責務です。

`View`という「受付」と、`UseCase`という「専門部署」の間に立つ、薄いながらも重要な仲介役です。

## ✅ このクラスにあるべき処理

⭕️ 含めるべき処理の例:

  * `View`からのイベント（メソッド呼び出し）を受け取ること。
  * `View`から渡された生データを、`InputData`オブジェクトに変換すること。
  * `InputBoundary`インターフェースを通じて、適切な`UseCase`を呼び出すこと。
  * 複数の`UseCase`を扱うこと。（例：`select_item`だけでなく、`insert_coin`や`return_change`といったメソッドもこのクラスが持つことになります）

❌ 含めてはいけない処理の例:

  * ビジネスロジックそのもの（`UseCase`の責務）。
  * 画面表示の準備（`Presenter`の責務）。
  * 画面を直接描画する処理（`View`の責務）。
  * データベースやハードウェアに関する知識。

## 💻 ソースコードの詳細解説

このコントローラーは、複数のユーザー操作（コイン投入、商品選択など）に対応するため、複数の`UseCase`に依存します。

```python
# adapters/controller.py

# 内側の世界の「境界」と「データ構造」「Entity」にのみ依存する
from application.boundaries import (
    SelectItemInputBoundary, InsertCoinInputBoundary, ReturnChangeInputBoundary
)
from application.data_structures import SelectItemInputData, InsertCoinInputData
from domain.entities import PaymentManager

# -----------------------------------------------------------------------------
# Controller
# - クラス図の位置: Controller
# - 同心円図の位置: Adapters (外側の円)
# -----------------------------------------------------------------------------
class VendingMachineController:
    """ControllerはUIイベントとビジネスロジックを繋ぐ薄い層。"""
    def __init__(self,
                 select_item_use_case: SelectItemInputBoundary,
                 insert_coin_use_case: InsertCoinInputBoundary,
                 return_change_use_case: ReturnChangeInputBoundary):
        """
        [依存先の定義]
        呼び出し先である複数のUseCaseの具体的な実装クラスではなく、
        その「窓口」であるInputBoundaryインターフェースにのみ依存する。
        """
        self._select_item_use_case = select_item_use_case
        self._insert_coin_use_case = insert_coin_use_case
        self._return_change_use_case = return_change_use_case

    def select_item(self, slot_id: str, payment_manager: PaymentManager):
        """[伝達処理] Viewからの商品選択イベントをUseCaseに伝える"""
        # 1. Viewからの生データを、InputDataオブジェクトに変換
        input_data = SelectItemInputData(slot_id=slot_id)
        
        # 2. 変換したデータを添えて、インターフェースを通じてUseCaseの実行を依頼
        self._select_item_use_case.handle(input_data, payment_manager)

    def insert_coin(self, coin: int, payment_manager: PaymentManager):
        """[伝達処理] Viewからのコイン投入イベントをUseCaseに伝える"""
        input_data = InsertCoinInputData(coin=coin)
        self._insert_coin_use_case.handle(input_data, payment_manager)

    def return_change(self, payment_manager: PaymentManager):
        """[伝達処理] Viewからのお釣り要求イベントをUseCaseに伝える"""
        # このUseCaseは入力データが不要なため、InputDataは作成しない
        self._return_change_use_case.handle(payment_manager)
```

  * `__init__`: このコントローラーは3つの異なるシナリオ（商品選択、コイン投入、お釣り排出）を扱うため、それぞれに対応する3つの`UseCase`のインターフェースを受け取ります。
  * `select_item`メソッド: `View`から渡された`slot_id`を`SelectItemInputData`に変換（箱詰め）し、`UseCase`の`handle`メソッドを呼び出します。
  * `Controller`自身は一切「考えず」、ただ情報を右から左へ受け流しているだけ、という点が重要です。

## 💡 ユニットテストでControllerの正しさを証明する

Controllerのテストでは、「交通整理」が正しく行われているかを検証します。`View`から特定のメソッドが呼ばれたときに、対応する`UseCase`が正しいデータで呼び出されているかだけを確認します。

```python
# tests/adapters/test_controller.py の例

def test_select_itemが呼ばれると対応するUseCaseが起動する():
    # 1. Arrange (準備): すべてのUseCaseをモック（偽物）にする
    mock_select_uc = MockSelectItemUseCase()
    mock_insert_uc = MockInsertCoinUseCase()
    mock_return_uc = MockReturnChangeUseCase()
    
    # 2. Act (実行)
    controller = VendingMachineController(
        mock_select_uc, mock_insert_uc, mock_return_uc
    )
    # Viewから select_item が呼ばれたと仮定
    controller.select_item(slot_id="A1", payment_manager=PaymentManager())

    # 3. Assert (検証)
    # 意図: 「select_itemが呼ばれたら、SelectItemUseCaseだけが起動し、
    #       他のUseCaseは呼ばれていないこと」をテスト
    assert mock_select_uc.handle_was_called is True
    assert mock_select_uc.received_input_data.slot_id == "A1"
    
    assert mock_insert_uc.handle_was_called is False
    assert mock_return_uc.handle_was_called is False
```

## 🐍 PythonとC言語の比較（初心者の方へ）

  * Python (オブジェクト指向): `VendingMachineController`というクラスに、UIイベントと`UseCase`を仲介する責務をカプセル化します。
  * C言語 (手続き型): UIライブラリのコールバック関数が`Controller`の役割を担うことが多いでしょう。ボタンクリック時に呼ばれる関数の中で、入力値に応じて`if-else`文で処理を分岐させ、対応するビジネスロジック関数を直接呼び出す形になります。これではUIとビジネスロジックが密結合になりがちです。

## 🛡️ このクラスの鉄則

このクラスは、交通整理員として、決められたルールに従うだけです。

> *入力を整理し、然るべき場所に渡せ。 (Organize the input and pass it to the right place.)*

  * `Controller`はビジネスロジックを持ちません。UIとビジネスロジックの間に立ち、両者が直接会話しないようにする緩衝材の役割に徹します。
  * この層が薄く保たれているおかげで、`UseCase`は`View`を知らず、`View`は`UseCase`を知らない、というクリーンな分離が実現できます。