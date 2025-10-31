# 03：Interface Adapter（インターフェースアダプタ）

**Controller／Presenter／Gateway をそろえて、UseCase を外の世界（Web／DB）とつなぐ“橋渡し層”**

---

## 🎯 この章の目的

* **Controller（コントローラ）**：外から来た入力（フォームやAPI引数）を **InputData（DTO：データの箱）** に詰めて **UseCase** を呼ぶ
* **Presenter（プレゼンター）**：UseCaseからの **OutputData** を **ViewModel（画面/JSON向け）** に整える
* **Gateway（ゲートウェイ）**：UseCaseの **Repository（保存の約束）** を **実装**（※DBの具体は次章の Infrastructure に委ねる）

> ポイント：**同心円図の“内側（UseCase）”に外の関心を入れない**ため、
> 外との翻訳は **この層** に集めます。

---

## 🧩 図との対応（どの箱にあたるか）

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

| 図の箱（クラス図）      | 役割                       | この章のファイル                                          |
| -------------- | ------------------------ | ------------------------------------------------- |
| **Controller** | 受付役：入力をDTOに詰めてUseCaseを呼ぶ | `interface/controller/reservation_controller.py`  |
| **Presenter**  | 整形役：OutputData→ViewModel | `interface/presenter/reservation_presenter.py`    |
| **Gateway**    | 橋渡し役：Repository契約の**実装** | `interface/gateway/*.py`（＋`data_access_ports.py`） |

> **Infrastructure（Web・DB・HTML）**は**次章**で登場します。
> 今章では **WebやDBの具体は書かず**、「橋渡しの形（ポート）」を決めます。

---

## 🧱 フォルダ構成（この章の範囲）

```
project_round3/
└── interface/
    ├── controller/
    │   └── reservation_controller.py
    ├── presenter/
    │   └── reservation_presenter.py
    └── gateway/
        ├── data_access_ports.py       ← ★ Infraが実装する“ポート”
        ├── reservation_gateway.py     ← Repository契約を実装
        ├── room_gateway.py
        └── member_gateway.py
```

> **設計のコツ**：
>
> * **data_access_ports.py** は **Protocol（プロトコル）** で**“Infraに期待する関数”**だけ定義
> * 次章の Infrastructure で **SQLite（など）実装**をこれに**適合**させます
>   ＝ **Infra → IA（このポート）** に依存するので、**内向き依存を守れる**（依存性逆転：DIP）

---

## 🧠 コード

### 1) Controller（入力→UseCase 呼び出し）

`interface/controller/reservation_controller.py`

```python
# --------------------------------------------------------------------
# [クラス図] Controller
# [同心円] Interface Adapters（橋渡し層）
# 役割：Webなど外からの入力を「DTO（InputData）」に詰めて UseCase を呼ぶ。
#       日付文字列→datetime 変換などの“入力整形”もここで。
# --------------------------------------------------------------------
from datetime import datetime
from typing import Optional
from usecase.dto import CreateReservationInput
from usecase.boundaries.reservation_boundaries import CreateReservationInputBoundary

class ReservationController:
    def __init__(self, create_usecase: CreateReservationInputBoundary):
        self.create_usecase = create_usecase

    def create(
        self,
        room_id: int,
        member_id: int,
        start_at_str: str,   # 例: "2025-10-19T10:00"
        end_at_str: str,     # 例: "2025-10-19T11:00"
        attendees: int,
        idempotency_key: Optional[str] = None,
        dt_format: str = "%Y-%m-%dT%H:%M",
    ) -> None:
        # 入力（文字列等）を UseCase が扱いやすい型に整える
        start_at = datetime.strptime(start_at_str, dt_format)
        end_at   = datetime.strptime(end_at_str, dt_format)

        inp = CreateReservationInput(
            room_id=room_id,
            member_id=member_id,
            start_at=start_at,
            end_at=end_at,
            attendees=attendees,
            idempotency_key=idempotency_key,
        )
        self.create_usecase.execute(inp)
```

**ポイント**

* **Controller は UseCase の具体型を知らない**（**Boundary**だけ知る）
* **日付のパースや入力検証（表層）はController**、**ビジネス検証（開始<終了、重複）はUseCase/Entity**

---

### 2) Presenter（OutputData→ViewModel）

`interface/presenter/reservation_presenter.py`

