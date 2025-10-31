# 第4章：Infrastructure（Frameworks & Drivers）

— 昔ながらのWebサーバ＋HTML表示＋SQLite —

## 🎯 この章の目的

* **同心円図の最外層（Frameworks & Drivers＝Infrastructure）** を、昔ながらの構成で実装する
* Controller/Presenter/Gateway の**先（外側）**にある **Web（受付 & 表示）／DB** をつなぐ


---

## 🧩 図との対応（まず全体像）

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

同心円（外→内）：

```
[Frameworks & Drivers]  ← この章（Webサーバ, HTML表示, SQLite）
       ↑
[Interface Adapters]    ← 3章（Controller, Presenter, Gateway）
       ↑
[Use Cases]              ← 2章（UseCaseInteractor, Boundary, DTO）
       ↑
[Entities]               ← 1章（ビジネスルール）
```

クラス図（左から右）：

```
View → ViewModel → Presenter <OutputBoundary> → UseCaseInteractor ←InputBoundary← Controller
                                                       ↓
                                        Data Access Interface（Repository契約）
                                                       ↓
                                              Gateway（翻訳）
                                                       ↓
                               Data Access（行データの出入）→ Database（SQLite）
```

> ここでのポイント：
>
> * **Controller の先＝Webの受付（HTTPサーバ）**
> * **Presenter の先＝View（HTML描画）**
> * **Gateway の先＝DB（SQLite）**

---

## 🧱 フォルダ構成（追加・修正）

```
project_root_v2/
├── interface/                            # 第3章（そのまま）
│   ├── controller/
│   │   └── book_controller.py
│   ├── presenter/
│   │   └── book_presenter.py
│   └── gateway/
│       └── book_gateway.py               # Entity⇔行データの翻訳
├── infrastructure/                       # ★ この章
│   ├── web/
│   │   ├── server.py                     # 昔ながらのHTTPサーバ（受付役）
│   │   └── view_renderer.py              # HTMLテンプレート描画（View）
│   └── db/
│       ├── sqlite_conn.py                # SQLite接続とテーブル作成
│       └── book_data_access.py           # データアクセス実装（行データの出入）
└── main.py                               # 配線（Composition Root）
```

---

## 💡 用語ミニ解説（初登場の言葉）

* **行データ（RowData）**：DBの1行に相当する素朴な入れ物（id, title, … などそのまま）。
* **データアクセス実装（Data Access）**：行データをSQLiteに「入れる・出す」を担当。
* **View**：HTMLの作成担当。Presenterから受け取ったViewModelを使って**画面を描くだけ**。
* **Webサーバ**：HTTPリクエストを受ける**受付**。ここでは `http.server`（標準）を使用。

> ※ 「DAO」「テンプレートエンジン」などの用語は使いません。機能説明に統一します。

---

## 🧠 実装（Bookの「登録＋一覧」を通す）

### 1) DB接続とテーブル作成（Database）

`infrastructure/db/sqlite_conn.py`

```python
# --------------------------------------------------------------------
# [クラス図] Database
# [同心円] Frameworks & Drivers（Infrastructure）
# 役割：SQLite接続の作成・テーブル初期化
# --------------------------------------------------------------------
import sqlite3
from pathlib import Path

DB_PATH = Path("app.db")

DDL_BOOKS = """
CREATE TABLE IF NOT EXISTS books (
  id         INTEGER PRIMARY KEY,
  title      TEXT NOT NULL,
  author     TEXT NOT NULL,
  isbn       TEXT NOT NULL UNIQUE,
  created_at TEXT NOT NULL
);
"""

def get_connection() -> sqlite3.Connection:
    conn = sqlite3.connect(DB_PATH)
    conn.execute("PRAGMA foreign_keys = ON;")
    return conn

def init_db() -> None:
    with get_connection() as conn:
        conn.executescript(DDL_BOOKS)
```

### 2) 行データの出し入れ（Data Access）

`infrastructure/db/book_data_access.py`

