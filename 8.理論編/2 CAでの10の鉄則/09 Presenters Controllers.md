# 09 Presenters / Controllers

# 🏛️ Presenters / Controllers

これまでの解説で `Adapters` 層（または Interface Adapters 層）には、外部システム（DB, ネットワークなど）とのやり取りを行う Gateway や Repository が含まれることを見てきました。この層にはもう一つ重要な役割があります。それは、**ユーザーインターフェース (UI) や Web API と UseCase との間でデータをやり取りし、形式を変換する**ことです。この役割を担うのが主に **Controllers** と **Presenters** です。

---

## ❓ 定義

- **Controllers (コントローラ)**:
    - **入力**を担当します。UI や Web API からのリクエストを受け取り、それを解析し、UseCase が理解できる形式（Input Data / DTO）に変換して、適切な UseCase を呼び出します。
- **Presenters (プレゼンター)**:
    - **出力**を担当します。UseCase から処理結果（Output Data / DTO / Entity）を受け取り、それを UI（画面）や API レスポンスが表示・利用しやすい形式（ViewModel / JSON）に変換します。

### 💡 簡単に言うと

- **Controller**: 外部からの「注文（リクエスト）」を受け付けて、中の人（UseCase）に分かる言葉で伝える「受付係」。
- **Presenter**: 中の人（UseCase）からの「報告書」を受け取って、外の人（UI）が見やすいように「清書・整形」する係。

### 🤔 なぜ重要か？

Controller と Presenter を導入し、UseCase から分離することで、以下の利点があります。

1. **UI/フレームワークからの独立性**:
    - UseCase は、HTTP リクエストのヘッダーがどうなっているか、レスポンスが HTML なのか JSON なのか、といった UI やフレームワーク固有の詳細を知る必要がなくなります。Controller と Presenter がその違いを吸収します。（フレームワーク独立性・UI独立性の実現）
2. **関心の分離**:
    - UseCase は純粋なビジネスロジックに集中できます。入力データの解析や出力データの整形といった「表示や伝達に関わる関心事」は Controller / Presenter に分離されます。
3. **テスト容易性**:
    - UseCase のテストは、UI やフレームワークを起動せずに行えます。Controller や Presenter も、UseCase をモックに差し替えるなどして、比較的容易にテストできます。

---

## ✅ これまでの実践例（どこで使ったか）

今回の SOA の例では直接的な UI がなかったため、Controller や Presenter は明確には登場しませんでした。しかし、その役割の一部は他のコンポーネントが担っていました。

### 📌 (参考) 第1巡・第2巡の Web アプリケーション

- **具体例**: もし Flask や FastAPI を使って Web アプリケーションを構築した場合、フレームワークが提供する機能が Controller の役割の一部を担います。
    - Flask のルートハンドラ (`@app.route(...) def handler(): ...`) や FastAPI のパスオペレーション関数 (`@app.post(...) async def operation(...): ...`) が HTTP リクエストを受け取り、リクエストボディやパスパラメータを解析します。
    - これらの関数の中から UseCase を呼び出し、必要なデータを渡します。
- **実践**: CA を適用する場合、これらのハンドラ/オペレーション関数は「薄く」保ち、リクエスト解析と UseCase 呼び出しに徹するのが理想です。ビジネスロジックは UseCase に委ねます。レスポンスの整形（JSON 化など）は、Presenter に相当する別のコンポーネント（またはハンドラ関数の後半部分）が担当します。

### 📌 (参考) SOA における `main.py` や Adapter

- **具体例**: 第5巡の `control_service/main.py` は `ExecutePlanUseCase` を呼び出しました。`MqttPlanSubscriber` は MQTT メッセージ (JSON 文字列) を `ParkingPlan` Entity に変換しました。
- **実践**:
    - `main.py` は、外部からの起動トリガー（OS による実行）を受け付けて UseCase を呼び出す点で、広義の Controller の役割の一部を担っています。
    - `MqttPlanSubscriber` の `_on_message` メソッド内で行われた「JSON 文字列から Entity への変換」は、Controller が行う「入力データの変換」に相当します。
    - 同様に `MqttWorldModelPublisher` が行った「Entity から JSON 文字列への変換」は、Presenter が行う「出力データの変換」に相当します。

---

## ❌ 間違った適用例（アンチパターン）

Controller や Presenter の役割が UseCase や他の層に漏れ出すと、関心の分離が崩れ、テストや変更が困難になります。

- **例1：UseCase が HTTP レスポンスを直接生成する**
(フレームワーク独立性のアンチパターン1と同じです)
    
    ```python
    # アンチパターン：UseCase が Presenter の役割を担う
    # from flask import jsonify
    
    class ProcessOrderUseCase:
        def handle(self, order_data):
            # ... ビジネスロジック ...
            order_id = ...
    
            # NG! UseCase が HTTP レスポンス形式 (JSON) とステータスコードを知っている！
            # return jsonify({"status": "success", "order_id": order_id}), 201
            pass # 代わりに Entity や DTO を返すべき
    
    ```
    
    UseCase が特定の出力形式（JSON や HTTP ステータスコード）を知ってしまっています。出力形式が変わると UseCase の修正が必要です。
    
- **例2：Controller がビジネスロジックを実行する**
Web フレームワークのハンドラ関数内に、UseCase を呼び出すだけでなく、複雑なビジネスルールやデータ永続化処理まで書いてしまうパターンです。
    
    ```python
    # アンチパターン：Controller が UseCase の役割を担う
    # from flask import request, jsonify
    # import db_driver # Adapter 層のモジュール
    
    # @app.route('/orders', methods=['POST'])
    # def create_order():
    #     order_data = request.get_json()
    
    #     # --- ここから下は UseCase がやるべき ---
    #     # 1. バリデーション (ビジネスルール)
    #     if not order_data.get('item') or order_data.get('quantity', 0) <= 0:
    #         return jsonify({"error": "Invalid order data"}), 400
    
    #     # 2. 在庫確認 (別のビジネスルール / DBアクセス)
    #     stock = db_driver.get_stock(order_data['item'])
    #     if stock < order_data['quantity']:
    #         return jsonify({"error": "Insufficient stock"}), 400
    
    #     # 3. 注文作成 & 保存 (DBアクセス)
    #     new_order_id = db_driver.save_order(order_data)
    #     db_driver.update_stock(order_data['item'], -order_data['quantity'])
    #     # --- ここまで UseCase の範囲 ---
    
    #     # レスポンス生成 (Presenter/Controller の役割)
    #     return jsonify({"status": "success", "order_id": new_order_id}), 201
    
    ```
    
    Controller があまりにも多くの責任（入力解析、バリデーション、在庫確認、注文永続化、レスポンス生成）を持ちすぎています（単一責任の原則違反）。ビジネスロジックのテストが困難になり、再利用もできません。
    

---

## 📝 まとめ

Controllers と Presenters は、Interface Adapters 層に属し、それぞれ**入力データの変換**と**出力データの変換**という重要な責任を担います。

これらを UseCase から明確に分離することで、UseCase を UI やフレームワークの詳細から守り、**関心の分離**、**テスト容易性**、そして\*\*各種独立性（フレームワーク、UI）\*\*を高めることができます。

- **Controller**: 外 → 内 (Request → UseCase Input)
- **Presenter**: 内 → 外 (UseCase Output → Response/ViewModel)

---

## ➡️ 次へ

いよいよ最後の復習項目、「**(10/10) DTO (Data Transfer Objects)**」について見ていきましょう。これは、レイヤー間でデータをやり取りする際の形式に関わる重要な概念です。