# 11 Main（Composition Root）

# 🧠 すべてをつなぐ「依存注入」の実践

これまでに構築した各レイヤーは、
**すべてが“抽象（インターフェース）”を介して疎結合**に保たれています。
そのため、実際に動かすには「どの実装を使うのか」を明示的に“接続”する必要があります。

その接続こそが `main.py` の役割です。
クリーンアーキテクチャでは、このファイルを **Composition Root（合成の根）** と呼びます。

---

## 🎯 この章の目的

* 各コンポーネントを手動で「依存注入（DI）」してつなぐ
* クリーンアーキテクチャが“動く構造”であることを確認する
* 層の依存方向（内→外）は崩れていないことを確認する

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## ✅ main.py の役割

`main.py` は、**アプリケーションの起動地点**です。
責務はただ1つ：

> 「どの実装を使うのか」を決め、全てのクラスをつないで動かすこと。

ここにはビジネスロジックもUI制御も書きません。
ひたすら「配線（wiring）」だけに集中します。

---

## 💻 コード全体（基本版）

```python
# vending_machine/main.py
# -----------------------------------------------------------------------------
# 🎬 Composition Root
# -----------------------------------------------------------------------------
# ここで全ての実装をnewして接続し、アプリケーションを起動する。
# クリーンアーキテクチャでは「外側」が「内側」に依存するため、
# 依存の向きは main.py から一方向に注入される。
# -----------------------------------------------------------------------------

from vending_machine.domain.entities import PaymentManager
from vending_machine.interface_adapters.controller import VendingMachineController
from vending_machine.interface_adapters.presenter import VendingMachinePresenter
from vending_machine.interface_adapters.view_console import ConsoleView
from vending_machine.interface_adapters.data_access import InMemoryItemDataAccess
from vending_machine.interface_adapters.hardware_adapter import ConsoleHardwareAdapter
from vending_machine.usecase.insert_coin_usecase import InsertCoinUseCase
from vending_machine.usecase.select_item_usecase import SelectItemUseCase

# -----------------------------------------------------------------------------
# 🧩 各コンポーネントの生成
# -----------------------------------------------------------------------------
def main():
    # --- 永続化層（データアクセス） ---
    item_repository = InMemoryItemDataAccess()

    # --- ドメイン層の状態オブジェクト ---
    payment_manager = PaymentManager()

    # --- 外部インターフェース（ハードウェア） ---
    hardware = ConsoleHardwareAdapter()

    # --- プレゼンター & ViewModel ---
    presenter = VendingMachinePresenter()

    # --- ユースケース層 ---
    insert_coin_use_case = InsertCoinUseCase(presenter, payment_manager)
    select_item_use_case = SelectItemUseCase(
        presenter,
        item_repository,
        hardware,
        payment_manager
    )

    # --- コントローラ層 ---
    controller = VendingMachineController(
        insert_coin_use_case=insert_coin_use_case,
        select_item_use_case=select_item_use_case
    )

    # --- View層（UI） ---
    view = ConsoleView(controller=controller, presenter=presenter)

    # --- 実行開始 ---
    view.run()


# -----------------------------------------------------------------------------
# 🚀 エントリポイント
# -----------------------------------------------------------------------------
if __name__ == "__main__":
    main()
```

---

## 🧩 main.py の構造解説

### ① ドメインとユースケースは「受け取る側」

`UseCase`は自分で何も`new`しません。
必要なリポジトリやハードウェアは**外から注入**されます。

これが **依存性逆転の原則（DIP）** の実践です。

```python
select_item_use_case = SelectItemUseCase(
    presenter, item_repository, hardware, payment_manager
)
```

→ UseCase は「抽象」を受け取り、具体的な実装を知らない。
（DBを変えても、UseCaseの中身は変わらない）

---

### ② Controller と Presenter の接続

Controller は UseCase に依存し、
Presenter は ViewModel に出力を整えます。
両者を繋ぐのも main.py の仕事です。

