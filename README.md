# 🏗️ Software Design Principles: SOLID, DRY, KISS & YAGNI

> A comprehensive guide with real-world code comparisons, theory, and practical examples.

---

## 📚 Table of Contents

1. [Why Design Principles Matter](#why-design-principles-matter)
2. [SOLID Principles](#solid-principles)
   - [S — Single Responsibility Principle (SRP)](#s--single-responsibility-principle-srp)
   - [O — Open/Closed Principle (OCP)](#o--openclosed-principle-ocp)
   - [L — Liskov Substitution Principle (LSP)](#l--liskov-substitution-principle-lsp)
   - [I — Interface Segregation Principle (ISP)](#i--interface-segregation-principle-isp)
   - [D — Dependency Inversion Principle (DIP)](#d--dependency-inversion-principle-dip)
3. [DRY — Don't Repeat Yourself](#dry--dont-repeat-yourself)
4. [KISS — Keep It Simple, Stupid](#kiss--keep-it-simple-stupid)
5. [YAGNI — You Aren't Gonna Need It](#yagni--you-arent-gonna-need-it)
6. [Principles Working Together](#principles-working-together)
7. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Why Design Principles Matter

Software design principles are **battle-tested guidelines** that help developers write code that is:

- **Maintainable** — Easy to change without breaking other things
- **Readable** — Understandable by other developers (including your future self)
- **Testable** — Components can be tested in isolation
- **Scalable** — Can grow without becoming a tangled mess
- **Reusable** — Logic can be shared across the codebase

Without design principles, codebases become what developers call **"Big Ball of Mud"** — a sprawling, spaghetti-like structure where no one dares to change anything. These principles are not rules to follow blindly; they are lenses to evaluate your code quality.

---

## SOLID Principles

**SOLID** is an acronym coined by Robert C. Martin (Uncle Bob). Each letter represents a principle that — when followed together — leads to software that is modular, flexible, and robust.

| Letter | Principle | Core Idea |
|--------|-----------|-----------|
| **S** | Single Responsibility | A class should have only one reason to change |
| **O** | Open/Closed | Open for extension, closed for modification |
| **L** | Liskov Substitution | Subtypes must be substitutable for their base types |
| **I** | Interface Segregation | Clients shouldn't depend on interfaces they don't use |
| **D** | Dependency Inversion | Depend on abstractions, not concretions |

---

## S — Single Responsibility Principle (SRP)

### Theory

> *"A class should have one, and only one, reason to change."*
> — Robert C. Martin

A **"reason to change"** maps directly to a **responsibility**. If a class handles user authentication AND sends emails AND logs activity, then it has three reasons to change:
- Business logic around authentication changes
- Email templates change
- Logging format changes

Every one of these changes forces you to open the same class, increasing the risk of introducing bugs and making the class harder to understand and test.

**Key Insight:** SRP is about **cohesion**. Code that belongs together stays together; code that doesn't, gets separated. A class should represent a single concept in your domain.

### ❌ Bad Example — Violating SRP

```python
class User:
    def __init__(self, name: str, email: str):
        self.name = name
        self.email = email

    def get_user_data(self):
        """Responsibility 1: Managing user data"""
        return {"name": self.name, "email": self.email}

    def save_to_database(self):
        """Responsibility 2: Persistence — belongs in a repository/DAO"""
        import sqlite3
        conn = sqlite3.connect("users.db")
        cursor = conn.cursor()
        cursor.execute(
            "INSERT INTO users (name, email) VALUES (?, ?)",
            (self.name, self.email)
        )
        conn.commit()
        conn.close()
        print(f"User {self.name} saved to database.")

    def send_welcome_email(self):
        """Responsibility 3: Email communication — belongs in a mailer service"""
        import smtplib
        server = smtplib.SMTP("smtp.gmail.com", 587)
        server.sendmail(
            "app@example.com",
            self.email,
            f"Welcome, {self.name}!"
        )
        server.quit()
        print(f"Welcome email sent to {self.email}.")

    def generate_report(self):
        """Responsibility 4: Reporting — belongs in a report service"""
        report = f"User Report:\nName: {self.name}\nEmail: {self.email}"
        with open("user_report.txt", "w") as f:
            f.write(report)
        print("Report generated.")
```

**Problems:**
- The `User` class has 4 different reasons to change
- Testing `get_user_data()` requires mocking a database AND an SMTP server
- Changing the database from SQLite to PostgreSQL forces you to edit the `User` class
- Hard to reuse the email logic for other entities (e.g., sending emails to admins)

### ✅ Good Example — Following SRP

```python
# Responsibility 1: Pure data/domain model
class User:
    def __init__(self, name: str, email: str):
        self.name = name
        self.email = email

    def get_data(self) -> dict:
        return {"name": self.name, "email": self.email}


# Responsibility 2: Persistence
class UserRepository:
    def __init__(self, db_connection):
        self.db = db_connection

    def save(self, user: User):
        self.db.execute(
            "INSERT INTO users (name, email) VALUES (?, ?)",
            (user.name, user.email)
        )
        self.db.commit()
        print(f"User {user.name} saved.")


# Responsibility 3: Email communication
class EmailService:
    def __init__(self, smtp_client):
        self.smtp = smtp_client

    def send_welcome_email(self, user: User):
        message = f"Welcome, {user.name}!"
        self.smtp.sendmail("app@example.com", user.email, message)
        print(f"Welcome email sent to {user.email}.")


# Responsibility 4: Reporting
class UserReportGenerator:
    def generate(self, user: User) -> str:
        return f"User Report:\nName: {user.name}\nEmail: {user.email}"

    def save_to_file(self, user: User, filepath: str):
        report = self.generate(user)
        with open(filepath, "w") as f:
            f.write(report)
        print(f"Report saved to {filepath}.")


# --- Usage ---
user = User("Alice", "alice@example.com")

repo = UserRepository(db_connection=get_db())
repo.save(user)

mailer = EmailService(smtp_client=get_smtp())
mailer.send_welcome_email(user)

reporter = UserReportGenerator()
reporter.save_to_file(user, "alice_report.txt")
```

**Benefits:**
- Each class has one job and one reason to change
- `UserRepository` can be swapped (SQLite → PostgreSQL) without touching `User`
- `EmailService` can be tested with a mock SMTP client
- `UserReportGenerator` can be used for any entity, not just `User`

---

## O — Open/Closed Principle (OCP)

### Theory

> *"Software entities (classes, modules, functions) should be open for extension, but closed for modification."*
> — Bertrand Meyer, popularized by Robert C. Martin

**Open for extension** means you can add new behaviors.
**Closed for modification** means you don't change existing, tested code to do so.

The mechanism that enables OCP is **abstraction** — interfaces, abstract classes, or polymorphism. By coding to an abstraction rather than a concrete class, you can plug in new implementations without touching the code that uses them.

**Why it matters:** Every time you modify working code, you risk introducing regressions. If you can *extend* behavior without modifying it, your existing tests remain valid and your risk stays low.

### ❌ Bad Example — Violating OCP

```python
class DiscountCalculator:
    def calculate(self, order, customer_type: str) -> float:
        """
        Every time a new customer type is introduced, this method must be
        modified. This is a classic OCP violation.
        """
        if customer_type == "regular":
            return order.total * 0.0   # No discount

        elif customer_type == "premium":
            return order.total * 0.10  # 10% discount

        elif customer_type == "vip":
            return order.total * 0.20  # 20% discount

        # What happens when we add "employee" or "student"?
        # We MUST come back and edit this class!
        else:
            raise ValueError(f"Unknown customer type: {customer_type}")
```

**Problems:**
- Adding a new customer type requires modifying this class
- Each modification risks breaking the `regular`, `premium`, and `vip` logic
- The `if/elif` chain grows endlessly as requirements grow
- Unit tests for existing discounts need to be re-run after every addition

### ✅ Good Example — Following OCP

```python
from abc import ABC, abstractmethod

# The abstraction — this never changes
class DiscountStrategy(ABC):
    @abstractmethod
    def get_discount(self, order_total: float) -> float:
        pass


# Concrete implementations — we ADD these without modifying others
class NoDiscount(DiscountStrategy):
    def get_discount(self, order_total: float) -> float:
        return 0.0


class PremiumDiscount(DiscountStrategy):
    def get_discount(self, order_total: float) -> float:
        return order_total * 0.10


class VIPDiscount(DiscountStrategy):
    def get_discount(self, order_total: float) -> float:
        return order_total * 0.20


# New requirement: Employee discount — just ADD a new class, touch nothing else!
class EmployeeDiscount(DiscountStrategy):
    def get_discount(self, order_total: float) -> float:
        return order_total * 0.35


# Student discount — same story
class StudentDiscount(DiscountStrategy):
    def get_discount(self, order_total: float) -> float:
        return order_total * 0.15


# The calculator is CLOSED for modification
class DiscountCalculator:
    def __init__(self, strategy: DiscountStrategy):
        self.strategy = strategy

    def calculate(self, order_total: float) -> float:
        return self.strategy.get_discount(order_total)


# --- Usage ---
order_total = 200.0

regular_calc = DiscountCalculator(NoDiscount())
print(regular_calc.calculate(order_total))    # 0.0

premium_calc = DiscountCalculator(PremiumDiscount())
print(premium_calc.calculate(order_total))   # 20.0

employee_calc = DiscountCalculator(EmployeeDiscount())
print(employee_calc.calculate(order_total))  # 70.0
```

**Benefits:**
- Adding `StudentDiscount` required zero changes to `DiscountCalculator` or existing strategies
- All existing tests remain valid when you add a new strategy
- The `DiscountCalculator` class is stable and never needs to change

---

## L — Liskov Substitution Principle (LSP)

### Theory

> *"If S is a subtype of T, then objects of type T may be replaced with objects of type S without altering any of the desirable properties of the program."*
> — Barbara Liskov, 1987

In plain English: **any child class should be usable wherever its parent class is expected, without breaking anything.**

LSP is about **behavioral correctness** in inheritance. A subclass shouldn't:
- Throw exceptions the base class doesn't throw
- Accept a narrower range of inputs
- Return a narrower range of outputs
- Weaken postconditions or strengthen preconditions

If you have to use `isinstance()` checks to figure out which subclass you're dealing with, that's often a sign of an LSP violation.

### ❌ Bad Example — Violating LSP

```python
class Rectangle:
    def __init__(self, width: float, height: float):
        self._width = width
        self._height = height

    def set_width(self, width: float):
        self._width = width

    def set_height(self, height: float):
        self._height = height

    def area(self) -> float:
        return self._width * self._height


class Square(Rectangle):
    """
    A Square IS-A Rectangle mathematically, but this inheritance
    breaks LSP because a Square's width and height are always equal.
    """
    def set_width(self, width: float):
        # A square must keep width == height
        self._width = width
        self._height = width  # ← This side effect breaks LSP!

    def set_height(self, height: float):
        self._width = height   # ← Same problem
        self._height = height


# This function works perfectly for Rectangle...
def double_width(shape: Rectangle):
    original_height = shape._height
    shape.set_width(shape._width * 2)
    # We expect: area = (width * 2) * original_height
    assert shape.area() == shape._width * original_height, "Area calculation is wrong!"

rect = Rectangle(4, 5)
double_width(rect)  # ✅ Works fine: area = 8 * 5 = 40

square = Square(4, 4)
double_width(square)  # ❌ FAILS! set_width also changes height, so area = 8 * 8 = 64, not 8 * 4
```

**The problem:** A `Square` cannot be substituted for a `Rectangle` without breaking the program's expected behavior. The mathematical "IS-A" relationship doesn't always translate to an inheritance relationship in code.

### ✅ Good Example — Following LSP

```python
from abc import ABC, abstractmethod

# Use a common abstraction instead of a broken inheritance chain
class Shape(ABC):
    @abstractmethod
    def area(self) -> float:
        pass

    @abstractmethod
    def perimeter(self) -> float:
        pass


class Rectangle(Shape):
    def __init__(self, width: float, height: float):
        self.width = width
        self.height = height

    def area(self) -> float:
        return self.width * self.height

    def perimeter(self) -> float:
        return 2 * (self.width + self.height)


class Square(Shape):
    def __init__(self, side: float):
        self.side = side

    def area(self) -> float:
        return self.side ** 2

    def perimeter(self) -> float:
        return 4 * self.side


# This function accepts ANY Shape and works correctly for ALL of them
def print_shape_info(shape: Shape):
    print(f"Area: {shape.area()}, Perimeter: {shape.perimeter()}")

shapes = [Rectangle(4, 5), Square(4), Rectangle(3, 7)]
for shape in shapes:
    print_shape_info(shape)   # ✅ Works correctly for all shapes
```

**Benefits:**
- `Square` and `Rectangle` each manage their own invariants cleanly
- Any `Shape` subtype can replace any other `Shape` in `print_shape_info`
- No unexpected side effects from method calls

---

## I — Interface Segregation Principle (ISP)

### Theory

> *"Clients should not be forced to depend on interfaces they do not use."*
> — Robert C. Martin

A **"fat interface"** is an interface with too many methods. When a class implements it, it may be forced to implement methods that are irrelevant to it, often leading to empty implementations or exceptions being thrown — which is a smell.

ISP says: **split large interfaces into smaller, more focused ones**. Clients then depend only on the slice they need.

**Key Insight:** ISP is the interface-level equivalent of SRP. Instead of one giant interface, prefer several small, cohesive ones. A class can implement multiple small interfaces if it genuinely supports all those behaviors.

### ❌ Bad Example — Violating ISP

```python
from abc import ABC, abstractmethod

# A "fat" interface that tries to describe all possible workers
class Worker(ABC):
    @abstractmethod
    def work(self):
        pass

    @abstractmethod
    def eat(self):
        pass

    @abstractmethod
    def sleep(self):
        pass

    @abstractmethod
    def attend_meeting(self):
        pass


class HumanEmployee(Worker):
    def work(self):
        print("Human is working...")

    def eat(self):
        print("Human is eating lunch...")

    def sleep(self):
        print("Human is sleeping...")

    def attend_meeting(self):
        print("Human is in a meeting...")


class Robot(Worker):
    def work(self):
        print("Robot is working...")

    def eat(self):
        # Robots don't eat! Forced to implement a meaningless method.
        raise NotImplementedError("Robots don't eat!")

    def sleep(self):
        # Robots don't sleep! Another meaningless method.
        raise NotImplementedError("Robots don't sleep!")

    def attend_meeting(self):
        # Robots don't attend meetings! (Usually)
        raise NotImplementedError("Robots don't attend meetings!")
```

**Problems:**
- `Robot` is forced to implement `eat()`, `sleep()`, and `attend_meeting()` which make no sense
- `NotImplementedError` at runtime instead of a compile-time/static check
- Any code calling `worker.eat()` must know whether it's dealing with a `Robot` or a `Human` — coupling!

### ✅ Good Example — Following ISP

```python
from abc import ABC, abstractmethod

# Segregated, focused interfaces
class Workable(ABC):
    @abstractmethod
    def work(self):
        pass


class Eatable(ABC):
    @abstractmethod
    def eat(self):
        pass


class Sleepable(ABC):
    @abstractmethod
    def sleep(self):
        pass


class MeetingAttendable(ABC):
    @abstractmethod
    def attend_meeting(self):
        pass


# Human implements ALL the interfaces it genuinely supports
class HumanEmployee(Workable, Eatable, Sleepable, MeetingAttendable):
    def work(self):
        print("Human is working...")

    def eat(self):
        print("Human is eating lunch...")

    def sleep(self):
        print("Human is sleeping...")

    def attend_meeting(self):
        print("Human is in a meeting...")


# Robot only implements what it genuinely supports
class Robot(Workable):
    def work(self):
        print("Robot is working 24/7...")


# Functions depend only on the interface they need
def start_work_shift(worker: Workable):
    worker.work()

def schedule_lunch(entity: Eatable):
    entity.eat()


human = HumanEmployee()
robot = Robot()

start_work_shift(human)  # ✅
start_work_shift(robot)  # ✅

schedule_lunch(human)    # ✅
# schedule_lunch(robot)  # ❌ Type error at compile time — robot is not Eatable
```

**Benefits:**
- `Robot` is never forced to implement `eat()` or `sleep()`
- Type errors are caught early by type checkers (mypy, pyright)
- Code is easier to read: if a function takes `Eatable`, you immediately know it eats things

---

## D — Dependency Inversion Principle (DIP)

### Theory

> *"High-level modules should not depend on low-level modules. Both should depend on abstractions. Abstractions should not depend on details. Details should depend on abstractions."*
> — Robert C. Martin

This is perhaps the most powerful SOLID principle. Let's unpack it:

- **High-level modules** contain business logic (e.g., `OrderService`)
- **Low-level modules** contain implementation details (e.g., `MySQLDatabase`, `SendGridMailer`)
- **Abstractions** are interfaces/abstract classes that define a contract

Without DIP, high-level business logic is tightly coupled to low-level infrastructure. Changing your database from MySQL to PostgreSQL means rewriting your `OrderService`. That's fragile.

With DIP, `OrderService` depends on an `IDatabase` interface. MySQL and PostgreSQL are just *implementations* of that interface. You can swap them without touching business logic at all.

**DIP enables Dependency Injection (DI):** Instead of a class creating its dependencies, dependencies are *injected* from the outside. This makes classes independently testable.

### ❌ Bad Example — Violating DIP

```python
class MySQLDatabase:
    def save(self, data: dict):
        print(f"Saving {data} to MySQL...")

    def find(self, query: str):
        print(f"Querying MySQL: {query}")
        return {"id": 1, "name": "Alice"}


class SendGridMailer:
    def send(self, to: str, subject: str, body: str):
        print(f"Sending email via SendGrid to {to}: {subject}")


class OrderService:
    """
    High-level business logic is DIRECTLY coupled to low-level details.
    OrderService creates its own dependencies — it's in charge of HOW
    data is saved and HOW emails are sent.
    """
    def __init__(self):
        # Hard dependency on concrete implementations!
        self.db = MySQLDatabase()
        self.mailer = SendGridMailer()

    def place_order(self, order: dict):
        # Business logic mixed with infrastructure concerns
        self.db.save(order)
        self.mailer.send(
            order["customer_email"],
            "Order Confirmed",
            f"Your order #{order['id']} has been placed."
        )
        print("Order placed successfully.")
```

**Problems:**
- Impossible to unit test `OrderService` without a real MySQL and SendGrid connection
- Switching to PostgreSQL requires editing `OrderService` — a high-level module
- `OrderService` and `MySQLDatabase` are inseparably coupled

### ✅ Good Example — Following DIP

```python
from abc import ABC, abstractmethod

# Abstractions (the "contracts")
class IDatabase(ABC):
    @abstractmethod
    def save(self, data: dict):
        pass

    @abstractmethod
    def find(self, query: str) -> dict:
        pass


class IMailer(ABC):
    @abstractmethod
    def send(self, to: str, subject: str, body: str):
        pass


# Low-level implementations depend on the abstractions (they implement them)
class MySQLDatabase(IDatabase):
    def save(self, data: dict):
        print(f"Saving {data} to MySQL...")

    def find(self, query: str) -> dict:
        print(f"Querying MySQL: {query}")
        return {"id": 1, "name": "Alice"}


class PostgreSQLDatabase(IDatabase):
    def save(self, data: dict):
        print(f"Saving {data} to PostgreSQL...")

    def find(self, query: str) -> dict:
        print(f"Querying PostgreSQL: {query}")
        return {"id": 1, "name": "Alice"}


class SendGridMailer(IMailer):
    def send(self, to: str, subject: str, body: str):
        print(f"Sending via SendGrid to {to}: {subject}")


class SMTPMailer(IMailer):
    def send(self, to: str, subject: str, body: str):
        print(f"Sending via SMTP to {to}: {subject}")


# For testing — a mock implementation!
class MockMailer(IMailer):
    def __init__(self):
        self.sent_emails = []

    def send(self, to: str, subject: str, body: str):
        self.sent_emails.append({"to": to, "subject": subject})


# High-level module depends only on abstractions — injected from outside
class OrderService:
    def __init__(self, db: IDatabase, mailer: IMailer):
        self.db = db
        self.mailer = mailer

    def place_order(self, order: dict):
        self.db.save(order)
        self.mailer.send(
            order["customer_email"],
            "Order Confirmed",
            f"Your order #{order['id']} has been placed."
        )
        print("Order placed successfully.")


# --- Production usage ---
service = OrderService(
    db=PostgreSQLDatabase(),    # Swap MySQL → PostgreSQL with zero code change in OrderService
    mailer=SendGridMailer()
)
service.place_order({"id": 42, "customer_email": "alice@example.com"})


# --- Testing with mocks --- (No real DB or email needed!)
mock_mailer = MockMailer()
test_db = PostgreSQLDatabase()  # or an InMemoryDatabase mock
test_service = OrderService(db=test_db, mailer=mock_mailer)
test_service.place_order({"id": 42, "customer_email": "test@example.com"})

assert len(mock_mailer.sent_emails) == 1
assert mock_mailer.sent_emails[0]["to"] == "test@example.com"
print("✅ Test passed!")
```

**Benefits:**
- `OrderService` is 100% testable in isolation with mock objects
- You can swap `MySQLDatabase` for `PostgreSQLDatabase` in one line
- Business logic and infrastructure concerns are cleanly separated

---

## DRY — Don't Repeat Yourself

### Theory

> *"Every piece of knowledge must have a single, unambiguous, authoritative representation within a system."*
> — Andrew Hunt & David Thomas, *The Pragmatic Programmer* (1999)

DRY is one of the most fundamental principles in software development. It states that **logic, data, or knowledge should never be duplicated** in a codebase.

**DRY is about knowledge, not just code.** Two identical code snippets that represent *different* concepts are not a DRY violation. But two pieces of code that represent the *same business rule* — even if they look different — are a violation.

**Symptoms of DRY violations:**
- "Copy-paste programming" — copying code and adjusting slightly
- The same validation logic appearing in multiple controllers
- Magic numbers or strings appearing in multiple places
- Updating one thing requires searching and updating many files

**When DRY is misapplied (over-DRY):**
Sometimes developers abstract things together too eagerly. If two pieces of code happen to look the same but represent *independent concepts*, merging them creates **accidental coupling**. Rule of thumb: apply DRY after you see something repeated **three times**, not just twice.

### ❌ Bad Example — Violating DRY

```python
# Each function duplicates the same validation and calculation logic
def calculate_employee_tax(salary: float, age: int) -> float:
    # Validation duplicated
    if salary <= 0:
        raise ValueError("Salary must be positive")
    if age < 18 or age > 70:
        raise ValueError("Age must be between 18 and 70")

    # Tax calculation duplicated across functions
    if salary < 30000:
        tax_rate = 0.10
    elif salary < 60000:
        tax_rate = 0.20
    else:
        tax_rate = 0.30

    return salary * tax_rate


def calculate_contractor_tax(salary: float, age: int) -> float:
    # Exact same validation — duplicated!
    if salary <= 0:
        raise ValueError("Salary must be positive")
    if age < 18 or age > 70:
        raise ValueError("Age must be between 18 and 70")

    # Exact same tax bracket logic — duplicated!
    if salary < 30000:
        tax_rate = 0.10
    elif salary < 60000:
        tax_rate = 0.20
    else:
        tax_rate = 0.30

    # Contractors get an additional 5% self-employment tax
    return salary * (tax_rate + 0.05)


def calculate_freelancer_tax(salary: float, age: int) -> float:
    # Duplicated a third time!
    if salary <= 0:
        raise ValueError("Salary must be positive")
    if age < 18 or age > 70:
        raise ValueError("Age must be between 18 and 70")

    if salary < 30000:
        tax_rate = 0.10
    elif salary < 60000:
        tax_rate = 0.20
    else:
        tax_rate = 0.30

    return salary * (tax_rate + 0.08)
```

**Problems:**
- If the tax bracket thresholds change (30000 → 35000), you must update 3 places
- If the validation logic changes (age limit → 65), you must hunt down all copies
- Bug in validation? It exists in 3 places

### ✅ Good Example — Following DRY

```python
def validate_payroll_inputs(salary: float, age: int):
    """Single source of truth for validation."""
    if salary <= 0:
        raise ValueError("Salary must be positive")
    if age < 18 or age > 70:
        raise ValueError("Age must be between 18 and 70")


def get_base_tax_rate(salary: float) -> float:
    """Single source of truth for tax brackets."""
    if salary < 30000:
        return 0.10
    elif salary < 60000:
        return 0.20
    else:
        return 0.30


def calculate_tax(salary: float, age: int, additional_rate: float = 0.0) -> float:
    """Shared calculation logic. Each worker type only specifies what's unique."""
    validate_payroll_inputs(salary, age)
    base_rate = get_base_tax_rate(salary)
    return salary * (base_rate + additional_rate)


# Each function is now concise, unique, and readable
def calculate_employee_tax(salary: float, age: int) -> float:
    return calculate_tax(salary, age)  # No additional rate

def calculate_contractor_tax(salary: float, age: int) -> float:
    return calculate_tax(salary, age, additional_rate=0.05)

def calculate_freelancer_tax(salary: float, age: int) -> float:
    return calculate_tax(salary, age, additional_rate=0.08)
```

**Benefits:**
- Changing the 30000 bracket threshold requires editing exactly ONE line
- Validation logic lives in one place — one fix fixes all
- Intent is clearer: you can immediately see that contractors pay 5% extra

---

## KISS — Keep It Simple, Stupid

### Theory

> *"Simplicity is the ultimate sophistication."*
> — Leonardo da Vinci

The KISS principle, coined in the U.S. Navy in 1960, states that **most systems work best when kept simple rather than made complicated**. Unnecessary complexity is a liability — it increases the chance of bugs, makes onboarding harder, and slows down development.

**KISS doesn't mean "write dumb code."** It means:
- Prefer straightforward solutions over clever ones
- Avoid premature optimization
- Don't over-engineer before you understand the problem
- Write code that the next developer can understand in 5 minutes

**Signs you're violating KISS:**
- You need to draw a diagram just to explain a single function
- A new teammate says "what does this even do?"
- You spent a day writing a "general solution" for a problem that only has one case
- You're using advanced language features where simple ones would suffice

### ❌ Bad Example — Violating KISS

```python
from functools import reduce
from typing import Callable, List, Any
import operator

def over_engineered_sum(numbers: List[float]) -> float:
    """
    Someone tried to be extremely clever and "functional" here.
    This is hard to read, debug, and offers zero benefits over the simple version.
    """
    pipeline: List[Callable[[List[float]], Any]] = [
        lambda xs: filter(lambda x: isinstance(x, (int, float)), xs),
        lambda xs: map(lambda x: float(x), xs),
        lambda xs: reduce(operator.add, xs, 0.0)
    ]
    return reduce(lambda acc, fn: fn(acc), pipeline, numbers)


def complex_is_even(n: int) -> bool:
    """Over-engineered even-number check using bit manipulation + ternary + lambda."""
    check = lambda x: True if not (x & 1) else False
    return check(n)


def verbose_greet(name: str) -> str:
    """
    Overly complex string handling with multiple unnecessary intermediate steps.
    """
    chars = list(name)
    stripped_chars = [c for c in chars if c.strip()]
    capitalized = ''.join(
        [c.upper() if i == 0 else c.lower() for i, c in enumerate(stripped_chars)]
    )
    greeting_components = ["Hello", ",", " ", capitalized, "!"]
    return ''.join(greeting_components)
```

### ✅ Good Example — Following KISS

```python
def calculate_sum(numbers: list) -> float:
    """Clear, direct, and immediately understandable."""
    return sum(x for x in numbers if isinstance(x, (int, float)))


def is_even(n: int) -> bool:
    """Anyone can read this. No need for bit manipulation."""
    return n % 2 == 0


def greet(name: str) -> str:
    """Python's built-in string methods handle this perfectly."""
    return f"Hello, {name.strip().capitalize()}!"


# --- Another KISS example: parsing user input ---

# ❌ COMPLEX (premature optimization)
import re
def parse_age_bad(input_str: str) -> int:
    pattern = re.compile(r'^\s*(\+?(?:0|[1-9]\d{0,2}))\s*$')
    match = pattern.fullmatch(input_str.strip())
    if not match:
        raise ValueError(f"Invalid age: {input_str!r}")
    value = int(match.group(1))
    if not (0 <= value <= 150):
        raise ValueError(f"Age {value} out of range")
    return value

# ✅ SIMPLE
def parse_age(input_str: str) -> int:
    age = int(input_str.strip())
    if not (0 <= age <= 150):
        raise ValueError(f"Age {age} is out of range (0-150)")
    return age
```

**Benefits:**
- Code is instantly readable by any developer, regardless of experience
- Less code = fewer bugs
- Easier to test, review, and modify

---

## YAGNI — You Aren't Gonna Need It

### Theory

> *"Always implement things when you actually need them, never when you just foresee that you need them."*
> — Ron Jeffries, Extreme Programming

YAGNI is a principle from Extreme Programming (XP) that warns against **speculative generality** — writing code for requirements that *might* appear in the future but don't exist today.

**Why YAGNI violations are costly:**
- You write code no one uses (wasted effort)
- The future requirements may never come, or may look completely different
- The speculative code becomes technical debt others must maintain
- It adds complexity, which can hide bugs in the code you actually need

**YAGNI does NOT mean:**
- Don't plan ahead at all
- Don't write clean code
- Don't refactor

It means: **solve today's problem with today's code**. When the new requirement arrives, refactor then — armed with real knowledge of what's needed.

### ❌ Bad Example — Violating YAGNI

```python
class UserService:
    """
    Developer thought: "We only need user creation today, but we might need
    roles, permissions, 2FA, and OAuth someday. Let me build all that now."
    
    Result: 200 lines of code, most of which is never used.
    """

    SUPPORTED_OAUTH_PROVIDERS = ["google", "github", "facebook", "twitter", "apple"]
    PERMISSION_LEVELS = {
        "viewer": 0, "editor": 1, "manager": 2, "admin": 3, "superadmin": 4
    }

    def __init__(self):
        self.users = {}
        self.roles = {}
        self.permissions = {}
        self.oauth_tokens = {}
        self.two_factor_secrets = {}
        self.audit_log = []

    def create_user(self, name: str, email: str,
                    role: str = "viewer",
                    oauth_provider: str = None,
                    enable_2fa: bool = False) -> dict:
        # The only current requirement is creating a user with name + email.
        # All this extra logic is speculative.
        user_id = len(self.users) + 1
        user = {"id": user_id, "name": name, "email": email}

        # Role system (not required yet)
        if role not in self.PERMISSION_LEVELS:
            raise ValueError(f"Invalid role: {role}")
        user["role"] = role
        user["permission_level"] = self.PERMISSION_LEVELS[role]

        # OAuth (not required yet)
        if oauth_provider:
            if oauth_provider not in self.SUPPORTED_OAUTH_PROVIDERS:
                raise ValueError(f"Unsupported OAuth provider: {oauth_provider}")
            user["oauth_provider"] = oauth_provider

        # 2FA (not required yet)
        if enable_2fa:
            import secrets
            user["2fa_secret"] = secrets.token_hex(16)
            self.two_factor_secrets[user_id] = user["2fa_secret"]

        # Audit logging (not required yet)
        self.audit_log.append({
            "action": "create_user",
            "user_id": user_id,
            "timestamp": "2024-01-01T00:00:00"
        })

        self.users[user_id] = user
        return user

    def verify_2fa_token(self, user_id: int, token: str) -> bool:
        # Nobody asked for this feature. Pure speculation.
        secret = self.two_factor_secrets.get(user_id)
        if not secret:
            return False
        return token == secret[:6]  # Oversimplified, but YAGNI-irrelevant

    def grant_oauth_access(self, user_id: int, provider: str, token: str):
        # Nobody asked for this either.
        self.oauth_tokens[user_id] = {"provider": provider, "token": token}
```

### ✅ Good Example — Following YAGNI

```python
class UserService:
    """
    Today's requirement: create users with name and email.
    That's it. We build exactly that. Nothing more.
    
    When roles are needed → we add them.
    When OAuth is needed → we add it.
    By then, we'll know the REAL requirements.
    """

    def __init__(self):
        self.users = {}

    def create_user(self, name: str, email: str) -> dict:
        if not name or not email:
            raise ValueError("Name and email are required")
        if "@" not in email:
            raise ValueError("Invalid email format")

        user_id = len(self.users) + 1
        user = {"id": user_id, "name": name, "email": email}
        self.users[user_id] = user
        return user

    def get_user(self, user_id: int) -> dict:
        if user_id not in self.users:
            raise KeyError(f"User {user_id} not found")
        return self.users[user_id]


# When roles are needed LATER, we refactor:
# class UserService:
#     def create_user(self, name, email, role="viewer"):
#         ...
```

**Benefits:**
- The codebase only contains code that delivers value today
- When real requirements arrive, you can design based on actual knowledge
- Less code to test, review, and maintain right now

---

## Principles Working Together

The real power comes when these principles are applied together. Here's a real-world scenario:

### Scenario: A payment processing system

```python
from abc import ABC, abstractmethod
from typing import Protocol

# ─── Abstractions (DIP + ISP) ───────────────────────────────
class PaymentGateway(ABC):
    """Interface for payment gateways (DIP)"""
    @abstractmethod
    def charge(self, amount: float, currency: str) -> dict:
        pass

class ReceiptSender(ABC):
    """Separate interface for sending receipts (ISP — not all gateways send receipts)"""
    @abstractmethod
    def send_receipt(self, email: str, amount: float) -> None:
        pass

# ─── Implementations (OCP — add new gateways without modifying anything) ────
class StripeGateway(PaymentGateway):
    def charge(self, amount: float, currency: str) -> dict:
        print(f"Charging ${amount} {currency} via Stripe")
        return {"status": "success", "transaction_id": "stripe_txn_123"}

class PayPalGateway(PaymentGateway):
    def charge(self, amount: float, currency: str) -> dict:
        print(f"Charging ${amount} {currency} via PayPal")
        return {"status": "success", "transaction_id": "paypal_txn_456"}

class EmailReceiptSender(ReceiptSender):
    def send_receipt(self, email: str, amount: float) -> None:
        print(f"Sending receipt for ${amount} to {email}")

# ─── Shared validation (DRY) ───────────────────────────────
def validate_payment(amount: float, currency: str) -> None:
    """Single source of truth for payment validation."""
    if amount <= 0:
        raise ValueError("Payment amount must be positive")
    if currency not in ("USD", "EUR", "GBP"):
        raise ValueError(f"Unsupported currency: {currency}")

# ─── Business Logic (SRP + DIP) ────────────────────────────
class PaymentService:
    """Only one responsibility: orchestrating a payment."""
    def __init__(self, gateway: PaymentGateway, receipt_sender: ReceiptSender):
        self.gateway = gateway
        self.receipt_sender = receipt_sender

    def process_payment(self, amount: float, currency: str, email: str) -> dict:
        validate_payment(amount, currency)           # DRY: shared validation
        result = self.gateway.charge(amount, currency)  # DIP: uses abstraction
        self.receipt_sender.send_receipt(email, amount)
        return result

# ─── Simple, focused usage (KISS + YAGNI) ──────────────────
service = PaymentService(
    gateway=StripeGateway(),
    receipt_sender=EmailReceiptSender()
)
result = service.process_payment(99.99, "USD", "customer@example.com")
```

**How all principles are present:**
| Principle | Applied where |
|-----------|---------------|
| **SRP** | `PaymentService` only orchestrates; validation is separate |
| **OCP** | New gateways (Apple Pay, etc.) added without changing `PaymentService` |
| **LSP** | Any `PaymentGateway` works wherever another is expected |
| **ISP** | `ReceiptSender` is separate from `PaymentGateway` |
| **DIP** | `PaymentService` depends on abstractions, not `StripeGateway` directly |
| **DRY** | `validate_payment` is one function, not copied into each gateway |
| **KISS** | Clean, minimal code — no speculative features |
| **YAGNI** | No multi-currency conversion, no fraud detection — until needed |

---

## Quick Reference Cheat Sheet

```
┌─────────┬────────────────────────────────────────┬─────────────────────────────────────┐
│ Principle│ Core Question                          │ Violation Smell                     │
├─────────┼────────────────────────────────────────┼─────────────────────────────────────┤
│   SRP   │ Does this class have ONE reason to     │ "God class" that does everything     │
│         │ change?                                 │ Hard to name a class concisely       │
├─────────┼────────────────────────────────────────┼─────────────────────────────────────┤
│   OCP   │ Can I add behavior without editing     │ Long if/elif chains on types         │
│         │ existing code?                          │ "Open this class to add a feature"   │
├─────────┼────────────────────────────────────────┼─────────────────────────────────────┤
│   LSP   │ Can I swap any subtype without         │ Subclass throws NotImplementedError  │
│         │ breaking things?                        │ isinstance() checks in core logic    │
├─────────┼────────────────────────────────────────┼─────────────────────────────────────┤
│   ISP   │ Do clients depend only on methods      │ Empty method implementations          │
│         │ they actually use?                      │ Classes implementing methods they    │
│         │                                         │ don't need                           │
├─────────┼────────────────────────────────────────┼─────────────────────────────────────┤
│   DIP   │ Does high-level code depend on         │ new ConcreteClass() inside business  │
│         │ abstractions, not concretions?          │ logic; untestable without real DBs   │
├─────────┼────────────────────────────────────────┼─────────────────────────────────────┤
│   DRY   │ Is every piece of knowledge            │ Copy-paste code, magic numbers,      │
│         │ represented exactly once?               │ same validation in 5 places          │
├─────────┼────────────────────────────────────────┼─────────────────────────────────────┤
│  KISS   │ Is this the simplest solution that     │ Clever one-liners, unnecessary        │
│         │ solves the problem?                     │ abstractions, regex for simple tasks │
├─────────┼────────────────────────────────────────┼─────────────────────────────────────┤
│  YAGNI  │ Is this code needed RIGHT NOW?         │ "We might need this later" features  │
│         │                                         │ Unused parameters, dead code paths   │
└─────────┴────────────────────────────────────────┴─────────────────────────────────────┘
```

---

## Further Reading

- 📖 **Clean Code** — Robert C. Martin
- 📖 **The Pragmatic Programmer** — Andrew Hunt & David Thomas
- 📖 **Design Patterns: Elements of Reusable Object-Oriented Software** — Gang of Four
- 📖 **Refactoring** — Martin Fowler
- 🌐 [SOLID Principles by Uncle Bob](https://blog.cleancoder.com)

---

*This guide is meant to be a living reference. Principles are tools, not laws — always apply judgment based on context.*
