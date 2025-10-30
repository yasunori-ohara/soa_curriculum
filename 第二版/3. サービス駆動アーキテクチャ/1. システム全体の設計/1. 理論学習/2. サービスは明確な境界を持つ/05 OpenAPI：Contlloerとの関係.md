## API仕様(OpenAPI)と実装(Controller)の具体的な関係

さきほど、`Interface`とOpenAPIが「内部」と「外部」という根本的な違いを持つことを見ました。このページでは、その「外部との契約」であるOpenAPIと、それを「実装」する`Controller`の具体的な関係を、コードレベルで詳しく見ていきます。

### 🛠️ 仕様が実装を決め、実装が仕様を生む

`Controller`とOpenAPIの関係は、一見「鶏が先か、卵が先か」のように見えることがあります。なぜなら、その関係は開発の進め方によって双方向になり得るからです。

  * **仕様 → 実装の骨格（スケルトン）**
    `openapi.yaml`という「設計図」を先に用意すれば、ツールを使って`Controller`クラスの「骨格（スケルトン）」を自動生成できます。これは、メソッド名や引数など、契約に必要な部分だけが記述された雛形コードです。開発者は、その骨格の中に具体的な処理（実行ロジック）を記述していきます。

  * **実装 → 仕様**
    逆に、`Controller`のコードには「外部とのAPI仕様」と「内部のロジック」の両方が含まれているため、人間やツールがそのコードを読み解き、「API仕様」の部分だけを抜き出して`openapi.yaml`という「設計図」を作成することも可能です。

### 🛠️ コードで理解する「契約」と「実行」

この関係性を、実際のコードでさらに詳しく見ていきましょう。

#### **前提知識：コードに出てくる用語**

  * **`DTO (Data Transfer Object)`**: データ転送専用のシンプルな入れ物（オブジェクト）です。この例の`UserResponseDto`は、「サービスからクライアントへユーザー情報を返す」目的のために、`id`と`name`という属性を持つ入れ物として別途定義されていると想定します。
  * **`デコレータ (@)`**: Pythonの`@`で始まる記述で、既存の関数に後から機能を追加・修飾するための仕組みです。Webフレームワークでは、「このURLへのリクエストが来たら、この関数を呼び出す」という**紐付け**によく使われます。

#### **OpenAPIの仕様（設計図）**

`openapi.yaml`ファイルは、サービスが提供するAPI全体の仕様を記述します。ここでは、その中から`/users/{userId}`というAPI仕様の部分に注目します。

```yaml
# openapi.yaml
paths:
  # ▼▼▼▼▼ 今回注目するAPI仕様の箇所 ▼▼▼▼▼
  /users/{userId}:
    get:
      summary: "ユーザー情報をIDで取得する"
      parameters:
        - name: userId
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: "成功"
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserResponse'
# ... 他のAPI定義やcomponentsなどが続く ...
```

#### **Controllerの実装（詳細コメント付き）**

`Controller`のメソッドには、「外部との契約部分」と「内部の実行ロジック」が同居しています。上記のOpenAPI仕様は、以下のコードの`(A)外部との契約`部分に相当します。

```python
# user_controller.py

from .dto import UserResponseDto
from .use_cases import GetUserUseCase

class UserController:
    def __init__(self, get_user_use_case: GetUserUseCase):
        self.get_user_use_case = get_user_use_case

    # --------------------------------------------------------------------------
    # ▼▼▼▼▼▼▼▼▼▼▼▼▼ このメソッド全体が OpenAPI の一つの操作に対応 ▼▼▼▼▼▼▼▼▼▼▼▼▼
    # --------------------------------------------------------------------------

    # --- (A) 外部との契約を定義する部分 ---
    # デコレータと関数シグネチャ（引数や戻り値の型）が、OpenAPIに記述される
    # 「APIの仕様」に直接対応します。

    # @router.get('/users/{userId}')  <-- 契約(1): 「GET /users/{userId}」というリクエストを処理すると宣言
    async def get_user(self, user_id: str) -> UserResponseDto:  # <-- 契約(2): 文字列のIDを受け取り、UserResponseDtoの形で返すと約束

        # --- (B) 契約を履行するための内部的な実行ロジック ---
        # この関数の中身は、外部の利用者からは見えません。
        # ここで定義された「契約(A)」をどうやって実現するかを記述します。

        # 内部処理1: Use Caseに渡すための入力データを作成
        input_data = {"id": user_id}

        # 内部処理2: ビジネスロジック層（Use Case）を呼び出して、結果を取得
        user_entity = await self.get_user_use_case.execute(input_data)

        # 内部処理3 -> 外部への出力:
        # 内部処理で得られた結果を、「契約(A)」で約束した `UserResponseDto` の
        # 形式に変換して返却します。
        return UserResponseDto(id=user_entity.id, name=user_entity.name)
```

このように、`Controller`は外部との厳密な「契約」をコードで表現しつつ、その契約を達成するための「実行」ロジックを持つ、サービスの内と外を繋ぐ非常に重要な役割を担っているのです。

