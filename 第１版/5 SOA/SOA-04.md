# SOA-04 : 🟡 `interface_adapters/inventory_service_adapter.py` (アダプター)

`SOA-02` で外部サービス（在庫管理）を定義し、`SOA-03` で私たちの `Use Case` が要求する「理想の鍵穴（`IInventoryServicePort`）」を定義しました。

このステップでは、その「鍵穴」にぴったり合う「**具体的な鍵（アダプター）**」を `Interface Adapters` レイヤーに実装します。このアダプターが、私たちのアプリケーションと外部サービスをつなぐ「**変換プラグ**」の役割を果たします。

## 🎯 この章のゴール

  * `Interface Adapters` レイヤーが、内部インターフェースと外部サービスを接続する「アダプター」であることを理解する。
  * `use_cases` で定義した「抽象（ポート）」を、「具象（アダプタークラス）」として実装する。
  * アダプターが、外部サービスのインスタンスを**依存性の注入 (DI)** で受け取ることを学ぶ。
  * アダプターの主な役割が、要求を外部サービスに「**委譲 (Delegate)**」することであることを理解する。

-----

## 🔌 このファイルの役割：外部サービスへの「変換プラグ」

このファイルは `Interface Adapters` レイヤーに属します。その名の通り、まさに「アダプター」としての役割を担います。

`Use Cases` レイヤーが定義した `IInventoryServicePort` という我々のアプリケーション内部の「コンセントの形（鍵穴）」と、`external_services` が提供する `InventoryService` という外部の「特殊な形状のプラグ（API）」を、うまく適合させるための**変換プラグ**だと考えてください 🔌。

このアダプターのおかげで、`Use Case` は接続先が `SOA-02` の模擬サービスであろうと、実際の `REST API` であろうと、常に標準的なコンセント（`IInventoryServicePort`）に接続するだけで済むのです。

-----

## 💻 ソースコードの詳細解説

### `InventoryServiceAdapter(IInventoryServicePort)`: 契約の履行

このクラスは、`SOA-03` の `use_cases/interfaces.py` で定義された `IInventoryServicePort` という契約書に「署名」しています (`implements` の意味)。これにより、このクラスが `check_stock` と `reduce_stock` というメソッドを必ず実装することが保証されます。

### `__init__(self, inventory_service: InventoryService)`: 外部サービスの注入

このアダプターは、自身で外部サービス（`InventoryService`）のインスタンスを生成しません。代わりに、外部（`main.py`）から\*\*完成品である `InventoryService` のインスタンスを注入（DI）\*\*されます。
これにより、アダプターと外部サービスとの結合も疎に保たれ、テスト時には偽物の `InventoryService` に差し替えることも可能になります。

### `check_stock(...)` と `reduce_stock(...)`: 処理の委譲

このアダプターの仕事は非常にシンプルです。`Use Case` から受け取った要求（`check_stock` の呼び出し）を、内部で保持している\*\*本物の `inventory_service` オブジェクトに、そのまま横流し（委譲：Delegate）\*\*しているだけです。

もし、外部サービスのメソッド名が我々のインターフェースと異なっていた場合（例：外部サービスが `decrease_stock` という名前だった場合）、このアダプターがその名前の違いを吸収し、「**翻訳**」する役割を担います。今回は名前が同じなので、単純な委譲となっています。

-----

## 🏛️ このレイヤーの鉄則（再確認）

1.  **契約書を必ず履行する:** `Use Cases` レイヤーで定義されたインターフェース（`IInventoryServicePort`）を必ず実装します。
2.  **依存の方向:** このファイルは、内側の `use_cases.interfaces` と、外部の `external_services.inventory_service` に依存します。**しかし、自身の存在を内側の `Use Case` に知らせることはありません。** (`UseCase` は `IInventoryServicePort` しか知らない)。
3.  **翻訳者・委譲者であれ:** 自身の責務は、インターフェースの変換と処理の委譲に徹します。ここに**ビジネスロジックを書いてはいけません**。

-----

## 📄 `interface_adapters/inventory_service_adapter.py` の実装

```python:interface_adapters/inventory_service_adapter.py
# 依存性のルール:
# このファイルは Interface Adapters レイヤーに属します。
# - 内側の use_cases.interfaces (IInventoryServicePort) に依存します。
# - 外部の external_services (InventoryService) に依存します。

# 「内側」の use_cases レイヤーからポート（インターフェース）をインポート
from use_cases.interfaces import IInventoryServicePort

# 「外部」のサービス（今回は模擬）をインポート
from external_services.inventory_service import InventoryService

class InventoryServiceAdapter(IInventoryServicePort):
    """
    【Interface Adaptersレイヤー / Adapter】
    IInventoryServicePort インターフェースの具体的な実装クラス。
    
    このクラスの責務は、我々のアプリケーション内部の要求（ポートへのメソッド呼び出し）を、
    外部の在庫管理サービスのAPI呼び出しへと「変換」し、「委譲」すること。
    """
    def __init__(self, inventory_service: InventoryService):
        """
        外部サービスのインスタンスをコンストラクタで受け取る（依存性の注入）。
        これにより、このアダプターは特定の外部サービス実装に依存しつつも、
        Use Case からはその実装を隠蔽できる。
        """
        self.inventory_service = inventory_service
        print("🔌 Inventory Service Adapter initialized.")

    def check_stock(self, product_id: str, quantity: int) -> bool:
        """
        【実装】IInventoryServicePort の契約に従い、check_stock を実装。
        処理を外部サービス (inventory_service) の同名メソッドに委譲する。
        """
        print(f"🔌 Adapter: Forwarding check_stock request for {product_id} to external service.")
        try:
            # 外部サービスを呼び出す (現実ならここで HTTPリクエストなど)
            result = self.inventory_service.check_stock(product_id, quantity)
            return result
        except Exception as e:
            # エラーハンドリング (例: 外部サービスがダウンしている場合など)
            print(f"🔌 Adapter ERROR: Failed to check stock - {e}")
            # Use Case には単純な False を返す (エラー詳細は伝えない)
            return False

    def reduce_stock(self, product_id: str, quantity: int):
        """
        【実装】IInventoryServicePort の契約に従い、reduce_stock を実装。
        処理を外部サービス (inventory_service) の同名メソッドに委譲する。
        """
        print(f"🔌 Adapter: Forwarding reduce_stock request for {product_id} to external service.")
        try:
            # 外部サービスを呼び出す
            self.inventory_service.reduce_stock(product_id, quantity)
        except ValueError as e:
            # 外部サービスが投げた在庫不足エラー(ValueError)をそのまま Use Case に伝える
            print(f"🔌 Adapter INFO: Stock reduction failed (handled by external service) - {e}")
            raise e # Use Case に例外を再スロー
        except Exception as e:
            # その他の予期せぬエラー
            print(f"🔌 Adapter ERROR: Failed to reduce stock - {e}")
            # Use Case にはより汎用的な例外を発生させる (詳細は隠蔽)
            raise RuntimeError(f"Failed to communicate with inventory service: {e}")

```

これで、「理想の鍵穴（`IInventoryServicePort`）」に合う「具体的な鍵（`InventoryServiceAdapter`）」が完成しました。
次のステップ（`SOA-05`）では、いよいよ `Use Case`（`ProcessOrderUseCase`）を修正し、この新しい「鍵」を使うように変更します。