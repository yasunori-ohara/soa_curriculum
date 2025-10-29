# 第4章：Infrastructure（インフラストラクチャ）層 

## 🎯 この章の目的

* **Frameworks & Drivers（最外層）** をすべて揃える
  → Database（保存先）＋ 入力装置（受付）＋ 出力装置（表示）
* **UseCase 層の抽象（NoteRepository）** を **SQLite（Database）** で実装
* **Controller / Presenter の先（外界）** を CLI 入出力として具現化
* 依存方向は **外→内**、制御方向は **内→外**

---

## 🧩 Infrastructure 層とは（図のどこ？）

![クリーンアーキテクチャ・依存と制御](../クリーンアーキテクチャ・依存と制御.png)

| 図の要素                 | Infrastructure層での実装例 | 説明              |
| -------------------- | -------------------- | --------------- |
| **Database（保存先）**    | SQLite               | Entityを永続化する物理層 |
| **Controllerの先（入力）** | Console/Keyboard     | CLIからの入力（受付）    |
| **Presenterの先（出力）**  | Console/Screen       | CLIへの出力（表示）     |

> **Frameworks & Drivers** は「技術的な入口と出口」です。
> UIがCLIであれHTTPであれ、入力／出力のデバイス層はこの外周に位置します。

---

## 🧱 フォルダ構成（この章の範囲）

```
project_root/
├── domain/
│   └── note.py
├── usecase/
│   ├── dto.py
│   ├── boundaries/
│   │   └── note_boundaries.py
│   ├── contracts/
│   │   └── note_repository.py
│   ├── create_note.py
│   └── get_all_notes.py
├── interface/
│   ├── controller.py
│   ├── presenter.py
│   └── repository_adapter.py
└── infrastructure/
    ├── sqlite/
    │   ├── db.py
    │   └── note_repository_sqlite.py
    └── cli/
        ├── console_input.py     ← 入力装置（Controllerの先）
        └── console_output.py    ← 出力装置（Presenterの先）
```

---

## 💡 この層のポイント

| 観点       | 内容                                                   |
| -------- | ---------------------------------------------------- |
| **責務**   | 技術的なI/Oをすべて担当（Database・Keyboard・Screen）              |
| **位置付け** | クリーンアーキテクチャの最外層「Frameworks & Drivers」                |
| **依存方向** | infrastructure → interface / usecase / domain（外→内のみ） |
| **差し替え** | CLI ⇄ Web、SQLite ⇄ File をMainで変更可                    |
| **安定性**  | 最も不安定な層（技術変化に最も依存）だが、内側には影響しない構造                     |

---

## ✳️ Database：`infrastructure/sqlite/db.py`

```python
# --------------------------------------------------------------------
# [クラス図] Database（保存先そのもの）
# [同心円] Frameworks & Drivers（最外層）
# --------------------------------------------------------------------
import sqlite3
from pathlib import Path

DB_PATH = Path("data/notes.db")

DDL = """
CREATE TABLE IF NOT EXISTS notes (
  id INTEGER PRIMARY KEY,
  title TEXT NOT NULL,
  content TEXT NOT NULL
);
"""

def get_connection() -> sqlite3.Connection:
    DB_PATH.parent.mkdir(parents=True, exist_ok=True)
    conn = sqlite3.connect(DB_PATH)
    conn.execute("PRAGMA foreign_keys = ON;")
    return conn

def init_db() -> None:
    with get_connection() as conn:
        conn.executescript(DDL)
```

---

## ✳️ Data Source（Repository契約の具象）：`note_repository_sqlite.py`

```python
# --------------------------------------------------------------------
# [クラス図] Data Source（Database実装）
# [同心円] Frameworks & Drivers（最外層：技術の世界）
# --------------------------------------------------------------------
from typing import List
from domain.note import Note
from usecase.contracts.note_repository import NoteRepository
from .db import get_connection

class SQLiteNoteRepository(NoteRepository):
    def next_id(self) -> int:
        with get_connection() as conn:
            cur = conn.execute("SELECT COALESCE(MAX(id), 0) + 1 FROM notes")
            (nid,) = cur.fetchone()
            return int(nid)

    def save(self, note: Note) -> Note:
        with get_connection() as conn:
            conn.execute(
                "INSERT OR REPLACE INTO notes (id, title, content) VALUES (?, ?, ?)",
                (note.id, note.title, note.content),
            )
        return note

    def get_all(self) -> List[Note]:
        with get_connection() as conn:
            cur = conn.execute("SELECT id, title, content FROM notes ORDER BY id ASC")
            return [Note(id=row[0], title=row[1], content=row[2]) for row in cur.fetchall()]
```

