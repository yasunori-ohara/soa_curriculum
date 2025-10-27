# DDD-06

```markdown
## `use_cases/process_order.py` (ユースケース)

ドメインモデルが豊か（リッチ）になったことで、このユースケースの役割がどのように変化したかに注目ください。

---
### 1. このファイルの役割：集約を指揮する「コーディネーター」

このファイルは、引き続きUse Cases（ユースケース）レイヤーに属します。しかし、DDDを適用したことで、その役割は大きく変化しました。

以前は、在庫チェックや合計金額の計算といったビジネスロジックを自身で実行する「働き者のマネージャー」でした。
しかし今、そのロジックの多くは`Order`集約に移譲されました。その結果、このユースケースは、必要な集約（Aggregate）を見つけ出し、仕事（メソッド呼び出し）を依頼し、結果をリポジトリに保存するという、より上位の**「コーディネーター」または「指揮者」**のような役割に変わりました。

### 2. ソースコードの詳細解説

`ProcessOrderInput` と `ProcessOrderOutput` (DTOs)
これらは**DTO (Data Transfer Object)**と呼ばれる、単純なデータの入れ物です。ユースケースの入り口と出口のデータ形式を明確に定義することで、`dict`（辞書）よりも意図が分かりやすく、型安全なコードになります。

`ProcessOrderUseCase` クラス
- `__init__(...)`: 依存性の注入（DI）が更新されています。このユースケースは、`Product`を探すための`ProductRepository`と、`Order`を保存するための`OrderRepository`という、**2つの異なるリポジトリ（インターフェース）**に依存するようになりました。

- `execute(...)`: このメソッドの処理の流れが、DDDによる役割の変化を最もよく表しています。

  1. 準備: `ProductRepository`を使い、注文に必要な`Product`エンティティを事前にすべて取得します。これは、料理人が調理を始める前に、すべての材料を揃えるのと同じです。

  2. 生成: `Order`集約のインスタンスを生成します。この時点では、まだ空の注文伝票です。

  3. 委譲: 注文明細を追加する処理は、`order.add_line_item(...)`を呼び出すことで、`Order`集約自身に完全に委譲します。ユースケースは、`add_line_item`の内部で在庫チェックが行われていることなどを知る必要はありません。

  4. 委譲: 注文を確定する処理も、`order.confirm()`を呼び出して`Order`集約に委譲します。

  5. 永続化: 完成し、整合性が保たれた`Order`集約を、`OrderRepository`に渡して保存を依頼します。

### 3. このレイヤーの鉄則（DDD適用後）

1. 内側にのみ依存: これは変わりません。

2. 集約を調整する: ユースケースは、ビジネスルールを自分で実装するのではなく、ドメインモデル（エンティティや集約）を調整し、指揮することに専念します。

3. トランザクションの境界: 1つのユースケースの実行は、通常、データベースにおける1つのトランザクションに対応します。executeメソッドが成功すればコミット、失敗すればロールバック、という流れを保証する責任を持ちます。

---
### `use_cases/process_order.py`

``` Python
# 依存性のルール:
# このファイルはUse Casesレイヤーに属します。
# 自分より内側のdomainレイヤーと、同じレイヤーのinterfacesにのみ依存します。

import uuid
from dataclasses import dataclass
from domain.product import Product
from domain.order import Order
from .interfaces import ProductRepository, OrderRepository

# --- DTO (Data Transfer Object) の定義 ---
# ユースケースの入出力を明確にするためのデータ構造

@dataclass
class ProcessOrderInput:
    """ユースケースへの入力データを格納するクラス"""
    items: list[dict] # 例: [{"product_id": "p-001", "quantity": 2}, ...]

@dataclass
class ProcessOrderOutput:
    """ユースケースからの出力データを格納するクラス"""
    order_id: str
    total_price: int

class ProcessOrderUseCase:
    """
    【Use Casesレイヤー / Use Case】
    「注文を処理する」というアプリケーションのユースケース。
    
    DDDを適用したことで、このクラスの役割は大きく変化した。
    以前は自身が持っていたビジネスロジックの多くを、Order集約に移譲し、
    自身は集約を見つけて実行し、結果を保存するという「調整役」に専念する。
    """
    def __init__(self, product_repo: ProductRepository, order_repo: OrderRepository):
        self.product_repo = product_repo
        self.order_repo = order_repo

    def execute(self, order_input: ProcessOrderInput) -> ProcessOrderOutput:
        # 1. 準備：注文に必要なProductエンティティをすべて取得する
        products: dict[str, Product] = {}
        for item in order_input.items:
            product = self.product_repo.find_by_id(item["product_id"])
            if not product:
                raise ValueError(f"商品IDが見つかりません: {item['product_id']}")
            products[item["product_id"]] = product
        
        # 2. 生成：新しいOrder集約のインスタンスを生成する
        new_order_id = "order-" + str(uuid.uuid4())
        order = Order(order_id=new_order_id)

        # 3. 委譲：ビジネスロジックの実行をOrder集約に委譲する
        for item in order_input.items:
            product = products[item["product_id"]]
            order.add_line_item(product, item["quantity"])
        
        # 4. 委譲：注文の確定処理もOrder集約に委譲する
        order.confirm()
        
        # 5. 永続化：完成したOrder集約をリポジトリに保存する
        self.order_repo.save(order)

        # 6. 結果を返す
        return ProcessOrderOutput(
            order_id=order.order_id,
            total_price=order.total_price
        )
```

```