# 20 機能拡張：UIをGUI版へ拡張

# 🖼 GUI版 View : `ui/view_gui.py` - `GuiView`

---

## 🧭 このクラスの役割

`GuiView` は、**ユーザーが直接操作するGUI（グラフィカル・ユーザー・インターフェース）**を提供するクラスです。

CLI版の `ConsoleView` と同様に、責務は以下の2つに集約されます：

1. **表示（Output）**`Presenter` によって更新された `ViewModel` の内容を、GUI上のラベルなどに描画する。
2. **入力（Input）**
ユーザーがGUI上で入力した内容を `Controller` に渡す。

![クリーンアーキテクチャ](https://www.notion.so../%E3%82%AF%E3%83%AA%E3%83%BC%E3%83%B3%E3%82%A2%E3%83%BC%E3%82%AD%E3%83%86%E3%82%AF%E3%83%81%E3%83%A3%E3%83%BB%E5%90%8C%E5%BF%83%E5%86%86.png)

---

## 🔁 なぜGUIへの差し替えが「簡単」なのか？

> 💡 クリーンアーキテクチャを採用したからこそ、UIの差し替えが容易になっています。
> 
- `View`は最外層に位置し、**ControllerとViewModelにのみ依存**しています。
- `Presenter`や`UseCase`は、Viewの具体的な実装（CLIかGUIか）を一切知りません。
- そのため、CLI版の `ConsoleView` を `GuiView` に置き換えるだけで、**他の層には一切変更が不要**です。
- これは、UI技術の変更に強い設計が実現されている証拠です。

---

## ✅ このクラスにあるべき処理

⭕️ **含めるべき処理の例**

- `tkinter` を使ったウィンドウ・ボタン・入力欄の構築
- `ViewModel` の内容をGUIに反映する処理
- ユーザー操作（ボタン押下）に応じて `Controller` を呼び出す処理

❌ **含めてはいけない処理の例**

- ビジネスロジック（Use Caseの責務）
- 表示用メッセージの整形（Presenterの責務）
- データ保存や取得（Repositoryの責務）

---

## 🔍 ソースコード（コメント充実）

```python
# --------------------------------------------------------------------
# View に相当（GUI版）
#
# 責務：ユーザーとの直接的なやり取り（入力受付・画面表示）を担当する。
# 依存：<DS> ViewModel を知っており、Controller を呼び出す。
#
# 位置づけ（クラス図／同心円図）：
# - クラス図：Interface Adapters（View）
# - 同心円図：Framework & Drivers（最外層）
#
# C言語との比較:
#   - GUIは「イベント駆動型のmainループ」に近く、割り込み処理のような構造。
# --------------------------------------------------------------------

import tkinter as tk
from interface_adapters.controller import AddTodoController
from data_structures import TodoViewModel

class GuiView:
    """
    GUI版のView。Tkinterを使ってウィンドウを構築し、
    ユーザー入力と画面表示を担当する。
    """

    def __init__(self, controller: AddTodoController, view_model: TodoViewModel):
        """
        [依存先の定義]
        - Controller: ユーザー操作をビジネスロジックに橋渡しする仲介役
        - ViewModel: 表示内容の情報源（Presenterが更新する）
        """
        self._controller = controller
        self._view_model = view_model

        # GUIウィンドウの初期化
        self.root = tk.Tk()
        self.root.title("TODOアプリ（GUI版）")

        # 入力欄
        self.entry = tk.Entry(self.root, width=40)
        self.entry.pack(pady=10)

        # 追加ボタン
        self.button = tk.Button(self.root, text="追加", command=self.on_add_clicked)
        self.button.pack(pady=5)

        # 表示ラベル（ViewModelの内容を表示）
        self.label = tk.Label(self.root, text="", fg="green", wraplength=400)
        self.label.pack(pady=10)

        # 初期表示
        self.render()

    def on_add_clicked(self):
        """
        [入力処理]
        Entry欄のテキストをControllerに渡す。
        """
        title = self.entry.get()
        self._controller.add_todo(title)
        self.render()

    def render(self):
        """
        [表示処理]
        ViewModelの内容をラベルに反映する。
        """
        self.label.config(text=self._view_model.display_text)

    def run(self):
        """
        [起動処理]
        GUIのメインループを開始する。
        """
        self.root.mainloop()

```

---

## 🛡 このクラスの鉄則

> 見せて、渡して、黙って待て。
> 
- Viewは「表示」と「入力の橋渡し」に徹する
- ビジネスロジックやデータ整形は一切行わない
- GUI技術に依存するが、責務はCLI版と同じ

---

## 🔧 `main.py` での差し替え例（GUI版）

```python
# --- GUI版Viewを使う場合 ---
from ui.view_gui import GuiView

# ...（Presenter, Controller, ViewModelなどの生成は同じ）

view = GuiView(controller, view_model)
view.run()

```