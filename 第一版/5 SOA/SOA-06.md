# SOA-06

```markdown
## `main.py` (フレームワーク＆ドライバ)

これまで作ってきた全ての部品を組み立てて命を吹き込む**`main.py`**に進みます。

---
### 1. このファイルの役割：全てのサービスと部品を組み立てる「総組立工場」

このmain.pyファイルは、引き続きFrameworks & Drivers（フレームワーク＆ドライバ）レイヤーに属し、**Composition Root（コンポジションルート）**としての役割を担います。

SOAへの移行に伴い、この組立工場の役割はさらに重要になりました。これまでは自分たちのアプリケーション内部の部品だけを組み立てていましたが、今回は**「在庫管理サービス」という独立した外部の部品と、それを我々のシステムに接続するための「アダプター」**も扱う必要があります。

このファイルは、まさにシステム全体の地図を唯一知っている場所であり、どの部品とどの部品を、どのようにつなぎ合わせるかを決定する、最終的な責任者です。

### 2. ソースコードの詳細解説

`import`文: 全ての登場人物の集結
`import`文を見てください。この`main.py`は、`domain`, `use_cases`, `interface_adapters`といった全ての内部レイヤーに加え、`external_services`とそのアダプターもインポートしています。アプリケーションを構成する全ての具体的な登場人物を知っているのは、このファイルだけです。

#### 1. 依存性の注入 (Dependency Injection): 外部サービスを含めた組み立て
ここがSOA化による組立方法の最も大きな変更点です。

1. まず、独立した外部サービスである**`InventoryService`**のインスタンスを生成します。

2. 次に、その外部サービス（`inventory_service`）を内包する**`InventoryServiceAdapter`**を生成します。これで、外部サービスに接続するための「変換プラグ」が完成しました。

3. そして、`ProcessOrderUseCase`を生成する際に、コンストラクタに必要な全ての具体的な部品（2つのリポジトリと、在庫サービス用のアダプター）を注入します。

`ProcessOrderUseCase`は、自分が渡された`inventory_adapter`が、内部で`InventoryService`と通信しているという事実を一切知りません。ただ、それが`InventoryServicePort`という契約書を守っていることだけを信頼して動作します。

#### 2. 初期データの設定: 各サービスの準備
ここでは、私たちの「注文受付サービス」が管理する商品マスターデータ（`product_repo`）と、外部の「在庫管理サービス」が管理する在庫データ（`inventory_service`）の両方に、初期値を設定しています。それぞれのサービスが独立したデータを持っていることが明確に分かります。

#### 3. アプリケーションの実行 & 4. 結果の確認: サービス間連携の確認
ユースケースを実行した後、注文データが`order_repo`に保存されていることを確認すると同時に、`inventory_service`の在庫が正しく削減されているかも確認します。これにより、私たちのサービスが外部サービスと正しく連携し、システム全体としてビジネスが進行したことを検証できます。

### 3. このレイヤーの鉄則（再確認）

- ビジネスロジックを書かない
- すべての具体的な実装を知っている
- 最も変更されやすい

---
### main.py
``` Python
# 依存性のルール:
# このファイルは最も外側のレイヤーに属します。
# アプリケーション全体を組み立てるため、すべての内側レイヤー、
# そして外部サービスやアダプターから具体的なクラスをインポートします。

from domain.product import Product
from use_cases.process_order import ProcessOrderUseCase, ProcessOrderInput
from interface_adapters.repositories import InMemoryProductRepository, InMemoryOrderRepository
from interface_adapters.inventory_service_adapter import InventoryServiceAdapter
from external_services.inventory_service import InventoryService

def main():
    """
    【Frameworks & Driversレイヤー / Composition Root】
    アプリケーションのすべての部品を組み立て、起動するためのメイン関数。
    SOA化に伴い、外部サービスとそのアダプターもここで組み立てる。
    """
    print("--- アプリケーション総組立開始 ---")

    # --- 1. 依存性の注入 (Dependency Injection) ---
    # 1-1. 外部サービスのインスタンスを生成
    inventory_service = InventoryService()

    # 1-2. 外部サービスに接続するためのアダプターを生成し、外部サービスを注入
    inventory_adapter = InventoryServiceAdapter(inventory_service=inventory_service)

    # 1-3. 内部で使うリポジトリのインスタンスを生成
    product_repo = InMemoryProductRepository()
    order_repo = InMemoryOrderRepository()

    # 1-4. Use Caseを生成し、必要なすべての依存（リポジトリとアダプター）を注入
    use_case = ProcessOrderUseCase(
        product_repo=product_repo,
        order_repo=order_repo,
        inventory_service=inventory_adapter # ⬅️ UseCaseはアダプターの存在しか知らない
    )
    
    print("--- 総組立完了 ---")

    # --- 2. 初期データの設定 ---
    print("\n--- 初期データ投入 ---")
    # 商品マスター情報をProductRepositoryに保存
    mouse = Product("p-001", "高機能マウス", 4000)
    keyboard = Product("p-002", "静音キーボード", 6000)
    product_repo.save(mouse)
    product_repo.save(keyboard)
    
    # 在庫情報を外部のInventoryServiceに設定
    inventory_service.add_stock("p-001", 10)
    inventory_service.add_stock("p-002", 5)
    
    # --- 3. アプリケーションの実行 ---
    print("\n--- ユースケース実行 ---")
    
    # シナリオ1: 正常な注文
    try:
        order_input_1 = ProcessOrderInput(items=[
            {"product_id": "p-001", "quantity": 2},
            {"product_id": "p-002", "quantity": 1},
        ])
        result = use_case.execute(order_input_1)
        print(f"✅ 成功: 注文ID={result.order_id}, 合計金額={result.total_price}円")
    except ValueError as e:
        print(f"❌ 失敗: {e}")

    # --- 4. 結果の確認 ---
    print("\n--- 最終状態確認 ---")
    # 注文がOrderRepositoryに保存されているか確認
    first_order_id = list(order_repo._orders.keys())[0]
    saved_order = order_repo.find_by_id(first_order_id)
    if saved_order:
        print(f"📝 保存された注文: ID={saved_order.order_id}, 合計金額={saved_order.total_price}円")
    
    # 在庫が外部のInventoryServiceで削減されているか確認
    final_mouse_stock = inventory_service.get_stock("p-001")
    print(f"📦 {mouse.name}の最終在庫: {final_mouse_stock}個") # 10 -> 8 になっているはず

if __name__ == "__main__":
    main()
```

```