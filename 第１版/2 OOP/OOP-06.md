# OOP-06 : 単一責任の原則(SRP)とリポジトリパターン

`OOP-04`でOCP（オープン・クローズドの原則）の問題は解決しましたが、`OOP-02`で指摘したもう一つの重要な課題、**S（単一責任の原則）違反**が残っています。

この章では、このSRP違反を解消し、システム全体の柔軟性をさらに高めます。

## 🎯 この章のゴール

  * `Store`クラスのSRP（単一責任の原則）違反を解消する。
  * 「ビジネスロジック」と「データ保存ロジック」を分離する動機を理解する。
  * **リポジトリパターン**の基本的な考え方と導入方法を学ぶ。
  * 「注文 (`Order`)」という概念を、辞書（`dict`）から`Order`クラス（エンティティ）に昇格させるメリットを理解する。
  * 依存性の注入（DI）を使い、`Store`が「抽象的なリポジトリ」に依存するように変更する（DIPの徹底）。

-----

## 🧐 なぜSRP違反（責任の混在）が問題なのか？

現在の`Store`クラスの`process_order`メソッドを見てみましょう。

### logic.py (OOP-04の問題点)
```python
class Store:
    def process_order(self, product_id: str, quantity: int):
        # ... (中略) ...

        # 責任1: 在庫チェックや削減指示（ビジネスロジック）
        product = self._find_product(product_id)
        if not product.check_stock(quantity):
            return
        product.reduce_stock(quantity)

        # 責任2: 注文記録の「形式」を決定する
        order_record = {
            "product_name": product.name,
            "quantity": quantity,
            "total_price": product.price * quantity,
            "order_date": datetime.datetime.now().isoformat()
        }
        
        # 責任3: 注文記録を「保存」する
        self._orders.append(order_record) 
```

`Store`クラスは、本来の責任である「注文フローの管理（責任1）」だけでなく、「注文データをどのような形式（`dict`）で作るか（責任2）」、「それをどこに（`_orders`リストに）保存するか（責任3）」という、**データ保存（永続化）に関する責任**まで持ってしまっています。

これがなぜ問題なのでしょうか？
もし将来、以下のような**変更要求**が来たらどうなるでしょう。

  * 「注文記録を、メモリ上のリストではなく、**データベース(DB)に保存**したい」
  * 「注文記録を、ファイルに書き出したい」

どちらの要求でも、`Store`クラスの`process_order`メソッド（ビジネスロジックの中核）に**修正**を加える必要があります。これは、`Store`が「修正に対して閉じていない」ことになり、OCP違反も引き起こします。

単一責任の原則（SRP）は、\*\*「変更する理由が複数あるクラスは、分割すべき」\*\*という原則です。「注文フローの変更」と「注文の保存先の変更」という2つの異なる理由で修正が必要になる`Store`は、明らかにSRPに違反しています。

-----

## 💡 解決策：責任の分離とリポジトリパターン

この問題を解決するには、`Store`から「データ保存」に関する責任を分離し、別のクラスに任せる必要があります。

### 1\. 「注文」を辞書(dict)からクラス(Order)へ昇格させる

まず、`Store`と「保存担当」の間で受け渡すデータ（＝注文記録）を、曖昧な`dict`（辞書）のままにしておくのは危険です。`dict`はキーのタイプミスが起きやすく、どのようなデータが含まれているかが不明確です。

そこで、「注文」というビジネス上の重要な\*\*概念（エンティティ）\*\*を、明確な`Order`**クラス**として定義（昇格）します。

### 2\. リポジトリパターン (Repository Pattern) の導入

次に、「`Order`クラスのインスタンスを保存する」という責任を持つクラスが必要です。この「**ビジネスロジック（`Store`）**」と「**データ保存（DBやリストなど）**」の間に立ち、データの永続化を仲介する設計パターンを**リポジトリパターン**と呼びます。

