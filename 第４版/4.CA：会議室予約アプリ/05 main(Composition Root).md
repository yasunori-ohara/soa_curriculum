# 05：Main（Composition Root：配線／依存注入）

**ここで全部つなぎます！**
Interface Adapter（Controller／Presenter／Gateway）と Infrastructure（FastAPI／SQLite）を、
**同心円のルールに従って一箇所で配線**し、アプリが実際に動く形にします。

---

## 🧩 図との対応（どの箱にあたるか）

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

| 図の箱（クラス図）                  | 役割                    | この章のファイル  |
| -------------------------- | --------------------- | --------- |
| **Main（Composition Root）** | 依存注入（DI）：外→内のつながりを一元化 | `main.py` |

> **ポイント**：Mainは**唯一**、外側（Infra）と内側（UseCase／Entity）を**同時に知ってよい**場所です。
> それ以外のクラスは自分の一つ外までしか知りません（＝**依存は内へ**）。

---

## 🧱 フォルダ（関係する箇所のおさらい）

```
project_root/
├── domain/ ...
├── usecase/
│   └── create_reservation.py     ← UseCase（具体型）
├── interface/
│   ├── controller/reservation_controller.py
│   ├── presenter/reservation_presenter.py
│   └── gateway/
│       ├── reservation_gateway.py
│       ├── room_gateway.py
│       └── member_gateway.py
└── infrastructure/
    ├── db/
    │   ├── sqlite_conn.py
    │   ├── reservation_data_access.py
    │   ├── room_data_access.py
    │   └── member_data_access.py
    └── web/fastapi_app.py        ← FastAPIアプリ本体（受付）
```

---

## 🔧 小さな前準備（Presenter の“差し込み先”を用意）

**Why?**
UseCase は処理結果を **Presenter（OutputBoundary）** に渡します。
Presenter は **リクエストごとに新しく**作るのが安全です（状態の取り違え防止）。
そのため、**FastAPI 側（受付）**で「今のリクエスト用 Presenter を UseCase にセット」できる**差し込み口**が必要です。

### fastapi_app.py に 1 点だけ足します（追記）：

```python
# infrastructure/web/fastapi_app.py （追記部分だけ抜粋）
# --- これをファイル先頭のDIブロックに追加 ---
_presenter_binder = None  # type: Optional[Callable[[ReservationPresenter], None]]

def set_presenter_binder(binder: Callable[["ReservationPresenter"], None]) -> None:
    global _presenter_binder
    _presenter_binder = binder

def bind_presenter(p: "ReservationPresenter") -> None:
    if _presenter_binder is None:
        raise RuntimeError("PresenterBinderが未設定です（MainでDIしてください）")
    _presenter_binder(p)
```

そして、エンドポイント内で **Presenter を作った直後**にこれを呼びます：

```python
# /api/reservations の直前で
presenter = new_presenter()
bind_presenter(presenter)  # ★ この1行を追加：UseCaseにPresenterを差し込む
# その後 controller.create(...) を呼ぶ
```

> これで **Controller は UseCase の具体型を知らなくてOK**、
> **FastAPI（受付）**が「今回のPresenterをUseCaseにセット」できます。
> （= **“制御は外へ”** を保ったまま、**“依存は内へ”** を破りません）

---

## 🧠 Main（配線コード）

`main.py`

