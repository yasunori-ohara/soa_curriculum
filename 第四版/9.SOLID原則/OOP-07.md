# OOP-07 : CA/DDDの適用 (3) - インフラ層とDIの修正

`OOP-06`で、私たちは`Product`を管理する`IProductRepository`を廃止し、代わりに`Store`集約を丸ごと管理する`IStoreRepository`を導入しました。

これにより、`OOP-04`で作成した`InMemoryProductRepository`（インフラ層）と、`main.py`（DI層）は、現在の「契約（インターフェース）」と食い違っており、動作しない状態になっています。

この章では、インフラ層と`main.py`を`OOP-06`の新しい設計（集約ベース）に追従させ、システムを完成させます。

## 🎯 この章のゴール

  * `OOP-06`のドメイン変更（集約の導入）に伴い、インフラ層を`IStoreRepository`に準拠させる。
  * `main.py`（DI層）を修正し、`Store`集約をリポジトリ経由で正しく注入してアプリケーションを動作させる。
  * DDDの集約パターンにおける「リポジトリの役割」をコードで理解する。

-----

## 🎯 1\. インフラストラクチャ層 (Infrastructure) の修正

`IProductRepository`が廃止され、`IStoreRepository`が新設されたため、`infrastructure.py`を修正します。

> **`infrastructure.py` (OOP-04からの改訂)**

```python
from application_boundaries import (
    IStoreRepository, IOrderRepository # IProductRepository が IStoreRepository に
)
from domain import Product, Order, Store # Storeエンティティをインポート
from domain_vo import Money, Quantity # VOもインポート

class InMemoryStoreRepository(IStoreRepository):
    """
    【変更】IStoreRepositoryの「インメモリ」実装。
    Store集約を丸ごと辞書で管理する。
    """
    def __init__(self):
        # 店舗名をキーに、Store集約インスタンスを保持
        self._stores: dict[str, Store] = {}

    def find_by_name(self, store_name: str) -> Store | None:
        """メモリ上の辞書からStoreインスタンスを探す"""
        # (注意) メモリ上なので「実体」を返すが、本来のDB実装では
        # DBデータからStore集約を「再構築」して返す
        return self._stores.get(store_name)

    def save(self, store: Store):
        """
        Store集約の状態を丸ごと保存（上書き）する。
        集約内のProductの在庫変更も、これで永続化される。
        """
        self._stores[store.name] = store
        print(f"[{store.name}] Store集約の状態を保存しました。")
    
    # (補足) main.pyから初期データを登録(add)するため、
    # 本来のsaveは「新規作成(add)」と「更新(update)」を区別すべきだが、
    # ここではsaveが両方を兼ねる(UPSERT)とする。

class InMemoryOrderRepository(IOrderRepository):
    """
    【変更】IOrderRepositoryの「インメモリ」実装。
    (OOP-04とほぼ同じだが、OrderエンティティがVOを使うようになった)
    """
    def __init__(self):
        self._orders_data: dict[str, list[Order]] = {}

    def save(self, order: Order, store_name: str):
        if store_name not in self._orders_data:
            self._orders_data[store_name] = []
            
        self._orders_data[store_name].append(order)
        print(f"[{store_name}] {order.product_name} の注文記録を保存しました。")

    # (補足) ユースケースが履歴取得を必要とするなら、
    # IOrderRepositoryにも find_all_by_store_name などを定義する必要がある
```

-----

## 🎯 2\. 起動層 (main.py) での依存性の注入 (DI)

リポジトリの実装が変わったため、`main.py`での「組み立て」と「初期データセットアップ」の方法も変更します。

> **`main.py` (OOP-05からの改訂)**

