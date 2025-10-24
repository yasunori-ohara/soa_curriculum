# 02 Use Case Interactor

# 👨‍🏫 Use Case : application/use_cases/select_item.py

`Entity`という素晴らしい「部品」ができたので、次はそれらを指揮して具体的なシナリオを実現する、`UseCase`を解説します。

今回は「ユーザーが商品選択ボタンを押す」という、自動販売機の中核となるユースケースを見ていきましょう。

## 🎯 このクラスの役割

`SelectItemUseCase`は、自動販売機の「制御基板」のように、「商品を選択する」という一つの業務シナリオ（ユースケース）の実行に責任を持つクラスです。

`Entity`が持つ個別のルール（例：「在庫がなければ商品は排出できない」）を尊重しつつ、`Item`と`PaymentManager`という複数の`Entity`や、外部の機能（在庫DBや物理的なハードウェア）を協調させて、業務全体の流れを制御（オーケストレーション）します。

ここには、個々の`Entity`だけでは完結しない、アプリケーション固有のビジネスルールが実装されます。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

## ✅ このクラスにあるべき処理

⭕️ 含めるべき処理の例:

- *複数のEntityの指揮（オーケストレーション）*:
`ItemRepository`から商品を探し、`PaymentManager`に投入金額を確認する、といった複数のオブジェクトをまたいだ処理。
- *Entityをまたがるビジネスルール*:
「`PaymentManager`の投入金額が、`Item`の価格以上であること」を確認する、といった複数のEntityの状態に依存する判断。
- *ハードウェアへの指示（インターフェース経由）*:
「商品を排出しろ」「お釣りを返せ」といった、物理デバイスへの指示を、抽象的なインターフェースを通じて行う。

❌ 含めてはいけない処理の例:

- *Entityが持つべきルールの重複実装*:
「在庫があるか？」の最終判断は`item.dispense()`メソッドに任せるべきです。`UseCase`が`if item.stock > 0`のようなチェックを直接行うのは、責務の侵害になります。
- *物理的なハードウェアの制御方法*:
商品を排出するモーターをどのくらいの時間回転させるか、といった具体的な技術詳細。
- *UIやデータベース、フレームワークに関する知識やコード。*

## 💻 ソースコードの詳細解説

（※このコードは、後ほど作成する`boundaries.py`と`data_structures.py`に依存していることを前提とします。）

```python
# application/use_cases/select_item.py

# 内側の世界の「境界」「データ構造」「Entity」にのみ依存する
from application.boundaries import (
    SelectItemInputBoundary, SelectItemOutputBoundary,
    ItemDataAccessInterface, HardwareInterface
)
from application.data_structures import SelectItemInputData, SelectItemOutputData
from domain.entities import Item, PaymentManager

# -----------------------------------------------------------------------------
# Use Case
# - クラス図の位置: UseCase
# - 同心円図の位置: Use Cases (内側から2番目の円)
# -----------------------------------------------------------------------------
class SelectItemUseCase(SelectItemInputBoundary):
    """「商品を選択する」というユースケースの実装"""
    def __init__(self,
                 presenter: SelectItemOutputBoundary,
                 item_repository: ItemDataAccessInterface,
                 hardware: HardwareInterface):
        """
        [依存性の注入]
        Presenter, Repository, そしてHardwareも、すべて抽象的なインターフェースとして受け取る。
        このUseCaseは、物理デバイスの具体的な仕組みを一切知らない。
        """
        self._presenter = presenter
        self._item_repository = item_repository
        self._hardware = hardware

    def handle(self, input_data: SelectItemInputData, payment_manager: PaymentManager):
        """
        [ビジネスロジックの実行]
        商品を選択するための具体的な処理手順（レシピ）を記述する。
        PaymentManagerはトランザクションの状態を持つため、ここでは引数で受け取る設計とする。
        """
        # 手順1: 関連するEntityを取得する
        item = self._item_repository.find_by_slot_id(input_data.slot_id)
        if not item:
            raise ValueError("指定されたスロットに商品はありません。")

        # 手順2: Entityにビジネスルールを実行させ、購入処理を行う
        #    - 「在庫はあるか？」
        #    - 「お金は足りているか？」
        #    - お釣りを計算し、投入金額をリセットする
        #   これらの複雑なチェックと計算は、Entityがすべて担当してくれる。
        change = payment_manager.process_purchase(item)

        # 手順3: ハードウェア制御のオーケストレーション
        # a. 物理的な商品排出を指示（ハードウェアの責務）
        self._hardware.dispense_item(item.slot_id)

        # b. 物理的なお釣り排出を指示（ハードウェアの責務）
        if change > 0:
            self._hardware.return_change(change)

        # c. 在庫の更新（Entityの責務）
        item.dispense()

        # 手順4: 変更を永続化する
        self._item_repository.save(item)

        # 手順5: 結果を出力データ（OutputData）に変換し、Presenterに渡す
        output_data = SelectItemOutputData(
            item_name=item.name,
            change=change
        )
        self._presenter.present(output_data)

```

