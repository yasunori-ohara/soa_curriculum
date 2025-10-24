# 05 Controller

# 🛂 Controller : `adapters/controller.py`

UI層の「交通整理員」、`CheckOutBookController`について解説します。

## 🎯 このクラスの役割

**`CheckOutBookController`は、`View`からのユーザー入力を受け取り、それを`UseCase`が理解できる形式（`InputData`）に整えて「伝達」する交通整理員**です。

`View`はユーザーが何をしたか（例：「貸出」ボタンが押され、IDとして「1」と「2」が入力された）を知っていますが、それがビジネス的に**何を意味するのか**は知りません。`Controller`は、そのUIイベントを、「会員ID:1の人に、書籍ID:2を貸し出す」というビジネス上のリクエストに変換し、適切な`UseCase`の窓口に渡すことが唯一の責務です。

`View`という「受付」と、`UseCase`という「専門部署」の間に立つ、薄いながらも重要な**仲介役**です。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

## ✅ このクラスにあるべき処理

⭕️ **含めるべき処理の例:**

- `View`からのイベント（メソッド呼び出し）を受け取ること。
- `View`から渡された生データ（例：IDの数値など）を、`InputData`オブジェクトに変換すること。
- `InputBoundary`インターフェースを通じて、適切な`UseCase`を呼び出すこと。

❌ **含めてはいけない処理の例:**

- ビジネスロジックそのもの（`UseCase`の責務）。
- 画面表示の準備（`Presenter`の責務）。
- 画面を直接描画する処理（`View`の責務）。
- データベースに関する知識。

## 💻 ソースコードの詳細解説

```python
# adapters/controller.py

# 内側の世界の「境界」と「データ構造」にのみ依存する
from application.boundaries import CheckOutBookInputBoundary
from application.data_structures import CheckOutBookInputData

# -----------------------------------------------------------------------------
# Controller
# - クラス図の位置: Controller
# - 同心円図の位置: Adapters (外側の円)
# -----------------------------------------------------------------------------
class CheckOutBookController:
    """
    ControllerはUIイベントとビジネスロジックを繋ぐ、薄い層。
    ロジックを持たず、交通整理に徹する。
    """
    def __init__(self, use_case: CheckOutBookInputBoundary):
        """
        [依存先の定義]
        呼び出し先であるUseCaseの具体的な実装クラスではなく、
        その「窓口」であるCheckOutBookInputBoundaryインターフェースにのみ依存する。
        （依存性反転の原則）
        """
        self._use_case = use_case

    def check_out(self, book_id: int, member_id: int):
        """
        [伝達処理]
        Viewからの生の入力(ID)を、InputDataという公式な形式に変換し、
        UseCaseの実行を依頼する。
        """
        # 1. Viewからの生データを、UseCaseが理解できるInputDataオブジェクトに変換する
        input_data = CheckOutBookInputData(book_id=book_id, member_id=member_id)

        # 2. 変換したデータを添えて、インターフェースを通じてUseCaseの実行を依頼する
        self._use_case.handle(input_data)

```

- `__init__`メソッドは、呼び出し先である`UseCase`の**インターフェース**を受け取ります。これにより、`Controller`は`UseCase`の内部実装がどうなっているかを一切知る必要がなくなります。
- `check_out`メソッドがこのクラスの仕事のすべてです。
    1. `View`から渡されたただの数値`book_id`と`member_id`を、ビジネス層の公式な書類である`CheckOutBookInputData`に変換（箱詰め）します。
    2. その`InputData`を引数として、`UseCase`の`handle`メソッドを呼び出します。

これだけです。`Controller`自身は一切「考えず」、ただ情報を右から左へ受け流しているだけ、という点が重要です。

## 💡 ユニットテストでControllerの正しさを証明する

Controllerのテストでは、**「交通整理」が正しく行われているか**を検証します。`View`からメソッドが呼ばれたときに、`UseCase`に**正しい形式のデータを渡して呼び出しているか**だけを確認します。

