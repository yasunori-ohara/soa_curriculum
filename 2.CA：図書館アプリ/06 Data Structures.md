# 06 Data Structures (DTO / ViewModel)

# 📦 Data Structures
### `core/usecase/boundary/dto.py`

---

## 🎯 このファイルの役割

`dto.py` は、レイヤー間で受け渡される**データだけを運ぶための構造体（DTO）と、画面表示用のViewModel**を定義します。

クリーンアーキテクチャでは、層と層の間をまたいでデータが流れますが、そのときに生の Entity（`Book`, `Loan` など）をそのまま渡すのは危険です。
代わりに「必要な情報だけを抜き出した、安全で用途特化の“箱”」をやり取りします。その“箱”がこのファイルで定義する **DTO（Data Transfer Object）** です。

このファイルで扱う主要なデータ構造：

* `CheckOutBookInputData`
  Controller → Use Case に渡す入力データ
* `CheckOutBookOutputData`
  Use Case → Presenter に渡す出力データ
* `BookViewModel`
  Presenter → View に渡す表示用データ（UIで表示する最終形）

これらは**ビジネスロジックを含みません。計算もしません。メソッドも基本持ちません。**
あくまで「値のまとまり」です。

---

## 📁 ファイルの配置（正式版）

図書館アプリでは、DTOは Use Case 層の `boundary` 配下に置きます。

```text
├─ core/
│   ├─ domain/
│   │   ├─ book.py                     # <E> Book Entity
│   │   ├─ member.py                   # <E> Member Entity
│   │   ├─ loan.py                     # <E> Loan Entity
│   │   └─ repository.py               # <I> Repositoryインターフェース群
│   │
│   └─ usecase/
│       ├─ boundary/
│       │   ├─ dto.py                  # <DS> ← このページで説明
│       │   ├─ input_boundary.py       # <I> 入力ポート (Controllerが呼ぶ)
│       │   └─ output_boundary.py      # <I> 出力ポート (Presenterが実装)
│       │
│       └─ interactor/
│           └─ check_out_book.py       # <UC> 「本を貸し出す」ユースケース
│
├─ interface_adapters/
│   ├─ presenters/
│   │   └─ checkout_presenter.py       # Presenter (OutputBoundary実装)
│   ├─ controllers/
│   │   └─ checkout_controller.py      # Controller (InputBoundaryを呼ぶ)
│   └─ views/
│       └─ ...                         # CLI / GUI / WebのView
```

この配置によって、アプリケーション内の全てのやり取りが「共通の箱の定義（dto.py）」を通ることになります。
Use Case 側が「必要な形」を主導できる、というのがポイントです。

---

## 🔍 ソースコード（正式版）

```python
# --------------------------------------------------------------------
# File: core/usecase/boundary/dto.py
# Layer: Use Case Boundary data structures (DTO / ViewModel)
#
# 目的:
#   - レイヤー間を移動するデータの「形」を定義する。
#   - ロジックを一切持たない、ただのデータの箱。
#
# C言語の感覚に近い説明:
#   - これは struct に近い。関数ポインタや処理は持たない。
#   - 値をまとめて別の関数に渡すための入れ物。
# --------------------------------------------------------------------

from datetime import date
from typing import NamedTuple


# --------------------------------------------------------------------
# <DS> CheckOutBookInputData
# Controller -> Use Case に渡す入力データ
#
# Controllerはユーザー操作（例: "会員123が本456を借ります"）
# を受け取り、このDTOに詰めてUse Caseに渡す。
#
# Use Case側はこの形だけを期待するので、
# WebでもGUIでもCLIでも同じUse Caseが呼べる。
# --------------------------------------------------------------------
class CheckOutBookInputData(NamedTuple):
    book_id: int
    member_id: int


# --------------------------------------------------------------------
# <DS> CheckOutBookOutputData
# Use Case -> Presenter に渡す出力データ
#
# Use Caseはビジネスロジックの結果（どの本を誰にいつまで貸したか等）
# をこのDTOにまとめ、OutputBoundary(=Presenter側のインターフェース)
# に渡す。
#
# ここでは人間向けの整形はまだ行わない。
# 期限日はそのまま date 型のまま持っておき、
# 「YYYY-MM-DD」などの文字列化は Presenter の責務。
# --------------------------------------------------------------------
class CheckOutBookOutputData(NamedTuple):
    book_title: str
    member_name: str
    due_date: date


# --------------------------------------------------------------------
# <DS> BookViewModel
# Presenter -> View に渡す表示用データ（ViewModel）
#
# PresenterはCheckOutBookOutputDataを受け取り、
# 画面/CLI/GUIにそのまま表示できる形の文字列を組み立てる。
#
# ViewはこのViewModelを読むだけでOK。
# Viewはビジネスロジックや整形ロジックを持たないのが理想。
# --------------------------------------------------------------------
class BookViewModel(NamedTuple):
    display_text: str
```

