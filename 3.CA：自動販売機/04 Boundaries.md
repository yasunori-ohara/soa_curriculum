# 04 Boundaries

# 🔌 Boundaries
### `vending_machine/usecase/boundaries.py`

`UseCase`は司令塔ですが、ひとりでは何もできません。
「商品データ取ってきて」「商品を落として」「お釣り出して」「結果を表示して」…
これらは全部、別の役割の人たちにやってもらう必要があります。

その「誰がどんなメソッドを持っていれば、UseCaseは正しく仕事を進められるのか？」
を明文化したのがこのファイルです。

これをまとめて **Boundary（境界）/ Interface（契約）** と呼びます。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## 🎯 このファイルの役割

* UseCaseが外の世界に要求する“契約”をすべてここに集めます。
* UseCaseは、**契約（インターフェース）には依存するけど、具体的な実装は知らない** という形にします。

結果として：

* UseCaseはデータベースを知らないまま、在庫の取得・保存を依頼できる
* UseCaseはハードウェアを知らないまま、「商品を出して」と依頼できる
* UseCaseはUIの形を知らないまま、「結果を見せて」と依頼できる

これがクリーンアーキテクチャの「依存の逆転」（内側がインターフェースを持ち、外側がそれを実装する）にあたります。

---

## ✅ このファイルに入れていいもの / 入れてはいけないもの

⭕ 入れていいもの

* 抽象クラス (`ABC`)
* 抽象メソッド (`@abstractmethod`)
* 型の宣言

❌ 入れてはいけないもの

* 具体的な処理内容
* print / DB呼び出し / ハードウェア制御などの副作用
* ビジネスロジック

ここは「約束事の一覧表」に徹します。
実装はあとで `interface_adapters/` 側に作ります。

## 💻 フォルダ構成

```text
vending_machine/
├─ domain/
│   ├─ entities.py          # Item, PaymentManager など
│   ├─ errors.py            # ドメインルール違反例外
│   └─ repositories.py      # DataAccessInterface / HardwareInterface
│
├─ usecase/
│   ├─ dto.py               # InputData / OutputData / ErrorData / ViewModel
│   ├─ boundaries.py        # InputBoundary / OutputBoundary    <- いまここ
│   ├─ insert_coin_usecase.py
│   └
```

## 💻 コード（`boundaries.py`）

