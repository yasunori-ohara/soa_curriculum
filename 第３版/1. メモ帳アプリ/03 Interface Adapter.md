# 第3章：Interface Adapter（インターフェースアダプタ）

## 🎯 この章の目的

* **Controller／Presenter／Repository Adapter（= Gateway的役割）**を、**クラス図の箱に1対1対応**で実装する
* **Controller は InputBoundary にだけ依存**／**Presenter は OutputBoundary を実装**を徹底して、
  「**依存は内へ、制御は外へ**」をコードで体感する
* **ViewModel（DS）**を明示して「表示都合の整形は Presenter の仕事」をはっきりさせる

> 用語ミニ解説（この章で出るものだけ）
>
> * **ViewModel（DS）**：View（表示側）がそのまま使える“整形済みの箱”。
> * **Repository Adapter**：UseCaseが知っている**Repository契約**を実装する**橋**。保存先（ファイル/DB）はこの外側（次章）で決める。

---

## 🧩 Interface Adapterとは（図対応）

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

| クラス図の箱                          | 役割                                      | この章のファイル                          |
| ------------------------------- | --------------------------------------- | --------------------------------- |
| **Controller**                  | 外の入力を **InputData** に“翻訳”して UseCase を呼ぶ | `interface/controller.py`         |
| **Presenter（OutputBoundary実装）** | **OutputData → ViewModel** に“翻訳”        | `interface/presenter.py`          |
| **ViewModel（DS）**               | 表示用の整形済みデータ                             | `interface/presenter.py`          |
| **Repository Adapter**          | **Repository契約**を実装（永続化は次章へ委譲）          | `interface/repository_adapter.py` |

👉 **依存の向き**：Controller/Presenter/RepositoryAdapter → UseCase → Entity（**内向き**のみ）


## 🧱 ファイル構成（第2章の契約に準拠）

```
project_root/
├── domain/
│   └── note.py
├── usecase/
│   ├── dto.py
│   ├── boundaries/
│   │   └── note_boundaries.py
│   ├── contracts/
│   │   └── note_repository.py
│   ├── create_note.py
│   └── get_all_notes.py
└── interface/
    ├── controller.py
    ├── presenter.py
    └── repository_adapter.py
```


## 💡 この層のポイント

| 要素                     | 要点                                                                  |
| ---------------------- | ------------------------------------------------------------------- |
| **Controller**         | **InputBoundary** にだけ依存。外の入力 → `CreateNoteInput` へ変換して `execute()`。 |
| **Presenter**          | **OutputBoundary** を**実装**。`present(out)` で **ViewModel** を作る。      |
| **Repository Adapter** | **Repository契約**を**実装**。ここでは**メモリ**に保存（次章でファイル/DBへ差し替え可能）。          |
| **ViewModel**          | “表示都合”の整形（プレビュー文字数など）は Presenter に集約。                               |

---

## 🧠 実装

### ✳️ `interface/controller.py`（Controller）

```python
# --------------------------------------------------------------------
# [クラス図] Controller（左端）
# [同心円] Interface Adapters（外→内の翻訳：入力→InputData）
# 役割：外部入力を UseCase の InputBoundary に沿って呼び出す
# --------------------------------------------------------------------
from usecase.boundaries.note_boundaries import (
    CreateNoteInputBoundary, GetAllNotesInputBoundary
)
from usecase.dto import CreateNoteInput
from interface.presenter import NotePresenter  # OutputBoundary実装

class NoteController:
    """
    コントローラは UseCase の「具体型」を知らない。
    InputBoundary（契約）にだけ依存している点がポイント。
    """

    def __init__(
        self,
        create_uc: CreateNoteInputBoundary,
        list_uc: GetAllNotesInputBoundary
    ):
        self._create_uc = create_uc
        self._list_uc = list_uc

    def create(self, title: str, content: str):
        """
        外の入力（たとえばCLI/HTTPフォーム）→ InputData へ翻訳して UseCase を実行。
        結果の ViewModel は Presenter が生成するので、ここでは Presenter を用意して注入し、
        返ってきた ViewModel をそのまま上位（View層）へ返す。
        """
        presenter = NotePresenter()                      # OutputBoundary実装
        # UseCase に「今回の出力はこの Presenter へ」とセット（第2章の設計）
        # 注意: set_output_boundary は UseCaseInteractor 側に用意してある前提
        self._create_uc.set_output_boundary(presenter)  # type: ignore[attr-defined]

        inp = CreateNoteInput(title=title, content=content)
        self._create_uc.execute(inp)

        return presenter.view_model()  # View（表示担当）に渡すための整形済みデータ

    def list_all(self):
        """
        一覧取得: Presenter を用意 → UseCase 実行 → ViewModel（配列）を返す。
        """
        presenter = NotePresenter()
        self._list_uc.set_output_boundary(presenter)     # type: ignore[attr-defined]
        self._list_uc.execute()
        return presenter.view_model_list()
```

