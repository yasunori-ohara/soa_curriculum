# 04：Infrastructure（インフラ：Web／DB／View）

**FastAPI（受付＝Controllerの先）＋ SQLite（DB）＋ HTML/JSON View（Presenterの先）**

---

## 🎯 この章の目的

* **Web受付（FastAPI）** を用意して **Controller の先** を実装する
* **SQLite** で **DataAccessPort**（第3章で定義）を**実装**して **Gateway の先** を作る
* **View（HTML/JSON）** で **Presenter の先** を用意する
* どれも **UseCase の内側は知らない**（＝外の技術を外側にまとめる）

---

## 🧩 図との対応（どの箱にあたるか）

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

| 図の箱（クラス図）                | 役割                    | この章のファイル                              |
| ------------------------ | --------------------- | ------------------------------------- |
| **View（Presenterの先）**    | 画面（HTML）やAPI（JSON）を作る | `infrastructure/web/view_renderer.py` |
| **Webサーバ（Controllerの先）** | 受付（HTTP）／ルーティング       | `infrastructure/web/fastapi_app.py`   |
| **Data Access（DBの前段）**   | 行の出し入れ・検索             | `infrastructure/db/*_data_access.py`  |
| **Database**             | SQLite本体・DDL          | `infrastructure/db/sqlite_conn.py`    |

> ここで**DBクエリ・トランザクション・ユニーク制約**等の**技術詳細**を外側に封じ込め、
> **UseCase／Gateway は変更不要**にします（**依存は内へ**）。

---

## 🧱 フォルダ構成（この章で追加）

```
project_root/
└── infrastructure/
    ├── db/
    │   ├── sqlite_conn.py
    │   ├── reservation_data_access.py
    │   ├── room_data_access.py
    │   └── member_data_access.py
    └── web/
        ├── fastapi_app.py
        └── view_renderer.py
```

---

## 💡 技術方針（初学者向けの要点）

* **SQLite**：まずは**単体アプリ**で動かしやすいDB。

  * **時間帯重複の検索**…SQL の `NOT (end <= ? OR start >= ?)` で**区間が重ならない**条件の否定
  * **冪等キー**…`idempotency` テーブルに **UNIQUE(key)**
* **FastAPI**：JSONフォームの受付を最小限で。Presenterの ViewModel を **HTML か JSON** に整形して返す。
* **トランザクション**：作成時は **BEGIN IMMEDIATE**（SQLite）で簡易排他。
* **依存注入（DI）**：**Main（次章）** から Controller／UseCase／PresenterFactory／DataAccess を差し込める作り。

---

## 🧠 実装（DB）

### 1) SQLite 接続と DDL（Database）

`infrastructure/db/sqlite_conn.py`

```python
# --------------------------------------------------------------------
# [クラス図] Database
# [同心円] Frameworks & Drivers（Infrastructure）
# 役割：SQLite 接続とテーブル作成
# --------------------------------------------------------------------
import sqlite3
from pathlib import Path

DB_PATH = Path("app_round3.db")

DDL = """
PRAGMA journal_mode = WAL;

CREATE TABLE IF NOT EXISTS rooms (
  id        INTEGER PRIMARY KEY,
  name      TEXT NOT NULL,
  capacity  INTEGER NOT NULL,
  equipments_csv TEXT NOT NULL DEFAULT ''
);

CREATE TABLE IF NOT EXISTS members (
  id    INTEGER PRIMARY KEY,
  name  TEXT NOT NULL,
  email TEXT NOT NULL UNIQUE
);

-- 時刻は ISO8601 文字列で保存（例: 2025-10-19T10:00:00）
CREATE TABLE IF NOT EXISTS reservations (
  id         INTEGER PRIMARY KEY,
  room_id    INTEGER NOT NULL,
  member_id  INTEGER NOT NULL,
  start_at   TEXT NOT NULL,
  end_at     TEXT NOT NULL,
  attendees  INTEGER NOT NULL,
  FOREIGN KEY(room_id) REFERENCES rooms(id),
  FOREIGN KEY(member_id) REFERENCES members(id)
);

-- 冪等キー：同じ操作の二重実行を防ぐための鍵→予約ID
CREATE TABLE IF NOT EXISTS idempotency (
  key TEXT PRIMARY KEY,
  reservation_id INTEGER NOT NULL,
  FOREIGN KEY(reservation_id) REFERENCES reservations(id)
);

-- 検索を速くするためのインデックス
CREATE INDEX IF NOT EXISTS idx_resv_room ON reservations(room_id);
CREATE INDEX IF NOT EXISTS idx_resv_time ON reservations(start_at, end_at);
"""

def get_connection() -> sqlite3.Connection:
    conn = sqlite3.connect(DB_PATH, isolation_level=None)  # autocommit (手動でBEGIN)
    conn.execute("PRAGMA foreign_keys = ON;")
    return conn

def init_db() -> None:
    with get_connection() as conn:
        conn.executescript(DDL)
```

