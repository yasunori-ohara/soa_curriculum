# DDD-02 : 🔵 `domain/product.py` (エンティティ)

それでは、DDDを適用したプログラムの解説を、最も中心に位置する `domain/product.py` から始めます。

## 🎯 この章のゴール

  * DDDリファクタリングの「土台」として、`domain/product.py` の役割を再確認する。
  * このエンティティが、`CA` の章で完成した「安定したビジネスオブジェクト」であることを理解する。
  * （復習）`Product` クラス群の「契約」と「実装」の関係性を、コード詳細解説で再確認する。

-----

## 💎 このファイルの役割：基本的なビジネスオブジェクト (Entities レイヤー)

このファイルは、クリーンアーキテクチャの時と同様、アプリケーションの **Entities（エンティティ）** レイヤーに属します。

今回のDDDリファクタリングにおいて、この `product.py` 自体には大きな変更はありません。`OOP` と `CA` の章を通して、私たちはこのクラスをすでに堅牢な（LSPなどを満たした）状態に磨き上げてあります。

`Product` は、私たちのECサイトというドメイン（ビジネス領域）における、最も基本的で独立した「モノ」の一つです。

この安定した「商品」エンティティを土台として、次の `DDD-03` で、より複雑なビジネス概念である「注文（`Order`）」集約（アグリゲート）を構築していきます。

-----

## 💻 ソースコードの詳細解説

（`CA` の章から久しぶりに学習する方のために、このコードの役割を復習します）

### `Product(ABC)`: 「商品」の契約書（インターフェース）

この抽象基底クラスは、「商品」という概念が持つべき基本的な能力（契約）を定義しています。

  * `check_stock()`: 在庫を確認する能力
  * `reduce_stock()`: 在庫を減らす能力
  * `get_stock()`: 在庫数を（数値で）報告する能力
  * `is_stock_managed()`: 在庫管理対象かを（真偽値で）報告する能力

この「契約書」があるおかげで、`use_cases` レイヤーは相手がどんな種類の商品であっても、これらの能力を持っていることを信頼して仕事を任せることができます。

### `PhysicalProduct` と `DigitalProduct`: 契約書への具体的な署名

これらは、`Product` という契約書に\*\*署名（継承）\*\*し、その内容を具体的に実装したクラスです。

  * `PhysicalProduct` は、契約通りに物理的な在庫数（`_stock`）を操作します。
  * `DigitalProduct` も同じ契約書に署名しますが、その振る舞いは異なります。在庫は常に「あり」とみなし（`check_stock` は `True`）、在庫数は `0` を返す（`get_stock`）、といった具合です。

-----

## 🏛️ このレイヤーの鉄則（再確認）

  * **自己完結している:** `Product` エンティティは、自分自身の状態とルールにのみ責任を持ちます。「注文」や「顧客」といった、他のビジネス概念のことは知りません。
  * **安定している:** ここで定義されているのは、ビジネスの基本的な構成要素です。ビジネスのフローが変わっても、商品そのものの性質が変わらない限り、このファイルが変更されることはほとんどありません。

-----

## 📄 `domain/product.py`

（このコードは `CA-02` (LSP修正後) と同一です）

```python:domain/product.py
# 依存性のルール:
# このファイルは Entities レイヤーに属します。
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
        """【契約】在庫を確認する能力"""
        pass

    @abstractmethod
    def reduce_stock(self, quantity: int):
        """【契約】在庫を減らす能力"""
        pass

    @abstractmethod
    def get_stock(self) -> int:
        """【契約】在庫数を報告する能力 (LSP遵守のため int)"""
        pass
    
    @abstractmethod
    def is_stock_managed(self) -> bool:
        """【契約】在庫管理の対象商品かどうかを返す (LSP遵守のため)"""
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
        """
        エンティティが自身のルール（不変条件）を守る例：
        在庫が不足している状態での在庫削減は不正な操作であるため、
        例外を発生させて処理を中断させる。
        """
        if not self.check_stock(quantity):
            # 不正な状態を防ぐため、処理を中断
            raise ValueError(f"在庫が不足しています (商品: {self.name})")
        self._stock -= quantity

    def get_stock(self) -> int:
        return self._stock
    
    def is_stock_managed(self) -> bool:
        return True

class DigitalProduct(Product):
    """
    【Entitiesレイヤー / Entity】
    在庫の概念がない「ダウンロード商品」を表す具象クラス。
    """
    def __init__(self, product_id: str, name: str, price: int):
        super().__init__(product_id, name, price)

    def check_stock(self, quantity: int) -> bool:
        # ダウンロード商品は常に在庫あり
        return True

    def reduce_stock(self, quantity: int):
        # 在庫を減らす処理は不要
        pass

    def get_stock(self) -> int:
        # 在庫の概念がないため0を返す (LSP遵守)
        return 0
    
    def is_stock_managed(self) -> bool:
        # 在庫管理の対象外 (LSP遵守)
        return False
```

この安定した `Product` エンティティを土台として、次（`DDD-03`）は今回のリファクタリングの主役である、`Order` 集約（アグリゲート）を見ていきましょう。