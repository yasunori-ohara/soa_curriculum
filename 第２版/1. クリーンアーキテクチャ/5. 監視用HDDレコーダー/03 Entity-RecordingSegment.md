## `domain/entities.py` - RecordingSegment

### このクラスの役割 📼

**`RecordingSegment`は、HDDに保存される、ひとかたまりの「録画データ」のメタ情報**を表現するEntityです。

これは、映像データそのもの（巨大なバイナリファイル）ではなく、その映像が「いつ」「どのカメラで」「どのような種類で」録画されたか、といった**管理情報**をカプセル化します。

「古い映像を消去する」「アラーム録画を検索する」といった、このシステムの根幹をなす機能は、すべてこの`RecordingSegment`オブジェクトを単位として操作することで実現されます。

-----

### このクラスにあるべき処理

⭕️ **含めるべき処理の例:**

  * 録画セグメントを定義するためのプロパティ（ID, カメラID, 開始/終了時刻, 録画タイプなど）。
  * 自身の状態に関する単純なルール（例：このセグメントは保護対象（緊急録画）か？を判定する`is_protected()`メソッド）。
  * 自己バリデーション（例：終了時刻が開始時刻より前になっていないかのチェック）。

❌ **含めてはいけない処理の例:**

  * 映像データをファイルに書き込む具体的な処理。
  * 他の`RecordingSegment`との比較（例：「自分が一番古いセグメントか？」といった判断。これは`StoragePolicy`の責務です）。
  * データベースへの保存やUIに関する処理。

-----

### ソースコードの詳細解説

このファイルでは、`RecordingSegment`本体に加え、録画タイプや解像度といった「選択肢」を明確に定義するための`Enum`（列挙型）も一緒に定義します。

```python
# domain/entities.py

import datetime
from enum import Enum

# --- 選択肢を定義するEnum ---

class RecordingType(Enum):
    """録画の種類"""
    NORMAL = "通常録画"
    ALARM = "アラーム録画"
    EMERGENCY = "緊急録画"

class Resolution(Enum):
    """解像度"""
    LOW = "低"
    MEDIUM = "中"
    HIGH = "高"

# ----------------------------------

class RecordingSegment:
    """録画セグメントエンティティ"""
    def __init__(self,
                 segment_id: int,
                 camera_id: int,
                 start_time: datetime.datetime,
                 end_time: datetime.datetime,
                 recording_type: RecordingType,
                 resolution: Resolution,
                 frame_rate: int):
        """
        [データ]
        録画セグメントが本質的に持つべき管理情報（メタデータ）を定義する。
        """
        if start_time >= end_time:
            raise ValueError("終了時刻は開始時刻より後でなければなりません。")
        
        self.segment_id = segment_id
        self.camera_id = camera_id
        self.start_time = start_time
        self.end_time = end_time
        self.recording_type = recording_type
        self.resolution = resolution
        self.frame_rate = frame_rate

    def is_protected(self) -> bool:
        """
        [ビジネスルール]
        このセグメントが保護対象（上書き禁止）であるかを判断する。
        """
        return self.recording_type == RecordingType.EMERGENCY

    def __repr__(self):
        return (f"RecordingSegment(id={self.segment_id}, cam={self.camera_id}, "
                f"type={self.recording_type.value}, start='{self.start_time}')")
```

  * `RecordingType` / `Resolution` (Enum): 文字列を直接使う代わりに`Enum`を定義することで、`"NORMAL"`と`"Normal"`のようなタイプミスを防ぎ、コードの安全性を高めています。
  * `__init__`: このEntityを構成するすべてのプロパティを定義します。また、「終了時刻は開始時刻より後でなければならない」という単純な**バリデーションルール**を実装しています。
  * `is_protected()`: このEntityが持つビジネスルールです。「録画タイプが緊急（`EMERGENCY`）であるセグメントは保護対象である」という、このシステムの仕様をカプセル化しています。`StoragePolicy`は、このメソッドを呼び出すだけで、セグメントを消して良いかどうかを判断できます。

-----

### このクラスの鉄則

Entityの鉄則は、これまでと全く同じです。

> **何物にも依存するな (Depend on Nothing)**

  * この`RecordingSegment`クラスは、録画システムの**最も安定した構成要素の一つ**です。HDDのメーカーが変わっても、カメラの機種が変わっても、このクラスの定義は変わりません。
  * `UseCase`や`Adapter`といった外側のレイヤーについて一切知ることなく、自己完結しています。
  * このクラスは、システムの他の部分から**利用される**だけの存在です。