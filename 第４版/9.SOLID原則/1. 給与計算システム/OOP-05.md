# OOP-05 : CAの「型」による組立 (3) - アダプター層とDI

`OOP-04`までに、クリーンアーキテクチャ（CA）の「型」に従い、`OOP-01`の`PayrollSystem`（神クラス）を、

  * **`domain.py`** (ドメイン層)
  * **`application_boundaries.py`** (境界インターフェース)
  * **`application.py`** (ユースケース層)
  * **`infrastructure.py`** (インフラ層)
    という、関心事が分離された「部品」に分解しました。

しかし、これらはまだ「部品」としてバラバラな状態です。

  * `ExecutePayrollUseCase`（ユースケース）が依存する`IEmployeeRepository`（抽象）に、どの「実装（`InMemory...`版）」を渡すのか、誰も決めていません。
  * `ExecutePayrollUseCase`を「誰が」呼び出すのか（外部からの入力を受け付ける役）が決まっていません。

この章では、これらの部品を**組み立て**、外部からの入力を受け付ける**アダプター層**を導入し、システム全体を完成させます。

## 🎯 この章のゴール

  * **アダプター層 (Interface Adapters)** の役割（＝データの変換・仲介）を理解する。
  * **コントローラー**（入力アダプター）を実装する。
  * `main.py`（起動層）で、**依存性の注入 (DI)** を行い、システム全体を組み立てる「組立工場」の役割に専念させる。
  * CAの「型」にはめたアーキテクチャで、`OOP-01`のシナリオが実行できることを確認する。

-----

## 🛠️ 1. アダプター層 (Interface Adapters) の実装

「アダプター層」は、CAの内側（ユースケース）と外側（`main.py`やWeb、コンソール）の間で、**データ形式を変換・仲介する**責務を持ちます。

今回は、`main.py`（外部）からの「実行指示」を、`UseCase`（内側）へのメソッド呼び出しに変換する「**コントローラー**」をアダプターとして定義します。

> **`adapters.py` (新設)**

```python
# 「境界（インターフェース）」をインポート
from application_boundaries import IExecutePayrollUseCase, IEmployeeRepository
from domain import Employee # 従業員追加のために domain もインポート

class PayrollController:
    """
    【入力アダプター】コントローラー。
    外部（今回は main.py）からの「リクエスト」を、
    ユースケース層（IExecutePayrollUseCase）や、
    （※今回は簡略化のため）リポジトリ層の
    適切なメソッド呼び出しに「変換・仲介」する。
    """
    def __init__(self,
                 use_case: IExecutePayrollUseCase,
                 employee_repo: IEmployeeRepository): # 従業員追加のためリポジトリも受け取る

        # 【CAの依存ルール】具象ではなく、抽象インターフェースに依存
        self._use_case = use_case
        self._employee_repo = employee_repo # (※)

    def add_employee(self, employee: Employee):
        """
        外部からの「従業員追加」リクエストをリポジトリに仲介する。
        (※本来は AddEmployeeUseCase を用意し、それを呼び出すべき)
        """
        print(f"従業員追加(Controller): {employee.name}")
        self._employee_repo.save(employee)

    def execute_payment(self):
        """
        外部からの「給与支払実行」リクエストをユースケースに仲介する。
        """
        print(f"支払実行(Controller): ユースケースを呼び出します。")
        self._use_case.execute()

    # (※) 補足:
    # Controller が Repository に直接依存するのは、厳密には
    # CAのショートカット（ユースケースバイパス）です。
    # より丁寧にするなら IAddEmployeeUseCase を作り、
    # Controller は UseCase にのみ依存するようにします。
    # 今回は教材の簡略化のため、この構成を採用します。
```

-----

## 🛠️ 2. 起動層 (main.py) での依存性の注入 (DI)

`main.py`は、CAの最も外側の層（あるいは層の外）です。
**責務は、すべての「具象クラス」をインスタンス化し、依存関係に従ってそれらを「組み立てる（注入する）」ことに専念します。**
アプリケーションの実行（ユースケースの呼び出し）は、アダプター（`PayrollController`）に任せます。

> **`main.py` (OOP-04からの改訂)**

