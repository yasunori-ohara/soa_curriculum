### `ui_layer.py` - SystemEventController

それではUI層の「交通整理員」、**Controller**について解説します。

-----

### このクラスの役割 🚦

**`SystemEventController`は、`View`からのユーザー操作や、今回の例のようにハードウェアから発生するシステムイベント**を受け取り、それを`Use Case`が理解できる形式（`InputData`）に整えて\*\*「伝達」\*\*する交通整理員です。

監視レコーダーでは、「ユーザーがボタンを押した」という操作だけでなく、「カメラが人物を検知した」というハードウェア起点のイベントも処理のきっかけになります。`Controller`は、これらの様々な種類の入力を受け付け、対応する`Use Case`の窓口に渡す責務を持ちます。

-----

### このクラスにあるべき処理

⭕️ **含めるべき処理の例:**

  * `View`やシステムからのイベント（メソッド呼び出し）を受け取ること。
  * 受け取った情報（例：カメラID）を、`InputData`オブジェクトに変換すること。
  * `InputBoundary`インターフェースを通じて、適切な`Use Case`を呼び出すこと。

❌ **含めてはいけない処理の例:**

  * ビジネスロジックそのもの（`Use Case`の責務）。
  * 画面表示の準備（`Presenter`の責務）。
  * ハードウェアを直接制御する処理（`Hardware Adapter`の責務）。

-----

### ソースコードの詳細解説

このクラスも、責務を明確にするために関数の実装は日本語のコメントによる擬似コードで表現します。

```python
# ui_layer.py (続き)

import datetime
from boundaries import HandleAlarmInputBoundary
from data_structures import HandleAlarmInputData

class SystemEventController:
    """
    Controllerは、ハードウェアなどからのシステムイベントと
    ビジネスロジックを繋ぐ薄い層。
    """
    def __init__(self, use_case: HandleAlarmInputBoundary):
        """
        [依存先の定義]
        呼び出し先であるUse Caseのインターフェースにのみ依存する。
        """
        self._handle_alarm_use_case = use_case

    def on_person_detected(self, camera_id: int):
        """
        [伝達処理（擬似コード）]
        「人物検知」というイベントを受け取り、Use Caseの実行を依頼する。
        """
        # 1. タイムスタンプなど、Use Caseが必要とする追加情報を取得する
        event_time = datetime.datetime.now()

        # 2. 受け取った情報を、InputDataという公式な形式に変換する
        input_data = HandleAlarmInputData(
            camera_id=camera_id,
            event_time=event_time
        )
        
        # 3. 変換したデータを添えて、インターフェースを通じてUse Caseの実行を依頼する
        self._handle_alarm_use_case.execute(input_data)

        pass

# -----------------------------------------------------
# 補足：ユーザー操作を扱うControllerの例
#
# class ControlPanelController:
#     def __init__(self, change_display_use_case: ...):
#         self._change_display_use_case = change_display_use_case
#
#     def on_display_change_button_pressed(self, layout: "DisplayLayout"):
#         # ... 対応するUse Caseを呼び出す ...
#         pass
```

  * `__init__`メソッドは、呼び出し先である`HandleAlarmInteractor`の**インターフェース**を受け取ります。
  * `on_person_detected`メソッドがこのクラスの仕事のすべてです。
    1.  このメソッドは、外部の何か（ハードウェアを監視する別のプログラムなど）から「カメラ5で人物が検知された！」という形で呼び出されることを想定しています。
    2.  `camera_id`に加え、イベントが発生した時刻という付加情報を取得します。
    3.  これらの情報を公式な書類である`HandleAlarmInputData`に変換（箱詰め）します。
    4.  その`InputData`を引数として、`Use Case`の`execute`メソッドを呼び出します。

`Controller`自身は一切「考えず」、ただ情報を右から左へ受け流しているだけ、という点が重要です。

-----

### このクラスの鉄則

このクラスは、交通整理員として、決められたルールに従うだけです。

> **入力を整理し、然るべき場所に渡せ。 (Organize the input and pass it to the right place.)**

  * `Controller`はビジネスロジックを持ちません。UIやハードウェアイベントと、ビジネスロジックの間に立ち、両者が直接会話しないようにする**緩衝材**の役割に徹します。
  * この層が薄く保たれているおかげで、`Use Case`はイベントの発生源（物理的なセンサー？AIによる画像解析？）を知る必要がなく、クリーンな分離が実現できます。