リポジトリは「データの倉庫番」のようなもので、`Store`（ビジネスロジック）は「この`Order`を保存しておいて」とリポジトリに依頼するだけです。リポジトリがその裏側で、実際にリストに`append`するのか、DBに`INSERT`するのかを`Store`は知る必要がありません。

### 3\. DIPの徹底（SRP達成の手段）

では、`Store`は「倉庫番（リポジトリ）」をどうやって利用すればよいでしょうか？

`OOP-04`で学んだDIP（依存性逆転の原則）をここでも徹底します。もし`Store`が`InMemoryOrderRepository`（メモリ保存）という**具象クラス**に依存してしまうと、`DatabaseOrderRepository`（DB保存）に切り替えるときに、結局`Store`の修正が必要になります。

`Store`が依存すべき相手は、「**注文を保存する能力（`save`メソッド）**」という\*\*抽象的な契約（インターフェース）\*\*です。

1.  `IOrderRepository`（インターフェース）を定義し、「`save`」という契約を定めます。
2.  `Store`は`IOrderRepository`（抽象）にのみ依存します（DIP）。
3.  `InMemoryOrderRepository`（具象）は`IOrderRepository`を実装します。
4.  `main.py`が、`Store`に`InMemoryOrderRepository`を\*\*注入（DI）\*\*します。

これにより、`Store`は「どう保存するか」という責任から完全に解放され、SRPを達成できます。

-----

## 🔗 変更の概要

上記の方針に基づき、以下の変更を行います。

1.  **`Order` クラスの新設:**
    注文記録（`dict`）を、`Order`クラスという明確な「概念」にします。
2.  **`IOrderRepository` (インターフェース) の新設:**
    「`Order`を保存する」という「契約（`save`メソッド）」を定義します。
3.  **`InMemoryOrderRepository` (具象) の新設:**
    `IOrderRepository`を実装し、`Store`が今まで`_orders`リストに保存していた処理を、このクラスが引き継ぎます。
4.  **`Store` クラスのリファクタリング:**
      * `__init__`で`IOrderRepository`を受け取るようにします（依存性の注入）。
      * `process_order`から、注文記録の作成と保存ロジックを削除し、`Order`オブジェクトを作成して`self._repository.save(...)`を呼び出すだけにします。

-----

## 💻 `logic.py` : リポジトリの導入と `Store` のスリム化

`logic.py` が、ビジネスの「概念（エンティティ）」と「ロジック（`Store`）」、そして「保存（リポジトリ）」の定義場所となります。

