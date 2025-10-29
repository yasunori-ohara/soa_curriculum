
# OOP-08 : OCP/LSPの達成 - ポリモーフィズムによるドメイン拡張

`OOP-07`までのリファクタリングにより、アーキテクチャはクリーンになり、ドメインロジックは（VOと集約によって）堅牢になりました。

しかし、`OOP-02`で指摘された「**ダウンロード商品の追加**」という変更要求には、まだ耐えられません。
現在の`domain.py`の`Product`エンティティは、「物理在庫（`stock: Quantity`）」を持つこと（`__init__`）を前提としています。`Store`集約の`process_order`ロジックも、`product.reduce_stock()`が必ず存在する物理在庫を減らすことを期待しています。

このままでは、「在庫の概念がない」`DigitalProduct`を追加しようとすると、`Product`クラスや`Store`集約の`process_order`メソッドに`if`文による**修正**が必要となり、\*\*OCP（オープン・クローズドの原則）\*\*に違反します。

この問題は、`DigitalProduct`が`Product`の振る舞い（物理在庫の管理）を正しく**置換**できないという\*\*LSP（リスコフの置換原則）\*\*違反でもあります。

この章では、OCP/LSPを達成するため、ドメイン層の内部に\*\*DIP（依存性逆転の原則）\*\*を適用し、**ポリモーフィズム**（多態性）を実現します。

## 🎯 この章のゴール

  * `Store`集約（高レベル）が、`Product`の「実装（物理在庫）」（低レベル）に依存しているという、**ドメイン層内部のDIP違反**を特定する。
  * OCP違反を解決する鍵が、LSPの遵守とDIPの適用にあることを理解する。
  * `Store`集約が依存すべき「**`IProduct`インターフェース**」をドメイン層に定義する。
  * `PhysicalProduct`と`DigitalProduct`が、**ポリモーフィズム**（多態性）によって`Store`集約から区別なく扱われる様子を実装する。
  * **OCP（オープン・クローズドの原則）を達成し、`Store`や`UseCase`を一切修正せず**に、新機能（ダウンロード商品）を**追加**する。

-----

## 1\. ドメイン層 (domain.py) へのDIP適用

OCP違反の根本原因は、`Store`集約（高レベルな方針）が、`Product`エンティティ（物理在庫を持つという低レベルな実装）という**具象クラス**に直接依存していることです。

この依存関係を**逆転**させるため、`Store`集約が本当に必要とする「**商品としての契約（インターフェース）**」を`IProduct`として定義します。

> **`domain.py` (OOP-06からの改訂)**