```python
controller = VendingMachineController(
    insert_coin_use_case, select_item_use_case
)
presenter = VendingMachinePresenter()
```

Presenter を単一インスタンスにしておくことで、
複数のUseCaseが共通の ViewModel を更新できる構造になります。

---

### ③ View はアプリの“最外層”

View は入力をControllerに渡し、
Presenterが用意したViewModelを表示します。
UIループもここに存在します。

```python
view = ConsoleView(controller=controller, presenter=presenter)
view.run()
```

---

## 🧭 実行の流れ（テキスト図）

```
[ユーザー]
   ↓ 入力（コイン投入や商品選択）
[View]
   ↓ 呼び出し
[Controller]
   ↓ 呼び出し
[UseCase]
   ↓ 処理依頼
[Repository / Hardware]
   ↓
   結果を Presenter に渡す
[Presenter]
   ↓ 整形
[ViewModel → View]
   ↓ 出力
[ユーザーに結果表示]
```

> Viewは「操作」と「表示」だけ、
> UseCaseは「ビジネスロジック」だけ、
> DataAccessやHardwareは「作業」だけを行う。
>
> そしてそれらを束ねるのが main.py です。

---

## ✅ main.py の責務まとめ

| 区分   | 責務               |
| ---- | ---------------- |
| new  | 各クラスを生成する        |
| wire | 依存関係を注入して接続する    |
| run  | 最初のUI（View）を起動する |

main.py はこれだけ。
**ビジネスロジックを1行も書かない**のが最大の特徴です。

---

## 🧩 実行例（対話）

```bash
$ python vending_machine/main.py
自動販売機へようこそ。
コインを投入してください。
> i 100
現在の投入金額: 100円
> s A1
エラー: お金が足りません。
> i 100
現在の投入金額: 200円
> s A1
「お茶」が出てきました。お釣りは40円です。
> q
ありがとうございました！
```

---

## 💡 発展編（Optional）

リファクタリング章で紹介した改善案を適用する場合は、
main.pyの配線も少しだけ変わります。

### 変更例：PaymentManagerをリポジトリ化した場合

```python
from vending_machine.interface_adapters.data_access import (
    InMemoryItemDataAccess,
    InMemoryPaymentManagerAccess,
)

# ...
item_repository = InMemoryItemDataAccess()
pm_repository = InMemoryPaymentManagerAccess()

select_item_use_case = SelectItemUseCase(
    presenter, item_repository, hardware, pm_repository
)
insert_coin_use_case = InsertCoinUseCase(
    presenter, pm_repository
)
```

→ このように、PaymentManagerが“状態オブジェクト”ではなく
“永続層の1データ”として扱われる構造に変わります。

---

## 🧠 まとめ

| 層                 | クラス例                                                | 役割            |
| ----------------- | --------------------------------------------------- | ------------- |
| Domain            | `Item`, `PaymentManager`                            | ビジネスルールの中心    |
| UseCase           | `SelectItemUseCase`                                 | アプリケーションの操作手順 |
| InterfaceAdapters | `Controller`, `Presenter`, `DataAccess`, `Hardware` | 外部との接続点       |
| Main              | `main.py`                                           | すべてをつなぐ構成ルート  |

---

## 🎓 学びのポイント

* クリーンアーキテクチャは「newする場所を外に出す」思想
* 依存方向は常に **内→外へ**（上位ルールは下位に依存しない）
* `main.py` が唯一の“全てを知る場所”
* 他のモジュールは全体を知らない（疎結合）

---

## 🚀 まとめの一文

> **main.pyは、設計思想が「動く」瞬間。**

ここまで作り上げた各層が、
「依存方向」「責務分離」「抽象による制御逆転」という
クリーンアーキテクチャの理論を、実際に**走るプログラムとして証明**します。

おめでとうございます。
あなたの自販機は、クリーンアーキテクチャで“動く”ようになりました。