```python
# --------------------------------------------------------------------
# [クラス図] Presenter
# [同心円] Interface Adapters（橋渡し層）
# 役割：UseCaseの出力（OutputData）を View/JSON 向けの ViewModel に整える。
#       View自体（HTML生成）は次章の Infrastructure に委ねる。
# --------------------------------------------------------------------
from typing import Optional, Dict, List
from usecase.boundaries.reservation_boundaries import (
    CreateReservationOutputBoundary, SearchAvailabilityOutputBoundary
)
from usecase.dto import CreateReservationOutput, SearchAvailabilityOutput

class ReservationPresenter(CreateReservationOutputBoundary, SearchAvailabilityOutputBoundary):
    def __init__(self):
        self._vm_create: Optional[Dict] = None
        self._vm_search: Optional[List[Dict]] = None
        self._error: Optional[str] = None

    # ---- 成功時の表示用（作成） ----
    def present(self, out: CreateReservationOutput) -> None:
        # ViewModel（画面/JSONに出して支障ない形）
        self._vm_create = {
            "reservation_id": out.reservation_id,
            "room_id": out.room_id,
            "member_id": out.member_id,
            "start_at": out.start_at.isoformat(timespec="minutes"),
            "end_at": out.end_at.isoformat(timespec="minutes"),
            "attendees": out.attendees,
            "message": "予約が作成されました。",
        }

    # ---- 成功時の表示用（空き検索） ----
    def present_search(self, out: SearchAvailabilityOutput) -> None:
        self._vm_search = [
            {"id": r.id, "name": r.name, "capacity": r.capacity}
            for r in out.rooms
        ]

    # ---- エラー時の表示（例：UseCaseが ValueError を投げた） ----
    def present_error(self, message: str) -> None:
        self._error = message

    # ---- View から取り出すAPI（Web層で使う） ----
    def get_view_model_create(self) -> Optional[Dict]:
        return self._vm_create

    def get_view_model_search(self) -> Optional[List[Dict]]:
        return self._vm_search

    def get_error(self) -> Optional[str]:
        return self._error
```

**ポイント**

* **Presenter は“見せ方のための整形”だけ**（時刻の`isoformat`など）
* **画面（HTML生成）は次章の Infrastructure** で行います（Presenterの先＝View）

---

### 3) Gateway（Repository契約の実装：DBの具体はまだ書かない）

#### 3-1. Data Access “ポート”（Infraが実装する口）

`interface/gateway/data_access_ports.py`

```python
# --------------------------------------------------------------------
# [クラス図] Port（データアクセスの期待インターフェース）
# [同心円] Interface Adapters
# 役割：Infrastructure が実装すべき“口”を定義。
#       → IA はこの口に対して Gateway を実装。DBの具体は次章で差し込む。
# --------------------------------------------------------------------
from dataclasses import dataclass
from datetime import datetime
from typing import List, Optional, Protocol, Iterable

# --- 行レベルの素朴な入れ物（DBの1行相当） ---
@dataclass(frozen=True)
class ReservationRecord:
    id: int
    room_id: int
    member_id: int
    start_at_iso: str
    end_at_iso: str
    attendees: int

@dataclass(frozen=True)
class RoomRecord:
    id: int
    name: str
    capacity: int
    equipments_csv: str  # "projector,whiteboard" のように保持

@dataclass(frozen=True)
class MemberRecord:
    id: int
    name: str
    email: str

# --- Infra が満たすべき口（Protocol） ---
class ReservationDataAccessPort(Protocol):
    def insert(self, rec: ReservationRecord) -> None: ...
    def find_overlaps(self, room_id: int, start_at: datetime, end_at: datetime) -> List[ReservationRecord]: ...
    def next_id(self) -> int: ...
    def upsert_idempotency(self, key: str, reservation_id: int) -> None: ...
    def check_idempotency(self, key: str) -> Optional[int]: ...
    def find_by_id(self, reservation_id: int) -> Optional[ReservationRecord]: ...
    def delete(self, reservation_id: int) -> None: ...

class RoomDataAccessPort(Protocol):
    def find_by_id(self, room_id: int) -> Optional[RoomRecord]: ...
    def search_candidates(self, min_capacity: int, required_equipments: Iterable[str]) -> List[RoomRecord]: ...

class MemberDataAccessPort(Protocol):
    def find_by_id(self, member_id: int) -> Optional[MemberRecord]: ...
```

> **狙い**：
>
> * **Interface Adapters（この層）** が **Infrastructure** に**直接**依存しないよう、
>   **Infraが“実装すべき口（Protocol）”**をここで定義。
> * 次章で **SQLite実装**をこの **Port** に**適合**させて注入します。
>   → **依存は内へ**（Infra → IA への依存）を守れる。

---

#### 3-2. ReservationRepository の実装（Gateway）

`interface/gateway/reservation_gateway.py`

```python
# --------------------------------------------------------------------
# [クラス図] Gateway（Repository契約の実装）
# [同心円] Interface Adapters
# 役割：Entity ⇔ 行（Record）の相互変換を担い、UseCaseの契約を満たす。
#       実際のDBアクセスは data_access_ports の Port に委譲（次章で実装）。
# --------------------------------------------------------------------
from datetime import datetime
from typing import List, Optional
from domain.reservation import Reservation
from usecase.contracts.reservation_repository import ReservationRepository
from .data_access_ports import ReservationDataAccessPort, ReservationRecord

class ReservationRepositoryGateway(ReservationRepository):
    def __init__(self, dao: ReservationDataAccessPort):
        self.dao = dao

    def next_id(self) -> int:
        return self.dao.next_id()

    def save(self, reservation: Reservation) -> Reservation:
        rec = ReservationRecord(
            id=reservation.id,
            room_id=reservation.room_id,
            member_id=reservation.member_id,
            start_at_iso=reservation.start_at.isoformat(),
            end_at_iso=reservation.end_at.isoformat(),
            attendees=reservation.attendees,
        )
        self.dao.insert(rec)
        return reservation

    def find_overlaps(self, room_id: int, start_at: datetime, end_at: datetime) -> List[Reservation]:
        rows = self.dao.find_overlaps(room_id, start_at, end_at)
        return [
            Reservation(
                id=r.id, room_id=r.room_id, member_id=r.member_id,
                start_at=datetime.fromisoformat(r.start_at_iso),
                end_at=datetime.fromisoformat(r.end_at_iso),
                attendees=r.attendees,
            )
            for r in rows
        ]

    def check_idempotency(self, key: str) -> Optional[int]:
        return self.dao.check_idempotency(key)
```

