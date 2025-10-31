# 第3章：Interface Adapter（Controller／Presenter／Gateway）


## 🎯 この章の目的

この章では、クリーンアーキテクチャの**3層目（外側から2番目）**にあたる
**Interface Adapter（インターフェースアダプタ）層**を実装します。

ここは、**外部世界（Web・DB・UI）と内側（UseCase層）をつなぐ橋**です。

> 💡 この層の主な登場人物は3つ：
>
> * **Controller**：外部（ユーザー入力）を受け取り、UseCaseを呼び出す。
> * **Presenter**：UseCaseの結果を受け取り、View（またはAPIレスポンス）向けに整形。
> * **Gateway**：UseCaseからのデータ操作要求を受け、DBやファイルにアクセス。

---

## 🧩 Interface Adapter の位置づけ

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

| クラス図の要素                   | 役割                                          | 層                 |
| ------------------------- | ------------------------------------------- | ----------------- |
| **Controller**            | ユーザー入力（View → UseCase）を仲介                   | Interface Adapter |
| **Presenter**             | UseCaseの出力をView向けに変換                        | Interface Adapter |
| **Gateway（Repository実装）** | UseCaseの契約（Repository Interface）を実際のDB操作で実装 | Interface Adapter |
| **ViewModel／View**        | 表示層（今回は次章以降で扱う）                             | Framework層        |

---

## 💡 設計方針

| 目的              | 実装方針                                        |
| --------------- | ------------------------------------------- |
| 外部との橋渡しを行う      | Controller／Presenter／Gatewayで担当を分離          |
| UseCase層を直接知らない | BoundaryやRepository Interface経由で依存          |
| 変換処理をまとめる       | Controllerは入力をDS（DTO）に、PresenterはDSを表示形式に変換 |
| フレームワーク非依存を保つ   | FastAPIなどの技術要素はMain層に集約                     |

---

## 🧱 フォルダ構成

```
project_root_v2/
├── interface/
│   ├── controller/
│   │   ├── book_controller.py
│   │   └── member_controller.py
│   ├── presenter/
│   │   ├── book_presenter.py
│   │   └── member_presenter.py
│   └── gateway/
│       ├── book_gateway.py
│       └── member_gateway.py
```

---

## 🧠 各クラスの役割（クラス図対応）

| クラス              | クラス図上の位置       | 主な依存先                          |
| ---------------- | -------------- | ------------------------------ |
| `BookController` | Controller（左上） | InputBoundary（UseCase層）        |
| `BookPresenter`  | Presenter（左下）  | OutputBoundary（UseCase層）       |
| `BookGateway`    | DataAccess（右下） | Repository Interface（UseCase層） |

---

## ✳️ Controller：`interface/controller/book_controller.py`

```python
# --------------------------------------------------------------------
# [クラス図] Controller
# [同心円] Interface Adapter層（外界入力 → UseCase層）
# --------------------------------------------------------------------
from usecase.boundaries.register_book_boundary import RegisterBookInputBoundary
from usecase.dto import RegisterBookInput

class BookController:
    """
    Controllerの責務：
    - View層（またはAPIリクエスト）から入力を受け取る
    - UseCase（InputBoundary）に渡せる形に変換する
    - UseCaseInteractorを呼び出す
    """

    def __init__(self, usecase: RegisterBookInputBoundary):
        self.usecase = usecase

    def handle_request(self, title: str, author: str, isbn: str):
        """
        ViewやAPIから渡されたリクエストを、DTO(InputData)に変換してUseCaseへ渡す
        """
        inp = RegisterBookInput(title=title, author=author, isbn=isbn)
        self.usecase.execute(inp)
```

> 💡 Controllerは「翻訳係」。
> View → UseCase間で、**リクエスト形式をDTO形式に変換**するだけの層です。
> UseCaseの詳細やPresenterは知らず、InputBoundaryの抽象にのみ依存します。

---

## ✳️ Presenter：`interface/presenter/book_presenter.py`

```python
# --------------------------------------------------------------------
# [クラス図] Presenter
# [同心円] Interface Adapter層（UseCase出力 → View層）
# --------------------------------------------------------------------
from usecase.boundaries.register_book_boundary import RegisterBookOutputBoundary
from usecase.dto import RegisterBookOutput

class BookPresenter(RegisterBookOutputBoundary):
    """
    Presenterの責務：
    - UseCaseから受け取った出力（OutputData）を
      ViewやAPIレスポンスで使いやすい形式に変換する
    """

    def __init__(self):
        self.view_model = None  # View層へ渡すための中間データ

    def present(self, out: RegisterBookOutput) -> None:
        """
        UseCase → Presenter間の契約（OutputBoundary）に基づく処理
        """
        self.view_model = {
            "id": out.id,
            "title": out.title,
            "author": out.author,
            "isbn": out.isbn,
            "created_at": out.created_at.strftime("%Y-%m-%d %H:%M:%S"),
        }

    def get_view_model(self):
        """
        View層やAPI層が参照できるようにViewModelを返す
        """
        return self.view_model
```

