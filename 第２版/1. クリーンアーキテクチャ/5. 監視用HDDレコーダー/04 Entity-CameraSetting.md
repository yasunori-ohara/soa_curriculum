## `domain/entities.py` - CameraSetting

各カメラの「設定情報」を司る`CameraSetting` Entityを実装します。

-----

### このクラスの役割 ⚙️

`CameraSetting`**は、16台ある各カメラの**「設定情報」を表現するEntityです。

通常録画やアラーム録画の品質（解像度、フレームレート）、プリ/アフター録画の時間といった、カメラ一台一台に紐づく**運用ルール**をカプセル化します。このEntityのインスタンスが、カメラの台数分（最大16個）存在することになります。

`RecordingSegment`が「録画された**結果**」を表現するのに対し、`CameraSetting`は「これから**どう録画するか**」というルールを定義します。

-----

### このクラスにあるべき処理

⭕️ **含めるべき処理の例:**

  * カメラ設定を定義するためのプロパティ（カメラID, 各種録画設定など）。
  * 自己バリデーション（例：カメラIDが1〜16の範囲内か、プリ録画時間が定められた選択肢（5秒 or 10秒）か、といったチェック）。
  * 設定値の整合性を保つルール（例：もし仕様にあれば、「アラーム録画の画質は通常録画の画質以上でなければならない」といったルール）。

❌ **含めてはいけない処理の例:**

  * 設定をファイルやデータベースに保存する具体的な処理。
  * これらの設定を使って、実際にカメラのハードウェアを制御する処理。
  * 他のカメラの設定に関する知識。

-----

### ソースコードの詳細解説

このEntityは設定項目が多いため、関連する設定を`dataclass`で小さなグループにまとめて、メインクラスを整理します。

```python
# domain/entities.py (続き)

import datetime
from enum import Enum
from dataclasses import dataclass

# --- 以前定義したEnum ---
# class RecordingType(Enum): ...
# class Resolution(Enum): ...

# --- 新しく定義するEnum ---
class FrameRate(Enum):
    """1秒間の記録枚数（フレームレート）"""
    FPS_30 = 30
    FPS_20 = 20
    FPS_10 = 10
    FPS_5 = 5
    FPS_1 = 1

# --- 設定値をグループ化するdataclass ---
@dataclass(frozen=True)
class RecordingQuality:
    """録画品質（解像度とフレームレート）を表現する"""
    resolution: Resolution
    frame_rate: FrameRate

@dataclass(frozen=True)
class AlarmSetting:
    """アラーム録画に関する設定を表現する"""
    quality: RecordingQuality
    pre_record_sec: int
    post_record_sec: int

    def __post_init__(self):
        # dataclassの生成後に呼ばれるバリデーション
        if self.pre_record_sec not in [5, 10]:
            raise ValueError("プリ録画時間は5秒または10秒でなければなりません。")
        if self.post_record_sec not in [10, 20, 30]:
            raise ValueError("アフター録画時間は10, 20, 30秒のいずれかでなければなりません。")

# ----------------------------------

class CameraSetting:
    """カメラ設定エンティティ"""
    def __init__(self,
                 camera_id: int,
                 normal_quality: RecordingQuality,
                 alarm_setting: AlarmSetting):
        """
        [データ]
        一台のカメラに関するすべての設定情報を定義する。
        """
        if not (1 <= camera_id <= 16):
            raise ValueError("カメラIDは1から16の間でなければなりません。")
        
        self.camera_id = camera_id
        self.normal_quality = normal_quality
        self.alarm_setting = alarm_setting

    def __repr__(self):
        return (f"CameraSetting(cam_id={self.camera_id}, "
                f"normal={self.normal_quality}, alarm={self.alarm_setting})")

```

  * `FrameRate(Enum)`: １秒間の記録枚数をEnumとして定義し、`30, 20, 10, 5, 1`以外の不正な値が設定されるのを防ぎます。
  * `RecordingQuality` (dataclass): 解像度とフレームレートをまとめた、再利用可能な小さなデータ構造です。
  * `AlarmSetting` (dataclass): アラーム録画に関する設定（品質、プリ/アフター録画時間）を一つのグループにまとめています。
      * `__post_init__`メソッド：これは`dataclass`の便利な機能で、オブジェクトが初期化された**直後**に実行されます。ここで、プリ/アフター録画の秒数が仕様で定められた値（`[5, 10]`など）であるかをチェックする**バリデーションルール**を実装しています。
  * `CameraSetting`クラス：このEntityの本体です。
      * `__init__`: `camera_id`が1〜16の範囲内であるかをチェックします。また、プロパティとして、先ほど定義した`RecordingQuality`と`AlarmSetting`のインスタンスを保持することで、コードが非常に整理され、可読性が高まっています。

> dataclassは、データを保持する責務を持つクラスを簡潔に書くための非常に便利な機能で、まさに本格的なプロジェクトで活躍します。
-----

### このクラスの鉄則

Entityの鉄則は、これまでと全く同じです。

> **何物にも依存するな (Depend on Nothing)**

  * この`CameraSetting`クラスは、レコーダーシステムの**設定に関するルール**をカプセル化します。UIのデザインが変わっても、HDDのモデルが変わっても、この設定のルール自体は安定しています。
  * `UseCase`や`Adapter`といった外側のレイヤーについて一切知ることなく、自己完結しています。
  * このクラスは、システムの他の部分から**利用される**だけの存在です。