```python
import datetime
from abc import ABC, abstractmethod

# --- OOP-04までの Product 関連 (変更なし) ---
# (※OOP-04で get_stock_info() -> str に修正済みとする)

class Product(ABC):
    def __init__(self, product_id: str, name: str, price: int): ...
    @abstractmethod
    def check_stock(self, quantity: int) -> bool: ...
    @abstractmethod
    def reduce_stock(self, quantity: int): ...
    @abstractmethod
    def get_stock_info(self) -> str: ... # OOP-04で修正済みの想定

class PhysicalProduct(Product):
    def __init__(self, product_id: str, name: str, price: int, stock: int): ...
    def check_stock(self, quantity: int) -> bool: ...
    def reduce_stock(self, quantity: int): ...
    def get_stock_info(self) -> str: ... # OOP-04で修正済みの想定

class DigitalProduct(Product):
    def __init__(self, product_id: str, name: str, price: int): ...
    def check_stock(self, quantity: int) -> bool: ...
    def reduce_stock(self, quantity: int): ...
    def get_stock_info(self) -> str: ... # OOP-04で修正済みの想定

# --- ここからが「追加・変更」したコード ---

class Order:
    """
    【新設】「注文」という概念を表すクラス（エンティティ）。
    今までの「辞書(dict)」から昇格させた。
    データとその構造をカプセル化する。
    """
    def __init__(self, product_name: str, quantity: int, total_price: int):
        self.product_name = product_name
        self.quantity = quantity
        self.total_price = total_price
        self.order_date = datetime.datetime.now().isoformat()
        
    def __repr__(self):
        # print() で表示したときに見やすくするためのメソッド
        return f"<Order: {self.product_name}, Qty: {self.quantity}, Price: {self.total_price}>"

class IOrderRepository(ABC):
    """
    【新設】注文を保存するための「抽象的」なインターフェース（契約）。
    Storeクラスは、このインターフェースに依存する。
    """
    @abstractmethod
    def save(self, order: Order):
        """【契約】注文(Order)を保存する"""
        pass
    
    @abstractmethod
    def get_all(self) -> list[Order]:
        """【契約】すべての注文履歴を取得する"""
        pass

class InMemoryOrderRepository(IOrderRepository):
    """
    【新設】注文を「メモリ上のリスト」に保存する「具体的」な実装クラス。
    Storeが持っていた _orders リストは、ここが管理する。
    """
    def __init__(self):
        self._orders: list[Order] = [] # 注文を保存するリスト

    def save(self, order: Order):
        """【実装】リストに Order オブジェクトを追加する"""
        self._orders.append(order)
        print(f"注文記録保存: {order.product_name} をリストに保存しました。")

    def get_all(self) -> list[Order]:
        """【実装】保持しているリストを返す"""
        return self._orders

class Store:
    """
    【変更】店舗を表すクラス。
    注文の「保存」ロジックが分離され、本来の「ビジネスロジック」に集中する。
    """
    def __init__(self, name: str, order_repository: IOrderRepository):
        self.name = name
        self._products: dict[str, Product] = {}
        
        # 変更点1: 自分で _orders リストを持つ代わりに、
        # 外部から「リポジトリ(保存担当)」を受け取る（依存性の注入）
        self._order_repository = order_repository
        
        # `self._orders = []` は削除された

    def add_product(self, product: Product):
        self._products[product.product_id] = product

    def _find_product(self, product_id: str) -> Product | None:
        return self._products.get(product_id)

    def process_order(self, product_id: str, quantity: int):
        """
        【変更】このメソッドは「注文のフロー管理」という単一責任になった。
        """
        print(f"\n--- [{self.name}] 注文処理開始: 商品ID={product_id}, 数量={quantity} ---")

        # --- 責任1: ビジネスロジックの実行 ---
        product = self._find_product(product_id)
        if not product:
            print(f"エラー: [{self.name}] 指定された商品が見つかりません。")
            return

        if not product.check_stock(quantity):
            print(f"エラー: [{self.name}] {product.name}の在庫が不足しています。")
            return

        product.reduce_stock(quantity)

        # --- 責任2 & 3 (データ保存) の分離 ---

        # 変更点2: 注文記録(dict)の「作成」ロジックを削除
        # 代わりに「Orderオブジェクト」を作成する
        order = Order(
            product_name=product.name,
            quantity=quantity,
            total_price=product.price * quantity
        )

        # 変更点3: 自分でリストに append する代わりに、
        # 保存担当（リポジトリ）に「保存を依頼」する（DIP）
        self._order_repository.save(order)
        
        # `self._orders.append(order_record)` は削除された

        print(f"注文成功: [{self.name}] {product.name}を{quantity}個受け付けました。")

    def get_current_stock_info(self) -> dict:
        # (このメソッドは変更なし)
        stock_info = {}
        for p_id, p in self._products.items():
            stock_info[p.name] = p.get_stock_info()
        return stock_info
    
    def get_order_history(self) -> list[Order]:
        """【新設】注文履歴の取得も、リポジトリに依頼する"""
        return self._order_repository.get_all()
```

-----

## 🏛️ `main.py` : 依存性の注入（DI）

`main.py`の役割は、`Store`が依存する「部品（具象リポジトリ）」を組み立てて（**依存性の注入**）、アプリケーションを実行することです。

