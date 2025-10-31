# OOP-01 : リファクタリングの題材「給与計算システム」

この章では、「オブジェクト指向で書かれたプログラムを、クリーンアーキテクチャの型にはめてリファクタリングしたら、結果、SOLID原則も適用できていた。」という例を、「給与計算システム」のプログラムを例題として実践していきます。

このプログラムは、「従業員」のクラス（「設計図」）を定義する`employees.py`、システムの中核ロジックを担う`system.py`、そしてシステムを起動する`main.py`の3つのファイルで構成されている、ごく一般的なOOPのコードです。

## 🎯 この章のゴール

  * リファクタリング対象のコード（給与計算システム）の仕様と構成を理解する。
  * クラスの「**継承**」が、異なる種類の従業員（正社員、パート）を定義するためにどのように使われているかを確認する。
  * システムが `employees.py`, `system.py`, `main.py` の3つのファイルで構成されていることを把握する。

-----

## 🛠️ 1\. `employees.py` : 従業員クラスの定義

このファイルは、「従業員」という概念をクラスとして定義する場所です。
ここでは、オブジェクト指向の基本的な技術である\*\*継承（Inheritance）\*\*が使われています。

まず、`Employee`という「**基底クラス（Base Class）**」を定義します。これは、「従業員であれば共通して持つべき情報（ID、名前）と、必ずできなければならないこと（`calculate_pay`＝給与計算）」の"原型"や"テンプレート"のようなものです。

次に、具体的な従業員である`FullTimeEmployee`（正社員）と`PartTimeEmployee`（パートタイマー）が、その`Employee`クラスを**継承**し、`calculate_pay`メソッドをそれぞれ独自の方法（月給制、時給制）で具体化（**実装**）しています。

```python
from abc import ABC, abstractmethod

class Employee(ABC):
    """
    従業員の「基底クラス」（テンプレート）。
    「給与を計算できる(calculate_pay)」という振る舞いを
    このクラスを継承するすべてのクラスに強制する。
    """
    def __init__(self, employee_id: str, name: str):
        self.employee_id = employee_id
        self.name = name

    @abstractmethod
    def calculate_pay(self) -> int:
        """【契約】月給を計算する"""
        pass

class FullTimeEmployee(Employee):
    """【具象】正社員。月給制。"""
    def __init__(self, employee_id: str, name: str, monthly_salary: int):
        super().__init__(employee_id, name)
        self.monthly_salary = monthly_salary

    def calculate_pay(self) -> int:
        """【実装】月給をそのまま返す"""
        pay = self.monthly_salary
        print(f"  正社員({self.name}): 月給 {pay}円")
        return pay

class PartTimeEmployee(Employee):
    """【具象】パートタイマー。時給制。"""
    def __init__(self, employee_id: str, name: str, hourly_rate: int, hours_worked: int):
        super().__init__(employee_id, name)
        self.hourly_rate = hourly_rate
        self.hours_worked = hours_worked # (簡略化のため今月の労働時間を持つ)

    def calculate_pay(self) -> int:
        """【実装】時給 * 時間 を返す"""
        pay = self.hourly_rate * self.hours_worked
        print(f"  パート({self.name}): 時給{self.hourly_rate}円 * {self.hours_worked}h = {pay}円")
        return pay
```

-----

## 🛠️ 2\. `system.py` : システムロジック

このファイルは、システムの中核的なロジックを実装しています。
`PayrollSystem`という一つのクラスが、従業員データの管理、給与計算の実行、支払ログの保存といった機能を提供します。

