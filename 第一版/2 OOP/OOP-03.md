# OOP-03 : 依存性逆転の原則(DIP)によるリファクタリング

`OOP-02` の評価で、`Store` クラスが `Product` という\*\*具体的なクラス（下位モジュール）\*\*に直接依存しているというDIP違反が見つかりました。

この章では、この問題をリファクタリングします。`Store` クラスが依存する相手を、具体的な `Product` クラスから、\*\*「商品とはこういうものだ」という抽象的な概念（インターフェース）\*\*に変更します。

## 🎯 この章のゴール

  * 依存性逆転の原則 (DIP) をコードで実践する。
  * Pythonの `abc` (抽象基底クラス) モジュールを使い、「インターフェース（契約）」を定義する方法を学ぶ。
  * 「具象」ではなく「抽象」に依存することのメリット（＝ `Store` 側のコード変更が不要になる）を体感する。

-----

## 🔗 変更前後の依存関係

このリファクタリングで、依存関係の流れが「逆転」します。

**変更前 (DIP違反):**
`Store` (上位) → `Product` (下位・具象)

**変更後 (DIP適用):**
`Store` (上位) → `Product` (抽象) ← `PhysicalProduct` (下位・具象)

`Store` と `PhysicalProduct`（元の `Product`）は、お互いを直接知らず、中間の「抽象 (`Product`)」だけを介して接続されるようになります。

-----

## 💻 `logic.py` : 「抽象」の定義と「具象」の実装

`logic.py` に、Pythonで「インターフェース」の役割を果たす「抽象基底クラス」を導入します。

```python:logic.py
import datetime
# Pythonで「抽象クラス」を作るための仕組みをインポート
from abc import ABC, abstractmethod

class Product(ABC):
    """
    商品の「抽象的な概念」を定義するインターフェース（抽象基底クラス）。
    Storeクラスはこちらに依存する。
    
    ABC (Abstract Base Class) を継承し、@abstractmethod を使うことで、
    「このクラスを継承する子クラスは、必ずこれらのメソッドを実装しなさい」
    という「契約」を強制できる。
    """
    def __init__(self, product_id: str, name: str, price: int):
        # 抽象クラスでも、共通の属性は定義できる
        self.product_id = product_id
        self.name = name
        self.price = price

    @abstractmethod
    def check_stock(self, quantity: int) -> bool:
        """【契約】在庫を確認するという振る舞いを約束させる。"""
        # 抽象メソッドは中身（実装）を持たない
        pass

    @abstractmethod
    def reduce_stock(self, quantity: int):
        """【契約】在庫を減らすという振る舞いを約束させる。"""
        pass

    @abstractmethod
    def get_stock(self) -> int:
        """【契約】在庫数を取得するという振る舞いを約束させる。"""
        pass

class PhysicalProduct(Product):
    """
    物理的な商品を表現する「具体的な実装」クラス。
    OOP-01の「Product」クラスがこれにあたる。
    
    Product(抽象)を継承し、「契約」で約束した全ての抽象メソッドを
    具体的に実装（オーバーライド）する。
    """
    def __init__(self, product_id: str, name: str, price: int, stock: int):
        # 親クラス(Product)の __init__ を呼び出す
        super().__init__(product_id, name, price)
        # PhysicalProduct 固有の属性
        self._stock = stock

    # --- Product(抽象)で「契約」したメソッドの実装 ---
    
    def check_stock(self, quantity: int) -> bool:
        """【実装】物理在庫が十分か確認する。"""
        return self._stock >= quantity

    def reduce_stock(self, quantity: int):
        """【実装】物理在庫を減らす。"""
        if self.check_stock(quantity):
            self._stock -= quantity
            print(f"在庫更新: {self.name}の在庫が{self._stock}になりました。")
            return True
        return False

    def get_stock(self) -> int:
        """【実装】物理在庫の数を返す。"""
        return self._stock

class Store:
    """
    店舗を表すクラス。
    DIP適用により、具体的な PhysicalProduct ではなく、
    抽象的な Product にのみ依存する。
    """
    def __init__(self, name: str):
        self.name = name
        # この辞書が保持するのは「Product(抽象)」を満たすオブジェクト
        self._products: dict[str, Product] = {} # 型ヒントを抽象クラスに
        self._orders = []

    def add_product(self, product: Product): # 引数の型ヒントが抽象クラスに
        """
        渡されるのが PhysicalProduct でも、将来作る DigitalProduct でも、
        「Product(抽象)」の契約さえ守っていれば受け入れ可能。
        """
        self._products[product.product_id] = product

    def _find_product(self, product_id: str) -> Product | None: # 戻り値も抽象クラス
        return self._products.get(product_id)

    def process_order(self, product_id: str, quantity: int):
        """
        このメソッド内のロジックは OOP-01 から「一切変更なし」。
        
        なぜなら、最初から product.check_stock() のように、
        具象（実装）を直接触らず、インターフェース（メソッド呼び出し）
        を通じて操作していたから。
        """
        print(f"\n--- [{self.name}] 注文処理開始: 商品ID={product_id}, 数量={quantity} ---")

        product = self._find_product(product_id)
        if not product:
            print(f"エラー: [{self.name}] 指定された商品が見つかりません。")
            return

        # product が PhysicalProduct かどうかを Store は知らない。
        # ただ「Product(抽象)」の契約通り check_stock を呼び出すだけ。
        if not product.check_stock(quantity):
            print(f"エラー: [{self.name}] {product.name}の在庫が不足しています。")
            return

        # 同様に、reduce_stock を呼び出すだけ。
        product.reduce_stock(quantity)

        # 注文記録の作成（※これはS違反の可能性ありとしてOOP-02で指摘済み）
        order_record = {
            "product_name": product.name,
            "quantity": quantity,
            "total_price": product.price * quantity,
            "order_date": datetime.datetime.now().isoformat()
        }
        self._orders.append(order_record)

        print(f"注文成功: [{self.name}] {product.name}を{quantity}個受け付けました。")

    def get_current_stock_info(self) -> dict:
        """店舗の現在の在庫情報を取得する。"""
        stock_info = {}
        for p_id, p in self._products.items():
            # p が何の具象クラスか知らなくても、
            # 「Product(抽象)」が get_stock() を契約しているので安全に呼び出せる。
            stock_info[p.name] = p.get_stock()
        return stock_info
```

