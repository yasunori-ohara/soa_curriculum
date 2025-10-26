# 02 Use Case

# 👨‍🏫 Use Case（アプリケーションの手順を司る層）

### `vending_machine/usecase/select_item_usecase.py`

### `vending_machine/usecase/insert_coin_usecase.py`

---

## 🎯 UseCaseとは何か

UseCaseは「自動販売機の具体的な操作フロー」を表現する層です。
たとえば次のような場面を考えます。

* 「100円玉を投入した」
* 「A1のボタンを押して商品を購入した」

このとき起きる処理は単純ではありません。複数の役割が連携します。

* お金の状態を更新する
* 在庫を確認する
* お釣りを計算する
* 商品を物理的に排出するように指示する
* 在庫を更新して保存する
* 画面メッセージを整えて表示側に渡す

この一連の流れを正しい順序で制御するのがUseCaseの役割です。
ここでは、次の2つのユースケースを扱います。

* 💰 `InsertCoinUseCase` …… 硬貨を投入する操作
* 🥤 `SelectItemUseCase` …… 商品ボタンを押して購入する操作

---

## 🧠 UseCase層が担う責任

UseCase層は「アプリケーション固有の業務手順」を表します。
Entityを直接操作するのではなく、**Entity同士と外部装置を正しく連携させる“司令塔”** の役割を持ちます。

⭕ UseCaseに含めるべき責務

* 複数のEntityを正しい順序で呼び出す（オーケストレーション）
* 必要な外部処理（在庫保存・ハードウェア制御など）を、対応するインターフェースに依頼する
* 実行結果を、プレゼンテーション層に渡せる中立なデータ（DTO）にまとめる

❌ UseCaseに含めてはいけない責務

* 画面表示の文言を決める（Presenterの責務）
* 物理的なモーター制御の詳細を書く（HardwareAdapter側の責務）
* DBやメモリなど具体的な保存方法を書く（Repository実装側の責務）
* エンティティが本来持つべきルールを複製する
  （例：「在庫があるか？」をUseCase側で直接 `if item.stock > 0` と判定しない。`Item.dispense()` に任せる）

> UseCaseは、「何が起きるべきか（ビジネスフロー）」は知っているが、
> 「どう実装されているか（詳細）」は知らないように設計します。

---

## 📂 関連ファイルの位置づけ

第三巡のフォルダでは、UseCase層は次のファイル群で構成されます。

```text
vending_machine/
├─ domain/
│   ├─ entities.py            # Entity: Item, PaymentManagerなど
│   └─ errors.py              # ドメイン固有の例外
│
├─ usecase/
│   ├─ dto.py                 # DTO / ViewModel（データの入出力用の箱）
│   ├─ boundaries.py          # インターフェース（契約）
│   ├─ insert_coin_usecase.py # 硬貨投入ユースケース
│   └─ select_item_usecase.py # 商品購入ユースケース
│
├─ interface_adapters/
│   ├─ controller.py          # 入力受付（ボタン操作など）
│   ├─ presenter.py           # 表示用メッセージ整形
│   ├─ view_console.py        # 実際の入出力（CLI想定）
│   ├─ data_access.py         # 在庫リポジトリ実装（メモリ版など）
│   └─ hardware_adapter.py    # ハードウェア実装（モーター/返却動作など）
│
└─ main.py                    # 依存性をつなぎ合わせて起動する場所
```

✍ 用語整理：

* `boundaries.py`
  → UseCaseが外部に依頼するための「契約インターフェース」を定義します。
  例：`ItemRepository`（在庫取得/保存の契約）、`HardwareInterface`（ハード制御の契約）、
  `SelectItemOutputBoundary`（Presenterに結果を伝えるための契約）

* `dto.py`
  → レイヤー間を渡るデータの箱（DTO）と、UI向けのViewModelを定義します。

UseCaseは、上の2つ（boundaries / dto）を使って他レイヤーとやり取りします。

