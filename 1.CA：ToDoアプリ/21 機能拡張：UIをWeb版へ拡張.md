# 21 機能拡張：UIをWeb版へ拡張

# 🌐 Web版 View : `ui/view_web.py` - `WebView`（Flask）

## 🧭 このクラスの役割

`WebView` は、**ユーザーがブラウザを通じて操作するWebインターフェース**を提供するクラスです。

CLI版やGUI版と同様に、責務は以下の2つに集約されます：

1. **表示（Output）**`Presenter` によって更新された `ViewModel` の内容を、HTMLテンプレートに描画する。
2. **入力（Input）**
ユーザーがフォームに入力した内容を `Controller` に渡す。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## 🔁 なぜWeb版への差し替えが「簡単」なのか？

> 💡 クリーンアーキテクチャを採用したからこそ、UIの差し替えが容易になっています。
> 
- `View`は最外層に位置し、**ControllerとViewModelにのみ依存**しています。
- `Presenter`や`UseCase`は、Viewの具体的な実装（CLI・GUI・Web）を一切知りません。
- そのため、CLI版の `ConsoleView` を `WebView` に置き換えるだけで、**他の層には一切変更が不要**です。
- これは、UI技術の変更に強い設計が実現されている証拠です。

---

## ✅ このクラスにあるべき処理

⭕️ **含めるべき処理の例**

- Flaskを使ったルーティングとフォーム処理
- `ViewModel` の内容をHTMLに反映する処理
- ユーザー操作（POSTリクエスト）に応じて `Controller` を呼び出す処理

❌ **含めてはいけない処理の例**

- ビジネスロジック（Use Caseの責務）
- 表示用メッセージの整形（Presenterの責務）
- データ保存や取得（Repositoryの責務）

---

## 🔍 ソースコード（コメント充実）

```python
# --------------------------------------------------------------------
# View に相当（Web版）
#
# 責務：ユーザーとの直接的なやり取り（入力受付・画面表示）を担当する。
# 依存：<DS> ViewModel を知っており、Controller を呼び出す。
#
# 位置づけ（クラス図／同心円図）：
# - クラス図：Interface Adapters（View）
# - 同心円図：Framework & Drivers（最外層）
#
# C言語との比較:
#   - Webは「イベント駆動型のサーバー処理」に近く、割り込み＋状態保持の構造。
# --------------------------------------------------------------------

from flask import Flask, request, render_template_string
from interface_adapters.controller import AddTodoController
from data_structures import TodoViewModel

class WebView:
    """
    Web版のView。Flaskを使ってWebサーバーを構築し、
    ユーザー入力と画面表示を担当する。
    """

    def __init__(self, controller: AddTodoController, view_model: TodoViewModel):
        self._controller = controller
        self._view_model = view_model
        self.app = Flask(__name__)
        self._setup_routes()

    def _setup_routes(self):
        """
        [ルーティング定義]
        GET: フォーム表示
        POST: 入力処理 → Controller呼び出し → ViewModel更新
        """
        @self.app.route("/", methods=["GET", "POST"])
        def index():
            if request.method == "POST":
                title = request.form.get("title", "")
                self._controller.add_todo(title)

            return render_template_string(self._html_template(), display=self._view_model.display_text)

    def _html_template(self):
        """
        [HTMLテンプレート]
        ViewModelの内容を {{ display }} に埋め込む。
        """
        return """
        <!DOCTYPE html>
        <html>
        <head><title>TODOアプリ（Web版）</title></head>
        <body>
            <h1>TODO追加</h1>
            <form method="POST">
                <input type="text" name="title" placeholder="TODOのタイトル" required>
                <button type="submit">追加</button>
            </form>
            <p style="color:green;">{{ display }}</p>
        </body>
        </html>
        """

    def run(self):
        """
        [起動処理]
        Flaskアプリケーションを起動する。
        """
        self.app.run(debug=True)

```

---

## 🛡 このクラスの鉄則

> 見せて、渡して、黙って待て。
> 
- Viewは「表示」と「入力の橋渡し」に徹する
- ビジネスロジックやデータ整形は一切行わない
- Web技術に依存するが、責務はCLI版・GUI版と同じ

---

## 🔧 `main.py` での差し替え例（Web版）

```python
# --- Web版Viewを使う場合 ---
from ui.view_web import WebView

# ...（Presenter, Controller, ViewModelなどの生成は同じ）

view = WebView(controller, view_model)
view.run()

```