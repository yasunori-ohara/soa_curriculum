# SOA-05: 🟢 `use_cases/process_order_use_case.py` (ユースケース)

`SOA-03` で外部サービス（在庫管理）への「接続口（`IInventoryServicePort`）」を定義し、`SOA-04` でその「具体的な変換プラグ（`InventoryServiceAdapter`）」を実装しました。

いよいよ、そのポート（接続口）を実際に利用する側である、`use_cases/process_order.py` の修正に進みます。

## 🎯 この章のゴール

  * `UseCase` が、新しい依存関係（`IInventoryServicePort`）を **DI (依存性の注入)** で受け取るように修正する。
  * `UseCase` の `execute` メソッドから、**在庫管理に関するロジック（内部での `check_stock`, `reduce_stock` 呼び出し）を削除**する。
  * 代わりに、`IInventoryServicePort` を通じて**外部の在庫管理サービス**に在庫確認と在庫削減を**委譲**するように修正する。
  * `UseCase` が、`Domain`（`Order` 集約）と外部サービスポートの両方を\*\*調整する「オーケストレーター」\*\*としての役割を担うことを理解する。

-----

## 🎼 このファイルの役割：外部サービスを指揮する「コーディネーター」

このファイルは、引き続き `Use Cases` レイヤーの核心です。`SOA` への移行に伴い、このクラスの責務はさらに洗練されます。

`DDD` のリファクタリングによって、このクラスはビジネスロジックを `Order` 集約に委譲する「指揮者」になりました。
今回はそれに加え、「**在庫管理**」という、自分たちのアプリケーション（注文受付サービス）の関心事の外にある処理を、新しく定義した `IInventoryServicePort` を通じて**外部の専門家（在庫管理サービス）に委譲**するようになります。

-----

## 💻 ソースコードの詳細解説

### `ProcessOrderUseCase` クラス

  * **`__init__(...)`**:
    依存性の注入（DI）が更新されています。このユースケースは、`IProductRepository` と `IOrderRepository` に加え、新しく `IInventoryServicePort` を必要とするようになりました。これにより、このユースケースが「在庫管理」という外部の能力に依存していることが、コンストラクタのシグネチャから明確に分かります。
  * **`execute(...)`**:
    このメソッドの処理の流れが、`SOA` による責務の分離を最もよく表しています。

<!-- end list -->

1.  **準備:** `IProductRepository` を使って、注文に必要な商品の不変情報（名前、価格など）を取得します。（変更なし）
2.  **委譲 (在庫確認):** 👈 **ここが重要な変更点①**
    以前は `Order.add_line_item` が内部で `product.check_stock` を呼び出していましたが、在庫の責任はもはや `Product` エンティティにはありません。代わりに、`ProcessOrderUseCase` が **`self.inventory_service.check_stock()`** を呼び出し、外部サービスに在庫の確認を依頼します。
3.  **生成:** `Order` 集約のインスタンスを生成します。（変更なし）
4.  **委譲 (注文明細追加):** `Order` 集約に、商品情報（在庫確認済み）と数量を渡して明細を追加させます。`Order` はもはや在庫チェックを行いません。
5.  **委譲 (在庫削減):** 👈 **ここが重要な変更点②**
    以前は `Order.confirm` が内部で `product.reduce_stock` を呼び出していましたが、代わりに `ProcessOrderUseCase` が **`self.inventory_service.reduce_stock()`** を呼び出し、外部サービスに在庫の削減（引当）を依頼します。
6.  **委譲 (注文確定):** `Order` 集約の `confirm` メソッドを呼び出します。（`Order` 内部の `product.reduce_stock` 呼び出しは削除されている必要があります - ※後述）
7.  **永続化:** `Order` 集約を `IOrderRepository` に保存します。（変更なし）
8.  **永続化 (Product):** 👈 **ここが重要な変更点③**
    `IProductRepository` の `save` は `SOA-03` で削除されたため、`Product` エンティティの状態（在庫）を `UseCase` が保存する必要はなくなりました。

-----

## 🏛️ このレイヤーの鉄則（SOA適用後）

1.  **内側にのみ依存:** 変更ありません。
2.  **集約と外部サービスポートを調整する:**
    このユースケースの責務は、`Order` 集約のような内側のドメインモデルと、`IInventoryServicePort` のような外部サービスへのポートの両方を指揮し、一つのビジネスフローとしてまとめる**オーケストレーター**としての役割がより明確になりました。
3.  **トランザクションの境界 (注意点):**
    外部サービス呼び出しを含むため、「注文は保存されたが、在庫削減に失敗した」といった不整合が発生する可能性があります。これを防ぐには分散トランザクション（例: Sagaパターン）などの高度な技術が必要になりますが、このカリキュラムではまず「疎結合」にすることのメリットに焦点を当てます。

-----

## 📄 `use_cases/process_order_use_case.py` の実装

(`DDD-05` の `UseCase` をリファクタリングします)

