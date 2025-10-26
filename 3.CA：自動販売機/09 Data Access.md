# 09 Data Access

# 💾 Data Access : `vending_machine/interface_adapters/data_access.py`

ここでは自販機の「在庫データ」を扱うアダプタを説明します。
このアダプタは、自販機の各スロットに何が入っていて、いま何本残っているのか、といった情報を管理します。

役割を一言で言うと：

> UseCase から「A1番スロットの情報ちょうだい」「この商品、在庫1本減ったから保存して」と依頼されたときに、
> 実際にデータを探したり更新したりする倉庫係。

`UseCase` は「在庫がどこに保存されているか」を知りません。
メモリなのか、DBなのか、クラウドAPIなのか、ファイル保存なのか。ぜんぶ知りません。
それを隠す壁の役が、この Data Access（リポジトリ）です。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## 🎯 どこに置くの？誰が誰に依存するの？

フォルダ構成ではこうなります👇

```text
vending_machine/
├─ domain/
│   ├─ entities.py             # Item / PaymentManager など（ビジネスルールの中心）
│   └─ errors.py               # ドメイン固有の例外
│
├─ usecase/
│   ├─ dto.py                  # InputData / OutputData / ViewModel
│   ├─ boundaries.py           # UseCaseが外部に求める契約 (ItemDataAccessInterface など)
│   ├─ select_item_usecase.py  # 商品購入フロー
│   └─ insert_coin_usecase.py  # コイン投入フロー
│
├─ interface_adapters/
│   ├─ controller.py           # ユーザーの操作をUseCaseに渡す
│   ├─ presenter.py            # UseCase結果をViewModelに整える
│   ├─ view_console.py         # ユーザーとの入出力ループ
│   ├─ data_access.py          # ← いまここ：在庫リポジトリの具体実装
│   └─ hardware_adapter.py     # ハードウェア操作の具体実装（今回はprintでシミュレート）
│
└─ main.py                     # 全部をnewして配線する場所
```

キモは依存方向です。

* `usecase/` 側には `ItemDataAccessInterface` という「契約(インターフェース)」だけがある
* `interface_adapters/data_access.py` 側が、その契約を“実装する”具体クラスを持つ

これはクリーンアーキテクチャの超大事なパターンです：
「内側（usecase）はインターフェースだけを知る。外側（adapter）が実装を担う。」

---

## ✅ Data Access がやっていいこと / やっちゃダメなこと

⭕ やっていいこと

* 在庫一覧を保持する（今回はインメモリ辞書）
* スロットIDで商品を検索する
* 商品の在庫を更新して保存する

❌ やっちゃダメなこと

* 「在庫が0なら売れない」と判断する（それは `Item` や `UseCase`）
* 「お金が足りるか？」を判断する（それは `PaymentManager` や `UseCase`）
* ハードウェアを動かす（それは `hardware_adapter.py`）

> Data Access は「倉庫の番人」。
> 倉庫からモノを取り出したり、戻したりするだけ。売っていいかどうかは考えない。

---

## 💻 コード（`data_access.py`）

いまはデータベースは使わず「インメモリ版（プログラムの中だけで完結する簡易DB）」で実装します。
これだけでも UseCase 側からは「ちゃんとリポジトリがある」ように見えるので、学習・テストには十分です。

```python
# vending_machine/interface_adapters/data_access.py

from typing import Dict, Optional

# 内側レイヤーの定義をインポートする
from vending_machine.domain.entities import Item
from vending_machine.usecase.boundaries import ItemDataAccessInterface


# -----------------------------------------------------------------------------
# InMemoryItemDataAccess
# - クラス図の位置: DataAccess (Repository)
# - 同心円図の位置: interface_adapters（外側の円）
#
# これは ItemDataAccessInterface（= UseCaseが期待する契約）を
# インメモリ辞書で実装した具体クラスです。
#
# UseCaseから見ると「商品保管庫に問い合わせできる相手」。
# 実際の保管場所・保存方法は知られない（隠蔽される）。
# -----------------------------------------------------------------------------
class InMemoryItemDataAccess(ItemDataAccessInterface):
    """
    自販機の中の商品スロットを、Pythonの辞書で管理する簡易リポジトリ。

    例:
        "A1": Item(slot_id="A1", name="お茶", price=160, stock=5)

    実務ではここがDB（SQL, MongoDBなど）や外部API呼び出しに
    差し替わることになりますが、UseCase側のコードは一切変わりません。
    """

    def __init__(self):
        # この辞書が「仮想の在庫データベース」というイメージです
        self._database: Dict[str, Item] = {
            "A1": Item(slot_id="A1", name="お茶",     price=160, stock=5),
            "B2": Item(slot_id="B2", name="コーヒー", price=130, stock=3),
            "C3": Item(slot_id="C3", name="水",       price=110, stock=0),  # 在庫切れサンプル
        }

    def find_by_slot_id(self, slot_id: str) -> Optional[Item]:
        """
        [インターフェースの実装]
        指定スロットの商品(Item)を返す。

        UseCase は「slot_id='A1'ください」とだけ言えばよく、
        この中身が辞書だろうがDBだろうが知らなくてよい。
        """
        return self._database.get(slot_id)

    def save(self, item: Item) -> Item:
        """
        [インターフェースの実装]
        在庫の変更結果を保存する。

        例:
            item.dispense() によって stock が 5 -> 4 になったあと、
            ここでその状態を"永続化"するイメージ。
        """
        self._database[item.slot_id] = item
        return item
```

