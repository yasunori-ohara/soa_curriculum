# 02 Use Case Interactor

# 👨‍🏫 Use Case Interactor
### `vending_machine/usecase/select_item_usecase.py`

`Entity` が「正しい状態とルール」を守る部品だとすると、
`UseCase` はそれらの部品を組み合わせて「実際の操作の流れ」を実現する役目です。

ここでは、自動販売機の中核となるシナリオである
**「ユーザーが商品ボタンを押して商品を購入する」** ユースケースを扱います。

このユースケースを担当するのが `SelectItemUseCase` です。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## 🎯 このクラスの役割

`SelectItemUseCase` は、自動販売機の「頭脳＝司令塔」です。
ひとつの「購入」というストーリー全体を正しい順番で進める責任があります。

やっていることを人間の言葉でいうとこうです：

1. 押されたボタンのスロット番号から商品を特定する
2. 投入された金額でそれが買えるか確認する
3. 問題なければ

   * 商品を排出するようにハードウェアに指示
   * お釣りがあれば返すように指示
   * 在庫を1本減らして保存する
4. 最終的な結果（出した商品名とお釣り）を画面側に伝える

重要なのは、UseCaseが自分で細かいことはしないことです。
具体的には：

* 「在庫はある？」の正しい判断は `Item` エンティティが持つルールに任せる
* 「お金は足りる？お釣りは？」の正しい計算は `PaymentManager` エンティティに任せる
* 「モーターをどう回す？」のようなハードウェアの詳細はアダプタ層に任せる
* 「ユーザーにどんな文言で表示する？」はPresenterに任せる

UseCaseはあくまで「次に何をさせるべきか」を順序だてて指示するだけ、という立場です。

---

## ✅ このクラスにあるべき処理 / あってはいけない処理

### ⭕ 含めるべき処理

* **複数のEntityをまたいだ流れの制御（オーケストレーション）**
  例：
  `item_repository` から商品を取得する →
  `payment_manager` に決済させる →
  `hardware` に排出させる →
  `item` の在庫を減らす →
  `item_repository` に保存する

* **アプリケーション固有の振る舞い**
  「お金が足りなかったら商品を出してはいけない」
  「在庫が減ったら保存し直す」など。

* **ユースケースごとの入出力の整形**
  `SelectItemInputData`（入力DTO）を受け取り、
  `SelectItemOutputData`（出力DTO）にまとめてPresenterに渡す。

### ❌ 含めてはいけない処理

* **Entity本来のルールを重複実装すること**
  例：`if item.stock > 0:` のような在庫チェックは `Item.dispense()` に任せるべきです。
  UseCaseが勝手にやるとルールが分散して危険になります。

* **ハードウェアの詳細やデータ保存のやり方**
  どのポートを叩くとモーターが回るか、DBが何を使ってるか、などはここでは知らない・書かない。

* **UIの見た目や文字列フォーマット**
  「画面にどう表示するか」は Presenter と View の仕事です。


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
│   ├─ boundaries.py        # InputBoundary / OutputBoundary
│   ├─ insert_coin_usecase.py    <- いまここ
│   └─ select_item_usecase.py    <- いまここ
```

## 💻 ソースコード（`SelectItemUseCase`）

このコードは、次のファイル・クラスを前提にしています：

* `usecase/dto.py`

  * `SelectItemInputData`
  * `SelectItemOutputData`
* `usecase/boundaries.py`

  * `SelectItemInputBoundary`
  * `SelectItemOutputBoundary`
  * `ItemDataAccessInterface`
  * `HardwareInterface`
* `domain/entities.py`

  * `Item`
  * `PaymentManager`

では、完成形を見ていきます👇

```python
# vending_machine/usecase/select_item_usecase.py

# -----------------------------------------------------------------------------
# 必要な型（契約）やデータの定義をインポート
# -----------------------------------------------------------------------------
from vending_machine.usecase.boundaries import (
    SelectItemInputBoundary,
    SelectItemOutputBoundary,
    ItemDataAccessInterface,
    HardwareInterface,
)
from vending_machine.usecase.dto import (
    SelectItemInputData,
    SelectItemOutputData,
)
from vending_machine.domain.entities import Item, PaymentManager


