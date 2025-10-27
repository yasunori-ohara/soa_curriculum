# OOP-03

## 依存性逆転の原則を適用したリファクタリング

現在のコードの問題点は、StoreクラスがProductという具体的なクラスに直接依存していることです。これを、Productという**抽象的な概念（インターフェース）**に依存するように変更します。

## 変更前後の依存関係

変更前: Store → Product (具体的なクラス)
変更後: Store → Product (抽象クラス) ← PhysicalProduct (具体的なクラス)

## 実装コード

以下の2つのファイルを修正します。

1. [logic.py](http://logic.py/): Productを抽象基底クラスとして定義し、元のProductをPhysicalProductと改名してそれを継承させます。Storeクラスが抽象に依存するように修正します。
2. [main.py](http://main.py/): 変更はほとんどありませんが、PhysicalProductをインスタンス化するように修正します。

---

### [logic.py](http://logic.py/)

```bash
import datetime
from abc import ABC, abstractmethod

class Product(ABC):
    """
    商品の「抽象的な概念」を定義するインターフェース（抽象基底クラス）。
    全ての商品は、このクラスが定義する「契約」を守らなければならない。
    """
    def __init__(self, product_id: str, name: str, price: int):
        self.product_id = product_id
        self.name = name
        self.price = price

    @abstractmethod
    def check_stock(self, quantity: int) -> bool:
        """在庫を確認する、という振る舞いを約束させる。"""
        pass

    @abstractmethod
    def reduce_stock(self, quantity: int):
        """在庫を減らす、という振る舞いを約束させる。"""
        pass

    @abstractmethod
    def get_stock(self) -> int:
        """在庫数を取得する、という振る舞いを約束させる。"""
        pass

class PhysicalProduct(Product):
    """
    物理的な商品を表現する「具体的な実装」クラス。
    Productという抽象的な概念を継承し、具体的な振る舞いを実装する。
    """
    def __init__(self, product_id: str, name: str, price: int, stock: int):
        super().__init__(product_id, name, price)
        self._stock = stock

    def check_stock(self, quantity: int) -> bool:
        return self._stock >= quantity

    def reduce_stock(self, quantity: int):
        if self.check_stock(quantity):
            self._stock -= quantity
            return True
        return False

    def get_stock(self) -> int:
        return self._stock

class Store:
    """
    店舗を表すクラス。
    具体的な PhysicalProduct ではなく、抽象的な Product に依存する。
    """
    def __init__(self, name: str):
        self.name = name
        self._products = {}
        self._orders = []

    def add_product(self, product: Product): # 引数の型ヒントが抽象クラスに
        self._products[product.product_id] = product

    def get_product(self, product_id: str) -> Product | None: # 戻り値の型ヒントが抽象クラスに
        return self._products.get(product_id)

    def process_order(self, product_id: str, quantity: int):
        print(f"\\n--- [{self.name}] 注文処理開始: 商品ID={product_id}, 数量={quantity} ---")

        # このメソッド内のロジックは一切変更なし！
        # なぜなら、最初から product.check_stock() のように
        # インターフェースを通じてメソッドを呼び出していたから。
        product = self.get_product(product_id)
        if not product:
            print(f"エラー: [{self.name}] 指定された商品が見つかりません。")
            return

        if not product.check_stock(quantity):
            print(f"エラー: [{self.name}] {product.name}の在庫が不足しています。")
            return

        product.reduce_stock(quantity)

        order_record = {
            "product_name": product.name,
            "quantity": quantity,
            "total_price": product.price * quantity,
            "order_date": datetime.datetime.now().isoformat()
        }
        self._orders.append(order_record)
        print(f"注文成功: [{self.name}] {product.name}を{quantity}個受け付けました。")

    def get_current_stock_info(self) -> dict:
        stock_info = {}
        for p_id, p in self._products.items():
            # PhysicalProductにしかget_stock()がない場合を考慮する必要があるが、
            # 今回はProductインターフェースにget_stock()が含まれるので安全。
            stock_info[p.name] = p.get_stock()
        return stock_info

```

### [main.py](http://main.py/)

```bash
# Product ではなく、具体的な PhysicalProduct をインポートする
from logic import PhysicalProduct, Store

if __name__ == "__main__":
    tokyo_store = Store("東京店")
    # インスタンス化するのは具体的なクラス
    tokyo_store.add_product(PhysicalProduct("p-001", "高機能マウス", 4000, 10))
    tokyo_store.add_product(PhysicalProduct("p-002", "静音キーボード", 6000, 5))
    tokyo_store.add_product(PhysicalProduct("p-003", "24インチモニター", 25000, 3))

    osaka_store = Store("大阪店")
    osaka_store.add_product(PhysicalProduct("p-001", "高機能マウス", 4100, 8))
    osaka_store.add_product(PhysicalProduct("p-002", "静音キーボード", 6000, 12))
    osaka_store.add_product(PhysicalProduct("p-003", "24インチモニター", 25500, 5))

    # --- 以下のコードは一切変更なし！ ---
    # Storeクラスは相手がPhysicalProductであることを知らないため、
    # 以前のコードがそのまま動作する。

    print("--- 初期在庫 ---")
    print(f"{tokyo_store.name}:", tokyo_store.get_current_stock_info())
    print(f"{osaka_store.name}:", osaka_store.get_current_stock_info())

    tokyo_store.process_order("p-001", 3)
    osaka_store.process_order("p-002", 5)
    tokyo_store.process_order("p-002", 10)
    osaka_store.process_order("p-002", 10)

    print("\\n--- 最終在庫 ---")
    print(f"{tokyo_store.name}:", tokyo_store.get_current_stock_info())
    print(f"{osaka_store.name}:", osaka_store.get_current_stock_info())

```

## この変更の効果

この変更によって、Storeクラスは**「商品とは、在庫を確認でき、在庫を減らせ、在庫数を取得できるものである」という抽象的な約束事**にのみ依存するようになりました。

これで、『ダウンロード商品』を追加する準備が完璧に整いました。
新しいDigitalProductクラスを作り、Product抽象クラスを継承して、3つの約束されたメソッドを実装するだけで、Storeクラスには一切手を加える必要がありません。

これこそが、依存性逆転の原則がもたらす真の力であり、オープン・クローズドの原則を実現するための土台なのです。