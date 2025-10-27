# 01 Entities

# Entity
### `core/domain/todo.py`

## 🧩 このクラスの役割

このファイル `core/domain/todo.py` は、前章で紹介した `core/domain/` ディレクトリの中にある、**ビジネスルールの核となるクラス**です。

**`Todo`クラス**は、アーキテクチャの最も内側に位置する**Entity**です。アプリケーション全体の**核となるビジネスルール**と*データをカプセル化（内包）*します。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

これは特定のアプリケーションのためではなく、このシステムのビジネスドメイン、つまり「TODO管理」そのものにとって**普遍的なルールとデータ**を表現します。

💡 **平たく言えば、「このビジネスにおける『TODO』とは何か、そして『TODO』が従うべき『物理法則』は何か」を定義する場所です。**

---

## ⚙️ このクラスにあるべき処理

`Todo` Entityに含めるべきなのは、**アプリケーションの都合に依存しない、純粋なビジネスのルール**です。

✅ **含めるべき処理の例:**

* **状態遷移のロジック**

  * `complete()`メソッド。タスクを「未完了」から「完了」へ状態を変えるのは、`Todo`という概念そのものが持つルールです。
* **不変条件の維持（自己バリデーション）**

  * `Todo`のタイトルは空文字列であってはならない、といったチェックを`__init__`内で行うこと。
* **自身のデータのみで行える計算**

  * もし`Todo`が期限（`deadline`）を持っていたら、残り日数を計算する`get_days_remaining()`のようなメソッド。

❌ **含めてはいけない処理の例:**

* **データベースへの保存処理**

  * `save_to_database()`のようなメソッドは**絶対に含めません**。Entityは自分がどう保存されるかを知るべきではありません。
* **UIに関する処理**

  * `display_on_screen()`のような画面表示ロジックは含めません。
* **外部システムとの連携**

  * `send_notification_email()`のような処理は、アプリケーション固有の都合であり、Entityの責務ではありません（これらはUse Case層が担当します）。

---

## 🔍 ソースコードの詳細解説

```python
# --------------------------------------------------------------------
# File: core/domain/todo.py
# Layer: Entity（クリーンアーキテクチャの最内層）
#
# 目的:
#   - このファイルは「TODO」というビジネス上の概念そのものを表現します。
#   - DB保存方法やUI表示方法など、アプリ固有の都合からは完全に独立しています。
#   - ここに書くのは、純粋なビジネスルールと、それに必要な最小限のデータだけです。
#
# C言語の感覚に近い説明:
#   - クラスは「構造体＋メソッド（関数）」のようなものです。
#   - self は「thisポインタ」に相当し、現在のインスタンス（自分自身）を指します。
#   - 例外（raise）は「エラーコードを返す」代わりに、呼び出し側へエラーを通知する仕組みです。
# --------------------------------------------------------------------

class Todo:
    """Todoエンティティ（ビジネスルールの核）

    - このクラスは「1件のタスク」を表します。
    - 属性（フィールド）:
        id        : 一意な識別子（int）
        title     : タスクのタイトル（str、空文字は禁止）
        completed : 完了フラグ（bool）
    - メソッド（関数）:
        complete(): タスクを完了状態にする（ただし二重完了はエラー）
    """

    def __init__(self, id: int, title: str, completed: bool = False):
        """
        コンストラクタ（初期化処理）

        不変条件（Invariant）をここで守る:
        - id は正の整数であること
        - title は空文字列でないこと（空白のみも禁止）
        """
        if id <= 0:
            raise ValueError("Todo ID must be a positive integer.")
        if not title.strip():
            raise ValueError("Todo title cannot be empty.")

        self.id = id
        self.title = title
        self.completed = completed

    def complete(self):
        """
        ドメイン固有のビジネスルール:
        - タスクを未完了（False）から完了（True）へ状態遷移する。
        - すでに完了済み（True）の場合は、二重完了を禁止（エラー）とする。
        """
        if self.completed:
            raise RuntimeError("Todo is already completed.")
        self.completed = True

    def __repr__(self):
        """
        __repr__ は「開発者向けの文字列表現」
        - Cでいう printf で構造体内容をフォーマット出力するイメージ。
        """
        return f"Todo(id={self.id}, title='{self.title}', completed={self.completed})"

    def __str__(self):
        """
        __str__ は「人間向けの見やすい文字列表現」
        - UI層やコンソールへの表示で使うことを想定。
        """
        status = "✔ Completed" if self.completed else "✗ Pending"
        return f"[{status}] {self.title} (ID: {self.id})"
```

* 🏗 `__init__`は、このEntityがどのようなデータで構成されるかという「構造」を定義しています。
* 🔐 `complete()`は、このEntityがどのような振る舞いをするかという「ルール」を定義しています。

このファイルは、外部のライブラリや他のどのレイヤーのファイルも一切`import`していません。Pythonの標準機能だけで完結しており、**完全に自己完結した独立した存在**です。

---

## ✅ この段階でユニットテストをすることができます

Entityは外部に依存しないため、この時点で単体テストが可能です。

以下は、Python初心者でも理解しやすい「ユニットテスト感覚」のサンプルです。

```python
if __name__ == "__main__":
    # 正常系: タスクを作成し、完了状態に変更する
    todo = Todo(id=1, title="ドキュメントを書く")
    print(repr(todo))  # 開発者向け表示
    print(str(todo))   # ユーザー向け表示

    todo.complete()
    print(str(todo))   # 完了状態に変化

    # 異常系: 二重完了は RuntimeError を投げる
    try:
        todo.complete()
    except RuntimeError as e:
        print(f"[期待通りのエラー] {e}")

    # 異常系: 不正な入力（id <= 0）
    try:
        invalid = Todo(id=0, title="不正なID")
    except ValueError as e:
        print(f"[期待通りのエラー] {e}")

    # 異常系: 空タイトル
    try:
        invalid = Todo(id=2, title="   ")
    except ValueError as e:
        print(f"[期待通りのエラー] {e}")
```

---

## 🛡 このクラスの鉄則

Entity層を守るべき最も重要なルールは、ただ一つです。

> 何物にも依存するな (Depend on Nothing)

* **最も安定した、変更の少ないコードであるべきです。**
  UIフレームワークやデータベース技術は数年で流行り廃りがありますが、ビジネスの核となるルール（例：「タスクは完了できる」）はそう簡単には変わりません。
* **外側のレイヤー（Use Cases, Adaptersなど）について一切知ってはいけません。**
  `import`文で外側のレイヤーのファイルを読み込むことは固く禁じられています。
* **Entityは、アプリケーションの他の部分から利用されるだけの存在です。**
  Entityが他のクラスを積極的に呼び出しにいくことはありません。

この鉄則を守ることで、システムの基盤であるビジネスルールが、UIの変更やデータベースの移行といった技術的な変更から完全に隔離され、**メンテナンス性が高く、テストが容易で、変化に強いソフトウェア**を構築することができるのです。
