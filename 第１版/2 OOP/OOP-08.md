# OOP-08 : デザインパターンとは？ (Factoryパターンの適用)

`OOP-07` までで、私たちのコードはSOLID原則を高いレベルで満たしました。
この章では、OOPによる設計をさらに洗練させるための強力な道具、「デザインパターン」について学びます。

## 🎯 この章のゴール

  * GoFデザインパターンの基本的な概念（＝設計の定石集）を理解する。
  * 「オブジェクトの生成」という責任を分離する **Factory パターン**を学ぶ。
  * `main.py`（クライアント）が、具象クラス（`PhysicalProduct` など）に依存しないようにリファクタリングする。

-----

## 🤔 GoFデザインパターンとは？

GoF (Gang of Four: 4人組) デザインパターンとは、オブジェクト指向設計で頻繁に現れる特定の問題に対して、先人たちが見つけ出した\*\*「成功事例のカタログ」あるいは「設計の定石集」\*\*です。

SOLID原則が、個々のクラスをどう作るかという「**ミクロな原則**（例：クラスの責任は一つに）」だとすれば、デザインパターンは、それらのクラスをどう組み合わせて、どう連携させるかという「**メソな（中規模の）戦術**（例：オブジェクトの作り方を専門家に任せる）」と言えます。

-----

## 🚧 現状の課題： `main.py` による具象クラスの直接生成

現在の `main.py` は、SOLID原則を満たした `Store` や `Product` を利用していますが、`main.py` 自身にはまだ問題が残っています。

```python:main.py (OOP-06の抜粋)
from logic import (
    PhysicalProduct, DigitalProduct, Store, # ← 具象クラスを直接知っている
    IOrderRepository, InMemoryOrderRepository, Order
)

if __name__ == "__main__":
    # ...
    # ↓ 具象クラスを直接インスタンス化している
    tokyo_store.add_product(PhysicalProduct("p-001", ...))
    tokyo_store.add_product(DigitalProduct("d-001", ...))
```

`main.py`（アプリケーションの起動・実行ファイル）が、`PhysicalProduct` や `DigitalProduct` といった**具体的なクラス名を知ってしまっています**。

もし将来「サブスクリプション商品（`SubscriptionProduct`）」クラスが追加されたら、`logic.py` にクラスを追加するだけでなく、`main.py` にも `import SubscriptionProduct` を追加し、`SubscriptionProduct(...)` というコードを追記する\*\*「修正」\*\*が必要になってしまいます。

-----

## 💡 Factoryパターンによる解決

この「オブジェクトを使う側 (`main.py`)」と「オブジェクトを作る側」の密接な関係を断ち切るために、**Factory（工場）パターン**を導入します。

「オブジェクトの生成」という責任を、専門の `ProductFactory` クラスに分離します。

  * `main.py` は `ProductFactory` に「"physical"（物理）の商品をください」と**依頼**するだけ。
  * `ProductFactory` が、`if` 文を使って `PhysicalProduct` を `new` する責任を**一手に引き受けます**。

-----

## 💻 `logic.py` : `ProductFactory` の追加

`OOP-06` の `logic.py` に、`ProductFactory` クラスを**追加**します。`Product`, `Store`, `Order` などの他のクラスは一切変更しません。

```python:logic.py
import datetime
from abc import ABC, abstractmethod

# --- OOP-06までのクラス (一切変更なし) ---
class Product(ABC):
    # (OOP-06 と同じコードのため省略)
    def __init__(self, product_id: str, name: str, price: int): ...
    @abstractmethod
    def check_stock(self, quantity: int) -> bool: ...
    @abstractmethod
    def reduce_stock(self, quantity: int): ...
    @abstractmethod
    def get_stock(self) -> int: ...
    @abstractmethod
    def is_stock_managed(self) -> bool: ...

class PhysicalProduct(Product):
    # (OOP-06 と同じコードのため省略)
    def __init__(self, product_id: str, name: str, price: int, stock: int): ...
    # (メソッド実装も省略) ...

class DigitalProduct(Product):
    # (OOP-06 と同じコードのため省略)
    def __init__(self, product_id: str, name: str, price: int): ...
    # (メソッド実装も省略) ...

class Order: ... # (OOP-05 と同じコードのため省略)
class IOrderRepository(ABC): ... # (OOP-05 と同じコードのため省略)
class InMemoryOrderRepository(IOrderRepository): ... # (OOP-05 と同じコードのため省略)

class Store: # (OOP-06 と同じコードのため省略)
    def __init__(self, name: str, order_repository: IOrderRepository): ...
    # (メソッド実装も省略) ...

# --- ここからが「追加」したコード ---

class ProductFactory:
    """
    【新設】商品を生成する専門の「工場」クラス。
    
    「どの種類(type)の商品を作るか」という知識（if文）を、
    main.py から引き離し、このクラスに集約する。
    """
    @staticmethod
    def create_product(product_type: str, **kwargs) -> Product:
        """
        指定された種類の商品インスタンスを生成して返す。
        
        `**kwargs` (キーワード引数) を使うことで、
        "physical" には stock が渡され、
        "digital" には stock が渡されない、といった違いを
        柔軟に吸収できる。
        """
        if product_type == "physical":
            # physical なら PhysicalProduct クラスをインスタンス化
            return PhysicalProduct(
                product_id=kwargs["product_id"],
                name=kwargs["name"],
                price=kwargs["price"],
                stock=kwargs["stock"]
            )
        elif product_type == "digital":
            # digital なら DigitalProduct クラスをインスタンス化
            return DigitalProduct(
                product_id=kwargs["product_id"],
                name=kwargs["name"],
                price=kwargs["price"]
            )
        
        # 将来「サブスクリプション商品」が追加されたら、
        # ここに elif product_type == "subscription": を追加するだけ。
        # main.py の修正は不要。
        
        else:
            raise ValueError(f"未知の商品種別です: {product_type}")

# --- ここまでが「追加」したコード ---
```

