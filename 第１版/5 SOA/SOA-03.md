# SOA-03 : 🟢 `use_cases/interfaces.py` (インターフェース)

`SOA-02` で連携先となる外部の「在庫管理サービス」の存在を定義しました。次は、推奨される開発順序に従い、私たちのアプリケーションの内側（`Use Cases` レイヤー）に視点を移します。

このステップでは、その外部サービスとどう対話するか（通信するか）の\*\*「契約書（インターフェース）」\*\*を、**私たちの都合に合わせて** `use_cases/interfaces.py` に定義します。

## 🎯 この章のゴール

  * `SOA` 化に伴い、外部サービスと連携するための新しい\*\*ポート（Port）\*\*として `InventoryServicePort` を定義する。
  * 既存の `ProductRepository` の**責務が変化**し、在庫管理（`save`）の責任を手放すことを理解する。
  * 依存性逆転の原則に基づき、`Use Cases` が外部サービスの「実装」ではなく、自ら定義した「抽象（インターフェース）」に依存するように設計する。

-----

## 🔌 このファイルの役割：更新された外部世界との「契約書 (ポート)」

このファイルは、引き続き `Use Cases` レイヤーに属します。`SOA` への移行に伴い、このファイルの役割はさらに重要になります。

以前は、永続化（データベース）という一つの種類の外部とのやり取りだけを定義していました。しかし今回、「在庫管理」という、自分たちでは管理しない、**外部のビジネス能力**と連携する必要が出てきました。

そのため、このファイルに新しい\*\*「ポート（Port）」\*\*を追加します。これは、`Use Case` が在庫管理サービスに対して「こういう機能（`check_stock`, `reduce_stock`）を提供してほしい」と要求する、**抽象的な接続口**です。(`OOP-09` の「コンセントの鍵穴」にあたります)。

また、この変更に伴い、既存のインターフェースの責務も見直されます。

-----

## 💻 ソースコードの詳細解説

### `IProductRepository(ABC)`: 責務が変化した商品保管庫

  * **変更点:** `save` メソッドがなくなりました (コメントアウト)。
  * **理由:** 商品の**状態（在庫数）はもはやこの「注文受付サービス」では管理せず、外部の `InventoryService` が責任を持つ**からです。このリポジトリの役割は、商品の不変的な情報（名前、価格など）を**見つけ出す (find)** ことに特化しました。

### `IOrderRepository(ABC)`: 注文集約の保管庫（変更なし）

  * 「注文」というトランザクションデータは、引き続き私たちの「注文受付サービス」が責任を持つため、このインターフェースの役割は変わりません。

### `IInventoryServicePort(ABC)`: 新しい外部サービスへの接続口（重要）

  * これが今回の核心部分です。この抽象基底クラスは、`Use Case` が在庫管理サービスに要求する能力（契約）を定義します。
  * `check_stock(...)`: 在庫を確認する能力。
  * `reduce_stock(...)`: 在庫を削減（引き当て）する能力。
  * `ProcessOrderUseCase` は、この `IInventoryServicePort` という**抽象的なポートにのみ依存します**。そのポートの先に繋がっているのが、`SOA-02` で作った模擬サービスなのか、実際の `REST API` なのか、あるいは `gRPC` なのかを、`Use Case` は一切知る必要がありません。

-----

## 🏛️ このレイヤーの鉄則（再確認）

1.  **内側の要求によって定義される:** `IInventoryServicePort` は、`ProcessOrderUseCase` が「在庫を確認・削減したい」と**要求したからこそ**存在します。外部の在庫管理サービスの API 仕様に合わせて直接定義するのではありません（依存性逆転）。
2.  **責務の分離:** `IProductRepository` は商品のマスターデータを、`IOrderRepository` は注文トランザクションデータを、そして `IInventoryServicePort` は外部の在庫状態を扱う、というように責務が明確に分離されました。
3.  **依存の矢印は内側を向く:** 変更ありません。このファイルは `domain` レイヤーにのみ依存します。

-----

## 📄 `use_cases/interfaces.py` の実装

(`DDD-04` のファイルに `IInventoryServicePort` を追加し、`IProductRepository` を修正します)

```python:use_cases/interfaces.py
# 依存性のルール:
# このファイルは Use Cases レイヤーに属します。
# 自分より内側の domain レイヤーにのみ依存します。

from abc import ABC, abstractmethod
from typing import List

# 「内側」の domain レイヤーからインポート
from domain.product import Product
from domain.order import Order

class IProductRepository(ABC):
    """
    【Use Casesレイヤー / Port】
    Productエンティティの永続化を担当するオブジェクトが満たすべき契約。
    SOA化に伴い、在庫（状態）の管理責任がなくなったため、
    商品の不変情報を見つける(find)責務に特化する。
    """
    @abstractmethod
    def find_by_id(self, product_id: str) -> Product | None:
        """【契約】商品IDを元に、商品の基本情報を持つエンティティを検索する"""
        pass

    # 在庫は外部サービスが管理するため、このサービスの ProductRepository は
    # 在庫を含む商品状態を保存(save)する責務を持たない。
    # @abstractmethod
    # def save(self, product: Product):
    #     pass

class IOrderRepository(ABC):
    """
    【Use Casesレイヤー / Port】
    Order集約の永続化を担当するオブジェクトが満たすべき契約。
    (この責務は DDD-04 から変更なし)
    """
    @abstractmethod
    def find_by_id(self, order_id: str) -> Order | None:
        pass

    @abstractmethod
    def save(self, order: Order):
        pass
    
    @abstractmethod
    def get_all(self) -> List[Order]:
        pass


class IInventoryServicePort(ABC):
    """
    【Use Casesレイヤー / Port】 (新規追加)
    外部の在庫管理サービスと通信するためのポート（接続口）を定義するインターフェース。
    
    これにより、Use Caseは具体的な通信方法(HTTP, gRPC, etc.)や
    外部サービスの内部実装を知ることなく、
    「在庫を確認する」「在庫を削減する」という要求を実現できる。
    """
    @abstractmethod
    def check_stock(self, product_id: str, quantity: int) -> bool:
        """【契約】指定された商品の在庫が十分かを確認する"""
        pass

    @abstractmethod
    def reduce_stock(self, product_id: str, quantity: int):
        """【契約】指定された商品の在庫を削減（引当）する"""
        pass

```

これで、「外部サービス（在庫管理）」と連携するための「理想の鍵穴」が `Use Cases` レイヤーに定義されました。
次のステップ（`SOA-04`）では、この鍵穴に合う「具体的な鍵（アダプター）」を `Interface Adapters` レイヤーに実装します。