```python
# --- インポート ---
# main.pyだけが、すべての「具象クラス」を知っている

# 1. ドメイン層（具象エンティティ）
from domain import FullTimeEmployee, PartTimeEmployee

# 2. アプリケーション層（具象ユースケース）
from application import ExecutePayrollUseCase

# 3. インフラ層（具象リポジトリ）
from infrastructure import InMemoryEmployeeRepository, InMemoryPayLogRepository

# 4. アダプター層（具象コントローラー）
from adapters import PayrollController

if __name__ == "__main__":

    # --- 1. 依存性の注入 (DI) ---
    # CAの「外側」から「内側」へと依存関係を組み立てていく
    # ここが「組立工場」

    # a. インフラ層（最も外側）の具象インスタンスを生成
    employee_repo = InMemoryEmployeeRepository()
    log_repo = InMemoryPayLogRepository()

    # b. アプリケーション層（ユースケース）を生成し、具象リポジトリを注入
    payroll_use_case = ExecutePayrollUseCase(
        employee_repo=employee_repo,
        pay_log_repo=log_repo
    )

    # c. アダプター層（コントローラー）を生成し、具象ユースケースとリポジトリを注入
    controller = PayrollController(
        use_case=payroll_use_case,
        employee_repo=employee_repo # Controller が従業員追加も担当するため
    )

    # --- 2. 初期データのセットアップ ---
    # Controller を経由してデータを登録
    controller.add_employee(
        FullTimeEmployee("e-001", "田中 太郎", 300000)
    )
    controller.add_employee(
        PartTimeEmployee("e-002", "鈴木 花子", 1500, 80) # 120,000円
    )
    controller.add_employee(
        FullTimeEmployee("e-003", "佐藤 次郎", 400000)
    )

    # --- 3. アプリケーションの実行 ---
    # OOP-01 と同じシナリオを、「コントローラー経由」で実行する

    # 3-a. 1回目の給与支払 (合計 820,000円)
    # main.py は Controller のメソッドを呼び出すだけ
    controller.execute_payment()

    # 3-b. 鈴木さんの労働時間が変わる
    controller.add_employee(
        PartTimeEmployee("e-002", "鈴木 花子", 1500, 100) # 150,000円
    )

    # 3-c. 2回目の給与支払 (合計 850,000円)
    controller.execute_payment()

    print("\n--- 最終支払ログ ---")
    # (ログ取得機能はController/UseCaseになく、Repoに直接アクセスする例)
    print(log_repo.get_all())
```

-----

## ✅ 3. リファクタリングの完了

これで、`OOP-01`の`PayrollSystem`（神クラス）を、クリーンアーキテクチャ（CA）の「型」に従ってリファクタリングする作業が完了しました。

リファクタリング後のコードは、以下の明確な「関心事（レイヤー）」を持つファイル群に分離されました。

  * **`domain.py`**:
      * **関心事**: ビジネスのルール（給与の計算方法）
  * **`application_boundaries.py`**:
      * **関心事**: レイヤー間の「契約（インターフェース）」
  * **`application.py`**:
      * **関心事**: ビジネスの手順（いつ、何を呼び出すか）
  * **`infrastructure.py`**:
      * **関心事**: データの保存方法（どうやって保存するか）
  * **`adapters.py`**:
      * **関心事**: 外部と内部の仲介・データ変換
  * **`main.py`**:
      * **関心事**: すべての「部品（具象クラス）」を組み立てる（DI）、処理を開始する（トリガー）

`main.py`を除くすべてのファイル（`application.py`, `adapters.py`など）は、`InMemory...`といった**具体的な実装**を一切知らず、\*\*抽象的なインターフェース（境界）\*\*にのみ依存する、非常に柔軟な構造になりました。
CAの「関心の分離」と「依存性のルール」という"型"が守られた状態です。

-----

## 🤔 次の課題（謎解き）

さて、**私たちのリファクタリングは完了しました**。
CAの「型」に従って「関心の分離」と「依存性のルール」を守ったことで、`OOP-01`よりもはるかに見通しが良く、変更に強い（例：DBへの変更が容易な）コードが完成したはずです。

（リファクタリング担当者Aは、ここで退場します）

-----

（ここからペルソナB / ナレーターが登場）

ところで、`OOP-02`のSOLID原則評価のことを覚えているでしょうか？
あの時、`OOP-01`のコードは\*\*SRP（単一責任の原則）**と**DIP（依存性逆転の原則）\*\*に甚だしく違反している、と評価しました。

**このCAリファクタリング（`OOP-03`〜`OOP-05`）は、これらの違反を解決できたのでしょうか？**

次の章（`OOP-06`）で、このリファクタリング結果を、もう一度**SOLID原則**という「ものさし」で評価し直し、その「答え合わせ」をしてみましょう。