# 04 Input Boundary

# 🚪 Input Boundary
### `core/usecase/boundary/input_boundary.py`

## 🎯 このファイルの役割

`InputBoundary` は、**Controller（入力を受け付ける側）と Use Case（業務ロジックを実行する側）の間に置かれる「入り口の契約（インターフェース）」**です。

* Controller はユーザーの操作（「この本を貸したい」など）を受け取ります。
* しかし Controller は、Use Case の「中身の実装」を知りません。
* Controller はあくまで 「この形式のデータを渡すので、処理してください」という**決められた呼び方**だけを知っていればよい。

その「決められた呼び方」を定義するのが `InputBoundary` です。

言い換えると、**InputBoundary は「Use Case を呼び出すためのドアの形」を決めている**存在です。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## 📁 ファイルの配置

`InputBoundary` は Use Case 層の一部なので、`core/usecase/boundary/` に置きます。

```text
├─ core/
│   ├─ domain/
│   │   ├─ book.py                   # <E> Book Entity
│   │   ├─ member.py                 # <E> Member Entity
│   │   ├─ loan.py                   # <E> Loan Entity
│   │   └─ repository.py             # <I> Repositoryインターフェース
│   │
│   └─ usecase/
│       ├─ boundary/
│       │   ├─ input_boundary.py     # <I> ← 今説明しているファイル
│       │   ├─ output_boundary.py    # <I> Presenter側の契約
│       │   └─ dto.py                # <DS> InputData/OutputData など
│       │
│       └─ interactor/
│           └─ check_out_book.py     # <UC> 「本を貸し出す」ユースケース実装
│
├─ interface_adapters/
│   ├─ controllers/
│   │   └─ checkout_controller.py    # Controller（UIイベント→Use Case 呼び出し）
│   ├─ presenters/
│   │   └─ checkout_presenter.py     # Presenter（OutputBoundaryを実装）
│   └─ views/
│       └─ ...                       # CLI / GUI / Web などのView
```

ここでのポイントは依存方向です：

* Controller（外側）は、`core/usecase/boundary/input_boundary.py` を import して使います。
* Use Case 実装（`check_out_book.py`）は、この InputBoundary を「実装」します。
* Use Case の中から Controller を import することはありません。逆方向の依存は禁止です。

---

## 🔍 ソースコード（正式版）

```python
# --------------------------------------------------------------------
# File: core/usecase/boundary/input_boundary.py
# Layer: Use Case Boundary（UI → Use Case の入力契約）
#
# 目的:
#   - Controller が「ユースケースを起動するときに呼ぶ入口」の
#     形式（メソッド名・引数の型）を定義する。
#   - 具体的な処理内容は知らない／書かない。
#
# C言語の感覚に近い説明:
#   - ここは「関数プロトタイプ宣言」に近い。
#     (= どう呼ぶかだけ宣言して、中身は書かない .h 的な場所)
# --------------------------------------------------------------------

from abc import ABC, abstractmethod
from core.usecase.boundary.dto import CheckOutBookInputData


class CheckOutBookInputBoundary(ABC):
    """
    「本を貸し出す」というユースケースを呼び出すための
    入り口インターフェース（InputBoundary）。

    - 呼び出し側: Controller
      （例: checkout_controller.py がこれを使う）
    - 実装する側: Use Case（Interactor）
      （例: CheckOutBookUseCase がこれを実装する）

    Controllerは、このインターフェースだけを知っていれば
    貸出処理を起動できる。
    Use Case の中身・手順・依存性までは一切知らなくていい。
    """

    @abstractmethod
    def handle(self, input_data: CheckOutBookInputData) -> None:
        """
        図書館の「貸し出し処理」を実行してほしいときに
        Controller から呼ばれるメソッド。

        Parameters
        ----------
        input_data : CheckOutBookInputData
            Controller が組み立てた入力データ。
            例: book_id と member_id を含む。
        """
        raise NotImplementedError
```

### ここで登場する型：`CheckOutBookInputData`

* `CheckOutBookInputData` は `core/usecase/boundary/dto.py` で定義される「ただのデータの箱」です。
* Controller はユーザー入力（例: `book_id=12`, `member_id=7`）をこの形に詰めてから `handle()` に渡します。
* こうすることで、Controller → Use Case 間のやり取りが「公式なフォーマット」として固定されます。

---

## 🧭 この層の関係（依存の流れ）

| 役者                    | 何を知っている？                                       | どこに依存する？                                  |
| --------------------- | ---------------------------------------------- | ----------------------------------------- |
| Controller            | `CheckOutBookInputBoundary` を知っている             | `core/usecase/boundary/input_boundary.py` |
| Use Case (Interactor) | `CheckOutBookInputBoundary` を「実装」する（＝この契約を満たす） | `core/usecase/boundary/input_boundary.py` |
| Controller → Use Case | **OK（呼び出せる）**                                  |                                           |
| Use Case → Controller | **NG（Use Case は Controller を import してはならない）** |                                           |

これによって、

* Controller は Use Case の「入り口の形」だけを知る
* Use Case は Controller の存在すら知らない
  → 依存が一方向になる（内側は外側を知らない）

この一方向性こそがクリーンアーキテクチャの「依存性逆転の原則 (DIP)」です。

---

## 🧪 疑似テスト例（InputBoundaryのイメージ確認）

InputBoundary 自体はロジックを持たないので、ふつうは直接テストしません。
ただし「Controller はこの契約に従って Use Case を呼ぶ」という感覚は、こんなふうに確認できます。

```python
from core.usecase.boundary.input_boundary import CheckOutBookInputBoundary
from core.usecase.boundary.dto import CheckOutBookInputData

# テスト用のダミーUseCase
class FakeCheckOutBookUseCase(CheckOutBookInputBoundary):
    def __init__(self):
        self.last_called_with = None

    def handle(self, input_data: CheckOutBookInputData) -> None:
        # Controllerから受け取った引数を記録するだけ
        self.last_called_with = input_data


# --- Controller側からの呼び出しイメージ ---
fake_use_case = FakeCheckOutBookUseCase()

request_dto = CheckOutBookInputData(
    book_id=123,
    member_id=42,
)

fake_use_case.handle(request_dto)

assert fake_use_case.last_called_with.book_id == 123
assert fake_use_case.last_called_with.member_id == 42
print("✅ Controller は InputBoundary の契約どおりに呼び出せる")
```

ここで大事なのは：

* Controller は `FakeCheckOutBookUseCase` の中身を知らなくても `.handle(dto)` と呼べる
* Fake を本物（`CheckOutBookUseCase`）に置き換えても、Controller のコードは一切変えなくていい

→ つまり Controller のコードはフレームワークに依存しないばかりか、「ビジネスロジックの実装」にも依存していない。これはテスト容易性にも直結します。

---

## 🛡 鉄則

> 「Use Case は外の世界を知らない」
> そのための防波堤が InputBoundary。

* Controller は InputBoundary を介してしか Use Case を叩けない
  → Controller は「どう処理するか（手順）」を知らない
* Use Case は Controller の存在を知らない
  → UI 技術（CLI / GUI / Web / API）が変わっても Use Case は無傷
* 双方は **InputBoundary（契約）** のみを共有する
  → 依存は内向き。外側は内側を知るが、内側は外側を知らない

