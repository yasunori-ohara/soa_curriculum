# 11 main

# 🚀 Composition Root : `main.py`

いよいよ最後のファイル、これまで作ってきた全ての部品を組み立てて命を吹き込む`main.py`の解説です。

## 🎯 このファイルの役割

`main.py`は、これまで作ってきた全ての部品（クラス）を組み立てて一つのアプリケーションとして完成させ、起動スイッチを入れる「組立工場」の役割を果たします。

クリーンアーキテクチャでは、この場所を**Composition Root**と呼びます。ここは、アプリケーション全体で**唯一**、具体的な実装クラス（`CheckOutBookPresenter`, `InMemoryDataAccess`など）が互いを直接知ることを許された場所です。

ここで全ての部品を配線（**依存性の注入 / Dependency Injection**）することで、他の全てのクラス（`UseCase`や`Controller`など）は、お互いの具体的な実装を知らない「クリーン」な状態を保つことができるのです。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

## ✅ このファイルにあるべき処理

⭕️ **含めるべき処理の例:**

- 全ての具体的な実装クラスのインスタンス化（オブジェクト生成）。
- あるクラスのインスタンスを、別のクラスのコンストラクタ（`__init__`）に渡すことによる、依存性の注入（DI）。
- アプリケーションの最初の処理（今回は`view.run()`）を呼び出すこと。

❌ **含めてはいけない処理の例:**

- ビジネスロジック、データ変換ロジック、UIの描画ロジックなど。このファイルの責務は、あくまで**組み立て**に専念します。

## 💻 ソースコードの詳細解説

```python
# main.py

# -----------------------------------------------------------------------------
# main.py
# - クラス図の位置: (図の外) 全ての部品を組み立てる責任を持つ
# - 同心円図の位置: (円の外) アプリケーションのエントリーポイント
# -----------------------------------------------------------------------------

# --- Adapters層の具体的な実装クラス ---
from adapters.presenter import CheckOutBookPresenter
from adapters.controller import CheckOutBookController
from adapters.view import ConsoleView
from adapters.data_access import (
    InMemoryBookDataAccess, InMemoryMemberDataAccess, InMemoryLoanDataAccess
)

# --- UseCase層の具体的な実装クラス ---
from application.use_cases.check_out_book import CheckOutBookUseCase

# --- 必要なデータ構造 ---
from application.data_structures import BookViewModel

def main():
    """アプリケーションのすべての部品を組み立て、起動する"""
    print("--- 1. アプリケーションの部品を組み立てます ---")

    # [ViewModel] を生成
    # ViewとPresenterの間で共有される、画面の状態を持つオブジェクト
    view_model = BookViewModel(display_text="")

    # [Presenter] を生成
    # 依存するViewModelを注入（DI）
    presenter = CheckOutBookPresenter(view_model)

    # [Data Access] を生成
    # 本、会員、貸出記録、それぞれのデータアクセスオブジェクトを作成
    book_data_access = InMemoryBookDataAccess()
    member_data_access = InMemoryMemberDataAccess()
    loan_data_access = InMemoryLoanDataAccess() # LoanのDataAccessも生成

    # [Use Case] を生成
    # 依存するPresenterと3種類のDataAccessを注入（DI）
    use_case = CheckOutBookUseCase(
        presenter=presenter,
        book_repository=book_data_access,
        member_repository=member_data_access,
        loan_repository=loan_data_access # Loanの依存も注入
    )

    # [Controller] を生成
    # 依存するUse Caseを注入（DI）
    controller = CheckOutBookController(use_case)

    # [View] を生成
    # 依存するControllerと、共有するViewModelを注入（DI）
    view = ConsoleView(controller, view_model)

    print("--- 2. 組み立て完了、アプリケーションを実行します ---\\n")

    # 全ての配線が完了したViewインスタンスを使って、アプリケーションの実行を開始
    view.run()

if __name__ == "__main__":
    main()

```

`main`関数の中では、一切のビジネスロジックが実行されていない点に注目してください。行われているのは、依存関係の連鎖を解決するためのオブジェクト生成と注入だけです。

1. まず、依存される側である`ViewModel`, `Presenter`, `DataAccess`を生成します。
2. 次に、それらを必要とする`UseCase`, `Controller`, `View`を、コンストラクタに必要な部品を渡しながら生成していきます。
3. 全ての部品が正しく接続された`view`オブジェクトが完成したら、最後に`view.run()`を呼び出してアプリケーションを起動します。

## 💡 ユニットテストは不要、だが...

**`main.py`自体には、ロジックがないためユニットテストは書きません。**

このファイルの正しさは、**アプリケーション全体がエラーなく起動し、意図通りに動作すること**で証明されます。このような、複数のコンポーネントを結合して行うテストを「インテグレーションテスト（結合テスト）」と呼びます。まさに`main.py`を実行することが、最もシンプルなインテグレーションテストと言えます。

## 🐍 PythonとC言語の比較（初心者の方へ）

- **Python (オブジェクト指向)**: `main`関数内で各クラスの**インスタンスを動的に生成**し、それを他のクラスのコンストラクタに渡すことで、柔軟に依存関係を組み立てています。
- **C言語 (手続き型)**: `main`関数がすべての中心となり、各機能に対応する関数を直接呼び出す形が基本です。依存性の注入のようなことを行うには、関数の引数として**関数ポインタ**を渡したり、グローバルな構造体に必要な情報をすべて詰め込んで初期化したり、といった工夫が必要になり、コードが複雑になりがちです。

## 🛡️ このファイルの鉄則

このファイルは、他の全てのクラスをクリーンに保つための、重要な役割を担います。

> すべてを知り、すべてを組み立て、そして仕事は最初に任せよ。 (Know everything, assemble everything, then delegate the first task.)
> 
- この`main.py`は、アプリケーションの具体的な実装クラスをすべて知っている、唯一の「汚れる」場所です。
- この場所が**依存関係を一手に引き受ける**ことで、他のすべてのクラスはインターフェースにのみ依存する「クリーン」な状態を保つことができます。
- 組み立てが終わったら、最初のきっかけ（`view.run()`）を与えるだけで、あとは他のクラスに仕事を任せます。