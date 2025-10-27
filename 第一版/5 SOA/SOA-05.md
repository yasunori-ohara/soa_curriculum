# SOA-05

```markdown
## `use_cases/process_order.py` (ユースケース)

`use_cases/interfaces.py`で外部サービスへの「接続口（ポート）」を定義した次は、そのポートを実際に利用する側である、**`use_cases/process_order.py`**の修正に進みます。

---
### 1. このファイルの役割：外部サービスを指揮する「コーディネーター」

このファイルは、引き続きUse Cases（ユースケース）レイヤーの核心です。SOAへの移行に伴い、このクラスの責務はさらに洗練されます。

DDDのリファクタリングによって、このクラスはビジネスロジックを`Order`集約に委譲する「指揮者」になりました。
今回はそれに加え、「在庫管理」という、自分たちのアプリケーションの関心事の外にある処理を、新しく定義した`InventoryServicePort`を通じて外部の専門家（在庫管理サービス）に委譲するようになります。

### 2. ソースコードの詳細解説

`ProcessOrderUseCase` クラス
- `__init__(...)`: 依存性の注入（DI）が更新されています。このユースケースは、`Product`と`Order`のリポジトリに加え、新しく`InventoryServicePort`を必要とするようになりました。これにより、このユースケースが「在庫管理」という外部の能力に依存していることが、コンストラクタのシグネチャから明確に分かります。

- `execute(...)`: このメソッドの処理の流れが、SOAによる責務の分離を最もよく表しています。

  1. 準備: 変更ありません。`ProductRepository`を使って、注文に必要な商品の不変情報（名前、価格など）を取得します。

  2. 生成: 変更ありません。`Order`集約のインスタンスを生成します。

  3. 委譲 (在庫確認): ここが重要な変更点です。以前は`Order`集約が`product.check_stock()`を呼び出していましたが、在庫の責任はもはや`Product`エンティティにはありません。代わりに、`ProcessOrderUseCase`が**`inventory_service`（外部サービスへのポート）**を使って在庫を確認します。

  4. 委譲 (注文明細追加): `Order`集約に、在庫確認済みの商品と数量を渡して明細を追加させます。

  5. 委譲 (在庫削減): 注文が確定する直前に、再び**`inventory_service`**を呼び出し、外部サービスの在庫を実際に削減させます。

  6. 永続化: `Order`集約を`OrderRepository`に保存します。

### 3. このレイヤーの鉄則（SOA適用後）

1. 内側にのみ依存: 変更ありません。

2. 集約と外部サービスを調整する: このユースケースの責務は、`Order`集約のような内側のドメインモデルと、`InventoryServicePort`のような外部サービスへのポートの両方を指揮し、一つのビジネスフローとしてまとめるオーケストレーターとしての役割がより明確になりました。

3. トランザクションの境界: 外部サービス呼び出しを含むため、分散トランザクション（Sagaパターンなど）を考慮する必要が出てきますが、このカリキュラムではまず疎結合にすることのメリットに焦点を当てます。

---
### use_cases/process_order.py
``` Python
# 依存性のルール:
# このファイルはUse Casesレイヤーに属します。
# 自分より内側のdomainレイヤーと、同じレイヤーのinterfacesにのみ依存します。

import uuid
from dataclasses import dataclass
from domain.product import Product
from domain.order import Order
from .interfaces import ProductRepository, OrderRepository, InventoryServicePort

# --- DTO (Data Transfer Object) の定義 (変更なし) ---
@dataclass
class ProcessOrderInput:
    items: list[dict]

@dataclass
class ProcessOrderOutput:
    order_id: str
    total_price: int

class ProcessOrderUseCase:
    """
    【Use Casesレイヤー / Use Case】
    「注文を処理する」というアプリケーションのユースケース。
    
    SOA化に伴い、「在庫管理」という関心事を外部サービスに委譲する。
    そのための新しい依存(InventoryServicePort)が追加された。
    """
    def __init__(self,
                 product_repo: ProductRepository,
                 order_repo: OrderRepository,
                 inventory_service: InventoryServicePort): # ⬅️ 新しい依存を追加
        self.product_repo = product_repo
        self.order_repo = order_repo
        self.inventory_service = inventory_service # ⬅️ 新しい依存を保持

    def execute(self, order_input: ProcessOrderInput) -> ProcessOrderOutput:
        # 1. 準備：注文に必要なProductエンティティをすべて取得する
        products: dict[str, Product] = {}
        for item in order_input.items:
            product = self.product_repo.find_by_id(item["product_id"])
            if not product:
                raise ValueError(f"商品IDが見つかりません: {item['product_id']}")
            products[item["product_id"]] = product
        
        # 2. 委譲 (在庫確認): 在庫確認の責務を外部サービスに委譲する
        print("指揮者(UseCase): 在庫管理サービスに在庫の確認を依頼します。")
        for item in order_input.items:
            if not self.inventory_service.check_stock(item["product_id"], item["quantity"]):
                raise ValueError(f"在庫不足: {item['product_id']}")
        
        # 3. 生成：新しいOrder集約のインスタンスを生成する
        new_order_id = "order-" + str(uuid.uuid4())
        order = Order(order_id=new_order_id)

        # 4. 委譲 (注文明細追加): Order集約に注文明細を追加させる
        for item in order_input.items:
            product = products[item["product_id"]]
            # 在庫確認は終わっているので、ここでは純粋に明細を追加するだけ
            order.add_line_item(product.name, product.price, item["quantity"])
        
        # 5. 委譲 (在庫削減): 在庫削減の責務を外部サービスに委譲する
        print("指揮者(UseCase): 在庫管理サービスに在庫の削減を依頼します。")
        for item in order_input.items:
            self.inventory_service.reduce_stock(item["product_id"], item["quantity"])
        
        # 6. 永続化：完成したOrder集約をリポジトリに保存する
        self.order_repo.save(order)

        # 7. 結果を返す
        return ProcessOrderOutput(
            order_id=order.order_id,
            total_price=order.total_price
        )
```

```