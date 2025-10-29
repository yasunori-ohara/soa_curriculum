# OOP-06 : DDDによるドメイン層の強化 - 値オブジェクトと集約

`OOP-05`でCAのレイヤー構造を導入し、`Store`クラスの責務を`UseCase`、`Repository`、`Controller`に分離しました。これにより、アーキテクチャは変更に強くなりました。

しかし、CAは「どこに何を置くか」という**配置**のルールであり、`domain.py`や`application.py`の\*\*中身の「質」\*\*までは保証してくれません。

現在の`application.py`（ユースケース）は、ドメインの調整役として多くのロジックを持っており、「太ったユースケース」になっています。逆に`domain.py`（エンティティ）は、ルールが不十分な「貧血ドメインモデル」です。

この章では、DDDのパターンを適用し、**ロジックをアプリケーション層からドメイン層へ移動**させ、より堅牢で関心の分離が明確な設計に進化させます。

## 🎯 この章のゴール

  * `int`などのプリミティブ型でドメインを表現する危険性（**プリミティブ固執**）を理解する。
  * **値オブジェクト (Value Object)** を導入し、ビジネスの不変条件（ルール）をカプセル化する。
  * 「太ったユースケース」の問題点を理解する。
  * **集約 (Aggregate)** の概念を導入し、`Store`を「集約ルート」としてドメイン層に復活させる。
  * ビジネスロジックをユースケース層から集約（ドメイン層）に移動させ、「**Tell, Don't Ask（尋ねるな、命じろ）**」の原則を実践する。

-----

## 🎯 1\. ドメイン層の強化 (1): 値オブジェクト(VO)の導入

現在の`domain.py`は、`price: int`, `stock: int`, `quantity: int`のように、単なる`int`型でビジネスの値を表現しています。

これは「**プリミティブ固執**」というアンチパターンです。`int`型は、ビジネスの重要なルール（不変条件）を表現できません。

  * `price`はマイナスになれません。
  * `stock`はマイナスになれません。
  * `quantity`（注文数）は0以下になれません。

`OOP-05`の`ProcessOrderUseCase`では、`product.check_stock(request.quantity)`を呼んでいます。もし`request.quantity`が`-5`だったらどうなるでしょう？
`Product.check_stock`（`self._stock >= quantity`）は`True`になり、`Product.reduce_stock`（`self._stock -= quantity`）が実行され、**在庫が増えてしまいます**。

この「不正な値のチェック」は、ユースケースやエンティティの**責務ではありません**。それは「値」そのものの責務です。
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
        return Money(self.value * other_val)

@dataclass(frozen=True)
class Quantity:
    """「数量」を表す値オブジェクト"""
    value: int
    
    def __post_init__(self):
        if self.value <= 0:
            raise ValueError("数量は1以上である必要があります")

    def __ge__(self, other: "Quantity") -> bool: # >= 演算子の定義
        return self.value >= other.value
    
    def __sub__(self, other: "Quantity") -> "Quantity": # - 演算子の定義
        new_val = self.value - other.value
        if new_val < 0:
            raise ValueError("計算結果がマイナスになります")
        return Quantity(new_val)
```

-----

## 🎯 2\. ドメイン層の強化 (2): 集約(Aggregate)の導入

`OOP-05`のユースケース（`ProcessOrderUseCase`）は、

1.  `IProductRepository`から`Product`を**読み出し**、
2.  `Product`のメソッドを呼び出し、
3.  `IProductRepository`に`Product`を**保存**し、
4.  `Order`エンティティを**作成**し、
5.  `IOrderRepository`に`Order`を**保存**する
    ...という、非常に多くの調整ロジック（トランザクションスクリプト）を持っていました。

DDDでは、このような「**一連のトランザクションで一貫性を保つべき固まり**」を\*\*集約（Aggregate）\*\*と呼びます。
`OOP-01`の`Store`がやろうとしていた「店舗内の商品の在庫を一貫して管理する」という概念は、まさに集約の責務です。

そこで、`OOP-03`で一度解体した`Store`を、今度は**集約ルート・エンティティ**として`domain.py`に**復活**させます。

> **`domain.py` (OOP-03からの全面改訂)**

```python
import datetime
from domain_vo import Money, Quantity # プリミティブ型の代わりにVOをインポート

