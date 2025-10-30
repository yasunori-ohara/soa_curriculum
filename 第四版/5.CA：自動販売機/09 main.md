# 09 Main（Composition Root）

## 🧠 すべてをつなぐ「依存注入」の最後の一手

`vending_machine/main.py`

これまでの実装では、各レイヤーは疎結合になるように分かれていました。

* `domain/`
  ビジネスルールの中心（商品・お金・在庫・価格など）

* `usecase/`
  アプリケーションとしての「流れ」（コイン投入・商品購入など）と、周辺契約（`boundaries.py`, `dto.py`）

* `interface_adapters/`
  現実世界を抽象化して扱うための実装（在庫データアクセス、ハードウェア操作、Presenter、Controller、View など）

それぞれは「何をするか」は知っていますが、「誰と組み合わせるべきか」は知りません。
それらを最終的に組み合わせてアプリとして起動させる役割を担うのが、`main.py` です。

この「全員を正しくつなぐ場所」のことを、クリーンアーキテクチャでは **Composition Root（合成の根）** と呼びます。

---

## 🎯 main.py の目的

* どの具象クラスを使うかを決める
  例：在庫にはインメモリ実装を使うのか、ハードはコンソール用のダミーか、など

* 依存関係を注入する（Dependency Injection）
  UseCaseにPresenterを渡す、PresenterにViewModelを渡す、ControllerにUseCaseを渡す、など

* UIループ（コンソールなど）を開始する

main.py は「配線」だけを担当します。
ビジネスロジックそのものは書きません。

---

## 📁 今回のフォルダ構成（おさらい）

```text
vending_machine/
├─ domain/
│   ├─ entities.py          # Item / PaymentManager などのドメインルール
│   └─ errors.py            # ドメイン固有の例外
│
├─ usecase/
│   ├─ dto.py               # InputData / OutputData / ViewModel
│   ├─ boundaries.py        # UseCaseと外界をつなぐインターフェース群
│   ├─ insert_coin_usecase.py
│   └─ select_item_usecase.py
│
├─ interface_adapters/
│   ├─ data_access.py       # 在庫・支払い状態を扱うインメモリ実装（リポジトリ）
│   ├─ hardware_adapter.py  # ハードウェア（実機/コンソール）の実装
│   ├─ presenter.py         # Presenter（OutputBoundary実装）
│   ├─ controller.py        # Controller（View→UseCaseの橋渡し）
│   ├─ view_console.py      # コンソールUI
│   └─ ...
│
└─ main.py                  # ← ここで全てを束ねて起動する
```

---

## 💻 main.py（サンプル実装）

```python
# vending_machine/main.py

# -----------------------------------------------------------------------------
# Composition Root
# すべての具象実装を組み立て、依存関係を注入し、アプリケーションを起動する。
#
# ポイント:
# - 各層のクラスは、自分から他人を勝手に new しません。
#   ここ (main.py) だけが「誰と誰を組み合わせるか」を知っています。
# - これは依存性逆転の原則 (DIP) を現実に落とし込んだ形です。
# -----------------------------------------------------------------------------

# ▼ UseCaseが必要とするインターフェースの実装たち
from vending_machine.interface_adapters.data_access import (
    InMemoryItemDataAccess,
    InMemoryPaymentManagerAccess,
)
from vending_machine.interface_adapters.hardware_adapter import (
    ConsoleHardwareAdapter,
)

# ▼ Presenter / ViewModel
from vending_machine.usecase.dto import VendingMachineViewModel  # 表示用データ入れ物
from vending_machine.interface_adapters.presenter import (
    VendingMachinePresenter,
)

# ▼ UseCase本体
from vending_machine.usecase.insert_coin_usecase import (
    InsertCoinUseCase,
)
from vending_machine.usecase.select_item_usecase import (
    SelectItemUseCase,
)

# ▼ Controller / View
from vending_machine.interface_adapters.controller import (
    VendingMachineController,
)
from vending_machine.interface_adapters.view_console import (
    ConsoleView,
)


def main() -> None:
    # -----------------------------------------------------------------
    # 1. データアクセス層の具体実装を準備
    #
    # InMemoryItemDataAccess:
    #   - スロットID→商品情報の管理
    #   - 在庫の取得・更新
    #
    # InMemoryPaymentManagerAccess:
    #   - 現在の投入金額や釣り計算のための状態 (PaymentManager)
    #   - その取得・保存
    #
    # これらは usecase/boundaries.py で定義された
    # ItemDataAccessInterface / PaymentManagerAccessInterface
    # を満たす実装クラスです。
    # -----------------------------------------------------------------
    item_repository = InMemoryItemDataAccess()
    payment_repository = InMemoryPaymentManagerAccess()

    # -----------------------------------------------------------------
    # 2. ハードウェアアダプタ（物理制御の具体実装）
    #
    # ConsoleHardwareAdapter:
    #   - 実機の代わりに print() で「ガコン！」などを表示するアダプタ
    #   - usecase/boundaries.py で定義された HardwareInterface を実装
    #
    # 将来的に本物のモーター制御・コイン機制御に差し替える場合も、
    # UseCaseは変更不要で、ここで渡すクラスだけ差し替えればよい。
    # -----------------------------------------------------------------
    hardware_adapter = ConsoleHardwareAdapter()

    # -----------------------------------------------------------------
    # 3. ViewModel と Presenter
    #
    # VendingMachineViewModel:
    #   - ユーザーに見せるためのテキストを保持するだけの入れ物
    #   - 例: "A1 のコーラを出しました。お釣り 20 円です"
    #
    # VendingMachinePresenter:
    #   - UseCase の結果 (成功 or 失敗) を受け取り、
    #     ViewModel.display_text にわかりやすい文章として書き込む。
    #
    # View はこの ViewModel を読むだけ。
    # UseCase は ViewModel を直接知らない。
    # -----------------------------------------------------------------
    view_model = VendingMachineViewModel()
    presenter = VendingMachinePresenter(view_model=view_model)

    # -----------------------------------------------------------------
    # 4. UseCase（アプリケーションロジック）
    #
    # InsertCoinUseCase:
    #   - コイン投入処理
    #   - PaymentManagerAccessInterface に依存し、
    #     「今いくら入っているか」を更新する。
    #
    # SelectItemUseCase:
    #   - 商品購入フロー
    #   - 在庫確認、料金チェック、お釣り計算、在庫減算、ハード制御呼び出し
    #   - 成否に応じて presenter.present_success() または
    #     presenter.present_failure() を呼ぶ。
    #
    # どちらの UseCase も「データアクセス抽象」「ハードウェア抽象」
    # 「出力境界（Presenter用インターフェース）」だけに依存します。
    # -----------------------------------------------------------------
    insert_coin_usecase = InsertCoinUseCase(
        presenter=presenter,
        payment_repository=payment_repository,
    )

    select_item_usecase = SelectItemUseCase(
        presenter=presenter,
        item_repository=item_repository,
        payment_repository=payment_repository,
        hardware=hardware_adapter,
    )

    # -----------------------------------------------------------------
    # 5. Controller
    #
    # VendingMachineController:
    #   - View から受け取った「ユーザー操作の意図」（例: "100円投入" / "A1を購入したい"）
    #     を、UseCaseが理解できるDTOに変換して呼び出す。
    #
    # Controller 自身はビジネスルールを判断しません。
    # 単に橋渡しを行うだけです。
    # -----------------------------------------------------------------
    controller = VendingMachineController(
        insert_coin_use_case=insert_coin_usecase,
        select_item_use_case=select_item_usecase,
    )

    # -----------------------------------------------------------------
    # 6. View（UI）
    #
    # ConsoleView:
    #   - ユーザーと直接やり取りする最外層
    #   - input() でユーザー操作を受け取り、
    #     controller.*() を呼び、
    #     presenter が書き込んだ view_model.display_text を print() する。
    #
    # View はビジネスルールやお金の残高ロジックを知りません。
    # Presenter が整形したメッセージを表示するだけです。
    # -----------------------------------------------------------------
    view = ConsoleView(
        controller=controller,
        view_model=view_model,
    )

    # -----------------------------------------------------------------
    # 7. 起動
    #
    # View に制御を渡します。
    # ConsoleView はループを持ち、ユーザーからの入力を繰り返し受け付けます。
    # -----------------------------------------------------------------
    view.run()


if __name__ == "__main__":
    main()
```

