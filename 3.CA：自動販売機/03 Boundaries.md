# 03 Boundaries（境界インターフェース群）

### `vending_machine/usecase/boundaries.py`

---

## 🎯 この章の目的

この章では、UseCaseが「外部にこう動いてほしい」と要求するためのインターフェース（境界）を定義します。
このファイルには大きく3種類のインターフェースが含まれます。

1. 🧾 入力境界（Input Boundary）

   * Controller → UseCase
   * 例：`SelectItemInputBoundary`

2. 📤 出力境界（Output Boundary）

   * UseCase → Presenter
   * 例：`SelectItemOutputBoundary`

3. 🗄 外部サービス境界（Repository / Hardware）

   * UseCase → データアクセス / ハードウェア
   * 例：`ItemRepository`, `HardwareInterface`

これらはすべて「契約（インターフェース）」だけを定義し、**実装は外側の層に任せる**というのがポイントです。

---

## 🧭 Boundariesが果たす役割

Boundariesは「内側（ユースケース側）が外側に依存しないための防波堤」です。
ここで定義されたインターフェースをもとに、UseCaseは必要な処理を“依頼”できます。

重要なのは順方向ではなく逆方向の依存です。

* UseCase（内側）は、RepositoryやPresenterの**具体的な実装**は知らない
* 代わりに、Boundariesに定義された**抽象インターフェース**に依存する
* 具体実装（例：InMemoryRepository、ConsolePresenter、ConsoleHardwareAdapter）は外側の層で作られ、あとから注入される

> これは「依存性逆転の原則（Dependency Inversion Principle）」の直接的な適用です。
> 内側のポリシー（ビジネスロジック）は、外側の詳細実装を知らないまま動けなければならない、という考え方です。

---

## 📂 ファイル構成における位置づけ

第三巡のディレクトリ構成では、Boundariesは `usecase/` に含まれます。

```text
vending_machine/
├─ domain/
│   ├─ entities.py            # Item, PaymentManager などのビジネスルール
│   └─ errors.py              # ドメイン固有の例外
│
├─ usecase/
│   ├─ dto.py                 # DTO / ViewModel（データの箱）
│   ├─ boundaries.py          # ← この章で説明するインターフェース群
│   ├─ insert_coin_usecase.py # 硬貨投入ユースケース
│   └─ select_item_usecase.py # 商品購入ユースケース
│
├─ interface_adapters/
│   ├─ controller.py          # 入力受付（ボタン操作など）
│   ├─ presenter.py           # 出力整形（メッセージ作成）
│   ├─ view_console.py        # CLIでの入出力
│   ├─ data_access.py         # 具体的な在庫管理の実装（メモリ版など）
│   └─ hardware_adapter.py    # 具体的なハード制御の実装（printで代用）
│
└─ main.py                    # 依存性をまとめて接続し、アプリを起動
```

ここで特に重要なのは、`infrastructure/` ディレクトリを作らず、`interface_adapters/` に外部詳細（データアクセス・ハードウェア）を集約している点です。

* `ItemRepository` の実装は `interface_adapters/data_access.py`
* `HardwareInterface` の実装は `interface_adapters/hardware_adapter.py`

→ つまり「データの保存方法」「ハードウェアの駆動方法」はアプリケーションの外側の詳細として扱われ、UseCaseはそれらを直接知らずに済むようになっています。

---

## 💻 ソースコード（`boundaries.py`）

以下に、`usecase/boundaries.py` の全体例を示します。
コメントは詳細に記載し、特に層間の役割がわかるようにしています。

