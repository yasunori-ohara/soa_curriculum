# 第2章：UseCase（ユースケース）

## 🎯 この章の目的

* **UseCaseInteractor** を「**入口（InputBoundary）**」と「**出口（OutputBoundary）**」つきで実装する
* **DTO（＝境界を渡るデータの箱）**を **InputData／OutputData** として明示する
* **Repository の契約**（抽象インターフェース）を **UseCase層に置く**（＝依存性の逆転）
* コードの各クラス・関数が**クラス図のどの箱に当たるか**をコメントで確認できる

> 用語ミニ解説
> **DTO（Data Transfer Object）**：層の境界をまたぐ“データだけの箱”。ロジックは持たない。ここでは **InputData／OutputData** として使います。
> **Boundary（境界）**：UseCase の **入口／出口の取り決め（インターフェース）**。Controller は **InputBoundary** に従って呼び出し、UseCase は **OutputBoundary** へ結果を渡します。

---

## 🧩 UseCaseの位置づけ（図対応）

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

| クラス図の箱                       | 役割                         | この章のファイル                                             |
| ---------------------------- | -------------------------- | ---------------------------------------------------- |
| **UseCaseInteractor**        | 手続きの中心（アプリ固有の流れ）           | `usecase/create_note.py`, `usecase/get_all_notes.py` |
| **InputBoundary（I）**         | Controller → UseCase の入口契約 | `usecase/boundaries/note_boundaries.py`              |
| **OutputBoundary（I）**        | UseCase → Presenter の出口契約  | `usecase/boundaries/note_boundaries.py`              |
| **InputData／OutputData（DS）** | 境界を渡る純粋データ                 | `usecase/dto.py`                                     |
| **Repository Interface（I）**  | データ出し入れの契約                 | `usecase/contracts/note_repository.py`               |

👉 依存は常に**内側へ**（Controller/Presenter → UseCase → Entity）。
UseCase は **DBやUIを知らず**、**抽象（契約）**にだけ依存します。

---

## 🧱 フォルダ構成（手直し後）

```
project_root/
├── domain/
│   └── note.py                  ← 第1章：Entity
└── usecase/
    ├── dto.py                   ← InputData / OutputData（DS）
    ├── boundaries/              ← Controller／Presenterとの契約（I）
    │   └── note_boundaries.py
    ├── contracts/               ← Repository契約（I）
    │   └── note_repository.py
    ├── create_note.py           ← UseCaseInteractor（作成）
    └── get_all_notes.py         ← UseCaseInteractor（一覧）
```

> ※ 既存の `usecase/note_repository.py` は **`usecase/contracts/note_repository.py`** に移動・改名するのがおすすめです（図の箱に合わせやすくなります）。


## 💡 設計方針（今回の追加点）

* **Boundaryを明示**：

  * Controller は UseCase の**具体型**ではなく **InputBoundary** に依存
  * UseCase は Presenter の**具体型**ではなく **OutputBoundary** に依存
* **DTOを明示**：HTTP/DB都合から独立した「**InputData／OutputData**」を使う
* **Repositoryは契約のみ**：保存先（ファイル／DB）は後の層（外側）で実装


## 🧠 コード（Noteの作成／一覧）

### ✳️ `usecase/dto.py`（InputData／OutputData：DS）

```python
# --------------------------------------------------------------------
# [クラス図] InputData / OutputData（DS）
# [同心円] UseCase層（外の形式から独立した“データだけの箱”）
# --------------------------------------------------------------------
from dataclasses import dataclass
from typing import List

# 作成ユースケース
@dataclass
class CreateNoteInput:   # = InputData
    title: str
    content: str

@dataclass
class CreateNoteOutput:  # = OutputData
    id: int
    title: str
    content: str

# 一覧ユースケース（今回は簡単化：Noteの要約を返す）
@dataclass
class NoteSummary:
    id: int
    title: str

@dataclass
class GetAllNotesOutput:  # = OutputData
    notes: List[NoteSummary]
```

---

### ✳️ `usecase/contracts/note_repository.py`（Repository契約：I）

