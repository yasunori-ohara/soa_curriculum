# OOP-01

手続き型で実装した複数店舗の注文処理機能を、オブジェクト指向プログラミングの原則に則って全面的に書き換えます。

手続き型で露呈した**「複数インスタンスを自然に扱えない」「データとロジックが分離している」**といった限界が、OOPによっていかに解決されるかにご注目ください。

1. [logic.py](http://logic.py/): ビジネスの「概念」であるProductクラスとStoreクラスを定義します。
2. [main.py](http://main.py/): それらのクラスから「実体（インスタンス）」である東京店と大阪店を作成し、注文処理を実行します。

---

### OOPによる改善点

手続き型のコードが抱えていた問題は、OOPの基本原則によって見事に解決されました。

カプセル化:
Productクラスは在庫（_stock）を、Storeクラスは商品リスト（_products）を内部に保持し、外部からの不正なアクセスを防ぎます。データとその操作ロジックが一体化し、安全性が向上しました。

複数インスタンスの生成:
Storeクラスという「設計図」から、tokyo_storeとosaka_storeという、それぞれが独立したデータを持つ「実体（インスタンス）」を簡単に生成できました。これは、手続き型では極めて困難だったことです。

コードの直感性:
logic.process_order("tokyo", ...)という不自然な関数呼び出しは、tokyo_store.process_order(...)という、現実世界のメンタルモデルに近い、非常に直感的で分かりやすいコードに変わりました。

### [logic.py](http://logic.py/)

```bash
import datetime

class Product:
    """
    一つの商品を表現するクラス。
    商品に関するデータ（属性）とロジック（メソッド）をカプセル化する。
    """
    def __init__(self, product_id: str, name: str, price: int, stock: int):
        self.product_id = product_id
        self.name = name
        self.price = price
        # 在庫データは外部から直接変更されないように、アンダースコアで保護する（カプセル化）
        self._stock = stock

    def check_stock(self, quantity: int) -> bool:
        """在庫が十分かを確認するロジック。"""
        return self._stock >= quantity

    def reduce_stock(self, quantity: int):
        """在庫を減らすロジック。"""
        if self.check_stock(quantity):
            self._stock -= quantity
            return True
        return False

    def get_stock(self) -> int:
        """外部が安全に在庫数を取得するためのメソッド。"""
        return self._stock

class Store:
    """
    一つの店舗を表現するクラス。
    店舗が持つべきデータ（商品リスト、注文履歴）と振る舞い（注文処理）をカプセル化する。
    """
    def __init__(self, name: str):
        self.name = name
        self._products = {}  # この店舗が扱う商品オブジェクトを格納
        self._orders = []    # この店舗の注文記録を格納

    def add_product(self, product: Product):
        """この店舗に商品を追加する。"""
        self._products[product.product_id] = product

    def get_product(self, product_id: str) -> Product | None:
        """この店舗から特定の商品を探す。"""
        return self._products.get(product_id)

    def process_order(self, product_id: str, quantity: int):
        """
        注文処理のメインフロー。
        手続き型と違い、引数に store_id は不要。
        このメソッドは、必ず自分自身のインスタンス（店舗）のデータを操作する。
        """
        print(f"\\n--- [{self.name}] 注文処理開始: 商品ID={product_id}, 数量={quantity} ---")

        product = self.get_product(product_id)
        if not product:
            print(f"エラー: [{self.name}] 指定された商品が見つかりません。")
            return

        # Productオブジェクトに在庫の確認を「依頼」する
        if not product.check_stock(quantity):
            print(f"エラー: [{self.name}] {product.name}の在庫が不足しています。")
            return

        # Productオブジェクトに在庫の削減を「依頼」する
        product.reduce_stock(quantity)

        # 注文記録を作成
        order_record = {
            "product_name": product.name,
            "quantity": quantity,
            "total_price": product.price * quantity,
            "order_date": datetime.datetime.now().isoformat()
        }
        self._orders.append(order_record)

        print(f"注文成功: [{self.name}] {product.name}を{quantity}個受け付けました。")

    def get_current_stock_info(self) -> dict:
        """この店舗の現在の在庫情報を取得する。"""
        return {
            p.name: p.get_stock()
            for p_id, p in self._products.items()
        }

```

### [main.py](http://main.py/)

```bash
from logic import Product, Store

if __name__ == "__main__":
    # --- 1. 「設計図」であるクラスから、「実体」であるインスタンスを生成 ---
    # 東京店のインスタンスを作成
    tokyo_store = Store("東京店")
    # 東京店で扱う商品インスタンスを作成し、店舗に追加
    tokyo_store.add_product(Product("p-001", "高機能マウス", 4000, 10))
    tokyo_store.add_product(Product("p-002", "静音キーボード", 6000, 5))
    tokyo_store.add_product(Product("p-003", "24インチモニター", 25000, 3))

    # 大阪店のインスタンスを作成
    osaka_store = Store("大阪店")
    # 大阪店で扱う商品インスタンスを作成し、店舗に追加
    osaka_store.add_product(Product("p-001", "高機能マウス", 4100, 8))
    osaka_store.add_product(Product("p-002", "静音キーボード", 6000, 12))
    osaka_store.add_product(Product("p-003", "24インチモニター", 25500, 5))

    # --- 2. 各店舗のインスタンスに対して、独立して処理を実行 ---
    print("--- 初期在庫 ---")
    print(f"{tokyo_store.name}:", tokyo_store.get_current_stock_info())
    print(f"{osaka_store.name}:", osaka_store.get_current_stock_info())

    # シナリオ1: 東京店で注文
    tokyo_store.process_order("p-001", 3)

    # シナリオ2: 大阪店で注文
    osaka_store.process_order("p-002", 5)

    # シナリオ3: 東京店で在庫不足
    tokyo_store.process_order("p-002", 10)

    # シナリオ4: 大阪店では在庫があるので同じ注文が成功
    # 大阪店のインスタンスは東京店の処理による影響を全く受けない
    osaka_store.process_order("p-002", 10)

    print("\\n--- 最終在庫 ---")
    print(f"{tokyo_store.name}:", tokyo_store.get_current_stock_info())
    print(f"{osaka_store.name}:", osaka_store.get_current_stock_info())

```

### 次への課題 - オブジェクト指向の限界

このOOPによる設計は非常に優れていますが、まだ完璧ではありません。以前の議論で出てきた**「拡張性の低さ」**の問題、つまり「ダウンロード商品」のような新しい種類の概念にどう対応するか、という課題が残っています。

現在のProductクラスは「在庫を持つ物理的な商品」であることだけを想定しています。
この問題をエレガントに解決するためには、OOPが提供するさらに強力な武器、継承とポリモーフィズムが必要となります。これが次のステップです。