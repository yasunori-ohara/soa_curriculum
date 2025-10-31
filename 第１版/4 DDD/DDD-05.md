# DDD-05 : 🟢 `use_cases/process_order_use_case.py` (ユースケース)

`domain` レイヤーが（`DDD-03` で）リッチになったことで、この `Use Cases` レイヤーの役割がどのように変化したかにご注目ください。

## 🎯 この章のゴール

  * `UseCase` の役割が、`DDD` の適用により「ロジックの実行者」から「**調整役（コーディネーター）**」に変わることを理解する。
  * `UseCase` が、`Domain`（集約）にビジネスロジックの実行を「**委譲**」する様子を実装する。
  * `DTO`（Data Transfer Object）を導入し、`UseCase` の入出力を明確にする。
  * `UseCase` が「**トランザクションの境界**」として、変更されたすべての集約（`Product` と `Order`）を保存する責任を持つことを学ぶ。

-----

## 🎼 このファイルの役割：集約を指揮する「コーディネーター」

このファイルは、引き続き `Use Cases` レイヤーに属します。しかし、`DDD` を適用したことで、その役割は大きく変化しました。

`CA` での `UseCase` は、在庫チェックや在庫削減といったビジネスロジックを**自身で実行する**「働き者のマネージャー」でした。

`DDD` では、そのロジックの多くは `Order` 集約に移譲されました。その結果、この `UseCase` は、必要なエンティティ（`Product`）を見つけ出し、`Order` 集約に仕事（メソッド呼び出し）を**依頼**し、結果（変更された `Product` と新しい `Order`）をリポジトリに保存するという、より上位の\*\*「コーディネーター」または「指揮者」\*\*のような役割に変わりました。

-----

## 💻 ソースコードの詳細解説

### `ProcessOrderInput` と `ProcessOrderOutput` (DTOs)

これらは **DTO (Data Transfer Object)** と呼ばれる、単純なデータの入れ物です。`UseCase` の「入り口（Input）」と「出口（Output）」のデータ形式を `dict` ではなく `dataclass` で明確に定義することで、意図が分かりやすく、型安全なコードになります。

### `ProcessOrderUseCase` クラス

  * `__init__(...)`:
    依存性の注入（DI）が更新されています。このユースケースは、`Product` を探すための `IProductRepository` と、`Order` を保存するための `IOrderRepository` という、\*\*2つの異なるリポジトリ（インターフェース）\*\*に依存するようになりました。
  * `execute(...)`:
    このメソッドの処理の流れが、`DDD` による役割の変化を最もよく表しています。

<!-- end list -->

1.  **準備:** `IProductRepository` を使い、注文に必要な `Product` エンティティをすべて取得します。（料理人が調理を始める前に、すべての材料を揃えるのと同じです）
2.  **生成:** `Order` 集約のインスタンスを生成します。（この時点では、まだ空の注文伝票です）
3.  **委譲:** 注文明細を追加する処理は、`order.add_line_item(...)` を呼び出すことで、`Order` 集約自身に完全に**委譲**します。`UseCase` は、`add_line_item` の内部で在庫チェックが行われていることを知る必要はありません。
4.  **委譲:** 注文を確定する処理も、`order.confirm()` を呼び出して `Order` 集約に**委譲**します。（このメソッド内部で、`Product` の在庫が減らされます）
5.  **永続化:**
    **ここが重要です**。`UseCase` は「一つの仕事（トランザクション）」の責任者です。`order.confirm()` によって状態が変化した `Product` エンティティと、新しく生成された `Order` 集約の**両方**を、各リポジトリに渡して保存を依頼します。

-----

## 🏛️ このレイヤーの鉄則（DDD適用後）

1.  **内側にのみ依存:** これは変わりません。
2.  **集約を調整する:** `UseCase` は、ビジネスルールを自分で実装するのではなく、`Domain`（エンティティや集約）を調整し、指揮することに専念します。
3.  **トランザクションの境界:** 1つの `UseCase` の実行は、通常、データベースにおける1つの「作業単位（Unit of Work / トランザクション）」に対応します。`execute` メソッドが成功すればコミット、失敗（例外発生）すればロールバック、という流れを保証する責任を持ちます。