---

## ✳️ 入力装置（Controllerの先）：`cli/console_input.py`

```python
# --------------------------------------------------------------------
# [クラス図] Controller の外側（受付装置）
# [同心円] Frameworks & Drivers（最外層：入力デバイス）
# --------------------------------------------------------------------
class ConsoleInput:
    """CLI入力を受け取ってControllerへ渡す（Controllerの先）"""

    def get_command(self) -> str:
        return input("> ").strip().lower()

    def get_note_info(self):
        title = input("Title: ").strip()
        content = input("Content: ").strip()
        return title, content
```

---

## ✳️ 出力装置（Presenterの先）：`cli/console_output.py`

```python
# --------------------------------------------------------------------
# [クラス図] Presenter の外側（表示装置）
# [同心円] Frameworks & Drivers（最外層：出力デバイス）
# --------------------------------------------------------------------
from typing import List

class ConsoleOutput:
    """Presenterが生成したViewModelを画面出力する（Presenterの先）"""

    def render_created(self, vm) -> None:
        print(f"[CREATED] [{vm.id}] {vm.title} - {vm.preview}")

    def render_list(self, vms: List) -> None:
        if not vms:
            print("(no notes)")
            return
        for vm in vms:
            print(f"[{vm.id}] {vm.title} - {vm.preview}")
```

---

## 🔎 依存と制御（図で再確認）

![クリーンアーキテクチャ・依存と制御](../クリーンアーキテクチャ・依存と制御.png)

| 層              | 依存（import）                     | 制御（実行）                                               |
| -------------- | ------------------------------ | ---------------------------------------------------- |
| Infrastructure | → Interface / UseCase / Domain | View→Controller→UseCase→Repository→DB→Presenter→View |
| Interface      | → UseCase / Domain             | Controller→UseCase, Presenter→View                   |
| UseCase        | → Domain                       | 処理実行                                                 |
| Domain         | なし                             | 内部ロジック                                               |

---

## 🧪 テストのヒント（この章の範囲）

| 対象                   | 検証項目                                     |
| -------------------- | ---------------------------------------- |
| SQLiteNoteRepository | `next_id()`が連番、`save()`と`get_all()`が整合する |
| ConsoleInput         | 入力を正しく返す（mockでテスト可）                      |
| ConsoleOutput        | ViewModelを正しい形式で出力（stdoutキャプチャ）          |

---

## 📘 理解ポイント

| 要素                | 学びのポイント                        |
| ----------------- | ------------------------------ |
| **Database（保存先）** | Repository契約の具象。永続化の責務を分離する。   |
| **入力装置（受付）**      | Controllerの“先”。ユーザー操作を抽象化する。   |
| **出力装置（表示）**      | Presenterの“先”。画面出力やレンダリングを抽象化。 |
| **依存方向**          | 外→内。技術詳細が内側のルールを知らない。          |
| **安定性**           | 最も変化しやすいが、内側は変更不要。差し替え容易。      |

---

## 📎 補足：〈DS〉と Database の関係

* 今回は Database（SQLite）を使って実装
* 実は 〈DS〉 は Database を含む「広い概念」
  → 外部API・ファイル・センサー入出力などもここに位置する
* クリーンアーキテクチャでは、「抽象（契約）」と「具象（技術）」を分離することが目的であり、
  Databaseを使ってもファイルを使っても設計上は同じ位置にある。

---

## 🔄 次のステップ：Main（Composition Root）

次章では、

* Controller / Presenter / Repository / 入出力装置
  をすべて依存注入で結線して、アプリ全体を「起動」できるようにします。
  CLI環境でも Controller と Presenter の先を明示し、**図の線がそのまま動作の線**になることを確認します。

---

これで、「保存先」「入力装置」「出力装置」をすべて同心円図上で明示し、
Controller・Presenterの先まで含めて完全に閉じた構造になります。
