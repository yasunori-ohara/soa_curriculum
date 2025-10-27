# OOP-02-1S

## S: 単一責任の原則 (Single Responsibility Principle)

### 「クラスを変更する理由は、一つだけであるべき」

### 評価：適用できています。

---

### `Product`クラス

「商品そのものの情報とルール（在庫管理など）」にのみ責任を持っています。<br>
価格の計算方法や在庫ルールの変更があれば`Product`クラスを修正します。

```bash
class Product:
    """
    一つの商品を表現するクラス。
    商品に関するデータ（属性）とロジック（メソッド）をカプセル化する。
    """
    def __init__(self, product_id: str, name: str, price: int, stock: int):
        """ 省略 """

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

```

---

### `Store`クラス

「商品を管理し、注文を受け付ける」という店舗としてのフローにのみ責任を持っています。<br>
注文処理の流れ（例：注文後の通知を追加する）が変われば`Store`クラスを修正します。

```bash
class Store:
    """
    一つの店舗を表現するクラス。
    店舗が持つべきデータ（商品リスト、注文履歴）と振る舞い（注文処理）をカプセル化する。
    """
    def __init__(self, name: str):
        """ 省略 """

    def add_product(self, product: Product):
        """この店舗に商品を追加する。"""
        self._products[product.product_id] = product

    def get_product(self, product_id: str) -> Product | None:
        """この店舗から特定の商品を探す。"""
        return self._products.get(product_id)

    def process_order(self, product_id: str, quantity: int):
        """ 省略 """

    def get_current_stock_info(self) -> dict:
        """この店舗の現在の在庫情報を取得する。"""
        return {
            p.name: p.get_stock()
            for p_id, p in self._products.items()
        }

```