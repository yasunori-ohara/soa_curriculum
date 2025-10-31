# OOP-04 : CAの「型」による実装 (2) - ユースケースとインフラ層

`OOP-03`では、`OOP-01`の`Store`（神クラス）が抱えていた複数の「関心事」を、3つの独立した\*\*インターフェース（境界）\*\*に分離しました。

この章では、その「契約書」に基づいて、**具体的な実装**クラスを作成します。
`OOP-01`の`logic.py`にあったコードが、CAのレイヤー（関心事）に従って異なるファイルに配置されます。

## 🎯 この章のゴール

  * アプリケーション層（**ユースケース**）を実装し、「アプリケーション固有のルール（手順）」をカプセル化する。
  * インフラストラクチャ層（**リポジトリ**）を実装し、「データ保存という外部の詳細」をカプセル化する。
  * ユースケースが**インターフェース（境界）にのみ依存**し、インフラ層（具象）を一切知らない状態（＝**CAの依存性のルール**）をコードで確認する。

-----

## 🛠️ 1\. アプリケーション層 (Use Cases) の実装

「ユースケース」は、アプリケーションの「やりたいこと（例：注文を処理する）」を実現するための、具体的な手順を定義します。
`OOP-01`の`Store`クラスが持っていた\*\*「責任C: 注文処理のメインフロー」\*\*がここに該当します。

**重要な点**: このファイルは、`domain.py`と`application_boundaries.py`（インターフェース）に**のみ**依存します。データベースやメモリ、具体的な保存方法（`InMemory...`）については一切知りません。

> **`application.py` (新設)**

```python
import datetime
# 「境界（インターフェース）」と「ドメイン」のみをインポート
from application_boundaries import (
    IProcessOrderUseCase,
    IProductRepository,
    IOrderRepository,
    OrderData # DTOもインポート
)
from domain import IProduct # ドメイン層のインターフェース

class ProcessOrderUseCase(IProcessOrderUseCase):
    """
    「注文処理実行」ユースケースの具体的な実装。
    OOP-01の「注文処理の実行手順」という"関心事"を担当する。
    """
    def __init__(self,
                 product_repo: IProductRepository,
                 order_repo: IOrderRepository):

        # ▼▼▼ CAの依存性のルール ▼▼▼
        # このクラスは、具体的な「InMemory...」クラスを知らない。
        # 抽象的な「I...Repository」というインターフェース（境界）にのみ
        # 依存している。
        self._product_repo = product_repo
        self._order_repo = order_repo

    def execute(self, product_id: str, quantity: int, store_name: str):
        """
        注文処理を実行する。
        OOP-01の Store.process_order のロジックがここに移動した。
        """
        print(f"\n--- [{store_name}] 注文処理開始 (UseCase): 商品ID={product_id}, 数量={quantity} ---")

        # 1. 商品を探す（リポジトリ[抽象]に依頼）
        product = self._product_repo.find_by_id(product_id, store_name)
        if not product:
            print(f"エラー(UseCase): [{store_name}] 指定された商品が見つかりません。")
            return # (エラーハンドリングはPresenter等で行うのが望ましい)

        # 2. 在庫を確認する（ドメイン[抽象]のロジックを呼び出す）
        if not product.check_stock(quantity):
            print(f"エラー(UseCase): [{store_name}] {product.name}の在庫が不足しています。")
            return

        # 3. 在庫を減らす（ドメイン[抽象]のロジックを呼び出す）
        product.reduce_stock(quantity)

        # 4. 在庫の変更を保存する（リポジトリ[抽象]に依頼）
        self._product_repo.save(product, store_name)

        # 5. 注文記録データ(DTO)を作成する
        order_data = OrderData(
            product_name=product.name,
            quantity=quantity,
            total_price=product.price * quantity,
            order_date=datetime.datetime.now().isoformat()
        )

        # 6. 注文記録を保存する（リポジトリ[抽象]に依頼）
        self._order_repo.save(order_data, store_name)

        print(f"注文成功(UseCase): [{store_name}] {product.name}を{quantity}個受け付けました。")

```

-----

## 🛠️ 2\. インフラストラクチャ層 (Infrastructure) の実装

「インフラ層」は、`application_boundaries.py`で定義されたインターフェース（契約）を**具体的に実装**する層です。
`OOP-01`の`Store`クラスが持っていた\*\*「責任A: 商品データの永続化」**と**「責任C: 注文記録の永続化」\*\*がここに該当します。

> **`infrastructure.py` (新設)**

