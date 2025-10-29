# OOP-05 : CAの「型」による組立 (3) - アダプター層とDI

`OOP-04`までに、CAの主要なレイヤー（ドメイン、ユースケース、インフラ）を実装しました。しかし、これらはまだ「部品」としてバラバラな状態です。

  * `ExecutePayrollUseCase`（ユースケース）が依存する`IEmployeeRepository`（抽象）に、どの「実装（`InMemory...`版）」を渡すのか、誰も決めていません。
  * `ExecutePayrollUseCase`を「誰が」呼び出すのか（入力の受付役）が、`main.py`と密結合しています。

この章では、これらの部品を組み立て、システム全体を完成させます。

## 🎯 この章のゴール

  * **アダプター層 (Interface Adapters)** の役割（＝データの変換・仲介）を理解する。
  * **コントローラー**（入力アダプター）を実装する。
  * `main.py`（起動層）で、**依存性の注入 (DI)** を行い、システム全体を組み立てる。
  * CAの「型」にはめたアーキテクチャで、`OOP-01`のシナリオが実行できることを確認する。

-----

## 1\. アダプター層 (Interface Adapters) の実装

「アダプター層」は、CAの内側（ユースケース）と外側（`main.py`やWeb、コンソール）の間で、**データ形式を変換・仲介する**責務を持ちます。

今回は、`main.py`（外部）からの「単純な入力」を、`UseCase`（内側）に渡す「**コントローラー**」をアダプターとして定義します。

> **`adapters.py` (新設)**

```python
# 「境界（インターフェース）」をインポート
from application_boundaries import IExecutePayrollUseCase
from domain import Employee # 従業員追加(add_employee)のために domain もインポート

class PayrollController:
    """
    【入力アダプター】コントローラー。
    外部（今回は main.py）からの「リクエスト」を、
    ユースケース層（IExecutePayrollUseCase）の
    適切なメソッド呼び出しに「変換」する。
    """
    def __init__(self, use_case: IExecutePayrollUseCase):
        # 【CAの依存ルール】具象(ExecutePayrollUseCase)ではなく、抽象に依存
        self._use_case = use_case

    def execute_payment(self):
        """
        外部からの「給与支払実行」リクエストをユースケースに仲介する
        """
        self._use_case.execute()

    # (補足)
    # 本来、従業員の「追加」も独立したユースケース
    # (例: IAddEmployeeUseCase) にすべきだが、
    # 今回は簡略化のため、Controllerがリポジトリを直接操作する
    # ショートカットを実装する。（※あまり良い設計ではない）
    
    # --- あまり良くない例（ユースケースをバイパス） ---
    # def __init__(self, 
    #              use_case: IExecutePayrollUseCase,
    #              employee_repo: IEmployeeRepository): # リポジトリも知ってしまう
    #     self._use_case = use_case
    #     self._employee_repo = employee_repo

    # def add_employee(self, employee: Employee):
    #     self._employee_repo.save(employee)
    
    # (今回は OOP-05 では Controller を導入せず、
    #  main.py で直接 UseCase を呼び出す構成にします。
    #  その方がDI層の役割が明確になるため)
```

**（設計の軌道修正）**
`OOP-04`までの構成（`UseCase`が`execute`のみ）では、`add_employee`の処理を`main.py`でどう扱うかが曖昧になります。

`OOP-01`の`main.py`は、`payroll_system.add_employee()`も`payroll_system.execute_payment()`も呼び出していました。

CAの「型」に厳密にはめるため、`main.py`の役割を「**DI（組み立て）**」と「**外部からのトリガー（実行）**」に限定します。
`UseCase`の呼び出しは`main.py`から直接行い、`adapters.py`は（`main.py`がその役割を兼ねるため）この章では**導入しない**こととします。

-----

## 1\. (改) 起動層 (main.py) での依存性の注入 (DI)

`main.py`は、CAの最も外側の層（あるいは層の外）です。
**唯一の責務は、すべての「具象クラス」をインスタンス化し、依存関係に従ってそれらを「組み立てる（注入する）」ことです。**

`main.py`だけが、`ExecutePayrollUseCase`（ユースケース）と`InMemoryEmployeeRepository`（インフラ）の**両方を知っている**、唯一の場所となります。

> **`main.py` (OOP-01からの全面改訂)**

