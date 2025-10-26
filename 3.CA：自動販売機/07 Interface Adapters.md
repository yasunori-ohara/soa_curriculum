# 07 Interface Adapters

### Data Access / Hardware Adapter の役割と設計

---

## 🧭 1. Interface Adapters 層とは何か

クリーンアーキテクチャでは、アプリケーションは「内側から外側」に向かって安定度が下がっていきます。
内側ほど長く生きるルール（ビジネスの本質）で、外側ほど交換可能な詳細です。

* `domain/`
  ビジネスルールそのもの（エンティティ・不変条件）

* `usecase/`
  ビジネスフロー（何を / いつ / どの順番で行うか）

* `interface_adapters/` ← いま説明している層
  内側と外側をつなぐ「変換レイヤー」。
  外の世界の事情（UI、DB、ハード）を、内側が使える形に変換する役割。

この層のキーワードは「変換と仲介」です。
※ ビジネスルールを決める場所ではありません。

---

## 🧩 2. Presenter / Controller / View はどこに置くのか

Presenter、Controller、View はいずれもユーザーインターフェースまわりの変換レイヤーであり、広い意味では Interface Adapters 層の一部です。

ただし、この教材ではこれらを「それぞれ1ファイルずつ、明確に分けて扱う」方針にします。理由は以下のとおりです。

* 学習時に、役割の境界線を意識してもらいたい

  * Controller … 入力を整理して UseCase に渡す
  * Presenter … UseCase の結果を表示用に整形する
  * View … ユーザーと直接やり取りする（入出力）

* 将来、UI技術ごと（CLI / Web / GUI）に簡単に差し替えたい

  * `view_console.py` だけを `view_web.py` に差し替える、など
  * Controller や Presenter は再利用できる

そのため、次のようにそれぞれはトップレベルのファイルとして用意します。

```text
vending_machine/
├─ interface_adapters/
│   ├─ controller.py          # Controller
│   ├─ presenter.py           # Presenter
│   ├─ view_console.py        # View（CLI版）
│   ├─ data_access.py         # Data Access（在庫等のリポジトリ実装）
│   └─ hardware_adapter.py    # Hardware Adapter（ハード制御実装）
```

→ Presenter / Controller / View は UI の組み立て担当として個別ファイル化しておき、
　Data Access / Hardware Adapter は「外部世界に橋をかける技術コンポーネント」として同じ階層に配置します。

どちらも Interface Adapters 層ではありますが、学習上は役割が異なるため、ファイルを分けて説明します。

---

## 🔌 3. Data Access / Hardware Adapter はなぜ Interface Adapters 層に入るのか

ここが今回いちばん大事なポイントです。

### 3-1. Data Access（データアクセス）

* UseCase は「商品の情報を取ってきて」「在庫を更新して」といった要求をします。
* しかし UseCase は、どこにその情報が保存されているか（メモリか？DBか？外部APIか？）は知りたくありませんし、知ってはいけません。

そこで `usecase/boundaries.py` などで定義された抽象インターフェース、たとえば `ItemDataAccessInterface` のようなものを、**具体的に実装するクラス**が必要になります。

それが `interface_adapters/data_access.py` に置かれるクラスです。

* 役割:
  「抽象的な `get_item(slot_id)` に応じて、実際の保存先（辞書やDB）から値を返す」

* 性質:
  技術に依存する（メモリ、RDB、ファイルなど）ため交換可能にしたい
  → 内側（UseCase）ではなく外側（Interface Adapters）に置くのが自然

---

### 3-2. Hardware Adapter（ハードウェアアダプタ）

* UseCase は「スロット A1 の商品を出して」「お釣りを返して」といった“論理的な指示”を出します。
* しかし UseCase は、実際にどうやってモーターを回すか、どうやってコインメカを開くかといった“物理的な詳細”を知るべきではありません。

そこで `usecase/boundaries.py` で定義された `HardwareInterface` を、**実際に動く形で実装するクラス**が必要になります。

それが `interface_adapters/hardware_adapter.py` です。

* 役割:
  「抽象的な `dispense_item(slot_id)` を、具体的な動作（ここでは print、将来はGPIO制御など）に変換する」

* 性質:
  ハードウェアという非常に環境依存の詳細に触れる
  → 内側には置けない。Interface Adapters 側で閉じ込める

---

💡 まとめると、Data Access も Hardware Adapter も本質的には同じ役割を持っています。

> 「UseCase が定義した抽象インターフェースを、現実世界の技術にマッピングするアダプタ」

この“現実世界の技術”が、DBアクセスであったり、物理ハードの制御であったりするだけの違いです。

---

## 💾 4. Data Access の実装イメージ

以下は、在庫データをインメモリで管理する例です。
（`interface_adapters/data_access.py` という想定）

