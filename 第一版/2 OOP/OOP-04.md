# OOP-04

## ダウンロード商品への対応

在庫の概念がない「ダウンロード商品」という新しい種類のビジネス要件に対応します。この変更によって、OOPの継承とポリモーフィズムがいかに強力であるかを明確に示します。

最大のポイント: `Store`クラスのコードには、一切手を加えません。新しい機能は、新しいクラスを**追加（拡張）**するだけで実現します。

これから、以下の2つのファイルを修正・更新します。

1. [logic.py](http://logic.py/): `Product`インターフェースを実装した、新しい`DigitalProduct`クラスを追加します。
2. [main.py](http://main.py/): 新しい`DigitalProduct`のインスタンスを生成し、店舗に追加して注文処理を実行します。

---

[logic.py](http://logic.py/)

```bash
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
        """在庫を確認する、という振る舞いを約束させる。"""
        pass

    @abstractmethod
    def reduce_stock(self, quantity: int):
        """在庫を減らす、という振る-舞いを約束させる。"""
        pass

    @abstractmethod
    def get_stock(self) -> int | str:
        """在庫数を取得する、という振る舞いを約束させる。"""
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
    【新しく追加】ダウンロード商品を表現する「具体的な実装」クラス。
    Productインターフェースを実装するが、振る舞いは全く異なる。
    """
    def check_stock(self, quantity: int) -> bool:
        # ダウンロード商品なので、在庫は常に無限
        return True

    def reduce_stock(self, quantity: int):
        # 在庫という概念がないので、何もしない
        pass

    def get_stock(self) -> str:
        # 在庫数ではなく、分かりやすい文字列を返す
        return "N/A (Downloadable)"

class Store:
    """
    店舗を表すクラス。
    このクラスのコードは一切変更されていない！
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
        print(f"\\n--- [{self.name}] 注文処理開始: 商品ID={product_id}, 数量={quantity} ---")

        product = self.get_product(product_id)
        if not product:
            print(f"エラー: [{self.name}] 指定された商品が見つかりません。")
            return

        # ポリモーフィズム：
        # productがPhysicalProductかDigitalProductかを全く意識せずに
        # check_stockメソッドを呼び出せる。
        if not product.check_stock(quantity):
            print(f"エラー: [{self.name}] {product.name}は購入できません。")
            return

        # ポリモーフィズム：
        # 相手に応じて、在庫を減らす処理 or 何もしない処理が自動で呼び出される。
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
            stock_info[p.name] = p.get_stock()
        return stock_info

```

[main.py](http://main.py/)

```python
# DigitalProduct を追加でインポートする
from logic import PhysicalProduct, DigitalProduct, Store

if __name__ == "__main__":
    tokyo_store = Store("東京店")
    tokyo_store.add_product(PhysicalProduct("p-001", "高機能マウス", 4000, 10))
    tokyo_store.add_product(PhysicalProduct("p-002", "静音キーボード", 6000, 5))
    # 東京店に新しい種類の商品（ダウンロード商品）を追加
    tokyo_store.add_product(DigitalProduct("d-001", "デザインソフト eBook", 8000))

    osaka_store = Store("大阪店")
    osaka_store.add_product(PhysicalProduct("p-001", "高機能マウス", 4100, 8))
    # 大阪店にもダウンロード商品を追加
    osaka_store.add_product(DigitalProduct("d-002", "プログラミング講座 動画", 12000))

    print("--- 初期在庫 ---")
    print(f"{tokyo_store.name}:", tokyo_store.get_current_stock_info())
    print(f"{osaka_store.name}:", osaka_store.get_current_stock_info())

    # --- 注文処理 ---

    # シナリオ1: 東京店で物理商品の在庫が足りる注文
    tokyo_store.process_order("p-001", 3)

    # シナリオ2: 東京店で物理商品の在庫が足りない注文
    tokyo_store.process_order("p-002", 10)

    # シナリオ3: 東京店でダウンロード商品を注文
    # Storeクラスは相手がダウンロード商品であることを知らなくても、正しく処理できる
    tokyo_store.process_order("d-001", 50)

    # シナリオ4: 大阪店でダウンロード商品を注文
    osaka_store.process_order("d-002", 100)

    print("\\n--- 最終在庫 ---")
    print(f"{tokyo_store.name}:", tokyo_store.get_current_stock_info())
    print(f"{osaka_store.name}:", osaka_store.get_current_stock_info())

```

### OOPによる真の改善点：「ポリモーフィズム」

このリファクタリングで、オブジェクト指向の真の力である**ポリモーフィズム（Polymorphism / 多態性）**が明確に示されました。

`Store`クラスの`process_order`メソッドは、`product`という変数が`PhysicalProduct`なのか`DigitalProduct`なのかを全く気にする必要がありません。ただ`product.check_stock()`と命令するだけで、相手（オブジェクト）が自分自身の種類に応じて、適切な振る舞い（在庫を本当に確認する or 常にOKと返す）を自動的に実行してくれます。

これにより、オープン・クローズドの原則が完全に満たされ、`Store`クラスを一切修正することなく、新しい機能（商品種別）を安全に拡張できました。

これこそが、手続き型プログラミングでは決して得られなかった、オブジェクト指向がもたらす柔軟性と保守性の核心です。