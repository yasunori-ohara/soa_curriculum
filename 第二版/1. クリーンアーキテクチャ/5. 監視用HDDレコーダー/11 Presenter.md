### `ui_layer.py` - AlarmPresenter

`Use Case`が依存する部品がすべて揃いましたので、いよいよUIを構成するアダプター群、`ui_layer.py`の中から**Presenter**を解説します。

-----
### このクラスの役割 🖥️

`AlarmPresenter`**は、`Use Case`からのビジネスロジックの実行結果（`OutputData`）を受け取り、それを画面表示に最適な形式（`ViewModel`）へと変換する**「ステータス表示設計者」です。

`Use Case`が返すのは、あくまでビジネス中心の純粋なデータです（例：`camera_id=5`, `segment_id=101`）。それを、ユーザーにとって分かりやすい「カメラ5でアラーム録画を開始しました」といった最終的なメッセージに整形したり、「現在アラーム状態のカメラリスト」を更新したりするのが`Presenter`の責務です。

これにより、`Use Case`は「アラーム状態がどのように表示されるか」（テキストメッセージ？画面の枠が赤く点滅する？）を一切気にすることなく、ビジネスロジックに集中できます。

-----

### このクラスにあるべき処理

⭕️ **含めるべき処理の例:**

  * `HandleAlarmOutputBoundary`インターフェースの実装。
  * `OutputData`から`ViewModel`へのデータ変換。
  * 表示用メッセージの組み立てロジック。
  * `ViewModel`の状態（例：アラーム中のカメラリスト）を更新すること。

❌ **含めてはいけない処理の例:**

  * ビジネスルールの判断（`Use Case`の責務）。
  * `print()`文など、具体的な画面描画処理（`View`の責務）。
  * `Controller`や`DataAccess`、`Hardware`に関する知識。

-----

### ソースコードの詳細解説

このクラスも、責務を明確にするために関数の実装は日本語のコメントによる擬似コードで表現します。

```python
# ui_layer.py

from boundaries import HandleAlarmOutputBoundary
from data_structures import HandleAlarmOutputData, SystemStatusViewModel

class AlarmPresenter(HandleAlarmOutputBoundary):
    """
    <I> Output Boundary（設計図）を実装するクラス。
    """
    def __init__(self, view_model: SystemStatusViewModel):
        """
        [状態の共有]
        Viewと共有するViewModelを受け取り、自身のプロパティとして保持する。
        """
        self._view_model = view_model

    def present(self, output_data: HandleAlarmOutputData):
        """
        [インターフェースの実装（擬似コード）]
        'present'という抽象的な要求を、OutputDataからViewModelを
        更新するという具体的な処理に変換して実行する。
        """
        # 1. 共有ViewModelのアラームカメラリストに、今回のカメラIDを追加する
        #    (重複しないように追加するロジック)
        if output_data.camera_id not in self._view_model.alarm_cameras:
            self._view_model.alarm_cameras.append(output_data.camera_id)

        # 2. 画面表示用のメッセージを生成する
        message = f"【アラーム】カメラ{output_data.camera_id}でイベントを検知。録画セグメントID: {output_data.segment_id} として保存しました。"
        
        # 3. 共有ViewModelのメッセージを更新する
        self._view_model.message = message

        pass
```

  * `__init__`メソッドで`View`と共有する`SystemStatusViewModel`を受け取ります。
  * `present`メソッドがこのクラスの核となる処理です。
    1.  `Use Case`から渡された`output_data`を元に、どのカメラがアラーム状態になったかを把握し、共有`ViewModel`の`alarm_cameras`リストを更新します。`View`はこのリストを見て、該当するカメラの表示を変える（例：枠を赤くする）ことができます。
    2.  ユーザーに通知するための、人間が読みやすいメッセージを組み立てます。
    3.  完成したメッセージを共有`ViewModel`にセットすることで、`View`に「表示すべき内容が変わった」ことを間接的に伝えています。

-----

### このクラスの鉄則

このクラスは、忠実な「翻訳家」に徹します。

> **表示のために翻訳せよ、ただしビジネスを語るな。 (Translate for presentation, but don't speak business.)**

  * `Presenter`はビジネスの判断を行いません。`Use Case`から渡された結果を、ただ表示用に美しく整えるだけです。
  * このクラスのおかげで、アラーム通知のメッセージ変更や、アラーム表示のデザイン変更（例：リストにIDを追加するだけ→画面全体を点滅させる）が、ビジネスロジックに一切影響を与えずに行えます。

