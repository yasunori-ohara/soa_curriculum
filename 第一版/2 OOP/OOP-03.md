# OOP-03 : 依存性逆転の原則(DIP)によるリファクタリング

`OOP-02`の評価で、`Store`（上位モジュール）が`Product`（下位モジュール）という\*\*具体的なクラス（具象クラス）\*\*に直接依存している、というDIP違反が見つかりました。

この章では、この「具体的なモノへの依存」がなぜ問題なのかを理解し、リファクタリングを通じて依存関係を「逆転」させます。

## 🎯 この章のゴール

  * 「具象」ではなく「抽象（インターフェース）」に依存するメリットを理解する。
  * なぜDIPが「依存性**逆転**」と呼ばれるのかを体感する。
  * Pythonの `abc` (抽象基底クラス) モジュールを使い、「インターフェース（契約）」を定義する方法を学ぶ。

-----

## 🧐 なぜ「具象への依存」が問題なのか？

現在の`Store`クラスは、「商品とは`Product`クラス（という*具体的な実装*）のことである」と決め打ちしています。

```python: logic.py (問題の箇所)
class Store:
    # ...
    # 依存先が「具象クラス Product」
    def add_product(self, product: Product): ...
    def _find_product(self, product_id: str) -> Product | None: ...
```

これが引き起こす最大の問題は、`OOP-02`で指摘された**OCP（オープン・クローズドの原則）違反**と直結しています。

もし将来、「在庫の概念がない**ダウンロード商品**」を追加したくなったとします。
この商品は、`check_stock`や`reduce_stock`の振る舞いが`Product`（物理商品）と全く異なります。

`Store`が`Product`（物理商品）の実装に依存していると、`Store`側は「もし物理商品ならこう処理し、もしダウンロード商品ならこう処理する」というような `if` 文による分岐を**追加修正**せざるを得なくなります。

これでは、`Store`（上位）が`Product`（下位）の種類の増加（＝拡張）によって、修正を強いられてしまいます。これこそが「変更に弱い」設計です。

## 💡 解決策：「抽象（インターフェース）」への依存

DIPの解決策は、`Store`（上位）と`Product`（下位）の間に、\*\*「抽象的な取り決め」\*\*を挟むことです。

### 🚀 「具象」と「抽象（インターフェース）」とは？

ここで、重要な概念が登場します。

  * **具象 (Concrete) クラス**:

      * 今までの`Product`クラスのように、具体的なロジック（実装）を持ち、インスタンス化できるクラス。
      * 例：「物理的な商品」「特定のメーカーのドライヤー」

  * **抽象 (Abstract) クラス / インターフェース (Interface)**:

      * 具体的なロジック（実装）を持たず、「どのようなメソッドを持っているべきか」という\*\*「契約（ルール、振る舞い）」\*\*だけを定義したもの。
      * 例：「『商品』という概念」「壁のコンセントの規格」

`Store`を「壁のコンセント」だと考えてみてください。
コンセントは、相手が「ドライヤー（具象）」だろうが「PCの充電器（具象）」だろうが、\*\*「コンセントの規格（抽象）」\*\*さえ満たしていれば、どれでも受け入れます。コンセント側は、相手がドライヤーかどうかを気にする必要はありません。

`Store`もこうあるべきです。`Store`は「物理商品（具象）」に依存するのではなく、「商品とは`check_stock`や`reduce_stock`ができるものである」という\*\*「商品の規格（抽象インターフェース）」\*\*に依存すべきです。

### 🚀 Pythonでの実現技術: `abc`モジュール

Pythonでこの「抽象インターフェース（契約）」を定義するために、標準ライブラリの **`abc` (Abstract Base Class)** モジュールを使用します。

  * `ABC`: これを継承すると「このクラスは抽象クラスですよ」という印になります。
  * `@abstractmethod`: メソッドの上にこれを付けると、「このクラスを継承する具象クラスは、**必ずこのメソッドを実装（オーバーライド）しなさい**」という「契約」を強制できます。

-----

## 💻 リファクタリングの手順

`logic.py`をステップ・バイ・ステップで変更していきます。

### 🚀 ステップ1: 「商品の契約（インターフェース）」を定義する

まず、`Store`が`Product`に求めている最低限の「契約」とは何かを明確にし、それを`Product`抽象クラスとして定義します。

