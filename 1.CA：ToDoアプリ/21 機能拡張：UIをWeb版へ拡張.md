# 21 機能拡張：UIをWeb版へ拡張

# 🌐 Web版 View
`interface_adapters/views/view_web.py` - `WebView`（Flask版）

---

## 🧭 このクラスの役割

`WebView` は、**ユーザーがブラウザを通じて操作するWebインターフェース**を提供するクラスです。

CLI版 (`ConsoleView`) や GUI版 (`GuiView`) と同様に、責務は次の2つだけです。

1. **表示（Output）**
   `Presenter` が更新した `TodoViewModel` の内容を、HTMLとしてブラウザに表示する。
2. **入力（Input）**
   ユーザーがフォームから送信したデータ（新しいTODOのタイトル）を `Controller` に渡す。

`WebView` は「HTTPリクエストを受ける窓口」「HTMLを返す窓口」に徹します。
ビジネスロジックの中身・エンティティのルール・保存の仕方などは一切扱いません。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## 🔁 なぜWeb版への差し替えが「簡単」なのか？

> 💡 クリーンアーキテクチャでは、UIの実装は最外層（Frameworks & Drivers）に置かれます。

* `View` は **Controller** と **ViewModel** にだけ依存します。
* `UseCase` や `Presenter` は、ViewがCLIかGUIかWebかを知りません。
* つまり、CLI版の `ConsoleView` を Web版の `WebView` に置き換えても、
  **UseCase / Presenter / Repository / Entity は1行も変える必要がありません。**

これは、見た目・入出力デバイス（コンソール / デスクトップGUI / ブラウザ）が変わっても、
ビジネスロジックを巻き込まずに差し替えできる設計になっていることの証拠です。

---

## ✅ このクラスにあるべき処理

⭕️ 含めるべき処理の例

* Flaskを使ったルーティングの定義（どのURLにどう応答するか）
* フォームから送られた内容を受け取り、Controllerに渡す
* `TodoViewModel` の内容をHTMLに組み込んで返す

❌ 含めてはいけない処理の例

* ビジネスロジックの判断（UseCaseの責務）
* 表示用メッセージの整形（Presenterの責務）
* データの保存・検索（Repositoryの責務）
* ID採番、DB操作などのインフラ詳細

---

## 📂 ファイルの配置

Web版Viewは、CLI版・GUI版と並列に配置します。

```text
├─ interface_adapters/
│   ├─ presenters/
│   │   └─ todo_presenter.py
│   ├─ controllers/
│   │   └─ todo_controller.py
│   └─ views/
│       ├─ view_console.py      # CLI版のView
│       ├─ view_gui.py          # GUI版のView
│       └─ view_web.py          # Web版のView（本章の主役）
```

この構成は「UIは差し替え可能な最外層」という考えをそのままフォルダで表現しています。

---

## 🔍 ソースコード（正式版）

