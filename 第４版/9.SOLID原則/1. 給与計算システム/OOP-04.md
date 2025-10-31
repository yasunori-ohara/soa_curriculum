# OOP-04 : CAの「型」による実装 (2) - ユースケースとインフラ層

`OOP-03`では、`PayrollSystem`（神クラス）が抱えていた複数の「関心事」を、3つの独立した\*\*インターフェース（境界）\*\*に分離しました。

この章では、その「契約書」に基づいて、**具体的な実装**クラスを作成します。
`OOP-01`の`system.py`にあったコードが、CAのレイヤー（関心事）に従って異なるファイルに配置されます。

## 🎯 この章のゴール

  * アプリケーション層（**ユースケース**）を実装し、「アプリケーション固有のルール（手順）」をカプセル化する。
  * インフラストラクチャ層（**リポジトリ**）を実装し、「データ保存という外部の詳細」をカプセル化する。
  * ユースケースが**インターフェース（境界）にのみ依存**し、インフラ層（具象）を一切知らない状態（＝**CAの依存性のルール**）をコードで確認する。

-----

## 🛠️ 1\. アプリケーション層 (Use Cases) の実装

「ユースケース」は、アプリケーションの「やりたいこと（例：給与支払を実行する）」を実現するための、具体的な手順を定義します。
`OOP-01`の`PayrollSystem`が持っていた\*\*「責任C: 給与計算ロジック」\*\*がここに該当します。

**重要な点**: このファイルは、`domain.py`と`application_boundaries.py`（インターフェース）に**のみ**依存します。データベースやメモリ、具体的な保存方法（`InMemory...`）については一切知りません。

> **`application.py` (新設)**

```python
import datetime
# 「境界（インターフェース）」と「ドメイン」のみをインポート
from application_boundaries import (
    IExecutePayrollUseCase, 
    IEmployeeRepository, 
    IPayLogRepository
)
from domain import Employee

class ExecutePayrollUseCase(IExecutePayrollUseCase):
    """
    「給与支払実行」ユースケースの具体的な実装。
    OOP-01の「給与計算の実行手順」という"関心事"を担当する。
    """
    def __init__(self,
                 employee_repo: IEmployeeRepository,
                 pay_log_repo: IPayLogRepository):
        
        # ▼▼▼ CAの依存性のルール ▼▼▼
        # このクラスは、具象クラス（InMemory...）を知らない。
        # 抽象的な「I...Repository」というインターフェース（境界）にのみ
        # 依存している。
        self._employee_repo = employee_repo
        self._pay_log_repo = pay_log_repo

    def execute(self):
        """
        給与支払処理を実行する。
        OOP-01の PayrollSystem.execute_payment のロジックがここに移動した。
        """
        print("\n=== 支払処理実行 (UseCase) ===")
        
        # 1. 従業員データを取得する（リポジトリ[抽象]に依頼）
        employees = self._employee_repo.find_all()
        
        # 2. 給与を計算する（ドメインのロジックを呼び出す）
        print("\n--- 全従業員の給与計算開始 (UseCase) ---")
        total_pay = 0
        for emp in employees:
            # (OOP-01から継承) 相手が正社員かパートかを知る必要はない
            total_pay += emp.calculate_pay()
        print(f"--- 合計支払額: {total_pay}円 ---")

        # 3. 支払ログを保存する（リポジトリ[抽象]に依頼）
        self._pay_log_repo.save(
            total_pay=total_pay,
            employee_count=len(employees)
        )
```

-----

## 🛠️ 2\. インフラストラクチャ層 (Infrastructure) の実装

「インフラ層」は、`application_boundaries.py`で定義されたインターフェース（契約）を**具体的に実装**する層です。
`OOP-01`の`PayrollSystem`が持っていた\*\*「責任A: 従業員管理」**と**「責任B: 支払記録」\*\*がここに該当します。

> **`infrastructure.py` (新設)**

```python
import datetime
# 「境界（インターフェース）」と「ドメイン」をインポート
from application_boundaries import IEmployeeRepository, IPayLogRepository
from domain import Employee

class InMemoryEmployeeRepository(IEmployeeRepository):
    """
    【具象】IEmployeeRepositoryの「インメモリ」実装。
    OOP-01の「従業員管理」という"関心事"を担当する。
    """
    def __init__(self):
        self._employees: dict[str, Employee] = {}

    def find_all(self) -> list[Employee]:
        """【実装】辞書の値をリストにして返す"""
        return list(self._employees.values())

    def save(self, employee: Employee):
        """【実装】辞書にIDをキーにして保存（追加・更新）"""
        print(f"従業員保存(Infra): {employee.name}")
        self._employees[employee.employee_id] = employee

class InMemoryPayLogRepository(IPayLogRepository):
    """
    【具象】IPayLogRepositoryの「インメモリ」実装。
    OOP-01の「支払記録」という"関心事"を担当する。
    """
    def __init__(self):
        self._pay_logs: list[dict] = []

    def save(self, total_pay: int, employee_count: int):
        """【実装】ログ形式（辞書）を作成し、リストに保存"""
        log_entry = {
            "payment_date": datetime.datetime.now().isoformat(),
            "total_amount": total_pay,
            "employee_count": employee_count
        }
        self._pay_logs.append(log_entry)
        print(f"支払記録保存(Infra): {log_entry}")

    # (補足) ログ読み出し用の get_all() メソッドも
    # 本来はインターフェースに定義して実装すべきだが、今回は省略
    def get_all(self) -> list[dict]:
        return self._pay_logs
```

-----

## 🛠️ 3\. リファクタリングの進捗

`OOP-01`の`PayrollSystem`（神クラス）は、**ついに消滅しました。**

CAの「型」に従って「**関心を分離**」した結果、`OOP-01`の`system.py`にあったロジックは、3つの異なるクラス（とファイル）に分離されました。

  * **`application.py`**:
      * `ExecutePayrollUseCase`
      * **関心事**: ビジネスの手順（いつ、何を呼び出すか）
  * **`infrastructure.py`**:
      * `InMemoryEmployeeRepository`
      * `InMemoryPayLogRepository`
      * **関心事**: データの保存方法（どうやって保存するか）
  * **`domain.py`**:
      * `Employee`, `FullTimeEmployee`...
      * **関心事**: ビジネスのルール（給与とはどう計算されるべきか）

`application.py`（ユースケース層）は、`infrastructure.py`（インフラ層）の存在を一切知らず、`application_boundaries.py`（境界）だけを見ている状態になりました。
**これでCAの「依存性のルール」が守られました。**

-----

## 🚧 次の課題

アーキテクチャの「内側」（ドメイン、ユースケース）と「外側」（インフラ）は完成しました。
しかし、これらはまだ「部品」としてバラバラな状態です。

  * `UseCase`が依存する`IRepository`（抽象）に、どの「実装（`InMemory...`版）」を渡すのか、誰も決めていません。
  * `UseCase`を「誰が」呼び出すのかが決まっていません。

次の章（`OOP-05`）では、`main.py`で、これらの部品をすべて「**組み立てる（＝依存性の注入）**」作業を行い、システムを完成させます。