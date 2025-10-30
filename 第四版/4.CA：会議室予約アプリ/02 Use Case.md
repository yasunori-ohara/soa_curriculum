# 第2章：UseCase（ユースケース）

## 🎯 この章の目的

* **Entity（ルール）を使って、アプリの手続きを表現する**
* 「予約を作る」「空きを調べる」など、**人間が行う操作**をコード化する
* 外側（WebやDB）に依存せず、**使うデータ・返すデータ・保存の約束**を明確にする
* **依存性逆転（DIP：Dependency Inversion Principle）**を実感する
  → 「使う側（内側）が、使われる側（外側）を決める」

---

## 🧩 UseCaseの位置づけ（図対応）

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

| 図の箱（クラス図）                       | 役割                           | この章で作るファイル                                     |
| ------------------------------- | ---------------------------- | ---------------------------------------------- |
| **UseCaseInteractor**           | アプリの手続きを実行する中心               | `usecase/create_reservation.py`                |
| **InputBoundary（入口の約束）**        | Controller → UseCase の呼び出し規約 | `usecase/boundaries/reservation_boundaries.py` |
| **OutputBoundary（出口の約束）**       | UseCase → Presenter の呼び出し規約  | 同上                                             |
| **InputData／OutputData（データの箱）** | 層をまたいでやり取りするシンプルなデータ         | `usecase/dto.py`                               |
| **Repository Interface（保存の約束）** | データの保存・取得方法の約束               | `usecase/contracts/` 以下                        |

👉 UseCase層は「**アプリの中心の手続き**」を担当します。
👉 DBやWebのことは知らず、「**契約（約束）**」を通してやり取りします。

---

## 🧱 フォルダ構成（この章の範囲）

```
project_round3/
└── usecase/
    ├── dto.py
    ├── boundaries/
    │   └── reservation_boundaries.py
    ├── contracts/
    │   ├── reservation_repository.py
    │   ├── room_repository.py
    │   └── member_repository.py
    ├── create_reservation.py
    ├── cancel_reservation.py        ← 後で扱う
    └── search_availability.py       ← 後で扱う
```

---

## 💬 初学者向けミニ用語解説

| 用語                    | わかりやすい説明                                                |
| --------------------- | ------------------------------------------------------- |
| **DTO（データ転送オブジェクト）**  | 「データの箱」。入力データ（InputData）と出力データ（OutputData）をまとめて渡す入れ物です。 |
| **Boundary（バウンダリ）**   | 「境界」。UseCaseが他の層とやり取りするための**約束ごと**です。                   |
| **Repository（リポジトリ）** | 「保存の担当者」。UseCaseは“どんな方法で保存するか”を知らず、契約だけ決めます。            |
| **依存性逆転（DIP）**        | 「下の技術に依存しないで、上のルールが主導する」考え方。                            |

---

## 💡 設計方針

* **UseCase層は「現実のルールをどう使うか」を書く**
* **Entityは呼び出すだけ**（保存や通信の仕方は知らない）
* **外の層（Web, DB）は後で実装**し、今は**「約束（インターフェース）」**だけ決めておく
* **InputData／OutputData**は、外側とやり取りする「安全な箱」
* **Repository契約**を通して、外側の実装を自由に差し替え可能にする

---

## 🧠 コード（予約作成ユースケース）

---

### ✳️ `usecase/dto.py`（データの箱：InputData／OutputData）

```python
# --------------------------------------------------------------------
# [クラス図] InputData / OutputData（データ転送オブジェクト）
# [同心円] UseCase層
# 役割：入力や出力をまとめる「データの箱」。DBやWebに依存しない。
# --------------------------------------------------------------------
from dataclasses import dataclass
from datetime import datetime
from typing import List, Optional

# 予約を作るときの入力データ
@dataclass
class CreateReservationInput:
    room_id: int
    member_id: int
    start_at: datetime
    end_at: datetime
    attendees: int
    idempotency_key: Optional[str] = None  # 同じ予約を二重に作らないための鍵

# 予約作成の出力データ
@dataclass
class CreateReservationOutput:
    reservation_id: int
    room_id: int
    member_id: int
    start_at: datetime
    end_at: datetime
    attendees: int

# 空き検索の入力データ
@dataclass
class SearchAvailabilityInput:
    start_at: datetime
    end_at: datetime
    min_capacity: int = 1
    required_equipments: Optional[List[str]] = None

# 空き検索の出力データ
@dataclass
class AvailableRoomSummary:
    id: int
    name: str
    capacity: int

@dataclass
class SearchAvailabilityOutput:
    rooms: List[AvailableRoomSummary]
```

---

### ✳️ `usecase/contracts/reservation_repository.py`（保存の約束）