```python
# --------------------------------------------------------------------
# File: interface_adapters/views/view_web.py
# Layer: Interface Adapters（View - Web版）
#
# 責務：
#   - HTTP経由でユーザー入力を受け取り（フォーム送信）、
#     ControllerにTodo追加のリクエストを渡す。
#   - Presenterが更新したViewModelをHTMLとして返す。
#
# 依存：
#   - <I> Controller（ユーザー操作の伝達先）
#   - <DS> TodoViewModel（画面表示のための状態）
#
# 同心円図での位置：
#   - Frameworks & Drivers（最外側）
#   - Flaskなど具体的なWebフレームワークはここに閉じ込める。
#
# C言語イメージ：
#   - Webサーバーのループが「割り込みハンドラ」役。
#   - それぞれのリクエストに対する処理がコールバックとして動く。
# --------------------------------------------------------------------

from flask import Flask, request, render_template_string
from interface_adapters.controllers.todo_controller import TodoController
from core.usecase.boundary.dto import TodoViewModel


class WebView:
    """
    Web版のView。
    Flaskアプリケーションを内部に持ち、
    HTTPリクエストをControllerとViewModelに橋渡しする。
    """

    def __init__(self, controller: TodoController, view_model: TodoViewModel):
        """
        [依存関係]
        - controller: フォーム入力をUseCaseまで届ける橋渡し役
        - view_model: Presenterが更新する表示用データ
        """
        self._controller = controller
        self._view_model = view_model

        # Flaskアプリ本体
        self.app = Flask(__name__)

        # ルーティング定義
        self._setup_routes()

    def _setup_routes(self):
        """
        画面表示とフォーム送信を1つのエンドポイント（"/"）で処理する。

        GET:
            現在のViewModelの状態をブラウザに表示する。
        POST:
            フォームから送られた文字列をControllerに渡し、
            UseCaseを実行させる。
            その結果PresenterがViewModelを更新するので、
            その最新状態を改めて表示する。
        """

        @self.app.route("/", methods=["GET", "POST"])
        def index():
            if request.method == "POST":
                title = request.form.get("title", "")
                self._controller.add_todo(title)

            # ViewModel.display_text をHTMLに埋め込んで返す
            return render_template_string(
                self._html_template(),
                display=self._view_model.display_text,
            )

    def _html_template(self) -> str:
        """
        画面のHTMLテンプレート。
        - {{ display }} に ViewModel.display_text が埋め込まれる。
        - できるだけロジックレスに保つことが重要。
        """
        return """
        <!DOCTYPE html>
        <html>
        <head>
            <meta charset="utf-8"/>
            <title>TODOアプリ（Web版）</title>
        </head>
        <body style="font-family: sans-serif; line-height: 1.5; max-width: 480px;">
            <h1>TODO追加</h1>

            <form method="POST" style="margin-bottom:1rem;">
                <input
                    type="text"
                    name="title"
                    placeholder="TODOのタイトル"
                    required
                    style="padding:0.5rem; width:70%;"
                >
                <button type="submit" style="padding:0.5rem 1rem;">追加</button>
            </form>

            <p style="color: green; white-space: pre-wrap;">{{ display }}</p>

            <hr/>
            <small style="color:#777;">
                この画面はWeb版Viewです。ビジネスロジックには一切触っていません。
            </small>
        </body>
        </html>
        """

    def run(self):
        """
        Flaskアプリケーションを起動する。
        デバッグモードは学習用のため True にしている。
        本番運用時には別のWSGIサーバー等を使う想定。
        """
        self.app.run(debug=True)
```

---

## 🛡 このクラスの鉄則

> 見せて、渡して、黙って待て。

* Viewは「Controllerに依頼する」「ViewModelの中身を表示する」だけ
* ビジネスロジックや整形ロジックは一切持たない
* Webという技術詳細（Flaskなど）はここ（最外層）に閉じ込める

GUI版・CLI版とまったく同じ考え方をWebに適用しているだけです。

---

## 🔧 `main.py` での差し替え例（Web版）

`main.py` はアプリ全体を「配線して起動する場所」でした。
ConsoleView や GuiView の代わりに、WebView を使うこともできます。

```python
# main.py (Web版で動かしたい場合の例)

from core.usecase.boundary.dto import TodoViewModel
from core.usecase.interactor.create_todo import TodoUseCase
from interface_adapters.presenters.todo_presenter import TodoPresenter
from interface_adapters.controllers.todo_controller import TodoController
from interface_adapters.views.view_web import WebView
from infrastructure.repositories.in_memory_todo_repository import InMemoryTodoRepository


def main():
    # ViewModel（PresenterとViewで共有する表示用の状態）
    view_model = TodoViewModel()

    # Repository（今回はメモリ上の実装を使用）
    repository = InMemoryTodoRepository()

    # Presenter（UseCaseの結果をViewModel用メッセージに整形）
    presenter = TodoPresenter(view_model)

    # UseCase（「TODOを追加する」というアプリ固有のビジネスルール）
    use_case = TodoUseCase(presenter, repository)

    # Controller（Viewから受け取った入力をUseCaseに渡す）
    controller = TodoController(use_case)

    # ★ WebViewを使う（これだけがConsoleView/GuiViewとの違い）
    web_view = WebView(controller, view_model)

    # Flaskサーバーを起動
    web_view.run()


if __name__ == "__main__":
    main()
```

ここでも同じことが確認できます。

* `core/` 以下（ドメイン・ユースケース）は一切書き換えていない
* PresenterもRepositoryもそのまま使える
* 変えているのは View の型だけ

これが「UI技術をどんどん差し替えていける設計」の具体例です。