```python
# --------------------------------------------------------------------
# File: vending_machine/interface_adapters/data_access.py
# Layer: Interface Adapters
#
# 目的:
#   - UseCase が期待するリポジトリインターフェース
#     (例: ItemDataAccessInterface)
#     を満たす具体的な実装を提供する。
#
#   - ここでは "インメモリ在庫" という技術的な保存方法で実現する。
#     後でDBに差し替える場合も、このファイルを差し替えればよい。
# --------------------------------------------------------------------

from vending_machine.usecase.boundaries import ItemDataAccessInterface
from vending_machine.domain.entities import Item


class InMemoryItemDataAccess(ItemDataAccessInterface):
    """
    自動販売機内の商品データを管理するインメモリ実装。

    - スロットID("A1"など)をキーとして Item を保持する。
    - UseCase からは抽象インターフェース越しに呼ばれるので、
      UseCase は "在庫がメモリにあるのかDBにあるのか" を知らない。
    """

    def __init__(self):
        # 簡易的な在庫データ
        # 実サービスならここはDBや外部APIなどになる
        self._items: dict[str, Item] = {
            "A1": Item(slot_id="A1", name="お茶", price=160, stock=10),
            "B2": Item(slot_id="B2", name="コーヒー", price=150, stock=3),
            "C3": Item(slot_id="C3", name="スポーツドリンク", price=180, stock=0),  # 売り切れ例
        }

    def get_item(self, slot_id: str) -> Item | None:
        """
        指定スロットの商品情報を返す。
        見つからなければ None。
        """
        return self._items.get(slot_id)

    def save_item(self, item: Item) -> None:
        """
        商品在庫を更新して保存する。
        UseCase からは「在庫を1減らした状態のItem」を渡されるので、
        ここではそれを単純に格納し直すだけ。
        """
        self._items[item.slot_id] = item
```

🔎 重要な点：

* `InMemoryItemDataAccess` は `ItemDataAccessInterface`（＝UseCase側が定義する契約）を実装しています。
* UseCase はこの抽象 (`ItemDataAccessInterface`) だけに依存します。
  具体クラス名 `InMemoryItemDataAccess` を知らずに動けます。

これが「内側が外側の詳細を知らない」状態です。

---

## 🤖 5. Hardware Adapter の実装イメージ

次はハードを操作するアダプタの例です。
（`interface_adapters/hardware_adapter.py` という想定）

```python
# --------------------------------------------------------------------
# File: vending_machine/interface_adapters/hardware_adapter.py
# Layer: Interface Adapters
#
# 目的:
#   - UseCase が定義した HardwareInterface(抽象) を
#     実際に動作させる具体クラスを提供する。
#
#   - この教材では print() によるシミュレーションで代用。
#     実機ならここにGPIO制御やモーター制御などが入る。
# --------------------------------------------------------------------

from vending_machine.usecase.boundaries import HardwareInterface


class ConsoleHardwareAdapter(HardwareInterface):
    """
    ハードウェア操作のコンソール版シミュレーション。

    UseCase は "dispense_item('A1')" のように論理命令だけを行い、
    実際の動き（モーターを回す/お釣りを返す）はこの層が担当する。
    """

    def dispense_item(self, slot_id: str) -> None:
        """
        指定スロットの商品を排出する。
        ここでは print() で代用しているが、
        実際には装置制御コードに置き換わる想定。
        """
        print(f"（ガコン！スロット {slot_id} の商品を排出しました）")

    def return_change(self, amount: int) -> None:
        """
        指定金額ぶんのお釣りを返す。
        ここでは print() で代用。
        """
        print(f"（チャリン！お釣り {amount} 円を返却しました）")
```

🔎 重要な点：

* `ConsoleHardwareAdapter` は `HardwareInterface` を実装しています。
* `HardwareInterface` は `usecase/boundaries.py` で定義される抽象です。つまり、**契約は内側（UseCase側）が決める**ということです。
* UseCase は「どういう動きが必要か」を宣言するだけで、実際の機構のことは一切知りません。

このおかげで、例えば将来「実機のラズパイ版」と「シミュレーション用のダミー版」を簡単に切り替えられるようになります。

---

## 🔄 6. 依存方向と差し替えのしやすさ

ここまで出てきた登場人物の依存関係を、テキストで整理します。

* UseCase は以下の「抽象インターフェース」に依存します。

  * `ItemDataAccessInterface`
  * `HardwareInterface`
  * `PresenterBoundary`（出力用）
  * `ControllerBoundary`（入力用）  ← 呼び出し元の契約として定義する場合

* Interface Adapters 層は、それらの抽象を**実装**します。

  * `InMemoryItemDataAccess` が `ItemDataAccessInterface` を実装
  * `ConsoleHardwareAdapter` が `HardwareInterface` を実装
  * `VendingMachinePresenter` が PresenterBoundary を実装
  * `VendingMachineController` が ControllerBoundary を呼び出す

このように、「内側が契約を定義し」「外側がそれを実装する」という形になります。
これが依存性逆転の原則（DIP）です。

---

## ✅ まとめ

* Interface Adapters 層は「内側（ビジネスロジックの世界）」と「外側（技術・環境依存の世界）」をつなぐ変換役です。

* Presenter / Controller / View は UI のための変換担当なので、それぞれ別ファイルとして明確に扱います。
  UIを差し替える（CLI→Webなど）ときにここを交換します。

* Data Access / Hardware Adapter は、データベースや実機ハードウェアといった“現実世界の詳細”を扱う具体実装です。
  UseCase が定義したインターフェースを、そのまま満たす形で `interface_adapters/` に実装します。

* UseCase（アプリケーションの核）は、これらの具体的な実装を知る必要がありません。
  逆に言えば、ここを差し替えるだけで「保存先をDBにする」「本物のハードを動かす」といった拡張が可能です。

---

> **Interface Adapters とは「詳しい話はここで受けます」という層です。**
> ドメインとユースケースが守るべき純粋なロジックを、技術的な事情から隔離する“クッション”として機能します。
