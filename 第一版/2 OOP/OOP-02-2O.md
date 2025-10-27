# OOP-02-2O

## O: オープン・クローズドの原則 (Open/Closed Principle)

### 「クラスは拡張に対して開いており、修正に対して閉じているべき」

### 評価：半分だけ適用できています

---

### `Product`クラス：閉じられていない点（Closed）があります

もし「ダウンロード商品」のような、在庫の概念がない新しい**商品の種類**を追加したい場合、`Product`クラスの`check_stock`や`reduce_stock`メソッドを修正する必要があります。<br>
これは「修正に対して閉じている」という原則に**違反**してしまいます。

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
        """ダウンロード商品は常に在庫は十分。"""
        return self._stock >= quantity

    def reduce_stock(self, quantity: int):
        """在庫を減らすロジック。"""
        """ダウンロード製品は在庫は減らない"""
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

新しい**商品インスタンス**を追加（拡張）する際に、`Store`クラスのコードを修正する必要はありません。<br>
これは原則に沿っています。

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

### **次の課題**<br>

この「閉じられていない」点を解決することこそが、次のステップで**継承**と**ポリモーフィズム**を学ぶ最大の動機となります。