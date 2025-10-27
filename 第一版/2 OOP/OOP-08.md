# OOP-08

## Factory Method パターンの適用

SOLID原則を満たした現在のコードにFactory Method パターンを適用します。このパターンを導入することで、オブジェクトを生成する部分の関心事を分離し、コードをさらに柔軟で保守性の高いものに進化させます。

---

現在の`main.py`は、`PhysicalProduct(...)`や`DigitalProduct(...)`といった具体的なクラス名を知っており、それらを直接インスタンス化しています。これは、もし将来「サブスクリプション商品」のような新しい商品クラスが追加された場合、`main.py`のコード自体を修正する必要があることを意味します。

### **Factory Method** パターンによる解決

この問題を解決するために、商品のインスタンスを生成する専門の「工場（Factory）」クラスを導入します。`main.py`のようなクライアントコードは、この工場に「物理商品をください」と依頼するだけで済み、具体的なクラス名を知る必要がなくなります。

---

### 実装コード

これから、以下の2つのファイルを修正・更新します。

1. [logic.py](http://logic.py/): 新しく`ProductFactory`クラスを追加します。
2. [main.py](http://main.py/): 具体的なクラスの代わりに`ProductFactory`を使って商品を生成するように変更します。

### logic.c

```python
import datetime
from abc import ABC, abstractmethod

class Product(ABC):
    """
    商品の「抽象的な概念」を定義するインターフェース（抽象基底クラス）。
    """
    def __init__(self, product_id: str, name: str, price: int):
        self.product_id = product_id
        self.name = name
        self.price = price

    @abstractmethod
    def check_stock(self, quantity: int) -> bool:
        pass

    @abstractmethod
    def reduce_stock(self, quantity: int):
        pass

    @abstractmethod
    def get_stock(self) -> int | str:
        pass

class PhysicalProduct(Product):
    """
    物理的な商品を表現する「具体的な実装」クラス。
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

class DigitalProduct(Product):
    """
    ダウンロード商品を表現する「具体的な実装」クラス。
    """
    def check_stock(self, quantity: int) -> bool:
        return True

    def reduce_stock(self, quantity: int):
        pass

    def get_stock(self) -> str:
        return "N/A (Downloadable)"

class Store:
    """
    店舗を表すクラス。（このクラスは変更なし）
    """
    def __init__(self, name: str):
        self.name = name
        self._products = {}
        self._orders = []

    def add_product(self, product: Product):
        self._products[product.product_id] = product

    def get_product(self, product_id: str) -> Product | None:
        return self._products.get(product_id)

    def process_order(self, product_id: str, quantity: int):
        # (処理は省略)
        print(f"\\n--- [{self.name}] 注文処理開始: 商品ID={product_id}, 数量={quantity} ---")
        product = self.get_product(product_id)
        if not product:
            print(f"エラー: [{self.name}] 指定された商品が見つかりません。")
            return
        if not product.check_stock(quantity):
            print(f"エラー: [{self.name}] {product.name}は購入できません。")
            return
        product.reduce_stock(quantity)
        order_record = { "product_name": product.name, "quantity": quantity, "total_price": product.price * quantity, "order_date": datetime.datetime.now().isoformat() }
        self._orders.append(order_record)
        print(f"注文成功: [{self.name}] {product.name}を{quantity}個受け付けました。")

    def get_current_stock_info(self) -> dict:
        stock_info = {}
        for p_id, p in self._products.items():
            stock_info[p.name] = p.get_stock()
        return stock_info

class ProductFactory:
    """
    【新しく追加】商品を生成する専門の「工場」クラス。
    どの種類の製品を作るかの知識を、このクラスに集約する。
    """
    @staticmethod
    def create_product(product_type: str, **kwargs) -> Product:
        """
        指定された種類の商品インスタンスを生成して返す。
        """
        if product_type == "physical":
            return PhysicalProduct(**kwargs)
        elif product_type == "digital":
            return DigitalProduct(**kwargs)
        # 将来「サブスクリプション商品」が追加されたら、ここにelifを追加するだけ
        # elif product_type == "subscription":
        #     return SubscriptionProduct(**kwargs)
        else:
            raise ValueError(f"未知の商品種別です: {product_type}")

```

### main.c

```python
# 具体的なクラス(PhysicalProduct, DigitalProduct)の代わりに、
# 工場(ProductFactory)をインポートする
from logic import ProductFactory, Store

if __name__ == "__main__":
    tokyo_store = Store("東京店")

    # ProductFactoryを使って商品を生成する
    # main.pyは、PhysicalProductという具体的なクラス名を知らなくて済む
    mouse = ProductFactory.create_product(
        "physical",
        product_id="p-001", name="高機能マウス", price=4000, stock=10
    )
    keyboard = ProductFactory.create_product(
        "physical",
        product_id="p-002", name="静音キーボード", price=6000, stock=5
    )
    ebook = ProductFactory.create_product(
        "digital",
        product_id="d-001", name="デザインソフト eBook", price=8000
    )

    tokyo_store.add_product(mouse)
    tokyo_store.add_product(keyboard)
    tokyo_store.add_product(ebook)

    osaka_store = Store("大阪店")
    osaka_store.add_product(ProductFactory.create_product(
        "physical",
        product_id="p-001", name="高機能マウス", price=4100, stock=8
    ))
    osaka_store.add_product(ProductFactory.create_product(
        "digital",
        product_id="d-002", name="プログラミング講座 動画", price=12000
    ))

    # --- 以下のコードは一切変更なし！ ---

    print("--- 初期在庫 ---")
    print(f"{tokyo_store.name}:", tokyo_store.get_current_stock_info())
    print(f"{osaka_store.name}:", osaka_store.get_current_stock_info())

    tokyo_store.process_order("p-001", 3)
    tokyo_store.process_order("d-001", 50)

    print("\\n--- 最終在庫 ---")
    print(f"{tokyo_store.name}:", tokyo_store.get_current_stock_info())
    print(f"{osaka_store.name}:", osaka_store.get_current_stock_info())

```

以前の`main.py`が具体的な商品クラス（`PhysicalProduct`など）を知っていたのに対し、今回の`main.py`は工場（`ProductFactory`）に依頼するだけで済むようになり、より疎結合になっている点にご注目ください。

### このリファクタリングの価値

1. 関心事の分離:
「商品を使う側（`main.py`）」と「商品を作る側（`ProductFactory`）」の関心事を明確に分離できました。
2. 保守性の向上:
新しい商品クラスを追加する際、修正が必要なのは`ProductFactory`クラスだけです。`main.py`のようなクライアントコードは一切変更する必要がなくなり、オープン・クローズドの原則をさらに高いレベルで実現できました。
3. 柔軟性の向上:
`ProductFactory`の`create_product`メソッドを改良して、設定ファイルやデータベースから商品情報を読み込んで自動的に適切なインスタンスを生成する、といった高度な機能に拡張することも容易になります。

### 次は

これでOOPの基礎から応用までを網羅し、「良いOOPとは何か」を深く学んだ上で、次のクリーンアーキテクチャに進む準備が整いました。