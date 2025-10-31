# DDD-07 : 🔴 `infrastructure/main.py` (組立と実行)

`domain`（宝物）、`use_cases`（指揮者と鍵穴）、`interface_adapters`（具体的な鍵）がすべて揃いました。
`DDD` リファクタリングの最後を飾る、すべてを組み立てる `infrastructure/main.py` に進みます。

## 🎯 この章のゴール

  * `domain` レイヤーの変更（`Order` 集約）に伴い、`main.py` の「組立方法」も変わることを理解する。
  * `UseCase` が要求する**複数**のリポジトリ（`IProductRepository`, `IOrderRepository`）を正しく\*\*注入（DI）\*\*する。
  * `UseCase` の入出力として定義した **DTO**（`ProcessOrderInput` など）を使って、アプリケーションを実行する。

-----

## 🏭 このファイルの役割：新しい部品で組み立てる「組立工場」

この `main.py` ファイルは、引き続き `Frameworks & Drivers` レイヤーに属し、\*\*Composition Root（コンポジションルート）\*\*としての役割を担います。

`domain` モデルが豊か（リッチ）になり、新しい部品（`Order` 集約、`IOrderRepository`）が作られたことで、この組立工場の仕事も変わりました。新しい部品（`InMemoryOrderRepository`）を正しく認識し、適切な場所（`ProcessOrderUseCase` のコンストラクタ）に組み込む必要があります。

-----

## 💻 ソースコードの詳細解説

### 1\. `import`文: 新しい部品の設計図を追加

`domain.order` から `Order`（デバッグ用）を、`use_cases.process_order` から `ProcessOrderInput` などのDTOを、`interface_adapters.repositories` から `InMemoryOrderRepository` をインポートしています。組立工場が、新しく作られた部品の存在を認識したことを示します。

### 2\. 依存性の注入 (Dependency Injection): 新しい部品の組み立て

ここが今回の最も重要な変更点です。

1.  `InMemoryProductRepository()`（商品リポジトリ）を生成します。
2.  **新しく `InMemoryOrderRepository()`（注文リポジトリ）を生成します。**
3.  `ProcessOrderUseCase` を生成する際に、コンストラクタに \*\*2つのリポジトリ（`product_repo` と `order_repo`）\*\*を注入します。

これにより、`ProcessOrderUseCase` は、仕事を遂行するために必要な道具（リポジトリ）をすべて外部から与えられ、自身の仕事に集中できます。

### 3\. アプリケーションの実行: 新しい指示書（DTO）で仕事を依頼

`UseCase` を呼び出す方法も変わりました。`CA-06` では単純な引数（`product_id`, `quantity`）を渡していましたが、今回は \*\*`ProcessOrderInput` という構造化されたデータ（DTO）\*\*で仕事を依頼します。
これにより、「誰が、どの商品を、何個」という複雑な注文内容を、より明確で安全な形で `UseCase` に伝えることができます。

-----

## 🏛️ このレイヤーの鉄則（再確認）

  * **ビジネスロジックを書かない:**
    `if` 文は `result` の成功/失敗を判定するだけで、在庫チェックなどのビジネスロジックは一切ありません。
  * **すべての「具象」を知っている:**
    `InMemoryProductRepository` と `InMemoryOrderRepository` という、具体的な実装クラスを知っている（`new` している）のは、このファイルだけです。
  * **最も変更されやすい:**
    `InMemory` を `MySQL` に変更する場合、修正が必要なのはこの `main.py` のDI部分だけです。

-----

## 📄 `infrastructure/main.py` の実装

（`CA-06` の `main.py` をリファクタリングします）

