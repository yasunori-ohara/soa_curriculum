# DDD-08

```markdown
## `main.py` (フレームワーク＆ドライバ)

すべてを組み立てる**`main.py`**に進みます。

---

### 1. このファイルの役割：新しい部品で組み立てる「組立工場」

この`main.py`ファイルは、引き続きFrameworks & Drivers（フレームワーク＆ドライバ）レイヤーに属し、**Composition Root（コンポジションルート）**としての役割を担います。

ドメインモデルが豊かになり、新しい部品（`Order`集約、`OrderRepository`）が作られたことで、この組立工場の仕事も少し変わりました。新しい部品を正しく認識し、適切な場所に組み込む必要があります。

### 2. ソースコードの詳細解説

#### 1. `import`文: 新しい部品の設計図を追加
`domain.order`から`Order`を、`use_cases.process_order`から`ProcessOrderInput`と`ProcessOrderOutput`を、そして`interface_adapters.repositories`から`InMemoryOrderRepository`をインポートしています。組立工場が、新しく作られた部品の存在を認識したことを示します。

#### 2. 依存性の注入 (`Dependency Injection`): 新しい部品の組み立て
ここが今回の最も重要な変更点です。

1. 以前と同様に`InMemoryProductRepository`を生成します。
2. 新しく`InMemoryOrderRepository`を生成します。
3. `ProcessOrderUseCase`を生成する際に、コンストラクタに2つのリポジトリ（`product_repo`と`order_repo`）を注入します。

これにより、`ProcessOrderUseCase`は、仕事を遂行するために必要な道具（リポジトリ）をすべて外部から与えられ、自身の仕事に集中できます。

#### 3. アプリケーションの実行: 新しい指示書（DTO）で仕事を依頼
ユースケースを呼び出す方法も変わりました。以前は単純な引数（`product_id`, `quantity`）を渡していましたが、今回は**`ProcessOrderInput`という構造化されたデータ（DTO）**で仕事を依頼します。  
これにより、「どの商品を何個」という複雑な注文内容を、より明確で安全な形でユースケースに伝えることができます。

#### 4. 結果の確認: 新しい保管庫から結果を確認
注文が成功したかどうかを確認するために、`order_repo`（注文専用の保管庫）を使って、保存された`Order`集約をIDで検索し、その内容（合計金額など）を表示しています。

### 3. このレイヤーの鉄則（再確認）

- ビジネスロジックを書かない: 変更ありません。
- べての具体的な実装を知っている: `InMemoryOrderRepository`という新しい具体的な実装を知っているのは、このファイルだけです。
- 最も変更されやすい: 変更ありません。

---

### `main.py`

``` Python
# 依存性のルール:
# このファイルは最も外側のレイヤーに属します。
# アプリケーション全体を組み立てるため、すべての内側のレイヤーから
# 具体的なクラスをインポートすることが許される、唯一の場所です。

from domain.product import PhysicalProduct, DigitalProduct
from use_cases.process_order import ProcessOrderUseCase, ProcessOrderInput
from interface_adapters.repositories import InMemoryProductRepository, InMemoryOrderRepository

def main():
    """
    【Frameworks & Driversレイヤー / Composition Root】
    アプリケーションのすべての部品を組み立て、起動するためのメイン関数。
    DDDを適用したことで、新しい部品(OrderRepository)が増え、組立方法が変わった。
    """
    print("--- アプリケーション組立開始 ---")

    # --- 1. 依存性の注入 (Dependency Injection) ---
    # 必要なリポジトリの具体的な実装クラスをすべてインスタンス化する
    product_repo = InMemoryProductRepository()
    order_repo = InMemoryOrderRepository() # ⬅️ 新しいリポジトリを追加

    # Use Caseが必要とするすべての依存（リポジトリ）をコンストラクタ経由で注入する
    use_case = ProcessOrderUseCase(product_repo=product_repo, order_repo=order_repo)
    
    print("--- 組立完了 ---")

    # --- 2. 初期データの設定 ---
    print("\n--- 初期在庫データ投入 ---")
    mouse = PhysicalProduct("p-001", "高機能マウス", 4000, 10)
    keyboard = PhysicalProduct("p-002", "静音キーボード", 6000, 5)
    ebook = DigitalProduct("d-001", "Python入門 eBook", 3000)
    product_repo.save(mouse)
    product_repo.save(keyboard)
    product_repo.save(ebook)
    
    # --- 3. アプリケーションの実行 ---
    print("\n--- ユースケース実行 ---")
    
    # シナリオ1: 正常な注文
    try:
        # Use Caseへの入力は、構造化されたDTO(ProcessOrderInput)で行う
        order_input_1 = ProcessOrderInput(items=[
            {"product_id": "p-001", "quantity": 2},
            {"product_id": "d-001", "quantity": 1},
        ])
        result = use_case.execute(order_input_1)
        print(f"成功: 注文ID={result.order_id}, 合計金額={result.total_price}円")
    except ValueError as e:
        print(f"失敗: {e}")

    # シナリオ2: 在庫不足
    try:
        order_input_2 = ProcessOrderInput(items=[
            {"product_id": "p-002", "quantity": 10}, # 在庫は5個なので失敗するはず
        ])
        result = use_case.execute(order_input_2)
        print(f"成功: 注文ID={result.order_id}, 合計金額={result.total_price}円")
    except ValueError as e:
        print(f"失敗: {e}")

    # --- 4. 結果の確認 ---
    # 成功した注文がOrderRepositoryに保存されているか確認する
    print("\n--- 注文データ確認 ---")
    saved_order = order_repo.find_by_id("order-") # IDの先頭部分で検索(簡易のため)
    if saved_order:
        # 実際のIDはUUIDなので、ここでは最初の注文を探す簡易的な処理
        first_order_id = list(order_repo._orders.keys())[0]
        first_order = order_repo.find_by_id(first_order_id)
        if first_order:
            print(f"保存された注文: ID={first_order.order_id}, 合計金額={first_order.total_price}円")
            for item in first_order._line_items:
                print(f"  - {item.product_name} x {item.quantity}")

if __name__ == "__main__":
    main()
```

```