-----

## 🏛️ `main.py` : Factory を使うように変更

`main.py` は、`PhysicalProduct` や `DigitalProduct` を直接インポートするのをやめ、代わりに `ProductFactory` をインポートして利用します。

```python:main.py
# 具象クラス (PhysicalProduct, DigitalProduct) の代わりに、
# 工場 (ProductFactory) をインポートする
from logic import (
    ProductFactory, Store, 
    IOrderRepository, InMemoryOrderRepository, Order
)

if __name__ == "__main__":
    
    # --- 1. 依存関係の構築 (DI) (変更なし) ---
    tokyo_repo: IOrderRepository = InMemoryOrderRepository()
    tokyo_store = Store("東京店", order_repository=tokyo_repo)
    
    osaka_repo: IOrderRepository = InMemoryOrderRepository()
    osaka_store = Store("大阪店", order_repository=osaka_repo)

    # --- 2. 商品のセットアップ (【変更】) ---
    
    # ProductFactory を使って商品を生成する
    # main.py は、PhysicalProduct という具体的なクラス名を知らなくて済む
    mouse = ProductFactory.create_product(
        "physical", 
        product_id="p-001", name="高機能マウス", price=4000, stock=10
    )
    keyboard = ProductFactory.create_product(
        "physical", 
        product_id="p-002", name="静音キーボード", price=6000, stock=5
    )
    ebook = ProductFactory.create_product(
        "digital", 
        product_id="d-001", name="デザインソフト eBook", price=8000
    )

    tokyo_store.add_product(mouse)
    tokyo_store.add_product(keyboard)
    tokyo_store.add_product(ebook)

    # 大阪店も同様に Factory 経由で生成
    osaka_store.add_product(ProductFactory.create_product(
        "physical", 
        product_id="p-001", name="高機能マウス", price=4100, stock=8
    ))
    osaka_store.add_product(ProductFactory.create_product(
        "digital", 
        product_id="d-002", name="プログラミング講座 動画", price=12000
    ))

    # --- 3. 在庫表示と注文処理 (変更なし) ---
    # Store を「使う側」のコードは一切変更する必要がない
    
    print("--- 初期在庫 ---")
    print(f"{tokyo_store.name}:", tokyo_store.get_current_stock_info())
    print(f"{osaka_store.name}:", osaka_store.get_current_stock_info())

    tokyo_store.process_order("p-001", 3)
    tokyo_store.process_order("p-002", 10) # 在庫不足
    tokyo_store.process_order("d-001", 50) # ダウンロード

    print("\n--- 最終在庫 ---")
    print(f"{tokyo_store.name}:", tokyo_store.get_current_stock_info())
    print(f"{osaka_store.name}:", osaka_store.get_current_stock_info())

    # --- 4. 注文履歴の表示 (変更なし) ---
    print(f"\n--- {tokyo_store.name} 注文履歴 ---")
    for order in tokyo_store.get_order_history():
        print(order)
```

-----

## ✨ この変更の効果

1.  **関心事の分離 (SRP):**
    「商品を使う側 (`main.py`)」と「商品を作る側 (`ProductFactory`)」の責任を明確に分離できました。`main.py` はアプリケーションの起動と実行フローに集中できます。
2.  **保守性の向上 (OCP):**
    新しい商品クラス（例：`SubscriptionProduct`）が追加された場合、修正が必要なのは `ProductFactory` クラスの `if` 文だけです。`main.py` のような利用側のコードは一切変更する必要がなくなり、OCPが `main.py` のレベルでも達成されました。
3.  **柔軟性の向上:**
    `ProductFactory` は今 `if` 文で分岐していますが、これを「設定ファイル（JSONなど）を読み込んで、動的にインスタンスを生成する」といった高度な機能に拡張することも容易になりました。

これでOOPの基礎からSOLID原則、そして代表的なデザインパターン（リポジトリ、ファクトリ）までを網羅しました。次のステップである「クリーンアーキテクチャ」に進む準備が整いました。