## `application/handle_alarm_interactor.py` (Use Case Interactor)


システムの核となるEntity群が定義できたので、いよいよそれらを指揮して具体的な機能を実現する **Use Case Interactor** を実装します。

今回は、この監視レコーダーの重要な機能である「人物検知によるアラーム録画」のユースケースを見ていきましょう。

-----

### このクラスの役割

`HandleAlarmInteractor`は、「カメラが人物を検知した」という外部イベントをきっかけに、アラーム録画の開始から終了までの一連のシナリオを監督する「ディレクター」の役割を担います。

このクラスは、`CameraSetting` Entityを参照してアラーム録画の品質を決定し、ハードウェア（のインターフェース）にプリ録画バッファの確保や本録画の開始を指示し、最終的に`RecordingSegment` Entityを生成して永続化を依頼する、といったアプリケーション固有のビジネスルールを実装します。

-----

### このクラスにあるべき処理

⭕️ **含めるべき処理の例:**

  * `CameraSetting`リポジトリから、該当カメラの設定情報を取得すること。
  * `HardwareInterface`を通じて、ハードウェアにプリ録画バッファの取得や、指定した品質での録画開始を指示すること。
  * `RecordingSegment` Entityを、録画タイプ`ALARM`として生成すること。
  * `StoragePolicy` Entityに「HDDがいっぱいなら、どのセグメントを上書きすべきか？」と問い合わせること。
  * `RecordingSegment`リポジトリに、新しい録画メタデータの保存を依頼すること。
  * `Presenter`に、UI（例：画面の枠を赤くするなど）の更新を依頼すること。

❌ **含めてはいけない処理の例:**

  * どのようにして人物を検知するかの具体的なアルゴリズム（それはハードウェア側の責務です）。
  * 映像データをファイルに書き込む具体的な方法（それもハードウェアアダプターの責務です）。
  * `Entity`が持つべき内部的なルール（例：「プリ録画時間は5秒か10秒か」という設定値のバリデーションは`CameraSetting` Entityの責務です）。

-----

### ソースコードの詳細解説

ご要望の通り、このクラスの責務は複雑なので、メソッドの\*\*実装は日本語のコメントによる擬似コード（シュードコード）\*\*で表現します。これにより、具体的な処理内容よりも、**責務の分離と連携の流れ**に集中できます。

（※このコードは、後ほど作成する`boundaries.py`と`data_structures.py`に、このユースケース用の定義が追加されていることを前提とします。）

```python
# application/handle_alarm_interactor.py

from boundaries import (
    HandleAlarmInputBoundary, HandleAlarmOutputBoundary,
    CameraSettingDataAccessInterface, RecordingSegmentDataAccessInterface,
    HardwareInterface
)
from data_structures import HandleAlarmInputData, HandleAlarmOutputData
from domain.entities import StoragePolicy

class HandleAlarmInteractor(HandleAlarmInputBoundary):
    """「アラーム録画」ユースケースの実装"""
    def __init__(self,
                 presenter: HandleAlarmOutputBoundary,
                 camera_setting_repo: CameraSettingDataAccessInterface,
                 segment_repo: RecordingSegmentDataAccessInterface,
                 hardware: HardwareInterface,
                 storage_policy: StoragePolicy):
        """
        [依存性の注入]
        このユースケースに必要な「部品」をすべてインターフェースとして受け取る。
        """
        self._presenter = presenter
        self._camera_setting_repo = camera_setting_repo
        self._segment_repo = segment_repo
        self._hardware = hardware
        self._storage_policy = storage_policy

    def execute(self, input_data: HandleAlarmInputData):
        """
        [ビジネスロジックの実行（擬似コード）]
        アラーム録画のシナリオを記述する。
        """
        # --- 準備フェーズ ---
        # 1. 入力データから、アラームが発生したカメラIDを取得する
        camera_id = input_data.camera_id
        
        # 2. カメラ設定リポジトリから、該当カメラの設定(CameraSetting)を取得する
        camera_setting = self._camera_setting_repo.find_by_id(camera_id)
        # (もし設定がなければエラー処理)

        # --- 実行フェーズ ---
        # 3. ハードウェアに、プリ録画バッファの取得を指示する
        pre_record_data = self._hardware.get_pre_record_buffer(camera_id)

        # 4. ハードウェアに、アラーム品質での本録画開始を指示し、アフター録画時間分だけ継続する
        #    (この処理は非同期で行われると仮定)
        self._hardware.start_recording(
            camera_id=camera_id,
            quality=camera_setting.alarm_setting.quality,
            duration_sec=camera_setting.alarm_setting.post_record_sec
        )

        # --- 保存フェーズ ---
        # 5. プリ録画とアフター録画の時間を元に、新しい録画セグメントの情報を計算する
        #    (IDの採番、開始・終了時刻の決定など)
        new_segment_info = self._segment_repo.get_next_id_and_timestamps(...)

        # 6. 新しいRecordingSegmentエンティティを生成する (種別はALARM)
        new_segment = RecordingSegment(info=new_segment_info, type=RecordingType.ALARM, ...)

        # 7. (必要であれば) HDDがいっぱいの場合、StoragePolicyに問い合わせて上書き対象を決定し、削除を依頼する
        all_segments = self._segment_repo.find_all()
        segment_to_overwrite = self._storage_policy.find_segment_to_overwrite(all_segments)
        if segment_to_overwrite:
            self._segment_repo.delete(segment_to_overwrite)
            # self._hardware.delete_file(...) のような物理削除の指示も必要になる

        # 8. 録画セグメントリポジトリに、新しいセグメントのメタデータを保存するよう依頼する
        self._segment_repo.save(new_segment)

        # --- 通知フェーズ ---
        # 9. OutputDataを生成し、Presenterに処理完了を通知する
        output_data = HandleAlarmOutputData(camera_id=camera_id, segment_id=new_segment.id)
        self._presenter.present(output_data)

        # このメソッド自体はすぐに終了し、実際の録画処理はバックグラウンドで行われる
        pass
```

  * `execute`メソッド内のコメントが、この複雑なユースケースの「レシピ」です。各ステップで、`Interactor`が様々な「部品」（リポジトリ、ハードウェア、Entity）に**何をすべきか指示を出している**様子が分かります。
  * `Interactor`自身は、具体的な録画データの扱いやHDDの物理的な操作は行わず、すべての詳細な作業を他の部品に**委任**しています。

-----

### このクラスの鉄則

このクラスは、システムの「ディレクター」として、各役者に的確な指示を出します。

> **ビジネスの流れを指揮せよ、ただし詳細には関わるな。 (Orchestrate the business flow, but delegate the details.)**

  * このクラスは、監視レコーダーの**アプリケーションとしての振る舞い**を定義します。
  * しかし、個々の部品（Entity, Hardware, DB）が**どのように**その振る舞いを実現するのか、という技術的な詳細には一切関与しません。この責務の分離が、システムの複雑さを整理し、見通しを良く保つ鍵となります。
  