```python
# tests/adapters/test_controller.py の例

def test_controllerは入力をInputDataに変換してUseCaseを呼び出す():
    # 1. Arrange (準備): 呼び出し先であるUseCaseをモック（偽物）にする
    #    MockUseCaseは、呼び出されたかどうかを記録するスパイの役割を持つ
    mock_use_case = MockCheckOutBookUseCase()

    # 2. Act (実行): テスト対象のControllerを作成し、実行する
    controller = CheckOutBookController(use_case=mock_use_case)
    controller.check_out(book_id=10, member_id=20)

    # 3. Assert (検証): モックのUseCaseが期待通りに呼び出されたか確認
    # 意図: 「check_out(10, 20)と呼ばれたら、ちゃんとhandleメソッドを
    #       CheckOutBookInputData(book_id=10, member_id=20)という引数で
    #       呼び出しているか？」をテスト
    assert mock_use_case.handle_was_called is True
    assert mock_use_case.received_input_data.book_id == 10
    assert mock_use_case.received_input_data.member_id == 20

```

- **テストの意図**: このテストは、`Controller`が持つ唯一のロジック（データ変換と伝達）に焦点を当てています。`UseCase`のビジネスロジックは一切実行せず、`Controller`の責務である「正しい形式でバトンを渡す」ことだけを独立して検証できます。

## 🐍 PythonとC言語の比較（初心者の方へ）

- **Python (オブジェクト指向)**: `CheckOutBookController`という**クラス**に、UIイベントと`UseCase`を仲介する責務をカプセル化します。依存性の注入（`__init__`で`UseCase`を受け取る）により、`UseCase`の実装を簡単に差し替えることができます。
- **C言語 (手続き型)**: UIライブラリのコールバック関数が`Controller`の役割を担うことが多いでしょう。例えば、ボタンがクリックされたときに呼ばれる`on_checkout_button_clicked`関数の中で、入力フィールドから値を取得し、ビジネスロジックを実行する別の関数を直接呼び出す形になります。これではUIとビジネスロジックが密結合になりがちです。

## 🛡️ このクラスの鉄則

このクラスは、交通整理員として、決められたルールに従うだけです。

> 入力を整理し、然るべき場所に渡せ。 (Organize the input and pass it to the right place.)
> 
- `Controller`はビジネスロジックを持ちません。UIとビジネスロジックの間に立ち、両者が直接会話しないようにする**緩衝材**の役割に徹します。
- この層が薄く保たれているおかげで、`UseCase`は`View`を知らず、`View`は`UseCase`を知らない、というクリーンな分離が実現できます。

## ❓ Q\&A

### **Q. 書籍のクラス図では`Controller`と`View`がつながっていませんが、この実装でつながっているのはなぜですか？**

**A.** それは、「依存性のルール」**と**「コントロールの流れ」という2つの異なる視点からシステムを見ているためです。これらは矛盾しません。

- **書籍のクラス図（依存性のルール）**
    - あの図の主目的は、**依存関係が必ず内側に向かう**というルールを示すことです。
    - `View`と`Controller`は同じ`Adapters`層にいるため、両者間の連携は、アーキテクチャの核心ルールを説明する上では副次的な情報なので、意図的に省略されています。
- **この実装（コントロールの流れ）**
    - アプリケーションが実際に動作するためには、ユーザー操作を起点とする制御の流れ（コントロールフロー）が必要です。
    - `ユーザー操作 → View → Controller → UseCase` という流れは、このコントロールフローを実装したものです。
    - この`View → Controller`という依存は、**同じレイヤー内での依存**であり、内側の`UseCase`には向かっていないため、クリーンアーキテクチャのルールには違反していません。

**結論として、書籍の図は「大通りの交通ルール（レイヤー間の依存）」を、私たちの実装は「路地も含めた詳細な地図（実際の動作に必要な連携）」を表していると理解してください。**