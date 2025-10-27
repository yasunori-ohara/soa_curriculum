# DDD-05

```markdown
## `use_cases/interfaces.py` (インターフェース)

ドメインモデルに新しい「宝物」（Order集約）が加わったことで、この外部世界との「契約書」も更新する必要があります。

---

### 1. このファイルの役割：更新された外部世界との「契約書」

このファイルは、引き続きUse Cases（ユースケース）レイヤーに属し、内側のビジネスロジックが外部の永続化層などと結ぶ**抽象的な「契約書（インターフェース）」**を定義します。

前回の`domain/order.py`で、私たちは「注文」という新しい、そして非常に重要なビジネス概念（集約）を定義しました。
したがって、ユースケースがその「注文」をデータベースなどに保存したり、後から見つけ出したりするためには、**新しい専用の出入り口（ポート）**が必要になります。

このファイルでは、既存の`ProductRepository`に加えて、新しく`OrderRepository`という契約書を定義します。

### 2. ソースコードの詳細解説

`ProductRepository(ABC)`: 商品の保管庫（変更なし）
このインターフェースの役割は以前と変わりません。Productエンティティを見つけたり、保存したりするための契約を定義しています。

`OrderRepository(ABC)`: 注文集約の保管庫（新規追加）
こちらが今回の重要な追加点です。この抽象基底クラスは、Order集約を永続化するオブジェクトが満たすべき契約を定義します。

- `find_by_id(...)`: 注文IDを受け取り、Order集約全体を復元して返すことを約束します。

- `save(...)`: Order集約全体を受け取り、その状態（注文自体の情報と、それに含まれる全てのLineItem）を永続化することを約束します。

このインターフェースのおかげで、`ProcessOrderUseCase`は、注文データが単一のテーブルに保存されているのか、複数のテーブルにまたがって保存されているのかといった、外側の世界の具体的な実装を一切気にすることなく、「注文を保存してくれ」と依頼できるのです。

### 3. このレイヤーの鉄則（再確認）

- 内側の要求によって定義される: `OrderRepository`は、`ProcessOrderUseCase`が「注文を保存したい」と要求したからこそ存在します。

- 依存の矢印は内側を向く: このファイルは、自分より内側の`domain`レイヤー（`Product`, `Order`）にのみ依存します。

- ドメインオブジェクト（集約）をやり取りする: `save`メソッドが渡すのは、バラバラのデータではなく、ビジネスルールで守られた**`Order`集約そのもの**です。これにより、データの整合性が保たれます。

---

### use_cases/interfaces.py

``` Python
# 依存性のルール:
# このファイルはUse Casesレイヤーに属します。
# 自分より内側のdomainレイヤーにのみ依存します。

from abc import ABC, abstractmethod
from typing import List
from domain.product import Product
from domain.order import Order # ⬅️ 新しいOrder集約をインポート

class ProductRepository(ABC):
    """
    【Use Casesレイヤー / Port】
    Productエンティティの永続化を担当するオブジェクトが満たすべき契約（インターフェース）。
    この役割は以前のバージョンから変更ありません。
    """
    @abstractmethod
    def find_by_id(self, product_id: str) -> Product | None:
        pass

    @abstractmethod
    def save(self, product: Product):
        pass

class OrderRepository(ABC):
    """
    【Use Casesレイヤー / Port】
    Order集約の永続化を担当するオブジェクトが満たすべき、新しい契約（インターフェース）。
    
    これにより、Use Caseは注文データの具体的な保存方法を知ることなく、
    「注文を保存する」というビジネス上の要求を実現できます。
    """
    @abstractmethod
    def find_by_id(self, order_id: str) -> Order | None:
        pass

    @abstractmethod
    def save(self, order: Order):
        pass

```

```