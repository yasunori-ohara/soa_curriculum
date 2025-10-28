# OOP-04 : CAによる実装 - ユースケースとインフラ層の分離

`OOP-03`では、ドメイン層（`domain.py`）とアプリケーションの境界（`application_boundaries.py`）を定義しました。

この章では、`OOP-01`の`Store`クラスが持っていた「**ユースケース（注文手順）**」と「**インフラ（データ保存）**」の責務を、それぞれ独立したクラスとして具体的に実装します。

## 🎯 この章のゴール

  * アプリケーション層（**ユースケース**）を実装し、「アプリケーション固有のルール」をカプセル化する。
  * インフラストラクチャ層（**リポジトリ**）を実装し、「データ永続化」のロジックをカプセル化する。
  * ユースケースが**インターフェース（抽象）にのみ依存**し、インフラ（具象）を一切知らない状態（＝**DIPの達成**）をコードで確認する。

-----

## 🎯 1\. アプリケーション層 (Use Cases) の実装

「ユースケース」は、アプリケーションの「やりたいこと（例：注文を処理する）」を実現するための、具体的な手順を定義します。

**重要な点**: このファイルは、`domain.py`と`application_boundaries.py`（インターフェース）に**のみ**依存します。DBやWeb、具体的な保存方法（`InMemory...`）については一切知りません。

> **`application.py` (新設)**

```python
from application_boundaries import (
    IProcessOrderUseCase, IProcessOrderPresenter,
    IProductRepository, IOrderRepository,
    ProcessOrderRequest, ProcessOrderResponse
)
from domain import Product, Order

class ProcessOrderUseCase(IProcessOrderUseCase):
    """
    「注文処理」ユースケースの具体的な実装（Interactor）。
    OOP-01のStoreが持っていた「責任1: フロー実行」と「責任2: 記録作成」を担当。
    """
    def __init__(self,
                 product_repo: IProductRepository,
                 order_repo: IOrderRepository,
                 presenter: IProcessOrderPresenter):
        
        # 【DIP】具象クラスではなく「インターフェース」に依存する
        self._product_repo = product_repo
        self._order_repo = order_repo
        self._presenter = presenter

    def handle(self, request: ProcessOrderRequest):
        """
        ユースケースの実行。
        OOP-01の process_order のロジックがここに移動した。
        """
        try:
            # 1. 商品を探す（リポジトリ[抽象]に依頼）
            product = self._product_repo.find_by_id(
                request.product_id, request.store_name
            )
            if not product:
                raise ValueError("指定された商品が見つかりません。")

            # 2. 在庫を確認する（ドメイン[エンティティ]のルール）
            if not product.check_stock(request.quantity):
                raise ValueError(f"{product.name}の在庫が不足しています。")

            # 3. 在庫を減らす（ドメイン[エンティティ]のルール）
            product.reduce_stock(request.quantity)

            # 4. 在庫の変更を保存する（リポジトリ[抽象]に依頼）
            self._product_repo.save(product, request.store_name)

            # 5. 注文記録（エンティティ）を作成する
            total_price = product.price * request.quantity
            order = Order(
                product_name=product.name,
                quantity=request.quantity,
                total_price=total_price
            )

            # 6. 注文記録を保存する（リポジトリ[抽象]に依頼）
            self._order_repo.save(order, request.store_name)

            # 7. 成功をPresenter（抽象）経由で通知する
            response = ProcessOrderResponse(
                order=order,
                updated_stock=product.get_stock()
            )
            self._presenter.present_success(response)

        except (ValueError, TypeError) as e:
            # 8. 失敗をPresenter（抽象）経由で通知する
            self._presenter.present_failure(str(e))
```

-----

## 🎯 2\. インフラストラクチャ層 (Infrastructure) の実装

「インフラ層」は、インターフェース（契約）を**具体的に実装**する層です。
`OOP-01`の`Store`が持っていた「責任3: データ保存」ロジック（`_products`辞書や`_orders`リスト）がここに移動します。