```python
# Order, IOrderRepository, InMemoryOrderRepository をインポート
from logic import (
    PhysicalProduct, DigitalProduct, Store, 
    IOrderRepository, InMemoryOrderRepository, Order # 追加
)

if __name__ == "__main__":
    
    # --- 1. 依存関係の構築 (DI) ---
    
    # 東京店用の「保存担当（リポジトリ）」インスタンスを作成
    tokyo_repo: IOrderRepository = InMemoryOrderRepository()
    
    # Store を作成する際、保存担当（リポジトリ）を「注入」する
    tokyo_store = Store("東京店", order_repository=tokyo_repo)
    
    # 大阪店用の「保存担当（リポジトリ）」インスタンスも作成
    # tokyo_repo とは別のインスタンスなので、注文履歴は独立する
    osaka_repo: IOrderRepository = InMemoryOrderRepository()
    osaka_store = Store("大阪店", order_repository=osaka_repo)

    # --- 2. 商品のセットアップ (変更なし) ---
    tokyo_store.add_product(PhysicalProduct("p-001", "高機能マウス", 4000, 10))
    tokyo_store.add_product(PhysicalProduct("p-002", "静音キーボード", 6000, 5))
    tokyo_store.add_product(DigitalProduct("d-001", "デザインソフト eBook", 8000))
    
    osaka_store.add_product(PhysicalProduct("p-001", "高機能マウス", 4100, 8))
    osaka_store.add_product(DigitalProduct("d-002", "プログラミング講座 動画", 12000))

    # --- 3. 在庫表示と注文処理 (変更なし) ---
    # Store を「使う側」のコードは一切変更する必要がない
    # Store内部の実装（保存方法）が変わったことを、使う側は知らない
    
    print("--- 初期在庫 ---")
    print(f"{tokyo_store.name}:", tokyo_store.get_current_stock_info())
    print(f"{osaka_store.name}:", osaka_store.get_current_stock_info())

    tokyo_store.process_order("p-001", 3)
    tokyo_store.process_order("p-002", 10) # 在庫不足
    tokyo_store.process_order("d-001", 50) # ダウンロード
    osaka_store.process_order("d-002", 100) # 大阪ダウンロード

    print("\n--- 最終在庫 ---")
    print(f"{tokyo_store.name}:", tokyo_store.get_current_stock_info())
    print(f"{osaka_store.name}:", osaka_store.get_current_stock_info())

    # --- 4. 注文履歴の表示 ---
    # Store経由で、リポジトリから履歴を取得する
    print(f"\n--- {tokyo_store.name} 注文履歴 ---")
    for order in tokyo_store.get_order_history():
        print(order)
        
    print(f"\n--- {osaka_store.name} 注文履歴 ---")
    for order in osaka_store.get_order_history():
        print(order)
```

-----

## ✨ この変更の効果

このリファクタリングにより、SOLID原則がさらに高いレベルで満たされました。

1.  **S (単一責任) の達成:**
    `Store`クラスは「注文フローの実行」というビジネスロジックに集中できました。「注文をどう保存するか（リストか？DBか？）」という責任は、`InMemoryOrderRepository`が担当します。`Store`から「変更する理由」が一つ減りました。
2.  **O (オープン・クローズド) の達成:**
    もし将来、「注文をリストではなく、**データベースに保存**したい」という要求が来ても、`Store`クラス（ビジネスロジック）を**一切修正する必要はありません**。
    `DatabaseOrderRepository`という新しいクラスを作り`IOrderRepository`を実装し、`main.py`で`Store`に注入するクラスを差し替えるだけです。
3.  **D (依存性逆転) の徹底:**
    `Store`（上位モジュール）は、`InMemoryOrderRepository`（具象）を知りません。`IOrderRepository`（抽象）にのみ依存しています。`OOP-04`の`Product`と同様に、`Order`の保存についてもDIPを徹底できました。

この「**ビジネスロジック**」と「**データ永続化（インフラストラクチャ）**」の分離こそが、次のステップである**クリーンアーキテクチャ**や**ドメイン駆動設計 (DDD)** の最も核心的なプラクティスです。