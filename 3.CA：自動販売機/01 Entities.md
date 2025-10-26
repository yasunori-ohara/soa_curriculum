# 01 Entities

# 🧩 Entities

### `vending_machine/domain/entities.py`

本章では、システムの中核に位置する **Entities（エンティティ）** を定義します。

自動販売機というドメインにおいて、最も本質的なビジネスルールと状態を表現するのがエンティティです。
本教材の自動販売機では、主に次の2つが中心的なエンティティとして登場します。

* 🥤 `Item` … 自販機内の「1つのスロット（商品枠）」を表す
* 💰 `PaymentManager` … 投入金額の管理と、支払いに関するルールを表す

---

## 🎯 Entities層の役割

Entitiesは、アプリケーション全体の最も内側に位置します。
ここには、自動販売機として必ず守られるべき性質・不変条件・状態遷移ルールのみを記述します。

* 在庫が0の商品は販売できない
* 価格は0円以下ではならない
* 投入された金額が不足している場合は販売できない
* 支払い後は残額をリセットし、お釣りを計算する

この層は、外部事情（UIがコンソールなのかGUIなのか、在庫をどこに保存するのか、どのモーターで商品を落とすのか）を一切知りません。

> 🔒 まとめると、Entitiesは「自販機としての約束事（ビジネスルールそのもの）」を定義し、それを自ら守り続ける層です。
> ここが壊れると、他のどの層がどれだけ工夫しても正しい動作にはなりません。

---

## 📂 ファイル構成（Entities関連）

```text
vending_machine/
├─ domain/
│   ├─ entities.py   # Item, PaymentManager など
│   └─ errors.py     # ドメイン固有の例外（ビジネスルール違反用など）
│
└─ ...
```

* `entities.py` は「自動販売機のビジネスルールを担う中心的なクラス」をまとめる場所です。
* `errors.py` は、ビジネスルールに違反した際に送出する専用の例外クラスを定義する場所として利用できます。

---

## ✅ Entity 1 : `Item`

### 🍾 `Item` は何を表すか

`Item` は「自動販売機のスロット1枠」に対応します。
たとえば「A1スロットにはコーラが入っていて、価格は130円で、現在の在庫は9本」といった状態を表します。

このエンティティは、次のような責務を持ちます。

* 自分自身の状態（スロットID、名前、価格、在庫数）を保持する
* 「まだ在庫があるか？」という判定を行う
* 実際に販売したときに在庫を減らす。ただし不正な販売は拒否する

  * 在庫0の商品は dispensed（排出）できない
  * 在庫がマイナスになるような状態遷移は許さない

一方で、`Item` は次のことは行いません。

* データベースへの保存方法は知らない
* ハードウェア（モーター）をどう駆動して商品を落とすかは知らない
* お金が足りているかどうかの判定はしない（それは `PaymentManager` の責務）

### 💻 コード例（`Item`）

```python
# vending_machine/domain/entities.py

class Item:
    """
    自動販売機の1つのスロットに存在する「商品」を表すエンティティ。

    このクラスは「ビジネス上、商品スロットが取りうる正しい状態」を保証する。
    たとえば、在庫が負にならない、0円の商品が存在しない、といった不変条件を守る。
    """

    def __init__(self, slot_id: str, name: str, price: int, stock: int):
        """
        slot_id: スロット番号（例: "A1", "B2" のようなセル位置）
        name:    商品名（例: "お茶", "コーラ"）
        price:   価格（整数, 単位: 円）
        stock:   在庫本数（0以上の整数）

        ここでは、ビジネス的にあり得ない状態を初期化で排除する。
        """
        # 価格は1円以上でなければならない（0円やマイナス価格は許されない）
        if price <= 0:
            raise ValueError("価格は1円以上である必要があります。")

        # 在庫は0本以上でなければならない
        if stock < 0:
            raise ValueError("在庫数は0以上である必要があります。")

        self.slot_id = slot_id
        self.name = name
        self.price = price
        self.stock = stock

    def is_in_stock(self) -> bool:
        """
        現時点で販売可能な状態かどうかを返す。

        戻り値:
            True  -> 在庫が1本以上残っている
            False -> 在庫切れ
        """
        return self.stock > 0

    def dispense(self):
        """
        商品を1本「売る／排出する」操作に対応する。

        - 在庫が0の状態で呼ばれた場合はビジネスルール違反とみなし、例外を送出する。
        - 正常に販売できる場合は、在庫を1減らす。

        ここでは「モーターを回す」「商品を物理的に落とす」などの物理動作は扱わない。
        あくまで論理的な在庫数の更新のみを担当する。
        """
        if not self.is_in_stock():
            raise ValueError(f"商品「{self.name}」は在庫切れです。")
        self.stock -= 1

    def __repr__(self):
        """
        デバッグ用途の文字列表現。
        print(item) などを行った際に、読みやすい形で内部状態を確認できるようにする。
        """
        return (
            f"Item(slot='{self.slot_id}', "
            f"name='{self.name}', price={self.price}, stock={self.stock})"
        )
```

---

## ✅ Entity 2 : `PaymentManager`

### 💰 `PaymentManager` は何を表すか

`PaymentManager` は、自動販売機に投入された金額を管理し、支払いが正しく行われるように制御するエンティティです。

このエンティティが扱う内容は、主に次のとおりです。

* 現在投入されている金額の記録
* 利用可能な硬貨の種類の管理（例：10円・50円・100円・500円のみ許可する）
* 指定された商品を購入可能かどうかの判定
* 実際に購入を確定し、お釣りの計算を行い、投入済み金額をリセットする
* 途中キャンセル時に全額返金する処理

一方で、`PaymentManager` は次のことは行いません。

