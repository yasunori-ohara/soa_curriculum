# SOA-06 : 🔴 `infrastructure/main.py` (組立と実行)

`SOA-05` で `UseCase` が外部サービスと連携するように修正されました。いよいよ `SOA` 編の最終ステップ、これまで作ってきた全ての部品を組み立てて命を吹き込む `infrastructure/main.py` に進みます。

## 🎯 この章のゴール

  * `SOA` 化に伴い、`main.py`（Composition Root）の役割が\*\*外部サービスも含めた「総組立工場」\*\*に拡張されることを理解する。
  * 外部サービス、アダプター、内部リポジトリ、`UseCase` を正しく**インスタンス化**し、\*\*依存性の注入（DI）\*\*を行う。
  * アプリケーションを実行し、内部状態（`Order`）と外部サービス（在庫）の両方が正しく更新されることを確認する。

-----

## 🏭 このファイルの役割：全てのサービスと部品を組み立てる「総組立工場」

この `main.py` ファイルは、引き続き `Frameworks & Drivers` レイヤーに属し、\*\*Composition Root（コンポジションルート）\*\*としての役割を担います。

`SOA` への移行に伴い、この組立工場の役割はさらに重要になりました。これまでは自分たちのアプリケーション内部の部品だけを組み立てていましたが、今回は **「在庫管理サービス」という独立した外部の部品** と、それを我々のシステムに接続するための **「アダプター」** も扱う必要があります 🏭🔧。

このファイルは、まさにシステム全体の地図を唯一知っている場所であり、どの部品とどの部品を、どのようにつなぎ合わせるかを決定する、最終的な責任者です。

-----

## 💻 ソースコードの詳細解説

### `import`文: 全ての登場人物の集結

`import` 文を見てください。この `main.py` は、`domain`, `use_cases`, `interface_adapters` といった全ての内部レイヤーに加え、`external_services`（外部サービス本体）とそのアダプター（`InventoryServiceAdapter`）もインポートしています。アプリケーションを構成する全ての**具体的な登場人物**を知っているのは、このファイルだけです。

### 1\. 依存性の注入 (Dependency Injection): 外部サービスを含めた組み立て

ここが `SOA` 化による組立方法の最も大きな変更点です。

1.  まず、独立した外部サービスである **`InventoryService`** のインスタンスを生成します。
2.  次に、その外部サービス（`inventory_service`）を内包する **`InventoryServiceAdapter`** を生成します。これで、外部サービスに接続するための「変換プラグ」が完成しました。
3.  そして、内部で使うリポジトリ（`InMemoryProductRepository`, `InMemoryOrderRepository`）のインスタンスを生成します。
4.  最後に、`ProcessOrderUseCase` を生成する際に、コンストラクタに必要な全ての具体的な部品（2つのリポジトリと、在庫サービス用の**アダプター**）を注入します。

`ProcessOrderUseCase` は、自分が渡された `inventory_adapter` が、内部で `InventoryService` と通信しているという事実を**一切知りません**。ただ、それが `IInventoryServicePort` という「契約書（鍵穴）」を守っていることだけを信頼して動作します。

### 2\. 初期データの設定: 各サービスの準備

ここでは、私たちの「注文受付サービス」が管理する商品マスターデータ（`product_repo`）と、外部の「在庫管理サービス」が管理する在庫データ（`inventory_service`）の**両方**に、初期値を設定しています。それぞれのサービスが独立したデータを持っていることが明確に分かります。

### 3\. アプリケーションの実行 & 4. 結果の確認: サービス間連携の確認

`UseCase` を実行した後、注文データが `order_repo` に保存されていることを確認すると同時に、`inventory_service` の在庫が正しく削減されているかも確認します。これにより、私たちのサービスが外部サービスと正しく連携し、システム全体としてビジネスが進行したことを検証できます。

-----

## 🏛️ このレイヤーの鉄則（再確認）

  * **ビジネスロジックを書かない:** 変更なし。
  * **すべての具体的な実装を知っている:** 外部サービス `InventoryService` や `InventoryServiceAdapter` という新しい具象クラスを知っているのは、このファイルだけです。
  * **最も変更されやすい:** 在庫管理サービスへの接続方法が変わった場合（例: `InventoryService` が `REST API` になった場合）、主に修正されるのはこのファイル（DI部分で `RestInventoryAdapter` を使うように変更するなど）です。

-----

## 📄 `infrastructure/main.py` の実装

(`DDD-07` の `main.py` をリファクタリングします)