```python
# vending_machine/usecase/boundaries.py
# -----------------------------------------------------------------------------
# このファイルは UseCase 層から外部へ向けた「依頼の窓口（契約）」をまとめて定義する。
#
# ✅ 目的
#   - Controller が UseCase を呼び出すための入り口（Input Boundary）
#   - UseCase が Presenter に結果を渡すための出口（Output Boundary）
#   - UseCase がデータ保存やハードウェアに依頼するための窓口（Repository / Hardware）
#
# ❗ ポイント
#   - ここには "インターフェース（抽象クラス）" だけを書く。
#   - 具体的に「どうやるか」の実装は書かない。
#   - UseCase はこれらの抽象に対してプログラムし、外側の詳細には依存しない。
#
# レイヤー関係（内→外）
#   UseCase ----(依頼)----> ItemRepository / HardwareInterface
#   UseCase ----(通知)----> OutputBoundary(Presenter)
#   Controller -(呼び出し)-> InputBoundary(UseCase)
#
# -----------------------------------------------------------------------------

from __future__ import annotations
from abc import ABC, abstractmethod
from typing import Protocol, Optional

from vending_machine.usecase.dto import (
    InsertCoinInputData,
    InsertCoinOutputData,
    SelectItemInputData,
    SelectItemOutputData,
)
from vending_machine.domain.entities import Item


# -----------------------------------------------------------------------------
# 🟦 1. Input Boundary（入力境界）
#    Controller → UseCase の「呼び出し口」の契約
# -----------------------------------------------------------------------------

class InsertCoinInputBoundary(ABC):
    """
    硬貨投入ユースケースを起動するための入り口。
    Controller はこのインターフェースだけを知っていれば、
    具体的な実装（どんな内部処理が走るのか）を知らなくても呼び出せる。
    """

    @abstractmethod
    def handle(self, input_data: InsertCoinInputData) -> None:
        """
        硬貨が投入されたときに呼び出されるメソッド。

        Parameters
        ----------
        input_data : InsertCoinInputData
            投入された硬貨の情報（例: coin=100）
        """
        raise NotImplementedError


class SelectItemInputBoundary(ABC):
    """
    商品購入ユースケースを起動するための入り口。
    Controller はユーザーの「A1を買いたい」という要求を受け取り、
    それを SelectItemInputData にしてこの handle() に渡す。
    """

    @abstractmethod
    def handle(self, input_data: SelectItemInputData) -> None:
        """
        商品が選択されたときに呼び出されるメソッド。

        Parameters
        ----------
        input_data : SelectItemInputData
            どのスロットの商品を購入したいか（例: slot_id="A1"）
        """
        raise NotImplementedError


# -----------------------------------------------------------------------------
# 🟩 2. Output Boundary（出力境界）
#    UseCase → Presenter の「結果通知」の契約
# -----------------------------------------------------------------------------

class InsertCoinOutputBoundary(ABC):
    """
    硬貨投入後の結果を Presenter に渡すための出口。
    UseCase は「現在の投入金額は〇円です」といった結果を
    DTO(InsertCoinOutputData)にまとめ、この present_coin_inserted() を呼ぶ。
    Presenter 側はこれを実装する。
    """

    @abstractmethod
    def present_coin_inserted(self, output_data: InsertCoinOutputData) -> None:
        """
        硬貨投入処理の結果を通知する。

        Parameters
        ----------
        output_data : InsertCoinOutputData
            現在の投入金額など、表示の元になるデータ。
            ※ ここではまだ「文言整形」はしない。
              人間向けのメッセージ作成は Presenter の責務。
        """
        raise NotImplementedError


class SelectItemOutputBoundary(ABC):
    """
    商品購入処理の結果を Presenter に渡すための出口。
    UseCase は「どの商品を出したか」「お釣りはいくらか」といった
    ビジネス結果をDTO(SelectItemOutputData)として渡すだけで、
    表示方法（CLIでprintする？GUIでダイアログを出す？）までは関知しない。
    """

    @abstractmethod
    def present_purchase_result(self, output_data: SelectItemOutputData) -> None:
        """
        商品購入の結果を通知する。

        Parameters
        ----------
        output_data : SelectItemOutputData
            - item_name: 出した商品の名前
            - change:    返却されたお釣りの金額
        """
        raise NotImplementedError


# -----------------------------------------------------------------------------
# 🟥 3. Repository境界（在庫や商品情報へのアクセス契約）
#    UseCase → データアクセス
#
#    いわゆる「リポジトリ（Repository）」。データの取得・保存の
#    インターフェースのみ定義し、実際の保存方法は外側が持つ。
#
#    第1巡・第2巡では domain/repository.py に分けていましたが、
#    第3巡ではシンプルさを優先し、usecase/boundaries.py に集約します。
#
# -----------------------------------------------------------------------------

class ItemRepository(ABC):
    """
    自販機内の商品スロット（在庫・価格など）にアクセスするための抽象インターフェース。

    UseCase はこのインターフェースだけに依存し、
    在庫データがどこに保存されているか（メモリ？ファイル？DB？）は知らない。
    """

    @abstractmethod
    def find_by_slot_id(self, slot_id: str) -> Optional[Item]:
        """
        指定されたスロット(例: "A1")の商品情報を返す。

        戻り値:
            Itemインスタンス、あるいは該当スロットが未設定なら None
        """
        raise NotImplementedError

    @abstractmethod
    def save(self, item: Item) -> None:
        """
        商品の状態を保存する。
        たとえば在庫が1本減った後の状態など。

        ※ ID採番や実際の格納先は具体実装側の責任。
        """
        raise NotImplementedError


# -----------------------------------------------------------------------------
# 🟨 4. Hardware境界（ハードウェア制御の契約）
#    UseCase → 物理デバイス
#
#    自販機の「モーターを回して商品を落とす」「お釣りを返す」
#    といった物理的な動作は、ここで定義する抽象インターフェースを
#    経由して呼び出される。
#
#    UseCase は「何をしてほしいか」だけを伝え、
#    「どうやって実現するか」（実機/シミュレータ/ログ出力など）は知らない。
# -----------------------------------------------------------------------------

class HardwareInterface(ABC):
    """
    自販機の物理的な動作（商品排出・お釣り返却など）を扱う抽象インターフェース。

    UseCase はこのインターフェース経由でハードウェアに指示を出す。
    実際にどう動くか（実機のモーター制御？printでの模擬？）は
    interface_adapters 側で実装される。
    """

    @abstractmethod
    def dispense_item(self, slot_id: str) -> None:
        """
        指定スロットの商品を物理的に排出するよう要求する。
        実装例:
            - 本物の自販機ならモーターを回す
            - 開発環境なら print("A1 を排出しました") とログに出す
        """
        raise NotImplementedError

    @abstractmethod
    def return_change(self, amount: int) -> None:
        """
        指定金額のお釣りを返却するよう要求する。
        実装例:
            - 実機ならコインメカに指示する
            - 開発環境なら print("お釣り 40円") と表示する
        """
        raise NotImplementedError
```

