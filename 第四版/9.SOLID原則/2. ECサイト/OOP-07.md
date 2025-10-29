# OOP-07: さらなる改善 - DDDによるドメイン層の強化

`OOP-06`で、CA適用によりSOLID原則（SRP/DIP）が達成されたことを確認しました。アーキテクチャは健全になりましたが、コードの「質」にはまだ改善の余地があります。

1.  **プリミティブ固執**: `domain.py`の`Product`クラスは、`price: int`のように\*\*`int`（プリミティブ型）\*\*を使っています。これでは「価格はマイナスになれない」といったビジネスルールを表現できません。
2.  **太ったユースケース**: `application.py`の`ProcessOrderUseCase`は、依然として多くの調整ロジック（商品を探す→在庫チェック→在庫減らす→注文作成→保存…）を持っています。これは「手続き型スクリプト」に近く、ドメインの知識がアプリケーション層に漏れ出ている兆候です。

この章では、これらの問題を解決するため、DDDの\*\*値オブジェクト（Value Object）**と**集約（Aggregate）\*\*というパターンを導入します。

## 🎯 この章のゴール

  * `int`などのプリミティブ型でドメインを表現する危険性（**プリミティブ固執**）を理解する。
  * **値オブジェクト (Value Object)** を導入し、ビジネスの不変条件（ルール）をカプセル化する（**SRPの更なる徹底**）。
  * 「太ったユースケース」の問題点を理解する。
  * **集約 (Aggregate)** の概念を導入し、`Store`を「集約ルート」としてドメイン層に復活させる。
  * ビジネスロジックをユースケース層から集約（ドメイン層）に移動させ、**SRPとDIPをさらに徹底**する。

-----

## 🛠️ 1\. ドメイン層の強化 (1): 値オブジェクト(VO)の導入

現在の`domain.py`や`application.py`は、`price: int`, `quantity: int`のように、単なる`int`型でビジネスの値を扱っています。

これは「**プリミティブ固執**」というアンチパターンです。`int`型は、ビジネスの重要なルール（不変条件）を表現できません。

  * `price`はマイナスになれません。
  * `quantity`（注文数）は0以下になれません。

`OOP-05`の`ProcessOrderUseCase`では、`product.check_stock(quantity)`を呼んでいます。もし`main.py`から渡された`quantity`が`-5`だったらどうなるでしょう？
`PhysicalProduct.check_stock`（`self._stock >= quantity`）は`True`になり、`PhysicalProduct.reduce_stock`（`self._stock -= quantity`）が実行され、**在庫が増えてしまいます**。

この「不正な値のチェック」は、ユースケースや`Product`クラスの**責務ではありません**。それは「値」そのものの責務です。
そこで、「値」と「その値が守るべき不変条件」をカプセル化した\*\*値オブジェクト(VO)\*\*を導入します。

> **`domain_vo.py` (新設)**

```python
from dataclasses import dataclass

@dataclass(frozen=True) # 不変(immutable)であることを示す
class Money:
    """「金額」を表す値オブジェクト"""
    value: int

    def __post_init__(self):
        # __init__ の直後に呼ばれるバリデーション
        if self.value < 0:
            raise ValueError("金額は0以上である必要があります")

    def __mul__(self, other_val: int) -> "Money":
        # (簡易的に)intとの乗算を定義
        if not isinstance(other_val, int):
            raise TypeError("乗算は整数のみ可能です")
        return Money(self.value * other_val)

@dataclass(frozen=True)
class Quantity:
    """「数量」を表す値オブジェクト"""
    value: int

    def __post_init__(self):
        if self.value <= 0:
            raise ValueError("数量は1以上である必要があります")

    def __ge__(self, other: "Quantity") -> bool: # >= 演算子の定義
        # 型が違う場合は比較エラー（より厳密に）
        if not isinstance(other, Quantity):
            return NotImplemented
        return self.value >= other.value

    def __sub__(self, other: "Quantity") -> "Quantity": # - 演算子の定義
        if not isinstance(other, Quantity):
            return NotImplemented
        new_val = self.value - other.value
        # 減算結果がマイナスになること自体は許可する（在庫VOの責務でチェック）
        # ただし、在庫VO側でQuantity(0)と比較するために必要
        # より良い設計はStock VOを作ること
        return Quantity(new_val)
```

**【謎解き - VOとSRP】**: 「**値の正しさを保証する**」という責任を、`UseCase`や`Product`から`Money`や`Quantity`という新しいクラスに分離しました。これも\*\*SRP（単一責任の原則）\*\*の実践です。

-----

## 🛠️ 2\. ドメイン層の強化 (2): 集約(Aggregate)の導入

`OOP-05`のユースケース（`ProcessOrderUseCase`）は、

1.  `IProductRepository`から`Product`を**読み出し**、
2.  `Product`の`check_stock`, `reduce_stock`を呼び出し、
3.  `IProductRepository`に`Product`を**保存**し、
4.  `OrderData`(DTO)を**作成**し、
5.  `IOrderRepository`に`OrderData`を**保存**する
    ...という、非常に多くの\*\*調整ロジック（手続き）\*\*を持っていました。

これは「**太ったユースケース**」であり、ドメイン（`Product`）の内部状態（在庫）をユースケースが直接操作しようとしている（`check_stock`してから`reduce_stock`を呼ぶ、など）ため、「**ドメイン知識の漏洩**」も起きています。

DDDでは、このような「**一連の操作で一貫性を保つべき、ビジネスルールを持つオブジェクトの固まり**」を\*\*集約（Aggregate）\*\*と呼びます。
`OOP-01`で最初に登場した`Store`クラスの本来の役割は、「店舗内の商品在庫の一貫性を管理する」ことでした。これはまさに集約の責務です。

