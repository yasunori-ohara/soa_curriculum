## main

アラーム関連の動作を実現するための**全ての「部品」の設計と、その役割分担の定義**が終わった、という状態です。

---
### 今できていること

オーケストラに例えるなら、以下がすべて揃いました。

* **楽譜 (アーキテクチャ)**: クリーンアーキテクチャという全体の構成。
* **各パートの譜面 (設計)**: 各クラスの役割と、擬似コードで書かれた処理内容。
* **楽器 (部品)**:
    * `Entity`という、音色の基礎となる最高品質の楽器。
    * `UseCase`という、演奏の解釈を行う優秀な演奏者。
    * `Adapter`という、特殊な演奏技法を担う専門家たち。

---
### 最後に残された、たった一つの作業

最後に残っているのは、これらの素晴らしい演奏者と楽器をステージ上に正しく配置し、指揮者がタクトを振る、という作業だけです。

これが、**`main.py`の役割（Composition Root）**です。

`main.py`の中で、これまで定義してきた全ての具象クラス（`HandleAlarmInteractor`, `InMemoryDataAccess`, `ConsoleHardwareAdapter`, `AlarmPresenter`, `SystemEventController`, `ConsoleView`など）をインスタンス化し、互いを正しく接続（依存性の注入）します。

そして、`view.start_monitoring()`を呼び出すことで、アプリケーション全体が動き始めます。

はい、承知いたしました。
それでは最後の締めくくりとして、監視レコーダーシステムの`main.py`を解説します。

-----

このファイルの役割

`main.py`は、これまで設計してきた全ての「部品」（クラス）を実際に組み立てて一つのアプリケーションとして完成させ、\*\*起動スイッチを入れる「組立工場」\*\*の役割を果たします。

ここは、アプリケーション全体で**唯一**、具体的な実装クラス（`HandleAlarmInteractor`, `InMemoryDataAccess`, `ConsoleHardwareAdapter`など）が互いを直接知ることを許された場所です。

ここで全ての部品を配線（**依存性の注入 / Dependency Injection**）することで、他の全てのクラスは、お互いの具体的な実装を知らない「クリーン」な状態を保つことができます。

-----

### このファイルにあるべき処理

⭕️ **含めるべき処理の例:**

  * 全ての具体的な実装クラスのインスタンス化（オブジェクト生成）。
  * あるクラスのインスタンスを、別のクラスのコンストラクタ（`__init__`）に渡すことによる、依存性の注入（DI）。
  * アプリケーションの最初の処理（今回は`view.start_monitoring()`）を呼び出すこと。

❌ **含めてはいけない処理の例:**

  * ビジネスロジック、データ変換ロジック、UIの描画ロジックなど。このファイルの責務は、あくまで**組み立て**に専念します。

-----

### ソースコードの詳細解説

この`main.py`が、これまで擬似コードで定義してきた各部品に命を吹き込みます。

```python
# main.py

# --- UI層のクラス ---
from ui_layer import AlarmPresenter, SystemEventController, ConsoleView

# --- アダプター層（UI以外）のクラス ---
from adapters.data_access import (
    InMemoryRecordingSegmentDataAccess,
    InMemoryCameraSettingDataAccess,
    InMemoryRecordingScheduleDataAccess
)
from adapters.hardware import ConsoleHardwareAdapter

# --- ユースケース ---
from application.handle_alarm_interactor import HandleAlarmInteractor

# --- エンティティとデータ構造 ---
from domain.entities import StoragePolicy
from data_structures import SystemStatusViewModel

def main():
    """アプリケーションのすべての部品を組み立て、起動する"""
    print("--- 1. 監視レコーダーシステムの部品を組み立てます ---")

    # --- Entityとポリシーの生成 ---
    # システム全体のストレージ戦略を定義するEntity
    storage_policy = StoragePolicy(emergency_area_percentage=10)

    # --- UIの状態を保持するViewModel ---
    view_model = SystemStatusViewModel(recording_cameras=[], alarm_cameras=[])
    
    # --- アダプター層の生成 ---
    # 各インターフェースに対応する具象クラスをインスタンス化
    presenter = AlarmPresenter(view_model)
    segment_repo = InMemoryRecordingSegmentDataAccess()
    camera_setting_repo = InMemoryCameraSettingDataAccess()
    # schedule_repo = InMemoryRecordingScheduleDataAccess() # 今回のUseCaseでは未使用
    hardware = ConsoleHardwareAdapter()

    # --- ユースケース層の生成 ---
    # 必要な「部品」（アダプターやポリシー）をすべてコンストラクタに注入（DI）
    handle_alarm_use_case = HandleAlarmInteractor(
        presenter=presenter,
        camera_setting_repo=camera_setting_repo,
        segment_repo=segment_repo,
        hardware=hardware,
        storage_policy=storage_policy
    )

    # --- UI層（Controller, View）の生成 ---
    # Controllerに必要なUseCaseを注入
    system_event_controller = SystemEventController(handle_alarm_use_case)
    
    # Viewに必要なControllerとViewModelを注入
    view = ConsoleView(
        system_event_controller=system_event_controller,
        view_model=view_model
    )

    print("--- 2. 組み立て完了、アプリケーションを起動します ---\n")
    
    # 組み立てたViewのインスタンスを使って、アプリケーションの実行を開始
    view.start_monitoring()

if __name__ == "__main__":
    main()
```

`main`関数の中で、依存関係が綺麗に解決されていく様子が分かります。

1.  まず、他の多くのクラスから依存される`StoragePolicy`や`ViewModel`、そして依存先のない`Adapter`群を生成します。
2.  次に、それらの`Adapter`を部品として必要とする`HandleAlarmInteractor`（`UseCase`）を生成します。
3.  さらに、その`UseCase`を必要とする`SystemEventController`を生成します。
4.  最後に、`Controller`と`ViewModel`を必要とする`View`を生成します。
5.  全ての部品が正しく接続された`view`オブジェクトが完成したら、`view.start_monitoring()`を呼び出してアプリケーションを起動します。

-----

### このファイルの鉄則

このファイルは、他の全てのクラスをクリーンに保つための、重要な役割を担います。

> **すべてを知り、すべてを組み立て、そして仕事は最初に任せよ。 (Know everything, assemble everything, then delegate the first task.)**

  * この`main.py`は、アプリケーションの具体的な実装クラスをすべて知っている、唯一の「汚れる」場所です。
  * この場所が**依存関係を一手に引き受ける**ことで、他のすべてのクラスはインターフェースにのみ依存する「クリーン」な状態を保つことができます。
  * 組み立てが終わったら、最初のきっかけ（`view.start_monitoring()`）を与えるだけで、あとは他のクラスに仕事を任せます。