-----

## 🏛️ `main.py` : 利用者側の変更

`Store` クラスを利用する `main.py` 側も、`Store` が抽象に依存するようになった恩恵を受けます。

```python:main.py
# インポートするのが Product(抽象) ではなく、
# 「具体的な実装」である PhysicalProduct に変わる
from logic import PhysicalProduct, Store

if __name__ == "__main__":
    # Storeインスタンスの作成（変更なし）
    tokyo_store = Store("東京店")
    
    # Storeに「注入」するのは、「具体的な」PhysicalProduct のインスタンス
    tokyo_store.add_product(PhysicalProduct("p-001", "高機能マウス", 4000, 10))
    tokyo_store.add_product(PhysicalProduct("p-002", "静音キーボード", 6000, 5))
    tokyo_store.add_product(PhysicalProduct("p-003", "24インチモニター", 25000, 3))

    # 大阪店も同様
    osaka_store = Store("大阪店")
    osaka_store.add_product(PhysicalProduct("p-001", "高機能マウス", 4100, 8))
    osaka_store.add_product(PhysicalProduct("p-002", "静音キーボード", 6000, 12))
    osaka_store.add_product(PhysicalProduct("p-003", "25インチモニター", 25500, 5))

    # --- 以下の「Storeを操作するコード」は一切変更なし！ ---
    # Storeクラスは相手が PhysicalProduct であることを知らず、
    # 「Product(抽象)」として扱っているため、以前のコードがそのまま動作する。

    print("--- 初期在庫 ---")
    print(f"{tokyo_store.name}:", tokyo_store.get_current_stock_info())
    print(f"{osaka_store.name}:", osaka_store.get_current_stock_info())

    # tokyo_store.process_order(...) は、
    # 内部で操作するのが PhysicalProduct だとは知らない
    tokyo_store.process_order("p-001", 3)
    osaka_store.process_order("p-002", 5)
    tokyo_store.process_order("p-002", 10)
    osaka_store.process_order("p-002", 10)

    print("\n--- 最終在庫 ---")
    print(f"{tokyo_store.name}:", tokyo_store.get_current_stock_info())
    print(f"{osaka_store.name}:", osaka_store.get_current_stock_info())
```

-----

## ✨ この変更の効果

このリファクタリングによって、`Store` クラスは\*\*「商品とは、`check_stock()`、`reduce_stock()`、`get_stock()` ができるものである」という抽象的な約束事\*\*にのみ依存するようになりました。

`Store` は、相手が「物理商品（`PhysicalProduct`）」であることを知らなくても動作します。

これで、`OOP-02` で指摘されたもう一つの課題、\*\*O（オープン・クローズドの原則）\*\*違反を解決する準備が完璧に整いました。

次の章では、新しい「ダウンロード商品（`DigitalProduct`）」クラスを作ります。そのクラスが `Product`（抽象）の契約さえ守れば、**`Store` クラスには一切手を加えることなく（＝修正に対して閉じたまま）**、新機能を追加（＝拡張に対して開く）できることを見ていきましょう。