---

### 2) 予約の DataAccess（Port 実装）

`infrastructure/db/reservation_data_access.py`

```python
# --------------------------------------------------------------------
# [クラス図] Data Access（DB前段：行の出入 & 検索）
# [同心円] Frameworks & Drivers
# 役割：ReservationDataAccessPort を SQLite で実装
# --------------------------------------------------------------------
from datetime import datetime
from typing import List, Optional
from .sqlite_conn import get_connection
from interface.gateway.data_access_ports import (
    ReservationDataAccessPort, ReservationRecord
)

class SqliteReservationDataAccess(ReservationDataAccessPort):
    def next_id(self) -> int:
        with get_connection() as conn:
            cur = conn.execute("SELECT COALESCE(MAX(id), 0) + 1 FROM reservations")
            (nid,) = cur.fetchone()
            return int(nid)

    def insert(self, rec: ReservationRecord) -> None:
        with get_connection() as conn:
            conn.execute("BEGIN IMMEDIATE")  # 予約作成は競合しやすいので早めにロック
            conn.execute(
                """
                INSERT INTO reservations (id, room_id, member_id, start_at, end_at, attendees)
                VALUES (?, ?, ?, ?, ?, ?)
                """,
                (rec.id, rec.room_id, rec.member_id, rec.start_at_iso, rec.end_at_iso, rec.attendees),
            )
            conn.execute("COMMIT")

    def find_overlaps(self, room_id: int, start_at: datetime, end_at: datetime) -> List[ReservationRecord]:
        s = start_at.isoformat()
        e = end_at.isoformat()
        # 区間重複: NOT (end <= s OR start >= e)
        with get_connection() as conn:
            cur = conn.execute(
                """
                SELECT id, room_id, member_id, start_at, end_at, attendees
                FROM reservations
                WHERE room_id = ?
                  AND NOT (end_at <= ? OR start_at >= ?)
                ORDER BY start_at
                """,
                (room_id, s, e),
            )
            rows = cur.fetchall()
        return [
            ReservationRecord(
                id=r[0], room_id=r[1], member_id=r[2],
                start_at_iso=r[3], end_at_iso=r[4], attendees=r[5]
            )
            for r in rows
        ]

    def check_idempotency(self, key: str) -> Optional[int]:
        with get_connection() as conn:
            cur = conn.execute("SELECT reservation_id FROM idempotency WHERE key = ?", (key,))
            row = cur.fetchone()
            return None if not row else int(row[0])

    def upsert_idempotency(self, key: str, reservation_id: int) -> None:
        with get_connection() as conn:
            # SQLite の UPSERT（ON CONFLICT DO NOTHING/UPDATE）
            conn.execute(
                """
                INSERT INTO idempotency(key, reservation_id)
                VALUES (?, ?)
                ON CONFLICT(key) DO UPDATE SET reservation_id=excluded.reservation_id
                """,
                (key, reservation_id),
            )

    def find_by_id(self, reservation_id: int) -> Optional[ReservationRecord]:
        with get_connection() as conn:
            cur = conn.execute(
                "SELECT id, room_id, member_id, start_at, end_at, attendees FROM reservations WHERE id=?",
                (reservation_id,),
            )
            r = cur.fetchone()
            if not r:
                return None
            return ReservationRecord(
                id=r[0], room_id=r[1], member_id=r[2],
                start_at_iso=r[3], end_at_iso=r[4], attendees=r[5]
            )

    def delete(self, reservation_id: int) -> None:
        with get_connection() as conn:
            conn.execute("DELETE FROM reservations WHERE id = ?", (reservation_id,))
```

---

### 3) 会議室の DataAccess

`infrastructure/db/room_data_access.py`

