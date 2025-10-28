# OOP-07 : リスコフの置換原則(LSP)の違反と修正

`OOP-04`で「ダウンロード商品」を追加した際、`Store`クラスを修正せずに機能追加（OCP）ができました。一見、成功したように見えましたが、この実装には`OOP-02`で保留にしていた\*\*L（リスコフの置換原則）\*\*の違反が潜んでいます。

この章では、このLSP違反がなぜ問題なのか、そしてOCP違反の根本的な再発防止にどう繋がるのかを学び、SOLID原則のすべてを満たす、より堅牢な設計を目指します。

## 🎯 この章のゴール

  * リスコフの置換原則 (LSP) 違反がコードに与える危険性を理解する。
  * `OOP-04`のコードがLSPに違反している箇所（`get_stock`）を特定する。
  * 「振る舞い（戻り値の型など）」の契約を守ることの重要性を学ぶ。
  * LSP違反を解消するようにリファクタリングする。

-----

## 🚧 LSP違反の特定

LSPとは、「**親クラス（抽象）のオブジェクトを、その子クラス（具象）のオブジェクトで置き換えても、プログラムの動作（振る舞い）が変わらないべき**」という原則です。

`OOP-04`の`Product`（抽象）の契約は、OCPを達成するために「`int`または`str`を返す」という**曖昧な契約**になっていました。

#### logic.py (OOP-04の契約)
```python
class Product(ABC):
    # ...
    @abstractmethod
    def get_stock(self) -> int | str: # 【契約】int または str を返す
        pass
```

そして、`Store`（利用者側）は、この曖昧な契約に従って`get_stock()`を呼び出します。

#### logic.py (OOP-04のStore)
```python
    def get_current_stock_info(self) -> dict:
        stock_info = {}
        for p_id, p in self._products.items():
            # p.get_stock() の戻り値が int か str かで、
            # 将来的にStore側の処理が変わる危険性を産んでいる
            stock_info[p.name] = p.get_stock()
        return stock_info
```

### 🚧 なぜこれが問題なのか？（LSP違反の危険性）

`Store`（利用者）は、「`get_stock`が`int`（在庫数）を返す」ことを期待して、将来「全店舗の合計在庫数を計算する」メソッドを追加したくなるかもしれません。

#### storeクラスの将来の拡張イメージ (悪い例)

```python
    def get_total_stock_count(self) -> int:
        total = 0
        for p in self._products.values():
            stock = p.get_stock()
            
            # ▼▼▼ LSP違反が引き起こす問題 ▼▼▼
            # ここで型チェックが必要になる！
            if isinstance(stock, int):
                total += stock
            # ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
            
        return total
```

`get_stock()`が`str` ("N/A...") を返す`DigitalProduct`（子クラス）が存在するため、`Store`（利用者側）は`isinstance`で型チェックをせざるを得なくなります。

これは、子クラス (`DigitalProduct`) が親クラス (`Product`) の「振る舞い（在庫数を**数値で**返す）」という**暗黙の期待**を破った（＝置換できない）ため、`Store`（利用者）側が`if`文（`isinstance`）による\*\*「修正」\*\*を強いられる典型的な例です。

`Store`が`DigitalProduct`という**具体的な子クラス**の存在を意識し始めた時点で、`Store`は抽象に依存しているとは言えなくなり、OCP違反（修正に対して閉じているべき）にも繋がります。

LSP違反とは、このように「**子クラスの振る舞いのせいで、親クラスの利用者に `if` 文（型チェックなど）を書かせる**」問題を引き起こすことです。

-----

## 💡 修正方針：契約（インターフェース）の改善

この問題を解決するには、`Store`側で`isinstance`を使って\*\*対処（Workaround）**するのではなく、`Product`（抽象）の**契約自体を修正（Fix）\*\*する必要があります。

`DigitalProduct`が「在庫数を数値で返せない」のは事実です。問題は、「在庫数を数値で返せない」クラスにまで`get_stock()`を**実装させてしまった**契約の曖昧さにあります。