```python
# --------------------------------------------------------------------
# [クラス図] Data Access Interface（保存の約束）
# [同心円] UseCase層
# 役割：予約の保存・取得・重複チェックを行う契約（まだ実装しない）。
# --------------------------------------------------------------------
from abc import ABC, abstractmethod
from datetime import datetime
from typing import List, Optional
from domain.reservation import Reservation

class ReservationRepository(ABC):
    """予約データを保存・取得するための約束（抽象クラス）"""

    @abstractmethod
    def next_id(self) -> int:
        """新しいIDを発行する"""
        raise NotImplementedError

    @abstractmethod
    def save(self, reservation: Reservation) -> Reservation:
        """予約を保存する"""
        raise NotImplementedError

    @abstractmethod
    def find_overlaps(self, room_id: int, start_at: datetime, end_at: datetime) -> List[Reservation]:
        """指定された時間帯と重なる予約を探す"""
        raise NotImplementedError

    @abstractmethod
    def check_idempotency(self, key: str) -> Optional[int]:
        """同じ操作を二度行わないための重複チェック。すでに登録済みなら予約IDを返す"""
        raise NotImplementedError
```

---

### ✳️ `usecase/contracts/room_repository.py`（会議室の約束）

```python
# --------------------------------------------------------------------
# [クラス図] Data Access Interface（保存の約束）
# [同心円] UseCase層
# 役割：会議室の情報を取得するための契約。
# --------------------------------------------------------------------
from abc import ABC, abstractmethod
from typing import List, Optional
from domain.room import Room

class RoomRepository(ABC):
    @abstractmethod
    def find_by_id(self, room_id: int) -> Optional[Room]:
        """指定されたIDの会議室を取得"""
        raise NotImplementedError

    @abstractmethod
    def search_candidates(self, min_capacity: int, required_equipments: List[str]) -> List[Room]:
        """条件に合う会議室を検索"""
        raise NotImplementedError
```

---

### ✳️ `usecase/contracts/member_repository.py`（会員の約束）

```python
# --------------------------------------------------------------------
# [クラス図] Data Access Interface（保存の約束）
# [同心円] UseCase層
# 役割：会員情報の取得を行う契約。
# --------------------------------------------------------------------
from abc import ABC, abstractmethod
from typing import Optional
from domain.member import Member

class MemberRepository(ABC):
    @abstractmethod
    def find_by_id(self, member_id: int) -> Optional[Member]:
        """指定されたIDの会員を取得"""
        raise NotImplementedError
```

---

### ✳️ `usecase/boundaries/reservation_boundaries.py`（境界の取り決め）

```python
# --------------------------------------------------------------------
# [クラス図] Boundary（境界：InputBoundary / OutputBoundary）
# [同心円] UseCase層
# 役割：Controller・Presenterとのやり取りを定義する約束。
# --------------------------------------------------------------------
from typing import Protocol
from usecase.dto import (
    CreateReservationInput, CreateReservationOutput,
    SearchAvailabilityInput, SearchAvailabilityOutput
)

# Controller → UseCase
class CreateReservationInputBoundary(Protocol):
    def execute(self, inp: CreateReservationInput) -> None: ...

# UseCase → Presenter
class CreateReservationOutputBoundary(Protocol):
    def present(self, out: CreateReservationOutput) -> None: ...

# 空き検索用のBoundary
class SearchAvailabilityInputBoundary(Protocol):
    def execute(self, inp: SearchAvailabilityInput) -> None: ...

class SearchAvailabilityOutputBoundary(Protocol):
    def present(self, out: SearchAvailabilityOutput) -> None: ...
```

---

### ✳️ `usecase/create_reservation.py`（ユースケース本体）