```python
# --- インポートするものが劇的に変わる ---

# 1. ドメイン層（具象エンティティ）
from domain import FullTimeEmployee, PartTimeEmployee

# 2. アプリケーション層（具象ユースケース）
from application import ExecutePayrollUseCase

# 3. インフラ層（具象リポジトリ）
from infrastructure import InMemoryEmployeeRepository, InMemoryPayLogRepository

# (注意)
# main.py は「起動層」であるため、唯一
# *すべて*の具象クラスを知っている（インポートしている）ファイルとなる。
# application_boundaries.py (抽象) はインポートしない。

if __name__ == "__main__":
    
    # --- 1. 依存性の注入 (DI) ---
    # CAの「外側」から「内側」へと依存関係を組み立てていく
    
    # a. インフラ層（最も外側）の具象インスタンスを生成
    employee_repo = InMemoryEmployeeRepository()
    log_repo = InMemoryPayLogRepository()
    
    # b. アプリケーション層（ユースケース）を生成
    #    -> コンストラクタ（__init__）を経由して、
    #       具象インスタンスを「注入」する
    #    -> UseCaseは、自分が InMemory を使っているとは知らない
    payroll_use_case = ExecutePayrollUseCase(
        employee_repo=employee_repo,
        pay_log_repo=log_repo
    )

    # --- 2. 初期データのセットアップ ---
    # (OOP-01の main.py が担っていた処理)
    # 本来はユースケース(AddEmployeeUseCase)経由で実行すべきだが、
    # 今回は簡略化のため、インフラ(リポジトリ)を直接操作する
    employee_repo.save(
        FullTimeEmployee("e-001", "田中 太郎", 300000)
    )
    employee_repo.save(
        PartTimeEmployee("e-002", "鈴木 花子", 1500, 80) # 120,000円
    )
    employee_repo.save(
        FullTimeEmployee("e-003", "佐藤 次郎", 400000)
    )

    # --- 3. アプリケーションの実行 ---
    # OOP-01 と同じシナリオを「ユースケース経由」で実行する
    
    # 3-a. 1回目の給与支払 (合計 820,000円)
    payroll_use_case.execute()
    
    # 3-b. 鈴木さんの労働時間が変わる (上書き保存)
    employee_repo.save(
        PartTimeEmployee("e-002", "鈴木 花子", 1500, 100) # 150,000円
    )
    
    # 3-c. 2回目の給与支払 (合計 850,000円)
    payroll_use_case.execute()
    
    print("\n--- 最終支払ログ ---")
    # (UseCase にログ取得機能がないため、リポジトリを直接参照)
    print(log_repo.get_all())
```

-----

## 4\. リファクタリングの完了

これで、`OOP-01`の`PayrollSystem`（神クラス）を、クリーンアーキテクチャ（CA）の「型」に従ってリファクタリングする作業が完了しました。

`OOP-01`のコードは、`system.py`という単一のファイルにロジックが集中していましたが、リファクタリング後のコードは、以下の4つの明確な「関心事（レイヤー）」に分離されました。

  * **`domain.py`**:
      * **関心事**: ビジネスのルール（給与の計算方法）
  * **`application_boundaries.py`**:
      * **関心事**: レイヤー間の「契約（インターフェース）」
  * **`application.py`**:
      * **関心事**: ビジネスの手順（いつ、何を呼び出すか）
  * **`infrastructure.py`**:
      * **関心事**: データの保存方法（どうやって保存するか）
  * **`main.py`**:
      * **関心事**: すべての「部品（具象クラス）」を組み立てる（DI）

`main.py`を除くすべてのファイル（`application.py`など）は、`InMemory...`といった**具体的な実装**を一切知らず、`I...Repository`という\*\*抽象的なインターフェース（境界）\*\*にのみ依存する、非常に柔軟な構造になりました。

-----

## 🚧 次の課題（謎解き）

さて、**私たちのリファクタリングは完了しました**。
CAの「型」に従って「関心の分離」と「依存性のルール」を守ったことで、`OOP-01`よりもはるかに見通しが良く、変更に強い（例：DBへの変更が容易な）コードが完成しました。

---

ところで、`OOP-02`のSOLID原則評価のことを覚えているでしょうか？
あの時、`OOP-01`のコードは\*\*SRP（単一責任の原則）**と**DIP（依存性逆転の原則）\*\*に甚だしく違反している、と評価しました。

**このCAリファクタリング（`OOP-03`〜`OOP-05`）は、これらの違反を解決できたのでしょうか？**

次の章（`OOP-06`）で、このリファクタリング結果を、もう一度**SOLID原則**という「ものさし」で評価し直し、その「答え合わせ」をしてみましょう。