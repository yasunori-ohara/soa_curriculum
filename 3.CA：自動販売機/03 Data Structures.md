# 03 Data Structures

# ✉️ Data Structures
### `vending_machine/usecase/dto.py`

`UseCase` は「現場監督（指揮者）」でした。
では、その指揮者と周りの人たちはどんな言葉・どんな形式で情報をやり取りするんでしょうか？

それを定義するのが **Data Structures（データ構造 / DTO）** です。
ここでは、`SelectItemUseCase` を動かすために必要な、入力と出力の“公式フォーマット”をまとめて扱います。

---

## 🎯 このファイルの役割

このファイルにあるクラスたちは、めちゃくちゃシンプルです。
やることはただ1つ：

> 層と層のあいだでデータを「安全に」「決まった形で」受け渡す。

それだけです。
計算もしないし、ビジネスルールも入れません。

* `Controller` → `UseCase` に渡す入力
* `UseCase` → `Presenter` に渡す結果
* `Presenter` → `View` に渡す表示用の状態

これらを「勝手に辞書とかでバラバラに渡す」のではなく、
明示的なクラス（型）で受け渡すことで、境界がくっきりします。

これは「依存方向をきれいに保つ」「壊れにくくする」ための超・重要パーツです。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## ✅ このファイルに“だけ”やらせたいこと

⭕️ やっていいこと

* フィールドを持つ
* 「このデータは何のためのものか」を説明する

❌ 絶対にやってはいけないこと

* ビジネスロジックを入れる
* 計算・検証ロジック（バリデーション）を入れる
* DB呼び出し、ハードウェア呼び出しなど副作用を持つ処理を入れる

> 🧠 合言葉：「ただ運べ。考えるな。」


## 💻 フォルダ構成

```text
vending_machine/
├─ domain/
│   ├─ entities.py          # Item, PaymentManager など
│   ├─ errors.py            # ドメインルール違反例外
│   └─ repositories.py      # DataAccessInterface / HardwareInterface
│
├─ usecase/
│   ├─ dto.py               # InputData / OutputData / ErrorData / ViewModel    <- いまここ
│   ├─ boundaries.py        # InputBoundary / OutputBoundary
│   ├─ insert_coin_usecase.py
│   └
```

## 💻 コード（`dto.py`）

```python
# vending_machine/usecase/dto.py
from dataclasses import dataclass

# -----------------------------------------------------------------------------
# Data Transfer Objects (DTO)
# - 役割:
#     - レイヤー間でやり取りする“公式の書類”
# - 位置づけ:
#     - クリーンアーキテクチャの円と円の境界線上を流れるデータ
# -----------------------------------------------------------------------------


# -----------------------------------------------------------------------------
# SelectItemInputData
# Controller -> UseCase に渡す入力データ
# -----------------------------------------------------------------------------
@dataclass(frozen=True)
class SelectItemInputData:
    """
    自動販売機の「この商品が欲しいです」というリクエスト情報。

    例:
        SelectItemInputData(slot_id="A1")

    slot_id:
        ユーザーが押したスロット番号。
        A1 や B3 のように「どの場所の商品か」を特定する。
    """
    slot_id: str


# -----------------------------------------------------------------------------
# SelectItemOutputData
# UseCase -> Presenter に渡す出力データ
# -----------------------------------------------------------------------------
@dataclass(frozen=True)
class SelectItemOutputData:
    """
    ユースケースの実行結果を表す「生の事実」。

    UseCaseは、最終的なメッセージや画面の形を気にしません。
    代わりに「何が起きたか」という事実だけをこの形にまとめ、
    Presenterに引き渡します。

    item_name:
        実際に排出された商品の名前（例: "お茶"）
    change:
        返却されたお釣りの金額（整数, 単位: 円）
    """
    item_name: str
    change: int


# -----------------------------------------------------------------------------
# VendingMachineViewModel
# Presenter -> View に渡す表示用データ
# -----------------------------------------------------------------------------
@dataclass
class VendingMachineViewModel:
    """
    画面（コンソールやUI）がそのまま使える「見せ方用の情報」。

    Presenterは SelectItemOutputData を受け取り、
    ユーザーに見せやすい1本の文字列などに整形して ViewModel を更新します。

    View は ViewModel の中身をそのまま表示するだけでOKになります。

    display_text:
        ユーザー向けの表示用テキスト。
        例: "お茶を購入しました。お釣りは30円です。"
    """
    display_text: str = ""
```

---

## 🔍 なぜ `@dataclass(frozen=True)` なの？

* `SelectItemInputData` と `SelectItemOutputData` は `frozen=True` になっています。
  これは「一度作ったら勝手に書き換えられない（イミュータブル）」という意味です。

* これで何が嬉しいかというと：

  * Controller から渡された入力が、UseCaseの途中で勝手に書き換わらない
  * UseCase が出した結果が、Presenterの途中でこっそり書き換わらない

境界を越えるデータは、誰かに後から“しれっと”改ざんされるとデバッグ不能になります。
なので「公式書類は書き換え禁止」という形にして安全に運びます。

逆に、`VendingMachineViewModel` は画面表示用の状態なので、Presenter側が書き換えていきます。
だからこちらは `frozen=True` にしていません（編集できるのが前提）。

---

## 💡 このファイル自体はテストすべき？

基本的に、このファイル専用のユニットテストは不要です。

理由はシンプルで、DTOはロジックを持たないからです。
「データを入れたらそのまま保持できるか？」以上の振る舞いがないので、
ここを個別にテストしても得るものは少ないです。

代わりに、

* UseCaseのテスト
* Presenterのテスト
  の中で、これらのDTOが正しく渡されていることを確認するのが現実的です。

---

## 🐍 PythonとC言語の比較（初心者の方へ）

* 🐍 Pythonでは `@dataclass` を使って、
  「このデータはこのキーだけ持てばいい」という入れ物を簡単に宣言できます。

* 🔧 C言語だと、これは `struct` に相当します。

  たとえば C ではこういうイメージになります：

  ```c
  struct SelectItemInputData {
      char* slot_id;
  };

  struct SelectItemOutputData {
      char* item_name;
      int change;
  };
  ```

  どちらも「ロジックは持たず、ただの入れ物」として使われる点は同じです。

---

## 🛡️ DTOの鉄則

> ただ運び、何も考えるな。
> (Just carry, don't think.)

DTOは境界を越えるときにだけ意味がある“書類”です。
ここにビジネスルールを書き始めると、一気に設計が崩れます。
（境界のあちこちで勝手に判断が走る＝バグ温床になる）

UseCaseは「公式な入力・公式な出力」だけを扱う。
Presenterは「公式な出力」だけを受け取る。
このルールで、層どうしの結合がゆるく、テストしやすい状態が保てます。

---

ここまでで「何を渡すか（dto.py）」が定義できました。
次は「誰に頼れるか」を決めます。
つまり `boundaries.py` です。

UseCaseが外部に対して「このメソッドを持っててくれれば君を使えるよ」と宣言する場所になります。
RepositoryもHardwareもPresenterも、全部そこで“契約”として並びます。