```python:infrastructure/main.py
# 依存性のルール:
# このファイルは最も外側の Infrastructure レイヤーに属します。
# アプリケーション全体を組み立てるため、すべての内側レイヤー、
# そして外部サービスやアダプターから具体的なクラスをインポートします。

# 🔵 domain から「エンティティ」をインポート (初期データ設定のため)
from domain.product import PhysicalProduct, DigitalProduct # Product を直接使わなくなった
from domain.order import Order # 結果確認用

# 🟢 use_cases から「ユースケース」と「DTO」をインポート
from use_cases.process_order_use_case import (
    ProcessOrderUseCase, 
    ProcessOrderInput, 
    ProcessOrderItemInput,
    ProcessOrderOutput # 結果表示用
)

# 🟡 interface_adapters から「内部リポジトリ」と「外部アダプター」をインポート
from interface_adapters.repositories import (
    InMemoryProductRepository, 
    InMemoryOrderRepository
)
from interface_adapters.inventory_service_adapter import InventoryServiceAdapter

# 📦 external_services から「外部サービス本体」をインポート
from external_services.inventory_service import InventoryService

def main():
    """
    【Infrastructureレイヤー / Composition Root】
    アプリケーションのすべての部品を組み立て、起動するためのメイン関数。
    SOA化に伴い、外部サービスとそのアダプターもここで組み立てる。
    """
    print("--- アプリケーション総組立開始 (SOA Version) ---")

    # --- 1. 依存性の注入 (Dependency Injection) ---
    
    # 1a. 外部サービスのインスタンスを生成
    inventory_service = InventoryService()

    # 1b. 外部サービスに接続するためのアダプターを生成し、外部サービスを注入
    inventory_adapter = InventoryServiceAdapter(inventory_service=inventory_service)

    # 1c. 内部で使うリポジトリのインスタンスを生成
    product_repo = InMemoryProductRepository()
    order_repo = InMemoryOrderRepository()

    # 1d. Use Caseを生成し、必要なすべての依存（リポジトリとアダプター）を注入
    use_case = ProcessOrderUseCase(
        product_repo=product_repo,
        order_repo=order_repo,
        inventory_service=inventory_adapter # ⬅️ UseCaseはアダプター(抽象Port)しか知らない
    )
    
    print("--- 総組立完了 ---")

    # --- 2. 初期データの設定 (各サービスの準備) ---
    print("\n--- 初期データ投入 ---")
    # 商品マスター情報を ProductRepository に保存 (Productエンティティはここで一時的に使用)
    # ProductRepository は find_by_id しか公開していないため、
    # 本来は別の方法(初期化スクリプト等)でデータを入れるべきだが、今回は便宜上直接操作
    product_repo._products["p-001"] = PhysicalProduct("p-001", "高機能マウス", 4000, 999) # 在庫は外部管理なのでダミー
    product_repo._products["p-002"] = PhysicalProduct("p-002", "静音キーボード", 6000, 999)
    print("🛒 商品マスター設定完了 (ProductRepository)")
    
    # 在庫情報を外部の InventoryService に設定
    inventory_service.add_stock("p-001", 10) # マウス在庫: 10
    inventory_service.add_stock("p-002", 5)  # キーボード在庫: 5
    
    # --- 3. アプリケーションの実行 ---
    print("\n--- ユースケース実行 ---")
    
    # シナリオ1: 正常な注文 (マウス x 2, キーボード x 1)
    print("\n[シナリオ1: 正常な注文 (マウス x 2, キーボード x 1)]")
    saved_order_id: str | None = None
    try:
        order_input_1 = ProcessOrderInput(
            customer_id="cust-101",
            items=[
                ProcessOrderItemInput(product_id="p-001", quantity=2),
                ProcessOrderItemInput(product_id="p-002", quantity=1),
            ]
        )
        # UseCase を実行し、出力 DTO を受け取る
        result_dto_1: ProcessOrderOutput = use_case.execute(order_input_1)
        print(f"✅ 注文成功: OrderID={result_dto_1.order_id}, Total={result_dto_1.total_price}円")
        saved_order_id = result_dto_1.order_id # 後の確認用
    except ValueError as e:
        print(f"❌ 注文失敗: {e}")
    except RuntimeError as e: # アダプターからの通信エラーなど
         print(f"❌ システムエラー: {e}")

    # シナリオ2: 在庫不足 (キーボード x 5)
    print("\n[シナリオ2: 在庫不足 (キーボード x 5)]")
    try:
         order_input_2 = ProcessOrderInput(
            customer_id="cust-102",
            items=[
                ProcessOrderItemInput(product_id="p-002", quantity=5), # 残り在庫は 5-1=4 のはず
            ]
        )
         result_dto_2 = use_case.execute(order_input_2)
         print(f"✅ 注文成功: OrderID={result_dto_2.order_id}, Total={result_dto_2.total_price}円")
    except ValueError as e:
        print(f"❌ 注文失敗: {e}") # ここで「在庫不足」が表示されるはず
    except RuntimeError as e:
         print(f"❌ システムエラー: {e}")

    # --- 4. 結果の確認 (サービス間連携の確認) ---
    print("\n--- 最終状態確認 ---")
    # 注文が OrderRepository に保存されているか確認
    if saved_order_id:
        print(f"\n📝 保存された注文 {saved_order_id} の確認 (OrderRepository):")
        saved_order = order_repo.find_by_id(saved_order_id)
        if saved_order:
            print(f" OrderID: {saved_order.order_id}, Total: {saved_order.total_price}円")
            print(" 明細:")
            for item in saved_order.line_items:
                 # Order集約はProductエンティティへの参照を持っている
                print(f"  - {item.product.name} x {item.quantity}") 
        else:
             print(" 保存された注文が見つかりません。")
    
    # 在庫が外部の InventoryService で正しく削減されているか確認
    print("\n📦 最終在庫の確認 (InventoryService):")
    final_mouse_stock = inventory_service.get_stock("p-001")
    final_keyboard_stock = inventory_service.get_stock("p-002")
    print(f" 高機能マウス: {final_mouse_stock}個 (初期: 10, 注文: 2 -> 期待値: 8)")
    print(f" 静音キーボード: {final_keyboard_stock}個 (初期: 5, 注文: 1 -> 期待値: 4)")

if __name__ == "__main__":
    main()

```