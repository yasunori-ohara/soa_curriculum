# 🧩 Entities : domain/entities.py

新しい題材「シンプルな自動販売機」のプログラムを、これまでと同じ詳細な解説形式で構築していきましょう。

まずはシステムの心臓部である`Entities`から始めます。このシステムには`Item`（商品）と`PaymentManager`（金銭管理）という2つの主要なEntityが存在します。

## 🎯 このファイル（クラス群）の役割

このファイルに定義される`Item`クラスと`PaymentManager`クラスは、アーキテクチャの最も内側に位置する`Entities`です。アプリケーション全体の核となるビジネスルールとデータをカプセル化（内包）します。

これらは、自動販売機というビジネスドメインそのものにとって普遍的なルールとデータを表現します。

平たく言えば、「このビジネスにおける『商品』や『金銭管理』とは何か、そしてそれらが従うべきルールは何か」を定義する場所です。

## ✅ Entity 1 : `Item`

`Item`クラスは、自動販売機の商品スロット一つ一つを表現するEntityです。

商品のデータ（名前、価格、在庫数）を保持するだけでなく、「在庫がなければ商品は排出できない」といった、商品そのものに関する純粋なビジネスルールを内包します。

### このクラスにあるべき処理

⭕️ 含めるべき処理の例:

  * 商品に必須のプロパティ（スロットID, 名前, 価格, 在庫数）。
  * 自身の状態をチェックするメソッド（例：`is_in_stock()`）。
  * 自身の状態を変更するロジックと、それに伴うルール（例：`dispense()`メソッド内で在庫数をチェックし、1減らす）。

❌ 含めてはいけない処理の例:

  * データベースへの保存処理。
  * 投入金額に関する知識（それは`PaymentManager`の責務です）。
  * 商品を物理的に排出するモーターの制御など、ハードウェアに関する知識。

### 💻 ソースコードの詳細解説

```python
# domain/entities.py

# -----------------------------------------------------------------------------
# Item Entity
# - クラス図の位置: Entities
# - 同心円図の位置: Entities (最も内側)
# -----------------------------------------------------------------------------
class Item:
    """商品エンティティ"""
    def __init__(self, slot_id: str, name: str, price: int, stock: int):
        """
        [データ]
        商品が本質的に持つべきデータ（属性）を定義する。
        """
        # 不変条件のチェック（自己バリデーション）
        if price <= 0 or stock < 0:
            raise ValueError("価格や在庫に負の値は設定できません。")
        
        self.slot_id = slot_id
        self.name = name
        self.price = price
        self.stock = stock

    def is_in_stock(self) -> bool:
        """[ビジネスルール] 在庫があるかどうかを判断する"""
        return self.stock > 0

    def dispense(self):
        """[ビジネスルール] 商品を1つ排出する際の、在庫の状態遷移ルール"""
        if not self.is_in_stock():
            raise ValueError(f"商品「{self.name}」は在庫切れです。")
        self.stock -= 1

    def __repr__(self):
        return f"Item(slot='{self.slot_id}', name='{self.name}', price={self.price}, stock={self.stock})"
```

  * `__init__`: 価格や在庫数が不正な値でないかをチェックするバリデーションルールを持っています。
  * `dispense()`: このEntityの核となるビジネスルールです。まず在庫を確認し、なければエラーを発生させます。在庫がある場合のみ、自身の`stock`を1減らします。

## ✅ Entity 2 : `PaymentManager`

`PaymentManager`クラスは、自動販売機の金銭管理の状態とルールを表現するEntityです。

現在投入されている金額を追跡し、「投入金額が足りているか」「お釣りはいくらか」といった、お金に関する純粋なビジネスルールを内包します。

### このクラスにあるべき処理

⭕️ 含めるべき処理の例:

  * 現在の投入金額を保持するプロパティ。
  * 状態を変更するメソッド（例：`insert_coin()`, `process_purchase()`）。
  * 自身のデータだけで完結する計算（例：お釣りの計算）。
  * 投入された硬貨が有効かどうかのチェック。

❌ 含めてはいけない処理の例:

  * どの商品が買われようとしているかの知識。ただし、購入処理の際には商品情報(`Item`)そのものを受け取り、価格を尋ねるのは良い設計です。
  * 物理的なコイン識別機や、お釣り排出装置の制御。

