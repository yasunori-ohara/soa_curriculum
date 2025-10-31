# OOP-04 : オープン・クローズドの原則(OCP)の実践 - 機能拡張

`OOP-02`の評価で、私たちのコードは「新しい商品の種類（例：ダウンロード商品）」の追加（＝拡張）に対して、`if`文による既存コードの**修正**が必要になるため、「修正に対して閉じられていない」（＝OCP違反）と指摘しました。

この章では、`OOP-03`で適用したDIP（依存性逆転の原則）が、このOCP違反をいかにして解決するのかを学びます。在庫の概念がない「ダウンロード商品」を**既存のコードを一切修正せずに追加**できることを実証します。

## 🎯 この章のゴール

  * DIP（抽象への依存）が、OCP（拡張に開く）を可能にする理由を理解する。
  * オープン・クローズドの原則（OCP）をコードで実践する。
  * \*\*ポリモーフィズム（多態性）\*\*の絶大な効果を理解する。
  * `Store` クラスを一切修正せずに新機能を追加できる、というOOPの真の価値を体感する。

-----

## 🧐 なぜDIPがOCPの「準備」になるのか？

`OOP-03`で「DIPを適用した今、準備が整いました」と述べましたが、これはどういう意味でしょうか？

  * **DIP適用前 (OOP-02)**

      * `Store`（上位）は `Product`（**具象**）に直接依存していました。
      * この状態で`DigitalProduct`（別の**具象**）を追加しようとすると、`Store`は`PhysicalProduct`と`DigitalProduct`の**両方**を知る必要があります。
      * `Store`の`process_order`メソッド内には、`if type(product) is PhysicalProduct:` のような分岐（＝**修正**）が必須となり、OCP違反が確定します。
      * 
  * **DIP適用後 (OOP-03)**

      * `Store`（上位）は `Product`（**抽象インターフェース**）にのみ依存するようになりました。
      * `Store`はもはや「物理商品」という具象を知りません。「`check_stock`できる」「`reduce_stock`できる」という\*\*「契約」\*\*だけを知っている状態です。
      * 
`Store`が「具体的な相手」ではなく「**契約**」に依存するようになったため、`Store`は「**その契約さえ守ってくれるなら、相手が誰であろうと構わない**」という非常に柔軟な立場になりました。

これが「準備が整った」状態です。
`DigitalProduct`が`Product`（抽象）の契約さえ守れば、`Store`は相手が`PhysicalProduct`なのか`DigitalProduct`なのかを区別する必要がなくなり（＝ポリモーフィズムが機能し）、`if`文による**修正**が不要になります。

-----

## 🛠️ 設計の事前見直し (OCPのための契約修正)

ただし、`OOP-03`の設計には一つ不備がありました。

`OOP-03`の`Product`（抽象）は、`get_stock() -> int`（在庫**数**を返す）という契約にしてしまいました。これは、`DigitalProduct`（在庫が数値でない）の登場を予期しておらず、無意識に「物理商品」の仕様に引きずられた設計です。

このままでは、`DigitalProduct`が契約（`int`を返す）を守れず、OCP達成のデモンストレーションが失敗します。

そこで、OCPを真に達成するため、`OOP-03`の時点の「契約書」を以下のように見直します。これは`DigitalProduct`追加のための修正ではなく、「**より柔軟な拡張に耐えうるための、既存の契約不備の修正**」です。

`get_stock() -> int`（在庫**数**） ではなく、 `get_stock_info() -> str`（在庫**情報**）を返す契約に変更します。

  * `Product`（抽象）: `get_stock_info() -> str` を契約する。
  * `PhysicalProduct`（具象）: `return str(self._stock)`（数値を文字列化して）を実装する。
  * `Store`（具象）: `get_stock_info()` を呼び出すように変更する。

（※この修正は`OOP-03`のファイルに遡って適用されたものとして、以下のコードに進んでください）

-----

## 💻 `logic.py` : `DigitalProduct` クラスの「追加」

上記の「契約見直し」が完了した`logic.py`に対して、`DigitalProduct`クラスを**追加**します。

`Product`（抽象）、`PhysicalProduct`、`Store`のコードには、**ここから一切触りません**。

