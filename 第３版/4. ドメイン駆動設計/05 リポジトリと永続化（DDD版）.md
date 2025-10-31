## 第5章：リポジトリと永続化（DDD版）

この章は、これまでの「ドメインの世界（純粋ロジック）」を、
実際の**データ永続化（DB・ストレージ）**と接続する部分です。

ここで再び、**クリーンアーキテクチャの依存方向**と**DDDの集約設計**が融合します。

## 🎯 この章の目的

1. **リポジトリ（Repository）**のDDD的な位置づけを理解する
2. **集約単位での保存・取得**の方法を知る
3. **永続化層（DB）とドメイン層を疎結合に保つ**設計を学ぶ
4. **テストのためのInMemoryリポジトリ**を実装する

---

## 🧭 1. Repositoryとは何か

DDDでのリポジトリは、
「**ドメインの世界とデータの世界をつなぐ“翻訳者”**」です。

---

### 📘 定義

> **Repository** は、ドメインオブジェクト（集約）を
> 永続化層（DBやAPI）に保存・再構築するための“窓口（Gateway）”。

---

### 🧩 図：リポジトリの位置づけ

```
+----------------------------------------------------+
| [UseCase層]                                        |
|   ReservationUseCase → ReservationRepository       |
+----------------------------------------------------+
| [Domain層]                                         |
|   ReservationAggregate, ReservationFactory         |
+----------------------------------------------------+
| [Infrastructure層]                                 |
|   ReservationRepositoryImpl (DB実装, ORM, SQL等)   |
+----------------------------------------------------+
```

> 💡 リポジトリは、**UseCaseがドメインを操作するための抽象化レイヤ**です。
> **UseCaseはDBを知らずにドメインを扱える**ようになります。

---

## 🧩 2. 集約単位で保存・再構築する

リポジトリの単位は「**テーブルではなく集約**」です。
すなわち、

* `ReservationAggregate` 単位で `save()`
* `RoomAggregate` 単位で `find_by_id()`

というように、「ルートエンティティを中心に扱う」のが基本です。

---

### 📘 インターフェース（契約）例：ReservationRepository

```python
# usecase/contracts/reservation_repository.py
from abc import ABC, abstractmethod
from typing import Optional, List
from domain.reservation import Reservation

class ReservationRepository(ABC):
    """ReservationAggregate の保存・取得を行う契約（抽象インターフェース）"""

    @abstractmethod
    def next_id(self) -> int:
        """次のIDを払い出す"""
        raise NotImplementedError

    @abstractmethod
    def save(self, reservation: Reservation) -> None:
        """Reservationを保存する"""
        raise NotImplementedError

    @abstractmethod
    def find_by_id(self, reservation_id: int) -> Optional[Reservation]:
        """IDからReservationを復元する"""
        raise NotImplementedError

    @abstractmethod
    def find_by_room(self, room_id: int) -> List[Reservation]:
        """Roomごとの予約を取得する"""
        raise NotImplementedError
```

> 📖 DDDでは、**リポジトリのインターフェースはUseCase層（アプリケーション層）に置く**。
> なぜなら、「ドメインをどう扱うか」という契約は**アプリケーションロジックの関心事**だからです。

---

## ⚙ 3. 実装例：SQLiteによるInfrastructure層

```python
# infrastructure/db/reservation_repository_impl.py
from domain.reservation import Reservation
from domain.value_objects import TimeRange
from domain.room import Room, Capacity
from domain.member import Member, Email
from usecase.contracts.reservation_repository import ReservationRepository
import sqlite3
from datetime import datetime

DB_PATH = "app.db"

class ReservationRepositoryImpl(ReservationRepository):
    """ReservationのSQLite実装（Infrastructure層）"""

    def _connect(self):
        return sqlite3.connect(DB_PATH)

    def next_id(self) -> int:
        with self._connect() as conn:
            cur = conn.execute("SELECT COALESCE(MAX(id),0)+1 FROM reservations")
            (nid,) = cur.fetchone()
            return int(nid)

    def save(self, reservation: Reservation) -> None:
        with self._connect() as conn:
            conn.execute("""
                INSERT OR REPLACE INTO reservations
                (id, member_id, member_name, member_email,
                 room_id, room_name, capacity,
                 start_time, end_time, attendees, canceled_at)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            """, (
                reservation.id,
                reservation.member.id,
                reservation.member.name,
                reservation.member.email.value,
                reservation.room.id,
                reservation.room.name,
                reservation.room.capacity.value,
                reservation.time_range.start.isoformat(),
                reservation.time_range.end.isoformat(),
                reservation.attendees,
                reservation.canceled_at.isoformat() if reservation.canceled_at else None,
            ))

    def find_by_id(self, reservation_id: int):
        with self._connect() as conn:
            cur = conn.execute("SELECT * FROM reservations WHERE id=?", (reservation_id,))
            row = cur.fetchone()
            if not row:
                return None
            return self._row_to_entity(row)

    def find_by_room(self, room_id: int):
        with self._connect() as conn:
            cur = conn.execute("SELECT * FROM reservations WHERE room_id=?", (room_id,))
            return [self._row_to_entity(r) for r in cur.fetchall()]

    def _row_to_entity(self, row):
        (
            id, member_id, member_name, member_email,
            room_id, room_name, capacity,
            start, end, attendees, canceled_at
        ) = row
        return Reservation(
            id=id,
            member=Member(member_id, member_name, Email(member_email)),
            room=Room(room_id, room_name, Capacity(capacity)),
            time_range=TimeRange(datetime.fromisoformat(start), datetime.fromisoformat(end)),
            attendees=attendees,
            canceled_at=datetime.fromisoformat(canceled_at) if canceled_at else None
        )
```