```python
# --------------------------------------------------------------------
# [クラス図] Data Access（Room）
# [同心円] Frameworks & Drivers
# 役割：RoomDataAccessPort を SQLite で実装
# --------------------------------------------------------------------
from typing import List, Optional, Iterable
from .sqlite_conn import get_connection
from interface.gateway.data_access_ports import RoomDataAccessPort, RoomRecord

class SqliteRoomDataAccess(RoomDataAccessPort):
    def find_by_id(self, room_id: int) -> Optional[RoomRecord]:
        with get_connection() as conn:
            cur = conn.execute(
                "SELECT id, name, capacity, equipments_csv FROM rooms WHERE id = ?",
                (room_id,),
            )
            r = cur.fetchone()
            if not r:
                return None
            return RoomRecord(id=r[0], name=r[1], capacity=r[2], equipments_csv=r[3])

    def search_candidates(self, min_capacity: int, required_equipments: Iterable[str]) -> List[RoomRecord]:
        # シンプル版：容量だけSQLで絞り、設備は呼び出し側（Gateway）で最終確認
        with get_connection() as conn:
            cur = conn.execute(
                "SELECT id, name, capacity, equipments_csv FROM rooms WHERE capacity >= ? ORDER BY capacity DESC",
                (min_capacity,),
            )
            rows = cur.fetchall()
        return [RoomRecord(id=r[0], name=r[1], capacity=r[2], equipments_csv=r[3]) for r in rows]
```

---

### 4) 会員の DataAccess

`infrastructure/db/member_data_access.py`

```python
# --------------------------------------------------------------------
# [クラス図] Data Access（Member）
# [同心円] Frameworks & Drivers
# 役割：MemberDataAccessPort を SQLite で実装
# --------------------------------------------------------------------
from typing import Optional
from .sqlite_conn import get_connection
from interface.gateway.data_access_ports import MemberDataAccessPort, MemberRecord

class SqliteMemberDataAccess(MemberDataAccessPort):
    def find_by_id(self, member_id: int) -> Optional[MemberRecord]:
        with get_connection() as conn:
            cur = conn.execute(
                "SELECT id, name, email FROM members WHERE id = ?",
                (member_id,),
            )
            r = cur.fetchone()
            if not r:
                return None
            return MemberRecord(id=r[0], name=r[1], email=r[2])
```

---

## 🧠 実装（View）

`infrastructure/web/view_renderer.py`

```python
# --------------------------------------------------------------------
# [クラス図] View（Presenter の先：表示担当）
# [同心円] Frameworks & Drivers
# 役割：Presenter が渡す ViewModel（辞書/配列）を HTML に整形（最小限）。
# --------------------------------------------------------------------
from typing import Dict, List

def render_create_result(vm: Dict) -> str:
    return f"""
<!doctype html>
<html><head><meta charset="utf-8"><title>Reservation Created</title></head>
<body>
  <h1>予約が作成されました</h1>
  <ul>
    <li>ID: {vm['reservation_id']}</li>
    <li>Room: {vm['room_id']}</li>
    <li>Member: {vm['member_id']}</li>
    <li>Start: {vm['start_at']}</li>
    <li>End: {vm['end_at']}</li>
    <li>Attendees: {vm['attendees']}</li>
  </ul>
  <a href="/">戻る</a>
</body></html>
""".strip()

def render_error(message: str) -> str:
    return f"""
<!doctype html>
<html><head><meta charset="utf-8"><title>Error</title></head>
<body>
  <h1>エラー</h1>
  <p>{message}</p>
  <a href="/">戻る</a>
</body></html>
""".strip()
```

---

## 🧠 実装（Web受付：FastAPI）

`infrastructure/web/fastapi_app.py`