# --- 子エンティティ ---
class Product:
    """
    【変更】「商品」エンティティ。
    Store集約の「内部」エンティティとして振る舞う。
    """
    def __init__(self, product_id: str, name: str, price: Money, stock: Quantity):
        self.product_id = product_id
        self.name = name
        self.price = price
        self._stock = stock # 在庫もVOで管理
    
    @property
    def stock(self) -> Quantity:
        return self._stock

    def check_stock(self, quantity: Quantity) -> bool:
        """VO同士の比較 (>=)"""
        return self._stock >= quantity

    def reduce_stock(self, quantity: Quantity):
        """VO同士の減算 (-)"""
        if not self.check_stock(quantity):
            raise ValueError(f"{self.name}の在庫が不足しています。")
        
        # VOは不変なので、新しいVOインスタンスで上書きする
        self._stock = self._stock - quantity
        print(f"在庫更新: {self.name}の在庫が{self._stock.value}になりました。")


# --- 別の集約 ---
class Order:
    """
    「注文」エンティティ（それ自身が集約ルート）。
    内容はOOP-03とほぼ同じだが、VOを使用する。
    """
    def __init__(self, product_name: str, quantity: Quantity, total_price: Money):
        self.product_name = product_name
        self.quantity = quantity
        self.total_price = total_price
        self.order_date = datetime.datetime.now().isoformat()
    
    def __repr__(self):
        return f"<Order: {self.product_name}, Qty: {self.quantity.value}>"

# --- 集約ルート ---
class Store:
    """
    【新設】「店舗」集約ルート・エンティティ。
    自身の持つ商品(子エンティティ)の一貫性を管理する。
    """
    def __init__(self, name: str):
        self.name = name
        # 店舗が持つ商品（子エンティティ）のリスト
        # (本来はIDで参照すべきだが、簡略化のため実体を持つ)
        self._products: dict[str, Product] = {}

    def add_product(self, product: Product):
        """集約内に商品を追加する"""
        self._products[product.product_id] = product

    def _find_product(self, product_id: str) -> Product | None:
        """集約内から商品を探す"""
        return self._products.get(product_id)

    def process_order(self, product_id: str, quantity: Quantity) -> Order:
        """
        【重要】
        注文処理ロジック（ユースケースの責務）が、ドメイン（集約）に移動した。
        これが「Tell, Don't Ask（尋ねるな、命じろ）」の実践。
        """
        product = self._find_product(product_id)
        if not product:
            raise ValueError("指定された商品が見つかりません。")

        # 在庫チェックと削減は、子エンティティに委任
        product.reduce_stock(quantity) # (ここで在庫不足なら例外が発生する)

        # 注文（別の集約）を作成して返す
        total_price = product.price * quantity.value
        order = Order(
            product_name=product.name,
            quantity=quantity,
            total_price=total_price
        )
        return order
```

-----

## 🎯 3\. 境界とユースケースの変更

ドメインロジックが`Store`集約に移動したため、`UseCase`（アプリケーション層）は劇的にスリムになります。また、`Product`を個別に扱うリポジトリは不要になり、`Store`集約を丸ごと扱うリポジトリが必要になります。

> **`application_boundaries.py` (OOP-03からの改訂)**

```python
from abc import ABC, abstractmethod
from domain import Store, Order # Productが消え、Storeが追加
# (DTOs: ProcessOrderRequest, ProcessOrderResponse も適宜VOを使うように修正)
# (IProcessOrderUseCase, IPresenter は変更なし)
# ...

# --- リポジトリの境界（契約）を変更 ---

class IStoreRepository(ABC):
    """
    【新設】Store集約を永続化するリポジトリ。
    IProductRepository は不要になった。
    """
    @abstractmethod
    def find_by_name(self, store_name: str) -> Store | None:
        pass
    
    @abstractmethod
    def save(self, store: Store):
        pass

class IOrderRepository(ABC):
    """(変更なし) Order集約を保存する"""
    @abstractmethod
    def save(self, order: Order, store_name: str):
        pass
```

> **`application.py` (OOP-04からの改訂)**

```python
from application_boundaries import (
    IProcessOrderUseCase, IProcessOrderPresenter,
    IStoreRepository, IOrderRepository, # IProductRepository が IStoreRepository に
    ProcessOrderRequest, ProcessOrderResponse
)
from domain import Store, Order
from domain_vo import Quantity # VOをインポート