> ✅ **ポイント：**
>
> * 永続化の詳細（SQL）はここに閉じ込める。
> * `ReservationRepositoryImpl` はドメインを知らずにSQLを扱う。
> * `Reservation` エンティティの再構築（再生）は `_row_to_entity()` にまとめる。

---

## 🧩 4. UseCaseとの接続（依存性逆転）

```python
# usecase/create_reservation.py
from usecase.contracts.reservation_repository import ReservationRepository
from domain.factory.reservation_factory import ReservationFactory

class CreateReservationUseCase:
    def __init__(self, repo: ReservationRepository, factory: ReservationFactory):
        self.repo = repo
        self.factory = factory

    def execute(self, member, room, time_range, attendees):
        new_id = self.repo.next_id()
        reservation = self.factory.create(new_id, member, room, time_range, attendees)
        self.repo.save(reservation)
        return reservation
```

> 💡 これで「**UseCaseはDBを知らない**」。
> `repo` に抽象的に依存しているだけなので、テスト時は差し替え可能です。

---

## 🧪 5. テスト用リポジトリ（InMemory）

```python
# tests/inmemory_reservation_repo.py
from usecase.contracts.reservation_repository import ReservationRepository

class InMemoryReservationRepository(ReservationRepository):
    def __init__(self):
        self._data = []
        self._next_id = 1

    def next_id(self):
        nid = self._next_id
        self._next_id += 1
        return nid

    def save(self, reservation):
        self._data.append(reservation)

    def find_by_id(self, reservation_id):
        for r in self._data:
            if r.id == reservation_id:
                return r
        return None

    def find_by_room(self, room_id):
        return [r for r in self._data if r.room.id == room_id]
```

> ✅ テストではこの「InMemory版」をDIすれば、DBがなくても動作確認可能。

---

## 📘 6. 契約テスト（Repositoryの保証）

> 「リポジトリの実装を差し替えても動作が変わらない」ことを保証するテストです。

```python
def test_repository_contract():
    from tests.inmemory_reservation_repo import InMemoryReservationRepository
    from infrastructure.db.reservation_repository_impl import ReservationRepositoryImpl
    from domain.factory.reservation_factory import ReservationFactory
    from domain.value_objects import TimeRange
    from domain.room import Room, Capacity
    from domain.member import Member, Email
    from datetime import datetime

    def run_test(repo):
        factory = ReservationFactory()
        member = Member(1, "Taro", Email("taro@example.com"))
        room = Room(1, "A", Capacity(10))
        time = TimeRange(datetime(2025,1,1,9), datetime(2025,1,1,10))

        res = factory.create(repo.next_id(), member, room, time, 5)
        repo.save(res)
        found = repo.find_by_id(res.id)
        assert found.member.email.value == "taro@example.com"

    run_test(InMemoryReservationRepository())
    run_test(ReservationRepositoryImpl())
```

> ✅ **Repository契約テスト**により、
> 永続化の仕組みが変わっても「ビジネスロジックが壊れない」ことを確認できます。

---

## 🧭 7. クリーンアーキテクチャとの統合図

```
[UseCase層]         ← CreateReservationUseCase
     ↓
[Repository契約]    ← ReservationRepository (抽象)
     ↓
[Infrastructure層]  ← ReservationRepositoryImpl (SQLite)
     ↑
[Domain層]          ← ReservationAggregate, Factory
```

> **依存方向（import）は内へ**、
> **制御の流れ（実行時）は外へ**。
> DDDでもこの原則はクリーンアーキと同じです。

---

## ✅ 8. まとめ

| 要点           | 内容                                            |
| ------------ | --------------------------------------------- |
| **リポジトリの単位** | 集約ごと（テーブルではない）                                |
| **責務**       | ドメインを永続化・再構築する（翻訳者）                           |
| **配置場所**     | インターフェース＝UseCase層／実装＝Infrastructure層          |
| **依存方向**     | UseCase → Repository(抽象) ← Implementation(具体) |
| **テスト**      | InMemory＋契約テストで動作保証                           |

---

## 🎓 演習①：自分のリポジトリを設計してみよう

| 集約名                  | リポジトリ名                | 主な操作                           | データソース   |
| -------------------- | --------------------- | ------------------------------ | -------- |
| ReservationAggregate | ReservationRepository | save, find_by_id, find_by_room | SQLite   |
| RoomAggregate        | RoomRepository        | save, find_all                 | JSONファイル |
| MemberAggregate      | MemberRepository      | save, find_by_email            | REST API |

---

## 🎓 演習②：テスト時の依存注入

| コンテキスト     | UseCase                  | Repository実装              | テスト時の置き換え                     |
| ---------- | ------------------------ | ------------------------- | ----------------------------- |
| Scheduling | CreateReservationUseCase | ReservationRepositoryImpl | InMemoryReservationRepository |
| Identity   | RegisterMemberUseCase    | MemberRepositoryImpl      | FakeMemberRepository          |

---

## 🔜 次章予告：第6章「DDD × クリーンアーキの総まとめ」

次章では、DDDの全パーツ（Entity／ValueObject／Aggregate／Service／Factory／Repository）を
クリーンアーキテクチャの層構造の中に**完全統合**して、
「DDD × Clean Architectureの完成モデル」を示します。

扱う内容：

* DDDとCAの完全対応マッピング
* DDD的フォルダ構成テンプレート
* 集約ごとの依存注入（Main）
* DDD版契約テストの全体構成