> **クラス図対応**：Controller → InputBoundary（**静的依存**）。
> **実行時**は、Controller が Presenter を用意し **UseCase に注入**し、
> UseCase → Presenter → ViewModel という**制御の流れ**になります。

---

### ✳️ `interface/presenter.py`（Presenter／ViewModel）

```python
# --------------------------------------------------------------------
# [クラス図] Presenter / ViewModel（左側：Presenter→ViewModel）
# [同心円] Interface Adapters（内部出力→表示都合のデータに翻訳）
# 役割：UseCaseの OutputData を ViewModel（DS）に整形
# --------------------------------------------------------------------
from typing import Optional, List
from usecase.boundaries.note_boundaries import (
    CreateNoteOutputBoundary, GetAllNotesOutputBoundary
)
from usecase.dto import CreateNoteOutput, GetAllNotesOutput

class NoteViewModel:
    """
    View（CLI/HTML/JSON）が直接使える整形済みデータ
    - クラス図での ViewModel（DS）に相当
    """
    def __init__(self, id: int, title: str, preview: str):
        self.id = id
        self.title = title
        self.preview = preview

    def __repr__(self) -> str:
        return f"[{self.id}] {self.title} - {self.preview}"

class NotePresenter(CreateNoteOutputBoundary, GetAllNotesOutputBoundary):
    """
    OutputBoundary を実装する Presenter。
    UseCase から present(out) が呼ばれると ViewModel を構築して保持する。
    """
    def __init__(self) -> None:
        self._single: Optional[NoteViewModel] = None
        self._list: List[NoteViewModel] = []

    # --- 作成ユースケースの出力を受け取る
    def present(self, out: CreateNoteOutput) -> None:  # type: ignore[override]
        preview = (out.content[:15] + "...") if len(out.content) > 15 else out.content
        self._single = NoteViewModel(id=out.id, title=out.title, preview=preview)

    # --- 一覧ユースケースの出力を受け取る
    def present_list(self, out: GetAllNotesOutput) -> None:
        # ↑ インターフェースを分けたいなら、GetAllNotesOutputBoundary.present を別名にしてもOK
        self._list = [NoteViewModel(id=n.id, title=n.title, preview="") for n in out.notes]

    # インターフェース整合（GetAll用の present 名）
    def present(self, out: GetAllNotesOutput) -> None:  # type: ignore[override]
        self.present_list(out)

    # --- View（表示側）が取得するアクセサ
    def view_model(self) -> NoteViewModel:
        assert self._single is not None, "present(CreateNoteOutput) が未実行です"
        return self._single

    def view_model_list(self) -> List[NoteViewModel]:
        return list(self._list)
```

> **クラス図対応**：UseCase → OutputBoundary（**静的依存**）→ Presenter が受け取り、**ViewModel（DS）** を作成。
> ここで“表示都合の整形（プレビュー文字数など）”を完結させます。

---

### ✳️ `interface/repository_adapter.py`（Repository Adapter）

