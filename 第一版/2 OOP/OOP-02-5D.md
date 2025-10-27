# OOP-02-5D

## D: 依存性逆転の原則 (Dependency Inversion Principle)

### 「上位モジュールは下位モジュールに依存すべきではない。両方とも抽象に依存すべき」

### 評価：部分的に適用できています.

---

`Store`クラス（上位モジュール）は、自身で`Product`インスタンス（下位モジュール）を作成していません。<br>
外部から`add_product`メソッドを通じて受け取っています（これは**依存性の注入(DI)**の第一歩です）。

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

```

---

しかし、`Store`クラスはまだ`Product`という**具体的なクラス**に依存しています。

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

---

次のステップで、`Product`という「抽象的な概念（抽象基底クラス）」を定義し、`Store`がそれに依存するように変更することで、この原則をさらに高いレベルで満たすことになります。

---

このように、現在のコードはSOLID原則の良い土台となっており、同時にいくつかの原則（特にOとD）を完全に満たすためには、さらなる進化が必要であることを示唆しています。