# OOP-01 : オブジェクト指向による再設計

手続き型プログラミング（PP章）では、「データ」と「ロジック（手続き）」が分離していることにより、複数店舗への拡張（`PP-03`）でコードが複雑化し、破綻していく様子を見ました。

この章では、その問題を根本から解決するために、**オブジェクト指向プログラミング (OOP)** を用いてシステムを全面的に再設計します。「店舗」や「商品」といったビジネス上の\*\*「概念」**を、データとロジックが一体化した**「クラス」\*\*として定義します。

## 🎯 この章のゴール

  * 手続き型の限界（データとロジックの分離、複数インスタンスの困難さ）をOOPで解決する。
  * 「クラス（設計図）」と「インスタンス（実体）」の概念をコードで理解する。
  * **カプセル化**がデータの安全性と保守性をいかに高めるかを学ぶ。
  * `tokyo_store.process_order()` のような直感的なコードの利点を体感する。

-----

## 💻 `logic.py` : 「概念」をクラスとして定義する

`logic.py` の役割が変わります。手続き型では「関数（ロジック）置き場」でしたが、OOPでは「**ビジネスの概念（クラス）**」を定義する場所となります。

```python:logic.py
import datetime

class Product:
    """
    一つの「商品」という概念を表現するクラス（設計図）。
    商品に関するデータ（属性）とロジック（メソッド）を一つにまとめる。
    """
    def __init__(self, product_id: str, name: str, price: int, stock: int):
        # --- データ（属性） ---
        self.product_id = product_id # 商品ID
        self.name = name           # 商品名
        self.price = price         # 価格
        
        # 在庫データは外部から直接変更されたくない「重要なデータ」。
        # アンダースコア(_)を付け、内部でのみ触るべきことを示す（カプセル化）。
        self._stock = stock

    # --- ロジック（メソッド） ---
    
    def check_stock(self, quantity: int) -> bool:
        """
        在庫が十分かを確認するロジック。
        このメソッドは自分自身（self）の _stock を参照する。
        """
        return self._stock >= quantity

    def reduce_stock(self, quantity: int):
        """
        在庫を減らすロジック。
        必ず check_stock を通してから、自分自身（self）の _stock を変更する。
        """
        if self.check_stock(quantity):
            self._stock -= quantity
            print(f"在庫更新: {self.name}の在庫が{self._stock}になりました。")
            return True
        return False

    def get_stock(self) -> int:
        """外部が安全に在庫数を「参照」するためだけの公開メソッド。"""
        return self._stock

class Store:
    """
    一つの「店舗」という概念を表現するクラス（設計図）。
    店舗が持つべきデータ（商品リスト、注文履歴）と
    店舗が行うべき振る舞い（注文処理）を一つにまとめる。
    """
    def __init__(self, name: str):
        # --- データ（属性） ---
        self.name = name # 店舗名
        
        # この店舗が扱う「商品インスタンス」を格納する辞書（カプセル化）
        self._products = {} 
        # この店舗の「注文記録」を格納するリスト（カプセル化）
        self._orders = []   

    # --- ロジック（メソッド） ---

    def add_product(self, product: Product):
        """この店舗に商品（Productインスタンス）を追加する。"""
        self._products[product.product_id] = product

    def _find_product(self, product_id: str) -> Product | None:
        """（内部用）この店舗から特定の商品インスタンスを探す。"""
        return self._products.get(product_id)

    def process_order(self, product_id: str, quantity: int):
        """
        注文処理のメインフロー。
        手続き型と違い、引数に store_id は不要。
        このメソッドは、必ず「自分自身（self）の店舗」のデータを操作する。
        """
        print(f"\n--- [{self.name}] 注文処理開始: 商品ID={product_id}, 数量={quantity} ---")

        # 1. 自分（self）の店舗から商品を探す
        product = self._find_product(product_id)
        if not product:
            print(f"エラー: [{self.name}] 指定された商品が見つかりません。")
            return

        # 2. Productオブジェクトに在庫の確認を「依頼」する（カプセル化）
        #    Storeクラスは、Productが「どうやって」在庫を確認するか知る必要はない。
        if not product.check_stock(quantity):
            print(f"エラー: [{self.name}] {product.name}の在庫が不足しています。")
            return

        # 3. Productオブジェクトに在庫の削減を「依頼」する（カプセル化）
        product.reduce_stock(quantity)

        # 4. 自分（self）の店舗の注文記録を作成し、追加する
        order_record = {
            "product_name": product.name,
            "quantity": quantity,
            "total_price": product.price * quantity,
            "order_date": datetime.datetime.now().isoformat()
        }
        self._orders.append(order_record)

        print(f"注文成功: [{self.name}] {product.name}を{quantity}個受け付けました。")

    def get_current_stock_info(self) -> dict:
        """この店舗の現在の在庫情報を取得する（公開メソッド）。"""
        
        # 各Productインスタンスに `get_stock()` で在庫数を問い合わせる
        return {
            p.name: p.get_stock()
            for p_id, p in self._products.items()
        }
```

