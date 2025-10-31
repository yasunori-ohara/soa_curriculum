# DDD-03 : 🔵 `domain/order.py` (注文集約)

DDDリファクタリングの主役である、`domain/order.py` に進みましょう。ここがクリーンアーキテクチャからDDDへと思考を深める上で、最も重要なファイルです。

`CA` の `domain/order.py` で定義した `Order` クラスは、`product_name` などを保持する、単なる「データの入れ物（貧血なドメインモデル）」でした。

`DDD` では、この `Order` を、\*\*ビジネスルールを持った「リッチなドメインモデル」\*\*に進化させます。

## 🎯 この章のゴール

  * DDDの「集約（Aggregate）」と「集約ルート（Aggregate Root）」の概念を理解する。
  * 「値オブジェクト（Value Object）」の概念を `LineItem` で理解する。
  * 「リッチなドメインモデル」を実装し、`UseCase` が持っていたビジネスロジックを `Order` エンティティ自身に移譲する。

-----

## 💎 このファイルの役割：ビジネスの整合性を守る「集約」

このファイルも `Entities` レイヤーに属しますが、`Product`（単一のオブジェクト）とは異なり、\*\*複数のオブジェクトが一体となって一つのビジネス概念を形成する「集約（Aggregate）」\*\*という、より高度なパターンを導入します。

### 集約（Aggregate）とは？

「注文（`Order`）」と「注文商品明細（`LineItem`）」のように、常にセットで扱われ、データの整合性が保たれなければならないオブジェクトのまとまりのことです。

  * **アナロジー: 紙の注文伝票**
    注文伝票（`Order`）と、そこに書かれた商品リスト（`LineItem`）は一体です。商品リストの項目を一つ追加したら、必ず伝票の合計金額も（概念上）更新されなければなりません。この「伝票全体として、常に正しい状態を保つ」という責任を持つのが集約です。

この集約の中で、中心的な役割を果たし、外部との唯一の窓口となるオブジェクトを\*\*集約ルート（Aggregate Root）\*\*と呼びます。今回の例では `Order` クラスがその役目を担います。

-----

## ✨ CAからの進化：ロジックの移譲

`CA-04` の `ProcessOrderUseCase` は、以下のような**ビジネスロジック**を持っていました。

1.  `Product` を見つける
2.  `product.check_stock()` を呼ぶ
3.  `product.reduce_stock()` を呼ぶ
4.  `Order` オブジェクト（データの入れ物）を作る
5.  `order_repo.save()` を呼ぶ

DDDでは、これらのロジックの**一部**を、`Order` 集約自身が担当するように移します。`Order` が単なる「データ」から、自分で考え、振る舞う「**主体**」に変わるのです。

-----

## 💻 ソースコードの詳細解説

### `LineItem` クラス: 注文の明細（値オブジェクト）

このクラスは、「どの商品を、何個」という注文の明細一件を表します。
これはDDDにおける\*\*値オブジェクト（Value Object）\*\*の一例です。値オブジェクトは、それ自体に独立したIDを持つのではなく、その属性（`product` や `quantity`）によって定義されます。

### `Order` クラス: 注文伝票（集約ルート）

このクラスが、今回の設計の核心です。

  * `__init__(...)`:
    `Order` は、一意の `order_id` を持つため、値オブジェクトではなく**エンティティ**です。
  * `add_line_item(...)`:
    **これがリッチなドメインモデルの真骨頂です。** `UseCase` が行っていた在庫チェック (`product.check_stock`) を、`Order` が明細追加時の**ビジネスルール**として自身で実行します。
  * `total_price` (プロパティ):
    合計金額は、常に内部の `_line_items` から計算されます。これにより、外部から合計金額だけを不正に書き換えられることがなく、データの**整合性**が常に保たれます。
  * `confirm()`:
    注文を確定するという、ビジネス上の重要なイベントを表すメソッドです。`UseCase` が行っていた `product.reduce_stock()` を、`Order` が自身の `confirm` 処理の一部として実行します。

-----

## 🏛️ このレイヤーの鉄則

1.  **リッチなドメインモデル:**
    エンティティは、単なるデータの入れ物（貧血）であってはなりません。それ自身がビジネスルール（`add_line_item` や `confirm`）を持ち、振る舞うべきです。
2.  **整合性の境界:**
    集約（`Order`）は、その内部の状態（`line_items` と `total_price` の関係など）が、常にビジネスルール上、正しい状態であることを保証する責任を持ちます。
3.  **集約ルートが唯一の窓口:**
    集約の外部（`UseCase` など）から、`LineItem` を直接操作することは許されません。集約に対するすべての操作は、必ず集約ルートである `Order` クラスを通じて行われなければなりません。

-----

## 📄 `domain/order.py`

（`CA-02` の `Order` クラスをリファクタリングします）