---

## 💰 例1: `InsertCoinUseCase`

硬貨が投入されたときの流れを扱うユースケースです。
このユースケースは「お金の状態を更新する」という1つの目的に特化しています。

### 👀 責務のイメージ

* ユーザーが100円玉を入れる
* `PaymentManager.insert_coin(100)` を呼び、ビジネスルールどおりに金額を積み上げる
* 現在の投入金額を Presenter に渡して、画面に反映させる

ここでは、ハードウェア制御も在庫処理も発生しません。
「支払い状態（投入金額）を更新して通知する」というシンプルなユースケースです。

### 💻 コード例（`insert_coin_usecase.py`）

```python
# vending_machine/usecase/insert_coin_usecase.py

from vending_machine.usecase.boundaries import (
    InsertCoinInputBoundary,        # Controllerが呼び出す入り口
    InsertCoinOutputBoundary,       # Presenterが実装する出口
)
from vending_machine.usecase.dto import (
    InsertCoinInputData,            # Controller -> UseCase への入力DTO
    InsertCoinOutputData,           # UseCase -> Presenter への出力DTO
)
from vending_machine.domain.entities import PaymentManager


class InsertCoinUseCase(InsertCoinInputBoundary):
    """
    硬貨投入ユースケース。
    ユーザーが硬貨を入れた際のビジネスフローを制御する。

    このクラスは「アプリケーションとして何をするべきか」を知っているが、
    - 入出力の方法（CLIかGUIか）
    - 表示文言の最終形
    などは知らない。
    """

    def __init__(
        self,
        presenter: InsertCoinOutputBoundary,
        payment_manager: PaymentManager,
    ):
        """
        presenter:
            結果をユーザー向けのメッセージに整形する担当。
            InsertCoinOutputBoundary インターフェースを満たす必要がある。

        payment_manager:
            現在の投入金額などを保持・管理するエンティティ。
            1台の自販機で共有される「お金の状態」を表す。
        """
        self._presenter = presenter
        self._payment_manager = payment_manager

    def handle(self, input_data: InsertCoinInputData) -> None:
        """
        ユーザーが硬貨を投入したときに呼び出されるメイン処理。

        1. PaymentManager にコイン投入を反映する
        2. 現在の投入金額をDTO化する
        3. Presenterに渡す
        """
        # 1. ドメインエンティティに処理を委譲する
        #    - 受け付け可能な硬貨かどうかは PaymentManager が判断する
        #      (UseCase側では重複チェックしない)
        self._payment_manager.insert_coin(input_data.coin)

        # 2. Presenterに渡すための出力DTOを作る
        output_data = InsertCoinOutputData(
            current_amount=self._payment_manager.current_amount
        )

        # 3. Presenterに結果を通知する
        self._presenter.present_coin_inserted(output_data)
```

---

## 🥤 例2: `SelectItemUseCase`

購入ボタンが押されたときのフローをまとめます。
これは本教材の中心となるユースケースです。

### 📘 処理の全体像

ユーザーが「A1のボタン」を押した場合、次のことが起こります。

1. スロットID（例: "A1"）から `Item`（商品情報＋在庫）を取得する
   → `ItemRepository` に依頼

2. 現在の投入金額でその商品が買えるか確認する
   → `PaymentManager` に依頼

3. 問題なければ購入を成立させる

   * お釣りを計算する
   * 商品を物理的に排出するように依頼する
     → `HardwareInterface.dispense_item(slot_id)`
   * お釣りがあれば返却動作を依頼する
     → `HardwareInterface.return_change(amount)`
   * 商品在庫を1つ減らす (`Item.dispense()` を呼ぶ)
   * 変更後の商品の状態を保存する
     → `ItemRepository.save(item)`

4. 最終的な結果

   * 「どの商品を出したか」
   * 「いくらお釣りを返したか」
     をDTOとしてまとめ、Presenterに渡す

