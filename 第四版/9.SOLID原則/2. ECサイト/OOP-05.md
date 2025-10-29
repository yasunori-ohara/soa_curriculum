
# OOP-05 : CAによる組立 - アダプター層と依存性の注入(DI)

`OOP-04`までに、CAの主要なレイヤー（ドメイン、ユースケース、インフラ）を実装しました。しかし、これらはまだ「部品」としてバラバラな状態です。

  * `UseCase`は`IPresenter`（抽象）に結果を渡すが、その「実装（画面表示役）」がいない。
  * `UseCase`を「誰が」呼び出すのか（入力の受付役）がいない。
  * `UseCase`が依存する`IRepository`（抽象）に、どの「実装（インメモリ版）」を渡すのか、誰も決めていない。

この章では、これらの部品を組み立て、システム全体を完成させます。

## 🎯 この章のゴール

  * **アダプター層 (Interface Adapters)** の役割（＝データの変換）を理解する。
  * **プレゼンター**（出力アダプター）と**コントローラー**（入力アダプター）を実装する。
  * `main.py`（起動層）で、**依存性の注入 (DI)** を行い、システム全体を組み立てる。
  * CAアーキテクチャで、`OOP-01`のシナリオが実行できることを確認する。

-----

## 🎯 1\. アダプター層 (Interface Adapters) の実装

「アダプター層」は、CAの内側（ユースケース）と外側（インフラ、Web、コンソール）の間で、**データ形式を変換する**責務を持ちます。

> **`adapters.py` (新設)**

```python
from application_boundaries import (
    IProcessOrderUseCase, IProcessOrderPresenter,
    ProcessOrderRequest, ProcessOrderResponse
)

class ConsolePresenter(IProcessOrderPresenter):
    """
    【出力アダプター】プレゼンターの「コンソール」実装。
    ユースケース層から渡された「レスポンス(DTO)」を、
    人間が読める「コンソール出力(文字列)」に変換する。
    """
    def present_success(self, response: ProcessOrderResponse):
        order = response.order
        print("\n--- 注文成功 ---")
        print(f"  商品名: {order.product_name}")
        print(f"  数量: {order.quantity}")
        print(f"  合計金額: {order.total_price}")
        print(f"  注文日時: {order.order_date}")
        print(f"  （処理後の在庫: {response.updated_stock}）")

    def present_failure(self, error: str):
        print(f"\n--- 注文失敗 ---")
        print(f"  エラー: {error}")

class OrderController:
    """
    【入力アダプター】コントローラー。
    外部（今回は main.py）からの「単純な入力(str, int)」を、
    ユースケース層が理解できる「リクエスト(DTO)」に変換して呼び出す。
    """
    def __init__(self, use_case: IProcessOrderUseCase):
        # 【DIP】具象(ProcessOrderUseCase)ではなく、抽象に依存
        self._use_case = use_case

    def process_order(self, product_id: str, quantity: int, store_name: str):
        """
        外部からの入力を受け付け、ユースケースを起動する
        """
        # 1. 外部からの入力を「リクエストDTO」に変換
        request = ProcessOrderRequest(
            product_id=product_id,
            quantity=quantity,
            store_name=store_name
        )
        # 2. ユースケース（抽象）のハンドルを呼び出す
        self._use_case.handle(request)
```

-----

## 🎯 2\. 起動層 (main.py) での依存性の注入 (DI)

`main.py`は、CAの最も外側の層（あるいは層の外）です。
**唯一の責務は、すべての「具象クラス」をインスタンス化し、依存関係に従ってそれらを「組み立てる（注入する）」ことです。**

（※`main.py`から初期データを登録できるよう、`OOP-04`の`infrastructure.py`に`add_product`メソッドを追加したと仮定します）

> **`infrastructure.py` (OOP-04からの修正・追加)**

```python
from domain import Product
# ... (InMemoryProductRepository 内) ...
class InMemoryProductRepository:
    # ... (init, find_by_id, save は同じ) ...
    
    def add_product(self, product: Product, store_name: str):
        """【新設】main.pyから初期データを登録するためのメソッド"""
        store_db = self._products_data.get(store_name)
        if store_db is not None:
            store_db[product.product_id] = product
        else:
            # (簡易的に店舗コンテキストも自動作成)
            self._products_data[store_name] = {product.product_id: product}
            self._orders_data[store_name] = [] # OrderRepo側も初期化
```

> **`main.py` (OOP-01からの全面改訂)**