#### logic.py (変更)
```python
import datetime
# Pythonで「抽象クラス」を作るための仕組みをインポート
from abc import ABC, abstractmethod

class Product(ABC): # (1) ABCを継承し、抽象クラス化
    """
    商品の「抽象的な概念」を定義するインターフェース（抽象基底クラス）。
    Storeクラスは、今後この「抽象」に依存する。
    """
    def __init__(self, product_id: str, name: str, price: int):
        # 抽象クラスでも、共通の属性は定義できる
        self.product_id = product_id
        self.name = name
        self.price = price

    # (2) @abstractmethodで「契約」を定義
    @abstractmethod
    def check_stock(self, quantity: int) -> bool:
        """【契約】在庫を確認するという振る舞いを約束させる。"""
        # 抽象メソッドは中身（実装）を持たない (pass)
        pass

    @abstractmethod
    def reduce_stock(self, quantity: int):
        """【契約】在庫を減らすという振る舞いを約束させる。"""
        pass

    @abstractmethod
    def get_stock(self) -> int:
        """【契約】在庫数を取得するという振る舞いを約束させる。"""
        pass
```

### 🚀 ステップ2: 「具体的な商品」を実装する

次に、`OOP-01`の旧`Product`クラスを、`PhysicalProduct`（物理商品）という名前に変更し、ステップ1で定義した`Product`（抽象）の「契約」を**実装**する具象クラスとして再定義します。


```python:
class PhysicalProduct(Product): # (3) Product(抽象)を継承
    """
    物理的な商品を表現する「具体的な実装」クラス。
    
    Product(抽象)を継承し、「契約」で約束した全ての抽象メソッドを
    具体的に実装（オーバーライド）する。
    """
    def __init__(self, product_id: str, name: str, price: int, stock: int):
        # 親クラス(Product)の __init__ を呼び出す
        super().__init__(product_id, name, price)
        # PhysicalProduct 固有の属性
        self._stock = stock

    # --- (4) Product(抽象)で「契約」したメソッドの具体的な実装 ---
    
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
```

### 🚀 ステップ3: `Store`の依存先を「抽象」に変更する

最後に、`Store`クラスが依存する相手を、具象の`PhysicalProduct`から、抽象の`Product`インターフェースに変更します。

### logic.py (Storeクラスの変更)
```python
class Store:
    """
    店舗を表すクラス。
    DIP適用により、具体的な PhysicalProduct ではなく、
    抽象的な Product にのみ依存する。
    """
    def __init__(self, name: str):
        self.name = name
        # (5) この辞書が保持するのは「Product(抽象)」を満たすオブジェクト
        self._products: dict[str, Product] = {} # 型ヒントを抽象クラスに変更
        self._orders = [] 

    # (6) 引数の型ヒントが「抽象クラス」に
    def add_product(self, product: Product): 
        """
        渡されるのが PhysicalProduct でも、将来作る DigitalProduct でも、
        「Product(抽象)」の契約さえ守っていれば受け入れ可能。
        """
        self._products[product.product_id] = product

    # (7) 戻り値も「抽象クラス」に
    def _find_product(self, product_id: str) -> Product | None:
        return self._products.get(product_id)

    def process_order(self, product_id: str, quantity: int):
        """
        このメソッド内のロジックは OOP-01 から「一切変更なし」。
        
        なぜなら、Storeは最初から product.check_stock() のように、
        具象（実装）を直接触らず、「契約（メソッド呼び出し）」
        を通じて操作していたから。
        """
        print(f"\n--- [{self.name}] 注文処理開始: 商品ID={product_id}, 数量={quantity} ---")

        # (8) product は「抽象 Product」型
        product = self._find_product(product_id)
        if not product:
            print(f"エラー: [{self.name}] 指定された商品が見つかりません。")
            return

        # Storeは相手が PhysicalProduct かどうかを「知らない」。
        # ただ「Product(抽象)」の契約通り check_stock を呼び出すだけ。
        if not product.check_stock(quantity):
            print(f"エラー: [{self.name}] {product.name}の在庫が不足しています。")
            return

        # 同様に、reduce_stock を呼び出すだけ。
        product.reduce_stock(quantity)

        # ... (注文記録の作成ロジックは変更なし) ...
        # (※これはS違反の可能性ありとしてOOP-02で指摘済み)
        order_record = {
            "product_name": product.name,
            "quantity": quantity,
            "total_price": product.price * quantity,
            "order_date": datetime.datetime.now().isoformat()
        }
        self._orders.append(order_record)

        print(f"注文成功: [{self.name}] {product.name}を{quantity}個受け付けました。")

    def get_current_stock_info(self) -> dict:
        """店舗の現在の在庫情報を取得する。"""
        stock_info = {}
        for p_id, p in self._products.items():
            # p が何の具象クラスか知らなくても、
            # 「Product(抽象)」が get_stock() を契約しているので安全に呼び出せる。
            stock_info[p.name] = p.get_stock()
        return stock_info
```

