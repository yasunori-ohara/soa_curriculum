# 03 <DS>and<I>

# Data Structure と Interface

# 🔌 Boundaries & Data Structures: レイヤー間の「契約書」

`UseCase`という「シェフ」の役割が定義できたので、次は、そのシェフが仕事をする上で必要となる「公式な書類（Data Structures）」**と**「契約書（Interfaces）」をまとめて解説します。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

## 🎯 このファイル群の役割

このページで作成するクラスやインターフェースは、クリーンアーキテクチャ図における**円と円の境界線上**に存在し、各レイヤーが「お互いにどういう形でコミュニケーションを取るか」というお約束事（契約）を定めます。

- **`Data Structures` (`<DS>`)**: レイヤー間を移動するだけの、ロジックを持たないシンプルなデータコンテナです。「データを運ぶための箱」と考えると分かりやすいでしょう。
- **`Interfaces`/`Boundaries` (`<I>`)**: `UseCase`と外部コンポーネント（PresenterやRepository）が互いに依存せず、疎結合になるための「仲介役」です。`UseCase`は具体的な実装クラスを知ることなく、この抽象的な「規格書」にのみ依存します。

この2つは、それぞれ「受け渡す**データ**の形」と「呼び出す**振る舞い**の形」を定義する、密接な役割を持っています。そのため、このカリキュラムでは1つのページでまとめて解説することで、レイヤー間の「契約」という概念を一度に理解できるようにしています。

この2つを定義することで、**依存性の方向**を内側（`Entities`や`UseCases`）へ向けるという、クリーンアーキテクチャの最重要ルール（依存性反転の原則）を実現します。

---

## 💻 ソースコードの詳細解説

それぞれの責務を持つファイルを作成しますが、ここではまとめて解説します。

### `application/data_structures.py` の中身

```python
# application/data_structures.py
from datetime import date
from typing import NamedTuple

# -----------------------------------------------------------------------------
# Data Structures (<DS>)
# - 同心円図の位置: 円と円の境界線上を流れるデータ
# -----------------------------------------------------------------------------

# --------------------------------------------------------------------
# - クラス図の位置: <DS>InputData
# --------------------------------------------------------------------
class CheckOutBookInputData(NamedTuple):
    """
    Controller -> UseCase へ渡すためのデータ構造。
    変更不可能な(immutable)データ構造として定義する。
    """
    book_id: int
    member_id: int

# --------------------------------------------------------------------
# - クラス図の位置: <DS>OutputData
# --------------------------------------------------------------------
class CheckOutBookOutputData(NamedTuple):
    """
    UseCase -> Presenter へ渡すためのデータ構造。
    こちらも変更不可能。
    """
    book_title: str
    member_name: str
    due_date: date

# --------------------------------------------------------------------
# - クラス図の位置: <DS>ViewModel
# --------------------------------------------------------------------
class BookViewModel(NamedTuple):
    """
    Presenter -> View へ渡すためのデータ構造（画面表示用）。
    UIが表示すべき全ての情報を持つ。
    """
    display_text: str

```

- **`NamedTuple`**: これらは「単なるデータの入れ物」であり、途中で値が書き換えられてしまう事故を防ぐため、変更不可能な`NamedTuple`で定義しています。
- **責務の分離**: `CheckOutBookOutputData`が持つ`due_date`は**日付オブジェクト**です。これを「2025年10月27日」のような人間が読みやすい**文字列**に変換するのは、`Presenter`の責務です。

### `application/boundaries.py` の中身

```python
# application/boundaries.py
from abc import ABC, abstractmethod
from typing import Optional

# 先ほど定義したDataStructuresと、内側のEntityをインポート
from .data_structures import CheckOutBookInputData, CheckOutBookOutputData
from domain.entities import Book, Member, Loan

# -----------------------------------------------------------------------------
# Boundaries / Interfaces (<I>)
# - 同心円図の位置: 円と円の境界線そのもの
# -----------------------------------------------------------------------------

# --------------------------------------------------------------------
# - クラス図の位置: <I>InputBoundary
# --------------------------------------------------------------------
class CheckOutBookInputBoundary(ABC):
    """
    UseCaseの「入力ポート」。
    - 実装するクラス: CheckOutBookUseCase
    - 呼び出すクラス: CheckOutBookController
    """
    @abstractmethod
    def handle(self, input_data: CheckOutBookInputData):
        raise NotImplementedError

# --------------------------------------------------------------------
# - クラス図の位置: <I>OutputBoundary
# --------------------------------------------------------------------
class CheckOutBookOutputBoundary(ABC):
    """
    UseCaseの「出力ポート」。
    - 実装するクラス: CheckOutBookPresenter
    - 呼び出すクラス: CheckOutBookUseCase
    """
    @abstractmethod
    def present(self, output_data: CheckOutBookOutputData):
        raise NotImplementedError

# --------------------------------------------------------------------
# - クラス図の位置: <I>DataAccessInterface
# --------------------------------------------------------------------
class BookDataAccessInterface(ABC):
    """Bookのデータ永続化を担当するクラスが実装すべきインターフェース。"""
    @abstractmethod
    def find_by_id(self, book_id: int) -> Optional[Book]:
        raise NotImplementedError

    @abstractmethod
    def save(self, book: Book) -> Book:
        raise NotImplementedError

class MemberDataAccessInterface(ABC):
    """Memberのデータ永続化を担当するクラスが実装すべきインターフェース。"""
    @abstractmethod
    def find_by_id(self, member_id: int) -> Optional[Member]:
        raise NotImplementedError

class LoanDataAccessInterface(ABC):
    """Loanのデータ永続化を担当するクラスが実装すべきインターフェース。"""
    @abstractmethod
    def save(self, loan: Loan) -> Loan:
        raise NotImplementedError

```