```python
# --------------------------------------------------------------------
# [クラス図] Data Access Interface（Repository契約）
# [同心円] UseCase層（DIP：実装は外側に追い出す）
# --------------------------------------------------------------------
from abc import ABC, abstractmethod
from typing import List
from domain.note import Note

class NoteRepository(ABC):
    """Noteの保存・取得を扱う抽象契約（実装は外側の層に任せる）"""

    @abstractmethod
    def next_id(self) -> int:
        """次のIDを払い出す（※ Entityは採番を知らない）"""
        raise NotImplementedError

    @abstractmethod
    def save(self, note: Note) -> Note:
        """Noteを保存し、永続化後のNoteを返す"""
        raise NotImplementedError

    @abstractmethod
    def get_all(self) -> List[Note]:
        """すべてのNoteを取得"""
        raise NotImplementedError
```

---

### ✳️ `usecase/boundaries/note_boundaries.py`（入出力の契約：I）

```python
# --------------------------------------------------------------------
# [クラス図] InputBoundary / OutputBoundary
# [同心円] UseCase層（Controller/Presenterとの取り決め）
# --------------------------------------------------------------------
from typing import Protocol
from usecase.dto import (
    CreateNoteInput, CreateNoteOutput,
    GetAllNotesOutput
)

# Controller → UseCase（入口）
class CreateNoteInputBoundary(Protocol):
    def execute(self, inp: CreateNoteInput) -> None: ...

class GetAllNotesInputBoundary(Protocol):
    def execute(self) -> None: ...

# UseCase → Presenter（出口）
class CreateNoteOutputBoundary(Protocol):
    def present(self, out: CreateNoteOutput) -> None: ...

class GetAllNotesOutputBoundary(Protocol):
    def present(self, out: GetAllNotesOutput) -> None: ...
```

---

### ✳️ `usecase/create_note.py`（UseCaseInteractor：作成）

```python
# --------------------------------------------------------------------
# [クラス図] UseCaseInteractor（Create）
# [同心円] UseCase層
# --------------------------------------------------------------------
from typing import Optional
from domain.note import Note
from usecase.dto import CreateNoteInput, CreateNoteOutput
from usecase.contracts.note_repository import NoteRepository
from usecase.boundaries.note_boundaries import (
    CreateNoteInputBoundary, CreateNoteOutputBoundary
)

class CreateNoteUseCase(CreateNoteInputBoundary):
    """
    - Controllerはこの InputBoundary（execute）だけを呼ぶ
    - 保存は Repository契約へ委譲（実装は外側）
    - 出力は OutputBoundaryへ通知（Presenterが受ける）
    """
    def __init__(self, repo: NoteRepository):
        self.repo = repo
        self._presenter: Optional[CreateNoteOutputBoundary] = None

    def set_output_boundary(self, presenter: CreateNoteOutputBoundary) -> None:
        self._presenter = presenter

    def execute(self, inp: CreateNoteInput) -> None:
        new_id = self.repo.next_id()
        note = Note(id=new_id, title=inp.title, content=inp.content)
        saved = self.repo.save(note)

        out = CreateNoteOutput(id=saved.id, title=saved.title, content=saved.content)
        assert self._presenter is not None, "OutputBoundary（Presenter）が未設定です"
        self._presenter.present(out)
```

---

### ✳️ `usecase/get_all_notes.py`（UseCaseInteractor：一覧）

```python
# --------------------------------------------------------------------
# [クラス図] UseCaseInteractor（GetAll）
# [同心円] UseCase層
# --------------------------------------------------------------------
from typing import Optional
from usecase.dto import GetAllNotesOutput, NoteSummary
from usecase.contracts.note_repository import NoteRepository
from usecase.boundaries.note_boundaries import (
    GetAllNotesInputBoundary, GetAllNotesOutputBoundary
)

class GetAllNotesUseCase(GetAllNotesInputBoundary):
    """
    - データ取得方法は知らない（Repository契約に委譲）
    - 取得結果を OutputData（List[NoteSummary]）に詰めて Presenter に渡す
    """
    def __init__(self, repo: NoteRepository):
        self.repo = repo
        self._presenter: Optional[GetAllNotesOutputBoundary] = None

    def set_output_boundary(self, presenter: GetAllNotesOutputBoundary) -> None:
        self._presenter = presenter

    def execute(self) -> None:
        notes = self.repo.get_all()
        out = GetAllNotesOutput(
            notes=[NoteSummary(id=n.id, title=n.title) for n in notes]
        )
        assert self._presenter is not None, "OutputBoundary（Presenter）が未設定です"
        self._presenter.present(out)
```


