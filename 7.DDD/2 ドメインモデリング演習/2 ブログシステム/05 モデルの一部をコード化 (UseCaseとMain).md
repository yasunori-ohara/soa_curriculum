# 05 モデルの一部をコード化 (UseCaseとMain)

# ステップ５：モデルの一部をコード化 (UseCaseとMain) ⚙️🚀

これまでのステップで定義・実装したドメインモデル（集約、値オブジェクト）とリポジトリを使って、いよいよ「記事を公開する」「コメントを投稿する」という具体的な**UseCase**を実装し、それを[**main.py**](http://main.py/)から呼び出して実行してみましょう。

---

## ⚙️ 1. UseCase の実装 - ビジネスフローの実現

📝 **課題**:

1. 「指定された記事IDの記事を公開状態にする」というフローが必要です。
2. 「指定された記事IDに対して、投稿者名とコメント本文を受け取り、新しいコメントを作成して保存する」というフローが必要です。
これらはアプリケーション固有の要求であり、ドメインオブジェクト（集約）やリポジトリを利用して実現する必要があります。

💡 **解決策**: これらのフローの責任を持つ「UseCase」を実装します。これはCA（クリーンアーキテクチャ）の UseCase と同じ役割です。

**ファイル:** `application/use_cases.py` (新規作成または追記)

```python
# application/use_cases.py
from application.boundaries import ArticleRepositoryInterface, CommentRepositoryInterface
from domain.aggregates import Article, Comment # 利用・生成する集約
from domain.value_objects import Title, Body, PublishStatus, CommenterName # 値オブジェクトを利用
from datetime import datetime
from typing import Optional, List # 戻り値・引数用

# -----------------------------------------------------------------------------
# Publish Article Use Case
# - CAの Use Cases 層と同じ役割
# - アプリケーション固有のビジネスフローを担当
# -----------------------------------------------------------------------------
class PublishArticleUseCase:
    """記事を公開するユースケース"""
    def __init__(self, article_repository: ArticleRepositoryInterface): # 必要な依存(Repo I/F)を注入
        self._article_repository = article_repository

    def handle(self, article_id: str):
        """
        ユースケースを実行する。
        Args:
            article_id: 公開する記事のID
        Raises:
            ValueError: 記事が見つからない場合など
        """
        print(f"[UseCase] Attempting to publish article {article_id}")

        # --- トランザクション管理の開始 (概念) ---
        try:
            # 1. リポジトリを使って集約(Article)を取得
            article = self._article_repository.find_by_id(article_id)
            if not article:
                raise ValueError(f"Article {article_id} not found.")

            # 2. 集約ルート(Article)のメソッドを呼び出してビジネスロジック(公開)を実行
            #    (状態変更や公開日時の設定は集約内で行われる)
            article.publish()

            # 3. 変更された集約をリポジトリを使って保存
            self._article_repository.save(article)

            # --- トランザクションのコミット (概念) ---
            print(f"[UseCase] Article {article_id} published successfully.")

        except Exception as e:
            # --- トランザクションのロールバック (概念) ---
            print(f"[UseCase] Failed to publish article {article_id}: {e}")
            raise # エラーを上位に伝える

# -----------------------------------------------------------------------------
# Add Comment Use Case
# - CAの Use Cases 層と同じ役割
# -----------------------------------------------------------------------------
class AddCommentUseCase:
    """記事にコメントを追加するユースケース"""
    def __init__(self,
                 comment_repository: CommentRepositoryInterface,
                 article_repository: ArticleRepositoryInterface): # 記事存在確認用
        self._comment_repository = comment_repository
        self._article_repository = article_repository # 記事の存在確認に使う

    def handle(self, article_id: str, commenter_name_str: str, body_str: str) -> Optional[str]:
        """
        ユースケースを実行する。
        Args:
            article_id: コメント対象の記事ID
            commenter_name_str: コメント投稿者名(文字列)
            body_str: コメント本文(文字列)
        Returns:
            成功した場合は生成されたコメントID、失敗した場合はNone (または例外送出)
        """
        print(f"[UseCase] Attempting to add comment to article {article_id}")

        # 1. 入力から値オブジェクトを生成 (形式検証)
        try:
            commenter_name = CommenterName(commenter_name_str)
            body = Body(body_str)
        except ValueError as e:
            print(f"[UseCase] Invalid input for comment: {e}")
            return None # または raise

        # --- トランザクション管理の開始 (概念) ---
        try:
            # 2. (任意) コメント対象の記事が存在するか確認
            #    コメント投稿時に厳密な記事チェックが必要な場合
            article = self._article_repository.find_by_id(article_id)
            if not article:
                # 存在しない記事へのコメントは許可しない
                raise ValueError(f"Cannot add comment: Article {article_id} not found.")
            # (さらに、記事が公開済み(Published)の場合のみコメント可、などのルールも追加可能)
            # if article.status != PublishStatus.PUBLISHED:
            #    raise ValueError(f"Cannot add comment: Article {article_id} is not published.")

            # 3. 新しいコメント(Comment)集約を生成
            #    (今回はシンプルな生成なのでFactoryは使わない)
            print("[UseCase] Creating new comment...")
            new_comment = Comment(
                article_id=article_id,
                commenter_name=commenter_name,
                body=body
                # comment_id は Comment クラス内で自動生成
            )

            # 4. 生成したコメント集約をリポジトリを使って保存
            print(f"[UseCase] Saving new comment {new_comment.comment_id}...")
            self._comment_repository.save(new_comment)

            # --- トランザクションのコミット (概念) ---
            print(f"[UseCase] Comment added successfully to article {article_id}. Comment ID: {new_comment.comment_id}")

            # 成功した場合はコメントIDを返す
            return new_comment.comment_id

        except Exception as e:
            # --- トランザクションのロールバック (概念) ---
            print(f"[UseCase] Error adding comment: {e}")
            # (エラーをログに記録するなど)
            return None # または raise

```

✅ **このステップのポイント**:

- 各 UseCase は CA と同じく、特定の**ビジネスフロー**を実現する責務を持ちます。
- **リポジトリインターフェース**を通じて集約を取得・保存します。
- **ドメインオブジェクト**（`Article.publish()`, `Comment` のコンストラクタ, `CommenterName` や `Body` の生成時の検証）を利用して、ドメインルールを実行・保証します。
- `AddCommentUseCase` では、関連する別の集約 (`Article`) の状態を確認するために、`ArticleRepository` も利用しました。

---

## 🚀 2. 実行ファイル ([main.py](http://main.py/)) の作成 - 全体の組み立てと実行

📝 **課題**: 作成した UseCase を実行し、システム全体（今回はインメモリ）が動作することを確認する「起点」が必要です。

💡 **解決策**: `main.py` で、具体的なリポジトリ実装を生成し、それらを UseCase に注入（DI）して、UseCase を呼び出します。これは CA の `main.py` と同じ役割です。

**ファイル:** `main.py` (プロジェクトルート、または `app/` など)

```python
# main.py
from adapters.repositories import InMemoryArticleRepository, InMemoryCommentRepository # 👈 実装
from application.use_cases import PublishArticleUseCase, AddCommentUseCase           # 👈 UseCase
from domain.aggregates import Article, Comment                                      # 初期データ用
from domain.value_objects import Title, Body, PublishStatus, CommenterName          # 初期データ用
from datetime import datetime                                                       # 初期データ用

# -----------------------------------------------------------------------------
# Main Application / Composition Root
# - CAの最も外側の層 (Frameworks & Drivers) と同じ役割
# - 依存関係の解決 (DI) とアプリケーションの起動を行う
# -----------------------------------------------------------------------------
if __name__ == "__main__":
    print("--- Blog System Example ---")

    # --- 依存関係の解決 (Dependency Injection) ---
    print("\\n--- Wiring dependencies (DI) ---")
    article_repo = InMemoryArticleRepository()
    comment_repo = InMemoryCommentRepository()
    publish_article_use_case = PublishArticleUseCase(article_repo)
    add_comment_use_case = AddCommentUseCase(comment_repo, article_repo) # 記事Repoも渡す
    print("--- Dependencies wired successfully ---")

    # --- 初期データ投入 (記事作成) ---
    print("\\n--- Setting up initial data (Article) ---")
    try:
        author_id = "user-1"
        article_title = Title("My First Blog Post")
        article_body = Body("This is the content of the post.")
        article = Article(author_id=author_id, title=article_title, body=article_body)
        article_repo.save(article)
        article_id = article.article_id
        print(f"Article '{article_title.value}' created with ID: {article_id}, Status: {article.status.name}") # -> DRAFT
    except ValueError as e:
        print(f"Error setting up initial data: {e}")
        exit()

    # --- UseCase実行1: 記事を公開 ---
    print("\\n--- Executing UseCase: Publish Article ---")
    try:
        publish_article_use_case.handle(article_id)
    except Exception as e:
        print(f"Failed to publish article: {e}")

    # --- UseCase実行2: コメントを追加 (成功例) ---
    print("\\n--- Executing UseCase: Add Comment (Success) ---")
    try:
        commenter = "Reader 1"
        comment_body = "Great post!"
        comment_id_1 = add_comment_use_case.handle(article_id, commenter, comment_body)
        if comment_id_1: print(f"Comment added successfully: {comment_id_1}")
    except Exception as e:
        print(f"Failed to add comment: {e}")

    # --- UseCase実行3: コメントを追加 (失敗例: 本文が短すぎる) ---
    print("\\n--- Executing UseCase: Add Comment (Invalid Body) ---")
    try:
        commenter = "Reader 2"
        comment_body = "" # Body値オブジェクトの検証でエラーになるはず
        comment_id_2 = add_comment_use_case.handle(article_id, commenter, comment_body)
        if comment_id_2: print(f"Comment added successfully: {comment_id_2}")
    except Exception as e: # 値オブジェクト生成時のValueErrorがUseCaseまで伝わる
        print(f"Failed to add comment as expected: {e}") # -> Body must be at least 1 character(s).

    # --- 最終結果の確認 (リポジトリから取得) ---
    print("\\n--- Verifying Final State ---")
    final_article = article_repo.find_by_id(article_id)
    if final_article:
        print(f"Article Status: {final_article.status.name}") # -> PUBLISHED
        print(f"Published At: {final_article.published_at}")

    final_comments = comment_repo.find_by_article_id(article_id)
    print(f"Number of comments for article {article_id}: {len(final_comments)}") # -> 1
    if final_comments:
        print(f" - Comment 1 ID: {final_comments[0].comment_id}")
        print(f"   Commenter: {final_comments[0].commenter_name.value}") # -> Reader 1

```

✅ **このステップのポイント**:

- `main.py` が依存関係を組み立て、`PublishArticleUseCase` と `AddCommentUseCase` を呼び出して動作を確認しました。
- UseCase がドメインルール（記事の公開状態変更、値オブジェクトの検証）を正しく実行・利用できることを確認しました。
- 値オブジェクトの検証エラーが UseCase まで伝播し、ハンドリングできることも示されました。

---

## 📝 この演習のまとめ

この演習では、「ブログシステム」のシンプルなシナリオを通じて、DDDの戦術的パターンを適用しました。

1. **ユビキタス言語**: 「記事」「コメント」「公開状態」などの言葉を特定しました。
2. **集約**: 「`Article`」と「`Comment`」を、それぞれのライフサイクルと一貫性を持つ独立した単位として定義しました。
3. **値オブジェクト**: 「`Title`」「`Body`」「`PublishStatus`」「`CommenterName`」などを定義し、ドメイン概念を豊かに表現し、不変性と自己検証ルールを適用しました。
4. **リポジトリ**: 各集約の永続化を抽象化しました。
5. **UseCase**: ドメインモデルとリポジトリを利用して、「記事公開」「コメント追加」のビジネスフローを実装しました。

特に、記事とコメントを集約レベルで分離したこと、そしてタイトルや本文などを値オブジェクトとしてモデル化したことで、関心が分離され、変更に強く、ドメインルールがコードで表現された設計の基礎が築けたかと思います。

---

## ✅ ドメインモデリング演習 (2/5) 完了！

これで、「ブログシステム」のモデリング演習は一区切りです。
次は、3つ目の題材「**オンライン書店**」に進み、複数のコンテキスト（カタログ、注文など）を少し意識しながらモデリングしてみましょうか？