→ UseCaseは、フローの順序や組み合わせを担当します。
→ ただし、細かいルール（在庫があるか/お金が足りるか）はEntityに任せます。

### 💻 コード例（`select_item_usecase.py`）

```python
# vending_machine/usecase/select_item_usecase.py

from vending_machine.usecase.boundaries import (
    SelectItemInputBoundary,        # Controllerが呼ぶ入り口
    SelectItemOutputBoundary,       # Presenterが実装する出口
    ItemRepository,                 # 在庫アクセスの契約
    HardwareInterface,              # ハードウェア制御の契約
)
from vending_machine.usecase.dto import (
    SelectItemInputData,            # Controller -> UseCase 入力DTO
    SelectItemOutputData,           # UseCase -> Presenter 出力DTO
)
from vending_machine.domain.entities import Item, PaymentManager


class SelectItemUseCase(SelectItemInputBoundary):
    """
    商品購入ユースケース。
    「指定スロットの商品を購入したい」という要求を処理する。

    このクラスは、1回の購入に関わる流れ全体を管理する。
    ただし、在庫の正しさ・支払いの正しさ・ハードウェア制御の詳細は
    それぞれの担当に委譲する。
    """

    def __init__(
        self,
        presenter: SelectItemOutputBoundary,
        item_repository: ItemRepository,
        hardware: HardwareInterface,
    ):
        """
        presenter:
            購入結果をユーザー向けの表示用データに整形する担当。
            SelectItemOutputBoundary インターフェースを満たす必要がある。

        item_repository:
            在庫や価格など、スロットに関するデータへアクセスする窓口。
            具体的にメモリで持つかDBで持つかは知らない。

        hardware:
            「商品を排出してほしい」「お釣りを返してほしい」といった
            物理的な動作を依頼する窓口。UseCaseは物理詳細を知らない。
        """
        self._presenter = presenter
        self._item_repository = item_repository
        self._hardware = hardware

    def handle(self, input_data: SelectItemInputData, payment_manager: PaymentManager) -> None:
        """
        購入処理の本体。

        引数:
            input_data:
                購入対象スロットを示すDTO（例: slot_id="A1"）
            payment_manager:
                現在の投入金額などの状態を持つエンティティ

        処理手順:
            1. 商品の取得
            2. 決済処理
            3. ハードウェアへの指示
            4. 在庫の更新と保存
            5. Presenterへの通知
        """

        # -------------------------------------------------
        # 1. スロットIDから商品情報を取得する
        # -------------------------------------------------
        item: Item | None = self._item_repository.find_by_slot_id(input_data.slot_id)
        if not item:
            # スロット自体が存在しない、あるいは未登録の場合
            raise ValueError("指定されたスロットに商品はありません。")

        # -------------------------------------------------
        # 2. 決済処理を行う（お金が足りるか？ お釣りはいくらか？）
        #    ルールは PaymentManager が知っている
        # -------------------------------------------------
        change = payment_manager.process_purchase(item)
        # ここでお釣りが算出され、payment_manager.current_amount は0にリセットされる

        # -------------------------------------------------
        # 3. ハードウェアへの指示
        #    UseCaseは「何をしてほしいか」だけを伝える
        # -------------------------------------------------
        # 3-1. 商品を物理的に排出するよう依頼
        self._hardware.dispense_item(item.slot_id)

        # 3-2. お釣りがあれば返金動作を依頼
        if change > 0:
            self._hardware.return_change(change)

        # -------------------------------------------------
        # 4. 在庫を1本減らし、保存する
        #    在庫の妥当性チェックは Item.dispense() が担当する
        # -------------------------------------------------
        item.dispense()            # 在庫が0ならここで例外になる
        self._item_repository.save(item)

        # -------------------------------------------------
        # 5. Presenterに渡すための結果をDTOにまとめる
        #    UseCaseはUI用の表現は行わず、事実のみ伝える
        # -------------------------------------------------
        output_data = SelectItemOutputData(
            item_name=item.name,
            change=change,
        )

        self._presenter.present_purchase_result(output_data)
```

