# DDD-04 : 🟢 `use_cases/interfaces.py` (インターフェース)

`domain` レイヤーに新しい「宝物」（`Order` 集約）が加わったことで、この外部世界との「契約書」も更新する必要があります。

## 🎯 この章のゴール

  * `domain` レイヤーの変更（`Order` 集約の追加）に伴い、`use_cases` のインターフェースも変更（`IOrderRepository` の追加）が必要になることを理解する。
  * 「集約（`Order`）」を単位としてデータをやり取りする、新しいリポジトリインターフェースを定義する。

-----

## 🔌 このファイルの役割：更新された外部世界との「契約書 (ポート)」

このファイルは、引き続き `Use Cases` レイヤーに属し、内側のビジネスロジックが外部の永続化層（DBなど）と結ぶ\*\*抽象的な「契約書（インターフェース）」\*\*を定義します。

`DDD-03` で、私たちは「注文」という新しい、そして非常に重要なビジネス概念（集約）を定義しました。
したがって、`Use Cases` がその「注文」をデータベースなどに保存したり、後から見つけ出したりするためには、\*\*新しい専用の出入り口（ポート）\*\*が必要になります。

このファイルでは、既存の `IProductRepository` に加えて、新しく `IOrderRepository` という契約書を定義します。

-----

## 💻 ソースコードの詳細解説

### `IProductRepository(ABC)`: 商品の保管庫（変更なし）

このインターフェースの役割は `CA` の時と変わりません。`Product` エンティティを見つけたり、保存したりするための契約を定義しています。

### `IOrderRepository(ABC)`: 注文集約の保管庫（重要）

こちらが今回の重要な追加点です。この抽象基底クラスは、`Order` 集約を永続化するオブジェクトが満たすべき契約を定義します。

  * `find_by_id(...)`: 注文IDを受け取り、`Order` 集約全体（`LineItem` なども含む）を復元して返すことを約束します。
  * `save(...)`: `Order` 集約全体を受け取り、その状態（注文自体の情報と、それに含まれる全ての `LineItem`）を永続化することを約束します。

このインターフェースのおかげで、`Use Cases` は、注文データが単一のテーブルに保存されているのか、複数のテーブル（`Orders` と `LineItems`）にまたがって保存されているのかといった、外側の世界の具体的な実装を一切気にすることなく、「注文を保存してくれ」と依頼できるのです。

-----

## 🏛️ このレイヤーの鉄則（再確認）

  * **内側の要求によって定義される:**
    `IOrderRepository` は、`Use Cases` が「`Order` を保存したい」と要求したからこそ存在します。
  * **依存の矢印は内側を向く:**
    このファイルは、自分より内側の `domain` レイヤー（`Product`, `Order`）にのみ依存します。
  * **ドメインオブジェクト（集約）をやり取りする:**
    `save` メソッドが渡すのは、バラバラのデータではなく、ビジネスルールで守られた\*\*`Order` 集約そのもの\*\*です。これにより、データの整合性が保たれます。

-----

## 📄 `use_cases/interfaces.py`

（`CA-03` のファイルに `IOrderRepository` を追加・整備します）

```python:use_cases/interfaces.py
# 依存性のルール:
# このファイルは Use Cases レイヤーに属します。
# 自分より内側の domain レイヤーにのみ依存します。

from abc import ABC, abstractmethod
from typing import List

# 「内側」の domain レイヤーからエンティティと集約をインポート
from domain.product import Product
from domain.order import Order # ⬅️ DDD-03 で作成した Order 集約をインポート

class IProductRepository(ABC):
    """
    【Use Casesレイヤー / Port】
    Productエンティティの永続化を担当するオブジェクトが満たすべき契約。
    (このインターフェースは CA-03 から変更なし)
    """
    @abstractmethod
    def find_by_id(self, product_id: str) -> Product | None:
        """【契約】商品IDを元に、単一の商品エンティティを検索する"""
        pass
    
    @abstractmethod
    def save(self, product: Product):
        """【契約】商品エンティティの状態を永続化（保存・更新）する"""
        pass

class IOrderRepository(ABC):
    """
    【Use Casesレイヤー / Port】
    Order集約の永続化を担当するオブジェクトが満たすべき、新しい契約。
    
    これにより、Use Caseは注文データの具体的な保存方法（RDB, NoSQL, etc.）
    を知ることなく、「注文を保存する」という要求を実現できます。
    """
    @abstractmethod
    def find_by_id(self, order_id: str) -> Order | None:
        """【契約】注文IDを元に、Order集約全体を復元して返す"""
        pass

    @abstractmethod
    def save(self, order: Order):
        """【契約】Order集約全体（LineItemも含む）の状態を永続化する"""
        pass
    
    @abstractmethod
    def get_all(self) -> List[Order]:
        """【契約】（補足）すべての注文履歴（Order集約のリスト）を取得する"""
        pass

```