> **`infrastructure.py` (新設)**

```python
from application_boundaries import IProductRepository, IOrderRepository
from domain import Product, Order

# この例では、OOP-01と同様に「メモリ上」にデータを保存する
# 「店舗ごと」に独立したデータを持つように実装する
class InMemoryProductRepository(IProductRepository):
    """
    IProductRepositoryの「インメモリ」実装。
    OOP-01のStoreが持っていた _products 辞書を管理する。
    """
    def __init__(self):
        # 店舗名をキーにした辞書で、全店舗の商品データを保持
        self._products_data: dict[str, dict[str, Product]] = {
            "東京店": {},
            "大阪店": {}
        }

    def find_by_id(self, product_id: str, store_name: str) -> Product | None:
        store_db = self._products_data.get(store_name)
        if store_db is None:
            return None
        return store_db.get(product_id)

    def save(self, product: Product, store_name: str):
        # 本来は在庫更新(UPDATE)だが、メモリなので
        # product_idをキーにまるごと上書きする
        store_db = self._products_data.get(store_name)
        if store_db is not None:
            store_db[product.product_id] = product
            print(f"[{store_name}] {product.name} のデータ(在庫)を保存しました。")
        else:
            print(f"エラー: {store_name} というDBコンテキストは存在しません。")

    # (補足) 本来は main.py からデータを登録(add)する口が必要だが、
    # 簡略化のため、ここに初期データを追加する
    def _add_initial_data(self, product: Product, store_name: str):
        self._products_data[store_name][product.product_id] = product

class InMemoryOrderRepository(IOrderRepository):
    """
    IOrderRepositoryの「インメモリ」実装。
    OOP-01のStoreが持っていた _orders リストを管理する。
    """
    def __init__(self):
        self._orders_data: dict[str, list[Order]] = {
            "東京店": [],
            "大阪店": []
        }

    def save(self, order: Order, store_name: str):
        store_db = self._orders_data.get(store_name)
        if store_db is not None:
            store_db.append(order)
            print(f"[{store_name}] {order.product_name} の注文記録を保存しました。")
        else:
            print(f"エラー: {store_name} というDBコンテキストは存在しません。")
```

-----

## ✨ この変更が解決したこと（設計上の進捗）

1.  **SRP（単一責任の原則）の達成**:
    `OOP-01`の`Store`クラスは完全に消滅しました。

      * **注文フロー（ユースケース）**: `ProcessOrderUseCase`が担当
      * **商品保存（インフラ）**: `InMemoryProductRepository`が担当
      * **注文保存（インフラ）**: `InMemoryOrderRepository`が担当
      * **商品ルール（ドメイン）**: `Product`エンティティが担当
      * **注文データ（ドメイン）**: `Order`エンティティが担当
        すべてのクラスが「単一の責任」を持つようになりました。

2.  **DIP（依存性逆転の原則）の達成**:
    **`ProcessOrderUseCase`（ユースケース層）のコードを見てください。**
    `InMemoryProductRepository`や`InMemoryOrderRepository`という\*\*具象クラス（インフラ層）**の名前は、`application.py`のどこにも登場しません。
    `UseCase`は`IProductRepository`や`IOrderRepository`という**インターフェース（境界）\*\*にのみ依存しています。

    依存関係が「`infrastructure` → `application_boundaries` ← `application`」となり、CAの「依存性のルール」が完全に守られました。

-----

## 🚧 次の課題

アーキテクチャの「内側」（ドメイン、ユースケース）と「外側」（インフラ）は完成しました。
しかし、これらを\*\*「繋ぎ合わせる」**ためのアダプター層と、**「起動する」\*\*ための`main.py`がまだありません。

次の章では、`main.py`からの入力を`UseCase`に渡す**コントローラー**と、`UseCase`からの出力を画面に表示する**プレゼンター**を実装し、すべてを組み立て（依存性の注入）てシステムを完成させます。