# OOP-08: DDD適用 (2) - 境界、ユースケース、インフラ、DIの追従

`OOP-07`では、ドメイン層（`domain.py`）に\*\*値オブジェクト（VO）**と**集約（`Store`）\*\*を導入し、ビジネスルールとロジックを強化しました。

この変更はドミノのように波及し、ドメイン層とやり取りする他のレイヤー（境界、ユースケース、インフラ、DI）にも修正が必要になります。

この章では、`OOP-07`の変更に合わせて、これらの外部レイヤーを修正し、システム全体を再び動作する状態に戻します。

## 🎯 この章のゴール

  * `OOP-07`のドメイン変更（VO、集約）に伴い、境界（`application_boundaries.py`）を修正する。
  * ビジネスロジックが集約に移動したことでスリム化されたユースケース（`application.py`）を実装する。
  * インフラ層（`infrastructure.py`）を`IStoreRepository`に準拠させる。
  * `main.py`（DI層）を修正し、集約をリポジトリ経由で正しく注入してアプリケーションを動作させる。

-----

## 🛠️ 1\. 境界 (Application Boundaries) の修正

`domain.py`の変更に合わせて、インターフェースとDTO（Data Transfer Object）を修正します。

> **`application_boundaries.py` (OOP-03からの改訂)**

```python
from abc import ABC, abstractmethod
# domain から Store 集約と IProduct (子要素アクセス用) をインポート
from domain import IProduct, Store, OrderData
# domain_vo から Quantity VO をインポート
from domain_vo import Quantity

# --- DTO (Data Transfer Objects) ---
# (OrderDataはdomain.pyに移動したので削除)

# --- ユースケース層の「境界」 ---
class IProcessOrderUseCase(ABC):
    @abstractmethod
    def execute(self, product_id: str, quantity: int, store_name: str): # 入力はまだint
        pass

# --- インフラ層の「境界」 ---
class IStoreRepository(ABC):
    """
    【変更】Store集約を永続化するリポジトリ。
    IProductRepository は不要になった。
    """
    @abstractmethod
    def find_by_name(self, store_name: str) -> Store | None:
        """【契約】名前でStore集約を取得する"""
        pass

    @abstractmethod
    def save(self, store: Store):
        """【契約】Store集約の状態を保存（追加・更新）する"""
        pass

class IOrderRepository(ABC):
    """(変更) 保存するデータ型をOrderDataに変更"""
    @abstractmethod
    def save(self, order_data: OrderData, store_name: str): # OrderData DTOを保存
        """【契約】注文記録(DTO)を保存する"""
        pass

    @abstractmethod
    def find_all_by_store(self, store_name: str) -> list[OrderData]: # OrderData DTOを返す
        """【契約】指定された店舗の全注文記録(DTO)を取得する"""
        pass

# (IProductRepositoryは削除)
```

-----

## 🛠️ 2\. アプリケーション層 (Use Cases) の修正

ドメインロジックが集約に移動したため、ユースケースは劇的にスリムになります。「オーケストレーター」としての役割に集中します。

> **`application.py` (OOP-04からの改訂)**

```python
import datetime
from application_boundaries import (
    IProcessOrderUseCase,
    IStoreRepository, # IProductRepository -> IStoreRepository
    IOrderRepository,
    # OrderData は domain に移動
)
from domain import Store, OrderData # StoreとOrderDataをdomainからインポート
from domain_vo import Quantity # VOをインポート

class ProcessOrderUseCase(IProcessOrderUseCase):
    """
    【変更】ユースケースが劇的にスリムになった。
    ドメインの調整役ではなく、薄い「オーケストレーター」になった。
    """
    def __init__(self,
                 store_repo: IStoreRepository, # 変更
                 order_repo: IOrderRepository):
        self._store_repo = store_repo
        self._order_repo = order_repo
        # (Presenterは省略)

    def execute(self, product_id: str, quantity: int, store_name: str):
        print(f"\n--- [{store_name}] 注文処理開始 (UseCase): 商品ID={product_id}, 数量={quantity} ---")
        try:
            # 1. 入力値(int)を「VO」に変換 (不正な値はここでエラー)
            quantity_vo = Quantity(quantity)

            # 2. Store集約をリポジトリ[抽象]から「読み出す」
            store = self._store_repo.find_by_name(store_name)
            if not store:
                raise ValueError(f"店舗 '{store_name}' が見つかりません。")

            # 3. ドメイン(集約)に処理を「命じる」(Tell, Don't Ask)
            #    ロジックの詳細はユースケースは「知らない」
            order_data: OrderData = store.process_order(product_id, quantity_vo)

            # 4. 変更されたStore集約をリポジトリ[抽象]に「保存する」
            self._store_repo.save(store)

            # 5. 生成された注文データ(OrderData)をリポジトリ[抽象]に「保存する」
            self._order_repo.save(order_data, store_name)

            print(f"注文成功(UseCase): [{store_name}] {order_data.product_name} を {order_data.quantity.value} 個受け付けました。")
            # (成功時のPresenter呼び出しなど)

        except (ValueError, TypeError) as e:
            # (失敗時のPresenter呼び出しなど)
            print(f"エラー(UseCase): [{store_name}] {e}")

```

-----

## 🛠️ 3\. インフラストラクチャ層 (Infrastructure) の修正

`IProductRepository`が廃止され、`IStoreRepository`が新設されたため、`infrastructure.py`を修正します。

> **`infrastructure.py` (OOP-04からの改訂)**

