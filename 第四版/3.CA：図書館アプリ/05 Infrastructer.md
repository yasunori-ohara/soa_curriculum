# 第4章（発展）：Infrastructure を FastAPI に差し替える

「昔ながらのWebサーバ＋HTML表示」から **FastAPI** に“外側だけ”差し替える章を用意しました。  
ゴールは **(1) 層を守ったままフレームワークを入れ替えられる**ことを体感し、**(2) FastAPIの最小限だけ学ぶ**こと。


## 🎯 この章の目的

* **Frameworks & Drivers（最外層）** を `http.server` → **FastAPI** に入れ替える
* **中の層（Entity／UseCase／Controller／Presenter／Gateway／DB）には手を入れない**ことを確認する
* FastAPIの**最小限**を理解（エンドポイントの作り方、入力の受け取り、レスポンスの返し方）

---

## 🧩 図との対応（変わるのは一番外の円だけ）

```
[Frameworks & Drivers]  ← これだけ http.server → FastAPI に差し替え
       ↑
[Interface Adapters]    ← Controller / Presenter / Gateway（変更なし）
       ↑
[Use Cases]              ← Interactor / Boundary / DTO（変更なし）
       ↑
[Entities]               ←（変更なし）
```

> 置き換え可能なのは、クリーンアーキテクチャの恩恵そのもの。
> **依存は内へ**を守っているから、外側を替えても内側は無風です。

---

## 💡 用語ミニ解説（FastAPI で出てくる新単語）

* **FastAPI**：PythonのWebフレームワーク。`@app.get`, `@app.post` の関数が **Webの受付** になる。
* **Pydantic BaseModel**：JSONボディの**型チェック**に使う箱（ここでは最小限だけ使用）。
* **Response**：関数の戻り。**JSON**や**HTML**を返せる。

> 今回は DTO（InputData/OutputData）は **UseCase層のものをそのまま使う**ため、
> Pydanticは「HTTPを受け取る時の軽いバリデーション」にだけ使います。

---

## 🧱 追加フォルダと差し替え点

```
project_root/
├── infrastructure/
│   ├── web/                 # ← （前章の素朴HTTPは残してよい）
│   └── web_fastapi/         # ★ 新規：FastAPI版
│       └── app.py
└── main_fastapi.py          # ★ 新規：FastAPIアプリのComposition Root
```