-----

## 🏛️ `main.py` : 利用者側の変更

`Store`を利用する`main.py`はどう変わるでしょうか？

`main.py`の役割は、「**具体的なモノ（具象クラス）**」をインスタンス化し、それを`Store`（抽象に依存するクラス）に**注入 (Inject)** することです。

### main.py (変更)
```python
# インポートするのが Product(抽象) ではなく、
# 「具体的な実装」である PhysicalProduct に変わる
from logic import PhysicalProduct, Store 

if __name__ == "__main__":
    # Storeインスタンスの作成（変更なし）
    tokyo_store = Store("東京店")
    
    # Storeに「注入」するのは、「具体的な」PhysicalProduct のインスタンス
    tokyo_store.add_product(PhysicalProduct("p-001", "高機能マウス", 4000, 10))
    tokyo_store.add_product(PhysicalProduct("p-002", "静音キーボード", 6000, 5))
    tokyo_store.add_product(PhysicalProduct("p-003", "24インチモニター", 25000, 3))

    # 大阪店も同様
    osaka_store = Store("大阪店")
    osaka_store.add_product(PhysicalProduct("p-001", "高機能マウス", 4100, 8))
    osaka_store.add_product(PhysicalProduct("p-002", "静音キーボード", 6000, 12))
    osaka_store.add_product(PhysicalProduct("p-003", "25インチモニター", 25500, 5))

    # --- 以下の「Storeを操作するコード」は一切変更なし！ ---
    # Storeクラスは相手が PhysicalProduct であることを知らず、
    # 「Product(抽象)」として扱っているため、以前のコードがそのまま動作する。

    print("--- 初期在庫 ---")
    print(f"{tokyo_store.name}:", tokyo_store.get_current_stock_info())
    print(f"{osaka_store.name}:", osaka_store.get_current_stock_info())

    tokyo_store.process_order("p-001", 3)
    osaka_store.process_order("p-002", 5)
    # ... (以降の処理もすべて変更なし) ...
```

-----

## ✨ この変更の効果と「依存性逆転」

このリファクタリングで、依存関係は以下のように変わりました。

**変更前 (DIP違反):**
`Store` (上位) → `Product` (下位・**具象**)

**変更後 (DIP適用):**
`Store` (上位) → `Product` (**抽象**) ← `PhysicalProduct` (下位・**具象**)

注目すべきは、`PhysicalProduct`（下位・具象）が `Product`（抽象）を**継承**している点です。矢印の向きが、具象から抽象に向かっています。

  * `Store`（上位）は「抽象」に依存する。
  * `PhysicalProduct`（下位）も「抽象」に依存する（＝契約を実装する）。

`Store` → `PhysicalProduct` だった依存関係が、`Store` → `Product` ← `PhysicalProduct` となり、**具象クラス（`PhysicalProduct`）への依存の矢印が「逆転」しました。これが「依存性逆転**の原則」と呼ばれる理由です。

`Store`は、相手が「物理商品」であることを知らなくても動作するようになりました。

これで、`OOP-02`で指摘されたもう一つの課題、\*\*O（オープン・クローズドの原則）\*\*違反を解決する準備が完璧に整いました。

次の章では、新しい「ダウンロード商品（`DigitalProduct`）」クラスを作ります。そのクラスが`Product`（抽象）の契約さえ守れば、**`Store`クラスには一切手を加えることなく（＝修正に対して閉じたまま）**、新機能を追加（＝拡張に対して開く）できることを確認します。