- `__init__`: このUseCaseは、物理デバイスを操作する必要があるため、新たに`HardwareInterface`という抽象的なインターフェースに依存しています。
- `handle`メソッドの引数: `PaymentManager`は、一連の購入トランザクション（コイン投入〜お釣り排出）の間だけ状態を保持すればよいため、ここではリポジトリ経由ではなく、直接引数で受け渡しする設計にしています。
- `payment_manager.process_purchase(item)`: `Entity`の章で行った改善がここで活きています。`UseCase`は価格を取り出す必要がなく、`item`オブジェクトをそのまま渡すだけで、`PaymentManager`が購入処理を実行してくれます。

## 💡 ユニットテストでUseCaseの正しさを証明する

UseCaseのテストでは、ハードウェアさえも「偽物」に差し替えます。これにより、物理デバイスがなくてもビジネスロジックを100%検証できます。

```python
# tests/application/use_cases/test_select_item.py の例

def test_お金が足りていれば商品とお釣りが正しく排出される():
    # 1. Arrange (準備): すべての依存先を偽物で用意
    fake_presenter = FakePresenter()
    fake_item_repo = FakeItemRepository(在庫がある商品)
    # ★ハードウェアも偽物！ メソッドが呼ばれたかを記録するスパイの役割を持つ
    fake_hardware = FakeHardwareAdapter()

    # 投入金額が十分にある状態のPaymentManagerを用意
    payment_manager = PaymentManager(current_amount=500)

    # 2. Act (実行)
    use_case = SelectItemUseCase(fake_presenter, fake_item_repo, fake_hardware)
    input_data = SelectItemInputData(slot_id="A1") # 120円のお茶
    use_case.handle(input_data, payment_manager)

    # 3. Assert (検証)
    # 意図: 「UseCaseは、正しい指示を各部品に出しているか？」をテスト
    # ✅ 偽物ハードウェアの商品排出メソッドが、正しいスロットIDで呼ばれたか？
    assert fake_hardware.dispensed_slot_id == "A1"
    # ✅ 偽物ハードウェアのお釣り排出メソッドが、正しいお釣り金額で呼ばれたか？
    assert fake_hardware.returned_change == 380
    # ✅ 在庫が正しく永続化されたか？
    assert fake_item_repo.find_by_slot_id("A1").stock == 9

```

## 🐍 PythonとC言語の比較（初心者の方へ）

- Python (オブジェクト指向): `self._hardware.dispense_item(item.slot_id)`のように、`UseCase`は`HardwareInterface`という\*\*抽象（契約）\*\*に依存します。具体的なモーターの制御方法を知らなくても、抽象的な「排出」という操作を指示できます。
- C言語 (手続き型): ハードウェアを直接制御するコードがビジネスロジックに混在しがちです。`PORTB |= (1 << PB3);`のように、特定のレジスタを直接操作するコードが、商品購入ロジックのど真ん中に現れることも少なくありません。これでは、ハードウェアの仕様変更がビジネスロジック全体に影響を及ぼしてしまいます。

## 🛡️ このクラスの鉄則

このクラスは、システムの「頭脳」として、各部品の役割を理解し、適切に指示を出します。

> ビジネスの流れを指揮せよ、ただし詳細には関わるな。 (Orchestrate the business flow, but delegate the details.)
> 
- このクラスは、自動販売機のアプリケーションとしての振る舞いを定義します。
- しかし、個々の部品（Entity, Hardware, DB）がどのようにその振る舞いを実現するのか、という技術的な詳細には一切関与しません。この責務の分離が、システムの柔軟性とテスト容易性を生み出します。