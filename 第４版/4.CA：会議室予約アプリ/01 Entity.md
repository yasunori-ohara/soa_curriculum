# 01：Entity（エンティティ）

## 🎯 この章の目的

* 「**会議室（Room）**」「**予約（Reservation）**」「**会員（Member）**」という**現実のルール**をクラスで表現する
* 「**時間が重ならない**」「**開始 < 終了**」「**定員を超えない**」といった**純粋ロジック**を、他の層（Web/DB）に依存せず書く
* 以降の **UseCase（ユースケース）** から**呼びやすい形**にする（テストが簡単になる）

---

## 🧩 Entityの位置づけ（図対応）

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

| 図の箱（クラス図）  | 役割                              | この章のファイル                                                      |
| ---------- | ------------------------------- | ------------------------------------------------------------- |
| **Entity** | 現実のビジネスルールの中心。外の技術（Web/DB）を知らない | `domain/reservation.py`, `domain/room.py`, `domain/member.py` |

👉 **依存は内へ**：Entityは**誰にも依存しない**層です（標準ライブラリのみ）。
👉 **実行の流れ（制御）は外から内へ**：後で出てくるUseCaseが、このEntityを**使う側**になります。

---

## 🧱 フォルダ構成（この章の範囲）

```
project_root/
└── domain/
    ├── reservation.py   ← 予約のルール（時間の重なり・開始/終了の検証）
    ├── room.py          ← 会議室の属性（定員・設備）
    └── member.py        ← 会員（予約者）
```

---

## 💡 設計方針（初学者向けの要点）

* **標準ライブラリだけ**を使う（`datetime`, `dataclasses` など）。外部ライブラリは使いません。
* **ルールは関数に切り出す**：テストしやすく、再利用もしやすい。
* **開始 < 終了** は予約の大前提。**時間帯の重なり**は `[start, end)`（**終端は開く**）で判定すると実務上の混乱が少ない。
* **定員チェック**は本来UseCase側で「人数 > 定員ならエラー」と使いますが、**基礎の属性**はEntityに置きます。

---

## 🧠 コード

### ✳️ `domain/reservation.py`（予約：時間ルールの中心）

```python
# --------------------------------------------------------------------
# [クラス図] Entity: Reservation（予約）
# [同心円] Entity層（現実のルール／純粋ロジック）
# 役割：会議室予約を表す。開始/終了の整合、時間帯の重なり判定を提供。
# --------------------------------------------------------------------
from dataclasses import dataclass
from datetime import datetime
from typing import Tuple

@dataclass(frozen=True)
class Reservation:
    id: int
    room_id: int
    member_id: int
    start_at: datetime
    end_at: datetime
    attendees: int  # 参加人数

    def time_range(self) -> Tuple[datetime, datetime]:
        """予約の時間帯（[start, end)）を返す。終端は開区間として扱う。"""
        return (self.start_at, self.end_at)

def validate_time_order(start_at: datetime, end_at: datetime) -> None:
    """
    開始時刻が終了時刻より前かを検証。
    逆転していたら ValueError を投げる。
    """
    if not (start_at < end_at):
        raise ValueError("開始時刻は終了時刻より前である必要があります。")

def overlaps(a: Tuple[datetime, datetime], b: Tuple[datetime, datetime]) -> bool:
    """
    時間帯の重なり判定（[start, end) どうし）。
    例：a=[10:00,11:00), b=[11:00,12:00) は重ならない（終端は開く）。
    """
    (a0, a1), (b0, b1) = a, b
    return (a0 < b1) and (b0 < a1)
```

**ポイント**

* `validate_time_order` と `overlaps` は**純粋関数**なので、**単体テストが超簡単**。
* 以降の UseCase では、**保存やWebを知らない**この関数を呼ぶだけで**正しい判定**ができます。

---

### ✳️ `domain/room.py`（会議室：定員・設備）

```python
# --------------------------------------------------------------------
# [クラス図] Entity: Room（会議室）
# [同心円] Entity層
# 役割：会議室の属性（定員・設備）を表す。外部技術は知らない。
# --------------------------------------------------------------------
from dataclasses import dataclass
from typing import Set

@dataclass(frozen=True)
class Room:
    id: int
    name: str
    capacity: int
    equipments: Set[str]  # 例 {"projector", "whiteboard"}
```

**ポイント**

* 「定員オーバーの禁止」そのものは**UseCase側**でチェックします。
* Entity には**必要な属性**だけを**シンプルに**置きます。

---

### ✳️ `domain/member.py`（会員）

```python
# --------------------------------------------------------------------
# [クラス図] Entity: Member（会員）
# [同心円] Entity層
# 役割：予約者を表す最小限の情報。
# --------------------------------------------------------------------
from dataclasses import dataclass

@dataclass(frozen=True)
class Member:
    id: int
    name: str
    email: str  # メールは後で重複禁止（UseCase/Repo側で扱う想定）
```

---

## 🧪 テストのヒント（この章の範囲だけ）

> ここは**Pythonの標準ユニットテスト**や **pytest** で簡単に試せます。
> DBもWebも不要です。

```python
# 例：reservation_entity_test.py（擬似コード）
from datetime import datetime, timedelta
from domain.reservation import validate_time_order, overlaps

def test_validate_time_order_ok():
    s = datetime(2025, 1, 1, 10, 0)
    e = s + timedelta(hours=1)
    validate_time_order(s, e)  # 例外が出なければOK

def test_validate_time_order_ng():
    s = datetime(2025, 1, 1, 10, 0)
    e = datetime(2025, 1, 1, 9, 59)
    try:
        validate_time_order(s, e)
        assert False, "例外が必要"
    except ValueError:
        pass

def test_overlaps_edges():
    s = datetime(2025, 1, 1, 10, 0)
    e = datetime(2025, 1, 1, 11, 0)
    a = (s, e)
    b = (e, e.replace(hour=12))  # [11:00,12:00)
    assert overlaps(a, b) is False  # 終端が接しているだけなら重ならない

def test_overlaps_true():
    a = (datetime(2025,1,1,10), datetime(2025,1,1,11))
    b = (datetime(2025,1,1,10,30), datetime(2025,1,1,11,30))
    assert overlaps(a, b) is True
```

---

## 📘 この章の理解ポイント（要点まとめ）

| 観点         | 内容                                              |
| ---------- | ----------------------------------------------- |
| **役割**     | 現実のルール（予約の時間・会議室の定員・会員情報）を表す中心                  |
| **依存先**    | **なし**（標準ライブラリのみ）                               |
| **知らないもの** | Web・DB・フレームワーク（外の技術）                            |
| **ロジック**   | `validate_time_order`（開始<終了）、`overlaps`（時間帯重複）  |
| **教育効果**   | 後の層（UseCase/Controller/DB）から**使い回せる純粋ロジック**ができた |

---

## ▶ 次のページ

**第2章：UseCase（ユースケース）**

* この Entity を使って、**「予約を作る」手続き**を実装します。
* **入力データの箱（DTO：データ転送オブジェクト）**、**境界の取り決め（Boundary）**、
  **保存役の約束（Repository）** を **UseCase層**に置き、
  **依存性逆転（DIP）**（＝内側が外側を知らない）を体感します。