# -----------------------------------------------------------------------------
# SelectItemUseCase
# - クラス図の位置: UseCase
# - 同心円図の位置: Use Cases（内側から2番目の円）
#
# このクラスは「あるスロットの商品を買いたい」という1つの要求を処理する。
# Controller から呼び出される想定。
# -----------------------------------------------------------------------------
class SelectItemUseCase(SelectItemInputBoundary):
    """
    「商品を選択して購入する」というユースケースの実装。

    このクラスは「制御の流れ」を持つが、
    実際の処理の詳細は他のオブジェクトに委ねる。
        - お金の正しさ: PaymentManager
        - 在庫更新ルール: Item
        - ハードウェア制御: HardwareInterface実装
        - データ永続化: ItemDataAccessInterface実装
        - 出力の整形: Presenter
    """

    def __init__(
        self,
        presenter: SelectItemOutputBoundary,
        item_repository: ItemDataAccessInterface,
        hardware: HardwareInterface,
    ):
        """
        [依存性の注入 / Dependency Injection]

        UseCaseは、自分が直接newしません。
        代わりに「こういう能力を持った相手が欲しい」とだけ宣言し、
        実際に誰が来るかは外側 (main.py など) で決めてもらいます。

        - presenter:
            結果をユーザー表示用の形式に変換する役 (Presenter)
        - item_repository:
            商品情報・在庫を取得/保存してくれる役
        - hardware:
            「商品を物理的に排出して」「お釣り出して」と指示できる相手
        """
        self._presenter = presenter
        self._item_repository = item_repository
        self._hardware = hardware

    def handle(self, input_data: SelectItemInputData, payment_manager: PaymentManager):
        """
        [ユースケース本体 / アプリケーションフロー管理]

        Controller から呼び出される入り口。
        ここで「購入」という一連の流れをオーケストレーションする。

        引数:
            input_data: Controllerが組み立てた入力DTO
                        例: SelectItemInputData(slot_id="A1")
            payment_manager: 現在の投入金額などを保持しているエンティティ。
                             同じ購入トランザクションの間だけ使われる。

        戻り値:
            なし（ただし最後にPresenterへ結果を渡す）
        """

        # ---------------------------------------------------------
        # 手順1. スロットIDから商品を取得する
        # ---------------------------------------------------------
        item = self._item_repository.find_by_slot_id(input_data.slot_id)
        if not item:
            # 商品自体が存在しない/売り切れケースは、正式には専用のエラー型にしてもよい。
            raise ValueError("指定されたスロットに商品はありません。")

        # ---------------------------------------------------------
        # 手順2. 決済処理（お金まわりのルールはPaymentManagerに任せる）
        #   - 十分なお金が入っていなければ例外が出る
        #   - お釣りの計算もここで行われる
        #   - 成功したら投入金額は0にリセットされる
        # ---------------------------------------------------------
        change = payment_manager.process_purchase(item)

        # ---------------------------------------------------------
        # 手順3. ハードウェアへの指示（UseCaseは「何をしてほしいか」だけ言う）
        #   a. 商品を物理的に排出してほしい
        #   b. お釣りがあるなら返してほしい
        #   具体的にモーターをどう回すか等はHardwareInterface実装側の責務
        # ---------------------------------------------------------
        self._hardware.dispense_item(item.slot_id)

        if change > 0:
            self._hardware.return_change(change)

        # ---------------------------------------------------------
        # 手順4. 在庫を1本減らす（在庫の妥当性ルールはItem自身が持っている）
        #   - Item.dispense() 内で「在庫が無いなら売っちゃダメ」が保証される
        # ---------------------------------------------------------
        item.dispense()

        # ---------------------------------------------------------
        # 手順5. 変更後の商品の状態を保存する
        #   - どう保存するか(DB? メモリ? ファイル?) は
        #     item_repositoryの実装側が決める
        # ---------------------------------------------------------
        self._item_repository.save(item)

        # ---------------------------------------------------------
        # 手順6. 結果をPresenterに渡す
        #   - UseCaseは「どの商品名を出して、いくらお釣りが出たか」だけ伝える
        #   - 文言や見た目の整形はPresenter側の責務
        # ---------------------------------------------------------
        output_data = SelectItemOutputData(
            item_name=item.name,
            change=change,
        )
        self._presenter.present(output_data)