---

#### 3-3. RoomRepository の実装（Gateway）

`interface/gateway/room_gateway.py`

```python
# --------------------------------------------------------------------
# [クラス図] Gateway（Repository契約の実装）
# [同心円] Interface Adapters
# 役割：Room の取得・検索。Record ⇔ Entity の変換。
# --------------------------------------------------------------------
from typing import List, Optional
from domain.room import Room
from usecase.contracts.room_repository import RoomRepository
from .data_access_ports import RoomDataAccessPort

class RoomRepositoryGateway(RoomRepository):
    def __init__(self, dao: RoomDataAccessPort):
        self.dao = dao

    def find_by_id(self, room_id: int) -> Optional[Room]:
        r = self.dao.find_by_id(room_id)
        if not r:
            return None
        equipments = set(filter(None, (e.strip() for e in r.equipments_csv.split(","))))
        return Room(id=r.id, name=r.name, capacity=r.capacity, equipments=equipments)

    def search_candidates(self, min_capacity: int, required_equipments: List[str]) -> List[Room]:
        rows = self.dao.search_candidates(min_capacity, required_equipments)
        rooms: List[Room] = []
        for r in rows:
            equipments = set(filter(None, (e.strip() for e in r.equipments_csv.split(","))))
            # 必要設備を満たすかの最終チェック（DBで完全にやれない場合の保険）
            if all(req in equipments for req in required_equipments):
                rooms.append(Room(id=r.id, name=r.name, capacity=r.capacity, equipments=equipments))
        return rooms
```

---

#### 3-4. MemberRepository の実装（Gateway）

`interface/gateway/member_gateway.py`

```python
# --------------------------------------------------------------------
# [クラス図] Gateway（Repository契約の実装）
# [同心円] Interface Adapters
# 役割：Member の取得。Record ⇔ Entity の変換。
# --------------------------------------------------------------------
from typing import Optional
from domain.member import Member
from usecase.contracts.member_repository import MemberRepository
from .data_access_ports import MemberDataAccessPort

class MemberRepositoryGateway(MemberRepository):
    def __init__(self, dao: MemberDataAccessPort):
        self.dao = dao

    def find_by_id(self, member_id: int) -> Optional[Member]:
        r = self.dao.find_by_id(member_id)
        if not r:
            return None
        return Member(id=r.id, name=r.name, email=r.email)
```

---

## 🧪 テストのヒント（この章の範囲）

* **Controller 単体**：

  * ダミーの UseCase（Spy 実装：`execute()` が呼ばれたら記録）に差し替え
  * 文字列日時→`datetime` 変換が正しいか
* **Presenter 単体**：

  * ダミーの `CreateReservationOutput` を渡して、`get_view_model_create()` の整形を確認
* **Gateway 単体**：

  * **Port のフェイク実装**（メモリ保持）を用意して往復変換を確認
  * Record ⇔ Entity の変換（時刻ISO文字列の往復）が崩れないか

> 次章（Infrastructure）で **SQLiteの DataAccess 実装（Port適合）** を作り、
> ここで作った **Gateway** に**依存性注入（DI）**します。

---

## 📘 この章の理解ポイント（まとめ）

| 観点             | 内容                                                     |
| -------------- | ------------------------------------------------------ |
| **Controller** | 入力を整え、**UseCase（Boundary）** を呼ぶ。                       |
| **Presenter**  | **OutputData** を **ViewModel** に整形（見せ方の責務）。            |
| **Gateway**    | **Repository（保存の約束）** を**実装**。Entity⇔Record 変換だけ担当。    |
| **Infra切り離し**  | **Port（Protocol）** で “Infraに期待する口” を定義 → 次章で実装注入。      |
| **依存の向き**      | **Infra → IA（Port） → UseCase**（内向き依存）、UseCaseは具体を知らない。 |

---

## ▶ 次のページ

**第4章：Infrastructure（技術層：Web／DB／View）**

* **FastAPI** で **Controllerの先＝受付** を作る
* **SQLite** で **DataAccessPort** の実装（重複検索・冪等キーの一意制御）
* **View（HTML/JSON）** で **Presenterの先＝表示** を作る
* すべてを **Main（Composition Root）** で組み立て、動く形にします。
