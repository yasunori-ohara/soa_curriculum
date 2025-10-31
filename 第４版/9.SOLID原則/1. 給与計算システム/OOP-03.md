# OOP-03 : CAの「型」による分離 (1) - 境界の定義

`OOP-01`のコード（特に`system.py`）を見て、クリーンアーキテクチャ（CA）を学んだ皆さんは、強い「違和感」を覚えたはずです。

`PayrollSystem`クラスは、

1.  **ユースケース**（アプリケーション固有のロジック、`execute_payment`の手順）
2.  **インフラストラクチャ**（データの保存方法、`self._employees = {}`や`self._pay_logs.append(...)`）

という、\*\*全く異なる「関心事（レイヤー）」\*\*が、一つのクラスに混在しています。

これは、CAの最も重要なルールである「**関心の分離 (Separation of Concerns)**」と「**依存性のルール (Dependency Rule)**」に明確に違反しています。
（`UseCase`が`Infrastructure`の実装に直接依存している状態です）

この章では、この「神クラス（God Class）」を、私たちが知っているCAの「型」に従って解体するリファクタリングに着手します。
まずは「設計図」として、CAの「**境界（Boundaries）**」を定義します。

## 🎯 この章のゴール

  * `OOP-01`の`PayrollSystem`（神クラス）を、CAのレイヤー（関心事）に基づいて分離する。
  * CAの「境界（Boundary）」として機能する**インターフェース**（Pythonの`ABC`）を定義する。
  * CAの「型」に当てはめるための「設計図」を完成させる。

-----

## 🛠️ 1\. ドメイン層 (Entities) の明確化

CAで最も内側（核）となるのは、ビジネスそのもののルールを定義する「ドメイン層（Entities）」です。

`OOP-01`の`employees.py`は、まさにこの層に該当します。このファイルには、従業員とその給与計算方法という、ビジネスの核となる「概念」と「ルール」が定義されています。

この層はCAの基盤であり、今回のリファクタリングでは（ファイル名を`domain.py`に変更する以外は）**変更の必要はありません**。

> **`domain.py` (旧 `employees.py`)**

```python
from abc import ABC, abstractmethod

# (内容は OOP-01 の employees.py と同じ)
# (Employee, FullTimeEmployee, PartTimeEmployee のクラス定義)

class Employee(ABC):
    """【ドメイン層】従業員の「概念」と「契約」"""
    def __init__(self, employee_id: str, name: str):
        self.employee_id = employee_id
        self.name = name

    @abstractmethod
    def calculate_pay(self) -> int: ...

class FullTimeEmployee(Employee):
    """【ドメイン層】正社員の「ルール」の実装"""
    def __init__(self, employee_id: str, name: str, monthly_salary: int): ...
    def calculate_pay(self) -> int: ...

class PartTimeEmployee(Employee):
    """【ドメイン層】パートタイマーの「ルール」の実装"""
    def __init__(self, employee_id: str, name: str, hourly_rate: int, hours_worked: int): ...
    def calculate_pay(self)R -> int: ...
```

-----

## 🛠️ 2\. アプリケーション境界 (Interfaces) の定義

ここがリファクタリングの核心です。
`PayrollSystem`（神クラス）が持っていた「**複数の関心事**」を、CAの「型」に従って分離します。

CAでは、**ユースケース（アプリケーション層）とインフラストラクチャ層**は分離され、両者は「**境界（インターフェース）**」を介してのみ通信します。

> **`application_boundaries.py` (新設)**

```python
from abc import ABC, abstractmethod
from domain import Employee # 依存は内側(domain)へ

# --- 1. ユースケース層の「境界」 ---
# PayrollSystemが持っていた「給与計算の実行手順」という"関心事"

class IExecutePayrollUseCase(ABC):
    """
    「給与支払を実行する」というユースケースの入力ポート（Input Port）。
    外部（main.py）は、このインターフェースを通じてユースケースを呼び出す。
    """
    @abstractmethod
    def execute(self):
        """給与支払処理を実行する"""
        pass

# --- 2. インフラ層の「境界」 ---
# PayrollSystemが持っていた「データ保存」という"関心事"

class IEmployeeRepository(ABC):
    """
    「従業員データ」を扱うリポジトリ（倉庫番）の「契約」。
    ユースケースは、このインターフェース（抽象）にのみ依存する。
    """
    @abstractmethod
    def find_all(self) -> list[Employee]:
        """【契約】全従業員を取得する"""
        pass
    
    @abstractmethod
    def save(self, employee: Employee):
        """【契約】従業員を追加・更新する"""
        pass

class IPayLogRepository(ABC):
    """
    「支払記録データ」を扱うリポジトリの「契約」。
    """
    @abstractmethod
    def save(self, total_pay: int, employee_count: int):
        """【契約】支払記録を保存する"""
        pass
```

-----

## 🛠️ 3\. 設計図の完成

`OOP-03`では、`OOP-01`の`PayrollSystem`を解体し、CAの「型」に従って3つの「**インターフェース（境界）**」を定義しました。

`PayrollSystem`が持っていた**3つの異なる関心事**（ユースケース、従業員保存、ログ保存）が、**3つの独立したインターフェース**に明確に分離されました。

また、CAの「**依存性のルール**」に従い、`UseCase`（高レベル）と`Infrastructure`（低レベル）が、`I-Repository`（抽象）を介して通信する「設計図」が完成しました。

これで、「神クラス」を解体するための準備が整いました。

-----

## 🚧 次の課題

`OOP-03`では、CAの「設計図（境界）」を定義しました。
次の章（`OOP-04`）では、この設計図に基づいて、`IExecutePayrollUseCase`（ユースケース層）と、`InMemory...Repository`（インフラストラクチャ層）の**具体的な実装**を行っていきます。