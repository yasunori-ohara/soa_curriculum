# 01 Entities

# 🧩 Entities
### `vending_machine/domain/entities.py`

ここから実装をスタートします。最初に定義するのは **Entities（エンティティ）** です。

自動販売機を動かすうえで、いちばん核心になる「ビジネスルール」と「状態」を表すクラスが `Entity` です。
この自販機では、主に 2 つのEntityが存在します：

* 🥤 `Item` ……「商品スロット1枠」を表す
* 💰 `PaymentManager` ……「投入金額の管理と支払いルール」を表す

---

## 🎯 Entities層とは？

* アプリ全体の **いちばん内側** にある層です。
* 自販機として**守らなきゃいけないルール（在庫が無いものは売れない／足りないお金では買えない）**がここに書かれます。
* 外の世界（UIが何か、DBは何を使うか、ハードウェアはどう動くか）には一切依存しません。

> 🔒 つまり「自販機としての約束」を定義する場所。
> ここが壊れると、外側がどれだけ頑張っても正しく動かない。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## ✅ Entity 1 : `Item`

### 🍾 何を表すクラス？

`Item` は、自動販売機の「スロット1つぶんの商品状態」を表します。

* “A1の枠に入っているお茶は120円で9本残っている”
  みたいな情報を1つのオブジェクトで持ちます。

### 👀 `Item` が責任を持つこと

* 自分の属性（スロットID・商品名・価格・在庫数）を正しく保持する
* 「在庫はある？」という判断
* 実際に1本売ったときの在庫の更新と、そのルール

  * 在庫が0のときは売れない（例外を出す）

ここで大事なポイント：

* ❌ データベースへの保存の仕方は知らない
* ❌ 実際にモーターを回して商品を落とす方法も知らない
* ❌ “お金が足りてるか” の判定も担当しない（それは後述の `PaymentManager` の責任）

Entityは、自分に関する正しさだけ守ればいい。その潔さが大事です。

## 💻 フォルダ構成

```text
vending_machine/
├─ domain/
│   ├─ entities.py          # Item, PaymentManager など     <- いまここ
│   ├─ errors.py            # ドメインルール違反例外
│   └─ repositories.py      # DataAccessInterface / HardwareInterface
```
### 💻 コード（`Item`）

```python
# vending_machine/domain/entities.py

# -----------------------------------------------------------------------------
# Item Entity
# - 役割: 自動販売機の「1つのスロットに入っている商品」を表す
# - 責務: 在庫数や価格などの本質的な状態と、その状態遷移ルールを管理する
# -----------------------------------------------------------------------------
class Item:
    """自動販売機の1スロット分の商品を表すエンティティ"""

    def __init__(self, slot_id: str, name: str, price: int, stock: int):
        """
        slot_id: 自販機内のスロット番号（例: "A1" や "B3"）
        name:    商品名（例: "お茶"、"コーラ"）
        price:   商品の価格（整数, 単位: 円）
        stock:   在庫本数（0 以上の整数）

        ここでは「この商品がそもそもあり得る状態なのか？」をチェックする。
        例えば「-5円のジュース」や「在庫-3本」は現実的におかしいので禁止する。
        """
        # ✅ エンティティは自分自身を壊れた状態にしない
        if price <= 0:
            raise ValueError("価格は1円以上である必要があります。")
        if stock < 0:
            raise ValueError("在庫数は0以上である必要があります。")

        self.slot_id = slot_id
        self.name = name
        self.price = price
        self.stock = stock

    def is_in_stock(self) -> bool:
        """
        [ビジネスルール] 今この商品は売れる状態か？

        戻り値:
            True  -> まだ在庫がある
            False -> 売り切れ
        """
        return self.stock > 0

    def dispense(self):
        """
        [ビジネスルール] 商品を1本出荷（排出）する。

        - 在庫が0本ならエラーにする（売ってはいけない）
        - 正常に売れた場合は在庫を1本減らす

        ※ ここでは「モーターを回す」などの物理作業は一切しない。
           それはハードウェア層(interface_adapters)の責務。
           Itemはあくまで「在庫を1本減らすのは合法か？」だけを見る。
        """
        if not self.is_in_stock():
            raise ValueError(f"商品「{self.name}」は在庫切れです。")
        self.stock -= 1

    def __repr__(self):
        """
        デバッグ用の文字列表現。
        print(item) したときに読みやすい形で表示できるようにする。
        """
        return (
            f"Item(slot='{self.slot_id}', "
            f"name='{self.name}', price={self.price}, stock={self.stock})"
        )
```

---

## ✅ Entity 2 : `PaymentManager`

### 💰 何を表すクラス？

`PaymentManager` は、自動販売機に入れられたお金の扱いを管理するエンティティです。

この人は「今、いくら入ってる？」「この商品を買えるほど入ってる？」「お釣りは？」といった**お金の正しさ**を保証します。

### 👀 `PaymentManager` が責任を持つこと

* 現在の投入金額を保持・更新する
* 有効な硬貨かどうかをチェックする（例：10円・50円・100円・500円だけOK）
* 商品を買おうとしたときに

  * お金が足りるかどうか判定する
  * 実際に購入を成立させる
  * お釣りの金額を決める
  * 状態（投入金額）をリセットする

ここで大事なポイント：

* ❌ `PaymentManager` はハードウェアコインメカ（コインを物理的に戻す装置）を知らない
* ❌ `PaymentManager` は在庫操作をしない（それは `Item` の責務）
* ✅ 「金の話しかしない」ようにとにかく絞ってある

---

### 💻 コード（`PaymentManager`）

