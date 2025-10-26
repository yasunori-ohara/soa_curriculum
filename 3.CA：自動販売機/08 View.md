# 08 View

# 📺 View : `vending_machine/interface_adapters/view_console.py`

いよいよ一番ユーザー寄りの層、`View` です。
この章ではコンソールUI（ターミナル上の入出力）を例にして、`ConsoleView` を実装します。

`ConsoleView` は自販機にとっての「ボタンと小さな画面」。
ユーザーが押した操作を受け取り（入力）、結果メッセージを表示します（出力）。

つまり責務はこの2つだけです👇

1. 🧑‍💻 ユーザーからの操作（「A1ください」「100円入れる」など）を受け取る
   → `Controller`に伝える

2. 🖥 表示すべきメッセージを画面に出す
   → `Presenter`が準備した `ViewModel` の内容をそのまま出す

`ConsoleView` は、ビジネスルールもハードウェアの扱いも知りません。
ただ橋渡しをする「窓口」です。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## 🎯 View はどこに属しているの？

フォルダ構成の中では、`View` は `interface_adapters` フォルダに入ります。

```text
vending_machine/
├─ domain/
│   ├─ entities.py             # Item / PaymentManager など（純粋なビジネスルール）
│   └─ errors.py               # ドメイン固有の例外
│
├─ usecase/
│   ├─ dto.py                  # InputData / OutputData / ViewModel
│   ├─ boundaries.py           # UseCaseが外部に求める契約
│   ├─ select_item_usecase.py  # 商品購入ユースケース
│   └─ insert_coin_usecase.py  # コイン投入ユースケース（後で紹介）
│
├─ interface_adapters/
│   ├─ controller.py           # 入力を受けてUseCaseを呼ぶ（受付係）
│   ├─ presenter.py            # UseCaseの結果→ViewModel（翻訳係）
│   ├─ view_console.py         # ← いまここ（画面そのもの）
│   ├─ data_access.py          # 在庫リポジトリのインメモリ実装
│   └─ hardware_adapter.py     # ハードウェア操作（printでシミュレート）
│
└─ main.py                     # 全部を配線して起動する場所
```

この図でいうと、`View` は最も外側の世界＝ユーザーと話す担当です。
でもビジネスルール（「買えるか？」「在庫あるか？」）は内側に聞きに行きます。直接決めません。

---

## ✅ View がやっていいこと / ダメなこと

⭕ やっていいこと

* `print()` / `input()` のような画面・入力デバイス操作
* ユーザー入力に対応して、`Controller` のメソッドを呼ぶ
* `ViewModel`（Presenterが最後に用意した表示用オブジェクト）を読んで、画面に出す

❌ やってはいけないこと

* 在庫があるか、支払いが成立しているか、といった「ビジネス判断」
  → それは `UseCase` と `Entity` の責務
* お釣りの金額やメッセージの文章を考える
  → それは `Presenter` の責務
* モーター回転やコイン払い出しなどハード制御
  → それは HardwareAdapter の責務

> View のモットーは「聞いて、伝えて、映すだけ」。
> かしこくならないことが大事です。

---

## 💻 コード（`view_console.py`）

```python
# vending_machine/interface_adapters/view_console.py

# ControllerとViewModelはどちらも interface_adapters / usecase の世界にいる仲間
from vending_machine.interface_adapters.controller import VendingMachineController
from vending_machine.usecase.dto import VendingMachineViewModel
from vending_machine.domain.entities import PaymentManager


# -----------------------------------------------------------------------------
# ConsoleView
# - クラス図の位置: View
# - 同心円図の位置: interface_adapters（最も外側の円のひとつ）
#
# 役割:
#   - ユーザーからの入力を受けて Controller に伝える
#   - Presenterが書き込んだ ViewModel の内容を画面に表示する
#
# つまり人間とアプリケーションの橋渡し係。ロジックは持たない。
# -----------------------------------------------------------------------------
class ConsoleView:
    """
    コンソール版の自動販売機UI。

    - 入力担当:
        ユーザーに「何をしますか？」と尋ねて、押された操作をControllerに伝える。
    - 出力担当:
        Presenterがまとめた ViewModel の内容をそのままprintする。
    """

    def __init__(
        self,
        controller: VendingMachineController,
        view_model: VendingMachineViewModel,
    ):
        """
        [依存性の注入]

        controller:
            ユーザー操作をアプリケーションに伝えるための窓口（受付係）。
            ViewはControllerを通してしかUseCaseに触れない。

        view_model:
            Presenterが更新してくれる「表示用の最新メッセージの置き場」。
            Viewはこれを読むだけでよい。自分で文面は作らない。
        """
        self._controller = controller
        self._view_model = view_model

        # 自販機の現金投入状態を表すエンティティ。
        # ここでは、1回のアプリ実行中ずっと使い回す想定。
        # （100円入れて、さらに100円入れて、最後に購入…という流れを成立させるため）
        self._payment_manager = PaymentManager()

    def run(self):
        """
        [UIメインループ]
        - ユーザーに操作メニューを提示する
        - 入力を受け付ける
        - Controllerを呼ぶ
        - Presenterが整えたメッセージ(ViewModel)を表示する

        無限ループで動き続け、'q' を押すと終了する想定。
        """
        while True:
            print("\n--------------------")
            print(f"現在の投入金額: {self._payment_manager.current_amount}円")
            print("操作を選んでください:")
            print("  c: コイン投入")
            print("  s: 商品を購入する")
            print("  r: お釣りを返却する")
            print("  q: 終了する")

            action = input("> ").strip().lower()

            try:
                if action == "c":
                    # コイン投入イベント
                    coin_str = input("投入する硬貨（10 / 50 / 100 / 500）: ").strip()
                    coin = int(coin_str)

                    # Viewは「ユーザーが100円入れましたよ」という事実だけをControllerに伝える。
                    self._controller.insert_coin(coin)

                elif action == "s":
                    # 商品購入イベント
                    slot_id = input("購入する商品のスロットID (例: A1): ").strip()

                    # Viewは「ユーザーがA1を買いたいと言ってます」という事実だけをControllerに伝える。
                    self._controller.select_item(slot_id)

                elif action == "r":
                    # お釣り返却イベント
                    self._controller.return_change()

                elif action == "q":
                    print("ご利用ありがとうございました。")
                    break

                else:
                    # ここはUIロジック（プレゼンテーションロジック）。
                    # ビジネスではなく「どのキーが押されたか」の話なので、Viewが持ってよい。
                    self._view_model.display_text = "無効な操作です。c/s/r/q から選んでください。"

            except ValueError:
                # 例: ユーザーが "abc" と入力して int() に失敗した など
                self._view_model.display_text = "エラー: 入力の形式が正しくありません。"
            except Exception as e:
                # 想定外エラーも、最終的にはユーザーに見せるメッセージに落とす
                # ここは「UIとしてのハンドリング」なのでViewに置いてOK
                self._view_model.display_text = f"予期せぬエラー: {e}"

            # Presenter or このViewがセットした display_text の内容を描画
            self.render()

    def render(self):
        """
        [表示処理]
        Presenterによって更新されたViewModelの内容をそのまま表示する。
        Viewは文面を組み立てない。あくまで「表示するだけ」。
        """
        print(f"\nディスプレイ: {self._view_model.display_text}")
```

