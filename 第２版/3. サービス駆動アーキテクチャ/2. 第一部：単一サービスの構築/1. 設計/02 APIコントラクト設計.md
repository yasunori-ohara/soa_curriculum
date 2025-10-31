## APIコントラクト設計

### 🛠️ はじめに：実装のための詳細な「設計図」を描く

序章では、システム全体の青写真として、どのサービスが存在し、それぞれが何に責任を持つかを定義しました。このステップでは、その中の一つである「記事サービス」**に焦点を当て、それを実装するための、より詳細な**「設計図」を作成します。

SOAの世界では、サービス同士は**APIという「契約」を通じてのみ対話します。したがって、サービスの「外部に公開される顔」であるAPIを、業界標準のOpenAPI**という共通言語で厳密に定義する訓練を、この最初の実装から行います。

今回は、記事サービスの数ある機能の中から、最も中核的で、かつサービス単体で完結する「新しい記事を投稿する」機能を取り上げます。この機能に集中することで、私たちはまずクリーンアーキテクチャの基本パターンを、他の複雑な要素に惑わされずに習得することができます。

それでは、この方針に沿って、具体的な**APIコントラクト設計**に進んでいきましょう。

-----

### 🛠️ APIコントラクト設計

#### **設計するAPIの要件**

  * **エンドポイント**: `POST /articles`
  * **機能**: 新しい記事を作成する。
  * **認証**: ログインしているユーザーのみ投稿可能とする。
  * **入力**: 記事の`タイトル`と`本文`。
  * **出力**: 成功した場合、作成された記事の情報を返す。

#### **OpenAPI仕様書の作成（article-service.yaml）**

```yaml
# openapi: 3.0.3
info:
  title: 記事サービス API
  version: 1.0.0

paths:
  /articles:
    post:
      summary: "新しい記事を投稿する"
      description: "認証されたユーザーとして、新しい記事を作成します。"
      operationId: "createArticle"
      security:
        - bearerAuth: []
      requestBody:
        description: "作成する記事のデータ"
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateArticleRequest'
      responses:
        '201':
          description: "記事が正常に作成された"
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Article'
        '400':
          description: "リクエストが無効です（例：タイトルが空）"
        '401':
          description: "認証されていません"

components:
  schemas:
    CreateArticleRequest:
      type: object
      properties:
        title:
          type: string
          example: "私の最初の記事"
        content:
          type: string
          example: "これは記事の本文です。"
      required:
        - title
        - content
    Article:
      type: object
      properties:
        articleId:
          type: string
          format: uuid
        authorId:
          type: string
          format: uuid
        title:
          type: string
        content:
          type: string
        createdAt:
          type: string
          format: date-time
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
```

-----

### 🛠️ 設計図の読み解き方：重要なポイント

この仕様書は、単なる機能の羅列ではありません。高品質なサービスに不可欠な、いくつかの重要な設計思想が込められています。

#### **1. 関心の分離と再利用 (`components`と`$ref`)**

`CreateArticleRequest`（リクエスト用）と`Article`（レスポンス用）のように、データの役割を明確に分離しています。`articleId`や`authorId`はサーバー側で決定するため、リクエストには含まれません。
また、これらのデータ構造は`components`**セクションに「部品」として一元管理され、**`$ref`を通じて参照されます。これにより、設計の重複をなくし、効率的で間違いのない仕様書を作成できます。

#### **2. 厳密なレスポンス定義（HTTPステータスコード）**

`responses`セクションでは、成功と失敗のケースを明確に定義しています。

  * **`201 Created`**: 単なる成功(`200 OK`)ではなく、「リソースが**新規作成**された」ことを示す、より具体的なステータスコードです。
  * **`400 Bad Request`**: クライアント側のリクエスト内容自体に問題がある（例: `title`が空）ことを示します。
  * **`401 Unauthorized`**: 認証に失敗したことを示します。
    これらのステータスコードを正しく使い分けることで、クライアントはエラーの原因を正確に把握できます。

#### **3. 認証の仕組み (`security`と`securitySchemes`)**

このAPIを利用するには、まず別の場所（例: ユーザーサービス）で認証を済ませ、JWT (JSON Web Token)と呼ばれるアクセストークンを取得する必要があります。そして、`Authorization: Bearer <トークン>`という形式でHTTPヘッダーに含めてリクエストを送る、という具体的な認証フローまでが、この仕様書で定義されています。

-----

以上で、単一サービスを実装するための詳細な設計図が完成しました。