```python
import datetime
# 「境界（インターフェース）」と「ドメイン」をインポート
from application_boundaries import (
    IProductRepository,
    IOrderRepository,
    OrderData
)
from domain import IProduct # ドメイン層のインターフェース

# この例では、OOP-01と同様に「メモリ上」にデータを保存する
# 「店舗ごと」に独立したデータを持つように実装する
class InMemoryProductRepository(IProductRepository):
    """
    【具象】IProductRepositoryの「インメモリ」実装。
    OOP-01の「商品管理」という"関心事"を担当する。
    """
    def __init__(self):
        # 店舗名をキーにした辞書で、全店舗の商品データを保持
        self._products_data: dict[str, dict[str, IProduct]] = {}

    def find_by_id(self, product_id: str, store_name: str) -> IProduct | None:
        """【実装】メモリ上の辞書から商品を探す"""
        store_db = self._products_data.get(store_name)
        if store_db is None:
            return None
        return store_db.get(product_id)

    def save(self, product: IProduct, store_name: str):
        """【実装】メモリ上の辞書に商品データを保存（更新）"""
        # (saveは更新のみと仮定。addメソッドで追加する)
        store_db = self._products_data.get(store_name)
        if store_db is not None and product.product_id in store_db:
             store_db[product.product_id] = product
             print(f"商品データ保存(Infra): [{store_name}] {product.name} を更新しました。")
        else:
             print(f"エラー(Infra): [{store_name}] 商品ID {product.product_id} が見つからないか、店舗が存在しません。")


    def add(self, product: IProduct, store_name: str):
        """【実装】メモリ上の辞書に商品データを追加"""
        if store_name not in self._products_data:
            self._products_data[store_name] = {}
        
        if product.product_id in self._products_data[store_name]:
            print(f"警告(Infra): [{store_name}] 商品ID {product.product_id} は既に存在します。上書きします。")
            
        self._products_data[store_name][product.product_id] = product
        print(f"商品データ追加(Infra): [{store_name}] {product.name} を追加しました。")


class InMemoryOrderRepository(IOrderRepository):
    """
    【具象】IOrderRepositoryの「インメモリ」実装。
    OOP-01の「注文記録」という"関心事"を担当する。
    """
    def __init__(self):
        # 店舗名をキーにした辞書で、全店舗の注文記録リストを保持
        self._orders_data: dict[str, list[OrderData]] = {}

    def save(self, order_data: OrderData, store_name: str):
        """【実装】メモリ上のリストに注文記録(DTO)を追加"""
        if store_name not in self._orders_data:
            self._orders_data[store_name] = []
            
        self._orders_data[store_name].append(order_data)
        print(f"注文記録保存(Infra): [{store_name}] {order_data.product_name} の記録を追加しました。")

    def find_all_by_store(self, store_name: str) -> list[OrderData]:
        """【実装】メモリ上のリストから指定店舗の全注文記録を取得"""
        return self._orders_data.get(store_name, [])

```

-----

## 🛠️ 3\. リファクタリングの進捗

`OOP-01`の`Store`（神クラス）は、**ついに消滅しました。**

CAの「型」に従って「**関心を分離**」した結果、`OOP-01`の`logic.py`にあったロジックは、4つの異なるファイル（レイヤー）に分離されました。

  * **`domain.py`**:
      * **関心事**: ビジネスのルール（在庫管理方法など）
  * **`application_boundaries.py`**:
      * **関心事**: レイヤー間の「契約（インターフェース）」
  * **`application.py`**:
      * **関心事**: ビジネスの手順（いつ、何を呼び出すか）
  * **`infrastructure.py`**:
      * **関心事**: データの保存方法（どうやって保存するか）

`application.py`（ユースケース層）は、`infrastructure.py`（インフラ層）の存在を一切知らず、`application_boundaries.py`（境界）だけを見ている状態になりました。
**これでCAの「依存性のルール」が守られました。**

-----

## 🚧 次の課題

アーキテクチャの「内側」（ドメイン、ユースケース）と「外側」（インフラ）は完成しました。
しかし、これらはまだ「部品」としてバラバラな状態です。

  * `UseCase`が依存する`IRepository`（抽象）に、どの「実装（`InMemory...`版）」を渡すのか、誰も決めていません。
  * `UseCase`を「誰が」呼び出すのかが決まっていません。

次の章（`OOP-05`）では、`main.py`で、これらの部品をすべて「**組み立てる（＝依存性の注入）**」作業を行い、システムを完成させます。