そこで、`OOP-03`で一度解体した`Store`を、今度は**集約ルート・エンティティ**として`domain.py`に**復活**させ、注文処理のビジネスロジックをユースケースから移動させます。

> **`domain.py` (OOP-03からの全面改訂)**

```python
import datetime
from domain_vo import Money, Quantity # VOをインポート
from abc import ABC, abstractmethod

# --- 商品インターフェースと実装 (VOを使うように修正) ---

class IProduct(ABC):
    # (OOP-03のIProductと同じだが、型ヒントをVOに変更)
    def __init__(self, product_id: str, name: str, price: Money): # int -> Money
        self._product_id = product_id
        self._name = name
        self._price = price
    # ... (プロパティ定義も Money に) ...
    @property
    def price(self) -> Money: return self._price

    @abstractmethod
    def check_stock(self, quantity: Quantity) -> bool: ... # int -> Quantity
    @abstractmethod
    def reduce_stock(self, quantity: Quantity): ... # int -> Quantity

class PhysicalProduct(IProduct):
    # (OOP-03のPhysicalProductと同じだが、型ヒントをVOに変更)
    def __init__(self, product_id: str, name: str, price: Money, stock: Quantity): # VOを使用
        super().__init__(product_id, name, price)
        self._stock = stock

    @property
    def stock(self) -> Quantity: return self._stock

    def check_stock(self, quantity: Quantity) -> bool:
        """【実装】VO同士で比較"""
        return self._stock >= quantity

    def reduce_stock(self, quantity: Quantity):
        """【実装】VO同士で減算し、新しいVOで更新"""
        if not self.check_stock(quantity):
            raise ValueError(f"{self.name}の在庫が不足しています。") # 例外を投げる
        
        # Quantityの減算結果がマイナスにならないかのチェックも本当は必要
        # (Stock VOを作るのがより良い)
        try:
            self._stock = self._stock - quantity
            print(f"在庫更新: {self.name}の在庫が{self._stock.value}になりました。")
        except ValueError as e: # 減算でマイナスになった場合など
             raise ValueError(f"在庫更新エラー: {e}")


class DigitalProduct(IProduct):
     # (OOP-03のDigitalProductと同じだが、型ヒントをVOに変更)
    def __init__(self, product_id: str, name: str, price: Money): ... # VOを使用
    def check_stock(self, quantity: Quantity) -> bool: ... # VOを使用
    def reduce_stock(self, quantity: Quantity): ... # VOを使用

# --- 注文データオブジェクト (DDDのエンティティ/VOではない簡易版) ---
# (application_boundaries.pyから移動し、VOを使うように修正)
class OrderData:
    def __init__(self, product_name: str, quantity: Quantity, total_price: Money, order_date: str):
        self.product_name = product_name
        self.quantity = quantity
        self.total_price = total_price
        self.order_date = order_date
    # (reprなども追加すると良い)

# --- 集約ルート ---
class Store:
    """
    【新設】「店舗」集約ルート・エンティティ。
    自身の持つ商品(IProduct)の一貫性を管理する。
    """
    def __init__(self, name: str):
        self.name = name
        # 店舗が持つ商品（インターフェース）の辞書
        self._products: dict[str, IProduct] = {}

    def add_product(self, product: IProduct):
        """集約内に商品を追加する"""
        self._products[product.product_id] = product

    def _find_product(self, product_id: str) -> IProduct | None:
        """集約内から商品を探す"""
        return self._products.get(product_id)

    def process_order(self, product_id: str, quantity: Quantity) -> OrderData:
        """
        【重要】
        注文処理のビジネスロジック（太ったユースケースの責務）が、
        ドメイン（集約）に移動した。
        """
        product = self._find_product(product_id)
        if not product:
            raise ValueError("指定された商品が見つかりません。")

        # 在庫チェックと削減は、子（IProduct）に委任 (ポリモーフィズム)
        # check_stockは例外を投げないのでここでチェック
        if not product.check_stock(quantity):
            raise ValueError(f"{product.name}の在庫が不足しています。")

        # reduce_stockは在庫不足なら例外を投げる可能性がある
        product.reduce_stock(quantity)

        # 注文データ(OrderData)を作成して返す
        total_price = product.price * quantity.value # Money * int -> Money
        order_data = OrderData(
            product_name=product.name,
            quantity=quantity,
            total_price=total_price,
            order_date=datetime.datetime.now().isoformat()
        )
        return order_data
```

**【謎解き - 集約とSRP/DIP】**:

  * **SRP**: 「**ビジネスルール（在庫の一貫性）を守る**」という責任が、`UseCase`から`Store`集約に移譲されました。`UseCase`の責任は「リポジトリの呼び出し順序を決める」ことに限定され、より単一責任に近づきました。
  * **DIP**: `UseCase`はもはや`product.check_stock()`や`product.reduce_stock()`という**詳細な手順**を知る必要がなくなりました。`store.process_order()`という、より\*\*抽象度の高い「依頼」\*\*をするだけになりました。これもDIPの精神（高レベルは低レベルの詳細に依存しない）の、より高度な実践です。

-----

## 🚧 次の課題

ドメイン層（`domain.py`）が大きく変更されました（VOの導入、`Store`集約の復活）。
この変更に伴い、以下のファイルを修正する必要があります。

  * `application_boundaries.py`: `IProductRepository`を`IStoreRepository`に変更。DTOの型ヒントをVOに。
  * `application.py`: `ProcessOrderUseCase`を大幅にスリム化。
  * `infrastructure.py`: `InMemoryProductRepository`を`InMemoryStoreRepository`に変更。
  * `main.py`: リポジトリの差し替え、集約の初期化方法の変更。

次の章（`OOP-08`）で、これらの追従修正を行います。