```python
import datetime
from application_boundaries import (
    IStoreRepository, # IProductRepository -> IStoreRepository
    IOrderRepository,
    # OrderData は domain に移動
)
# domain から Store集約, IProduct, OrderData をインポート
from domain import IProduct, Store, OrderData
# domain_vo から VO をインポート (初期データ生成用)
from domain_vo import Money, Quantity

class InMemoryStoreRepository(IStoreRepository):
    """
    【変更】IStoreRepositoryの「インメモリ」実装。
    Store集約を丸ごと辞書で管理する。
    """
    def __init__(self):
        # 店舗名をキーに、Store集約インスタンスを保持
        self._stores: dict[str, Store] = {}

    def find_by_name(self, store_name: str) -> Store | None:
        """【実装】メモリ上の辞書からStoreインスタンスを探す"""
        # メモリなので「実体」を返す (DBなら再構築)
        found = self._stores.get(store_name)
        if found:
            print(f"店舗データ取得(Infra): '{store_name}' のStoreオブジェクトを返します。")
        else:
            print(f"店舗データ取得(Infra): '{store_name}' は見つかりませんでした。")
        return found

    def save(self, store: Store):
        """
        【実装】Store集約の状態を丸ごと保存（上書き）する。
        集約内のProductの在庫変更も、これで永続化される。
        """
        self._stores[store.name] = store
        print(f"店舗データ保存(Infra): [{store.name}] Store集約の状態を保存しました。")

class InMemoryOrderRepository(IOrderRepository):
    """
    【変更】IOrderRepositoryの「インメモリ」実装。
    OrderData(DTO)を保存・取得するように変更。
    """
    def __init__(self):
        self._orders_data: dict[str, list[OrderData]] = {}

    def save(self, order_data: OrderData, store_name: str):
        """【実装】メモリ上のリストに注文記録(DTO)を追加"""
        if store_name not in self._orders_data:
            self._orders_data[store_name] = []
        self._orders_data[store_name].append(order_data)
        print(f"注文記録保存(Infra): [{store_name}] {order_data.product_name} の記録を追加しました。")

    def find_all_by_store(self, store_name: str) -> list[OrderData]:
        """【実装】メモリ上のリストから指定店舗の全注文記録(DTO)を取得"""
        return self._orders_data.get(store_name, [])

```

-----

## 🛠️ 4\. 起動層 (main.py) での依存性の注入 (DI)

リポジトリのインターフェースと実装が変わり、ドメインの初期化方法も変わったため、`main.py`を修正します。

> **`main.py` (OOP-05からの改訂)**

```python
# --- インポート ---
# ドメイン層: Store集約, 具体的なProduct, VO
from domain import Store, PhysicalProduct, DigitalProduct, IProduct
from domain_vo import Money, Quantity

# アプリケーション層: 具体的なUseCase
from application import ProcessOrderUseCase

# インフラ層: 具体的なRepository
from infrastructure import InMemoryStoreRepository, InMemoryOrderRepository

# (adapters.py は今回未使用)

if __name__ == "__main__":

    # --- 1. 依存性の注入 (DI) ---
    # a. インフラ層（具象）を生成
    store_repo = InMemoryStoreRepository() # IProductRepository -> IStoreRepository
    order_repo = InMemoryOrderRepository()

    # b. アプリケーション層（ユースケース）を生成し、具象リポジトリを注入
    order_use_case = ProcessOrderUseCase(
        store_repo=store_repo,
        order_repo=order_repo
    )

    # --- 2. 初期データのセットアップ (集約スタイルに変更) ---
    # a. 東京店（Store集約）の作成と初期化
    tokyo_store = Store("東京店")
    tokyo_store.add_product(
        PhysicalProduct("p-001", "高機能マウス", Money(4000), Quantity(10))
    )
    tokyo_store.add_product(
        PhysicalProduct("p-002", "静音キーボード", Money(6000), Quantity(5))
    )
    tokyo_store.add_product(
        DigitalProduct("d-001", "デザインソフト eBook", Money(8000))
    )
    # b. 作成した集約をリポジトリに保存
    store_repo.save(tokyo_store)

    # (大阪店のセットアップは省略)

    # --- 3. アプリケーションの実行 ---
    # UseCaseのexecuteメソッドを呼び出す (OOP-05と同じ)

    # シナリオ1: 物理商品 (成功)
    order_use_case.execute("p-001", 3, "東京店")

    # シナリオ2: 物理商品 (在庫不足 -> 例外発生)
    order_use_case.execute("p-002", 10, "東京店")

    # シナリオ3: デジタル商品 (成功)
    order_use_case.execute("d-001", 10, "東京店")

    # シナリオ4: 不正な数量 (VOで例外発生)
    order_use_case.execute("p-001", -5, "東京店")


    print("\n--- 東京店 注文履歴 ---")
    # (UseCase に履歴取得機能がないため、リポジトリを直接参照)
    history = order_repo.find_all_by_store("東京店")
    for record in history:
        print(f"  - {record.product_name}, Qty: {record.quantity.value}, Total: {record.total_price.value}")

```

-----

## ✨ この変更の効果

`OOP-07`で設計した「**値オブジェクト**」「**集約**」「**スリムなユースケース**」のアーキテクチャが、`main.py`まで一貫して適用され、動作するようになりました。

  * **ドメイン中心**: ビジネスロジックがドメイン層（`Store`集約）に集約され、アプリケーション層（`UseCase`）は薄い調整役に徹しています。
  * **堅牢性向上**: 値オブジェクト（`Quantity`など）が不正なデータをドメイン層の手前（`UseCase`）で弾くため、バグが入り込みにくくなりました。
  * **責務の明確化**: 各レイヤー（ファイル）の責任がより明確になりました。

-----

これで、CA+DDDによるリファクタリングは一旦完了です。
SOLID原則は`OOP-06`の時点で達成されていましたが、DDDのパターンを適用することで、コードの「質」（堅牢性、ドメイン知識の表現力）がさらに向上しました。