---

## 📦 DTO（Data Transfer Object）とViewModel

UseCaseは、外部とやり取りする際に「DTO」というデータ専用の入れ物を使います。
これは `vending_machine/usecase/dto.py` に定義されます。

例として、ここで利用した型は以下のようなイメージです。

```python
# vending_machine/usecase/dto.py
from dataclasses import dataclass

# 硬貨投入の入力DTO
@dataclass(frozen=True)
class InsertCoinInputData:
    coin: int

# 硬貨投入の出力DTO
@dataclass(frozen=True)
class InsertCoinOutputData:
    current_amount: int  # 現在の投入合計

# 商品購入の入力DTO
@dataclass(frozen=True)
class SelectItemInputData:
    slot_id: str         # "A1" など

# 商品購入の出力DTO
@dataclass(frozen=True)
class SelectItemOutputData:
    item_name: str       # 出した商品の名前
    change: int          # 返却されたお釣りの金額

# View向けの最終表示データ（Presenterが生成する）
@dataclass
class VendingMachineViewModel:
    display_text: str = ""
```

* `@dataclass(frozen=True)` は「イミュータブル（後から書き換え禁止）」です。

  * Controller から UseCase に渡す入力
  * UseCase から Presenter に渡す出力
    など、境界を越えるデータは勝手に変更されないほうが安全なので凍結します。

* `VendingMachineViewModel` は凍結しません。
  Presenterが更新し、Viewがそれを表示します。

これらの詳細（DTOとViewModel）は、後の章「Data Structures」で改めて整理します。

---

## 🧪 なぜUseCaseはテストしやすいのか

UseCaseは、具体的なハードウェアや保存先を知りません。
代わりに、抽象インターフェース（`boundaries.py` で定義されたもの）に依存します。

テスト時には、次のような「フェイク実装」を渡します。

* フェイクの `ItemRepository`（メモリ上の辞書など）
* フェイクの `HardwareInterface`（呼ばれた内容を記録するだけ）
* フェイクの `Presenter`（受け取ったDTOを保持しておくだけ）

これにより、実機もDBもなく、ローカル環境だけでユースケースのロジック全体を検証できます。

---

## 🐍 Python と 組み込み的な書き方の比較

🧠 クリーンアーキテクチャ的なPython設計

* `SelectItemUseCase` が「購入」というシナリオの順序と条件を定義する。
* 在庫の正しさは `Item`、支払いの正しさは `PaymentManager`、
  ハードの制御は `HardwareInterface`、表示整形は `Presenter` に任せる。
* 責務が明確に分かれるため、部品ごとに交換可能。

🔧 組み込みでありがちな一体型の実装

* メインループ（または割り込みハンドラ）の中で
  ボタン入力→在庫チェック→金額判定→モーター操作→LED表示
  をひと続きの処理として記述しがち。
* 修正時に全体へ影響が波及しやすい。テストも実機依存になりやすい。

クリーンアーキテクチャでは、この「全部まとめて1か所に書く」という状態を避け、
**ユースケース単位に責務を分離**することで、保守性・テスト性・交換性を高めます。

---

## 🛡 UseCase層の鉄則

> ビジネスフローを管理せよ。
> しかし詳細実装（保存方法・表示形式・ハード駆動）は他者に委ねよ。

* UseCaseは、ユーザーの操作に対する正式な応答手順を定義します。
* エンティティのルールを尊重し、外部とのやり取りはインターフェース越しに依頼します。
* UI・ハードウェア・データ保存方法が変わっても、UseCaseは変わらない設計を目指します。

次章では、この「インターフェース越しの依頼」を可能にする `boundaries.py`（境界インターフェース）を扱います。
そこでは、UseCaseが外部に対して「こういう機能を持っていてほしい」と要求する契約を定義します。
