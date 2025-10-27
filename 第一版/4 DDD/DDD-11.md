# DDD-11

```markdown
はい、承知いたしました。「貧血ドメインモデル」について、具体例を交えながら解説します。

-----

### 🩸 貧血ドメインモデル (Anemic Domain Model) とは？

**貧血ドメインモデル**とは、ドメインオブジェクト（エンティティや値オブジェクト）が、**振る舞い（ビジネスロジック）をほとんど持たず、単なるデータの入れ物（プロパティのgetter/setterだけを持つ）になってしまっている状態**を指す、アンチパターン（悪い設計）の一つです。

その名の通り、オブジェクトが「貧血」状態で、本来持つべき「生命力（ビジネスロジック）」が欠けてしまっていることを揶揄した言葉です。

その結果、本来ドメインオブジェクトが担うべきだったビジネスロジックは、**Use CaseやApplication Serviceといった、外側のレイヤーに漏れ出してしまいます**。

### 具体例で見る「貧血」とその「治療」

まさに、私たちがクリーンアーキテクチャからDDDへリファクタリングする**前**の状態が、典型的な貧血ドメインモデルでした。

#### Before: 貧血ドメインモデル（DDD適用前）

この段階では、ビジネスロジックのほとんどが`ProcessOrderUseCase`に集中していました。

  * **`Product`クラス**: `name`や`price`といったデータを保持しているだけの、単なるデータの入れ物でした。
  * **`ProcessOrderUseCase`**: 在庫チェックや合計金額の計算といった\*\*「賢さ（ビジネスロジック）」\*\*を、すべてこのクラスが肩代わりしていました。

<!-- end list -->

```python
# --- 貧血な状態 ---

# Productはただのデータ入れ物
class Product:
    def __init__(self, name, price, stock):
        self.name = name
        self.price = price
        self.stock = stock

# UseCaseがすべてのロジックを持つ
class ProcessOrderUseCase:
    def execute(self, product, quantity):
        # 在庫チェックロジックがここにある
        if product.stock < quantity:
            raise ValueError("在庫不足")
        
        # 在庫を減らすロジックもここにある
        product.stock -= quantity
        
        # ...
```

-----

#### After: 豊かなドメインモデル (Rich Domain Model) へ（DDD適用後）

DDDのリファクタリングによって、この貧血状態を「治療」しました。

  * **`Order`集約**: 「注文明細を追加する（`add_line_item`）」際に、自分自身で在庫チェックを行い、合計金額を更新するという\*\*「賢さ（ビジネスロジック）」\*\*を持つようになりました。
  * **`ProcessOrderUseCase`**: 複雑なロジックを手放し、`Order`集約に仕事を依頼する\*\*「調整役」\*\*に専念できるようになりました。

<!-- end list -->

```python
# --- 豊かな状態 ---

# Orderは自身のルール（ロジック）を知っている
class Order:
    def __init__(self):
        self._line_items = []
        self.total_price = 0

    # 在庫チェックと合計金額更新ロジックがここにある
    def add_line_item(self, product, quantity):
        if not product.check_stock(quantity):
            raise ValueError("在庫不足")
        # ...
        self.total_price += product.price * quantity

# UseCaseは調整役に専念
class ProcessOrderUseCase:
    def execute(self, ...):
        order = Order()
        # ただ仕事を依頼するだけ
        order.add_line_item(...) 
        # ...
```

### なぜ貧血ドメインモデルは問題なのか？

1.  **オブジェクト指向の利点の喪失**: データと振る舞いを一体化させるという、OOPの最大の利点である**カプセル化**が損なわれてしまいます。ドメインオブジェクトは、手続き型言語における「構造体」と何ら変わりありません。
2.  **ビジネスロジックの重複**: 似たようなビジネスロジックが、複数の異なるUse Caseにコピペされ、重複して実装される危険性が高まります。
3.  **ドメインの知識の埋没**: ビジネスの最も重要なルールが、ドメインモデルではなく、アプリケーションのあちこちに散らばってしまうため、システムの全体像を理解するのが困難になります。

クリーンアーキテクチャはビジネスロジックを守る「金庫室」を提供しますが、DDDはさらに一歩踏み込み、その金庫室の中身であるドメインモデル自体が「貧血」に陥らないように、豊かで健康な状態に保つための具体的な設計パターンを提供するのです。
```