-----

## 🏛️ `main.py` : 「実体（インスタンス）」の生成と実行

`main.py` は、`logic.py` で定義された「設計図（クラス）」を使って、具体的な「実体（インスタンス）」を作成し、それらを操作します。

```python:main.py
# logic.py から「設計図」である Product クラスと Store クラスをインポート
from logic import Product, Store

if __name__ == "__main__":
    # --- 1. 「設計図」であるクラスから、「実体」であるインスタンスを生成 ---
    
    # Storeクラスから「東京店」インスタンスを作成
    tokyo_store = Store("東京店")
    
    # Productクラスから「商品」インスタンスを作成し、東京店インスタンスに追加
    tokyo_store.add_product(Product("p-001", "高機能マウス", 4000, 10))
    tokyo_store.add_product(Product("p-002", "静音キーボード", 6000, 5))
    tokyo_store.add_product(Product("p-003", "24インチモニター", 25000, 3))

    # Storeクラスから「大阪店」インスタンスを作成
    # tokyo_store と osaka_store は、データが完全に独立した「別の実体」
    osaka_store = Store("大阪店")
    
    # 大阪店インスタンスに、別の商品インスタンスを追加
    osaka_store.add_product(Product("p-001", "高機能マウス", 4100, 8))
    osaka_store.add_product(Product("p-002", "静音キーボード", 6000, 12))
    osaka_store.add_product(Product("p-003", "24インチモニター", 25500, 5))

    # --- 2. 各店舗のインスタンスに対して、独立して処理を実行 ---
    print("--- 初期在庫 ---")
    # 東京店インスタンスに「在庫情報を教えて」と依頼
    print(f"{tokyo_store.name}:", tokyo_store.get_current_stock_info())
    # 大阪店インスタンスに「在庫情報を教えて」と依頼
    print(f"{osaka_store.name}:", osaka_store.get_current_stock_info())

    # シナリオ1: 東京店インスタンスに「注文を処理して」と依頼
    tokyo_store.process_order("p-001", 3)

    # シナリオ2: 大阪店インスタンスに「注文を処理して」と依頼
    osaka_store.process_order("p-002", 5)

    # シナリオ3: 東京店インスタンスで在庫不足
    tokyo_store.process_order("p-002", 10)

    # シナリオ4: 大阪店インスタンスでは同じ注文が成功
    # 大阪店のインスタンスは、東京店の処理による影響を全く受けない
    osaka_store.process_order("p-002", 10)

    print("\n--- 最終在庫 ---")
    print(f"{tokyo_store.name}:", tokyo_store.get_current_stock_info())
    print(f"{osaka_store.name}:", osaka_store.get_current_stock_info())
```

-----

## ✨ OOPによる改善点

手続き型（PP章）が抱えていた根本的な問題は、OOPの基本原則によって見事に解決されました。

### 1\. 🛡️ カプセル化 (安全性向上)

`Product` クラスは在庫（`_stock`）を、`Store` クラスは商品リスト（`_products`）を内部に保持し、外部（`main.py`）からの不正なアクセスを防ぎます。
`main.py` は `tokyo_store._stock = 9999` のような不正操作（`PP-02` の紳士協定破り）を試みる必要がなく、`process_order` という安全な窓口に「依頼」するだけです。**データとそれを操作するロジックが一体化**し、安全性が格段に向上しました。

### 2\. 👥 複数インスタンスの生成 (概念の実現)

`Store` クラスという「店舗の設計図」から、`tokyo_store` と `osaka_store` という、それぞれが独立したデータ（在庫や注文履歴）を持つ「実体（インスタンス）」を簡単に生成できました。これは、`PP-03` で巨大な辞書（`_STORES_DATA`）を使って擬似的に実現しようとして破綻したことと対照的です。

### 3\. 🧠 コードの直感性 (現実との一致)

`logic.process_order("tokyo", ...)` という不自然な関数呼び出しは、`tokyo_store.process_order(...)` という、\*\*「東京店（というモノ）に、注文処理を（依頼）する」\*\*という現実世界のメンタルモデルに近い、非常に直感的で分かりやすいコードに変わりました。

-----

## 🚧 次の課題: 「良い設計」への探求

このOOPによる設計は、手続き型プログラミングが抱えていた問題を見事に解決しました。

しかし、このコードは本当に「**良い設計**」と言えるでしょうか？

例えば、`Store` クラスの `process_order` メソッドは、`Product` クラスの具体的な処理（`check_stock`, `reduce_stock`）を「知っている」状態、すなわち**強く依存**しています。また、注文記録（`order_record`）を作成するという別の責任も持っています。

このコードは「動く」ものの、将来の変更（例えば「ダウンロード商品」への対応や、「注文記録」の仕様変更）に対して、まだ脆（もろ）さを抱えています。

次の章では、この設計が「**変更に強い、良い設計**」かどうかを判断するための有名な指針である **SOLID原則** に照らし合わせて評価してみましょう。