```python
# --------------------------------------------------------------------
# [クラス図] Data Access（Databaseの前で行データを出し入れ）
# [同心円] Frameworks & Drivers
# 役割：行データ（DBの1行相当）を SQLite へ入れる/取り出す
# --------------------------------------------------------------------
from typing import Optional, List
from dataclasses import dataclass
from .sqlite_conn import get_connection

# 行データ（RowData）= DBの1行に相当するシンプルな入れ物
@dataclass
class BookRow:
    id: int
    title: str
    author: str
    isbn: str
    created_at_iso: str

class BookDataAccess:
    """行データの入出力担当（INSERT/SELECTなどの実務）"""

    def insert(self, row: BookRow) -> None:
        with get_connection() as conn:
            conn.execute(
                "INSERT INTO books (id, title, author, isbn, created_at) VALUES (?, ?, ?, ?, ?)",
                (row.id, row.title, row.author, row.isbn, row.created_at_iso),
            )

    def select_by_id(self, id_: int) -> Optional[BookRow]:
        with get_connection() as conn:
            cur = conn.execute(
                "SELECT id, title, author, isbn, created_at FROM books WHERE id = ?",
                (id_,),
            )
            r = cur.fetchone()
            return None if not r else BookRow(*r)

    def select_by_isbn(self, isbn: str) -> Optional[BookRow]:
        with get_connection() as conn:
            cur = conn.execute(
                "SELECT id, title, author, isbn, created_at FROM books WHERE isbn = ?",
                (isbn,),
            )
            r = cur.fetchone()
            return None if not r else BookRow(*r)

    def select_search(self, keyword: str, limit: int = 50) -> List[BookRow]:
        kw = f"%{keyword.lower()}%"
        with get_connection() as conn:
            cur = conn.execute(
                """
                SELECT id, title, author, isbn, created_at
                FROM books
                WHERE lower(title) LIKE ? OR lower(author) LIKE ?
                ORDER BY id DESC LIMIT ?
                """,
                (kw, kw, limit),
            )
            return [BookRow(*row) for row in cur.fetchall()]

    def next_id(self) -> int:
        with get_connection() as conn:
            cur = conn.execute("SELECT COALESCE(MAX(id), 0) + 1 FROM books")
            (nid,) = cur.fetchone()
            return int(nid)
```

### 3) Gateway（翻訳：Entity ⇔ 行データ）

※ 第3章の `interface/gateway/book_gateway.py` を、**この行データ実装に接続**します。

```python
# interface/gateway/book_gateway.py（抜粋・確定版）
# [クラス図] Gateway（Repository契約の実装）
# 役割：Entity と 行データ を相互に変換し、UseCaseの契約を満たす
from datetime import datetime
from typing import Optional, List
from domain.book import Book
from usecase.contracts.book_repository import BookRepository
from infrastructure.db.book_data_access import BookDataAccess, BookRow

class BookRepositoryGateway(BookRepository):
    def __init__(self, data_access: BookDataAccess):
        self.dao = data_access

    def next_id(self) -> int:
        return self.dao.next_id()

    def save(self, book: Book) -> Book:
        self.dao.insert(BookRow(
            id=book.id, title=book.title, author=book.author,
            isbn=book.isbn, created_at_iso=book.created_at.isoformat(),
        ))
        return book

    def find_by_id(self, book_id: int) -> Optional[Book]:
        r = self.dao.select_by_id(book_id)
        return None if not r else Book(
            id=r.id, title=r.title, author=r.author, isbn=r.isbn,
            created_at=datetime.fromisoformat(r.created_at_iso),
        )

    def find_by_isbn(self, isbn: str) -> Optional[Book]:
        r = self.dao.select_by_isbn(isbn)
        return None if not r else Book(
            id=r.id, title=r.title, author=r.author, isbn=r.isbn,
            created_at=datetime.fromisoformat(r.created_at_iso),
        )

    def search(self, keyword: str, limit: int = 50) -> List[Book]:
        rows = self.dao.select_search(keyword, limit)
        return [Book(
            id=r.id, title=r.title, author=r.author, isbn=r.isbn,
            created_at=datetime.fromisoformat(r.created_at_iso),
        ) for r in rows]
```

> [クラス図 対応] Data Access Interface（Repository契約）→ Gateway（翻訳）→ Data Access（行の出入）→ Database

---

### 4) Presenter の先（View＝HTML描画）

`infrastructure/web/view_renderer.py`