- **`ABC`, `@abstractmethod`**: Pythonの`abc`モジュールを使い、これらが具体的な実装を持たない**抽象クラス（インターフェース）であることを明示しています。これにより、これらのインターフェースを実装するクラスは、定義されたメソッドを必ず実装しなければならないというルールを強制**できます。
- **3つの`DataAccessInterface`**: このシステムでは「本」「会員」「貸出」という3種類のデータを永続化するため、それぞれの責務に対応したインターフェースを定義しています。

---

## 💡 これらのファイルにユニットテストは必要？

**原則として、これらのファイル自体に専用のユニットテストは不要です。**

なぜなら、これらはロジックを持たない「定義」や「契約書」だからです。テストすべき振る舞いがありません。

これらの`DataStructures`や`Interfaces`が正しく使われているかは、**それらを利用する側のクラス（`UseCase`や`Presenter`など）のユニットテスト**で間接的に検証されます。

---

## 🐍 PythonとC言語の比較（初心者の方へ）

- **Python (オブジェクト指向)**: `ABC`を使って抽象クラス（インターフェース）を明確に定義し、「このインターフェースを実装するクラスは、必ず`handle`メソッドを持たなければならない」という契約をコードで表現できます。
- **C言語 (手続き型)**: C言語にはクラスやインターフェースの概念がありません。似た役割を果たすのは**ヘッダーファイル (`.h`)** です。ヘッダーファイルに関数のシグネチャ（名前、引数、戻り値の型）を宣言することで、「このヘッダーをインクルードするソースファイルは、この関数を使える」という一種の**契約**を定義します。

---

## 🛡️ このファイル群の鉄則

このファイル群が実現する最も重要なルールは、クリーンアーキテクチャの根幹をなす「依存性反転の原則」です。

> 具象に依存するな。抽象に依存せよ。 (Depend on abstractions, not on concretions.)
> 
- `UseCase`は、具体的な`PostgresBookRepository`や`ConsolePresenter`のような実装（具象）**を一切知りません。ただ、`BookDataAccessInterface`や`CheckOutBookOutputBoundary`という**規格書（抽象）が存在することだけを知っています。
- これにより、将来データベースをPostgreSQLからMySQLに変更したり、UIをコンソールからWebアプリに変更したりしても、**`UseCase`のコードには一切変更が必要なくなります**。まさに「プラグ」を差し替えるように、コンポーネントを交換できるのです。

この原則を守ることが、変更に強く、テストが容易なソフトウェアの鍵となります。

はい、承知いたしました。
ページ末尾に自然に追加できるよう、いただいた2つの補足事項をQ&A形式にまとめます。

---

## ❓ Q&A

### 🤔 **Q. `Interface`、`Boundary`、`Contract`という言葉は何が違うのですか？**

**A.** これらは似た意味で使われますが、それぞれ少しずつニュアンスや視点が異なります。

- **`Interface` (インターフェース)**
    - **分類**: プログラミングの**道具** 🧰
    - **意味**: 「こういう名前のメソッドがあり、こういう引数を取る」という実装の**規格**を定義する言語機能です。Pythonでは`ABC`がこれにあたります。
- **`Boundary` (境界)**
    - **分類**: アーキテクチャの**概念** 🗺️
    - **意味**: `UseCase`と`Adapters`のように、レイヤー間を隔てる**境界そのもの**を指す言葉です。クリーンアーキテクチャにおける特定の役割を持つ`Interface`と言えます。
- **`Contract` (契約)**
    - **分類**: 設計上の**考え方** 📜
    - **意味**: 「この機能を使うには、このルールを守ってください」という、部品間の社会的な**約束事**です。`Interface`は`Contract`をコードで表現したものです。

**結論として、このカリキュラムでは「`Interface`という道具を使って、アーキテクチャ上の`Boundary`という役割を実現している」と理解してください。**

---

### 🤔 **Q. クラス図では`Input/OutputData`と`ViewModel`は違う場所にありますが、なぜ一緒に解説しているのですか？**

**A.** それは非常に良い質問で、アーキテクチャの重要なポイントを突いています。

**結論から言うと、アーキテクチャ上の「所有者」は異なりますが、コーディング上の「パターン」が同じだからです。**

- **所有者の違い（なぜ場所が違うか）**
    - **`InputData` / `OutputData`**: これらは**`UseCase`層が所有**します。`UseCase`が「私と会話する時はこのデータ形式でお願いします」とルールを定義しているため、`UseCase`の近くにいます。
    - **`ViewModel`**: これは**`Adapters`層（Presenter）が所有**します。`Presenter`が`View`（画面）の都合に合わせて「画面にはこの形式で渡します」と定義しているため、外側にいます。
- **パターンの共通点（なぜ一緒に解説するか）**
    - 3つとも**「ロジックを持たない、単なるデータの入れ物」**というコーディング上の役割と実装パターンは全く同じです。

そのため、このカリキュラムでは学習効率を優先し、「ロジックを持たない単純なデータ構造」という同じ仲間としてまとめて解説しています。