class ProcessOrderUseCase(IProcessOrderUseCase):
    """
    【変更】ユースケースが劇的にスリムになった。
    ドメインの調整役ではなく、薄い「オーケストレーター」になった。
    """
    def __init__(self,
                 store_repo: IStoreRepository, # 変更
                 order_repo: IOrderRepository,
                 presenter: IProcessOrderPresenter):
        
        self._store_repo = store_repo
        self._order_repo = order_repo
        self._presenter = presenter

    def handle(self, request: ProcessOrderRequest):
        try:
            # 1. 入力値を「VO」に変換 (不正な値はここでエラー)
            quantity_vo = Quantity(request.quantity)

            # 2. Store集約を「読み出す」
            store = self._store_repo.find_by_name(request.store_name)
            if not store:
                raise ValueError(f"店舗 {request.store_name} が見つかりません。")

            # 3. ドメイン(集約)に「命じる」(Tell, Don't Ask)
            #    ロジックの詳細はユースケースは「知らない」
            order = store.process_order(request.product_id, quantity_vo)

            # 4. 変更されたStore集約を「保存する」
            self._store_repo.save(store)
            
            # 5. 新しいOrder集約を「保存する」
            self._order_repo.save(order, request.store_name)

            # 6. 成功をPresenterに通知
            response = ProcessOrderResponse(
                order=order,
                # (簡易的に) Store集約から在庫情報を取り直す
                updated_stock=store._find_product(request.product_id).stock.value
            )
            self._presenter.present_success(response)

        except (ValueError, TypeError) as e:
            self._presenter.present_failure(str(e))
```

-----

## 🎯 4\. 謎解き - なぜこの変更がSOLID原則を改善したのか？

`OOP-05`のCA導入でSRPとDIPは「アーキテクチャレベル」で解決しました。
今回のDDD導入は、`application.py`と`domain.py`の\*\*「実装レベル」\*\*で、残っていた違反を解消しました。

### 1\. 値オブジェクト(VO)が解決したこと

  * **「プリミティブ固執」の解消**: `int`型が持つ「曖昧さ」を排除しました。
  * **バグの防止**: `quantity = -5`で在庫が増える、といった致命的なバグを**型レベルで**防止しました。`Quantity(-5)`はインスタンス化の瞬間に例外を発生させ、不正なデータがドメイン層に侵入することを防ぎます。
  * **SRP（単一責任の原則）の徹底**:
    「**値の不変条件（ルール）を検証する**」という責任は、ユースケースやエンティティのものではありませんでした。
    VO（`Quantity`クラス）を新設し、その責任を移譲しました。`ProcessOrderUseCase`も`Product`も`Store`も、もはや「数量が0以下でないか」を気にする必要がありません。**責任が正しく分離されました**。

### 2\. 集約(Aggregate)が解決したこと

  * **「太ったユースケース」の解消**: `OOP-05`の`ProcessOrderUseCase`は、手続き型スクリプトであり、多くの責任（在庫調整、注文作成、リポジトリ調整）を持っていました。これは**SRP違反**です。
  * **ロジックのドメイン層への移動**: `store.process_order()`という形で、ビジネスロジック（一連のトランザクション）をドメイン層（`Store`集約）にカプセル化しました。
  * **SRP（単一責任の原則）の徹底**:
      * `ProcessOrderUseCase`（アプリケーション層）の責任は、「**HTTPリクエストやDBをオーケストレーションすること**」に限定されました。
      * `Store`（ドメイン層）の責任は、「**自身のビジネスルール（店舗内の在庫と注文の一貫性）を守ること**」に限定されました。
  * **DIP（依存性逆転の原則）の徹底**:
    `OOP-05`では、ユースケースが`Product`エンティティの`check_stock`や`reduce_stock`といった\*\*詳細なメソッド（実装）**を知っていました。
    `OOP-06`では、ユースケースは`store.process_order()`という、より**抽象度の高い「能力」\*\*を呼び出すだけになりました。ユースケースは、注文処理の*具体的な手順*を知らなくてもよくなりました。これもDIPの精神（高レベルなモジュールは低レベルな実装の詳細に依存しない）の、より高度な実践です。

-----

## 🚧 次の課題

`infrastructure.py`と`main.py`が、`IStoreRepository`という新しい境界（インターフェース）に対応していません。
また、`OOP-02`で指摘されたOCP違反（ダウンロード商品の追加）は、まだ解決されていません。

次の章で、インフラ層を修正し、OCP違反の解決に取り組みます。