```python
# --------------------------------------------------------------------
# [クラス図] View（Presenterの先）
# [同心円] Frameworks & Drivers（Infrastructure）
# 役割：Presenterが作ったViewModelをHTMLにする（表示担当）
# --------------------------------------------------------------------
from typing import List, Dict

def render_book_list(view_models: List[Dict]) -> str:
    # 最小限の素朴なHTML（テンプレートエンジンは使わない）
    items = "\n".join(
        f"<li>#{vm['id']} {vm['title']} / {vm['author']} (ISBN: {vm['isbn']})</li>"
        for vm in view_models
    )
    return f"""
<!doctype html>
<html><head><meta charset="utf-8"><title>Books</title></head>
<body>
  <h1>Books</h1>
  <form method="post" action="/books/create">
    <input name="title" placeholder="title">
    <input name="author" placeholder="author">
    <input name="isbn" placeholder="isbn">
    <button type="submit">Create</button>
  </form>
  <ul>
    {items}
  </ul>
</body></html>
""".strip()
```

> **Presenter → View**：Presenterが作る**ViewModel（辞書）**を、Viewが**HTML文字列**に整えるだけ。

---

### 5) Controller の先（Webサーバ＝受付）

`infrastructure/web/server.py`

```python
# --------------------------------------------------------------------
# [クラス図] View → Controller（HTTPの受付）
# [同心円] Frameworks & Drivers（Infrastructure）
# 役割：HTTPを受け、Controllerに入力を渡す。Presenter→ViewでHTMLを返す。
# --------------------------------------------------------------------
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import parse_qs
from interface.controller.book_controller import BookController
from interface.presenter.book_presenter import BookPresenter
from infrastructure.web.view_renderer import render_book_list
from usecase.dto import RegisterBookInput

class WebHandler(BaseHTTPRequestHandler):
    # DIポイント（Mainで差し込む）
    controller: BookController = None
    usecase = None  # RegisterBook（InputBoundary実装）

    def do_GET(self):
        if self.path == "/" or self.path.startswith("/books"):
            # 一覧表示のため、Presenterを作ってUseCaseにセットし、検索ユースケース等があれば呼ぶ
            presenter = BookPresenter()
            # 今回は簡略化：Gatewayの検索を直接呼ばず、必要ならUseCaseの「一覧取得」を別途用意して呼ぶ想定。
            # ここでは「登録直後のメモリ保持はしない」ため、ViewModel生成を空で返す例。
            html = render_book_list([])  # 本格対応は次章の「一覧UseCase」で
            self._respond_html(html)
        else:
            self.send_error(404, "Not Found")

    def do_POST(self):
        if self.path == "/books/create":
            length = int(self.headers.get('Content-Length', 0))
            body = self.rfile.read(length).decode()
            form = parse_qs(body)
            title  = (form.get("title") or [""])[0]
            author = (form.get("author") or [""])[0]
            isbn   = (form.get("isbn") or [""])[0]

            # Controller → InputBoundary：HTTP入力をDTOに変換してユースケースを実行
            presenter = BookPresenter()
            self.usecase.set_output_boundary(presenter)  # OutputBoundary注入
            self.controller.handle_request(title, author, isbn)

            # Presenter → View：ViewModelはPresenterが持つ
            vm = presenter.get_view_model()
            html = render_book_list([vm])
            self._respond_html(html)
        else:
            self.send_error(404, "Not Found")

    def _respond_html(self, html: str):
        data = html.encode("utf-8")
        self.send_response(200)
        self.send_header("Content-Type", "text/html; charset=utf-8")
        self.send_header("Content-Length", str(len(data)))
        self.end_headers()
        self.wfile.write(data)

def run_server(host="127.0.0.1", port=8080):
    httpd = HTTPServer((host, port), WebHandler)
    print(f"Serving on http://{host}:{port}")
    httpd.serve_forever()
```

> **ここが肝**：
>
> * **Webサーバ（受付）**は **Controller** を呼ぶだけ。
> * **Presenter** が作った **ViewModel → View（HTML文字列）** で応答。
> * これで **Controllerの先＝Web受付**、**Presenterの先＝HTML View** が図どおりに可視化されます。

---

### 6) 配線（Composition Root）

`main.py`