---

## 👀 ざっくりデータの流れ（どこで使われる？）

1. **Controller → Use Case**

   * Controller はユーザー入力を受け取る（例：「会員ID=1 が 本ID=42 を借りたい」）。
   * それを `CheckOutBookInputData(book_id=42, member_id=1)` に詰めて
     `use_case.handle(input_data)` のように呼びます。

2. **Use Case → Presenter**

   * Use Case はビジネスロジックを実行し、

     * 借りた本のタイトル
     * 借りた会員の名前
     * 返却期限（date型）
       を `CheckOutBookOutputData` にして Presenter に渡します。

3. **Presenter → View**

   * Presenter は「人間が読める文章」に整形します。
   * 例：

     ```python
     f"📕『{book_title}』を {member_name} さんに貸し出しました。返却期限: {due_date:%Y-%m-%d}"
     ```
   * それを `BookViewModel(display_text=...)` に入れます。
   * View はその `display_text` をそのまま表示するだけ。

👉 これで View はビジネスルールや日付整形に触れずに済みます。
👉 逆に Use Case は「CLI用の文言」「HTMLのタグ」などを一切知らずに済みます。

役割を分離できていて、とてもいい ✨

---

## ❌ やってはいけないこと

* DTO や ViewModel にロジックを入れない
  → ここで if 文や日付計算をし始めると境界が濁ります

* Entity（`Book`, `Loan` など）をそのまま UI に渡さない
  → UIが Entity を勝手にいじったり、ビジネスルールに直接触ったりできてしまうのは危険

* Presenter が勝手に新しいDTO型を発明しない
  → 型が散らかると依存関係も散らかるので、原則として DTO と ViewModel の定義はここ一箇所に集約します

---

## 🧪 このファイルはテストすべき？

基本的には**このファイル単体のユニットテストは不要**です。

理由はシンプルで、`NamedTuple` はロジックを持たない「ただの箱」だからです。
テストすべきポイントは「これらが正しく使われているか？」であって、それは下位ではなく上位のテストで検証されます：

* Use Case のテスト
  → 正しい `CheckOutBookOutputData` が Presenter に渡ったか
* Presenter のテスト
  → `BookViewModel.display_text` が期待どおりの文字列になっているか

---

## 🐍 Python と C言語の比較（初心者向け）

* Pythonでは、`CheckOutBookInputData` のような `NamedTuple` は
  「フィールド名つきの読み取り専用データ構造」を簡単に作れます。

* C言語で言うなら、`struct CheckOutBookInputData { int book_id; int member_id; };` のようなイメージです。
  ただしCの `struct` は可変だけど、ここでは「意図せぬ変更を防ぐ」ために不変（イミュータブル）として扱っている点がちょっと違います。

---

## 🛡 鉄則

> データは箱で渡せ。意思決定は箱の外でやれ。

* DTOは「何が起きたか／何をしたいか」という事実だけを運ぶ
* 見せ方（文字列整形など）は Presenter の責務
* 表示するだけの最終形（ViewModel）を受け取った View は “バカでいい”

  * View は `print(view_model.display_text)` するだけで成立するのが理想

このルールを守ることで、
UIの変更（CLI→GUI→Web）や、表示文言の変更、DBの種類の変更に強い、長生きする設計になります。
