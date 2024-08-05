+++
title = "DRY vs WET Code: A Comparative Study with Python and Go"
date = 2024-08-05T15:09:19-03:00
draft = false
tags = ["DRY", "Code Quality", "Clean Code", "Programming Principles", "Software Design"]
+++

# DRY vs WET Code: A Comparative Study with Python and Go

## Introduction

The DRY principle, which stands for "Don't Repeat Yourself," is a fundamental concept in software development aimed at reducing repetition of software patterns, replacing it with abstractions or using data normalization to avoid redundancy. This article provides an in-depth look at the DRY principle, its implementation in Python and Go, and its application in software design using UML diagrams.

## Understanding DRY

The DRY principle was formulated by Andy Hunt and Dave Thomas in their book "The Pragmatic Programmer." It states:

> "Every piece of knowledge must have a single, unambiguous, authoritative representation within a system."

This means that in a well-designed system, each piece of functionality should be implemented in just one place. The goal is to avoid duplicating code and data, which can lead to maintenance nightmares, poor factoring, and logical contradictions.

## Benefits of DRY

1. **Improved Maintainability**: Changes need to be made in only one place.
2. **Reduced Complexity**: Less code generally means less complexity.
3. **Enhanced Readability**: Code is more concise and often clearer.
4. **Easier Testing**: With functionality centralized, testing becomes more straightforward.
5. **Better Scalability**: DRY code is often more modular and easier to scale.

## DRY vs WET Code

WET stands for "Write Everything Twice" or "We Enjoy Typing." It's the antithesis of DRY. Let's compare:

```python
# WET code
def validate_email(email):
    if '@' in email and '.' in email:
        return True
    return False

def validate_user_email(user_email):
    if '@' in user_email and '.' in user_email:
        return True
    return False

# DRY code
def validate_email(email):
    return '@' in email and '.' in email

# Now we can use validate_email for both regular emails and user emails
```

In the WET example, we have two nearly identical functions. In the DRY version, we have a single function that can be reused.

## Implementing DRY in Python

Let's look at a more complex example in Python, this time using a banking system:

```python
# WET code
class SavingsAccount:
    def __init__(self, account_number, balance):
        self.account_number = account_number
        self.balance = balance

    def deposit(self, amount):
        if amount > 0:
            self.balance += amount
            print(f"Deposited ${amount}. New balance: ${self.balance}")
        else:
            print("Invalid deposit amount")

    def withdraw(self, amount):
        if 0 < amount <= self.balance:
            self.balance -= amount
            print(f"Withdrawn ${amount}. New balance: ${self.balance}")
        else:
            print("Invalid withdrawal amount")

class CheckingAccount:
    def __init__(self, account_number, balance):
        self.account_number = account_number
        self.balance = balance

    def deposit(self, amount):
        if amount > 0:
            self.balance += amount
            print(f"Deposited ${amount}. New balance: ${self.balance}")
        else:
            print("Invalid deposit amount")

    def withdraw(self, amount):
        if 0 < amount <= self.balance:
            self.balance -= amount
            print(f"Withdrawn ${amount}. New balance: ${self.balance}")
        else:
            print("Invalid withdrawal amount")

# DRY code
class BankAccount:
    def __init__(self, account_number, balance):
        self.account_number = account_number
        self.balance = balance

    def deposit(self, amount):
        if amount > 0:
            self.balance += amount
            print(f"Deposited ${amount}. New balance: ${self.balance}")
        else:
            print("Invalid deposit amount")

    def withdraw(self, amount):
        if 0 < amount <= self.balance:
            self.balance -= amount
            print(f"Withdrawn ${amount}. New balance: ${self.balance}")
        else:
            print("Invalid withdrawal amount")

class SavingsAccount(BankAccount):
    def __init__(self, account_number, balance, interest_rate):
        super().__init__(account_number, balance)
        self.interest_rate = interest_rate

    def apply_interest(self):
        interest = self.balance * self.interest_rate
        self.deposit(interest)
        print(f"Applied interest: ${interest}")

class CheckingAccount(BankAccount):
    def __init__(self, account_number, balance, overdraft_limit):
        super().__init__(account_number, balance)
        self.overdraft_limit = overdraft_limit

    def withdraw(self, amount):
        if 0 < amount <= (self.balance + self.overdraft_limit):
            self.balance -= amount
            print(f"Withdrawn ${amount}. New balance: ${self.balance}")
        else:
            print("Invalid withdrawal amount or overdraft limit exceeded")
```

In this DRY version, we create a base `BankAccount` class with common functionality, and then create `SavingsAccount` and `CheckingAccount` as subclasses. This eliminates code duplication and allows for easy extension with account-specific features.