```python
import datetime
from abc import ABC, abstractmethod

class Product(ABC):
    """
    商品の「抽象的な概念」を定義するインターフェース（抽象基底クラス）。
    この「契約書」は【見直し済み】であり、これ以上変更しない。
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
    def get_stock_info(self) -> str: # 【見直し済み】契約を「文字列」に変更
        """【契約】在庫情報を取得する"""
        pass

class PhysicalProduct(Product):
    """
    物理的な商品を表現する「具体的な実装」クラス。
    このクラスは【見直し済み】であり、これ以上変更しない。
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

    def get_stock_info(self) -> str: # 【見直し済み】契約（str）を実装
        """【実装】物理在庫の数を「文字列」で返す。"""
        return str(self._stock)

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

    def get_stock_info(self) -> str: # 【見直し済み】契約（str）を実装
        """【実装】在庫数ではなく、分かりやすい文字列を返す"""
        return "N/A (ダウンロード可)"

# --- ここまでが「追加」したコード ---


class Store:
    """
    店舗を表すクラス。
    このクラスのコードは OOP-03(見直し後) から「一切変更なし」！
    """
    def __init__(self, name: str):
        self.name = name
        # この辞書は「Product(抽象)」の契約を守るものなら何でも格納できる
        self._products: dict[str, Product] = {}
        self._orders = []

    def add_product(self, product: Product):
        """
        渡されるのが PhysicalProduct か DigitalProduct か、
        Store は気にする必要がない。（抽象に依存しているため）
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
        # Store は全く意識する必要がない。(if文が不要！)
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
        get_stock_info() を呼ぶと、相手に応じて str(在庫数) か str("N/A") が返る。
        """
        stock_info = {}
        for p_id, p in self._products.items():
            stock_info[p.name] = p.get_stock_info() # 【見直し済み】
        return stock_info
```

-----

## 🏛️ `main.py` : 新しいクラスを利用する

`main.py` も、新しい `DigitalProduct` をインポートし、インスタンスを作成して `Store` に**追加**するだけです。`Store` を操作するロジック（`process_order` の呼び出し）は一切変更ありません。

```python
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

### ポリモーフィズムとは？

ポリモーフィズムは、ギリシャ語の「ポリ（多くの）」と「モルフェー（形）」を組み合わせた言葉で、\*\*「多様な形を持つこと」\*\*を意味します。

プログラミングにおいては、「**同じ呼び出し方（インターフェース）をしているのに、呼び出されたモノ（インスタンス）の種類によって、実行される振る舞い（実装）が異なる**」という性質を指します。

### `Store`クラスで何が起きたか？

`Store` クラスの `process_order` メソッドは、このポリモーフィズムの恩恵を最大限に受けています。

```python
# product は「Product(抽象)」型として扱われる
product = self._find_product(product_id) 
# ...
# 呼び出し方は「product.check_stock()」のただ一つ
if not product.check_stock(quantity): 
    # ...
```

`Store`は、`product`変数が`PhysicalProduct`なのか`DigitalProduct`なのかを**全く気にする必要がありません**。`Store`側は、`product`変数を「`Product`（抽象）という**一つの形**」として**同一視**して扱っています。

`Store`はただ「抽象的な契約（`Product`）」に従い `product.check_stock()` と命令するだけです。

すると、`product` 変数に入っている「**実体（インスタンス）の形**」に応じて、実行される具体的な振る舞いが**自動的に変わります**。

  * `product`の中身が`PhysicalProduct`なら: 在庫数を比較する処理（`PhysicalProduct.check_stock`）が実行される。
  * `product`の中身が`DigitalProduct`なら: 常にOKと返す処理（`DigitalProduct.check_stock`）が実行される。

これが\*\*多態性（ポリモーフィズム）\*\*です。
もしポリモーフィズムがなければ、`Store`は`if type(product) is ...`のように、自分で相手の「形」を調べて処理を分岐させる（＝修正が必要になる）必要がありました。

### OCPの達成

ポリモーフィズムのおかげで、**オープン・クローズドの原則（OCP）が完全に満たされました**。
`Store` クラス（`logic.py`の既存コード）を一切**修正することなく（＝Closed）**、新しい機能（`DigitalProduct` クラス）を安全に\*\*追加（＝Open）\*\*できました。

これこそが、手続き型プログラミング（`PP-03`）では `if` 文だらけになって破綻してしまった処理を、OOPがエレガントに解決する核心的なメカニズムです。