---

### 👀 あれ？ `PaymentManager` は使ってないように見えるけど…？

ここで少しだけ構成の話をします。

* `PaymentManager` は「今いくら投入されてるか？」を覚えておくエンティティです。
* これはユーザーがコインを入れるたびに更新され、購入後や返金後にリセットされます。
* 第三巡の設計では、`PaymentManager` は **アプリ起動中に生きている1つの状態** として扱うことを想定しています。

どこがそれを持つべきか？は複数パターンがあります：

1. Controllerが保持して UseCaseに渡す
2. Viewが保持して Controllerに見せる
3. main.py で作って両方に渡す（依存注入）

この教材では最終的に (3) を採用します。
つまり、`main.py` の Composition Root で `PaymentManager` を1個作り、
ControllerにもViewにも共有させる形にするとスッキリします。

このあと `main.py` の章で、この配線（依存注入）をはっきり書きます。

---

## 🧪 View のテスト

`View` のテストは少し特殊です。
`input()` と `print()` が生で入っているからです。
そこは `unittest.mock.patch` を使って差し替えます。

ゴールは「ユーザーがこう入力したら、Controllerのこのメソッドが呼ばれるか？」です。
ビジネス結果（在庫が減ったか、お釣りが返ったか）までは見ません。
それはUseCaseやPresenterのテストでやれば十分なので、ここは責務をしぼります。

```python
# tests/interface_adapters/test_view_console.py

from unittest.mock import patch, MagicMock
from vending_machine.interface_adapters.view_console import ConsoleView
from vending_machine.usecase.dto import VendingMachineViewModel


def test_cが押されたら_controllerのinsert_coinが呼ばれる():
    # 1. Arrange
    fake_controller = MagicMock()
    vm = VendingMachineViewModel()

    # Viewを作成
    view = ConsoleView(
        controller=fake_controller,
        view_model=vm,
    )

    # 2. Act
    # ユーザーが:
    #   "c" (コイン投入モードに入る)
    #   "100" (100円投入)
    #   "q" (終了)
    with patch("builtins.input", side_effect=["c", "100", "q"]):
        with patch("builtins.print"):  # print汚染を抑える
            view.run()

    # 3. Assert
    # Controller.insert_coin() が 100 で呼ばれたことを確認
    fake_controller.insert_coin.assert_called_once_with(100)
```

ポイント：

* Viewの責務テストに集中している
* PresenterやUseCaseの中身はモック化して気にしない
* 「正しいレイヤーで正しい仕事をしてるか？」だけを見る

---

## 🐍 Python と 🔧 C言語での違い（イメージ）

* Python + クリーンアーキテクチャ版：

  * `View` は「入出力の担当」だけ
  * `Controller` は「ユースケースを呼ぶ受付」
  * `UseCase` は「ビジネスの流れ」
  * `HardwareAdapter` は「物理的にガコンと動く手足」

* Cでありがちな自販機コード：

  * `while(1)`のループの中で

    * ボタン読み取り
    * 在庫チェック
    * お金チェック
    * モーター回転
    * LED表示
      を全部やる
  * → 1ファイルが巨大化する
  * → 一部分だけテストするのが難しい
  * → ハードが変わると全部書き直し

今回の分割は、「現場での変更コスト」と「テストのしやすさ」を両立するための現実的なやり方です。

---

## 🛡️ View層の鉄則

> 愚直であれ (Be Dumb)

* Viewは「どのキーが押されたか」を判断して Controller に伝える
* Presenterが用意した文をそのまま出す
* それ以上の“頭の良さ”を持たないことが、長生きするUIの秘訣

ここまでで、

* Entity（ビジネスの芯）
* UseCase（業務フロー）
* Adapter群（Controller / Presenter / Hardware / View）
  がそろいました。