## 🧩 依存性の逆転（DIP）をこの章で“体感”する

* **UseCase → Repository契約（I）** に依存することで、
  **保存先（ファイル／DB）は外側に追い出される** → UseCaseは変更不要。
* **Controller は InputBoundary に**、**Presenter は OutputBoundary に** だけ依存。
  → **UseCase の具体型**や**Presenter の具体型**を知らなくてよい。
* **DTO（InputData／OutputData）**の導入で、**HTTPやDBの形から独立**。

> これが **「依存は内へ、制御は外へ」**。
> クラス図の線（依存）は**内向き**、実行時の制御（シーケンス）は**外から入り外へ返る**。



## 🧪 テストのヒント（この章の範囲）

* **CreateNoteUseCase 単体**

  * **フェイクRepository**（メモリ）＋ **テスト用Presenter**（OutputBoundaryを実装）
  * タイトル空で `ValueError`（Entity由来）／正常時は `present()` が呼ばれて `OutputData` が正しい
* **GetAllNotesUseCase 単体**

  * フェイクリポジトリにダミーデータを数件入れて `execute()`
  * `present()` に流れてきた `GetAllNotesOutput.notes` が件数・順序とも期待どおり

```python
# 例：Createの最小テスト（擬似コード）
class InMemoryRepo:
    def __init__(self): self._data = []; self._next = 1
    def next_id(self): nid=self._next; self._next+=1; return nid
    def save(self, note): self._data.append(note); return note
    def get_all(self): return list(self._data)

class SpyPresenter:
    def __init__(self): self.captured = None
    def present(self, out): self.captured = out

repo = InMemoryRepo()
uc = CreateNoteUseCase(repo)
p = SpyPresenter()
uc.set_output_boundary(p)

uc.execute(CreateNoteInput(title="T", content="C"))
assert p.captured.title == "T"
```



## 📘 この章の理解ポイント（要点まとめ）

| 観点         | 内容                                                        |
| ---------- | --------------------------------------------------------- |
| **役割**     | Entityを使い、アプリのユースケースを記述                                   |
| **依存先**    | Entity／Repository契約／（出力先としての）OutputBoundary               |
| **知らないもの** | DB実装・UI実装（外側）                                             |
| **DTOの効果** | HTTPやDBの形から独立した“箱”で受け渡しできる                                |
| **教育効果**   | クラス図の **Input/OutputBoundary, DS, Repository(I)** をコードで実感 |



## 📎 補足：UseCase層における「契約ファイル」の正体とは？

UseCase層には2種類の契約があります（いずれも **“外とやり取りするための約束”**）。

| 契約の種類                                   | フォルダ                  | 対象                   | 役割               |
| --------------------------------------- | --------------------- | -------------------- | ---------------- |
| **Repository契約（Data Access Interface）** | `usecase/contracts/`  | Gateway／外部保存先        | 保存・取得などデータ操作の約束  |
| **入出力の契約（Input／Output Boundary）**       | `usecase/boundaries/` | Controller／Presenter | 呼び出し方法と結果受け渡しの約束 |

> 🧭 一般の現場ではこれらを `interfaces/` や `ports/` に **まとめる**こともあります。
> 本教材では **図の箱を1対1で実感**するために分けています（教育上の整理）。

---

## 🔄 次のステップ：Interface Adapter層へ

この章で定義した **契約（Boundary／Repository）** を使って、
**Controller／Presenter／Gateway** を実装します。
ここでも**コードと図の対応コメント**を丁寧に入れていきます。