### 💻 ソースコードの詳細解説

```python
# domain/entities.py (続き)

# -----------------------------------------------------------------------------
# PaymentManager Entity
# - クラス図の位置: Entities
# - 同心円図の位置: Entities (最も内側)
# -----------------------------------------------------------------------------
class PaymentManager:
    """金銭管理エンティティ"""
    VALID_COINS = {10, 50, 100, 500} # 有効な硬貨の種類というビジネスルール

    def __init__(self, current_amount: int = 0):
        """
        [データ]
        現在の投入金額を状態として保持する。
        """
        self.current_amount = current_amount

    def insert_coin(self, coin: int):
        """[ビジネスルール] 有効な硬貨が投入されたら、投入金額を増やす"""
        if coin not in self.VALID_COINS:
            raise ValueError(f"{coin}円硬貨は使用できません。")
        self.current_amount += coin

    def is_sufficient(self, price: int) -> bool:
        """[ビジネスルール] 投入金額が価格に対して十分かを判断する"""
        return self.current_amount >= price

    def process_purchase(self, item: Item) -> int:
        """
        [ビジネスルール] 購入を処理し、お釣りを計算する。
        投入金額はリセットされる。
        Itemオブジェクト自身を渡すことで、価格の取り違えを防ぎ、責務の連携がより明確になる。
        """
        if not self.is_sufficient(item.price):
            raise ValueError("投入金額が不足しています。")
        
        change = self.current_amount - item.price
        self.current_amount = 0
        return change

    def return_change(self) -> int:
        """[ビジネスルール] 現在の投入金額をそのままお釣りとして返し、リセットする"""
        change = self.current_amount
        self.current_amount = 0
        return change

    def __repr__(self):
        return f"PaymentManager(current_amount={self.current_amount})"
```

  * `VALID_COINS`: 投入を許可する硬貨の種類を定義しています。これも純粋なビジネスルールです。
  * `process_purchase()`: このEntityのもう一つの核となるビジネスルールです。単なる価格(`price`)ではなく`Item`オブジェクトそのものを受け取ることで、`UseCase`は`PaymentManager`に「この商品の購入を処理して」と依頼するだけで済み、より安全で責務が明確になります。

## 💡 ユニットテストでEntityの正しさを証明する

EntityはハードウェアやDBに依存しないため、簡単に単体でテストできます。

```python
# tests/domain/test_entities.py の例
import pytest
from domain.entities import PaymentManager, Item

def test_有効な硬貨は投入できる():
    # 1. Arrange (準備)
    pm = PaymentManager()

    # 2. Act (実行)
    pm.insert_coin(100)
    pm.insert_coin(50)

    # 3. Assert (検証)
    assert pm.current_amount == 150

def test_金額が足りていれば購入処理が成功する():
    # 1. Arrange (準備)
    pm = PaymentManager(current_amount=150)
    tea = Item(slot_id="A1", name="お茶", price=120, stock=10)

    # 2. Act (実行)
    change = pm.process_purchase(tea)

    # 3. Assert (検証)
    assert change == 30          # お釣りが正しいか
    assert pm.current_amount == 0 # 投入金額がリセットされたか
```

## 🐍 PythonとC言語の比較（初心者の方へ）

  * Python (オブジェクト指向): `PaymentManager`というクラスが、データ（`current_amount`）とルール（`insert_coin`メソッド）を一つにまとめて管理します。
  * C言語 (手続き型): おそらく、`VendingMachineState`のような巨大な構造体（`struct`）の中に`current_amount`というメンバがあり、`insert_coin(struct VendingMachineState* state, int coin)`のような外部関数がその構造体を変更する、という形になるでしょう。データとロジックが分離しているため、どこで何が変更されるのかを追跡するのが難しくなりがちです。

## 🛡️ このクラス群の鉄則

Entity層の鉄則は、これまでと全く同じです。

> *何物にも依存するな (Depend on Nothing)*

  * 自動販売機の基本的なルール（「お金が足りなければ買えない」など）は、物理的な筐体のデザインや通信機能の有無が変わっても、不変です。
  * `UseCase`や`Adapters`といった外側のレイヤーについて一切知ってはいけません。
  * Entityは、アプリケーションの他の部分から利用されるだけの存在です。