```python
import datetime
from domain_vo import Money, Quantity
from abc import ABC, abstractmethod # インターフェース定義のためにABCをインポート

# --- 1. 【新設】Productの抽象インターフェース ---
class IProduct(ABC):
    """
    「商品」としてStore集約から扱われるための「契約（インターフェース）」。
    Storeは、具象クラスではなく、この抽象に依存する。
    """
    @property
    @abstractmethod
    def product_id(self) -> str: ...
    
    @property
    @abstractmethod
    def name(self) -> str: ...
    
    @property
    @abstractmethod
    def price(self) -> Money: ...

    @abstractmethod
    def check_stock(self, quantity: Quantity) -> bool:
        """【契約】指定された数量の在庫があるか"""
        pass

    @abstractmethod
    def reduce_stock(self, quantity: Quantity):
        """【契約】指定された数量の在庫を減らす"""
        pass

# --- 2. 【変更】OOP-06のProductを「物理商品」として実装 ---
class PhysicalProduct(IProduct):
    """
    「物理商品」の具象実装クラス。
    IProductインターフェースの契約を実装する。
    """
    def __init__(self, product_id: str, name: str, price: Money, stock: Quantity):
        self._product_id = product_id
        self._name = name
        self._price = price
        self._stock = stock # 物理在庫を持つ

    @property
    def product_id(self) -> str: return self._product_id
    @property
    def name(self) -> str: return self._name
    @property
    def price(self) -> Money: return self._price
    
    @property
    def stock(self) -> Quantity: return self._stock

    def check_stock(self, quantity: Quantity) -> bool:
        """【実装】物理在庫が十分か確認する"""
        return self._stock >= quantity

    def reduce_stock(self, quantity: Quantity):
        """【実装】物理在庫を減らす"""
        if not self.check_stock(quantity):
            raise ValueError(f"{self.name}の在庫が不足しています。")
        self._stock = self._stock - quantity
        print(f"在庫更新: {self.name}の在庫が{self._stock.value}になりました。")


# --- 3. 【新設】OCP達成のための「ダウンロード商品」実装 ---
class DigitalProduct(IProduct):
    """
    「ダウンロード商品」の具象実装クラス。
    IProductインターフェースの契約を「異なる振る舞い」で実装する。
    """
    def __init__(self, product_id: str, name: str, price: Money):
        self._product_id = product_id
        self._name = name
        self._price = price
        # (LSP) stock属性は持たない
    
    @property
    def product_id(self) -> str: return self._product_id
    @property
    def name(self) -> str: return self._name
    @property
    def price(self) -> Money: return self._price

    def check_stock(self, quantity: Quantity) -> bool:
        """【実装】ダウンロード商品は在庫をチェックしない（常にTrue）"""
        return True

    def reduce_stock(self, quantity: Quantity):
        """【実装】ダウンロード商品は在庫を減らさない（何もしない）"""
        print(f"在庫更新: {self.name} はダウンロード商品のため在庫変動なし。")
        pass # LSPを遵守：例外を投げず、振る舞いを実装する

# --- (Orderエンティティは変更なし) ---
class Order: ...

# --- 4. 【変更】Store集約は「IProduct(抽象)」に依存する ---
class Store:
    """
    「店舗」集約ルート。
    DIPにより、具象(Physical/Digital)ではなく、抽象(IProduct)に依存する。
    """
    def __init__(self, name: str):
        self.name = name
        # 【DIP】IProductインターフェースの辞書を保持
        self._products: dict[str, IProduct] = {}

    def add_product(self, product: IProduct): # 【DIP】引数もIProduct
        """集約内に「IProduct(抽象)」を追加する"""
        self._products[product.product_id] = product

    def _find_product(self, product_id: str) -> IProduct | None: # 【DIP】戻り値もIProduct
        return self._products.get(product_id)

    def process_order(self, product_id: str, quantity: Quantity) -> Order:
        """
        【重要】このメソッドのコードは OOP-06 から「一切変更なし」
        """
        product = self._find_product(product_id)
        if not product:
            raise ValueError("指定された商品が見つかりません。")

        # ▼▼▼ ポリモーフィズム（多態性） ▼▼▼
        # product が PhysicalProduct か DigitalProduct かを
        # Store集約は「一切意識する必要がない」
        #
        # ただ「IProduct(契約)」通りに check_stock() を呼び出すだけ。
        #
        # 実行される「具体的な処理」は、Pythonが自動的に判断する。
        # - 相手が PhysicalProduct なら：在庫数を比較する処理
        # - 相手が DigitalProduct なら：常に True を返す処理
        if not product.check_stock(quantity): # <- Storeは変更不要
             # DigitalProductの場合、ここは絶対にFalseにならない
            raise ValueError(f"{product.name}の在庫が不足しています。")

        # ▼▼▼ ポリモーフィズム（多態性） ▼▼▼
        # reduce_stock() も同様。
        # - 相手が PhysicalProduct なら：在庫数を減らす処理
        # - 相手が DigitalProduct なら：何もしない処理
        product.reduce_stock(quantity) # <- Storeは変更不要

        # --- (注文作成ロジックも変更なし) ---
        total_price = product.price * quantity.value
        order = Order(
            product_name=product.name,
            quantity=quantity,
            total_price=total_price
        )
        return order
```

-----

## 2\. 起動層 (main.py) での「拡張」

`domain.py`（ドメイン層）と`application.py`（ユースケース層）のコードは**一切修正していません**。
変更が必要なのは、DI（依存性の注入）を担当する`main.py`で、「**どの具象クラスを注入するか**」を決定する部分だけです。

> **`main.py` (OOP-07からの修正)**

