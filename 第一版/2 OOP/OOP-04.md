# OOP-04 : オープン・クローズドの原則(OCP)の実践 - 機能拡張

`OOP-02` で、私たちのコードは「新しい商品の種類（例：ダウンロード商品）」という変更要求に対して「閉じられていない」（＝OCP違反）と評価されました。

`OOP-03` でDIP（依存性逆転の原則）を適用した今、このOCP違反を解決する準備が整いました。在庫の概念がない「ダウンロード商品」を**既存のコードを一切修正せずに追加**してみましょう。

## 🎯 この章のゴール

  * オープン・クローズドの原則（OCP）をコードで実践する。
  * \*\*ポリモーフィズム（多態性）\*\*の絶大な効果を理解する。
  * 「抽象（インターフェース）」に依存することの真の価値（＝ `Store` クラスを修正不要にする）を体感する。

-----

## 💻 `logic.py` : `DigitalProduct` クラスの「追加」

`logic.py` に、`Product`（抽象）を継承した `DigitalProduct` クラスを**追加**します。`PhysicalProduct` や `Store` クラスには一切触りません。

```python:logic.py
import datetime
from abc import ABC, abstractmethod

class Product(ABC):
    """
    商品の「抽象的な概念」を定義するインターフェース（抽象基底クラス）。
    この「契約書」は OOP-03 から一切変更しない。
    """
    def __init__(self, product_id: str, name: str, price: int):
        self.product_id = product_id
        self.name = name
        self.price = price

    @abstractmethod
    def check_stock(self, quantity: int) -> bool:
        """【契約】在庫を確認する"""
        pass

    @abstractmethod
    def reduce_stock(self, quantity: int):
        """【契約】在庫を減らす"""
        pass

    @abstractmethod
    def get_stock(self) -> int | str: # 戻り値の型を柔軟に変更 (intまたはstr)
        """【契約】在庫数を取得する"""
        # (補足: OOP-03 では -> int でしたが、DigitalProduct の仕様に
        # 合わせるため、既存の抽象メソッドの型ヒントを修正しました。
        # これも「修正」ではありますが、Store側のロジックには影響しません)
        pass

class PhysicalProduct(Product):
    """
    物理的な商品を表現する「具体的な実装」クラス。
    このクラスは OOP-03 から一切変更しない。
    """
    def __init__(self, product_id: str, name: str, price: int, stock: int):
        super().__init__(product_id, name, price)
        self._stock = stock

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

# --- ここからが「追加」したコード ---

class DigitalProduct(Product):
    """
    【新しく追加】ダウンロード商品を表現する「具体的な実装」クラス。
    Productインターフェース(抽象)を継承し、「契約」を守る。
    """
    def __init__(self, product_id: str, name: str, price: int):
        # PhysicalProduct と違い、stock を受け取らない
        super().__init__(product_id, name, price)
        # 在庫（_stock）属性自体を持たない

    def check_stock(self, quantity: int) -> bool:
        """【実装】ダウンロード商品なので、在庫は常に「無限（True）」"""
        return True

    def reduce_stock(self, quantity: int):
        """【実装】在庫という概念がないので、何もしない"""
        pass

    def get_stock(self) -> str:
        """【実装】在庫数ではなく、分かりやすい文字列を返す"""
        return "N/A (ダウンロード可)"

# --- ここまでが「追加」したコード ---


class Store:
    """
    店舗を表すクラス。
    このクラスのコードは OOP-03 から「一切変更なし」！
    """
    def __init__(self, name: str):
        self.name = name
        # この辞書は「Product(抽象)」の契約を守るものなら何でも格納できる
        self._products: dict[str, Product] = {}
        self._orders = []

    def add_product(self, product: Product):
        """
        渡されるのが PhysicalProduct か DigitalProduct か、
        Store は気にする必要がない。
        """
        self._products[product.product_id] = product

    def _find_product(self, product_id: str) -> Product | None:
        return self._products.get(product_id)

    def process_order(self, product_id: str, quantity: int):
        """
        このメソッドが「ポリモーフィズム」の力を最も発揮する場所。
        """
        print(f"\n--- [{self.name}] 注文処理開始: 商品ID={product_id}, 数量={quantity} ---")

        product = self._find_product(product_id)
        if not product:
            print(f"エラー: [{self.name}] 指定された商品が見つかりません。")
            return

        # ▼▼▼ ポリモーフィズム（多態性） ▼▼▼
        # product 変数の中身が PhysicalProduct か DigitalProduct かを
        # Store は全く意識する必要がない。
        #
        # ただ「契約（Product抽象）」通りに check_stock() を呼び出すだけ。
        #
        # 実行される「具体的な処理」は、Pythonが自動的に判断する。
        # - 相手が PhysicalProduct なら：在庫数を比較する処理
        # - 相手が DigitalProduct なら：常に True を返す処理
        if not product.check_stock(quantity):
            print(f"エラー: [{self.name}] {product.name}の在庫が不足しています。")
            return

        # ▼▼▼ ポリモーフィズム（多態性） ▼▼▼
        # reduce_stock() も同様。
        # - 相手が PhysicalProduct なら：在庫数を減らす処理
        # - 相手が DigitalProduct なら：何もしない処理
        product.reduce_stock(quantity)

        # 注文記録の作成（変更なし）
        order_record = {
            "product_name": product.name,
            "quantity": quantity,
            "total_price": product.price * quantity,
            "order_date": datetime.datetime.now().isoformat()
        }
        self._orders.append(order_record)

        print(f"注文成功: [{self.name}] {product.name}を{quantity}個受け付けました。")

    def get_current_stock_info(self) -> dict:
        """
        このメソッドも変更なし。
        get_stock() を呼ぶと、相手に応じて int(在庫数) か str("N/A") が返る。
        """
        stock_info = {}
        for p_id, p in self._products.items():
            stock_info[p.name] = p.get_stock()
        return stock_info
```