* 商品在庫を減らさない（それは `Item` の責務）
* 商品を物理的に排出させない（それはハードウェアアダプタの責務）
* 画面表示用の文言を作らない（それはPresenterの責務）

### 💻 コード例（`PaymentManager`）

```python
# vending_machine/domain/entities.py（Itemクラスの下などに続けて定義）

class PaymentManager:
    """
    自動販売機に投入された金額を管理するエンティティ。

    - 現在いくら入っているか
    - その金額で商品を購入できるか
    - 購入後にいくらお釣りを返すべきか

    といった、お金まわりのビジネスルールをまとめて担当する。
    """

    # 自販機が受け付ける硬貨の種類（ビジネスルール）
    VALID_COINS = {10, 50, 100, 500}

    def __init__(self, current_amount: int = 0):
        """
        current_amount:
            現時点で投入済みの合計金額（円）。
            例：100円玉×2枚なら 200。
        """
        self.current_amount = current_amount

    def insert_coin(self, coin: int):
        """
        ユーザーが硬貨を投入したときの処理。

        - 受け付け可能な硬貨（VALID_COINS）以外は拒否する。
        - 受け付けた硬貨の額面を現在の投入金額に加算する。

        ここでは物理的なコインメカ（硬貨を飲み込む装置）には触れない。
        純粋に論理的な金額計算のみを扱う。
        """
        if coin not in self.VALID_COINS:
            raise ValueError(f"{coin}円硬貨は使用できません。")
        self.current_amount += coin

    def is_sufficient(self, price: int) -> bool:
        """
        指定された価格の商品を購入できるだけの金額が
        すでに投入されているかどうかを判定する。
        """
        return self.current_amount >= price

    def process_purchase(self, item: Item) -> int:
        """
        「この商品を購入します」と確定したときの処理。

        - 投入金額が不足している場合は例外を送出し、購入を拒否する。
        - 購入が可能な場合は、お釣りを計算する。
        - その後、投入済み金額を0にリセットする。
          （1回の取引が完了したものとして扱う）

        戻り値:
            change (int): 返却すべきお釣りの金額（円）
        """
        if not self.is_sufficient(item.price):
            raise ValueError("投入金額が不足しています。")

        # お釣りを計算する
        change = self.current_amount - item.price

        # 取引完了後は、投入金額をクリアする
        self.current_amount = 0
        return change

    def return_change(self) -> int:
        """
        ユーザーが「やはり購入しません」とキャンセルした場合の処理。

        - 現在投入されている金額をそのまま返金する。
        - 返金後に、投入金額を0にリセットする。

        戻り値:
            返却した金額（円）
        """
        change = self.current_amount
        self.current_amount = 0
        return change

    def __repr__(self):
        """
        デバッグ用の文字列表現。
        例: PaymentManager(current_amount=150)
        """
        return f"PaymentManager(current_amount={self.current_amount})"
```

---

## 🧪 Entitiesはテストしやすい

`Item` と `PaymentManager` は、データベース・ハードウェア・ネットワーク・UIなど、外部の詳細に依存しません。
したがって、純粋なPythonオブジェクトとしてそのままユニットテストできます。

以下は簡単なテストイメージです。

```python
# tests/domain/test_entities.py

from vending_machine.domain.entities import Item, PaymentManager

def test_有効な硬貨を投入すると金額が加算される():
    pm = PaymentManager()
    pm.insert_coin(100)
    pm.insert_coin(50)
    assert pm.current_amount == 150

def test_十分な金額が投入されていれば購入できてお釣りが返る():
    pm = PaymentManager(current_amount=150)
    tea = Item(slot_id="A1", name="お茶", price=120, stock=10)

    change = pm.process_purchase(tea)

    assert change == 30          # 150円入っていて120円の商品を買った -> お釣り30円
    assert pm.current_amount == 0

def test_在庫ゼロの商品はdispenseできない():
    cola = Item(slot_id="B2", name="コーラ", price=130, stock=0)

    try:
        cola.dispense()
        assert False, "在庫0なら例外になるべき"
    except ValueError as e:
        assert "在庫切れ" in str(e)
```

このように、実機ハードウェアやDBがなくても、ローカルで安全にロジックを検証できます。
これは内側の層（Entities）が外側の詳細（Adapterやハードウェア）に依存していないことの直接的な利点です。

---

## 🐍 Python的な見方と組み込み的な見方

* 🐍 Python的な設計では、
  データ（状態）とそれを正しく扱うための操作（メソッド）を同じクラスにまとめます。
  例：`PaymentManager` が現在額とその正しい扱い手順を一体で保持します。
  →「お金のことはこのエンティティに問い合わせればよい」という責任の集中ができます。

* 🔧 組み込み寄りの従来スタイルでは、
  状態はグローバル構造体で持ち、処理は別関数で書かれがちです。
  その場合、「どこで誰が状態を壊したのか」を追跡しづらくなります。
  エンティティとして自己完結させることで、ビジネスルールを守る責任範囲が明確になります。

---

## 🛡 Entities層の鉄則

> 何物にも依存しない。
> ただしビジネスルールには妥協しない。

* `Item` は「在庫がマイナスになるべきではない」という不変条件を守ります。
* `PaymentManager` は「足りない金額では商品を売らない」という支払いルールを守ります。
* どちらも、UI・ハードウェア・データ保存方法・表示文言には依存しません。

この“純粋な中心”が守られることで、外側（UseCase、Adapter、ハードウェア実装など）は安心して差し替えることが可能になります。
次章では、このEntitiesをどのように組み合わせて実際の「購入フロー」などを進行させるのかを、UseCase層の観点から見ていきます。
