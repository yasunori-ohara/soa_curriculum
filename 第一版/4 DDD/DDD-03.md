# DDD-03

## domain/product.py (エンティティ)

それでは、DDDを適用したプログラムの解説を、最も中心に位置する**domain/product.py**から始めます。

---

### 1. このファイルの役割：基本的なビジネスオブジェクト (Entities レイヤー)

このファイルは、クリーンアーキテクチャの時と同様、アプリケーションのEntities（エンティティ）レイヤーに属します。

今回のDDDリファクタリングにおいて、このファイル自体には大きな変更はありません。しかし、これから導入する、より複雑なビジネス概念である「注文（Order）」集約が、この`Product`エンティティを構成要素として利用するため、このファイルが安定した土台として存在していることを再確認することが非常に重要です。

`Product`は、私たちのECサイトというドメイン（ビジネス領域）における、最も基本的で独立した「モノ」の一つです。

### 2. ソースコードの詳細解説

`Product(ABC)`: 「商品」の契約書（インターフェース）
この抽象基底クラスは、「商品」という概念が持つべき基本的な能力を定義しています。この「契約」は、DDDを適用した後も変わりません。

`PhysicalProduct` と `DigitalProduct`: 契約の具体的な実装
これらも同様に、物理的な商品とダウンロード商品という、ビジネス上の具体的な「モノ」を表現しています。これらのクラスが持つビジネスルール（在庫を減らす、など）は、このオブジェクト自身の責任であり、他のオブジェクトが関知すべきではありません。

### 3. このレイヤーの鉄則（再確認）

- 自己完結している: `Product`エンティティは、自分自身の状態とルールにのみ責任を持ちます。「注文」や「顧客」といった、他のビジネス概念のことは知りません。
- 安定している: ここで定義されているのは、ビジネスの基本的な構成要素です。ビジネスのフローが変わっても、商品そのものの性質が変わらない限り、このファイルが変更されることはほとんどありません。

---

### domain/product.py

```
# 依存性のルール:
# このファイルはEntitiesレイヤーに属します。
# アプリケーションの最も中心に位置するため、他のどのレイヤーにも依存しません。

from abc import ABC, abstractmethod

class Product(ABC):
    """
    【Entitiesレイヤー / Entity】
    「商品」というビジネス上の概念を表す抽象基底クラス（インターフェース）。

    このクラスは、特定のアプリケーションの機能（ユースケース）に依存しない、
    商品そのものが持つべき普遍的な属性と振る舞いを定義します。
    """
    def __init__(self, product_id: str, name: str, price: int):
        self.product_id = product_id
        self.name = name
        self.price = price

    @abstractmethod
    def check_stock(self, quantity: int) -> bool:
        """在庫を確認する能力"""
        pass

    @abstractmethod
    def reduce_stock(self, quantity: int):
        """在庫を減らす能力"""
        pass

    @abstractmethod
    def get_stock(self) -> int:
        """在庫数を報告する能力"""
        pass

class PhysicalProduct(Product):
    """
    【Entitiesレイヤー / Entity】
    在庫を持つ「物理的な商品」を表す具象クラス。
    """
    def __init__(self, product_id: str, name: str, price: int, stock: int):
        super().__init__(product_id, name, price)
        self._stock = stock

    def check_stock(self, quantity: int) -> bool:
        return self._stock >= quantity

    def reduce_stock(self, quantity: int):
        if not self.check_stock(quantity):
            raise ValueError("在庫が不足しています。")
        self._stock -= quantity

    def get_stock(self) -> int:
        return self._stock

class DigitalProduct(Product):
    """
    【Entitiesレイヤー / Entity】
    在庫の概念がない「ダウンロード商品」を表す具象クラス。
    """
    def check_stock(self, quantity: int) -> bool:
        # ダウンロード商品は常に在庫ありと見なす
        return True

    def reduce_stock(self, quantity: int):
        # 在庫を減らす処理は不要
        pass

    def get_stock(self) -> int:
        # 在庫の概念がないため0を返す（あるいは特定の値を返すなど、ビジネスルールによる）
        return 0

```

この安定した`Product`エンティティを土台として、次は今回のリファクタリングの主役である、より豊かなドメインモデルOrder集約を見ていきましょう。