---

## 🔍 main.py の読みどころ

### 1. `main.py` は唯一「全体像を知っていい場所」

* `usecase/` 配下のコードは、「どの具体実装が来るのか」を知りません。

  * たとえば `SelectItemUseCase` は「HardwareInterface を持ってきてください」と要求するだけで、実際に渡されるのが `ConsoleHardwareAdapter` なのか、本物の `GpioHardwareAdapter` なのかは知りません。
* それを知っていて、実際に渡すのは `main.py` の仕事です。

こういう「全体の配線を知る唯一の場所」が Composition Root です。

---

### 2. 依存方向の最終チェック

依存の矢印はすべて内向き（中心向き）です。

* View → Controller
* Controller → UseCase
* UseCase → Boundaries（抽象インターフェースたち）
* Presenter, DataAccess, HardwareAdapter → それぞれの Boundary を実装する具象

`main.py` はその“外向き”のクラス群（Presenter / DataAccess / HardwareAdapter / View / Controller）を作ってから、UseCaseに注入しています。
つまり、**内側の層は外側の実装を import していない**ことを最後にもう一度保証しています。

---

### 3. 「差し替えやすさ」の実感ポイント

もし本番ハードウェアをつなげたい場合、`ConsoleHardwareAdapter` を `RealHardwareAdapter` に差し替えるだけで済みます。それは main.py だけで完結します。UseCaseには手を入れません。

```python
# hardware_adapter = ConsoleHardwareAdapter()
hardware_adapter = RealHardwareAdapter(gpio_driver=..., coin_mech=...)
```

同じことはデータ保存にも言えます。
`InMemoryItemDataAccess` を `PostgresItemDataAccess` に差し替えるだけです。

これは「ハードウェアもデータストアもUIもすべて詳細（detail）である」という考え方が、ちゃんと現実に落ちている証拠です。

---

## 🧪 実行イメージ（コンソール動作例）

```text
=== 自動販売機 ===
(i) お金を入れる
(s) 商品を購入する
(q) 終了
> i
投入金額を入力してください: 100
現在の投入金額: 100円です。

> s
購入したいスロットIDを入力してください (例: A1): A1
（ガコンッ！スロット A1 の商品を排出しました）
（チャリン！お釣り 40 円を返却しました）
「お茶」が出てきました。 お釣りは40円です。

> q
ありがとうございました。
```

ここで表示されるメッセージはすべて `Presenter` が `ViewModel.display_text` に書き込んだ結果であり、View はそれを `print()` しているだけです。

---

## ✅ まとめ

* `main.py` は「誰が誰を使うか」を決める唯一の場所です。
* ビジネスロジック（UseCase）は、具体的な入出力手段（コンソールUIや実機ハード）を知りません。
* UIやハードウェアは入れ替えても、UseCaseは同じまま動かせます。
* これによって、アプリケーションはテストしやすくなり、拡張しやすくなります。

> **main.py は、すべての抽象を具体に結びつける最終工程であり、クリーンアーキテクチャが「動く」ことの証明です。**