そこで、契約をより厳密にし、`Store`が`if`文（型チェック）を使わずに済むようにします。

1.  `Product`（抽象）に、「**そもそも在庫管理の対象か？**」を問い合わせる契約 `is_stock_managed() -> bool` を追加します。
2.  `get_stock()` は、LSPを守るため「**在庫管理対象なら、必ず `int` を返す**」という厳密な契約に変更します。
3.  `DigitalProduct`は、`is_stock_managed()`を`False`と実装し、`get_stock()`は（`is_stock_managed()`が`False`の時は呼び出されない想定だが）契約遵守のため`0`を返すようにします。

これにより、`Store`は`isinstance`（型のチェック）の代わりに、`is_stock_managed()`（**能力のチェック**）を呼び出すだけでよくなります。

-----

## 💻 `logic.py` : LSP違反の修正

上記の方針で`logic.py`をリファクタリングします。

```python
import datetime
from abc import ABC, abstractmethod

# (Order, Repository 関連は OOP-06 と同じため省略)
class Order: ...
class IOrderRepository(ABC): ...
class InMemoryOrderRepository(IOrderRepository): ...


class Product(ABC):
    """
    【変更】商品の抽象クラス。契約をより厳密にする。
    """
    def __init__(self, product_id: str, name: str, price: int):
        self.product_id = product_id
        self.name = name
        self.price = price

    # --- check_stock, reduce_stock は変更なし ---
    @abstractmethod
    def check_stock(self, quantity: int) -> bool: ...
    @abstractmethod
    def reduce_stock(self, quantity: int): ...

    @abstractmethod
    def get_stock(self) -> int: # 【変更】契約を「必ず int を返す」に強化
        """【契約】現在の在庫数を「数値」で取得する"""
        pass
    
    @abstractmethod
    def is_stock_managed(self) -> bool: # 【新設】
        """【契約】在庫管理の対象商品かどうかを返す"""
        pass

class PhysicalProduct(Product):
    """
    物理商品クラス。（get_stock の型ヒント以外は変更なし）
    """
    def __init__(self, product_id: str, name: str, price: int, stock: int):
        super().__init__(product_id, name, price)
        self._stock = stock

    def check_stock(self, quantity: int) -> bool: ... # (省略)
    def reduce_stock(self, quantity: int): ... # (省略)

    def get_stock(self) -> int: # 【契約遵守】int を返す
        """【実装】物理在庫の数を返す。"""
        return self._stock
    
    def is_stock_managed(self) -> bool: # 【新設・実装】
        """【実装】物理商品は在庫管理の対象"""
        return True

class DigitalProduct(Product):
    """
    【変更】ダウンロード商品クラス。LSPを遵守するように修正。
    """
    def __init__(self, product_id: str, name: str, price: int):
        super().__init__(product_id, name, price)
        
    def check_stock(self, quantity: int) -> bool:
        """【実装】在庫は常に無限（変更なし）"""
        return True

    def reduce_stock(self, quantity: int):
        """【実装】在庫は減らない（変更なし）"""
        pass

    def get_stock(self) -> int: # 【変更】契約（intを返す）を遵守する
        """
        【実装】LSPを遵守するため、「数値」を返す。
        在庫の概念がないため 0 を返す。
        (このメソッドは is_stock_managed() が True の時だけ
         呼び出されるべき、というのが新しい契約の意図)
        """
        return 0 # "N/A" (str) を返すのはLSP違反だった
    
    def is_stock_managed(self) -> bool: # 【新設・実装】
        """【実装】ダウンロード商品は在庫管理の対象外"""
        return False

class Store:
    """
    【変更】店舗クラス。LSPが守られたことで、Store側がより安全になる。
    """
    def __init__(self, name: str, order_repository: IOrderRepository):
        self.name = name
        self._products: dict[str, Product] = {}
        self._order_repository = order_repository

    # (add_product, _find_product, process_order は変更なし)
    def add_product(self, product: Product): ...
    def _find_product(self, product_id: str) -> Product | None: ...
    def process_order(self, product_id: str, quantity: int): ...
    
    def get_current_stock_info(self) -> dict:
        """
        【変更】在庫表示ロジック。
        is_stock_managed() を使うことで、より明確な表示が可能になる。
        """
        stock_info = {}
        for p_id, p in self._products.items():
            # 新しい契約(is_stock_managed)に基づいて、Store側の振る舞いを決定
            # isinstance(p, DigitalProduct) のような型チェックは不要！
            if p.is_stock_managed():
                # 在庫管理対象なら、LSPにより必ず int が返ってくる
                stock_info[p.name] = p.get_stock()
            else:
                # 在庫管理対象外なら、Store側が "N/A" などを表示する
                stock_info[p.name] = "N/A (在庫管理対象外)"
        return stock_info

    def get_order_history(self) -> list[Order]: ... # (変更なし)

    # (LSP違反の特定で挙げた「悪い例」が、このように安全になる)
    def get_total_stock_count(self) -> int:
        """【新設】合計在庫数を計算するメソッド（LSP遵守により安全）"""
        total = 0
        for p in self._products.values():
            # p.get_stock() は「必ず int を返す」と契約されているため、
            # Store側は isinstance などの型チェックが一切不要になる。
            # 必要なのは「能力」のチェックだけ。
            if p.is_stock_managed():
                total += p.get_stock()
        return total
```