```python
# --------------------------------------------------------------------
# [クラス図] Repository Adapter（右側：Data Access Interface の実装）
# [同心円] Interface Adapters（Entity ⇔ 保管形式 の橋）
# 役割：UseCase層の Repository契約 を満たす。保存先は次章で差し替え可能。
# --------------------------------------------------------------------
from typing import List
from usecase.contracts.note_repository import NoteRepository
from domain.note import Note

class InMemoryNoteRepository(NoteRepository):
    """
    学習用の最小実装：メモリに保持するだけ。
    - 次章（Infrastructure）で「ファイル保存」や「DB保存」に差し替えられる。
    - UseCaseは契約（NoteRepository）しか知らないため、UseCase側の変更は不要。
    """
    def __init__(self) -> None:
        self._notes: List[Note] = []
        self._next_id: int = 1

    def next_id(self) -> int:
        nid = self._next_id
        self._next_id += 1
        return nid

    def save(self, note: Note) -> Note:
        self._notes.append(note)
        return note

    def get_all(self) -> List[Note]:
        return list(self._notes)
```

> **クラス図対応**：UseCase → **Repository契約（I）** → **Repository Adapter（実装）**。
> **保存先の技術は外側**（次章）で自由に入れ替え可能です。

---

## ⚙️ 依存と制御の整理（図で再確認）

![クリーンアーキテクチャ・依存と制御](../クリーンアーキテクチャ・依存と制御.png)

| 層 / 要素             | 静的依存（import）                             | 動的制御（実行の向き）                  |
| ------------------ | ---------------------------------------- | ---------------------------- |
| Controller         | → InputBoundary（UseCase）                 | 外→内へ呼び出す                     |
| UseCase            | → Entity / Repository契約 / OutputBoundary | Controllerから呼ばれ、Presenterへ通知 |
| Presenter          | （UseCaseに静的依存しない）OutputBoundaryを実装       | UseCaseから呼ばれ、ViewModelを生成    |
| Repository Adapter | → Entity（Note）                           | UseCaseから呼ばれる（保存・取得）         |

> **依存は内へ**（静的依存の矢印は内側だけ） / **制御は外へ**（実行は外から入り、結果は外へ戻る）。

---

## 💡 理解ポイント

| 観点                        | 内容                                                        |
| ------------------------- | --------------------------------------------------------- |
| **Controllerの責務**         | 入力の“翻訳係”。**具体UseCaseを知らず**、InputBoundaryに従って呼ぶ。           |
| **Presenterの責務**          | 出力の“整形係”。**OutputBoundaryを実装**し、**ViewModel**を作る。         |
| **Repository Adapterの責務** | **Repository契約**の実装。保存先は次章で差し替え可能。                        |
| **テスタビリティ**               | Controller/Presenterは**モック可能**。UseCase/Entityは外の技術に依存しない。 |

---

## 🧪 テストのヒント（この章の範囲）

* **Controller単体**：InputBoundary を**テスト用ダミー**に差し替え、`create()` が **InputData** を期待どおり渡すか。
* **Presenter単体**：`present()` に **OutputData** を与えて **ViewModel** の整形（プレビュー長など）を確認。
* **Repository Adapter単体**：`next_id()` の連番、`save()`→`get_all()` の往復。

> コツ：**“翻訳点”を境にテストする**（HTTPやファイルは出さない）。外部技術は次章で。

---

## 🔄 次のステップ：Infrastructure層へ

この章で「内外の翻訳係」が揃いました。
次は最外層の **Infrastructure（Frameworks & Drivers）** で、

* **View（CLI/HTML/JSON）** の実装（Presenterの **ViewModel** を表示）
* **保存先の具体化**（メモリ → ファイル/SQLite など）
  を行い、**差し替え可能**であることを体験します。

---

### 🔍 最後に：図とコードの対比チェック

* **Controller**：左端の箱→**InputBoundary**依存のみでOK ✅
* **Presenter**：左下の箱→**OutputBoundary実装＋ViewModel作成** ✅
* **Repository Adapter**：右下の箱→**Repository契約の実装** ✅
* **依存矢印**：すべて**内向き**（外側は内側を import、内側は外側を知らない）✅