> 既存の **interface/**（Controller/Presenter/Gateway）・**usecase/**・**infrastructure/db/** は**変更なし**。

---

## ✳️ FastAPIアプリ（受付＝Frameworks & Drivers）

### `infrastructure/web_fastapi/app.py`

```python
# --------------------------------------------------------------------
# [クラス図] View→Controller の「受付」を FastAPI で実装
# [同心円] Frameworks & Drivers（Infrastructure）
# 役割：HTTPを受け、Controllerに入力を渡し、Presenter→ViewModelを返す
# --------------------------------------------------------------------
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field

# IA層（変更なし）を利用
from interface.controller.book_controller import BookController
from interface.presenter.book_presenter import BookPresenter
from usecase.register_book import RegisterBook
from usecase.dto import RegisterBookInput

# DBまわり（Infra）と Gateway（IA層）
from infrastructure.db.sqlite_conn import init_db
from infrastructure.db.book_data_access import BookDataAccess
from interface.gateway.book_gateway import BookRepositoryGateway

# ---- HTTP受け取り用（軽いバリデーション）: 外の形
class BookCreateBody(BaseModel):
    title: str = Field(..., min_length=1)
    author: str = Field(..., min_length=1)
    isbn: str = Field(..., min_length=3)

def create_app() -> FastAPI:
    # DB初期化（テーブル作成）
    init_db()

    # 依存性注入（外→内）：Infra → IA → UseCase
    dao = BookDataAccess()
    repo = BookRepositoryGateway(dao)
    usecase = RegisterBook(repo)
    controller = BookController(usecase)

    app = FastAPI(title="Library App (FastAPI)")

    @app.get("/")
    def hello():
        # 軽い案内（HTMLで返したい場合は JSON ではなく HTMLResponse を使ってもOK）
        return {"message": "POST /books に title/author/isbn を送ると登録できます"}

    @app.post("/books", status_code=201)
    def register_book(body: BookCreateBody):
        """
        [Controllerの先=受付]
        - HTTP(JSON) → Controller → UseCase
        - UseCase → Presenter → ViewModel
        - ViewModel(JSON) を返却
        """
        try:
            presenter = BookPresenter()           # 出力の受け手
            usecase.set_output_boundary(presenter) # OutputBoundary注入

            # Controllerは“翻訳担当”：HTTP → InputData
            controller.handle_request(title=body.title, author=body.author, isbn=body.isbn)

            # PresenterはViewModelを持つ：今回はそのままJSONで返す
            return presenter.get_view_model()

        except ValueError as e:
            # 例：ISBN重複など UseCaseの業務エラー → HTTP 400 に変換
            raise HTTPException(status_code=400, detail=str(e))

    return app
```

---

## ✳️ Composition Root（起動スクリプト）

### `main_fastapi.py`

```python
# --------------------------------------------------------------------
# [クラス図] Main（Composition Root）
# 役割：FastAPIアプリを生成（配線は app.create_app 内で実施）
# --------------------------------------------------------------------
import uvicorn
from infrastructure.web_fastapi.app import create_app

if __name__ == "__main__":
    uvicorn.run(create_app(), host="127.0.0.1", port=8000)
```

> 起動：
>
> ```bash
> python main_fastapi.py
> ```
>
> 動作確認（別端末などで）：
>
> ```bash
> curl -X POST http://127.0.0.1:8000/books \
>   -H "Content-Type: application/json" \
>   -d '{"title":"Refactoring","author":"Martin","isbn":"978-0134757599"}'
> ```

---

## ✅ ここがポイント（“外だけ入れ替えた”証拠）

* **変えていないもの**：

  * `domain/*`（Entity）
  * `usecase/*`（Interactor／Boundary／DTO）
  * `interface/controller/*`（Controller）
  * `interface/presenter/*`（Presenter）
  * `interface/gateway/*`（Gateway）
  * `infrastructure/db/*`（SQLite接続・行データの出し入れ）

* **変えたもの**：

  * **受付**だけ `http.server` → **FastAPI**
  * HTMLで返す代わりに、今回は **Presenterの ViewModel をJSONで返却**
    （= **View を「JSON表示」に置き換えた**だけ。HTMLにしたい場合は `HTMLResponse` を使って `render_book_list([vm])` を返せばOK）

> つまり、**ControllerもPresenterもGatewayもそのまま**。
> **“Webの受付（フレームワーク）”と“Viewの表現”** を差し替えただけで動く＝**アーキテクチャが効いている**状態です。

---

## 🧪 テストのヒント（FastAPI版）

* **APIテスト**（フレームワーク層）：`TestClient` を使って `/books` に POST → 200/400 を確認
* **内側のテストは従来どおり**：UseCaseの単体／Gatewayの往復などは一切変更なし

```python
# 例：framework層の最小テスト
from fastapi.testclient import TestClient
from infrastructure.web_fastapi.app import create_app

def test_register_book_api():
    client = TestClient(create_app())
    res = client.post("/books", json={"title":"DDD","author":"Evans","isbn":"123"})
    assert res.status_code == 201
    assert res.json()["title"] == "DDD"
```

---

## 📘 まとめ（入れ替えの学び）

* **入れ替えたのは最外層（Frameworks & Drivers＝Infrastructure）だけ**。
* **内側（UseCase／Controller／Presenter／Gateway）は不変**。
* JSON（API）で返すか、HTML（画面）で返すかは **Presenterの先（View）** の問題で、**入れ替え自在**。
* これが「依存は内へ」を守った設計の威力。

---

### 補足：HTMLで返したい場合の最小例（任意）

FastAPIでHTMLを返すなら、`fastapi.responses.HTMLResponse` を使います。
`render_book_list([vm])`（前章のView）をそのまま流用できます。

```python
from fastapi.responses import HTMLResponse
# ...
@app.post("/books/html", response_class=HTMLResponse)
def register_book_html(body: BookCreateBody):
    presenter = BookPresenter()
    usecase.set_output_boundary(presenter)
    controller.handle_request(title=body.title, author=body.author, isbn=body.isbn)
    vm = presenter.get_view_model()
    return render_book_list([vm])  # ← 以前のViewをそのまま利用
```

> **外側（フレームワーク / 表示形式）だけ差し替え**、内側はそのまま。
> “図どおり作れば、後から取り替えが効く” をここで実感できます。
