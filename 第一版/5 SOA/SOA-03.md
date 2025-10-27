# SOA-03

```markdown
## use_cases/interfaces.py (インターフェース)

外部サービスである「在庫管理サービス」の存在を定義した次は、私たちのアプリケーションの内側に視点を移します。最初に手をつけるべきは、その外部サービスとどう対話するかの**「契約書」を定義する、use_cases/interfaces.py**です。

---
### 1. このファイルの役割：更新された外部世界との「契約書」

このファイルは、引き続きUse Cases（ユースケース）レイヤーに属します。SOAへの移行に伴い、このファイルの役割はさらに重要になります。

以前は、永続化（データベース）という一つの種類の外部とのやり取りだけを定義していました。しかし今回、「在庫管理」という、自分たちでは管理しない、外部のビジネス能力と連携する必要が出てきました。

そのため、このファイルに新しい**「ポート（Port）」**を追加します。これは、Use Caseが在庫管理サービスに対して「こういう機能を提供してほしい」と要求する、抽象的な接続口です。

また、この変更に伴い、既存のインターフェースの責務も見直されます。`ProductRepository`が持っていた「在庫」に関する情報は、もはやこのサービスの関心事ではなくなったためです。

### 2. ソースコードの詳細解説

` ProductRepository(ABC)` : 責務が変化した商品保管庫
- 変更点: saveメソッドがなくなりました。なぜなら、商品の**状態（在庫数）はもはやこのサービスでは管理せず、外部の` InventoryService` が責任を持つからです。このリポジトリの役割は、商品の不変的な情報（名前、価格など）**を見つけ出すことに特化しました。

`OrderRepository(ABC)` : 注文集約の保管庫（変更なし）
- 「注文」というトランザクションデータは、引き続き私たちの「注文受付サービス」が責任を持つため、このインターフェースの役割は変わりません。

`InventoryServicePort(ABC)` : 新しい外部サービスへの接続口（新規追加）
- これが今回の核心部分です。この抽象基底クラスは、Use Caseが在庫管理サービスに要求する能力を定義します。

- `check_stock(...)` : 在庫を確認する能力。

- `reduce_stock(...)` : 在庫を削減する能力。

- `ProcessOrderUseCase` は、この`InventoryServicePort` という抽象的なポートにのみ依存します。そのポートの先に繋がっているのが、REST APIなのか、gRPCなのか、あるいは今回のサンプルのような直接的なクラス呼び出しなのかを、Use Caseは一切知る必要がありません。

### 3. このレイヤーの鉄則（再確認）

1. 内側の要求によって定義される: `InventoryServicePort`は、`ProcessOrderUseCase`が「在庫を確認・削減したい」と要求したからこそ存在します。

2. 務の分離: `ProductRepository`は商品のマスターデータを、`OrderRepository`は注文のトランザクションデータを、そして`InventoryServicePort`は外部の在庫状態を扱う、というように責務が明確に分離されました。

3. 依存の矢印は内側を向く: 変更ありません。このファイルは`domain`レイヤーにのみ依存します。

---
### use_cases/interfaces.py
``` Python
# 依存性のルール:
# このファイルはUse Casesレイヤーに属します。
# 自分より内側のdomainレイヤーにのみ依存します。

from abc import ABC, abstractmethod
from domain.product import Product
from domain.order import Order

class ProductRepository(ABC):
    """
    【Use Casesレイヤー / Port】
    Productエンティティの永続化を担当するオブジェクトが満たすべき契約（インターフェース）。
    SOA化に伴い、在庫（状態）の管理責任がなくなったため、
    商品の不変情報を見つける(find)責務に特化する。
    """
    @abstractmethod
    def find_by_id(self, product_id: str) -> Product | None:
        pass

    # 在庫は外部サービスが管理するため、saveの責務はなくなった
    # @abstractmethod
    # def save(self, product: Product):
    #     pass

class OrderRepository(ABC):
    """
    【Use Casesレイヤー / Port】
    Order集約の永続化を担当するオブジェクトが満たすべき契約（インターフェース）。
    この責務は変更なし。
    """
    @abstractmethod
    def find_by_id(self, order_id: str) -> Order | None:
        pass

    @abstractmethod
    def save(self, order: Order):
        pass

class InventoryServicePort(ABC):
    """
    【Use Casesレイヤー / Port】
    外部の在庫管理サービスと通信するためのポート（接続口）を定義する、新しいインターフェース。
    
    これにより、Use Caseは具体的な通信方法を知ることなく、
    「在庫を確認する」「在庫を削減する」というビジネス上の要求を実現できる。
    """
    @abstractmethod
    def check_stock(self, product_id: str, quantity: int) -> bool:
        pass

    @abstractmethod
    def reduce_stock(self, product_id: str, quantity: int):
        pass
```

```