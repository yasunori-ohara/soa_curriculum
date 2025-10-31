# 20 機能拡張：UIをGUI版へ拡張

# 🖼 GUI版 View
### `interface_adapters/views/view_gui.py` - `GuiView`

---

## 🧭 このクラスの役割

`GuiView` は、**ユーザーが直接操作するGUI（グラフィカル・ユーザー・インターフェース）**を提供するクラスです。

CLI版の `ConsoleView` と同様に、責務は次の2つに集約されます：

1. **表示（Output）**
   `Presenter` によって更新された `TodoViewModel` の内容を、GUI上のウィジェット（ラベルなど）に描画する。
2. **入力（Input）**
   ユーザーがGUIで入力したTodoタイトルを `Controller` に渡す。

`GuiView` は、画面とのやり取りにだけ集中します。
ビジネスロジックそのものは扱わず、アプリの内部構造（UseCaseやRepositoryなど）も知りません。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## 🔁 なぜGUIへの差し替えが「簡単」なのか？

> 💡 クリーンアーキテクチャを採用したからこそ、UI技術の差し替えが容易になっています。

* `View` はアーキテクチャの最外層（Frameworks & Drivers）にあり、**ControllerとViewModelにだけ依存**します。
* `Presenter` や `UseCase` は、ViewがCLIかGUIかWebかを一切知りません。
* そのため、CLI版の `ConsoleView` を `GuiView` に置き換えても、**UseCase / Presenter / Repository / Entity は1行も変更しなくてよい**状態になっています。

これは、UI技術（コンソール、デスクトップGUI、Webフロントなど）が変わっても、
ビジネスルールを巻き込まずに差し替えできる設計になっていることの証明です。

---

## ✅ このクラスにあるべき処理

⭕️ 含めるべき処理の例

* `tkinter` を使ったウィンドウ・ボタン・入力欄などのGUI部品の構築
* `TodoViewModel` の内容（Presenterが更新する）を画面のラベルに反映する
* 「追加」ボタンが押されたときに、入力値を `Controller` に渡す

❌ 含めてはいけない処理の例

* ビジネスロジック（UseCaseの責務）
* 表示用メッセージの整形（Presenterの責務）
* データ保存・取得（Repositoryの責務）
* IDの採番などインフラ寄りの処理（Repositoryの責務）

---

## 📂 ファイルの配置

`GuiView` は CLI版の `ConsoleView` と同じ並びに置きます。

```
├─ interface_adapters/
│   ├─ presenters/
│   │   └─ todo_presenter.py
│   ├─ controllers/
│   │   └─ todo_controller.py
│   └─ views/
│       ├─ view_console.py      # 既存: CLI版 View
│       └─ view_gui.py          # 新規: GUI版 View（本章の主役）
```

この「横並び」そのものが、UI差し替えのしやすさをそのまま表現しています。

---

## 🔍 ソースコード（正式版）