```python
from domain import Product  # ドメイン層（エンティティ）
from application import ProcessOrderUseCase  # アプリケーション層（ユースケース）
from infrastructure import (
    InMemoryProductRepository,  # インフラ層（リポジトリ実装）
    InMemoryOrderRepository
)
from adapters import ConsolePresenter, OrderController  # アダプター層

# (注意)
# main.py は「起動層」であるため、唯一
# *すべて*の具象クラスを知っている（インポートしている）ファイルとなる。

if __name__ == "__main__":
    
    # --- 1. 依存性の注入 (DI) ---
    # CAの「外側」から「内側」へと依存関係を組み立てていく
    
    # a. インフラ層（最も外側）の具象インスタンスを生成
    product_repo = InMemoryProductRepository()
    order_repo = InMemoryOrderRepository()
    
    # b. アダプター層（プレゼンター）の具象インスタンスを生成
    presenter = ConsolePresenter()
    
    # c. アプリケーション層（ユースケース）を生成
    #    -> インターフェース(I...Repository, I...Presenter)越しに
    #       具象インスタンスを「注入」する
    order_use_case = ProcessOrderUseCase(
        product_repo=product_repo,
        order_repo=order_repo,
        presenter=presenter
    )
    
    # d. アダプター層（コントローラー）を生成
    #    -> インターフェース(I...UseCase)越しに
    #       具象インスタンスを「注入」する
    controller = OrderController(use_case=order_use_case)

    # --- 2. 初期データのセットアップ ---
    # (本来はDBマイグレーション等が担うが、今回はDI層で行う)
    # (※インフラ層に add_product が必要)
    product_repo.add_product(Product("p-001", "高機能マウス", 4000, 10), "東京店")
    product_repo.add_product(Product("p-002", "静音キーボード", 6000, 5), "東京店")
    product_repo.add_product(Product("p-003", "24インチモニター", 25000, 3), "東京店")
    
    product_repo.add_product(Product("p-001", "高機能マウス", 4100, 8), "大阪店")
    product_repo.add_product(Product("p-002", "静音キーボード", 6000, 12), "大阪店")
    product_repo.add_product(Product("p-003", "24インチモニター", 25500, 5), "大阪店")

    # --- 3. アプリケーションの実行 ---
    # OOP-01 と同じシナリオを「コントローラー経由」で実行する
    
    # (在庫表示ロジックは今回省略)
    
    # シナリオ1: 東京店インスタンスに「注文を処理して」と依頼
    print("--- シナリオ1 東京店 注文 ---")
    controller.process_order("p-001", 3, "東京店")

    # シナリオ2: 大阪店インスタンスに「注文を処理して」と依頼
    print("\n--- シナリオ2 大阪店 注文 ---")
    controller.process_order("p-002", 5, "大阪店")

    # シナリオ3: 東京店インスタンスで在庫不足
    print("\n--- シナリオ3 東京店 在庫不足 ---")
    controller.process_order("p-002", 10, "東京店")

    # シナリオ4: 大阪店インスタンスでは同じ注文が成功
    print("\n--- シナリオ4 大阪店 注文（成功） ---")
    controller.process_order("p-002", 10, "大阪店") # OOP-01では失敗例だったが、在庫12なので成功
```

-----

## ✨ CAリファクタリングによるSOLID原則の達成

`OOP-01`のコードをCAのレイヤーに分離・再配置しただけで、`OOP-02`で指摘された主要なSOLID原則違反が解消されました。

1.  **S (単一責任) の達成**:
    `Store`という神クラス（God Class）は消滅しました。

      * **ドメインルール**: `Product`, `Order` (エンティティ)
      * **アプリ手順**: `ProcessOrderUseCase` (ユースケース)
      * **データ保存**: `InMemory...Repository` (インフラ)
      * **入出力変換**: `OrderController`, `ConsolePresenter` (アダプター)
        すべてのクラスが「変更される理由」を一つだけ持つようになりました。

2.  **D (依存性逆転) の達成**:
    CAの「依存性のルール」を強制した結果、DIPが完全に達成されました。

      * `ProcessOrderUseCase`（アプリ層）は、`InMemory...Repository`（インフラ層）の**具象を知りません**。`IRepository`という\*\*抽象（インターフェース）\*\*にのみ依存します。
      * `main.py`が、抽象に具象を「注入（DI）」することで、依存関係の制御が一箇所に集約されました。

3.  **O (オープン・クローズド) の達成**:
    このアーキテクチャは「変更に強い」ものになりました。

      * **もし「ダウンロード商品」を追加したくなったら？**
        `domain.py`に`DigitalProduct`を追加し、`application_boundaries.py`の`IProduct`インターフェースを実装させます。`OOP-01`のように`if`文で`Product`クラスを**修正**する必要はありません（※）。`UseCase`は`IProduct`（抽象）にしか依存していないため、何も修正は不要です。
      * **もし「保存先をDB」に変更したくなったら？**
        `infrastructure.py`に`DatabaseProductRepository`クラスを新設し、`main.py`で注入する具象クラスを`InMemory...`から`Database...`に差し替えるだけです。`UseCase`や`Domain`のコードには一切触る必要がありません。

（※ただし、`OOP-04`の`Product`クラス（物理商品）は、OCP/LSPの観点ではまだ課題を抱えています。これが次のステップです。）