> 💡 Presenterは**整形係**。
> UseCaseが返す生データを、**View層に適した形式（ViewModel）**に変換します。

---

## ✳️ Gateway：`interface/gateway/book_gateway.py`

```python
# --------------------------------------------------------------------
# [クラス図] Gateway（Repository実装）
# [同心円] Interface Adapter層（DBアクセスの抽象実装）
# --------------------------------------------------------------------
from typing import List, Optional
from datetime import datetime
from domain.book import Book
from usecase.contracts.book_repository import BookRepository

class BookGateway(BookRepository):
    """
    Gatewayの責務：
    - UseCaseのRepository契約（Interface）を実装する
    - DBやファイルとやり取りする（今回は簡易的にメモリ内で模倣）
    """

    def __init__(self):
        self._books: List[Book] = []

    def next_id(self) -> int:
        return len(self._books) + 1

    def save(self, book: Book) -> Book:
        self._books.append(book)
        return book

    def find_by_id(self, book_id: int) -> Optional[Book]:
        return next((b for b in self._books if b.id == book_id), None)

    def find_by_isbn(self, isbn: str) -> Optional[Book]:
        return next((b for b in self._books if b.isbn == isbn), None)

    def search(self, keyword: str, limit: int = 50) -> List[Book]:
        return [b for b in self._books if keyword.lower() in b.title.lower()][:limit]
```

> 💡 Gatewayは**データアクセスの翻訳係**。
> UseCase層の「Repository契約」を実装し、
> 外部データ（DB・ファイル）を扱う責務を持ちます。
>
> 実際のDB接続は次章（Infrastructure層）で行います。

---

## 🧪 動作確認（Controller→UseCase→Presenter）

```python
from usecase.register_book import RegisterBook
from interface.controller.book_controller import BookController
from interface.presenter.book_presenter import BookPresenter
from interface.gateway.book_gateway import BookGateway

# 依存性注入（内向きの依存を構築）
repo = BookGateway()
usecase = RegisterBook(repo)
presenter = BookPresenter()
usecase.set_output_boundary(presenter)
controller = BookController(usecase)

# Controllerを通して実行
controller.handle_request("クリーンアーキテクチャ実践", "おはらやすのり", "978-1111")

print(presenter.get_view_model())
```

出力例：

```python
{
    'id': 1,
    'title': 'クリーンアーキテクチャ実践',
    'author': 'おはらやすのり',
    'isbn': '978-1111',
    'created_at': '2025-10-18 12:34:56'
}
```

---

## 🧠 依存と制御の流れ（図で確認）

![クリーンアーキテクチャ・依存と制御](../クリーンアーキテクチャ・依存と制御.png)

| 層          | 依存方向                        | 制御方向             |
| ---------- | --------------------------- | ---------------- |
| Controller | → InputBoundary             | ← ユーザー入力から制御     |
| UseCase    | → Repository／OutputBoundary | ← Controllerから制御 |
| Presenter  | → OutputData                | ← UseCaseから制御    |
| Gateway    | → Entity／DB                 | ← UseCaseから制御    |

> ✅ **依存は内へ、制御は外へ**。
> コードの中で、**矢印が「内向き依存・外向き制御」**になっていることを確認しましょう。

---

## 📘 この章の理解ポイント

| 観点             | 内容                                       |
| -------------- | ---------------------------------------- |
| **Controller** | 外部入力をUseCaseのInputBoundaryに変換する          |
| **Presenter**  | UseCaseの出力をView層に渡すため整形する                |
| **Gateway**    | Repository契約を実装し、DBアクセスを担当               |
| **依存方向**       | すべてのクラスはUseCase層（内側）に依存する                |
| **制御方向**       | Controller → UseCase → Presenter／Gateway |

---

## 📎 補足：Gateway層を明示的に作る理由（クラス図との対応）

同心円図やクラス図では、UseCase層とDB（Infrastructure）の間に
**Gateway** という中間層が描かれています。

Gatewayは、**Entityの世界とDBの世界をつなぐ翻訳層**です。
この層を作ることで、
DBの構造やアクセス手段が変わっても、UseCase層を変更せずにすみます。

> 💬 教材では「図に描かれているすべての層を実装して体感する」ことを目的としているため、
> 実務上は省略されることもあるGateway層をあえて明示的に作ります。
>
> これにより、**クラス図とコードの対応**が明確になります。


## ✅ **この章のゴール**

* Controller／Presenter／Gatewayの役割を理解する
* 「依存は内へ、制御は外へ」の流れを体感する
* 次章のInfrastructure層へスムーズに進める状態にする

## 🔄 次のステップ：Infrastructure層へ

次章では、**Frameworks & Drivers（Infrastructure層）**を実装します。

ここでは：

* Gatewayで定義した抽象操作を、実際のSQLiteデータベースで実装
* データの保存・読み出しを具体化
* FastAPIなどの外部フレームワークを統合

していきます。