```python
# --------------------------------------------------------------------
# [クラス図] Main（Composition Root）
# 役割：依存性注入（外→内へつなぐ）。Web/DBを1か所で接続。
# --------------------------------------------------------------------
from infrastructure.db.sqlite_conn import init_db
from infrastructure.db.book_data_access import BookDataAccess
from interface.gateway.book_gateway import BookRepositoryGateway
from usecase.register_book import RegisterBook
from interface.controller.book_controller import BookController
from infrastructure.web.server import run_server, WebHandler

def main():
    # 1) DB初期化
    init_db()

    # 2) Gateway（IA） ← Data Access（Infra）
    data_access = BookDataAccess()
    repository  = BookRepositoryGateway(data_access)

    # 3) UseCase（Interactor）
    usecase = RegisterBook(repository)

    # 4) Controller（IA）
    controller = BookController(usecase)

    # 5) WebサーバにDI（Controller/UseCaseを差し込む）
    WebHandler.controller = controller
    WebHandler.usecase = usecase

    # 6) サーバ起動（受付開始）
    run_server()

if __name__ == "__main__":
    main()
```

> **依存は内へ**：Web（外）→ Controller → UseCase → Gateway → DataAccess → DB（内への参照はなし）
> **制御は外へ**：ユーザーのリクエストが外から入り、結果が外（HTML）へ返る


---

## 🧪 テストのヒント（この章の範囲）

* **DataAccessテスト**：`insert`→`select` の往復で値が一致するか
* **Gatewayテスト**：Entity ⇔ 行データの変換で情報が欠落しないか
* **結合テスト**：`main.py` を立てて `/books/create` にフォームPOST → HTMLに新規行が表示されるか

---

## 📘 まとめ（この章の理解ポイント）

* **Frameworks & Drivers（＝Infrastructure）** は**技術の世界**（Web, View, DB）。
* **Controllerの先＝Web受付**、**Presenterの先＝View（HTML）**、**Gatewayの先＝DB** が**図どおり**接続できた。
* 用語はやさしく：「行データ＝DBの1行」「データアクセス実装＝DBへの出し入れ担当」。
* **依存は内へ**（外が内に依存し、内は外を知らない）／**制御は外へ**（外から入り外へ返る）。


## 📎 補足：静的依存と動的制御のちがい（なぜクラス図に Controller—View の線がない？）

クリーンアーキテクチャでは、**「静的依存（設計）」と「動的制御（実行時の流れ）」を分けて考える**ことが大切です。
クラス図では依存関係を、実行時には制御の流れをそれぞれ表します。

---

### 🔹 静的依存（クラス図に描かれるもの）

クラス図は「どのクラスがどのクラスを知っているか（import しているか）」を表す図です。
そのため、下記のような依存関係だけが描かれます。

* Controller → **InputBoundary（UseCaseの入口）**
* UseCase → **OutputBoundary（Presenterの入口）**
* Presenter → **ViewModel（表示用データ）**

✅ **Controller と View は静的には無関係**なので、クラス図には線が引かれません。

---

### 🔹 動的制御（実行時の流れ）

実際にアプリを動かすときには、制御は外側から内側へ流れ、結果がまた外側へ戻ります。
これは**シーケンス図（実行の流れ）**で表すとわかりやすくなります。

（簡易のシーケンスを言葉で）

1. **Web（受付／HTTP）** がフォーム入力を受け取る
2. **Controller** が入力を **InputData** に変換し、**UseCase（InputBoundary）** を呼ぶ
3. **UseCase** が処理し、**Presenter（OutputBoundary）** に **OutputData** を渡す
4. **Presenter** が **ViewModel** を作る
5. **View** が **ViewModel** を **HTML** に整形して返す

---

### 🔹 各層の関係性（誤解しやすいポイント）

この連携では、**Web(受付)** がアプリ全体の“オーケストレーター”になります。

| 層          | 関係                            |
| ---------- | ----------------------------- |
| Controller | View を呼びません                   |
| Presenter  | Controller を知りません             |
| View       | Presenter の ViewModel だけを使います |

→ したがって、**Controller ⇄ View の直接依存は存在せず**、
　**クラス図に線がないのが正解**です。

---

### 🔹 まとめ

> **設計（静的依存）は内向きに閉じる。**
> **実行（動的制御）は外から内へ入り、内から外へ返る。**
>
> これが「依存は内へ、制御は外へ」というクリーンアーキテクチャの本質です。
