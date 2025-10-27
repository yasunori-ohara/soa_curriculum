# 06 Data Structures (DTO / ViewModel)

# 📦 Data Structures
### `core/usecase/boundary/dto.py`

## ✉️ このファイルの役割

このファイルに集められたクラス群は、レイヤーの境界を越えてデータを運ぶためだけの、単純なデータコンテナ（運び屋）です。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

`Controller`から`Use Case`へ、`Use Case`から`Presenter`へといった情報の受け渡しの際に、辞書のような曖昧な形式ではなく、公式に定められた構造を持つオブジェクトを使うことで、各レイヤー間の**結合を疎に保ちます**。

---

## ✅ このファイルにあるべき処理

⭕️ **含めるべき処理の例**

* 運搬するデータフィールド（属性）の定義のみ。

❌ **含めてはいけない処理の例**

* ビジネスロジック、計算、バリデーションなど、**いかなるロジックも禁止**
* データベースへのアクセスやUI更新など、他レイヤーの責務を持つ処理

---

## 📁 このファイルの配置

```
├─ core/
│   └─ usecase/
│       └─ boundary/
│           ├─ input_boundary.py    # <I> InputBoundary（Controllerが呼ぶ）
│           ├─ output_boundary.py   # <I> OutputBoundary（Presenterが実装）
│           └─ dto.py               # <DS> DataStructure（Input/Output/ViewModel）
```

---

## 🔍 ソースコード（コメント充実）

```python
# --------------------------------------------------------------------
# File: core/usecase/boundary/dto.py
# Layer: Data Structures（レイヤー間のデータ運搬専用）
#
# 目的:
#   - 各レイヤー間でデータを受け渡すための「公式な箱」を定義する。
#   - ロジックは一切持たず、ただの構造体として振る舞う。
#
# C言語の感覚に近い説明:
#   - dataclass は「構造体 struct」に似ており、フィールド定義だけで使える。
#   - frozen=True は「const構造体」のようなもので、変更不可を保証する。
# --------------------------------------------------------------------

from dataclasses import dataclass

# --------------------------------------------------------------------
# <DS> Input Data に相当
#
# Controllerが生成し、Use Caseへ渡すためのデータ構造。
# UIからの生データを、ビジネスロジックが扱いやすい形に整える役割。
# frozen=True は、このオブジェクトが生成後に変更されない「イミュータブル」
# であることを示し、データの不変性を保証する。
# --------------------------------------------------------------------
@dataclass(frozen=True)
class TodoInputData:
    """
    Controller → Use Case に渡すデータ構造。

    - title: ユーザーが入力したタスク名
    - frozen=True により、生成後に属性を変更できない（安全性向上）
    """
    title: str


# --------------------------------------------------------------------
# <DS> Output Data に相当
#
# Use Caseが生成し、Presenterへ渡すためのデータ構造。
# ビジネスロジックの結果を、プレゼンテーション層が扱いやすい形に整える。
# Entityそのものではなく、必要な情報だけを抽出して渡す。
# --------------------------------------------------------------------
@dataclass(frozen=True)
class TodoOutputData:
    """
    Use Case → Presenter に渡すデータ構造。

    - id: 新しく追加されたTodoのID
    - title: Todoのタイトル
    """
    id: int
    title: str


# --------------------------------------------------------------------
# <DS> View Model に相当
#
# Presenterが生成し、View（画面）へ渡すためのデータ構造。
# ViewはこのViewModelを描画するだけで、ロジックは持たない。
# --------------------------------------------------------------------
@dataclass
class TodoViewModel:
    """
    Presenter → View に渡すデータ構造（画面表示用）

    - display_text: 表示用の文字列（例: "✅ タスク名 (ID: 1)"）

    frozen=False（デフォルト）にすることで、
    Presenterがこのオブジェクトを加工できるようにしている。
    """
    display_text: str = ""
```

---

## 🧩 なぜ `@dataclass` を使うのか？

* Pythonで構造体的なクラスを簡潔に定義できる。
* `__init__` や `__repr__` などを自動生成してくれるため、コード量を削減できる。
* 型ヒント付きで明確な構造を持つため、IDEや静的解析ツールによる補助が効く。

---

## 🔐 なぜ `frozen=True` が重要なのか？

* イミュータブル（変更不可）にすることで、**安全性と予測可能性が向上**する。
* 受け渡し中にデータが意図せず変更されることを防ぎ、**バグの温床を減らす**。
* 特に `InputData` や `OutputData` は、**一度生成されたら変更されるべきではない**ため、`frozen=True` が適している。

---

## 🔄 なぜ `TodoViewModel` はミュータブルなのか？

* Presenterが画面表示用に加工する必要があるため。
* ViewModelは「表示のための状態」を保持する役割なので、**Presenterが自由に編集できるようにしておく**のが自然。

---

## 🧪 この段階でユニットテストをすることができます

```python
# --------------------------------------------------------------------
# ユニットテスト例: Data Structures の振る舞い確認
# - Input/OutputData は変更不可（frozen=True）
# - ViewModel は変更可能（frozen=False）
# --------------------------------------------------------------------

import unittest
from core.usecase.boundary.dto import TodoInputData, TodoOutputData, TodoViewModel


class TestDataStructures(unittest.TestCase):
    def test_input_data_is_immutable(self):
        """InputDataは変更不可であることを確認"""
        data = TodoInputData(title="タスク")
        with self.assertRaises(AttributeError):
            data.title = "変更不可"

    def test_output_data_is_immutable(self):
        """OutputDataも変更不可であることを確認"""
        data = TodoOutputData(id=1, title="タスク")
        with self.assertRaises(AttributeError):
            data.id = 99

    def test_view_model_is_mutable(self):
        """ViewModelは変更可能であることを確認"""
        vm = TodoViewModel(display_text="初期表示")
        vm.display_text = "更新可能"
        self.assertEqual(vm.display_text, "更新可能")


if __name__ == "__main__":
    unittest.main()
```

---

## 🛡 鉄則

> ただ運び、何も考えるな。

* ロジック禁止（計算・判定・条件分岐などは一切書かない）
* DBやUIの責務禁止（保存・表示などは他レイヤーに任せる）
* 不変性を守る（Input/Outputは `frozen=True`）
