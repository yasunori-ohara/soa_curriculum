# 03 OpenAPI 仕様の定義

# ステップ１：OpenAPI 仕様の定義

理論を学んだところで、いよいよ実践に入ります。
最初のステップとして、サービス間でやり取りするAPIの「契約書」を OpenAPI 形式 (YAML) で定義します。これにより、各サービスがどのようなデータを要求し、どのようなデータを返すかが厳密に決まります。

## ✍️ 契約の定義

今回は、第5巡（MQTT）での情報の流れを、同期的なAPI呼び出しに置き換えます。具体的には、以下の2つのAPIを定義します。

1. **GET /world\_model:**
経路計算サービスが、認識サービスから現在の世界モデル（障害物や駐車スペースの情報）を取得するためのAPI。
2. **GET /parking\_plan:**
車両制御サービスが、経路計算サービスから計算された駐車計画（経路）を取得するためのAPI。

## 📄 openapi.yaml の作成

これら2つのAPIと、それらが使用するデータモデル（スキーマ）を、`openapi.yaml` という1つのファイルにまとめて定義します。

```yaml
# CA: Infrastructure (Adapter)
# 処理内容: サービス間通信の仕様（契約）をOpenAPI 3.0.0 形式で定義する。
# このファイルが、APIサーバーとAPIクライアントの実装における共通の「設計図」となる。
openapi: 3.0.0
info:
  title: Autonomous Parking SOA API
  description: Recognition, Planning, and Control services API definition.
  version: 1.0.0

# APIエンドポイントの定義
paths:
  # 1. 認識サービスが提供するAPI
  /world_model:
    get:
      summary: Get the current World Model
      description: 経路計算サービスが現在の世界モデルを取得するために呼び出す。
      operationId: getWorldModel
      responses:
        # 成功した場合 (HTTP 200 OK)
        '200':
          description: A successful response with the World Model.
          content:
            application/json:
              schema:
                # 戻り値のデータ型を 'components/schemas/WorldModel' で定義
                $ref: '#/components/schemas/WorldModel'

  # 2. 経路計算サービスが提供するAPI
  /parking_plan:
    get:
      summary: Get the calculated Parking Plan
      description: 車両制御サービスが計算済みの駐車計画を取得するために呼び出す。
      operationId: getParkingPlan
      responses:
        # 成功した場合 (HTTP 200 OK)
        '200':
          description: A successful response with the Parking Plan.
          content:
            application/json:
              schema:
                # 戻り値のデータ型を 'components/schemas/ParkingPlan' で定義
                $ref: '#/components/schemas/ParkingPlan'

# 共通データモデル（スキーマ）の定義
components:
  schemas:
    # --- WorldModel とその構成要素 ---

    Point:
      type: object
      properties:
        x:
          type: number
          format: double
        y:
          type: number
          format: double
      required: [x, y]

    Pose:
      type: object
      properties:
        x:
          type: number
          format: double
        y:
          type: number
          format: double
        theta:
          type: number
          format: double
      required: [x, y, theta]

    Obstacle:
      type: object
      properties:
        id:
          type: integer
        polygon:
          type: array
          items:
            $ref: '#/components/schemas/Point'
      required: [id, polygon]

    ParkingSpot:
      type: object
      properties:
        id:
          type: integer
        is_occupied:
          type: boolean
        polygon:
          type: array
          items:
            $ref: '#/components/schemas/Point'
      required: [id, is_occupied, polygon]

    WorldModel:
      type: object
      description: 認識サービスが提供する世界モデル。
      properties:
        timestamp:
          type: string
          format: date-time
        obstacles:
          type: array
          items:
            $ref: '#/components/schemas/Obstacle'
        parking_spots:
          type: array
          items:
            $ref: '#/components/schemas/ParkingSpot'
        vehicle_pose:
          $ref: '#/components/schemas/Pose'
      required: [timestamp, obstacles, parking_spots, vehicle_pose]

    # --- ParkingPlan ---

    ParkingPlan:
      type: object
      description: 経路計算サービスが提供する駐車計画。
      properties:
        plan_id:
          type: string
          format: uuid
        status:
          type: string
          enum: [success, failure, pending]
        path:
          type: array
          items:
            # 駐車経路はPose(位置と向き)のリストとする
            $ref: '#/components/schemas/Pose'
      required: [plan_id, status, path]

```

## 📜 仕様の解説

このYAMLファイルは、APIの仕様を機械可読な形で定義しています。

- `paths:`:
APIのエンドポイント（`/world_model` と `/parking_plan`）を定義します。今回はどちらも `get`（データ取得）操作のみを定義しています。
- `responses: '200':`:
`GET` リクエストが成功した場合（HTTPステータスコード200）のレスポンスを定義しています。
- `content: application/json: schema: $ref:`:
レスポンスの形式がJSONであり、その具体的なデータ構造（スキーマ）を `components/schemas/` 以下で定義した `WorldModel` や `ParkingPlan` への参照 (`$ref`) として指定しています。
- `components: schemas:`:
APIで再利用されるデータモデルを定義する場所です。ここで `WorldModel` や `ParkingPlan`、およびそれらが内部で使う `Pose` や `Obstacle` などの詳細なデータ型（プロパティ名、データ型、必須項目）を厳密に定義しています。

この「契約書」ファイル (`openapi.yaml`) を中心に、次のステップから各サービス（サーバー側とクライアント側）を実装していきます。