```python:domain/order.py
# 依存性のルール:
# このファイルは Entities レイヤーに属します。
# 同じレイヤーの Product エンティティに依存しますが、
# 他のどのレイヤーにも依存しません。

import datetime
import uuid # 注文IDをユニークにするために使用

# 「内側」の domain レイヤーから Product をインポート
from .product import Product

class LineItem:
    """
    【Entitiesレイヤー / Value Object】
    注文の明細一件を表す「値オブジェクト」。

    それ自体に独立したIDを持つのではなく、その属性によって定義される。
    「どの商品を、何個」という情報を持つ。
    """
    def __init__(self, product: Product, quantity: int):
        if quantity <= 0:
            raise ValueError("数量は1以上でなければなりません。")
        
        # Productエンティティへの参照を持つ
        self.product = product 
        self.quantity = quantity

    @property
    def total_price(self) -> int:
        """この明細の合計金額を計算するビジネスルール。"""
        return self.product.price * self.quantity
    
    def __repr__(self):
        return f"<LineItem: {self.product.name} x {self.quantity}>"

class Order:
    """
    【Entitiesレイヤー / Aggregate Root & Entity】
    「注文」というビジネス概念を表す「集約ルート」。

    LineItemを内部に持ち、注文全体としての一貫性を保つ責任を持つ。
    注文IDを持つため、それ自体もエンティティである。
    """
    def __init__(self, order_id: str, customer_id: str):
        # (簡略化のため、顧客IDを追加)
        if not order_id:
             # IDが指定されない場合は自動生成する（不変条件）
            order_id = str(uuid.uuid4())
            
        self.order_id = order_id
        self.customer_id = customer_id
        self._line_items: list[LineItem] = []
        self._created_at = datetime.datetime.now()
        self._confirmed: bool = False

    @property
    def total_price(self) -> int:
        """
        合計金額は、常に内部の明細から計算されることを保証する。
        これにより、データが不整合な状態になることを防ぐ。
        """
        return sum(item.total_price for item in self._line_items)

    @property
    def line_items(self) -> list[LineItem]:
        """外部からは読み取り専用として明細リストを公開する"""
        return self._line_items

    def add_line_item(self, product: Product, quantity: int):
        """
        【リッチなドメインロジック 1】
        注文に商品を追加するビジネスルール。
        
        CAでは UseCase が行っていた「在庫チェック」を、
        エンティティ自身が「不変条件（ルール）」として実行する。
        """
        if self._confirmed:
            raise ValueError("確定済みの注文は変更できません。")

        # UseCase が行っていた在庫チェックを、ドメインが担当
        if not product.check_stock(quantity):
            raise ValueError(f"{product.name}の在庫が不足しています。")

        self._line_items.append(LineItem(product, quantity))
        print(f"ドメイン: {product.name} x {quantity} を注文明細に追加")

    def confirm(self):
        """
        【リッチなドメインロジック 2】
        注文を確定するビジネスルール。
        
        CAでは UseCase が行っていた「在庫の削減」を、
        エンティティが自身の状態遷移（確定）の責務として実行する。
        """
        if self._confirmed:
            raise ValueError("注文はすでに確定済みです。")
            
        if not self._line_items:
            raise ValueError("注文明細が空です。注文を確定できません。")

        # UseCase が行っていた在庫削減を、ドメインが担当
        for item in self._line_items:
            item.product.reduce_stock(item.quantity)
            print(f"ドメイン: {item.product.name} の在庫を {item.quantity} 減らしました。")

        self._confirmed = True
        print(f"ドメイン: 注文 {self.order_id} が確定されました。")

    def __repr__(self):
        return f"<Order: {self.order_id}, Total: {self.total_price}, Confirmed: {self._confirmed}>"
```


## 📖 (補足) なぜ「集約（Aggregate）」が重要なのか？

`DDD-03` で行ったリファクタリングは、単にコードを `Order` クラスに移動しただけではありません。これは、`CA`（クリーンアーキテクチャ）から `DDD` へと移行する上で、**最も重要なパラダイムシフト**です。

### 1. 🛡️ ビジネスルールの「真の番人」を決める
`CA` の設計では、`ProcessOrderUseCase` がビジネスルールの「番人」でした。
`UseCase` が「① `product.check_stock()` して、OKなら ② `product.reduce_stock()` して、③ `Order` を作る」という**手順**を知っていました。

**これは危険です。** もし、別のユースケース（例：`CancelOrderUseCase`）が作られたとして、その開発者が「在庫を戻す」処理を忘れたらどうなるでしょう？ ビジネスの整合性（`domain`）が、`use_cases` レイヤーの「うっかりミス」によって簡単に壊れてしまいます。

`DDD` の「集約」は、この責任を逆転させます。
`Order` 集約（アグリゲート）自身が「**自分のことは自分で守る**」という、リッチなドメインモデル（豊かなドメインモデル）になります。

* `Order` の `add_line_item` メソッドが、`check_stock` を**内包**します。
* `Order` の `confirm` メソッドが、`reduce_stock` を**内包**します。

`UseCase` は、もはや「手順」を知りません。ただ「`Order` 集約よ、明細を追加せよ（`add_line_item`）」と**「依頼」**するだけです。`Order` 自身が「番人」として、在庫チェックや整合性チェックというビジネスルールを実行します。

これにより、`use_cases` がどれだけ増えようとも、`domain` のビジネスルールが破られることはありません。

### 2. 🗃️ 「トランザクションの単位」を明確にする
`Order` と `LineItem` は、**「常に一体」**でなければなりません。`Order` だけ保存されて `LineItem` が保存されない、といった事態はビジネス上のエラーです。

「集約」は、`use_cases` レイヤーや `interface_adapters` レイヤー（リポジトリ）に対して、「**私（`Order` 集約）を保存するときは、私の配下にある `LineItem` も含めて、すべてを一つの作業単位（トランザクション）として扱え**」という強力な境界線を引きます。

リポジトリは、集約ルート（`Order`）経由でのみ、データの保存・取得を行うべき、というルールが生まれます。

この「**整合性の境界**」と「**トランザクションの境界**」を明確に定義することこそが、`UseCase` を薄く保ち、ビジネスの複雑さに立ち向かうための「集約」の真の力なのです。