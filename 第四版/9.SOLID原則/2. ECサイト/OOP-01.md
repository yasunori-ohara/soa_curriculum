# OOP-01 : リファクタリングの題材「ECサイト注文処理」

この章では、「オブジェクト指向で書かれたプログラムを、クリーンアーキテクチャの型にはめてリファクタリングしたら、結果、SOLID原則も適用できていた。」という例を、「ECサイトの注文処理」プログラムを例題として実践していきます。

まず、リファクタリングの対象となる初期コードを示します。
このコードは、異なる種類の商品（物理商品、ダウンロード商品）に対応するため、オブジェクト指向の\*\*ポリモーフィズム（多態性）\*\*を活用しています。しかし、アーキテクチャ全体としては多くの課題を抱えています。

プログラムは、「商品」のクラス（「設計図」）を定義する`logic.py`と、システムを起動する`main.py`の2つのファイルで構成されています。

## 🎯 この章のゴール

  * リファクタリング対象のコード（ECサイト注文処理）の仕様と構成を理解する。
  * クラスの「**継承**」と「**ポリモーフィズム（多態性）**」が、異なる種類の商品（物理、デジタル）に対応するためにどのように使われているかを確認する。
  * `Store`クラスに多くの機能が集中していることを把握する。

-----

## 🛠️ 1\. `logic.py` : 商品クラスとシステムロジック

このファイルは、「商品」という概念をクラスとして定義し、さらに注文処理システムの中核ロジックも実装しています。

### 商品クラス定義（ポリモーフィズム適用済み）

まず、`IProduct`という**インターフェース（抽象クラス）を定義し、「商品であれば必ず満たすべき契約（属性や`check_stock`メソッドなど）」を定めています。
次に、具体的な商品である`PhysicalProduct`（物理商品）と`DigitalProduct`（ダウンロード商品）が、その`IProduct`インターフェースを継承**し、契約されたメソッドをそれぞれ独自の方法で**実装**しています。

### Storeクラス（システムロジック）

`Store`クラスは、注文処理のメインフロー（`process_order`）や、商品データの管理（`_products`）、注文記録の保存（`_orders`）といった機能を提供します。`process_order`メソッド内では、ポリモーフィズムを活用し、商品の種類（物理かデジタルか）を意識せずに処理を行っています。

#### logic.py
```python
import datetime
from abc import ABC, abstractmethod

# --- 商品インターフェースと具象クラス ---

class IProduct(ABC):
    """
    【抽象】商品のインターフェース（契約）。
    Storeクラスは、このインターフェースに依存する。
    """
    def __init__(self, product_id: str, name: str, price: int):
        self._product_id = product_id
        self._name = name
        self._price = price
        
    @property
    def product_id(self) -> str: return self._product_id
    @property
    def name(self) -> str: return self._name
    @property
    def price(self) -> int: return self._price

    @abstractmethod
    def check_stock(self, quantity: int) -> bool:
        """【契約】指定された数量の在庫があるか"""
        pass

    @abstractmethod
    def reduce_stock(self, quantity: int):
        """【契約】指定された数量の在庫を減らす"""
        pass

class PhysicalProduct(IProduct):
    """【具象】物理商品。在庫を持つ。"""
    def __init__(self, product_id: str, name: str, price: int, stock: int):
        super().__init__(product_id, name, price)
        self._stock = stock # 物理在庫

    @property
    def stock(self) -> int: return self._stock

    def check_stock(self, quantity: int) -> bool:
        """【実装】物理在庫が十分か確認する"""
        return self._stock >= quantity

    def reduce_stock(self, quantity: int):
        """【実装】物理在庫を減らす"""
        if not self.check_stock(quantity):
            # (簡易化のため例外ではなくFalseを返す or 何もしない)
            print(f"エラー(内部): {self.name}の在庫({self._stock})が不足しています({quantity}要求)。")
            return False # 本来は例外を投げるべき
        
        self._stock -= quantity
        print(f"在庫更新: {self.name}の在庫が{self._stock}になりました。")
        return True

class DigitalProduct(IProduct):
    """【具象】ダウンロード商品。在庫の概念がない。"""
    def __init__(self, product_id: str, name: str, price: int):
        super().__init__(product_id, name, price)
        # 在庫(stock)属性は持たない

    def check_stock(self, quantity: int) -> bool:
        """【実装】常に在庫あり(True)"""
        return True

    def reduce_stock(self, quantity: int):
        """【実装】在庫は減らない（何もしない）"""
        print(f"在庫更新: {self.name} はダウンロード商品のため在庫変動なし。")
        pass # Trueを返す (PhysicalProductと合わせる)

# --- システムロジッククラス ---

class Store:
    """
    店舗での注文処理に関するすべての機能を提供するクラス。
    """
    def __init__(self, name: str):
        self.name = name
        # 責任A: 商品データの「保存場所」（メモリ上の辞書）
        self._products: dict[str, IProduct] = {} 
        # 責任B: 注文記録の「保存場所」（メモリ上のリスト）
        self._orders: list[dict] = [] 

    def add_product(self, product: IProduct):
        """責任A: 商品をインメモリDBに追加する"""
        print(f"[{self.name}] 商品追加: {product.name}")
        self._products[product.product_id] = product

    def _find_product(self, product_id: str) -> IProduct | None:
        """責任A: インメモリDBから商品を探す"""
        return self._products.get(product_id)

    def process_order(self, product_id: str, quantity: int):
        """責任C: 注文処理のメインフローを実行する"""
        print(f"\n--- [{self.name}] 注文処理開始: 商品ID={product_id}, 数量={quantity} ---")

        # 1. 商品を探す (責任Aの一部)
        product = self._find_product(product_id)
        if not product:
            print(f"エラー: [{self.name}] 指定された商品が見つかりません。")
            return

        # ▼▼▼ ポリモーフィズム活用 ▼▼▼
        # productがPhysicalかDigitalかをStoreは知らない。
        # IProductの契約通り check_stock を呼ぶだけ。
        if not product.check_stock(quantity):
            print(f"エラー: [{self.name}] {product.name}の在庫が不足しています。")
            return

        # ▼▼▼ ポリモーフィズム活用 ▼▼▼
        # reduce_stockも同様。実装は相手に任せる。
        product.reduce_stock(quantity)
        # ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲

        # 4. 注文記録を作成し、追加する (責任Bの一部)
        order_record = {
            "product_name": product.name,
            "quantity": quantity,
            "total_price": product.price * quantity,
            "order_date": datetime.datetime.now().isoformat()
        }
        self._orders.append(order_record)
        print(f"注文記録保存: {product.name} をリストに保存しました。")

        print(f"注文成功: [{self.name}] {product.name}を{quantity}個受け付けました。")

    def get_order_history(self) -> list[dict]:
        """責任B: 保存されている注文記録を取得する"""
        return self._orders

    # (在庫表示メソッドは省略)
```