```

---

## 🔍 コードの読みどころ

* `class SelectItemUseCase(SelectItemInputBoundary):`
  → これは「このクラスは `SelectItemInputBoundary`（ユースケースの入り口インターフェース）を満たす正規の実装ですよ」という宣言です。
  つまり、「ユースケースの実体」であることを明示しています。

* `__init__()`
  UseCaseは `presenter`, `item_repository`, `hardware` に依存していますが、
  どれも「インターフェース越し」に依存しています。
  なので、このクラスは特定のDBや特定のハードウェアにロックインされません。

* `handle()`
  1メソッドの中に、アプリケーションとしての「買う」という物語のすべてがまとまっています。
  ただしビジネスルール自体はEntityに委譲しています。
  UseCaseは“順番と連携”に集中します。

---

## 💡 ユニットテストの考え方

UseCaseは、すべての外部依存（DB、ハードウェア、表示）をインターフェース経由で扱います。

つまりテストでは、それらを「偽物（フェイク／モック）」に置き換えるだけで、
物理ハードも本物DBもなしでロジック全体を検証できます。

```python
# tests/usecase/test_select_item_usecase.py のイメージ例

def test_お金が足りていれば商品とお釣りが正しく処理される():
    # 1. Arrange: 依存先の「偽物」を準備する
    fake_presenter = FakePresenter()
    fake_item_repo = FakeItemRepository({
        "A1": Item(slot_id="A1", name="お茶", price=120, stock=10)
    })
    fake_hardware = FakeHardwareAdapter()

    # 投入金額が十分にある状態にする
    payment_manager = PaymentManager(current_amount=500)

    # ユースケース本体を作る
    use_case = SelectItemUseCase(
        presenter=fake_presenter,
        item_repository=fake_item_repo,
        hardware=fake_hardware,
    )

    # 2. Act: 購入フローを実行
    input_data = SelectItemInputData(slot_id="A1")
    use_case.handle(input_data, payment_manager)

    # 3. Assert: 正しく指揮できているか？
    assert fake_hardware.dispensed_slot_id == "A1"
    assert fake_hardware.returned_change == 380  # 500 - 120
    assert fake_item_repo.find_by_slot_id("A1").stock == 9
    assert payment_manager.current_amount == 0
    assert fake_presenter.last_output.item_name == "お茶"
```

ポイントは「UseCaseそのものは、printもしないし、DBにも触れないし、GPIOも叩かない」ということ。
UseCaseをテストできるなら、あなたのビジネスロジックはほぼ安全に育てられる、ということでもあります。

---

## 🐍 Python vs 🔧 C言語でもう一度

* 🐍 Python側では、`SelectItemUseCase` が `HardwareInterface` に命令します。
  この時点では「商品を排出して」という抽象的なお願いしかしません。

* 🔧 Cっぽい書き方だと、次のような危険が起きがちです：

  * `PORTB |= (1 << PB3);` のような、特定ピンを直接叩く命令がビジネスロジックのど真ん中に生えてくる。
  * ハードウェアが変わった瞬間にビジネスロジックのコード全体を書き換えるハメになる。

クリーンアーキテクチャでは逆にします。

> ビジネスロジック（買うという行為）は不変であるべき。
> ハードウェア（どう排出するか）は交換可能な詳細であるべき。

UseCaseはその分離を徹底する場所です。

---

## 🛡️ このクラスの鉄則

> ビジネスの流れを指揮せよ。
> ただし細部のやり方には口を出すな。
> (Orchestrate the business flow, but delegate the details.)

* UseCaseは「どの順番で・何をするか」を決める。
* でも、実際のやり方（在庫の正当性チェックや、ハードをどう動かすか、表示をどうするか）は、それぞれの担当（EntityやAdapter）に任せる。
* この分離のおかげで、同じユースケースをGUIでもCLIでも物理筐体でも使い回すことができます。

---

この `SelectItemUseCase` を軸に、
次は `usecase/dto.py` と `usecase/boundaries.py` を正式に定義していきます。
DTOは「どんな形のデータを受け渡すか」、boundariesは「誰がどんなメソッドを持っていれば協力できるのか」という契約書です。
