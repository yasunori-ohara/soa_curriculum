# SOA-02 : 📦 `external_services/inventory_service.py` (外部サービス)

`SOA-01` で概説した推奨される開発順序に従い、まずは連携先となる外部サービス、すなわち模擬的な「在庫管理サービス」から実装を始めます。

## 🎯 この章のゴール

  * 外部サービスが、自分たちのアプリケーションとは **独立したコンポーネント** であることを理解する。
  * 外部サービスが提供する **契約（API/メソッド）** を定義する。
  * この模擬サービスが、後のステップで「注文受付サービス」から **呼び出される側** になることを認識する。

-----

## 🏢 このファイルの役割：独立した「在庫管理サービス」

この `external_services/inventory_service.py` ファイルは、私たちの「注文受付アプリケーション」とは **完全に独立した、別のビジネスコンポーネント** を表現しています。

これは、倉庫管理チームが開発・運用している、独立した「在庫管理サービス」だと考えてください 🏭。このサービスは、自分自身のデータベース（今回は `_products_stock` という辞書）を持ち、「在庫を確認する」「在庫を減らす」というビジネス能力を、明確なインターフェース（メソッド）を通じて外部に公開しています。

私たちの「注文受付サービス」（`use_cases` など）は、この独立したサービスを **利用者（クライアント）** として呼び出すことになります。

-----

## 💻 ソースコードの詳細解説

### `InventoryService` クラス

このクラスが、外部サービスそのものを模倣しています。現実のシステムでは、これは別のサーバーで動作しているWeb APIかもしれませんし、共有のメッセージキューかもしれません。

  * **`__init__(self)`**:
    このサービスが管理する、独自の在庫データベース（`_products_stock`）を初期化します。重要なのは、このデータが「注文受付サービス」が持つ `InMemoryProductRepository` のデータとは **完全に分離されている** 点です。
  * **`add_stock(...)`**:
    在庫を追加するための公開メソッド（契約）です。外部から在庫を初期化するために使います。
  * **`check_stock(...)`**:
    在庫が十分かを確認するための公開メソッド（契約）です。`True` か `False` を返します。「注文受付サービス」は、このメソッドを呼び出して注文可能か判断します。
  * **`reduce_stock(...)`**:
    在庫を減らす（引き当てる）ための公開メソッド（契約）です。在庫が不足している場合は例外を発生させます。「注文受付サービス」は、注文確定時にこのメソッドを呼び出します。
  * **`get_stock(...)`**:
    現在の在庫数を取得するための公開メソッド（契約）です。

-----

## 🏛️ このコンポーネントの鉄則

  * **自己完結している:** 在庫管理に関するすべてのデータとロジックは、このサービス内にカプセル化されています。
  * **独立している:** このサービスは、「注文受付サービス」の存在を **一切知りません**。ただ、不特定多数のクライアント（注文受付サービスを含む）からの要求に応えるだけです。
  * **明確な契約を持つ:** 外部のクライアントは、このサービスが公開しているメソッド（`check_stock` など）だけを頼りに連携します。内部の `_products_stock` が辞書なのか、巨大なデータベースなのかを知る必要はありません。

-----

## 📄 `external_services/inventory_service.py` の実装

```python:external_services/inventory_service.py
# 依存性のルール:
# このファイルは、私たちのアプリケーションとは独立した外部サービスを模倣しています。
# したがって、私たちのアプリケーションのどのファイル
# (domain, use_cases, interface_adapters, infrastructure) にも依存しません。
# Python標準ライブラリのみ使用可能です。

class InventoryService:
    """
    【外部サービス】
    独立した「在庫管理サービス」を模倣したクラス。
    
    このサービスは、自分自身の在庫データベースを持ち、
    「在庫確認」や「在庫削減」といったビジネス能力を、
    明確なインターフェース（メソッド）を通じて外部に提供する。
    """
    def __init__(self):
        # このサービスが専有する、独立した在庫データベース（辞書で模倣）
        self._products_stock: dict[str, int] = {}
        print("📦 Mock Inventory Service initialized.")

    def add_stock(self, product_id: str, quantity: int):
        """【契約/API】在庫を追加するためのインターフェース"""
        current_stock = self._products_stock.get(product_id, 0)
        self._products_stock[product_id] = current_stock + quantity
        print(f"📦 Inventory Added: {product_id}, New Stock: {self._products_stock[product_id]}")

    def check_stock(self, product_id: str, quantity: int) -> bool:
        """【契約/API】在庫が十分か確認するためのインターフェース"""
        stock = self._products_stock.get(product_id, 0)
        print(f"📦 Checking stock for {product_id}: Available={stock}, Required={quantity}")
        return stock >= quantity

    def reduce_stock(self, product_id: str, quantity: int):
        """【契約/API】在庫を削減（引当）するためのインターフェース"""
        print(f"📦 Attempting to reduce stock for {product_id} by {quantity}")
        if not self.check_stock(product_id, quantity):
            # 在庫不足の場合はエラー（現実のAPIはエラーコードなどを返す）
            raise ValueError(f"Insufficient stock for product {product_id}")
        
        self._products_stock[product_id] -= quantity
        print(f"📦 Stock Reduced: {product_id}, Remaining Stock: {self._products_stock[product_id]}")

    def get_stock(self, product_id: str) -> int:
        """【契約/API】現在の在庫数を取得するためのインターフェース"""
        stock = self._products_stock.get(product_id, 0)
        print(f"📦 Getting stock for {product_id}: {stock}")
        return stock

```

これで連携先の「仕様」が確定しました。
次のステップ（`SOA-03`）では、私たちのアプリケーション側（`use_cases`）で、この外部サービスに接続するための「理想の鍵穴（インターフェース）」を定義します。