---

## 🔎 ここであえて扱わないもの：PaymentManager の「保存」

「PaymentManager（投入金額などの状態）も Data Access で扱わなくていいの？」という疑問はすごく自然です。

結論：この第三巡では、**扱いません**。理由はこの3つです。

1. `PaymentManager` はビジネス上「今この客がいくら入れているか？」というリアルタイムのワーク中の状態。
   「棚卸ししてDBに残しておくべき履歴」ではない。

2. 自販機アプリが動いている間、`PaymentManager` は1つだけ生きていて、
   コイン投入や返金のたびにその1つが更新され続けます。
   これはDBよりも「現在のセッションオブジェクト」に近い。

3. この「どこで持つか？」という話は、実はアーキテクチャの応用編ネタなんです。

   * main.py で1つ作って全体に注入する
   * Controller が持ってUseCaseに渡す
   * ハードウェア側から渡す
     などいくつか戦略があるので、そこは main.py の章で整理します。

なので Data Access 章では、対象を **Item の在庫** に絞るほうがシンプルで学びやすいです。
「まずは在庫リポジトリ」を理解できればOKです。

---

## 🧪 DataAccess のユニットテスト

Data Access のテストはすごくストレートです。
やりたいことはただひとつ：「saveした内容が find で取れること」を確かめること。

```python
# tests/interface_adapters/test_data_access.py

from vending_machine.interface_adapters.data_access import InMemoryItemDataAccess
from vending_machine.domain.entities import Item


def test_saveした商品が_find_by_slot_idで取得できる():
    # 1. Arrange (準備)
    repo = InMemoryItemDataAccess()

    new_item = Item(
        slot_id="Z9",
        name="エナジードリンク",
        price=200,
        stock=7,
    )

    # 2. Act (実行)
    repo.save(new_item)                    # 在庫を保存
    result = repo.find_by_slot_id("Z9")    # 取り出しテスト

    # 3. Assert (検証)
    assert result is not None
    assert result.name == "エナジードリンク"
    assert result.stock == 7
```

ここでチェックしているのはビジネスルールではありません。

* `Item.dispense()` をちゃんと呼んだか？
* お金は足りていたか？
  みたいなことは一切見ていないですよね。

テストしているのは純粋に「倉庫から同じものが出てくるか？」だけ。
これが Data Access の責務そのものです。

---

## 🐍 Python と 🔧 C言語のイメージ比較

* 🐍 Python＋クリーンアーキテクチャ

  * `ItemDataAccessInterface` という抽象（契約）を UseCase が使う
  * `InMemoryItemDataAccess` という具体実装は外側にある
  * 後でDBに変えても UseCase は一切変更不要

* 🔧 C言語でありがちなやり方

  * 在庫配列やグローバル変数を main.c 内で直接更新
  * 在庫を更新する関数の中で、ついでにハードウェアを動かしたり、ログをprintfしたりしがち
  * つまり「データ管理」と「ビジネス判断」と「ハード操作」が混ざってテスト不能になる

今回の分割は、「在庫データを触る場所」を一箇所に閉じ込めて、他の層に漏らさないためのものです。

---

## 🛡️ このクラスの鉄則

> データを取り出し、保存せよ。ただし考えるな。
> (Get and save. Do not think.)

* Data Access は「倉庫番」。在庫を探し、更新して、戻すだけ。
* 「これは売っていいのか？」の判断はしない。
* 「売った結果こう表示するべき」は考えない。
* それらは必ず UseCase / Entity / Presenter に任せる。

この鉄則を守ると、
「DBを変えたい」「Web API にしたい」「テストではインメモリでいい」
といった現実的な要求にめちゃくちゃ強いアーキテクチャになります。