```python
# vending_machine/usecase/boundaries.py
from abc import ABC, abstractmethod
from typing import Optional

# DTOとEntityを参照する
from vending_machine.usecase.dto import (
    SelectItemInputData,
    SelectItemOutputData,
)
from vending_machine.domain.entities import Item, PaymentManager


# -----------------------------------------------------------------------------
# InputBoundary / OutputBoundary
# - Controller が UseCase を呼ぶときの入り口
# - UseCase が Presenter に結果を渡すときの出口
# -----------------------------------------------------------------------------
class SelectItemInputBoundary(ABC):
    """
    ユースケースを「実行させる側」(Controllerなど) が呼び出すための窓口。

    UseCase本体 (SelectItemUseCase) はこの抽象クラスを実装することで、
    「私は正しいユースケースの実体ですよ」と名乗ることになる。
    """
    @abstractmethod
    def handle(self, input_data: SelectItemInputData, payment_manager: PaymentManager):
        """
        購入処理を開始する。

        input_data:
            Controllerが組み立てたリクエスト情報（どのスロットか、など）

        payment_manager:
            現在の投入金額など、お金の状態を保持しているエンティティ。
        """
        raise NotImplementedError


class SelectItemOutputBoundary(ABC):
    """
    ユースケースの結果を受け取り、
    ユーザーに見せる形へ整える側 (Presenter) のインターフェース。
    """
    @abstractmethod
    def present(self, output_data: SelectItemOutputData):
        """
        output_data:
            ユースケース結果の「生データ」
            （何を買えたか／お釣りはいくらか 等）

        Presenterはこれをもとに画面表示用のViewModelを組み立てる。
        """
        raise NotImplementedError


# -----------------------------------------------------------------------------
# ItemDataAccessInterface
# - 在庫や商品情報を取得・保存するための「倉庫係」
# - DBかメモリかファイルかネットワークかは関係ない
# -----------------------------------------------------------------------------
class ItemDataAccessInterface(ABC):
    """
    商品スロットの情報にアクセスするためのインターフェース。
    UseCaseはこれに依存し、具体的な保存先のことは知らない。
    """

    @abstractmethod
    def find_by_slot_id(self, slot_id: str) -> Optional[Item]:
        """
        指定されたスロットIDのItemを返す。
        スロットが空なら None を返す。
        """
        raise NotImplementedError

    @abstractmethod
    def save(self, item: Item) -> Item:
        """
        更新されたItemを保存し、保存後の最終状態を返す。
        （在庫が1本減った、など）
        """
        raise NotImplementedError


# -----------------------------------------------------------------------------
# HardwareInterface
# - ハードウェア制御（商品を落とす／お釣りを返す）を抽象化した窓口
# - UseCaseは物理的な仕組みを一切知らずに指示できる
# -----------------------------------------------------------------------------
class HardwareInterface(ABC):
    """
    自販機の物理的な動作を担当する「手足」側の契約。
    実際にどう動くかはAdapter側で実装される。
    """

    @abstractmethod
    def dispense_item(self, slot_id: str):
        """
        指定されたスロットの商品を物理的に排出する。

        例:
            - 実機ならモーターを回す
            - 今回の学習用コンソール版なら print で代用
        """
        raise NotImplementedError

    @abstractmethod
    def return_change(self, amount: int):
        """
        指定された金額のお釣りを物理的に返却する。

        例:
            - 実機ならコインメカに信号を送る
            - 学習用なら print で「チャリン！」と表示する
        """
        raise NotImplementedError
```

---

## 🔍 このファイルがすごく大事な理由

ここに書かれているのは「UseCaseが他人に頼るときのお願いの仕方」です。
逆に言うと、ここに載っていない相手にはUseCaseは頼れません。

ここが明文化されていると：

* UseCaseは **具体的な技術にロックインされずに済む**

  * DBがSQLiteでもMongoDBでも、さらには完全インメモリでもOK
* ハードウェアが変わっても **UseCaseを一切書き換えなくていい**

  * Raspberry Piでも実機筐体でもシミュレーターでもOK
* UIがCLIでもWebでも画面付き端末でも **UseCaseを使い回せる**

これがアーキテクチャの肝です。「詳細は交換できる」が現実になります。

---

## 💡 このファイル自体の単体テストは必要？

基本的には不要です。
なぜなら、このファイル自体はロジックを持っていないからです。

テストすべきなのは、

* UseCaseがこれらのインターフェース通りに呼び出しているか
* Adapter側（実装側）がこの契約どおりに動くか
  のほうです。

---

## 🐍 PythonとC言語の比較（初心者の方へ）

* 🐍 Pythonでは `ABC`（抽象基底クラス）と `@abstractmethod` を使うことで、
  「このクラスを名乗るならこのメソッドは必須ですよ」という“契約”をコードで表現できます。

* 🔧 C言語にはクラスやインターフェースという仕組みはありません。
  近いものは「ヘッダーファイル(.h)で関数の宣言だけ提示し、実装は別ファイルに書く」スタイルです。
  つまり「こういう関数があることにしてください」という約束を先に置く、という発想は似ています。

---

## 🛡️ Boundaries層の鉄則

> 契約は内側が決める。
> 実装は外側が合わせる。

UseCaseは「私はこういう相手がほしい」と宣言します。
外側（interface_adapters側）は、その契約に合わせてクラスを実装します。

これが「依存性逆転」の実感そのものです。
これを押さえれば、あとは Adapter 側（例：`hardware_adapter.py`）を書くだけで、
自販機が現実に“ガコンッ”と動き出します。

---

次は `interface_adapters/hardware_adapter.py` に進みます。
ここで、今定義した `HardwareInterface` を実装して、
「UseCaseの指示を実際のアクションに変える手足」を作ります。
