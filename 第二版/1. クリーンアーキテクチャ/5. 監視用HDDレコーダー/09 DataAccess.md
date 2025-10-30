## `adapters/data_access.py` (Data Access)

`Use Case`が必要とする「契約書」(`boundaries.py`)が定義できたので、次はそれを具体的に実装する**Adapters**層を見ていきましょう。

まずは、データベースとのやり取りを担当する**Data Access**について解説します。今回は3種類のデータ（録画セグメント、カメラ設定、録画スケジュール）を扱うため、それぞれのリポジトリを実装します。

-----
### このファイルの役割 💾

このファイルは、`boundaries.py`で定義されたデータアクセスインターフェース（`RecordingSegmentDataAccessInterface`など）を、**具体的な技術（今回はインメモリのリストや辞書）で実装するアダプター**を集めたものです。

`Use Case`からの抽象的なデータ永続化の要求を、Pythonのデータ構造を操作するという具体的なストレージへの読み書き処理に**変換**して実行します。

-----

### 各クラスにあるべき処理

⭕️ **含めるべき処理の例:**

  * インターフェースで定義されたメソッドの具体的な実装。
  * `Entity`を永続化するためのデータソースとのやり取り。

❌ **含めてはいけない処理の例:**

  * ビジネスロジック（`Use Case`の責務）。

-----

### ソースコードの詳細解説

3つのインターフェースに対応する、3つの具体的な実装クラスを記述します。

```python
# adapters/data_access.py

from typing import Dict, List, Optional
import datetime

from domain.entities import (
    RecordingSegment, CameraSetting, RecordingSchedule,
    Resolution, FrameRate, RecordingQuality, AlarmSetting, DayOfWeek
)
from boundaries import (
    RecordingSegmentDataAccessInterface, CameraSettingDataAccessInterface, RecordingScheduleDataAccessInterface
)

class InMemoryRecordingSegmentDataAccess(RecordingSegmentDataAccessInterface):
    """RecordingSegmentDataAccessInterfaceのインメモリ版実装"""
    _database: List[RecordingSegment] = []
    _next_id: int = 1

    def find_all(self) -> List[RecordingSegment]:
        return self._database

    def save(self, segment: RecordingSegment) -> RecordingSegment:
        # 擬似コード：IDを採番してセグメントをリストに追加する
        segment.segment_id = self._next_id
        self._database.append(segment)
        self._next_id += 1
        return segment

    def delete(self, segment: RecordingSegment):
        # 擬似コード：リストから該当するセグメントを削除する
        self._database.remove(segment)

class InMemoryCameraSettingDataAccess(CameraSettingDataAccessInterface):
    """CameraSettingDataAccessInterfaceのインメモリ版実装"""
    _database: Dict[int, CameraSetting] = {}

    def __init__(self):
        # サンプルとしてカメラ1にデフォルト設定を用意しておく
        default_quality = RecordingQuality(resolution=Resolution.MEDIUM, frame_rate=FrameRate.FPS_10)
        default_alarm_setting = AlarmSetting(
            quality=RecordingQuality(resolution=Resolution.HIGH, frame_rate=FrameRate.FPS_30),
            pre_record_sec=5,
            post_record_sec=10
        )
        self._database[1] = CameraSetting(
            camera_id=1,
            normal_quality=default_quality,
            alarm_setting=default_alarm_setting
        )

    def find_by_id(self, camera_id: int) -> Optional[CameraSetting]:
        return self._database.get(camera_id)

class InMemoryRecordingScheduleDataAccess(RecordingScheduleDataAccessInterface):
    """RecordingScheduleDataAccessInterfaceのインメモリ版実装"""
    _database: List[RecordingSchedule] = []

    def __init__(self):
        # サンプルとしてカメラ1に月曜日のスケジュールを登録しておく
        self._database.append(RecordingSchedule(
            schedule_id=1,
            camera_id=1,
            day_of_week=DayOfWeek.MONDAY,
            start_time=datetime.time(9, 0),
            end_time=datetime.time(17, 0)
        ))

    def find_for_camera(self, camera_id: int) -> List[RecordingSchedule]:
        return [schedule for schedule in self._database if schedule.camera_id == camera_id]

```

  * **`InMemoryRecordingSegmentDataAccess`**:
    `RecordingSegment`のリストを`_database`として保持します。`save`メソッドでは、新しいIDを採番してリストに追加するという、ID管理のロジックも（簡易的に）担当しています。
  * **`InMemoryCameraSettingDataAccess`**:
    カメラIDをキーとする辞書を`_database`として持ちます。`__init__`で、カメラ1番のデフォルト設定をあらかじめ登録しています。
  * **`InMemoryRecordingScheduleDataAccess`**:
    `RecordingSchedule`のリストを`_database`として持ちます。`find_for_camera`メソッドでは、リスト内を検索し、指定された`camera_id`に一致するすべてのスケジュールを返します。

-----

### このファイル群の鉄則

このファイル内のすべてのクラスは、同じ鉄則に従います。

> **データを変換し、黙って実行せよ。 (Translate and execute, but do not think.)**

  * これらのクラスはビジネスルールについて**思考しません**。`Use Case`という管理者からの指示を、技術的な作業として忠実に実行するだけです。
  * このファイル群のおかげで、`Use Case`は設定情報や録画履歴がメモリ上にあるのか、設定ファイル（XMLやJSON）に保存されているのか、あるいは本格的なデータベースに入っているのかを一切気にせずに済みます。