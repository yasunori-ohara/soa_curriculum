## APIコントラクト設計

### 👏 **設計するAPIの要件**

  * **エンドポイント**: `GET /articles/{articleId}/details`
      * 備考: 記事の基本的な情報を取得する`/articles/{articleId}`とは別に、関連情報も含めて取得するための専用エンドポイントを用意します。
  * **機能**: 指定されたIDの記事情報に、その著者名とコメント一覧を付加して返す。
  * **入力**: `articleId`（パスパラメータ）。
  * **出力**: 成功した場合は、記事、著者名、コメントが結合されたデータ構造を返す。

-----

### 👏 **OpenAPI仕様書の作成（記事サービスに追加）**

`article-service.yaml`に、新しいエンドポイントの定義を追加します。

```yaml
# openapi: 3.0.3
info:
  title: 記事サービス API
  version: 1.0.0

paths:
  # ... (既存の /articles の定義は省略) ...

  # ▼▼▼▼▼ 今回新しく設計するエンドポイント ▼▼▼▼▼
  /articles/{articleId}/details:
    get:
      summary: "記事の詳細情報（著者名、コメント含む）を取得する"
      operationId: "getArticleDetails"
      parameters:
        - name: articleId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      
      responses:
        # 成功した場合（200 OK）
        '200':
          description: "成功"
          content:
            application/json:
              schema:
                # 下で定義するArticleDetailという特別なデータ構造を参照
                $ref: '#/components/schemas/ArticleDetail'
        # 記事が見つからない場合
        '404':
          description: "指定された記事が見つかりません"

components:
  schemas:
    # ... (既存のArticleなどの定義は省略) ...

    # ▼▼▼▼▼ 記事詳細情報のための特別なデータ構造を定義 ▼▼▼▼▼
    
    # コメントの構造
    Comment:
      type: object
      properties:
        commentId:
          type: string
        content:
          type: string
        authorId:
          type: string
          format: uuid
          
    # 最終的に返す、結合されたデータの構造
    ArticleDetail:
      type: object
      properties:
        articleId:
          type: string
          format: uuid
        title:
          type: string
        content:
          type: string
        # ユーザーサービスから取得した著者名を格納する
        authorName:
          type: string
          example: "山田 太郎"
        # コメントサービスから取得したコメント一覧を格納する
        comments:
          type: array
          items:
            $ref: '#/components/schemas/Comment'
```

-----

### 👏 **設計のポイント**

  * **特別なデータ構造（DTO）の定義**:
    レスポンスとして返す`ArticleDetail`は、「記事サービス」のドメインエンティティである`Article`とは異なる、この**エンドポイント専用のデータ構造**です。他のサービスから取得した情報（`authorName`, `comments`）が含まれていることが明確に分かります。

  * **オーケストレーションの責務**:
    このAPIエンドポイントを設けることで、「記事サービス」は単に自身のデータを返すだけでなく、他のサービスに問い合わせて情報を集約する\*\*「まとめ役（オーケストレーター）」\*\*としての責務を負うことが、契約書上で明確になりました。

  * **クライアントの利便性**:
    このAPIを利用するクライアントは、一度リクエストを送るだけで、画面表示に必要な全ての情報が手に入ります。裏側で複数のサービスが動いているという複雑さは、このAPIによって隠蔽されます。

-----

以上で、サービス間連携のための設計が完了しました。