```python
# --------------------------------------------------------------------
# [クラス図] Main（Composition Root）
# [同心円] ルール外（全層の組み立て専用）
# 役割：依存注入（DI）を一箇所に集約。技術（Infra）と手続き（UseCase）を接続して起動する。
# --------------------------------------------------------------------
from typing import Callable
from infrastructure.db.sqlite_conn import init_db, get_connection
from infrastructure.db.reservation_data_access import SqliteReservationDataAccess
from infrastructure.db.room_data_access import SqliteRoomDataAccess
from infrastructure.db.member_data_access import SqliteMemberDataAccess

from interface.gateway.reservation_gateway import ReservationRepositoryGateway
from interface.gateway.room_gateway import RoomRepositoryGateway
from interface.gateway.member_gateway import MemberRepositoryGateway

from usecase.create_reservation import CreateReservationUseCase
from interface.controller.reservation_controller import ReservationController
from interface.presenter.reservation_presenter import ReservationPresenter

# FastAPI アプリとDI用ファクトリ
from infrastructure.web.fastapi_app import (
    app,
    set_controller,
    set_presenter_factory,
    set_presenter_binder,
)

import uvicorn


# ---- データ投入（デモ用：最低限の部屋/会員を用意） --------------------------
def seed_if_empty() -> None:
    with get_connection() as conn:
        # rooms
        cur = conn.execute("SELECT COUNT(1) FROM rooms")
        (cnt_rooms,) = cur.fetchone()
        if cnt_rooms == 0:
            conn.executemany(
                "INSERT INTO rooms(id, name, capacity, equipments_csv) VALUES (?, ?, ?, ?)",
                [
                    (1, "A会議室", 6, "projector,whiteboard"),
                    (2, "B会議室", 10, "whiteboard"),
                ],
            )
        # members
        cur = conn.execute("SELECT COUNT(1) FROM members")
        (cnt_members,) = cur.fetchone()
        if cnt_members == 0:
            conn.executemany(
                "INSERT INTO members(id, name, email) VALUES (?, ?, ?)",
                [
                    (1, "山田太郎", "taro@example.com"),
                    (2, "鈴木花子", "hanako@example.com"),
                ],
            )


# ---- Composition Root ------------------------------------------------
def compose() -> None:
    # 1) DB 初期化（Infra層）
    init_db()
    seed_if_empty()

    # 2) DataAccess を作る（Infra → IA の Port 実装）
    resv_dao  = SqliteReservationDataAccess()
    room_dao  = SqliteRoomDataAccess()
    mem_dao   = SqliteMemberDataAccess()

    # 3) Gateway を作る（IA層：Repository契約の実装）
    resv_repo = ReservationRepositoryGateway(resv_dao)
    room_repo = RoomRepositoryGateway(room_dao)
    mem_repo  = MemberRepositoryGateway(mem_dao)

    # 4) UseCase を作る（UseCase層：手続きの中心）
    create_reservation_uc = CreateReservationUseCase(
        reservations=resv_repo,
        rooms=room_repo,
        members=mem_repo,
    )

    # 5) Controller を作る（IA層）
    controller = ReservationController(create_usecase=create_reservation_uc)

    # 6) Presenter のファクトリ（毎リクエスト新規に作る）
    def presenter_factory() -> ReservationPresenter:
        return ReservationPresenter()

    # 7) 「今回のPresenterをUseCaseへ差し込む」Binder
    def presenter_binder(p: ReservationPresenter) -> None:
        # ここで“具体型”のUseCaseに Presenter(OutputBoundary) をセット
        create_reservation_uc.set_output_boundary(p)

    # 8) FastAPI 側へ DI（受付＝Controllerの先）
    set_controller(controller)
    set_presenter_factory(presenter_factory)
    set_presenter_binder(presenter_binder)


def main() -> None:
    compose()
    # Uvicornで起動（開発用）
    uvicorn.run(app, host="127.0.0.1", port=8000, log_level="info")


if __name__ == "__main__":
    main()
```

---

## ▶ 起動と動作確認

1. 依存インストール（例）

```
pip install fastapi uvicorn "pydantic<3"
```

2. 起動

```
python main.py
```

3. 作成API（JSON）

```
POST http://127.0.0.1:8000/api/reservations
{
  "room_id": 1,
  "member_id": 1,
  "start_at": "2025-10-19T10:00",
  "end_at":   "2025-10-19T11:00",
  "attendees": 4,
  "idempotency_key": "req-123"
}
```

4. 画面（HTML）

```
POST http://127.0.0.1:8000/reservations?room_id=1&member_id=1&start_at=2025-10-19T10:00&end_at=2025-10-19T11:00&attendees=4
```

> 2回目を**同じ `idempotency_key`** で投げると、
> **同じ予約ID**の結果が返る（＝**同じ操作を2回しても結果は変わらない**）。

---

## 🧪 テストのヒント（Main／配線）

* **E2E（最小）**：`compose()` を呼んだテスト用ブート関数を用意し、
  `fastapi.testclient.TestClient(app)` で `/api/reservations` に POST

  * 初回：201/200 & 予約IDが返る
  * 2回目（同じ鍵）：**同じID**が返る
  * 重複時間で別予約：**400** が返る
* **代替実装差し替え**：`SqliteReservationDataAccess` を **メモリ版**に置き換えても、
  UseCase／Controller／Presenter は**無変更**でテストが通ることを確認（＝**依存性逆転（DIP）**の効果）

---

## 📘 まとめ（この章の理解ポイント）

| 観点                    | 内容                                                   |
| --------------------- | ---------------------------------------------------- |
| **Mainの責務**           | **配線だけ**。全層のオブジェクト生成とDIを一箇所に集める。                     |
| **Presenterのライフサイクル** | **毎リクエスト新規**が基本。**Binder**でUseCaseへ“その都度”差し込む。       |
| **依存の向き**             | **Infra → IA → UseCase → Entity**（内向き）。Mainだけが全体を知る。 |
| **変更に強い**             | DBやWebの技術が変わっても、**UseCaseのコードはそのまま**。                |

---

## ▶ 次のページ（任意の発展）

* **第6章：理論編（静的依存と動的制御の違い）**
* **空き検索／予約取消ユースケースの追加**
* **Repository契約の“契約テスト”テンプレート**（メモリ実装とSQLite実装の**同等性を確認**）

必要でしたら、このまま **契約テストの雛形** と **空き検索ユースケース** の実装に進みます。
