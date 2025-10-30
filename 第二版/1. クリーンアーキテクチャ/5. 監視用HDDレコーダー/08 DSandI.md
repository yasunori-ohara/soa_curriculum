## DS and I

## `data_structures.py` (Data Structures)

では、今しがた定義した`HandleAlarmInteractor`が必要とする「公式な書類」(\<DS\>)**と**「契約書」(\<I\>)を解説します。

-----

### このファイルの役割 ✉️

このファイルは、レイヤーの境界を越えてデータを運ぶためだけの、\*\*単純なデータコンテナ（運び屋）\*\*です。`HandleAlarm`ユースケースに関連するデータ構造をここに追加します。

-----

### このファイルにあるべき処理

⭕️ **含めるべき処理の例:**

  * 運搬するデータフィールド（属性）の定義のみ。

❌ **含めてはいけない処理の例:**

  * いかなるロジックも持ちません。

-----

### ソースコードの詳細解説

```python
# data_structures.py
from dataclasses import dataclass
from typing import List

# --- HandleAlarmユースケース用のデータ構造 ---

@dataclass(frozen=True)
class HandleAlarmInputData:
    """イベント検知 -> Use Case へ渡すデータ構造"""
    camera_id: int
    event_time: "datetime.datetime"

@dataclass(frozen=True)
class HandleAlarmOutputData:
    """Use Case -> Presenter へ渡すデータ構造"""
    camera_id: int
    segment_id: int

@dataclass
class SystemStatusViewModel:
    """Presenter -> View へ渡すデータ構造（画面表示用）"""
    # 例：どのカメラが録画中か、アラーム状態かなどを保持
    recording_cameras: List[int]
    alarm_cameras: List[int]
    message: str = ""

```

  * `HandleAlarmInputData`: どのカメラでアラームが検知されたかを`camera_id`で伝えます。
  * `HandleAlarmOutputData`: 処理の結果として、アラーム録画がどの`segment_id`で保存されたかを`Presenter`に伝えます。
  * `SystemStatusViewModel`: `View`は、このオブジェクトの状態を見て「カメラ5の枠を赤くする」といった画面描画を行います。

-----

### このクラス群の鉄則

> **ただ運び、何も考えるな。 (Just carry, don't think.)**

-----

## `boundaries.py` (Interfaces / Boundaries)

### このファイルの役割 🔌

このファイルは、`Use Case`と外側の層（`Adapters`）との間の「契約」を定義するインターフェースを集めたものです。`HandleAlarm`ユースケースに必要な契約をここに追加します。

-----

### このファイルにあるべき処理

⭕️ **含めるべき処理の例:**

  * 抽象クラス (`ABC`) と 抽象メソッド (`@abstractmethod`) の定義のみ。

❌ **含めてはいけない処理の例:**

  * 具体的なロジックの実装。

-----

### ソースコードの詳細解説

```python
# boundaries.py
from abc import ABC, abstractmethod
from typing import List, Optional
from .data_structures import HandleAlarmInputData, HandleAlarmOutputData
from domain.entities import (
    RecordingSegment, CameraSetting, RecordingSchedule, StoragePolicy
)

# --- HandleAlarmユースケース用の境界 ---

class HandleAlarmInputBoundary(ABC):
    @abstractmethod
    def execute(self, input_data: HandleAlarmInputData):
        pass

class HandleAlarmOutputBoundary(ABC):
    @abstractmethod
    def present(self, output_data: HandleAlarmOutputData):
        pass

# --- Data Access Interfaces ---
# UseCaseが必要とするデータアクセスの契約をすべて定義する

class RecordingSegmentDataAccessInterface(ABC):
    @abstractmethod
    def find_all(self) -> List[RecordingSegment]:
        pass
    
    @abstractmethod
    def save(self, segment: RecordingSegment) -> RecordingSegment:
        pass

    @abstractmethod
    def delete(self, segment: RecordingSegment):
        pass

class CameraSettingDataAccessInterface(ABC):
    @abstractmethod
    def find_by_id(self, camera_id: int) -> Optional[CameraSetting]:
        pass

class RecordingScheduleDataAccessInterface(ABC):
    @abstractmethod
    def find_for_camera(self, camera_id: int) -> List[RecordingSchedule]:
        pass

# --- Hardware Interface ---
# (以前定義したものに、プリ録画バッファ取得のメソッドを追加)
class HardwareInterface(ABC):
    @abstractmethod
    def get_pre_record_buffer(self, camera_id: int) -> bytes: # 映像データはbytes型と仮定
        pass
        
    @abstractmethod
    def start_recording(self, camera_id: int, quality: "RecordingQuality", duration_sec: int):
        pass

    # 他にも stop_recording, delete_file などが必要になる
```

  * **複数の`DataAccessInterface`**: `HandleAlarmInteractor`は、「カメラ設定」と「録画セグメント」の両方のデータを扱うため、`CameraSettingDataAccessInterface`と`RecordingSegmentDataAccessInterface`という2つのインターフェースに依存します。
  * **`HardwareInterface`の拡張**: 以前は`dispense_item`など自動販売機用のメソッドがありましたが、これを監視レコーダー用に書き換えています。`get_pre_record_buffer`（プリ録画バッファ取得）や`start_recording`（録画開始）といった、ハードウェアへの指示が抽象メソッドとして定義されています。

-----

### このファイル群の鉄則

> **契約を定義せよ、実装は他に任せよ。 (Define the contract, delegate the work.)**