```python
from domain import Product, Store # ドメイン層（エンティティ）
from domain_vo import Money, Quantity # ドメイン層（VO）
from application import ProcessOrderUseCase  # アプリケーション層（ユースケース）
from infrastructure import (
    InMemoryStoreRepository,  # インフラ層（リポジトリ実装）
    InMemoryOrderRepository
)
from adapters import ConsolePresenter, OrderController  # アダプター層

if __name__ == "__main__":
    
    # --- 1. 依存性の注入 (DI) ---
    #    (OOP-05からの変更点)
    
    # a. インフラ層（具象）を生成
    store_repo = InMemoryStoreRepository() # 変更
    order_repo = InMemoryOrderRepository()
    
    # b. アダプター層（プレゼンター）を生成
    presenter = ConsolePresenter()
    
    # c. アプリケーション層（ユースケース）を生成
    order_use_case = ProcessOrderUseCase(
        store_repo=store_repo, # 変更
        order_repo=order_repo,
        presenter=presenter
    )
    
    # d. アダプター層（コントローラー）を生成
    controller = OrderController(use_case=order_use_case)

    # --- 2. 初期データのセットアップ (DDD集約スタイルに変更) ---
    
    # 東京店（Store集約）の作成
    tokyo_store = Store("東京店")
    tokyo_store.add_product(Product("p-001", "高機能マウス", Money(4000), Quantity(10)))
    tokyo_store.add_product(Product("p-002", "静音キーボード", Money(6000), Quantity(5)))
    tokyo_store.add_product(Product("p-003", "24インチモニター", Money(25000), Quantity(3)))
    
    # 東京店（集約）をリポジトリに保存
    store_repo.save(tokyo_store)

    # 大阪店（Store集約）の作成
    osaka_store = Store("大阪店")
    osaka_store.add_product(Product("p-001", "高機能マウス", Money(4100), Quantity(8)))
    osaka_store.add_product(Product("p-002", "静音キーボード", Money(6000), Quantity(12)))
    osaka_store.add_product(Product("p-003", "24インチモニター", Money(25500), Quantity(5)))
    
    # 大阪店（集約）をリポジトリに保存
    store_repo.save(osaka_store)

    # --- 3. アプリケーションの実行 ---
    #    (OOP-05と「呼び出し側」は全く同じ)
    
    # シナリオ1: 東京店 注文
    print("--- シナリオ1 東京店 注文 ---")
    # 不正な値（-5）は、ユースケース内のVO変換で弾かれる
    controller.process_order("p-001", -5, "東京店") 
    controller.process_order("p-001", 3, "東京店")

    # シナリオ2: 大阪店 注文
    print("\n--- シナリオ2 大阪店 注文 ---")
    controller.process_order("p-002", 5, "大阪店")

    # シナリオ3: 東京店 在庫不足
    print("\n--- シナリオ3 東京店 在庫不足 ---")
    # 在庫5に対し10を注文 -> Store集約の reduce_stock で例外が発生
    controller.process_order("p-002", 10, "東京店")

    # シナリオ4: 大阪店 注文（成功）
    print("\n--- シナリオ4 大阪店 注文（成功） ---")
    controller.process_order("p-002", 10, "大阪店")
```

-----

## ✨ この変更の効果

`OOP-06`で設計した「**集約（`Store`）**」と「**スリムなユースケース**」のアーキテクチャが、`main.py`まで一貫して適用され、動作するようになりました。

  * **SRP/DIPの徹底**: CAとDDDのパターンを組み合わせることで、SOLID原則がアーキテクチャ全体で遵守されました。
  * **ロジックの配置**:
      * **`domain_vo.py`**: 値の不変条件（例: `Quantity > 0`）
      * **`domain.py`**: ビジネスルール（例: 在庫チェック、注文生成）
      * **`application.py`**: アプリケーション手順（例: リポジトリの呼び出し順序）
      * **`infrastructure.py`**: 保存方法（例: メモリ辞書への保存）
      * **`main.py`**: 組み立て（DI）
  * **バグの撲滅**: `quantity = -5`のような不正なデータは、ドメイン層に到達する**前**（ユースケース層の`Quantity(request.quantity)`の変換時点）で弾かれるようになり、堅牢性が飛躍的に向上しました。

-----

## 🚧 次の課題: OCP（オープン・クローズドの原則）違反

`OOP-02`で指摘された、残る最後の課題です。

「**もし『ダウンロード商品』を追加したくなったら？**」

現在の`domain.py`の`Product`エンティティは、「物理在庫（`stock: Quantity`）」を持つこと（`__init__`）を前提としています。
`check_stock`や`reduce_stock`のロジックも、物理在庫を前提に書かれています。

このままでは、「在庫の概念がない」ダウンロード商品を追加する際に、`Product`クラスや`Store`集約の`process_order`メソッドに`if`文による**修正**が必要になり、OCP違反が発生します。

次の章では、このOCP/LSP違反を、CA/DDDのアーキテクチャの中で解決します。