# 06 Presenter

# 🎨 Presenter
### `vending_machine/interface_adapters/presenter.py`

UI側のアダプターのうち、まずは `Presenter` を解説します。
`Presenter` は、`UseCase` の実行結果をユーザーに見せる形まで“翻訳”する担当です。

* UseCase は「事実（商品名・お釣りなど）」を返す
* Presenter は、それを「伝える文章」にする
* View は、Presenterが整えた表示用データを画面に出すだけ

この分業によって、ビジネスロジックと表示ロジックが完全に分離されます。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## 🎯 Presenter の役割

`VendingMachinePresenter` は、`UseCase` から受け取った結果（`SelectItemOutputData`）をもとに、
ユーザーにとってわかりやすい表示用メッセージを組み立て、
`ViewModel` に反映します。

ここで大事なのは：

* UseCaseは「お釣り30円」という“数字の事実”しか知らない
* 画面の文言（「お釣りは30円です。」など）は Presenter が決める
* View は Presenter が用意した文字列をそのまま出すだけにする

👉 これによって「UIの言い回しを変えたいからUseCaseを書き換える」という地獄から解放されます。

---

## ✅ Presenterに書くべきこと / 書いてはいけないこと

⭕ Presenterに含めるべきこと

* `SelectItemOutputBoundary`（UseCaseが期待する“出力の相手”）の実装
* `SelectItemOutputData` → `VendingMachineViewModel` への変換
* 表示用の細かいロジック
  例：「お釣りが0円なら、お釣りメッセージは書かない」など

❌ Presenterに含めてはいけないこと

* ビジネスルールの判断
  （「この商品は売っていいか？」はUseCaseやEntityで決まっている）
* ハードウェア制御（それはHardwareAdapterの役割）
* 直接 print() などの画面出力（それはViewの役割）

> 🎤 Presenter は「表示用メッセージを作るところまで」。
> それを実際にどこに出すかは View の仕事。

## 💻 フォルダ構成
```text
vending_machine/
├─ domain/
│   ├─ entities.py          # Item, PaymentManager など
│   ├─ errors.py            # ドメインルール違反例外
│
├─ usecase/
│   ├─ dto.py               # InputData / OutputData / ViewModel
│   ├─ boundaries.py        # UseCaseが外部とやり取りする契約
│   ├─ insert_coin_usecase.py
│   └─ select_item_usecase.py
│
├─ interface_adapters/
│   ├─ controller.py        # Controller（入力受付）
│   ├─ presenter.py         # Presenter（出力整形）    <- いまここ
│   ├─ view_console.py      # CLIでの動作確認
│   ├─ data_access.py       # 在庫リポジトリのインメモリ実装
│   └─ hardware_adapter.py  # ハードウェア操作のコンソール版実装
│
└─ main.py                  # Composition Root（全配線）

```

## 💻 コード（`presenter.py`）

```python
# vending_machine/interface_adapters/presenter.py

# UseCaseが定義した「出力先の契約」と「やりとりするデータ型」をインポート
from vending_machine.usecase.boundaries import SelectItemOutputBoundary
from vending_machine.usecase.dto import (
    SelectItemOutputData,
    VendingMachineViewModel,
)


# -----------------------------------------------------------------------------
# VendingMachinePresenter
# - クラス図の位置: Presenter
# - 同心円図の位置: interface_adapters（外側の円）
#
# このクラスは UseCase の結果を、View がそのまま表示できる形に整える。
# -----------------------------------------------------------------------------
class VendingMachinePresenter(SelectItemOutputBoundary):
    """
    UseCaseの「出力先」として呼び出される役。
    役割は、ビジネス結果(SelectItemOutputData)を
    ユーザー向けの文言(ViewModel)に変換して保持すること。
    """

    def __init__(self, view_model: VendingMachineViewModel):
        """
        [状態の注入 / 共有]

        Presenterは ViewModel を保持します。
        この ViewModel は View と共有されており、
        Presenterが書き換えると View は最新の表示内容を読めるようになります。

        ここで渡される view_model は「表示用の状態の入れ物」です。
        """
        self._view_model = view_model

    def present(self, output_data: SelectItemOutputData):
        """
        [インターフェースの実装 / OutputBoundaryの実体]

        UseCase から呼び出されるメソッド。
        「購入処理の結果」を、ユーザーに見せる文章として組み立て、
        ViewModelに反映する。

        output_data:
            UseCaseが返してきた“事実だけ”のデータ
              - item_name: 出した商品名（例: "お茶"）
              - change:    返却したお釣りの金額（例: 30）
        """

        # 1. ベースとなるメッセージを組み立てる
        message = f"「{output_data.item_name}」が出てきました。"

        # 2. お釣りがあるときだけ、追記する
        #    「お釣りは0円です。」のような不要な文言は避けたいので
        #    この分岐は Presenter の責務になります。
        if output_data.change > 0:
            message += f" お釣りは{output_data.change}円です。"

        # 3. ViewModelを更新する
        #    ViewはこのViewModelを見て表示するだけでよい
        self._view_model.display_text = message
```