```python:use_cases/process_order_use_case.py
# 依存性のルール:
# このファイルは Use Cases レイヤーに属します。
# 自分より内側の domain レイヤーと、同じレイヤーの interfaces にのみ依存します。

import uuid
import datetime # Output DTO で使用
from dataclasses import dataclass

# 「内側」の domain レイヤーからインポート
from domain.product import Product
from domain.order import Order # Order 集約の confirm() から reduce_stock() 呼び出しを削除する必要あり

# 「同じ」レイヤーの interfaces からインポート
from .interfaces import (
    IProductRepository, 
    IOrderRepository, 
    IInventoryServicePort # ⬅️ 新しいポートを追加
)

# --- DTO (Data Transfer Object) の定義 (DDD-05 と同じ) ---
@dataclass
class ProcessOrderItemInput:
    product_id: str
    quantity: int

@dataclass
class ProcessOrderInput:
    customer_id: str
    items: list[ProcessOrderItemInput]

@dataclass
class ProcessOrderOutput:
    order_id: str
    total_price: int
    confirmed_at: str

class ProcessOrderUseCase:
    """
    【Use Casesレイヤー / Use Case】
    「注文を処理する」というアプリケーションのユースケース。
    
    SOA化に伴い、「在庫管理」という関心事を外部サービスに委譲する。
    そのための新しい依存(IInventoryServicePort)が追加された。
    """
    def __init__(self,
                 product_repo: IProductRepository,
                 order_repo: IOrderRepository,
                 inventory_service: IInventoryServicePort): # ⬅️ 新しい依存を追加
        """依存性の注入 (DI): 必要なリポジトリとポート（抽象）をすべて受け取る"""
        self.product_repo = product_repo
        self.order_repo = order_repo
        self.inventory_service = inventory_service # ⬅️ 新しい依存を保持
        print("🟢 Process Order UseCase initialized.")

    def execute(self, order_input: ProcessOrderInput) -> ProcessOrderOutput:
        
        # 1. 準備：注文に必要な Product エンティティ（基本情報のみ）を取得
        products: dict[str, Product] = {}
        for item in order_input.items:
            product = self.product_repo.find_by_id(item.product_id)
            if not product:
                raise ValueError(f"商品IDが見つかりません: {item.product_id}")
            products[item.product_id] = product
            
        # 2. 委譲 (在庫確認): 👈【変更点①】在庫確認を外部サービスに委譲
        print("🟢 UseCase: Requesting stock check from Inventory Service Port.")
        for item in order_input.items:
            # IInventoryServicePort の check_stock を呼び出す
            if not self.inventory_service.check_stock(item.product_id, item.quantity):
                raise ValueError(f"在庫不足: {products[item.product_id].name}")
        
        # 3. 生成：新しい Order 集約のインスタンスを生成
        new_order_id = "order_" + str(uuid.uuid4())
        order = Order(order_id=new_order_id, 
                      customer_id=order_input.customer_id)

        # 4. 委譲 (注文明細追加): Order 集約に注文明細を追加させる
        #    (Order.add_line_item は在庫チェックをしなくなった)
        for item in order_input.items:
            product = products[item.product_id]
            # Productオブジェクト自体を渡すことで、価格などの情報をOrderが利用できるようにする
            order.add_line_item(product, item.quantity) 
        
        # 5. 委譲 (在庫削減): 👈【変更点②】在庫削減を外部サービスに委譲
        print("🟢 UseCase: Requesting stock reduction from Inventory Service Port.")
        for item in order_input.items:
             # IInventoryServicePort の reduce_stock を呼び出す
            self.inventory_service.reduce_stock(item.product_id, item.quantity)
            
        # 6. 委譲 (注文確定): Order 集約に自身の状態を「確定済み」に遷移させる
        #    (Order.confirm は在庫削減をしなくなった)
        order.confirm() 
        
        # 7. 永続化 (Order): 完成した Order 集約をリポジトリに保存する
        self.order_repo.save(order)

        # 8. 永続化 (Product): 👈【変更点③】Product の保存は不要になった
        #    在庫状態は外部サービスが管理するため、このサービスでは Product の状態を保存しない。
        # for item in order.line_items:
        #     if item.product.is_stock_managed():
        #         self.product_repo.save(item.product) # 不要になった

        # 9. 結果を返す (DTOを使用)
        return ProcessOrderOutput(
            order_id=order.order_id,
            total_price=order.total_price,
            confirmed_at=datetime.datetime.now().isoformat() # 仮
        )

```

**(⚠️重要: この変更に伴う `domain/order.py` の修正)**
`SOA-05` で `UseCase` が在庫確認と削減の責任を持つようになったため、`DDD-03` で `Order` 集約に追加した以下のロジックを**削除**する必要があります。

  * `Order.add_line_item` 内の `product.check_stock()` 呼び出しと関連するエラーハンドリング。
  * `Order.confirm` 内の `item.product.reduce_stock()` 呼び出し。

これにより、`Order` 集約は純粋に「注文」の整合性を守る責任に集中し、`UseCase` が「在庫」という**外部ドメイン**との連携を担当するという、`SOA` の責務分担が明確になります。