-----

## 🏛️ `main.py` : 利用者側の変更

`logic.py`の契約（`Product`インターフェース）が変更され、`Store`の内部実装（`get_current_stock_info`）も変わりましたが、`Store`を**利用**する`main.py`は`OOP-06`から一切変更ありません。

#### main.py

```python
# OOP-06 とまったく同じコード
from logic import (
    PhysicalProduct, DigitalProduct, Store, 
    IOrderRepository, InMemoryOrderRepository, Order
)

if __name__ == "__main__":
    
    # --- 1. 依存関係の構築 (DI) ---
    tokyo_repo: IOrderRepository = InMemoryOrderRepository()
    tokyo_store = Store("東京店", order_repository=tokyo_repo)
    osaka_repo: IOrderRepository = InMemoryOrderRepository()
    osaka_store = Store("大阪店", order_repository=osaka_repo)

    # --- 2. 商品のセットアップ (変更なし) ---
    tokyo_store.add_product(PhysicalProduct("p-001", "高機能マウス", 4000, 10))
    tokyo_store.add_product(DigitalProduct("d-001", "デザインソフト eBook", 8000))
    # (中略：他の商品の追加)
    
    # --- 3. 在庫表示と注文処理 (変更なし) ---
    print("--- 初期在庫 ---")
    # get_current_stock_info() の内部実装が変わったが、
    # 呼び出し側(main.py)は何も変更する必要がない。
    print(f"{tokyo_store.name}:", tokyo_store.get_current_stock_info())

    # (中略：注文処理)

    print("\n--- 最終在庫 ---")
    print(f"{tokyo_store.name}:", tokyo_store.get_current_stock_info())
    
    # --- 4. 注文履歴の表示 (変更なし) ---
    # (中略)
```

*(実行結果の `get_current_stock_info()` の表示が "N/A (ダウンロード可)" から "N/A (在庫管理対象外)" に変わりますが、これは `Store` のロジックがLSPを遵守して改善された結果です)*

-----

## ✨ この変更の効果

LSP違反を修正し、`Product`の「契約」をより厳密（`is_stock_managed()`の追加、`get_stock()`の戻り値を`int`に統一）にしました。

これにより、`Store`（利用者側）は、`Product`の種類（`Physical`か`Digital`か）を**一切気にすることなく**（`isinstance`不要）、安心して`get_stock()`（`int`が返ると保証されている）を呼び出せるようになり、コードの堅牢性と保守性がさらに向上しました。

`isinstance`による型チェックは、LSP違反の兆候であり、OCP違反（修正の必要性）に直結します。LSPを守ることで、OCPも同時に達成できる、より良い設計が実現しました。