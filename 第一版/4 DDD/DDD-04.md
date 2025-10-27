# DDD-04

## `domain/order.py` (注文集約)

DDDリファクタリングの主役である、**domain/order.py**に進みましょう。ここがクリーンアーキテクチャからDDDへと思考を深める上で、最も重要なファイルです。

---

### 1. このファイルの役割：ビジネスの整合性を守る「集約」

この`domain/order.py`ファイルも、引き続きEntities（エンティティ）レイヤーに属します。しかし、Productが単一のビジネスオブジェクトだったのに対し、このファイルでは**複数のオブジェクトが一体となって一つのビジネス概念を形成する「集約（Aggregate）」**という、より高度なパターンを導入します。

### 集約（Aggregate）とは？

「注文（Order）」と「注文商品明細（LineItem）」のように、常にセットで扱われ、データの整合性が保たれなければならないオブジェクトのまとまりのことです。

- アナロジー: 紙の注文伝票
注文伝票（`Order`）と、そこに書かれた商品リスト（`LineItem`）は一体です。商品リストの項目を一つ追加したら、必ず伝票の合計金額も更新されなければなりません。この「伝票全体として、常に正しい状態を保つ」という責任を持つのが集約です。

この集約の中で、中心的な役割を果たし、外部との唯一の窓口となるオブジェクトを**集約ルート（Aggregate Root）**と呼びます。今回の例では`Order`クラスがその役目を担います。

### 2. ソースコードの詳細解説

`LineItem` クラス: 注文の明細（値オブジェクト）
このクラスは、「どの商品を、何個、いくらで」という注文の明細一件を表します。
これはDDDにおける**値オブジェクト（Value Object）**の一例です。値オブジェクトは、それ自体に独立したIDを持つのではなく、その属性（商品IDや数量）によって定義されるオブジェクトです。

`Order` クラス: 注文伝票（集約ルート）
このクラスが、今回の設計の核心です。

- `__init__(...)`: `Order`は、一意の注文ID（`order_id`）を持つため、値オブジェクトではなくエンティティです。
- `add_line_item(...)`:
これが豊かなドメインモデルの真骨頂です。このメソッドは、単にリストに商品を追加するだけではありません。
1. 商品の在庫を確認する（`product.check_stock`）
2. 在庫があれば、新しい`LineItem`を生成する
3. 合計金額を更新する
という、一連のビジネスルールをカプセル化しています。以前はUse Caseが担っていたロジックの一部が、ドメインモデル自身に移動してきました。
- `confirm()`:
注文を確定するという、ビジネス上の重要なイベントを表すメソッドです。このメソッドが呼ばれると、`Order`は自分が保持している`Product`オブジェクトたちに「在庫を減らせ」と命令します。集約ルートである`Order`が、自身の整合性を保つためのすべての処理を指揮しています。
- `total_price` プロパティ:
合計金額は、常に内部の`_line_items`から計算されます。これにより、外部から合計金額だけを不正に書き換えられることがなく、データの整合性が常に保たれます。

### 3. このレイヤーの鉄則

1. 豊かなドメインモデル: エンティティは、単なるデータの入れ物（貧血ドメインモデル）であってはなりません。それ自身がビジネスルールを持ち、振る舞うべきです。
2. 整合性の境界: 集約は、その内部の状態が常にビジネスルール上、正しい状態であることを保証する責任を持ちます。
3. 集約ルートが唯一の窓口: 集約の外部から、`LineItem`を直接操作することは許されません。集約に対するすべての操作は、必ず集約ルートである`Order`クラスを通じて行われなければなりません。

---

```
# 依存性のルール:
# このファイルはEntitiesレイヤーに属します。
# 同じレイヤーの Product エンティティに依存しますが、
# 他のどのレイヤーにも依存しません。

import datetime
from .product import Product

class LineItem:
    """
    【Entitiesレイヤー / Value Object】
    注文の明細一件を表す「値オブジェクト」。

    それ自体に独立したIDを持つのではなく、その属性によって定義される。
    「どの商品を、何個、いくらで」という情報を持つ。
    """
    def __init__(self, product: Product, quantity: int):
        if quantity <= 0:
            raise ValueError("数量は1以上でなければなりません。")
        self.product = product
        self.quantity = quantity

    @property
    def total_price(self) -> int:
        """この明細の合計金額を計算するビジネスルール。"""
        return self.product.price * self.quantity

class Order:
    """
    【Entitiesレイヤー / Aggregate Root & Entity】
    「注文」というビジネス概念を表す「集約ルート」。

    LineItemを内部に持ち、注文全体としての一貫性を保つ責任を持つ。
    注文IDを持つため、それ自体もエンティティである。
    """
    def __init__(self, order_id: str):
        self.order_id = order_id
        self._line_items: list[LineItem] = []
        self._total_price: int = 0
        self.confirmed: bool = False

    @property
    def total_price(self) -> int:
        """合計金額は、常に内部の明細から計算されることを保証する。"""
        return sum(item.total_price for item in self._line_items)

    @property
    def line_items(self) -> list[LineItem]:
        return self._line_items

    def add_line_item(self, product: Product, quantity: int):
        """
        注文に商品を追加するビジネスルール。
        このメソッドは、単にリストに値を追加するだけでなく、
        在庫チェックという重要なビジネスルールをカプセル化している。
        """
        if self.confirmed:
            raise ValueError("確定済みの注文は変更できません。")
        if not product.check_stock(quantity):
            raise ValueError(f"{product.name}の在庫が不足しています。")

        self._line_items.append(LineItem(product, quantity))
        print(f"注文明細追加: {product.name} x {quantity}")

    def confirm(self):
        """
        注文を確定するビジネスルール。
        このメソッドが呼ばれると、注文は変更不可になり、
        関連する商品の在庫が実際に減らされる。
        """
        if not self._line_items:
            raise ValueError("注文明細が空です。")

        for item in self._line_items:
            item.product.reduce_stock(item.quantity)

        self.confirmed = True
        print(f"注文確定: {self.order_id}")

```