```python
# --------------------------------------------------------------------
# [クラス図] UseCaseInteractor（予約作成）
# [同心円] UseCase層
# 役割：予約を作る手続きを実装。EntityとRepository契約だけを使う。
# --------------------------------------------------------------------
from typing import Optional
from domain.reservation import Reservation, overlaps, validate_time_order
from usecase.dto import CreateReservationInput, CreateReservationOutput
from usecase.contracts.reservation_repository import ReservationRepository
from usecase.contracts.room_repository import RoomRepository
from usecase.contracts.member_repository import MemberRepository
from usecase.boundaries.reservation_boundaries import (
    CreateReservationInputBoundary, CreateReservationOutputBoundary
)

class CreateReservationUseCase(CreateReservationInputBoundary):
    """予約を作成するユースケース"""

    def __init__(
        self,
        reservations: ReservationRepository,
        rooms: RoomRepository,
        members: MemberRepository,
    ):
        self.reservations = reservations
        self.rooms = rooms
        self.members = members
        self._presenter: Optional[CreateReservationOutputBoundary] = None

    def set_output_boundary(self, presenter: CreateReservationOutputBoundary) -> None:
        """Presenter（結果の受け取り役）を設定する"""
        self._presenter = presenter

    def execute(self, inp: CreateReservationInput) -> None:
        """予約作成のメイン処理"""
        assert self._presenter is not None, "OutputBoundaryが設定されていません。"

        # 1) 開始・終了の整合チェック
        validate_time_order(inp.start_at, inp.end_at)

        # 2) 会議室の存在と定員チェック
        room = self.rooms.find_by_id(inp.room_id)
        if room is None:
            raise ValueError("指定された会議室が存在しません。")
        if inp.attendees > room.capacity:
            raise ValueError("参加人数が定員を超えています。")

        # 3) 会員確認
        if self.members.find_by_id(inp.member_id) is None:
            raise ValueError("指定された会員が存在しません。")

        # 4) 同じ操作を二重に実行していないか確認（冪等性チェック）
        if inp.idempotency_key:
            existed = self.reservations.check_idempotency(inp.idempotency_key)
            if existed is not None:
                out = CreateReservationOutput(
                    reservation_id=existed,
                    room_id=inp.room_id,
                    member_id=inp.member_id,
                    start_at=inp.start_at,
                    end_at=inp.end_at,
                    attendees=inp.attendees,
                )
                self._presenter.present(out)
                return

        # 5) 時間帯の重なりチェック
        conflicts = self.reservations.find_overlaps(inp.room_id, inp.start_at, inp.end_at)
        if conflicts:
            raise ValueError("同じ時間帯にすでに予約があります。")

        # 6) 新しい予約を作成・保存
        rid = self.reservations.next_id()
        new_resv = Reservation(
            id=rid,
            room_id=inp.room_id,
            member_id=inp.member_id,
            start_at=inp.start_at,
            end_at=inp.end_at,
            attendees=inp.attendees,
        )
        saved = self.reservations.save(new_resv)

        # 7) 出力データをPresenterへ渡す
        out = CreateReservationOutput(
            reservation_id=saved.id,
            room_id=saved.room_id,
            member_id=saved.member_id,
            start_at=saved.start_at,
            end_at=saved.end_at,
            attendees=saved.attendees,
        )
        self._presenter.present(out)
```

---

## 🧪 テストのヒント（この章の範囲）

> WebもDBも不要です。
> 「メモリ上の仮の保存クラス（FakeRepository）」を使えばテストできます。

```python
# 例：usecase_create_reservation_test.py
from datetime import datetime, timedelta
from domain.reservation import Reservation
from domain.room import Room
from domain.member import Member
from usecase.create_reservation import CreateReservationUseCase
from usecase.dto import CreateReservationInput

class FakeRepo:
    def __init__(self):
        self._data = []
        self._next = 1
    def next_id(self): nid=self._next; self._next+=1; return nid
    def save(self, r): self._data.append(r); return r
    def find_overlaps(self, room_id, s, e): return []
    def check_idempotency(self, key): return None
    def find_by_id(self, *_): return None

class FakeRoomRepo:
    def find_by_id(self, id): return Room(id=id, name="A", capacity=5, equipments=set())

class FakeMemberRepo:
    def find_by_id(self, id): return Member(id=id, name="M", email="m@example.com")

class SpyPresenter:
    def __init__(self): self.captured = None
    def present(self, out): self.captured = out

def test_create_reservation_basic():
    uc = CreateReservationUseCase(FakeRepo(), FakeRoomRepo(), FakeMemberRepo())
    spy = SpyPresenter()
    uc.set_output_boundary(spy)

    s = datetime(2025, 1, 1, 10)
    e = s + timedelta(hours=1)
    uc.execute(CreateReservationInput(1, 1, s, e, 3))
    assert spy.captured is not None
    assert spy.captured.room_id == 1
```

---

## 📘 この章の理解ポイント（まとめ）

| 観点         | 内容                                        |
| ---------- | ----------------------------------------- |
| **役割**     | アプリの手続きを表す中心。Entityを使い、保存契約や入力出力の取り決めを行う。 |
| **依存先**    | Entity／Repository契約／Boundary（すべて抽象）       |
| **知らないもの** | DBの仕組み、Webの通信、画面の表示方法                     |
| **設計効果**   | 外の技術を変えても、手続きのロジックを変えずに済む。                |
| **教育効果**   | 「ルールを内側に、技術を外側に」という考え方を実感できる。             |

---

## ▶ 次のページ

**第3章：Interface Adapter（インターフェースアダプタ）**
ここでは、

* **Controller（受付役）**
* **Presenter（結果を整える役）**
* **Gateway（DBとの橋渡し役）**
  を作り、今作った UseCase を外の世界（WebやDB）とつなぎます。

> これで「依存は内へ、制御は外へ」の流れが、実際に動く形になります。