```python
# vending_machine/domain/entities.py (つづき)

# -----------------------------------------------------------------------------
# PaymentManager Entity
# - 役割: 現在投入されている金額と、その金額でできることを管理する
# - 責務:
#     - 硬貨の受け入れルール
#     - 購入可能判定
#     - お釣りの計算
#     - 取引後のリセット
# -----------------------------------------------------------------------------
class PaymentManager:
    """自動販売機に投入された金額を管理するエンティティ"""

    # 自販機が受け付ける硬貨の種類（ビジネスルール）
    VALID_COINS = {10, 50, 100, 500}

    def __init__(self, current_amount: int = 0):
        """
        current_amount:
            現時点で投入済みの金額（円）
            例えば100円玉×2枚入れたら200になるイメージ
        """
        self.current_amount = current_amount

    def insert_coin(self, coin: int):
        """
        [ビジネスルール] 硬貨が投入されたときの処理。

        - 有効な硬貨（10, 50, 100, 500円）だけを受け付ける
        - それ以外はエラーとして拒否する
        - 受け付けたら現在の投入金額に加算する
        """
        if coin not in self.VALID_COINS:
            raise ValueError(f"{coin}円硬貨は使用できません。")
        self.current_amount += coin

    def is_sufficient(self, price: int) -> bool:
        """
        [ビジネスルール] 指定された価格の商品を買うだけの金額が
        すでに投入されているかどうかを判定する。
        """
        return self.current_amount >= price

    def process_purchase(self, item: Item) -> int:
        """
        [ビジネスルール] 商品を購入しようとする処理。

        - 現在の投入金額が足りていなければエラー
        - 足りていれば購入成功として扱い、お釣りを計算する
        - お釣りの金額（int）を返す
        - この取引が終わったので投入金額を0にリセットする

        引数:
            item: 購入対象の商品そのもの
                  -> item.price を見ることで価格を参照する

        戻り値:
            change(int): お釣りの金額
        """
        if not self.is_sufficient(item.price):
            raise ValueError("投入金額が不足しています。")

        # お釣り = 今入っている金額 - 商品の価格
        change = self.current_amount - item.price

        # 1回の購入が終わったので、投入されていたお金はいったんクリア
        self.current_amount = 0
        return change

    def return_change(self) -> int:
        """
        [ビジネスルール] キャンセル等で「やっぱり買いません」となった場合。

        - いま入っている金額をまるごと返す（お釣りとして返却）
        - その後、投入金額は0にリセットする
        """
        change = self.current_amount
        self.current_amount = 0
        return change

    def __repr__(self):
        """
        デバッグ用表示。
        print(payment_manager) としたときに
        "PaymentManager(current_amount=150)" のように可視化できる。
        """
        return f"PaymentManager(current_amount={self.current_amount})"
```

---

## 🧪 テストはめちゃくちゃ書きやすい

ここまでの2つのEntity (`Item`, `PaymentManager`) は、
データベースもネットワークもハードウェア（モーターやコインメカ）も知りません。

つまり、**ふつうのPythonオブジェクトとしてそのままテストできます。**

```python
# tests/domain/test_entities.py のイメージ
from vending_machine.domain.entities import PaymentManager, Item

def test_有効な硬貨は投入できる():
    pm = PaymentManager()
    pm.insert_coin(100)
    pm.insert_coin(50)
    assert pm.current_amount == 150

def test_投入金額が足りていれば購入できてお釣りが返る():
    pm = PaymentManager(current_amount=150)
    tea = Item(slot_id="A1", name="お茶", price=120, stock=10)

    change = pm.process_purchase(tea)

    assert change == 30            # 150円入っていて120円のものを買えば30円お釣り
    assert pm.current_amount == 0  # 購入後はリセットされている

def test_在庫がない商品はdispenseできない():
    cola = Item(slot_id="B2", name="コーラ", price=130, stock=0)

    try:
        cola.dispense()
        assert False, "在庫0なら例外になるはず"
    except ValueError as e:
        assert "在庫切れ" in str(e)
```

これが「エンティティが外の事情を知らない」ことの強さです。
どんな環境でも、ローカルですぐ検証できます。

---

## 🐍 Python視点 / C言語視点

* 🐍 Python的な設計では、`PaymentManager` のように「状態（current_amount）＋その状態を正しく扱うためのメソッド（insert_coin, process_purchase）」をひとつのまとまりとして扱います。
  → 「お金のことはこの人に聞けばいい」がはっきりする。

* 🔧 C言語的な設計では、よくこうなります：

  * 状態は `struct VendingMachineState { int current_amount; ... };`
  * ロジックは別の関数 `void insert_coin(struct VendingMachineState* s, int coin)` などで操作する
  * その結果、「どの関数がどこで状態を壊しているのか」を追うのが難しくなりがち

エンティティとしてクラス化してしまうことで、
「このデータには、この正しい使い方がセットになっている」という約束を明文化できるのがオブジェクト指向のいいところです。

---

## 🛡️ Entities層の鉄則

> 何物にも依存するな。
> ただしビジネスルールには絶対に妥協するな。

* `Item` は「在庫がマイナスになる」というありえない状態を拒否します。
* `PaymentManager` は「足りないお金では売らない」という約束を徹底します。
* どちらも、ハードウェアやUIやDBに左右されません。

この“純粋な中心”を確立したうえで、次のステップでは
**UseCase層（usecase/）** に進みます。
そこでは、複数のEntityを組み合わせて「実際の操作フロー」（お金入れる→商品選ぶ→商品が出る）を実現させていきます。