---

## 🔍 コード解説のポイント

### 1. `class VendingMachinePresenter(SelectItemOutputBoundary):`

この継承が宣言しているのはこういうことです👇
「私はUseCaseが期待する“出力先インターフェース（OutputBoundary）”を満たします」

UseCaseは `SelectItemOutputBoundary` 型として Presenter に依存するので、
「誰が画面に出すか」を知らなくても仕事できます。
（CLI用Presenterでも、Web用PresenterでもOKになる）

これが依存性逆転の力。

---

### 2. `present()` の中では“表示用ロジック”だけやる

ここでは「お釣りが0円ならメッセージに書かない」といった、
人間にとって読みやすい表現の分岐をやっています。

これは**ビジネスルールではない**ので、EntityやUseCaseに入れるべきではありません。
このレイヤーにあるべきロジックです。

---

### 3. `self._view_model.display_text = ...`

Presenterは最終的なメッセージを ViewModel に書き込みます。
View はただそれを読むだけでよくなります。

→ View側の責務が「画面に出す」だけになり、めっちゃ薄くなる
→ コンソールUIでもGUIでもWebでも、Viewを差し替えるだけで再利用できる

---

## 🧪 Presenterのユニットテスト

Presenter単体のテストはとてもやりやすいです。
なぜなら、ハードウェアにもDBにも依存せず、「文字列をどう作るか」だけを見るから。

```python
# tests/interface_adapters/test_presenter.py

from vending_machine.usecase.dto import (
    SelectItemOutputData,
    VendingMachineViewModel,
)
from vending_machine.interface_adapters.presenter import VendingMachinePresenter


def test_お釣りがある場合_メッセージにお釣りが含まれる():
    # 1. Arrange (準備)
    view_model = VendingMachineViewModel()
    output_data = SelectItemOutputData(item_name="お茶", change=30)

    presenter = VendingMachinePresenter(view_model)

    # 2. Act (実行)
    presenter.present(output_data)

    # 3. Assert (検証)
    expected = "「お茶」が出てきました。 お釣りは30円です。"
    assert view_model.display_text == expected


def test_お釣りがゼロの場合_メッセージにお釣りは含まれない():
    # 1. Arrange
    view_model = VendingMachineViewModel()
    output_data = SelectItemOutputData(item_name="コーヒー", change=0)

    presenter = VendingMachinePresenter(view_model)

    # 2. Act
    presenter.present(output_data)

    # 3. Assert
    expected = "「コーヒー」が出てきました。"
    assert view_model.display_text == expected
```

このテストはこういうことを保証します：

* UseCaseが返してきた「事実のデータ」から
* Presenterが正しく「ユーザー向けの言い回し」を構築している
* ViewModelがその結果を保持している

UI変更に強い設計、というのはこうして証明できます。

---

## 🐍 Python と 🔧 C言語のイメージ比較

* 🐍 Python（オブジェクト指向 / クリーンアーキテクチャ）
  Presenterは「文字列に整える」ことに専念し、Viewに渡す最終形を責任もって作るクラスとして定義される。

* 🔧 C言語でよくあるパターン

  * `render_message()` のような関数が直接 `printf()` までやってしまいがち
  * つまり「メッセージの組み立て」と「画面への出力」がくっつきやすい
  * その結果、画面表示の仕様変更＝ロジック全体の改修になりやすい

Presenterを分けることで、「どう見せるか」はいつでも安全に差し替えられるようになります。

---

## 🛡️ Presenter層の鉄則

> 表示のために翻訳せよ、ただしビジネスを語るな。
> (Translate for presentation, but don't speak business.)

* Presenterは、Userに見せる“ことば”を決める場所。
* UseCaseのビジネスルールに口出ししない。
* Viewへの最終的な受け渡しを、きれいな形に整える。

これで、UIは何度でも作り変えられるようになります。
CLIでも、Webでも、ディスプレイ付き自販機でも。