-----

## 📄 `use_cases/process_order_use_case.py`

（`CA-04` の `ProcessOrderUseCase` をリファクタリングします）

```python:use_cases/process_order_use_case.py
# 依存性のルール:
# このファイルは Use Cases レイヤーに属します。
# 自分より内側の domain レイヤーと、同じレイヤーの interfaces にのみ依存します。

import uuid
from dataclasses import dataclass # DTOを作るためにインポート

# 「内側」の domain レイヤーからインポート
from domain.product import Product
from domain.order import Order

# 「同じ」レイヤーの interfaces からインポート
from .interfaces import IProductRepository, IOrderRepository

# --- DTO (Data Transfer Object) の定義 ---
# UseCase の入出力を明確にするためのデータ構造

@dataclass
class ProcessOrderItemInput:
    """注文する商品1件分の入力データ"""
    product_id: str
    quantity: int

@dataclass
class ProcessOrderInput:
    """UseCase への入力データを格納するクラス"""
    customer_id: str
    items: list[ProcessOrderItemInput]

@dataclass
class ProcessOrderOutput:
    """UseCase からの出力データを格納するクラス"""
    order_id: str
    total_price: int
    confirmed_at: str # 仮: 注文確定日時など

class ProcessOrderUseCase:
    """
    【Use Casesレイヤー / Use Case】
    「注文を処理する」というアプリケーションのユースケース。
    
    DDDを適用したことで、このクラスの役割は大きく変化した。
    ロジックの多くを Order 集約に移譲し、自身は「調整役」に専念する。
    """
    def __init__(self, 
                 product_repo: IProductRepository, 
                 order_repo: IOrderRepository):
        """依存性の注入 (DI): 必要なリポジトリ（抽象）をすべて受け取る"""
        self.product_repo = product_repo
        self.order_repo = order_repo

    def execute(self, order_input: ProcessOrderInput) -> ProcessOrderOutput:
        
        # 1. 準備：注文に必要な Product エンティティをすべて取得する
        # (ドメインロジックではなく、ユースケースの準備処理)
        products: dict[str, Product] = {}
        for item in order_input.items:
            product = self.product_repo.find_by_id(item.product_id)
            if not product:
                raise ValueError(f"商品IDが見つかりません: {item.product_id}")
            products[item.product_id] = product
            
        # 2. 生成：新しい Order 集約のインスタンスを生成する
        # (Order IDの採番もユースケースの責務)
        new_order_id = "order_" + str(uuid.uuid4())
        order = Order(order_id=new_order_id, 
                      customer_id=order_input.customer_id)

        # 3. 委譲：ビジネスロジックの実行を Order 集約に「委譲」する
        # (ここで Order.add_line_item が product.check_stock を呼ぶ)
        for item in order_input.items:
            product = products[item.product_id]
            order.add_line_item(product, item.quantity)
        
        # 4. 委譲：注文の確定処理も Order 集約に「委譲」する
        # (ここで Order.confirm が product.reduce_stock を呼ぶ)
        order.confirm()
        
        # 5. 永続化：トランザクション内のすべての変更を保存する
        #    UseCase は「作業単位（Unit of Work）」の責任を持つ
        
        # 5a. 状態が変化した Product エンティティを保存する
        for item in order.line_items:
            # 在庫管理対象の商品のみ保存する
            if item.product.is_stock_managed():
                self.product_repo.save(item.product)
        
        # 5b. 新しく作成された Order 集約を保存する
        self.order_repo.save(order)

        # 6. 結果を返す (DTOを使用)
        return ProcessOrderOutput(
            order_id=order.order_id,
            total_price=order.total_price,
            confirmed_at=datetime.datetime.now().isoformat() # 仮
        )

```