```python
# --------------------------------------------------------------------
# [クラス図] Webサーバ（Controller の先：受付）
# [同心円] Frameworks & Drivers
# 役割：HTTP を受けて Controller を呼ぶ。Presenter の ViewModel を HTML/JSON にして返す。
#       Controller/UseCase/PresenterFactory は Main（次章）から差し込む（DI）。
# --------------------------------------------------------------------
from fastapi import FastAPI, Depends, HTTPException, Query
from pydantic import BaseModel, Field
from typing import Callable, Optional
from interface.controller.reservation_controller import ReservationController
from interface.presenter.reservation_presenter import ReservationPresenter
from .view_renderer import render_create_result, render_error

app = FastAPI(title="Round3 Reservation")

# --- DI（Mainから差し込み） ---
_controller: Optional[ReservationController] = None
_presenter_factory: Optional[Callable[[], ReservationPresenter]] = None

def set_controller(controller: ReservationController) -> None:
    global _controller
    _controller = controller

def set_presenter_factory(factory: Callable[[], ReservationPresenter]) -> None:
    global _presenter_factory
    _presenter_factory = factory

def get_controller() -> ReservationController:
    if _controller is None:
        raise RuntimeError("Controllerが未設定です（MainでDIしてください）")
    return _controller

def new_presenter() -> ReservationPresenter:
    if _presenter_factory is None:
        raise RuntimeError("PresenterFactoryが未設定です（MainでDIしてください）")
    return _presenter_factory()

# --- 入力スキーマ（JSON） ---
class CreateReservationBody(BaseModel):
    room_id: int = Field(..., ge=1)
    member_id: int = Field(..., ge=1)
    start_at: str = Field(..., examples=["2025-10-19T10:00"])
    end_at: str   = Field(..., examples=["2025-10-19T11:00"])
    attendees: int = Field(..., ge=1)
    idempotency_key: Optional[str] = None

# --- API: JSON で作成（/api） ---
@app.post("/api/reservations")
def api_create_reservation(
    body: CreateReservationBody,
    controller: ReservationController = Depends(get_controller),
):
    presenter = new_presenter()
    try:
        # Controller が DTO を作って UseCase を呼ぶ
        controller.create(
            room_id=body.room_id,
            member_id=body.member_id,
            start_at_str=body.start_at,
            end_at_str=body.end_at,
            attendees=body.attendees,
            idempotency_key=body.idempotency_key,
        )
        vm = presenter.get_view_model_create()
        if vm is None:
            raise RuntimeError("Presenterに結果がありません。")
        return {"ok": True, "data": vm}
    except ValueError as e:
        presenter.present_error(str(e))
        raise HTTPException(status_code=400, detail=str(e))

# --- 画面向け（HTML） ---
@app.post("/reservations")
def create_reservation_html(
    room_id: int = Query(..., ge=1),
    member_id: int = Query(..., ge=1),
    start_at: str = Query(...),
    end_at: str = Query(...),
    attendees: int = Query(..., ge=1),
    idempotency_key: Optional[str] = Query(None),
    controller: ReservationController = Depends(get_controller),
):
    presenter = new_presenter()
    try:
        controller.create(
            room_id=room_id,
            member_id=member_id,
            start_at_str=start_at,
            end_at_str=end_at,
            attendees=attendees,
            idempotency_key=idempotency_key,
        )
        vm = presenter.get_view_model_create()
        return render_create_result(vm)  # HTML
    except ValueError as e:
        return render_error(str(e))

@app.get("/healthz")
def healthz():
    return {"ok": True}
```

> **ポイント**
>
> * **fastapi_app** は **Controller／PresenterFactory を受け取るだけ**。
>   **UseCase や Gateway を“直接”知らない**（= 内向き依存を守る）
> * **JSON と HTML** を**別エンドポイント**で用意（Presenterの ViewModel を両対応で使い回し）

---

## 🧪 テストのヒント（この章の範囲）

* **DB 初期化**：`init_db()` 実行 → 予約・会員・部屋をテストデータ投入
* **DataAccess 単体**：

  * `find_overlaps`…重複/非重複パターンを作って期待件数になるか
  * `idempotency`…同じキーで 2 回 `upsert` しても 1 つに束ねられるか
* **Web（FastAPI TestClient）＋ Presenter/Controller の結合**：

  * Mainを仮組み（次章の配線コードをテスト用に少し短縮）して `/api/reservations` へ POST
  * 期待の JSON、重複時は 400 が返るか

---

## 📘 まとめ（この章の理解ポイント）

| 観点                | 内容                                                                             |
| ----------------- | ------------------------------------------------------------------------------ |
| **Controller の先** | **FastAPI** が受付。Controller を呼び、Presenter の ViewModel を返す（HTML/JSON）。           |
| **Gateway の先**    | **SQLite DataAccess** が **Port** を実装。重複検索・冪等キー・採番を担当。                          |
| **Presenter の先**  | **View** が **ViewModel** を HTML に整形（テンプレ最小）。                                   |
| **依存の向き**         | **Infra → IA（Port/Gateway, Controller, Presenter） → UseCase → Entity**（常に内向き）。 |
| **拡張性**           | DBやWebの技術を差し替えても、UseCaseはそのまま。                                                 |

---

## ▶ 次のページ

**第5章：Main（Composition Root）**

* いよいよ **配線（依存注入）** を一箇所に集約し、

  * DataAccess(SQLIte) → Gateway
  * Gateway → UseCase
  * UseCase → Presenter
  * Controller → UseCase
  * FastAPI へ Controller/PresenterFactory を注入
* **動くアプリ**として起動できる形に仕上げます。