-----

## 🛠️ 2\. `main.py` : システムの実行

このファイルは、`Store`クラスをインスタンス化（実体化）し、実際にシステムを動かす役割を持ちます。`PhysicalProduct`と`DigitalProduct`の両方を`Store`に追加して利用します。

#### main.py
```python
# logic.py から「具体的な」商品クラスと「ロジック」クラスをインポート
from logic import PhysicalProduct, DigitalProduct, Store

if __name__ == "__main__":
    # 1. システム（ロジック）のインスタンスを作成
    tokyo_store = Store("東京店")
    
    # 2. 商品（データ）のインスタンスを作成し、システムに追加
    tokyo_store.add_product(
        PhysicalProduct("p-001", "高機能マウス", 4000, 10)
    )
    tokyo_store.add_product(
        PhysicalProduct("p-002", "静音キーボード", 6000, 5)
    )
    tokyo_store.add_product(
        DigitalProduct("d-001", "デザインソフト eBook", 8000) # デジタル商品
    )

    # (大阪店のインスタンス作成と商品追加は省略)

    # --- 3. システムのメインロジック（注文処理）を実行 ---
    
    # シナリオ1: 物理商品 (成功)
    tokyo_store.process_order("p-001", 3)

    # シナリオ2: 物理商品 (在庫不足)
    tokyo_store.process_order("p-002", 10)

    # シナリオ3: デジタル商品 (成功)
    # 在庫5のキーボードと同じ数量(10)でも、デジタル商品なら成功する
    tokyo_store.process_order("d-001", 10)

    print("\n--- 東京店 注文履歴 ---")
    print(tokyo_store.get_order_history())
```

-----

## `OOP-02`への準備

このコードは、`logic.py`でポリモーフィズムを活用することで、「新しい種類の商品（例：予約商品）」の追加には柔軟に対応できる**可能性**があります (OCP/LSP)。

しかし、`Store`クラスに、注文処理フロー、商品データ管理、注文ログ保存など、多くの機能が**集中**しているようにも見えます。

このコードは「**変更**」に対してどれだけ強いのでしょうか？ 特に、**データ保存の方法**（今はメモリ）を変えたい場合や、**注文処理のルール**を変えたい場合に、どこを修正する必要があるでしょうか？

次の章では、この`OOP-01`のコードが「**良い設計**」と呼べるかどうか、**SOLID原則**というものさしを使って詳しく評価（健康診断）してみましょう。


