## `adapters/hardware.py` (Hardware Adapter)

この組み込みシステムの**物理的な側面**を担当する`Hardware Adapter`を解説します。

-----

### このファイルの役割 📹

このファイルは、`boundaries.py`で定義された`HardwareInterface`を、**具体的な技術（今回はコンソールへのシミュレーション出力）で実装するアダプター**です。

`Use Case`からの「カメラ1のプリ録画バッファを取得しろ」「指定された品質で録画を開始しろ」といった抽象的なハードウェアへの指示を、コンソールの`print`文を使って「（カメラ1からプリ録画バッファを取得しました）」のように、**物理的な動作をシミュレートする**処理に変換します。

`Use Case`という「頭脳」と、物理的なカメラやHDDといった「手足」の間に立つ、\*\*「神経系」\*\*のような役割を果たします。

-----

### このクラスにあるべき処理

⭕️ **含めるべき処理の例:**

  * `HardwareInterface`で定義されたメソッドの具体的な実装。
  * 物理デバイスを制御するための低レベルなコード（例：カメラの映像取得ライブラリの呼び出し、HDDへの直接書き込みなど）。
  * 今回の例では、ハードウェアの動作をコンソールに表示するシミュレーションコード。

❌ **含めてはいけない処理の例:**

  * ビジネスロジック（`Use Case`の責務）。
      * 例：「このカメラは今録画すべき時間か？」といった判断は**行いません**。ただ「録画を開始しろ」という指示を忠実に実行するだけです。

-----

### ソースコードの詳細解説

このクラスも、責務を明確にするために関数の実装は日本語のコメントによる擬似コードで表現します。

```python
# adapters/hardware.py

import datetime
from boundaries import HardwareInterface
from domain.entities import RecordingQuality

class ConsoleHardwareAdapter(HardwareInterface):
    """
    HardwareInterfaceの「コンソールシミュレーション」版実装。
    物理デバイスの動作をprint文で模倣する。
    """

    def get_pre_record_buffer(self, camera_id: int) -> bytes:
        """
        [インターフェースの実装]
        プリ録画バッファの取得を、コンソールへのメッセージ表示でシミュレートする。
        """
        print(f"（ハードウェア：カメラ{camera_id}からプリ録画バッファ（5秒分）を取得しました）")
        
        # 実際には、ここでカメラデバイスからメモリ上の映像データを取得する。
        # 今回はダミーのデータを返す。
        dummy_video_data = b"dummy_pre_record_video_data"
        return dummy_video_data

    def start_recording(self, camera_id: int, quality: "RecordingQuality", duration_sec: int):
        """
        [インターフェースの実装]
        指定された品質・時間での録画開始を、コンソールへのメッセージ表示でシミュレートする。
        """
        print(f"（ハードウェア：カメラ{camera_id}の録画を開始しました）")
        print(f"  品質: {quality.resolution.value} / {quality.frame_rate.value}fps")
        print(f"  録画時間: {duration_sec}秒")
        
        # 実際には、ここで映像コーデックハードウェアを設定し、
        # HDDへのストリーム書き込みをバックグラウンドで開始する。
        # このメソッドはすぐにリターンし、録画は非同期で行われる。
        pass

    # --- UseCaseで必要になった場合に追加するメソッドの例 ---
    # def stop_recording(self, camera_id: int):
    #     print(f"（ハードウェア：カメラ{camera_id}の録画を停止しました）")
    #     pass
    #
    # def delete_file(self, segment_id: int):
    #     print(f"（ハードウェア：セグメントID {segment_id} に対応する録画ファイルをHDDから削除しました）")
    #     pass

```

  * `class ConsoleHardwareAdapter(HardwareInterface):`
    この一行で、「私は`HardwareInterface`という契約を守る実装クラスです」と宣言しています。

  * `get_pre_record_buffer`メソッド
    `Use Case`からのプリ録画データ取得の指示を、コンソールへのメッセージ表示でシミュレートします。実際のハードウェアでは、常に映像を保持しているリングバッファから、指定された秒数分のデータを読み出す処理になります。

  * `start_recording`メソッド
    `Use Case`からの録画開始指示をシミュレートします。実際のハードウェアでは、このメソッドが呼ばれると、映像コーデックLSI（大規模集積回路）に指定された解像度・フレームレートを設定し、カメラからの映像ストリームをHDDに書き込み始める、といった低レベルな処理が実行されます。

-----

### このクラスの鉄則

このクラスは、忠実な「手足」に徹します。

> **指示に従い、物理的に動け。 (Follow instructions and act physically.)**

  * このクラスはビジネスルールについて**思考しません**。`Use Case`という頭脳からの指示を、物理的な作業として忠実に実行するだけです。
  * この`HardwareInterface`と`ConsoleHardwareAdapter`があるおかげで、私たちは**実際のカメラや録画用HDDがなくても`Use Case`のロジックをテストできます**。また、将来カメラのモデルが新しくなったり、録画チップが変わったりしても、修正が必要なのはこの`hardware.py`ファイルだけであり、ビジネスロジックには一切影響がありません。これこそが、組み込み開発においてクリーンアーキテクチャが非常に強力である理由です。
