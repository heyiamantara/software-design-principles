# SOLID Principles & Software Design Principles
### A Comprehensive Guide with Java Examples

---

## Table of Contents

1. [Introduction](#introduction)
2. [SOLID Principles](#solid-principles)
   - [S — Single Responsibility Principle (SRP)](#1-single-responsibility-principle-srp)
   - [O — Open/Closed Principle (OCP)](#2-openclosed-principle-ocp)
   - [L — Liskov Substitution Principle (LSP)](#3-liskov-substitution-principle-lsp)
   - [I — Interface Segregation Principle (ISP)](#4-interface-segregation-principle-isp)
   - [D — Dependency Inversion Principle (DIP)](#5-dependency-inversion-principle-dip)
3. [Software Design Principles](#software-design-principles)
   - [DRY — Don't Repeat Yourself](#dry--dont-repeat-yourself)
   - [KISS — Keep It Simple, Stupid](#kiss--keep-it-simple-stupid)
   - [YAGNI — You Aren't Gonna Need It](#yagni--you-arent-gonna-need-it)
4. [How These Principles Work Together](#how-these-principles-work-together)
5. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## Introduction

Writing code that works is not enough. Professional software engineers write code that is **readable**, **maintainable**, **scalable**, and **testable**. These goals are achieved by following well-established design principles.

This guide covers:
- **SOLID** — Five object-oriented design principles introduced by Robert C. Martin (Uncle Bob)
- **DRY** — Avoid duplication
- **KISS** — Avoid unnecessary complexity
- **YAGNI** — Avoid speculative features

> 💡 These principles are not strict rules — they are **guidelines** that help you make better design decisions.

---

## SOLID Principles

The acronym **SOLID** stands for:

| Letter | Principle | Core Idea |
|--------|-----------|-----------|
| **S** | Single Responsibility | A class should have only one reason to change |
| **O** | Open/Closed | Open for extension, closed for modification |
| **L** | Liskov Substitution | Subtypes must be substitutable for their base types |
| **I** | Interface Segregation | Don't force clients to depend on interfaces they don't use |
| **D** | Dependency Inversion | Depend on abstractions, not concretions |

---

## 1. Single Responsibility Principle (SRP)

### Theory

> **"A class should have one, and only one, reason to change."**
> — Robert C. Martin

A class should be focused on doing **one thing** and doing it well. When a class has multiple responsibilities, a change in one responsibility can unintentionally break another.

**Why it matters:**
- Reduces coupling between unrelated concerns
- Makes code easier to understand and test
- Isolates the impact of changes
- Enables better code reuse

**Signs of SRP violation:**
- A class name contains "And" (e.g., `UserManagerAndLogger`)
- A class has methods that operate on completely different domains
- A single change requires modifying the same class for multiple reasons

---

### ❌ Bad Example — Violating SRP

```java
// This class has THREE responsibilities:
// 1. Managing user data
// 2. Saving to database
// 3. Sending email notifications
public class UserService {

    public void createUser(String name, String email) {
        // Responsibility 1: Business logic
        if (name == null || name.isEmpty()) {
            throw new IllegalArgumentException("Name cannot be empty");
        }

        // Responsibility 2: Database logic
        String sql = "INSERT INTO users (name, email) VALUES ('" + name + "', '" + email + "')";
        System.out.println("Executing SQL: " + sql); // Simulated DB call

        // Responsibility 3: Email notification logic
        String emailBody = "Welcome, " + name + "! Your account has been created.";
        System.out.println("Sending email to " + email + ": " + emailBody);
    }

    public void deleteUser(int userId) {
        // Responsibility 2: Database logic
        String sql = "DELETE FROM users WHERE id = " + userId;
        System.out.println("Executing SQL: " + sql);

        // Responsibility 3: Email notification logic
        System.out.println("Sending account deletion email to user " + userId);
    }
}
```

**Problems:**
- If the email provider changes, you have to modify `UserService`
- If the DB schema changes, you have to modify `UserService`
- You can't test business logic without the DB and email logic interfering
- The class grows endlessly as the application scales

---

### ✅ Good Example — Following SRP

```java
// Responsibility 1: Business logic only
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;

    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }

    public void createUser(String name, String email) {
        if (name == null || name.isEmpty()) {
            throw new IllegalArgumentException("Name cannot be empty");
        }
        userRepository.save(name, email);
        emailService.sendWelcomeEmail(email, name);
    }

    public void deleteUser(int userId) {
        userRepository.delete(userId);
    }
}

// Responsibility 2: Database operations only
public class UserRepository {
    public void save(String name, String email) {
        String sql = "INSERT INTO users (name, email) VALUES ('" + name + "', '" + email + "')";
        System.out.println("Executing SQL: " + sql);
    }

    public void delete(int userId) {
        String sql = "DELETE FROM users WHERE id = " + userId;
        System.out.println("Executing SQL: " + sql);
    }
}

// Responsibility 3: Email communication only
public class EmailService {
    public void sendWelcomeEmail(String email, String name) {
        String body = "Welcome, " + name + "! Your account has been created.";
        System.out.println("Sending email to " + email + ": " + body);
    }

    public void sendDeletionEmail(String email) {
        System.out.println("Sending deletion confirmation to " + email);
    }
}
```

**Benefits:**
- Each class has exactly **one reason to change**
- You can swap the email provider by only modifying `EmailService`
- You can change the DB by only modifying `UserRepository`
- Each class is independently testable

---

## 2. Open/Closed Principle (OCP)

### Theory

> **"Software entities should be open for extension, but closed for modification."**
> — Bertrand Meyer, popularized by Robert C. Martin

A class should be designed so that new behavior can be **added without changing existing code**. This is typically achieved through **abstraction**, **interfaces**, and **polymorphism**.

**Why it matters:**
- Existing, tested code is not broken when adding new features
- Reduces the risk of introducing bugs in stable code
- Encourages thinking in abstractions
- Makes the system naturally extensible

**Signs of OCP violation:**
- Adding a new feature requires modifying an existing class's core logic
- Long `if-else` or `switch` chains that grow with every new type

---

### ❌ Bad Example — Violating OCP

```java
public class DiscountCalculator {

    // Every time a new customer type is added,
    // we must modify this existing method — VIOLATION!
    public double calculateDiscount(String customerType, double amount) {
        if (customerType.equals("REGULAR")) {
            return amount * 0.05;
        } else if (customerType.equals("PREMIUM")) {
            return amount * 0.10;
        } else if (customerType.equals("VIP")) {
            return amount * 0.20;
        }
        // Adding "STUDENT" requires touching this class again and again!
        return 0;
    }
}
```

---

### ✅ Good Example — Following OCP

```java
// Abstraction: Define the contract
public interface DiscountStrategy {
    double calculate(double amount);
}

// Extension point 1
public class RegularDiscount implements DiscountStrategy {
    @Override
    public double calculate(double amount) {
        return amount * 0.05;
    }
}

// Extension point 2
public class PremiumDiscount implements DiscountStrategy {
    @Override
    public double calculate(double amount) {
        return amount * 0.10;
    }
}

// Extension point 3
public class VIPDiscount implements DiscountStrategy {
    @Override
    public double calculate(double amount) {
        return amount * 0.20;
    }
}

// Adding a new type: NO modification to existing classes!
public class StudentDiscount implements DiscountStrategy {
    @Override
    public double calculate(double amount) {
        return amount * 0.15;
    }
}

// Closed for modification: This class never needs to change
public class DiscountCalculator {
    private final DiscountStrategy strategy;

    public DiscountCalculator(DiscountStrategy strategy) {
        this.strategy = strategy;
    }

    public double calculateDiscount(double amount) {
        return strategy.calculate(amount);
    }
}

// Usage
public class Main {
    public static void main(String[] args) {
        DiscountCalculator calc = new DiscountCalculator(new VIPDiscount());
        System.out.println("VIP Discount: " + calc.calculateDiscount(1000)); // 200.0

        // Easily swap strategies
        DiscountCalculator studentCalc = new DiscountCalculator(new StudentDiscount());
        System.out.println("Student Discount: " + studentCalc.calculateDiscount(1000)); // 150.0
    }
}
```

---

## 3. Liskov Substitution Principle (LSP)

### Theory

> **"Objects of a superclass should be replaceable with objects of its subclasses without breaking the application."**
> — Barbara Liskov, 1987

If `S` is a subtype of `T`, then objects of type `T` may be replaced with objects of type `S` **without altering any of the desirable properties of the program**.

In simpler terms: **a child class should be able to do everything its parent can do**, and should not weaken the contract of the parent.

**Why it matters:**
- Guarantees the correctness of polymorphism
- Prevents unexpected behavior when using base class references
- Builds trust in inheritance hierarchies
- Makes code predictable and reliable

**Signs of LSP violation:**
- Subclass overrides a method by throwing `UnsupportedOperationException`
- Subclass narrows the behavior of a parent method
- Code checks `instanceof` before calling a method

---

### ❌ Bad Example — Violating LSP

```java
public class Rectangle {
    protected int width;
    protected int height;

    public void setWidth(int width) {
        this.width = width;
    }

    public void setHeight(int height) {
        this.height = height;
    }

    public int getArea() {
        return width * height;
    }
}

// A Square "is-a" Rectangle? Seems logical... but it breaks LSP!
public class Square extends Rectangle {

    @Override
    public void setWidth(int width) {
        // A square must keep width == height
        this.width = width;
        this.height = width; // Side effect! Violates parent's contract
    }

    @Override
    public void setHeight(int height) {
        this.width = height; // Side effect!
        this.height = height;
    }
}

// This test passes for Rectangle, but FAILS for Square!
public class LSPViolationTest {
    public static void testRectangle(Rectangle r) {
        r.setWidth(5);
        r.setHeight(4);
        // Expected area = 20, but for Square: 4*4 = 16 — WRONG!
        System.out.println("Area: " + r.getArea()); // Fails for Square
    }
}
```

---

### ✅ Good Example — Following LSP

```java
// Common abstraction — no broken contracts
public interface Shape {
    int getArea();
}

public class Rectangle implements Shape {
    private final int width;
    private final int height;

    public Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }

    @Override
    public int getArea() {
        return width * height;
    }
}

public class Square implements Shape {
    private final int side;

    public Square(int side) {
        this.side = side;
    }

    @Override
    public int getArea() {
        return side * side;
    }
}

// Now both can be used interchangeably via the Shape interface
public class AreaPrinter {
    public static void printArea(Shape shape) {
        System.out.println("Area: " + shape.getArea()); // Always correct!
    }

    public static void main(String[] args) {
        printArea(new Rectangle(5, 4)); // Area: 20
        printArea(new Square(4));       // Area: 16 — correct for a square!
    }
}
```

**Another Classic LSP Example — Birds:**

```java
// BAD: Penguin can't fly, but it's forced to inherit fly()
public class Bird {
    public void fly() {
        System.out.println("Flying...");
    }
}
public class Penguin extends Bird {
    @Override
    public void fly() {
        throw new UnsupportedOperationException("Penguins can't fly!"); // LSP broken!
    }
}

// GOOD: Segregate behaviors properly
public interface Bird {
    void eat();
    void sleep();
}

public interface FlyingBird extends Bird {
    void fly();
}

public class Eagle implements FlyingBird {
    public void fly()   { System.out.println("Eagle soaring..."); }
    public void eat()   { System.out.println("Eagle eating..."); }
    public void sleep() { System.out.println("Eagle sleeping..."); }
}

public class Penguin implements Bird {
    public void eat()   { System.out.println("Penguin eating fish..."); }
    public void sleep() { System.out.println("Penguin sleeping..."); }
    // No fly() — correct! Penguin doesn't need it.
}
```

---

## 4. Interface Segregation Principle (ISP)

### Theory

> **"A client should not be forced to depend on interfaces it does not use."**
> — Robert C. Martin

Large, fat interfaces force implementing classes to provide implementations for methods they don't need. Instead, interfaces should be **small, focused, and cohesive** — clients should only know about the methods that are relevant to them.

**Why it matters:**
- Prevents implementing classes from carrying unnecessary method baggage
- Reduces the impact of interface changes
- Produces leaner, more focused abstractions
- Naturally aligns with SRP at the interface level

**Signs of ISP violation:**
- Classes implement an interface but leave some methods empty or throw exceptions
- An interface name is very generic (e.g., `Worker`, `Manager`)
- Adding a method to an interface forces changes across many unrelated classes

---

### ❌ Bad Example — Violating ISP

```java
// A "fat" interface that tries to do everything
public interface Worker {
    void work();
    void eat();
    void sleep();
    void attendMeeting();
    void writeCode();
    void designUI();
}

// A Developer has to implement ALL methods
public class Developer implements Worker {
    public void work()          { System.out.println("Developer working..."); }
    public void eat()           { System.out.println("Developer eating..."); }
    public void sleep()         { System.out.println("Developer sleeping..."); }
    public void attendMeeting() { System.out.println("Attending standup..."); }
    public void writeCode()     { System.out.println("Writing Java code..."); }
    public void designUI()      {
        // A backend developer doesn't design UI!
        throw new UnsupportedOperationException("Not my job!"); // BAD!
    }
}

// A Robot worker doesn't eat or sleep but is forced to implement it!
public class RobotWorker implements Worker {
    public void work()          { System.out.println("Robot working 24/7..."); }
    public void eat()           { throw new UnsupportedOperationException("Robots don't eat!"); }
    public void sleep()         { throw new UnsupportedOperationException("Robots don't sleep!"); }
    public void attendMeeting() { throw new UnsupportedOperationException("Robots don't meet!"); }
    public void writeCode()     { System.out.println("Robot generating code..."); }
    public void designUI()      { System.out.println("Robot designing UI..."); }
}
```

---

### ✅ Good Example — Following ISP

```java
// Segregated, focused interfaces
public interface Workable {
    void work();
}

public interface Eatable {
    void eat();
}

public interface Sleepable {
    void sleep();
}

public interface Codeable {
    void writeCode();
}

public interface UIDesignable {
    void designUI();
}

// Backend Developer: only implements what's relevant
public class BackendDeveloper implements Workable, Eatable, Sleepable, Codeable {
    public void work()      { System.out.println("Backend developer working..."); }
    public void eat()       { System.out.println("Eating lunch at desk..."); }
    public void sleep()     { System.out.println("Getting 8 hours of sleep..."); }
    public void writeCode() { System.out.println("Writing Spring Boot APIs..."); }
}

// UI Designer: only implements what's relevant
public class UIDesigner implements Workable, Eatable, Sleepable, UIDesignable {
    public void work()     { System.out.println("UI designer working..."); }
    public void eat()      { System.out.println("Eating at the cafeteria..."); }
    public void sleep()    { System.out.println("Sleeping..."); }
    public void designUI() { System.out.println("Designing Figma mockups..."); }
}

// Robot: only implements what it actually does
public class RobotWorker implements Workable, Codeable, UIDesignable {
    public void work()     { System.out.println("Robot working non-stop..."); }
    public void writeCode(){ System.out.println("Robot auto-generating code..."); }
    public void designUI() { System.out.println("Robot generating UI layouts..."); }
    // No eat(), no sleep() — clean and honest!
}
```

---

## 5. Dependency Inversion Principle (DIP)

### Theory

> **"High-level modules should not depend on low-level modules. Both should depend on abstractions. Abstractions should not depend on details. Details should depend on abstractions."**
> — Robert C. Martin

This principle has two key rules:
1. **High-level modules** (business logic) should not import **low-level modules** (DB, email, file system) directly
2. Both should depend on **interfaces/abstractions**, not concrete implementations

**Why it matters:**
- Decouples business logic from infrastructure concerns
- Makes the system easy to test (swap real DB with mock)
- Makes switching implementations trivial (e.g., MySQL → MongoDB)
- Aligns with the architecture of clean/hexagonal architecture

**Signs of DIP violation:**
- A service class directly instantiates a repository: `new UserRepository()`
- Business logic imports low-level classes directly
- Changing a DB requires modifying business logic classes

---

### ❌ Bad Example — Violating DIP

```java
// Low-level module
public class MySQLDatabase {
    public void save(String data) {
        System.out.println("Saving to MySQL: " + data);
    }
}

// High-level module directly depends on a concrete low-level class — VIOLATION!
public class OrderService {
    private MySQLDatabase database; // Tightly coupled to MySQL!

    public OrderService() {
        this.database = new MySQLDatabase(); // High-level creates low-level — BAD!
    }

    public void placeOrder(String orderDetails) {
        System.out.println("Processing order: " + orderDetails);
        database.save(orderDetails); // Can't swap to MongoDB without changing this class!
    }
}
```

---

### ✅ Good Example — Following DIP

```java
// Abstraction: the contract both sides depend on
public interface Database {
    void save(String data);
}

// Low-level module 1: implements the abstraction
public class MySQLDatabase implements Database {
    @Override
    public void save(String data) {
        System.out.println("Saving to MySQL: " + data);
    }
}

// Low-level module 2: another implementation
public class MongoDatabase implements Database {
    @Override
    public void save(String data) {
        System.out.println("Saving to MongoDB: " + data);
    }
}

// Mock for testing: yet another implementation
public class MockDatabase implements Database {
    @Override
    public void save(String data) {
        System.out.println("[TEST] Mock save: " + data);
    }
}

// High-level module: depends on abstraction, NOT on a concrete class
public class OrderService {
    private final Database database; // Depends on interface!

    // Dependency is INJECTED from outside (Dependency Injection)
    public OrderService(Database database) {
        this.database = database;
    }

    public void placeOrder(String orderDetails) {
        System.out.println("Processing order: " + orderDetails);
        database.save(orderDetails);
    }
}

// Usage: You can swap any implementation with zero changes to OrderService
public class Main {
    public static void main(String[] args) {
        // Production: use MySQL
        OrderService prodService = new OrderService(new MySQLDatabase());
        prodService.placeOrder("Order #1001");

        // Migrate to MongoDB: just swap here
        OrderService mongoService = new OrderService(new MongoDatabase());
        mongoService.placeOrder("Order #1002");

        // Testing: use mock
        OrderService testService = new OrderService(new MockDatabase());
        testService.placeOrder("Order #TEST");
    }
}
```

---

## Software Design Principles

---

## DRY — Don't Repeat Yourself

### Theory

> **"Every piece of knowledge must have a single, unambiguous, authoritative representation within a system."**
> — Andrew Hunt & David Thomas, *The Pragmatic Programmer*

DRY means: **do not write the same logic in more than one place**. Duplication is the root cause of many bugs — when you need to change something, you have to find and update every copy. Miss one, and you have a bug.

**DRY is about knowledge, not just code:**
- Duplicated logic → extract into a method or class
- Duplicated configuration → use constants
- Duplicated structure → use generics or templates
- Duplicated documentation → write it once and reference it

**When DRY can go too far:**
- Don't over-abstract unrelated code just because it looks similar today
- Two pieces of code that do similar things for different reasons should stay separate (see YAGNI)

---

### ❌ Bad Example — Violating DRY

```java
public class ReportGenerator {

    public void generateSalesReport(List<Double> sales) {
        // Validation logic duplicated
        if (sales == null || sales.isEmpty()) {
            System.out.println("No data available.");
            return;
        }
        double total = 0;
        for (double sale : sales) total += sale;
        double average = total / sales.size();
        System.out.println("=== Sales Report ===");
        System.out.println("Total: " + total);
        System.out.println("Average: " + average);
    }

    public void generateExpenseReport(List<Double> expenses) {
        // SAME validation logic duplicated again!
        if (expenses == null || expenses.isEmpty()) {
            System.out.println("No data available.");
            return;
        }
        // SAME calculation logic duplicated!
        double total = 0;
        for (double expense : expenses) total += expense;
        double average = total / expenses.size();
        System.out.println("=== Expense Report ===");
        System.out.println("Total: " + total);
        System.out.println("Average: " + average);
    }

    public void generateInventoryReport(List<Double> values) {
        // SAME validation and calculation — THIRD COPY!
        if (values == null || values.isEmpty()) {
            System.out.println("No data available.");
            return;
        }
        double total = 0;
        for (double value : values) total += value;
        double average = total / values.size();
        System.out.println("=== Inventory Report ===");
        System.out.println("Total: " + total);
        System.out.println("Average: " + average);
    }
}
```

---

### ✅ Good Example — Following DRY

```java
public class ReportGenerator {

    // Single source of truth for validation
    private boolean isValidData(List<Double> data) {
        return data != null && !data.isEmpty();
    }

    // Single source of truth for calculation
    private double calculateTotal(List<Double> data) {
        return data.stream().mapToDouble(Double::doubleValue).sum();
    }

    private double calculateAverage(List<Double> data) {
        return calculateTotal(data) / data.size();
    }

    // Single source of truth for report printing
    private void printReport(String title, List<Double> data) {
        if (!isValidData(data)) {
            System.out.println("No data available.");
            return;
        }
        System.out.println("=== " + title + " ===");
        System.out.println("Total: "   + calculateTotal(data));
        System.out.println("Average: " + calculateAverage(data));
    }

    public void generateSalesReport(List<Double> sales)         { printReport("Sales Report",     sales);    }
    public void generateExpenseReport(List<Double> expenses)    { printReport("Expense Report",   expenses); }
    public void generateInventoryReport(List<Double> values)    { printReport("Inventory Report", values);   }
}
```

**Now if the calculation logic or validation changes, you update it in ONE place.**

---

### DRY with Constants

```java
// BAD: Magic numbers scattered everywhere
public class OrderValidator {
    public boolean isValidOrder(int quantity, double price) {
        return quantity > 0 && quantity <= 1000 && price >= 0.01 && price <= 99999.99;
    }
}

public class CartValidator {
    public boolean isValidCart(int itemCount, double totalPrice) {
        return itemCount > 0 && itemCount <= 1000 && totalPrice >= 0.01; // 1000 and 0.01 again!
    }
}

// GOOD: Constants defined once
public final class BusinessConstants {
    public static final int    MAX_QUANTITY = 1000;
    public static final double MIN_PRICE    = 0.01;
    public static final double MAX_PRICE    = 99999.99;

    private BusinessConstants() {} // Prevent instantiation
}

public class OrderValidator {
    public boolean isValidOrder(int quantity, double price) {
        return quantity > 0
            && quantity <= BusinessConstants.MAX_QUANTITY
            && price   >= BusinessConstants.MIN_PRICE
            && price   <= BusinessConstants.MAX_PRICE;
    }
}

public class CartValidator {
    public boolean isValidCart(int itemCount, double totalPrice) {
        return itemCount   > 0
            && itemCount   <= BusinessConstants.MAX_QUANTITY
            && totalPrice  >= BusinessConstants.MIN_PRICE;
    }
}
```

---

## KISS — Keep It Simple, Stupid

### Theory

> **"Simplicity is the ultimate sophistication."**
> — Leonardo da Vinci (often applied in software engineering)

> **"Everything should be made as simple as possible, but not simpler."**
> — Albert Einstein

KISS says: **don't over-engineer**. Most systems work best when they are kept simple rather than made complex. Complexity is the enemy of reliability, maintainability, and readability.

**What KISS is NOT:**
- It doesn't mean "write sloppy code"
- It doesn't mean "avoid abstraction always"
- It doesn't mean "use trivial solutions that don't scale"

**KISS means:**
- Prefer the simplest solution that solves the problem correctly
- Avoid clever tricks that obscure intent
- Write code for the next developer to read, not to impress

---

### ❌ Bad Example — Violating KISS

```java
// Over-engineered: excessive abstraction, indirection, and complexity for a simple task
public interface NumberTransformer<T, R> {
    R transform(T input);
}

public abstract class AbstractNumberProcessor<T extends Number> {
    protected abstract NumberTransformer<T, String> getTransformerStrategy();

    public String process(T number) {
        return getTransformerStrategy().transform(number);
    }
}

public class ConcreteEvenOddProcessor extends AbstractNumberProcessor<Integer> {
    @Override
    protected NumberTransformer<Integer, String> getTransformerStrategy() {
        return (number) -> {
            if ((number & 1) == 0) { // Bitwise trick — clever but unclear
                return "EVEN";
            } else {
                return "ODD";
            }
        };
    }
}

// Usage: 3 classes just to check if a number is even or odd!
public class Main {
    public static void main(String[] args) {
        AbstractNumberProcessor<Integer> processor = new ConcreteEvenOddProcessor();
        System.out.println(processor.process(4)); // EVEN
    }
}
```

---

### ✅ Good Example — Following KISS

```java
// Simple, clear, and direct — does exactly what it needs to
public class NumberUtils {
    public static String getParityLabel(int number) {
        return (number % 2 == 0) ? "EVEN" : "ODD";
    }
}

// Usage: clean, readable, zero ceremony
public class Main {
    public static void main(String[] args) {
        System.out.println(NumberUtils.getParityLabel(4)); // EVEN
        System.out.println(NumberUtils.getParityLabel(7)); // ODD
    }
}
```

---

### Another KISS Example — Password Validation

```java
// BAD: Hard-to-read regex that nobody can maintain or debug
public class PasswordValidator {
    private static final String COMPLEXITY_PATTERN =
        "^(?=(?:.*[A-Z]){1,})(?=(?:.*[a-z]){1,})(?=(?:.*\\d){1,})(?=(?:.*[!@#$%]){1,}).{8,}$";

    public boolean validate(String password) {
        return password != null && password.matches(COMPLEXITY_PATTERN);
        // Changing one rule means rewriting the entire regex
    }
}

// GOOD: Step-by-step validation — readable, debuggable, easy to extend
public class PasswordValidator {

    public boolean validate(String password) {
        if (password == null || password.length() < 8) return false;

        boolean hasUpperCase   = password.chars().anyMatch(Character::isUpperCase);
        boolean hasLowerCase   = password.chars().anyMatch(Character::isLowerCase);
        boolean hasDigit       = password.chars().anyMatch(Character::isDigit);
        boolean hasSpecialChar = password.chars().anyMatch(c -> "!@#$%".indexOf(c) >= 0);

        return hasUpperCase && hasLowerCase && hasDigit && hasSpecialChar;
    }
}
```

**The second version is readable, easily debuggable, and simple to modify.**

---

## YAGNI — You Aren't Gonna Need It

### Theory

> **"Always implement things when you actually need them, never when you just foresee that you might need them."**
> — Ron Jeffries, Extreme Programming

YAGNI means: **don't write code for features you think you'll need in the future, but don't need right now**. Speculative code adds complexity without delivering value.

**Why YAGNI matters:**
- Unused code still needs to be read, tested, and maintained
- Speculative features are often built wrong (requirements change)
- It leads to bloated, confusing codebases
- Time spent on future features is time NOT spent on current needs

**YAGNI is different from DRY:**
- DRY removes duplication of existing requirements
- YAGNI avoids implementing requirements that don't exist yet

**When to apply YAGNI:**
- Every time you think "I might need this later..."
- When adding configuration options no one asked for
- When building a plugin system before the first plugin exists

---

### ❌ Bad Example — Violating YAGNI

```java
// Current requirement: store and retrieve user name and email.
// But the developer builds for "future needs" nobody asked for!
public class User {
    private String name;
    private String email;

    // YAGNI violation: No one asked for multiple roles yet
    private List<String> roles = new ArrayList<>();

    // YAGNI violation: No one asked for OAuth integration yet
    private String oauthProvider;
    private String oauthToken;

    // YAGNI violation: No one asked for audit trail
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private String createdBy;
    private String updatedBy;

    // YAGNI violation: Nobody needs export to CSV, XML, JSON all at once
    public String toCsv() {
        return name + "," + email;
    }

    public String toXml() {
        return "<user><name>" + name + "</name><email>" + email + "</email></user>";
    }

    public String toJson() {
        return "{\"name\":\"" + name + "\",\"email\":\"" + email + "\"}";
    }

    // YAGNI violation: no requirement for multi-language support yet
    public String getWelcomeMessage(String language) {
        if (language.equals("en")) return "Welcome, " + name;
        if (language.equals("es")) return "Bienvenido, " + name;
        if (language.equals("fr")) return "Bienvenue, " + name;
        return "Welcome, " + name;
    }
}
```

---

### ✅ Good Example — Following YAGNI

```java
// Only build what's needed NOW.
// Extend later when requirements are actually confirmed.
public class User {
    private String name;
    private String email;

    public User(String name, String email) {
        this.name = name;
        this.email = email;
    }

    public String getName()  { return name; }
    public String getEmail() { return email; }
}

// Later, when the requirement for roles actually arrives,
// you add it — and you'll do it right because you know the real use case.
public class User {
    private String name;
    private String email;
    private List<String> roles; // Added ONLY when actually required

    // ...
}
```

---

### YAGNI in Architecture

```java
// BAD: Building a plugin system before the first plugin exists
public interface Plugin {
    void init(PluginContext context);
    void execute(Map<String, Object> params);
    void destroy();
    String getName();
    String getVersion();
    List<String> getDependencies();
}

public class PluginRegistry {
    private Map<String, Plugin> plugins = new HashMap<>();
    // Complex registration, lifecycle management, dependency resolution...
    // For a feature that hasn't been asked for!
}

// GOOD: Just solve today's problem simply
public class DataProcessor {
    public void process(List<String> data) {
        // Direct implementation that works for the current use case
        data.stream()
            .filter(s -> !s.isEmpty())
            .map(String::toUpperCase)
            .forEach(System.out::println);
    }
}
// Refactor into a plugin system WHEN multiple pluggable processors are actually needed.
```

---

## How These Principles Work Together

These principles are complementary, not competing:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   SRP   ──► Each class does ONE thing well                      │
│   OCP   ──► Add features WITHOUT changing existing code         │
│   LSP   ──► Subtypes are safe substitutes for supertypes        │
│   ISP   ──► Interfaces are lean and focused                     │
│   DIP   ──► Depend on abstractions, not concretions             │
│                                                                 │
│   DRY   ──► No duplicated logic anywhere                        │
│   KISS  ──► No unnecessary complexity                           │
│   YAGNI ──► No speculative future features                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**A real-world example combining all principles:**

```java
// SRP:   Each class has one job
// DIP:   Depend on abstractions, not concretions
// OCP:   Easy to add new notification channels without touching existing code
// ISP:   NotificationSender is a small, focused interface
// DRY:   Notification logic is in one place
// KISS:  Design is straightforward
// YAGNI: No extra features until they're requested

public interface NotificationSender {          // ISP: focused interface
    void send(String recipient, String message);
}

public class EmailSender implements NotificationSender {   // SRP: only sends email
    @Override
    public void send(String recipient, String message) {
        System.out.println("Email to " + recipient + ": " + message);
    }
}

public class SmsSender implements NotificationSender {     // OCP: new type, no old code changed
    @Override
    public void send(String recipient, String message) {
        System.out.println("SMS to " + recipient + ": " + message);
    }
}

public class NotificationService {                         // SRP: orchestrates notifications
    private final NotificationSender sender;               // DIP: depends on abstraction

    public NotificationService(NotificationSender sender) {
        this.sender = sender;
    }

    public void notifyUser(String userId, String message) { // DRY: single place for notify logic
        String recipient = resolveRecipient(userId);
        sender.send(recipient, message);                    // KISS: simple and direct
    }

    private String resolveRecipient(String userId) {
        return userId + "@example.com";                     // YAGNI: no complex lookup until needed
    }
}
```

---

## Summary Cheat Sheet

| Principle | One-Line Question to Ask | How to Fix |
|-----------|--------------------------|------------|
| **SRP** | Does this class have more than one reason to change? | Split into focused classes |
| **OCP** | Do I need to modify existing code to add a new feature? | Use abstractions and polymorphism |
| **LSP** | Can I replace a parent class with its child without breaking things? | Fix inheritance hierarchy or use composition |
| **ISP** | Is a class forced to implement methods it doesn't need? | Split the fat interface into smaller ones |
| **DIP** | Does high-level code directly depend on low-level code? | Introduce an interface; inject dependencies |
| **DRY** | Is the same logic written in more than one place? | Extract to a shared method, class, or constant |
| **KISS** | Is this solution more complex than the problem requires? | Simplify; favor clarity over cleverness |
| **YAGNI** | Am I building something nobody asked for yet? | Delete it; add it when the requirement arrives |

---

> 💡 **Final Thought:** These principles exist to serve you — not to be followed blindly. The goal is **clean, maintainable, understandable code** that solves real problems. Apply these principles with judgment, and they will serve you well throughout your entire software engineering career.