```python
# --------------------------------------------------------------------
# File: interface_adapters/views/view_gui.py
# Layer: Interface Adapters（View - GUI版）
#
# 責務：
#   - ユーザーからの入力（GUI操作）を受け取り、Controllerに渡す。
#   - Presenterが更新したViewModelをもとに画面を描画する。
#
# 依存：
#   - <I> Controller（ユーザー操作の伝達先）
#   - <DS> TodoViewModel（画面表示のための状態）
#
# 同心円図での位置：
#   - Frameworks & Drivers（最外層）
#   - 具体的な入出力の実装（ここではTkinter）を扱う層
#
# 備考：
#   - CLI版(ConsoleView)と責務は同じ。
#   - 見せ方が「printとinput」か「GUIウィジェット」かだけが違う。
# --------------------------------------------------------------------

import tkinter as tk
from interface_adapters.controllers.todo_controller import TodoController
from core.usecase.boundary.dto import TodoViewModel


class GuiView:
    """
    GUI版のView。Tkinterでウィンドウを構築し、
    ユーザー入力イベントと表示更新を担当する。
    """

    def __init__(self, controller: TodoController, view_model: TodoViewModel):
        """
        [依存関係]
        - controller: ユーザー操作をUseCaseに橋渡しする仲介役
        - view_model: Presenterが更新し、画面表示の元データになるオブジェクト
        """
        self._controller = controller
        self._view_model = view_model

        # GUIウィンドウの初期化
        self.root = tk.Tk()
        self.root.title("TODOアプリ（GUI版）")

        # 入力欄（ユーザーがTodoタイトルを打ち込む場所）
        self.entry = tk.Entry(self.root, width=40)
        self.entry.pack(pady=10)

        # 追加ボタン（クリックでControllerに依頼）
        self.button = tk.Button(self.root, text="追加", command=self.on_add_clicked)
        self.button.pack(pady=5)

        # Presenterが用意した表示メッセージを貼り付けるラベル
        self.label = tk.Label(self.root, text="", fg="green", wraplength=400, justify="left")
        self.label.pack(pady=10)

        # 画面の初期描画
        self.render()

    def on_add_clicked(self):
        """
        [入力処理]
        Entry欄の文字列をControllerに渡し、UseCaseを実行させる。
        その後、最新のViewModelの状態で画面を再描画する。
        """
        title = self.entry.get()
        self._controller.add_todo(title)
        self.render()

    def render(self):
        """
        [表示処理]
        ViewModelの現在の状態をラベルに反映する。
        Presenterが整形した「人間向けメッセージ」をそのまま表示する。
        """
        self.label.config(text=self._view_model.display_text)

    def run(self):
        """
        [起動処理]
        GUIのメインループ（イベントループ）を開始する。
        これ以降、ボタンクリックなどのイベントでon_add_clickedが呼ばれる。
        """
        self.root.mainloop()
```

---

## 🛡 このクラスの鉄則

> 見せて、渡して、黙って待て。

* Viewは**画面に表示する**（Presenterが作ってくれた文言をそのまま出す）
* Viewは**ユーザーの操作をControllerに渡す**（入力値だけを渡す）
* Viewは**それ以上なにもしない**（ロジック・整形・保存は一切しない）

CLI版と同じ役割を、ただ別のUI技術（tkinter）で担っているだけ、というのが重要なポイントです。

---

## 🔧 GUI版を使う `main.py` の例

`main.py` は「すべての部品を配線してアプリを起動する場所」でした。
ここで `ConsoleView` の代わりに `GuiView` を組み込むだけで、GUIアプリとして動かすことができます。

```python
# main.py (GUI版で動かしたい場合の例)

from core.usecase.boundary.dto import TodoViewModel
from core.usecase.interactor.create_todo import TodoUseCase
from interface_adapters.presenters.todo_presenter import TodoPresenter
from interface_adapters.controllers.todo_controller import TodoController
from interface_adapters.views.view_gui import GuiView
from infrastructure.repositories.in_memory_todo_repository import InMemoryTodoRepository


def main():
    # ViewModel（PresenterとViewで共有される表示用の状態）
    view_model = TodoViewModel()

    # Repository（今回はメモリ上にTodoを保存する実装）
    repository = InMemoryTodoRepository()

    # Presenter（UseCaseの結果をViewModel用メッセージに整形）
    presenter = TodoPresenter(view_model)

    # UseCase（アプリケーション固有のビジネスロジック）
    use_case = TodoUseCase(presenter, repository)

    # Controller（Viewからの入力をUseCaseに橋渡しする）
    controller = TodoController(use_case)

    # ★ ここがCLI版との違い：ConsoleViewではなくGuiViewを使う
    view = GuiView(controller, view_model)

    # GUIのイベントループを開始
    view.run()


if __name__ == "__main__":
    main()
```

大事なのは：

* `core/`（ドメインやユースケース層）はいっさい変更していない
* `interface_adapters/presenters`（Presenter）もそのまま再利用
* `infrastructure/repositories`（Repository実装）もそのまま再利用
* 本当に差し替わっているのは **View** だけ

これが「UI技術の変更に強い」＝「フロントエンドの寿命とバックエンドの寿命を分離できる」設計の力です。