```python
import datetime
# employees.py で定義した Employee (基底クラス) をインポート
from employees import Employee 

class PayrollSystem:
    """
    給与計算に関するすべての機能を提供するクラス。
    """
    def __init__(self):
        # 従業員データを保存するための辞書
        self._employees: dict[str, Employee] = {}
        
        # 支払ログを保存するためのリスト
        self._pay_logs: list[dict] = []

    def add_employee(self, employee: Employee):
        """従業員をシステムに追加する"""
        self._employees[employee.employee_id] = employee
        print(f"従業員追加: {employee.name}")

    def calculate_payroll(self) -> int:
        """全従業員の給与を計算し、合計額を返す"""
        print("\n--- 全従業員の給与計算開始 ---")
        total_pay = 0
        
        # このループ処理に注目：
        # ループは、取り出した `emp` が「正社員」か「パート」かを
        # 一切気にしていません (if文での分岐がない)。
        #
        # `emp` は `Employee` を継承しているので、
        # 「必ず calculate_pay() メソッドを持っている」ことを知っているため、
        # ただそのメソッドを呼び出すだけでよいのです。
        for emp_id, emp in self._employees.items():
            total_pay += emp.calculate_pay()
            
        print(f"--- 合計支払額: {total_pay}円 ---")
        return total_pay

    def execute_payment(self):
        """給与支払処理（計算とログ保存）を実行する"""
        print("\n=== 支払処理実行 ===")
        
        # 計算ロジックを呼び出す
        total_pay = self.calculate_payroll()
        
        # ログの形式（辞書）を決定する
        log_entry = {
            "payment_date": datetime.datetime.now().isoformat(),
            "total_amount": total_pay,
            "employee_count": len(self._employees)
        }
        
        # ログをリストに保存する
        self._pay_logs.append(log_entry)
        print(f"支払記録保存: {log_entry}")
    
    def get_pay_logs(self) -> list[dict]:
        """保存されている支払ログを取得する"""
        return self._pay_logs
```

-----

## 🛠️ 3\. `main.py` : システムの実行

このファイルは、`PayrollSystem`クラスをインスタンス化（実体化）し、実際にシステムを動かす役割を持ちます。

```python
# employees.py から「具体的な」従業員クラスをインポート
from employees import FullTimeEmployee, PartTimeEmployee
# system.py から「ロジック」クラスをインポート
from system import PayrollSystem

if __name__ == "__main__":
    # 1. システム（ロジック）のインスタンスを作成
    payroll_system = PayrollSystem()

    # 2. 従業員（データ）のインスタンスを作成し、システムに追加
    payroll_system.add_employee(
        FullTimeEmployee("e-001", "田中 太郎", 300000)
    )
    payroll_system.add_employee(
        PartTimeEmployee("e-002", "鈴木 花子", 1500, 80) # 120,000円
    )
    payroll_system.add_employee(
        FullTimeEmployee("e-003", "佐藤 次郎", 400000)
    )

    # 3. システムのメインロジック（給与支払）を実行
    #    (合計 300,000 + 120,000 + 400,000 = 820,000円)
    payroll_system.execute_payment()
    
    # (翌月... 鈴木さんの労働時間が変わったと仮定)
    # ※同じIDで追加すると、辞書なので上書きされる
    payroll_system.add_employee(
        PartTimeEmployee("e-002", "鈴木 花子", 1500, 100) # 150,000円
    )
    
    # 4. システムのメインロジックを再度実行
    #    (合計 300,000 + 150,000 + 400,000 = 850,000円)
    payroll_system.execute_payment()
    
    print("\n--- 最終支払ログ ---")
    print(payroll_system.get_pay_logs())
```

-----

## `OOP-02`への準備

このコードは、`employees.py`で継承をうまく活用することで、「新しい従業員の種類（例：契約社員）」の追加には柔軟に対応できる可能性があります。

しかし、`system.py`の`PayrollSystem`クラスに、従業員管理、給与計算、ログ保存など、多くの機能が集中しているようにも見えます。

このコードは「**変更**」に対してどれだけ強いのでしょうか？

次の章では、この`OOP-01`のコードが「**良い設計**」と呼べるかどうか、**SOLID原則**というものさしを使って詳しく評価（健康診断）してみましょう。