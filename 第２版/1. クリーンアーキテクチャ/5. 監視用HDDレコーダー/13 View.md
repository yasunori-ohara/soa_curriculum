## `ui_layer.py` - ConsoleView

UI層の最後にして、ユーザー（または監視員）との直接の接点となる`ConsoleView`について解説します。

-----

### このクラスの役割 🖼️

**`ConsoleView`は、ユーザーが直接目にし、操作するユーザーインターフェース（UI）そのもの**です。監視レコーダーのモニター画面と操作パネルに相当します。

このクラスの責務は、大きく分けて2つです。

1.  **表示（Output）**: `Presenter`によって状態が更新された`ViewModel`のデータを、画面（今回はコンソール）に具体的に描画し、現在のシステム状態（どのカメラがアラームかなど）をリアルタイムに表示し続ける。
2.  **入力（Input）**: ユーザーからのキーボード入力（操作パネルのボタン操作をシミュレート）を受け付け、その操作を`Controller`に伝える。

-----

### このクラスにあるべき処理

⭕️ **含めるべき処理の例:**

  * `print()`や`input()`など、具体的なUI技術を直接扱うコード。
  * `ViewModel`のデータを読み取って、画面に表示するロジック（例：アラーム中のカメラIDのリストを元に、「[ALARM] Camera 5」のように表示を切り替える）。
  * ユーザーの操作に応じて、適切な`Controller`のメソッドを呼び出すこと。

❌ **含めてはいけない処理の例:**

  * ビジネスロジック（`Use Case`の責務）。
  * 表示用データの整形ロジック（`Presenter`の責務）。

-----

### ソースコードの詳細解説

この`View`は、システムの現在の状態を表示し続けながら、ユーザーや外部からのイベント入力を受け付けるメインループを持ちます。ここでも実装は擬似コードで表現します。

```python
# ui_layer.py (続き)

import time
from data_structures import SystemStatusViewModel
# ControllerやPresenterのimportは省略

class ConsoleView:
    """
    ユーザーとの直接的なやり取り（入力受付・画面表示）を担当する。
    """
    def __init__(self,
                 system_event_controller: "SystemEventController",
                 # control_panel_controller: "ControlPanelController", # ボタン操作用のController
                 view_model: SystemStatusViewModel):
        """
        [依存先の定義]
        指示を出す相手である各種Controllerと、
        表示内容の元となるViewModelを保持する。
        """
        self._system_event_controller = system_event_controller
        # self._control_panel_controller = control_panel_controller
        self._view_model = view_model

    def start_monitoring(self):
        """
        [入力と表示のメインループ（擬似コード）]
        監視システムのメインループを実行する。
        """
        print("--- 監視システムを起動しました ---")
        
        # 実際のアプリケーションでは、ここで別スレッドを開始して
        # キーボード入力と外部イベント（アラーム検知）を待ち受けることになる。
        # 今回はそれをシミュレートする。
        
        while True:
            # 1. 現在のViewModelの状態を画面に描画する
            self.render()

            # 2. ユーザー操作（またはシミュレーション）を待つ
            print("\n操作を入力してください (例: 'alarm 5' -> カメラ5でアラーム発生, 'q' -> 終了)")
            command_str = input("> ")
            parts = command_str.split()
            command = parts[0]

            # 3. 入力に応じてControllerを呼び出す
            if command == "alarm" and len(parts) > 1:
                # 「カメラ5で人物検知」という外部イベントをシミュレート
                try:
                    camera_id = int(parts[1])
                    self._system_event_controller.on_person_detected(camera_id)
                except ValueError:
                    self._view_model.message = "エラー: カメラIDは数字で入力してください。"
                except Exception as e:
                    self._view_model.message = f"エラー: {e}"

            elif command == "q":
                print("--- 監視システムを終了します ---")
                break
            
            else:
                # ここに再生ボタンや表示切替ボタンが押された際の
                # control_panel_controllerの呼び出し処理が入る
                self._view_model.message = f"'{command_str}' は無効な操作です。"

            # 少し待って画面を更新（リアルタイム感を出すためのダミー）
            time.sleep(1)

    def render(self):
        """
        [表示処理（擬似コード）]
        ViewModelの現在の状態を元に画面を描画する。
        """
        # 画面をクリアする (コンソール用のおまじない)
        # print("\033[H\033[J", end="")

        print("--- 監視モニター ---")
        
        # ViewModelからアラーム中のカメラリストを取得して表示する
        if self._view_model.alarm_cameras:
            print(f"!! アラーム発生中: {self._view_model.alarm_cameras}")
        
        # ViewModelから最新のメッセージを取得して表示する
        if self._view_model.message:
            print(f"通知: {self._view_model.message}")
            self._view_model.message = "" # 一度表示したらメッセージをクリア

        print("--------------------")
```

  * `start_monitoring()`メソッド：この`View`のメインループです。
    1.  `render()`を呼び出して現在のシステム状態を表示します。
    2.  `input()`でユーザーからのコマンド入力を待ち受けます。
    3.  入力が`alarm 5`のような形式だった場合、「カメラ5で人物が検知された」という**外部イベントをシミュレート**し、`SystemEventController`の`on_person_detected`メソッドを呼び出します。
  * `render()`メソッド：`ViewModel`の`alarm_cameras`リストや`message`プロパティを見て、コンソールに現在の状態を出力します。実際の監視画面では、このメソッドが1秒間に何回も呼び出され、映像を描画したり、カメラの枠の色を変えたりする処理を担当します。

-----

### このクラスの鉄則

このクラスは、ロジックを持たず、単純な作業に徹します。

> **愚直であれ（Be Dumb）**

  * `View`はビジネス上の判断を一切行いません。ただ、\*\*「ユーザーや外部のイベントをControllerに伝え、ViewModelを画面に表示する」\*\*という2つの単純な責務だけを果たします。
  * このクラスを「おバカさん」に保つことで、UIの変更（例：コンソールから専用の液晶パネルへ）があったとしても、影響範囲をこの`View`と`Presenter`だけに限定することができます。