```python
# 【追加】DigitalProduct もインポート
from domain import PhysicalProduct, DigitalProduct, Store
from domain_vo import Money, Quantity
from application import ProcessOrderUseCase
from infrastructure import InMemoryStoreRepository, InMemoryOrderRepository
from adapters import ConsolePresenter, OrderController

if __name__ == "__main__":
    
    # --- 1. 依存性の注入 (DI) ---
    # (OOP-07から一切変更なし)
    store_repo = InMemoryStoreRepository()
    order_repo = InMemoryOrderRepository()
    presenter = ConsolePresenter()
    order_use_case = ProcessOrderUseCase(
        store_repo=store_repo,
        order_repo=order_repo,
        presenter=presenter
    )
    controller = OrderController(use_case=order_use_case)

    # --- 2. 初期データのセットアップ (DigitalProduct を「追加」) ---
    
    tokyo_store = Store("東京店")
    # 【変更】Product -> PhysicalProduct
    tokyo_store.add_product(PhysicalProduct("p-001", "高機能マウス", Money(4000), Quantity(10)))
    tokyo_store.add_product(PhysicalProduct("p-002", "静音キーボード", Money(6000), Quantity(5)))
    # 【新設】DigitalProduct を「追加」
    tokyo_store.add_product(DigitalProduct("d-001", "デザインソフト eBook", Money(8000)))
    store_repo.save(tokyo_store)

    osaka_store = Store("大阪店")
    osaka_store.add_product(PhysicalProduct("p-001", "高機能マウス", Money(4100), Quantity(8)))
    # 【新設】DigitalProduct を「追加」
    osaka_store.add_product(DigitalProduct("d-002", "プログラミング講座 動画", Money(12000)))
    store_repo.save(osaka_store)

    # --- 3. アプリケーションの実行 (DigitalProduct のシナリオを「追加」) ---
    
    # シナリオ1: 東京店 物理商品 (成功)
    print("--- シナリオ1 東京店 物理商品 注文 ---")
    controller.process_order("p-001", 3, "東京店")

    # シナリオ2: 東京店 物理商品 (在庫不足)
    print("\n--- シナリオ2 東京店 物理商品 在庫不足 ---")
    controller.process_order("p-002", 10, "東京店")

    # シナリオ3: 【新】東京店 デジタル商品 (成功)
    print("\n--- シナリオ3 東京店 デジタル商品 注文 ---")
    # 物理在庫が5の「静音キーボード」と同じ数量(10)でも、
    # DigitalProductなら在庫チェックがTrueになり、注文が成功する。
    # UseCaseもStoreも、この振る舞いの違いを「知らない」。
    controller.process_order("d-001", 10, "東京店")
```

-----

## ✨ 謎解き - なぜこの変更がSOLID原則を達成したのか？

### OCP（オープン・クローズドの原則）の達成

私たちは、「ダウンロード商品（`DigitalProduct`）」という**新機能**を追加しました。
この変更のために、**既存のロジック**（`Store.process_order`や`ProcessOrderUseCase.handle`）を**一切「修正」しませんでした**。
`DigitalProduct`という新しいクラスを「**追加**」し、`main.py`でそれを「**追加**」しただけです。

`Store`（集約）と`ProcessOrderUseCase`（ユースケース）は、\*\*修正に対して閉じて（Closed）\*\*おり、\*\*拡張（`IProduct`を実装する新しいクラス）に対して開いて（Open）\*\*います。
OCPは完全に達成されました。

### LSP（リスコフの置換原則）の達成

`Store`集約は、`IProduct`インターフェースに依存しています。
`DigitalProduct`は、`IProduct`の**契約**（`check_stock`は`bool`を返す、`reduce_stock`は例外を投げない）を**完全に満たしています**。
`reduce_stock`の振る舞いが「何もしない」ことであっても、それは`PhysicalProduct`の「在庫を減らす」という振る舞いと\*\*同等の契約（シグネチャ）\*\*を満たしています。

`Store`（利用者側）は、`IProduct`（親）の代わりに`DigitalProduct`（子）を**置換**されても、`if isinstance(...)` のような型チェックを**必要とせず**、プログラムは期待通りに動作します。LSPは達成されました。

### DIP（依存性逆Tの原則）の達成

このOCP/LSP達成の**最大の功労者**がDIPです。
`Store`（高レベルなロジック）が`PhysicalProduct`（低レベルな実装）に依存していた状態を**反転**させ、`Store`も`PhysicalProduct`も`DigitalProduct`も、全員が「`IProduct`」という**抽象（インターフェース）に依存する**ように設計しました。

これにより、CA/DDD/SOLIDのすべての原則を満たした、柔軟で堅牢、かつ保守性の高いアーキテクチャが完成しました。