## Implementing DRY in Go

Now let's see how we can apply DRY in Go using a different example - a simple task management system:

```go
// WET code
type WorkTask struct {
    ID          int
    Description string
    IsCompleted bool
}

func (t *WorkTask) MarkAsComplete() {
    t.IsCompleted = true
    fmt.Printf("Work task %d marked as complete\n", t.ID)
}

func (t *WorkTask) MarkAsIncomplete() {
    t.IsCompleted = false
    fmt.Printf("Work task %d marked as incomplete\n", t.ID)
}

type PersonalTask struct {
    ID          int
    Description string
    IsCompleted bool
}

func (t *PersonalTask) MarkAsComplete() {
    t.IsCompleted = true
    fmt.Printf("Personal task %d marked as complete\n", t.ID)
}

func (t *PersonalTask) MarkAsIncomplete() {
    t.IsCompleted = false
    fmt.Printf("Personal task %d marked as incomplete\n", t.ID)
}

// DRY code
type Task interface {
    MarkAsComplete()
    MarkAsIncomplete()
    GetDescription() string
}

type BaseTask struct {
    ID          int
    Description string
    IsCompleted bool
}

func (t *BaseTask) MarkAsComplete() {
    t.IsCompleted = true
}

func (t *BaseTask) MarkAsIncomplete() {
    t.IsCompleted = false
}

func (t *BaseTask) GetDescription() string {
    return t.Description
}

type WorkTask struct {
    BaseTask
}

func (t *WorkTask) MarkAsComplete() {
    t.BaseTask.MarkAsComplete()
    fmt.Printf("Work task %d marked as complete\n", t.ID)
}

func (t *WorkTask) MarkAsIncomplete() {
    t.BaseTask.MarkAsIncomplete()
    fmt.Printf("Work task %d marked as incomplete\n", t.ID)
}

type PersonalTask struct {
    BaseTask
}

func (t *PersonalTask) MarkAsComplete() {
    t.BaseTask.MarkAsComplete()
    fmt.Printf("Personal task %d marked as complete\n", t.ID)
}

func (t *PersonalTask) MarkAsIncomplete() {
    t.BaseTask.MarkAsIncomplete()
    fmt.Printf("Personal task %d marked as incomplete\n", t.ID)
}
```

In this DRY version, we define a `Task` interface and a `BaseTask` struct with common functionality. The `WorkTask` and `PersonalTask` types embed `BaseTask` and implement the `Task` interface, allowing for code reuse while maintaining type-specific behavior.

## DRY in Software Design: UML Examples

Let's update our UML diagram to reflect the new banking system example:

```
┌───────────────┐
│ BankAccount   │
├───────────────┤
│-accountNumber │
│-balance       │
├───────────────┤
│+deposit()     │
│+withdraw()    │
└───────────────┘
      ▲
      │
      ├───────────────┐
      │               │
┌─────────────────┐ ┌───────────────┐
│SavingsAccount   │ │CheckingAccount│
├─────────────────┤ ├───────────────┤
│-interestRate    │ │-overdraftLimit│
├─────────────────┤ ├───────────────┤
│+applyInterest() │ │+withdraw()    │
└─────────────────┘ └───────────────┘
```

This UML diagram shows how the `SavingsAccount` and `CheckingAccount` classes inherit from the `BankAccount` base class, demonstrating the DRY principle in the class hierarchy.

## Common Pitfalls and Misconceptions

1. **Over-DRYing**: Sometimes, trying to eliminate all repetition can lead to overly complex abstractions.
2. **Premature Abstraction**: Applying DRY before fully understanding the problem domain can result in incorrect abstractions.
3. **Ignoring Semantic Differences**: Two pieces of code may look similar but serve different purposes. Blindly combining them might be a mistake.

## Best Practices for Applying DRY

1. **Identify Common Patterns**: Look for recurring code or logic in your system.
2. **Create Abstractions Thoughtfully**: Ensure your abstractions truly represent the problem domain.
3. **Use Inheritance and Composition**: Leverage OOP principles to avoid repetition.
4. **Utilize Functions and Methods**: Extract common logic into reusable functions.
5. **Apply DRY to Data**: Use normalization techniques in databases to avoid data redundancy.
6. **Refactor Regularly**: Continuously look for opportunities to apply DRY as your codebase evolves.

## Conclusion

The DRY principle is a powerful tool for creating maintainable, scalable, and efficient code. By avoiding repetition and creating meaningful abstractions, developers can significantly improve the quality of their software. However, it's crucial to apply DRY thoughtfully, always considering the specific context and requirements of your project. Remember, the goal is not to eliminate all repetition, but to ensure that every piece of knowledge has a single, authoritative representation in your system.