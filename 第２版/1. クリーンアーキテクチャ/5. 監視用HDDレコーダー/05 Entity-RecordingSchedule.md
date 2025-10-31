## `domain/entities.py` - RecordingSchedule

タイマー予約録画のルールを司る`RecordingSchedule` Entityを実装します。

-----

### このクラスの役割

`RecordingSchedule`**は、一台のカメラに対する一つの**「タイマー予約録画ルール」を表現するEntityです。

「月曜日の9時から17時までカメラ1を録画する」といった、時間に紐づく運用ルールをカプセル化します。システム全体では、この`RecordingSchedule` Entityのインスタンスが複数存在し、一週間分の複雑な録画スケジュールを構成することになります。

-----

### このクラスにあるべき処理

⭕️ **含めるべき処理の例:**

  * スケジュールを定義するためのプロパティ（カメラID, 曜日, 開始/終了時刻など）。
  * 自己バリデーション（例：終了時刻が開始時刻より後であるかのチェック）。
  * 自身の状態に関するルール（例：「指定された現在時刻は、このスケジュールの録画時間内か？」を判断する`is_active_at()`メソッド）。

❌ **含めてはいけない処理の例:**

  * 実際にHDDへの録画を開始したり停止したりする処理（それは`UseCase`の責務です）。このEntityは、あくまで「今、録画すべき時間か？」を**判断して答える**だけです。
  * データベースへの保存やUIに関する処理。

-----

### ソースコードの詳細解説

このEntityでは、曜日を安全に扱うために`Enum`を定義します。

```python
# domain/entities.py (続き)

import datetime
from enum import Enum
from dataclasses import dataclass

# --- 以前定義したEnumやdataclass ---
# class RecordingType(Enum): ...
# (中略)

# --- 新しく定義するEnum ---
class DayOfWeek(Enum):
    """曜日を表現する"""
    MONDAY = 0
    TUESDAY = 1
    WEDNESDAY = 2
    THURSDAY = 3
    FRIDAY = 4
    SATURDAY = 5
    SUNDAY = 6

# ----------------------------------

class RecordingSchedule:
    """録画スケジュールエンティティ"""
    def __init__(self,
                 schedule_id: int,
                 camera_id: int,
                 day_of_week: DayOfWeek,
                 start_time: datetime.time,
                 end_time: datetime.time,
                 is_enabled: bool = True):
        """
        [データ]
        一つのタイマー予約ルールを定義する。
        """
        if start_time >= end_time:
            raise ValueError("終了時刻は開始時刻より後でなければなりません。")
        
        self.schedule_id = schedule_id
        self.camera_id = camera_id
        self.day_of_week = day_of_week
        self.start_time = start_time
        self.end_time = end_time
        self.is_enabled = is_enabled

    def is_active_at(self, current_datetime: datetime.datetime) -> bool:
        """
        [ビジネスルール]
        指定された日時が、このスケジュールの有効期間内であるかを判断する。
        """
        if not self.is_enabled:
            return False

        # 曜日のチェック (Pythonのdatetime.weekday()は月曜=0)
        is_correct_day = (current_datetime.weekday() == self.day_of_week.value)
        if not is_correct_day:
            return False

        # 時間のチェック
        current_time = current_datetime.time()
        is_in_time_range = (self.start_time <= current_time < self.end_time)
        
        return is_in_time_range

    def __repr__(self):
        return (f"RecordingSchedule(id={self.schedule_id}, cam_id={self.camera_id}, "
                f"day={self.day_of_week.name}, start='{self.start_time}', end='{self.end_time}')")
```

  * `DayOfWeek(Enum)`: 曜日を0（月曜）〜6（日曜）の数値として定義しています。これはPythonの`datetime`モジュールの`weekday()`メソッドの仕様と一致しており、後の比較処理を容易にします。
  * `__init__`: スケジュールを定義するプロパティを保持します。開始時刻と終了時刻には、日付を含まない`datetime.time`型を使用するのが適切です。
  * `is_active_at()`メソッド：このEntityの核となるビジネスルールです。引数として渡された`datetime.datetime`（現在時刻など）が、「有効なスケジュール」であり、「曜日が一致」し、かつ「時間帯の範囲内」であるか、という3つの条件をすべてチェックして`True`か`False`を返します。このロジックがEntityの内部にカプセル化されているため、`UseCase`はこのメソッドを呼び出すだけで簡単にスケジュール判定ができます。

-----

### このクラスの鉄則

Entityの鉄則は、これまでと全く同じです。

> **何物にも依存するな (Depend on Nothing)**

  * この`RecordingSchedule`クラスは、レコーダーシステムの**スケジュールに関するルール**をカプセル化します。このルールは、UIやHDDの実装とは全く無関係です。
  * `UseCase`や`Adapter`といった外側のレイヤーについて一切知ることなく、自己完結しています。
  * このクラスは、システムの他の部分から**利用される**だけの存在です。