---

## 🔄 依存関係のイメージ

「誰が誰に依存しているか」を、よく使う3つのやりとりごとに確認します。

### 1️⃣ Controller → UseCase

* Controller はユーザーの操作を受け取ります（例：「i 100」「s A1」などのコマンド）。
* その内容を DTO（`InsertCoinInputData`, `SelectItemInputData`）に詰めます。
* それを `InsertCoinInputBoundary.handle()` や `SelectItemInputBoundary.handle()` 経由でUseCaseに渡します。

→ Controller は「どのUseCase実装か」は知りません。
→ Controller は「このInputBoundaryインターフェースを満たす何かを呼んでいる」だけです。

---

### 2️⃣ UseCase → Repository / Hardware

* UseCaseは、在庫情報を取得・更新する必要があります。
  そのときは `ItemRepository` に依頼します。
* UseCaseは、商品を実際に出す必要があります。
  そのときは `HardwareInterface` に依頼します。

→ UseCaseは、データの保存方法やハードウェアの実装詳細は一切知らないまま動きます。
→ これにより、メモリ実装やダミーハードウェアをテスト用に差し込めます。

---

### 3️⃣ UseCase → Presenter

* UseCaseは処理の結果を `SelectItemOutputData` や `InsertCoinOutputData` といったDTOにまとめます。
* それを `SelectItemOutputBoundary.present_purchase_result()` などに渡します。

→ UseCaseは「どのように画面に整形されるか」「どんな文言になるか」を知りません。
→ CLIだろうがGUIだろうがWebだろうが、Presenter側の実装次第で自由に差し替え可能です。

---

## 🔬 テスト容易性の観点

🧪 ユースケースの単体テストでは、以下のようにフェイクを用意できます。

* フェイクの `ItemRepository`
* フェイクの `HardwareInterface`
* フェイクの `SelectItemOutputBoundary`

これらはすべて、`boundaries.py` のインターフェースを満たすシンプルなクラスとして書けます。
実機もデータベースも不要です。
UseCaseのロジックだけを確実に検証できます。

これは開発の初期段階で非常に重要です。
「まだ本番用ハードウェアがない」「まだ本番DBが決まっていない」という状態でも、ユースケースロジックを先に完成させ、動作保証をとることができます。

---

## ⚖️ なぜ `boundaries.py` に集約してよいのか

以前（第1巡・第2巡）は、下記のようにファイルを細かく分けていました。

* `repository.py`
* `input_boundary.py`
* `output_boundary.py`

これは大規模化や長期運用を考えると、とても良い分割です。

一方、第三巡は次の意図で構成を簡略化しています。

* 学習の最終段階では概念を重視し、フォルダ階層の複雑さを下げたい
* 「ユースケースは外部と抽象契約で会話する」という一点を強調したい
* ハードウェア制御（`HardwareInterface`）も、データ取得（`ItemRepository`）も、表示（Presenter）も、すべて「UseCaseの外部」として同じように扱えることを示したい

つまり `boundaries.py` は「UseCaseが世界とやり取りするためのすべての契約」を集約した宣言ファイルです。

---

## 🛡 まとめ（この章の鉄則）

> 内側はルールを決める。外側はそのルールに従って実装する。

* InputBoundary
  → Controller はこれを呼ぶだけで、ビジネスロジックに触れられる

* OutputBoundary
  → UseCase は結果をPresenterに渡すだけで、表示形式を知らなくてよい

* Repository / Hardware
  → UseCase はデータ保存や機器制御を指示できるが、具体的な実装は知らない

これにより、UIやハードウェア、永続化戦略を何度でも差し替えても、
UseCaseのコードは不変のまま保ち続けることができます。これはクリーンアーキテクチャの中心的な価値です。
