# DDD-06 : 🟡 `interface_adapters/repositories.py` (リポジトリ実装)

`DDD-05` では `UseCase` が `IOrderRepository` という新しい「鍵穴（インターフェース）」を要求するようになりました。
この章では、その鍵穴に差し込むための「具体的な鍵（実装クラス）」を、この `Interface Adapters` レイヤーに作成します。

## 🎯 この章のゴール

  * `CA-05` で作成したリポジトリ実装に、`IOrderRepository` の実装を追加する。
  * `Order` 集約（`LineItem` を含む）を、インメモリ（辞書）で保存・取得する方法を学ぶ。
  * 依存関係のルール（「外側」が「内側」に依存する）が守られていることを `import` 文で再確認する。

-----

## 🔌 このファイルの役割：新しい契約を実装する「アダプター」

このファイルは、引き続き `Interface Adapters` レイヤーに属します。

`Use Cases` レイヤーが「注文（`Order` 集約）を保存するための、こういう形のポート（`IOrderRepository`）が必要です」と `DDD-04` で宣言しました。
このレイヤーは「はい、承知いたしました。そのポートにぴったり合う、この技術（インメモリDB）を使ったアダプターです」と、具体的な実装クラスを提供します。

-----

## 💻 ソースコードの詳細解説

### `InMemoryProductRepository`: 商品の保管庫（変更なし）

このクラスの役割は `CA-05` から変わりません。`IProductRepository` という契約書を忠実に実装し、商品データをインメモリの辞書で管理します。

### `InMemoryOrderRepository(IOrderRepository)`: 注文の保管庫（重要）

こちらが今回の重要な追加点です。このクラスは、`IOrderRepository` という新しい契約書を履行します。

  * `__init__(self)`: このクラスは、`Order` 集約を保存するための、専用のインメモリデータベース（`self._orders` という辞書）を内部に保持します。`Product` のDBとは完全に分離されています。
  * `find_by_id(...)` と `save(...)`:
    `IOrderRepository` という契約書で約束されたメソッドを具体的に実装します。
    `save` メソッドは、`UseCase` から渡された、ビジネスルールによって整合性が保証された `Order` 集約オブジェクトをそのまま受け取り、辞書に保存します。

-----

## 🏛️ このレイヤーの鉄則（再確認）

1.  契約書を必ず履行する: `InMemoryOrderRepository` は、`IOrderRepository` インターフェースを必ず継承し、そのすべてのメソッドを実装します。
2.  依存の矢印は必ず内側を向く: このファイルは、自分より内側の `domain` や `use_cases` に依存します。
3.  翻訳者であれ: このシンプルな例ではオブジェクトを直接保存していますが、本来であれば、このレイヤーはドメインオブジェクト（`Order` や `LineItem`）とデータベースのテーブル形式（`orders` テーブル、`line_items` テーブル）との間でデータを相互に変換する責任を持ちます。

-----

## 📄 `interface_adapters/repositories.py` の実装

（`CA-05` のファイルに `InMemoryOrderRepository` の実装を追加・更新します）

```python:interface_adapters/repositories.py
# 依存性のルール:
# このファイルは Interface Adapters レイヤーに属します。
# 自分より内側の domain レイヤーと use_cases レイヤーにのみ依存します。

from typing import List

# 「内側」の domain レイヤーからエンティティと集約をインポート
from domain.product import Product
from domain.order import Order

# 「内側」の use_cases レイヤーからインターフェース（契約書）をインポート
from use_cases.interfaces import IProductRepository, IOrderRepository

class InMemoryProductRepository(IProductRepository):
    """
    【Interface Adaptersレイヤー / Adapter】
    IProductRepositoryインターフェースの、インメモリDBによる具体的な実装。
    
    (このクラスの役割は、CA-05 から変更ありません)
    """
    def __init__(self):
        self._products: dict[str, Product] = {}
        print("インメモリ商品リポジトリが初期化されました。")

    def find_by_id(self, product_id: str) -> Product | None:
        """【実装】辞書から商品IDで検索して返す"""
        print(f"リポジトリ: 商品ID'{product_id}'を検索します。")
        return self._products.get(product_id)

    def save(self, product: Product):
        """【実装】辞書に商品データを保存（または更新）する"""
        print(f"リポジトリ: {product.name} を保存します。")
        self._products[product.product_id] = product

class InMemoryOrderRepository(IOrderRepository):
    """
    【Interface Adaptersレイヤー / Adapter】
    DDD-04で新しく定義された IOrderRepositoryインターフェースの、
    インメモリDBによる具体的な実装。
    
    Use Caseレイヤーで定義された「契約」を、ここで具体的に履行します。
    """
    def __init__(self):
        # 注文(Order)集約を保存するための、専用の辞書
        self._orders: dict[str, Order] = {}
        print("インメモリ注文リポジトリが初期化されました。")

    def find_by_id(self, order_id: str) -> Order | None:
        """【実装】辞書から注文IDで Order 集約を検索して返す"""
        print(f"リポジトリ: 注文ID'{order_id}'を検索します。")
        return self._orders.get(order_id)

    def save(self, order: Order):
        """
        【実装】Use Caseから渡された Order 集約を、そのまま辞書に保存します。
        
        もしこれが本物のDBなら、このメソッドの中で
        Orderオブジェクトの情報を分解し、ordersテーブルやline_itemsテーブルに
        書き込むといった、複雑な処理（トランザクション）が行われます。
        """
        print(f"リポジトリ: 注文ID'{order.order_id}' を保存します。")
        self._orders[order.order_id] = order
        
    def get_all(self) -> List[Order]:
        """【実装】保持しているすべての Order 集約をリストとして返す"""
        print("リポジトリ: すべての注文履歴を取得します。")
        return list(self._orders.values())
```