-----

## 🏛️ `main.py` : 新しいクラスを利用する

`main.py` も、新しい `DigitalProduct` をインポートし、インスタンスを作成して `Store` に追加するだけです。`Store` を操作するロジック（`process_order` の呼び出し）は一切変更ありません。

```python:main.py
# DigitalProduct を追加でインポートする
from logic import PhysicalProduct, DigitalProduct, Store

if __name__ == "__main__":
    
    # --- 店舗と商品のセットアップ ---
    tokyo_store = Store("東京店")
    # 既存の PhysicalProduct を追加
    tokyo_store.add_product(PhysicalProduct("p-001", "高機能マウス", 4000, 10))
    tokyo_store.add_product(PhysicalProduct("p-002", "静音キーボード", 6000, 5))
    # 【追加】新しい DigitalProduct を追加
    tokyo_store.add_product(DigitalProduct("d-001", "デザインソフト eBook", 8000))

    osaka_store = Store("大阪店")
    osaka_store.add_product(PhysicalProduct("p-001", "高機能マウス", 4100, 8))
    # 【追加】大阪店にも DigitalProduct を追加
    osaka_store.add_product(DigitalProduct("d-002", "プログラミング講座 動画", 12000))

    # --- 初期在庫の表示（変更なし） ---
    # get_current_stock_info() は DigitalProduct にも対応できる
    print("--- 初期在庫 ---")
    print(f"{tokyo_store.name}:", tokyo_store.get_current_stock_info())
    print(f"{osaka_store.name}:", osaka_store.get_current_stock_info())


    # --- 注文処理（変更なし） ---

    # シナリオ1: 物理商品（在庫あり）
    tokyo_store.process_order("p-001", 3)

    # シナリオ2: 物理商品（在庫なし）
    tokyo_store.process_order("p-002", 10)

    # シナリオ3: 【新】ダウンロード商品の注文
    # Storeクラスは相手がダウンロード商品だと知らなくても、正しく処理できる
    tokyo_store.process_order("d-001", 50) 

    # シナリオ4: 【新】大阪店のダウンロード商品を注文
    osaka_store.process_order("d-002", 100)

    # --- 最終在庫の表示（変更なし） ---
    print("\n--- 最終在庫 ---")
    print(f"{tokyo_store.name}:", tokyo_store.get_current_stock_info())
    print(f"{osaka_store.name}:", osaka_store.get_current_stock_info())
```

-----

## ✨ OOPによる真の改善点： ポリモーフィズム

このリファクタリングで、オブジェクト指向の真の力である **ポリモーフィズム（Polymorphism / 多態性）** が明確に示されました。

`Store` クラスの `process_order` メソッドは、`product` という変数が `PhysicalProduct` なのか `DigitalProduct` なのかを全く気にする必要がありません。

`Store` はただ「抽象的な契約（`Product`）」に従い `product.check_stock()` と命令するだけです。すると、`product` 変数に入っている「実体（インスタンス）」が、自分自身の種類に応じて、適切な振る舞い（在庫を本当に確認する or 常にOKと返す）を自動的に実行してくれます。

これにより、**オープン・クローズドの原則（OCP）が完全に満たされました**。
`Store` クラスを一切**修正することなく（＝Closed）**、新しい機能（`DigitalProduct` クラス）を安全に\*\*追加（＝Open）\*\*できました。

これこそが、手続き型プログラミング（`PP-03`）では `if` 文だらけになって破綻してしまった処理を、OOPがエレガントに解決する核心的なメカニズムです。