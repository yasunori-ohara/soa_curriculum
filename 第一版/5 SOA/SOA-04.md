# SOA-04

```markdown
## `interface_adapters/inventory_service_adapter.py`

外部サービスとの「契約書」を定義した次は、その契約書を具体的に履行するための部品、**`interface_adapters/inventory_service_adapter.py`**に進みます。

---

### 1. このファイルの役割：外部サービスへの「変換プラグ」

このファイルは、Interface Adapters（インターフェースアダプター）レイヤーに属します。その名の通り、まさに「アダプター」としての役割を担います。

Use Casesレイヤーが定義した`InventoryServicePort`という我々のアプリケーション内部の「コンセントの形」と、`external_services`が提供する`InventoryService`という外部の「特殊な形状のプラグ」を、うまく適合させるための変換プラグだと考えてください。

このアダプターのおかげで、Use Caseは接続先がどんな形のサービスであっても、常に標準的なコンセント（ポート）に接続するだけで済むのです。

### 2. ソースコードの詳細解説

`InventoryServiceAdapter(InventoryServicePort)`: 契約の履行
このクラスは、`use_cases/interfaces.py`で定義された`InventoryServicePort`という契約書に「署名」しています。これにより、このクラスが`check_stock`と`reduce_stock`というメソッドを必ず実装することが保証されます。

`__init__(self, inventory_service: InventoryService)`: 外部サービスの注入
このアダプターは、自身で外部サービス（`InventoryService`）のインスタンスを生成しません。代わりに、外部から**完成品である`InventoryService`のインスタンスを注入（DI）**されます。
これにより、アダプターと外部サービスとの結合も疎に保たれ、テスト時には偽物の`InventoryService`に差し替えることも可能になります。

`check_stock(...)` と `reduce_stock(...)`: 処理の委譲
このアダプターの仕事は非常にシンプルです。Use Caseから受け取った要求（`check_stock`の呼び出し）を、内部で保持している**本物の`inventory_service`オブジェクトに、そのまま横流し（委譲：Delegate）**しているだけです。

もし、外部サービスのメソッド名が我々のインターフェースと異なっていた場合（例：外部サービスが`decrease_stock`という名前だった場合）、このアダプターがその名前の違いを吸収し、翻訳する役割を担います。

### 3. このレイヤーの鉄則（再確認）

1. 契約書を必ず履行する: Use Casesレイヤーで定義されたインターフェースを必ず実装します。

2. 依存の矢印は必ず内側を向く: このファイルは、内側の`use_cases/interfaces.py`と、外部の`external_services/inventory_service.py`に依存します。しかし、自身の存在を内側のUse Caseに知らせることはありません。

3. 翻訳者・委譲者であれ: 自身の責務は、インターフェースの変換と処理の委譲に徹します。ここにビジネスロジックを書いてはいけません。

---
### interface_adapters/inventory_service_adapter.py

``` Python
# 依存性のルール:
# このファイルはInterface Adaptersレイヤーに属します。
# 内側のuse_cases.interfacesと、外部のexternal_servicesに依存します。

from use_cases.interfaces import InventoryServicePort
from external_services.inventory_service import InventoryService

class InventoryServiceAdapter(InventoryServicePort):
    """
    【Interface Adaptersレイヤー / Adapter】
    InventoryServicePortインターフェースの具体的な実装クラス。
    
    このクラスの責務は、我々のアプリケーション内部の要求（ポートへのメソッド呼び出し）を、
    外部の在庫管理サービスのAPI呼び出しへと「変換」し、「委譲」すること。
    """
    def __init__(self, inventory_service: InventoryService):
        # 外部サービスのインスタンスをコンストラクタで受け取る（依存性の注入）
        self.inventory_service = inventory_service

    def check_stock(self, product_id: str, quantity: int) -> bool:
        """
        Use Caseからの在庫確認要求を、外部サービスの同名メソッドに委譲する。
        """
        print(f"🔌 アダプター: 在庫確認要求を外部サービスに転送します。(商品ID: {product_id})")
        return self.inventory_service.check_stock(product_id, quantity)

    def reduce_stock(self, product_id: str, quantity: int):
        """
        Use Caseからの在庫削減要求を、外部サービスの同名メソッドに委譲する。
        """
        print(f"🔌 アダプター: 在庫削減要求を外部サービスに転送します。(商品ID: {product_id})")
        self.inventory_service.reduce_stock(product_id, quantity)
```

```