```python:infrastructure/main.py
# 依存性のルール:
# このファイルは最も外側の Infrastructure レイヤーに属します。
# アプリケーション全体を組み立てるため、すべての内側のレイヤーから
# クラスをインポートすることが許される、唯一の場所です。

# 🔵 domain から「エンティティ」をインポート (初期データ設定のため)
from domain.product import PhysicalProduct, DigitalProduct
from domain.order import Order # (結果確認用にインポート)

# 🟢 use_cases から「ユースケース」と「DTO」をインポート
from use_cases.process_order_use_case import (
    ProcessOrderUseCase, 
    ProcessOrderInput, 
    ProcessOrderItemInput
)

# 🟡 interface_adapters から「具体的な実装（リポジトリ）」をインポート
from interface_adapters.repositories import (
    InMemoryProductRepository, 
    InMemoryOrderRepository
)

def main():
    """
    【Infrastructureレイヤー / Composition Root】
    アプリケーションのすべての部品を組み立て、起動するためのメイン関数。
    DDDを適用したことで、新しい部品(OrderRepository)が増え、組立方法が変わった。
    """
    print("--- アプリケーション組立開始 (Composition Root) ---")
    
    # --- 1. 依存性の注入 (Dependency Injection) ---
    # 必要なリポジトリの具体的な実装クラスをすべてインスタンス化する
    product_repo = InMemoryProductRepository()
    order_repo = InMemoryOrderRepository() # ⬅️ 新しいリポジトリを追加
    
    # Use Caseが必要とするすべての依存（リポジトリ）をコンストラクタ経由で注入する
    use_case = ProcessOrderUseCase(
        product_repo=product_repo, 
        order_repo=order_repo
    )
    
    print("--- 組立完了 ---")

    # --- 2. 初期データの設定 (インフラ層の責任) ---
    print("\n--- 初期在庫データ投入 ---")
    mouse = PhysicalProduct("p-001", "高機能マウス", 4000, 10)
    keyboard = PhysicalProduct("p-002", "静音キーボード", 6000, 5)
    ebook = DigitalProduct("d-001", "Python入門 eBook", 3000)
    product_repo.save(mouse)
    product_repo.save(keyboard)
    product_repo.save(ebook)
    
    # --- 3. アプリケーションの実行 (インフラ層の責任) ---
    print("\n--- ユースケース実行 ---")
    
    # シナリオ1: 正常な注文（マウス x 2, eBook x 1）
    print("\n[シナリオ1: 正常な注文 (マウス x 2, eBook x 1)]")
    # Use Caseへの入力は、構造化されたDTO(ProcessOrderInput)で行う
    order_input_1 = ProcessOrderInput(
        customer_id="cust-001",
        items=[
            ProcessOrderItemInput(product_id="p-001", quantity=2),
            ProcessOrderItemInput(product_id="d-001", quantity=1),
        ]
    )
    
    saved_order_id: str | None = None
    try:
        # UseCase を実行し、出力 DTO (ProcessOrderOutput) を受け取る
        result_dto_1 = use_case.execute(order_input_1)
        print(f"成功: 注文ID={result_dto_1.order_id}, 合計金額={result_dto_1.total_price}円")
        # 成功した注文IDを、後の確認用に保存しておく
        saved_order_id = result_dto_1.order_id
        
    except ValueError as e:
        print(f"失敗: {e}")

    # シナリオ2: 在庫不足
    print("\n[シナリオ2: 在庫不足 (キーボード x 10)]")
    try:
        order_input_2 = ProcessOrderInput(
            customer_id="cust-002",
            items=[
                ProcessOrderItemInput(product_id="p-002", quantity=10), # 在庫は5個
            ]
        )
        result_dto_2 = use_case.execute(order_input_2)
        print(f"成功: 注文ID={result_dto_2.order_id}, 合計金額={result_dto_2.total_price}円")
    except ValueError as e:
        print(f"失敗: {e}") # ここで「在庫が不足しています」と表示されるはず

    # --- 4. 結果の確認 (インフラ層の責任) ---
    print("\n--- 最終状態の確認 ---")
    
    # 在庫が減っているか確認
    final_mouse = product_repo.find_by_id("p-001")
    if final_mouse:
        print(f"{final_mouse.name}の最終在庫: {final_mouse.get_stock()} (初期値: 10)")

    # 成功した注文がOrderRepositoryに保存されているか確認
    if saved_order_id:
        print(f"\n--- 保存された注文 {saved_order_id} の確認 ---")
        saved_order = order_repo.find_by_id(saved_order_id)
        if saved_order:
            print(f" 注文ID: {saved_order.order_id}")
            print(f" 合計金額: {saved_order.total_price}円")
            print(f" 確定状態: {saved_order._confirmed}")
            print(" 明細:")
            for item in saved_order.line_items:
                print(f"  - {item.product.name} x {item.quantity}")

if __name__ == "__main__":
    main()
```