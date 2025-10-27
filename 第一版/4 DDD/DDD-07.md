# DDD-07

```markdown
## `interface_adapters/repositories.py` (リポジトリ実装)

ここでは、Use Casesレイヤーで新しく定義されたOrderRepositoryという契約書を、具体的に実装します。

### 1. このファイルの役割：新しい契約を実装する「アダプター」

このファイルは、引き続きInterface Adapters（インターフェースアダプター）レイヤーに属します。

Use Casesレイヤーが「注文を保存するための、こういう形のポート（`OrderRepository`）が必要です」と宣言したのに対し、このレイヤーは「はい、承知いたしました。そのポートにぴったり合う、この技術（インメモリDB）を使ったアダプターです」と、具体的な実装クラスを提供します。

### 2. ソースコードの詳細解説

#### `InMemoryProductRepository`: 商品の保管庫（変更なし）
このクラスの役割は以前と変わりません。ProductRepositoryという契約書を忠実に実装し、商品データをインメモリの辞書で管理します。

#### `InMemoryOrderRepository(OrderRepository)`: 注文の保管庫（新規追加）
こちらが今回の重要な追加点です。このクラスは、`OrderRepository`という新しい契約書を履行します。

- `__init__(self)`: このクラスは、Order集約を保存するための、専用のインメモリデータベース（`self._orders`という辞書）を内部に保持します。`Product`のDBとは完全に分離されています。

- `find_by_id(...)` と `save(...)`:
`ProductRepository`と同様に、`OrderRepository`という契約書で約束されたメソッドを具体的に実装します。
saveメソッドは、ビジネスルールによって整合性が保証された`Order`集約オブジェクトをそのまま受け取り、辞書に保存します。

### 3. このレイヤーの鉄則（再確認）

1. 契約書を必ず履行する: `InMemoryOrderRepository`は、`OrderRepository`インターフェースを必ず継承し、そのすべてのメソッドを実装します。

2. 依存の矢印は必ず内側を向く: このファイルは、自分より内側の`domain`や`use_cases`に依存します。

3. 翻訳者であれ: このシンプルな例ではオブジェクトを直接保存していますが、本来であれば、このレイヤーはドメインオブジェクトとデータベースのテーブル形式との間でデータを相互に変換する責任を持ちます。

---
### `interface_adapters/repositories.py`

``` Python
# 依存性のルール:
# このファイルはInterface Adaptersレイヤーに属します。
# 自分より内側のdomainレイヤーとuse_casesレイヤーにのみ依存します。

from domain.product import Product, PhysicalProduct, DigitalProduct
from domain.order import Order
from use_cases.interfaces import ProductRepository, OrderRepository

class InMemoryProductRepository(ProductRepository):
    """
    【Interface Adaptersレイヤー / Adapter】
    ProductRepositoryインターフェースの、インメモリDBによる具体的な実装。
    
    このクラスの役割は、クリーンアーキテクチャの段階から変更ありません。
    """
    def __init__(self):
        self._products: dict[str, Product] = {}

    def find_by_id(self, product_id: str) -> Product | None:
        return self._products.get(product_id)

    def save(self, product: Product):
        self._products[product.product_id] = product

class InMemoryOrderRepository(OrderRepository):
    """
    【Interface Adaptersレイヤー / Adapter】
    新しく追加されたOrderRepositoryインターフェースの、インメモリDBによる具体的な実装。
    
    Use Caseレイヤーで定義された「契約」を、ここで具体的に履行します。
    """
    def __init__(self):
        self._orders: dict[str, Order] = {}

    def find_by_id(self, order_id: str) -> Order | None:
        return self._orders.get(order_id)

    def save(self, order: Order):
        """
        Use Caseから渡されたOrder集約を、そのまま辞書に保存します。
        
        もしこれが本物のデータベースであれば、このメソッドの中で
        Orderオブジェクトの情報を分解し、ordersテーブルやline_itemsテーブルに
        書き込むといった、複雑な処理が行われることになります。
        """
        self._orders[order.order_id] = order
```

```