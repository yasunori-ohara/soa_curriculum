# 第5章：Main（Composition Root）


## 🧱 Frameworks & Drivers：CLIの受付＆View

```
infrastructure/
└── cli/
    ├── console_server.py    ← 受付（Controllerの先）：キーボード入力を解釈して Controller を呼ぶ
    └── console_view.py      ← View（Presenterの先）：ViewModel をテキストに整形して表示
```

## ✳️ `infrastructure/cli/console_view.py`（Presenterの先＝View）

```python
# --------------------------------------------------------------------
# [クラス図] View（Presenterの先）
# [同心円] Frameworks & Drivers（最外層）
# 役割：Presenter が作った ViewModel を、端末表示に“最終整形”して出す
# --------------------------------------------------------------------
from typing import List

class ConsoleView:
    def render_created(self, vm) -> None:
        print(f"[CREATED] [{vm.id}] {vm.title} - {vm.preview}")

    def render_list(self, vms: List) -> None:
        if not vms:
            print("(no notes)")
            return
        for vm in vms:
            print(f"[{vm.id}] {vm.title} - {vm.preview}")
```

## ✳️ `infrastructure/cli/console_server.py`（Controllerの先＝受付）

```python
# --------------------------------------------------------------------
# [クラス図] （View→）Controller の“さらに外”：受付（入力ドライバ）
# [同心円] Frameworks & Drivers（最外層）
# 役割：ユーザーの操作（キーボード入力）を受け取り、Controller に渡す
# --------------------------------------------------------------------
class ConsoleServer:
    def __init__(self, controller, view):
        self.controller = controller
        self.view = view

    def run(self):
        print("== Note App (Clean Architecture / CLI) ==")
        while True:
            print("\nCommands: [1] Create  [2] List  [q] Quit")
            cmd = input("> ").strip().lower()
            if cmd == "1":
                title = input("Title: ").strip()
                content = input("Content: ").strip()
                vm = self.controller.create(title, content)  # Controller→UseCase→Presenter→VM
                self.view.render_created(vm)                  # Presenterの先（View）
            elif cmd == "2":
                vms = self.controller.list_all()
                self.view.render_list(vms)
            elif cmd == "q":
                print("Bye.")
                break
            else:
                print("unknown command")
```

> これで、**Controller の先＝受付（ConsoleServer）**、**Presenter の先＝View（ConsoleView）** が**明示化**され、**第二巡（Web/HTML）と同じ見え方**になります。

---

## ✳️ `main.py`（配線：RepositoryにSQLiteを選ぶ／CLI受付とViewを結線）

```python
# --------------------------------------------------------------------
# [クラス図] Composition Root（Main）
# [同心円] 最外層（Frameworks & Drivers）…配線（依存注入）の決定点
# 役割：外側(受付/表示/Database) と 内側(Controller/UseCase/Entity) を「外→内」で結線
# --------------------------------------------------------------------
from usecase.create_note import CreateNoteUseCase
from usecase.get_all_notes import GetAllNotesUseCase
from usecase.contracts.note_repository import NoteRepository

from interface.controller import NoteController              # [IA] Controller（左端の箱）
# ※ 未使用だった NotePresenter の import を削除

from infrastructure.sqlite.db import init_db                 # [Infra] Database 初期化
from infrastructure.sqlite.note_repository_sqlite import SQLiteNoteRepository  # [Infra] Database 実装
# from interface.repository_adapter import InMemoryNoteRepository               # ← 差し替え例（任意）

from infrastructure.cli.console_view import ConsoleView      # [Infra] View（Presenter の先）
from infrastructure.cli.console_server import ConsoleServer  # [Infra] 受付（Controller の先）


def build_app():
    # [Infra/Database] 物理保存層のセットアップ
    init_db()

    # [Infra → UseCase契約] Repository(〈Database〉) を選択（差し替え点）
    repo: NoteRepository = SQLiteNoteRepository()
    # repo = InMemoryNoteRepository()  # ← ここ1行で切り替え可能

    # [UseCase] Interactor（アプリケーションルール）…抽象(Repo契約) にのみ依存
    create_uc = CreateNoteUseCase(repo)
    list_uc = GetAllNotesUseCase(repo)

    # [IA/Controller] 入力を UseCase の InputBoundary へ受け渡す翻訳係
    controller = NoteController(create_uc, list_uc)

    # [Infra/View] Presenter が作る ViewModel を最終表示に整形
    view = ConsoleView()

    # [Infra/受付] 入力デバイス（キーボード）からの操作を受け取り Controller を呼ぶ
    server = ConsoleServer(controller, view)
    return server


if __name__ == "__main__":
    app = build_app()
    app.run()
```


## 🔎 依存と制御（図で再確認）

* **依存（import）**：
  受付/View/SQLite（Infra） → IA（Controller/Presenter） → UseCase（契約/DTO） → Entity
  **外→内のみ**。逆は無し。
* **制御（実行）**：
  ConsoleServer（受付）→ Controller → UseCase → Repository（契約）→ **SQLite** →（戻る）Presenter → **ConsoleView（表示）**


## 🧪 テストのヒント

* **SQLiteNoteRepository 単体**：`next_id` / `save` / `get_all`（テスト用DBパス）
* **ConsoleView 単体**：`render_*` が与えられた ViewModel を所定のテキストに整形（stdout をキャプチャ）
* **ConsoleServer は最小限**：入力依存なので、対話を避けて関数分割 or E2E は手動に留める


## 📘 章末補足：〈DS〉と Database の関係（教育の順番）

* **まずは Database を使って実装**（今回）→ 図の「Database」の箱がそのまま理解できる
* その上で **「実は 〈DS〉 は Database を含む広い概念」**（ファイル保存・外部APIなどもここに入る）と解説
* **差し替えは Main の1行**：InMemory ⇄ **SQLite** ⇄ File… と切替可能


### これで：

* **図の箱が“全部”コード上の箱に対応**（CLIでも「受付」「View」を明示）
* **第1巡でも第2巡でも同じ見え方**（外側が HTTP/HTML か Console の違いだけ）
* 学習者は **「図どおりに並べれば動く」** をそのまま体験できます。


