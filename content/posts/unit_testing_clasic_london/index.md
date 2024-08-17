---
title: 'A Deep Dive into Unit Testing: Classic vs. London Schools'
date: 2024-08-08T11:24:15-03:00
draft: false
tags: ["Code Quality", "Code Maintenance", "Programming Principles", "Testing"]

resources:
- name: "featured-image"
  src: "testing_classic_london.png"
---

# A Deep Dive into Unit Testing: Classic vs. London Schools

Unit testing is a fundamental practice in software development, allowing developers to verify the correctness of individual components in isolation. This article explores the theory behind unit testing, presents practical examples in Python, and contrasts the classic and London schools of unit testing, using a complex example for a clear comparison.

## The Example: Order Processing System

### Overview

Our example is an `Order Processing System` that includes the following components:

* **Order**: Represents a customer's order.
* **Inventory**: Manages stock levels for products.
* **PaymentProcessor**: Handles payment transactions.

The `Order` class interacts with both the Inventory and PaymentProcessor to complete a purchase.



### The Inventory Class

```python
class Inventory:
    def __init__(self):
        self.stock = {}

    def add_stock(self, product, quantity):
        if product in self.stock:
            self.stock[product] += quantity
        else:
            self.stock[product] = quantity

    def remove_stock(self, product, quantity):
        if self.stock.get(product, 0) < quantity:
            raise ValueError("Not enough stock")
        self.stock[product] -= quantity

    def has_stock(self, product, quantity):
        return self.stock.get(product, 0) >= quantity
```

### The PaymentProcessor Class

```python
class PaymentProcessor:
    def process_payment(self, amount):
        # Simulate payment processing
        if amount <= 0:
            raise ValueError("Invalid payment amount")
        return True
```

### The Order Class

```python
class Order:
    def __init__(self, inventory, payment_processor):
        self.inventory = inventory
        self.payment_processor = payment_processor
        self.items = []

    def add_item(self, product, quantity):
        if self.inventory.has_stock(product, quantity):
            self.items.append((product, quantity))
        else:
            raise ValueError("Not enough stock to add item")

    def checkout(self):
        total = sum(product.price * quantity for product, quantity in self.items)
        if self.payment_processor.process_payment(total):
            for product, quantity in self.items:
                self.inventory.remove_stock(product, quantity)
            return True
        return False
```

### Product Class (for completeness)

```python
class Product:
    def __init__(self, name, price):
        self.name = name
        self.price = price
```

## Classic Unit Testing

In classic unit testing, the focus is on testing the behavior of the Order class by verifying its outputs and state changes based on inputs. Dependencies (Inventory and PaymentProcessor) are real objects.

### Classic Unit Test Example in Python

```python
import unittest

class TestOrderClassic(unittest.TestCase):
    def setUp(self):
        self.inventory = Inventory()
        self.payment_processor = PaymentProcessor()
        self.order = Order(self.inventory, self.payment_processor)

        self.product1 = Product("Widget", 10.0)
        self.product2 = Product("Gadget", 15.0)
        
        self.inventory.add_stock(self.product1, 10)
        self.inventory.add_stock(self.product2, 5)

    def test_add_item(self):
        self.order.add_item(self.product1, 2)
        self.assertEqual(len(self.order.items), 1)
        self.assertEqual(self.order.items[0], (self.product1, 2))

    def test_checkout_success(self):
        self.order.add_item(self.product1, 2)
        self.order.add_item(self.product2, 1)
        result = self.order.checkout()
        self.assertTrue(result)
        self.assertEqual(self.inventory.stock[self.product1], 8)
        self.assertEqual(self.inventory.stock[self.product2], 4)

    def test_checkout_insufficient_stock(self):
        self.order.add_item(self.product1, 12)  # More than available stock
        with self.assertRaises(ValueError):
            self.order.checkout()

if __name__ == '__main__':
    unittest.main()
```

### Pros of Classic Unit Testing

* **Realistic Scenarios**: Tests use real dependencies, leading to more realistic scenarios.
* **Simplicity**: The test cases are straightforward to understand and implement.
* **Fewer Mocks**: Fewer mocks mean less maintenance and easier debugging.

### Cons of Classic Unit Testing

* **Tight Coupling**: Tests are tightly coupled with real dependencies, which can make it harder to isolate issues.
* **Slower Execution**: Depending on the complexity of the dependencies, tests can be slower.

## London School of Unit Testing (Mockist Testing)

In the London school of unit testing, the emphasis is on testing the Order class in isolation by mocking out the Inventory and PaymentProcessor dependencies. The focus is on ensuring that the Order class interacts correctly with these dependencies.

## London Unit Test Example in Python

```python
import unittest
from unittest.mock import Mock

class TestOrderLondon(unittest.TestCase):
    def setUp(self):
        self.mock_inventory = Mock()
        self.mock_payment_processor = Mock()
        self.order = Order(self.mock_inventory, self.mock_payment_processor)

        self.product1 = Product("Widget", 10.0)
        self.product2 = Product("Gadget", 15.0)

    def test_add_item(self):
        self.mock_inventory.has_stock.return_value = True
        self.order.add_item(self.product1, 2)
        
        self.mock_inventory.has_stock.assert_called_once_with(self.product1, 2)
        self.assertEqual(len(self.order.items), 1)
        self.assertEqual(self.order.items[0], (self.product1, 2))

    def test_checkout_success(self):
        self.order.add_item(self.product1, 2)
        self.order.add_item(self.product2, 1)
        
        self.mock_payment_processor.process_payment.return_value = True
        result = self.order.checkout()
        
        self.mock_payment_processor.process_payment.assert_called_once_with(35.0)
        self.mock_inventory.remove_stock.assert_any_call(self.product1, 2)
        self.mock_inventory.remove_stock.assert_any_call(self.product2, 1)
        self.assertTrue(result)

    def test_checkout_insufficient_stock(self):
        self.mock_inventory.has_stock.side_effect = lambda p, q: p == self.product1 and q <= 10
        self.order.add_item(self.product1, 12)  # Mock will return False for insufficient stock
        
        with self.assertRaises(ValueError):
            self.order.checkout()

if __name__ == '__main__':
    unittest.main()
```

### Pros of London School Unit Testing

* **Isolation**: Each unit is tested in isolation, reducing the potential for side effects.
* **Controlled Environment**: Mocks provide precise control over the behavior of dependencies.
* **Faster Execution**: Tests typically run faster because they avoid real dependencies.

### Cons of London School Unit Testing

* **Complexity**: Writing and managing mocks adds complexity to the test suite.
* **Less Realism**: Tests may not reflect real-world scenarios as closely because of the extensive use of mocks.
* **Brittleness**: Over-mocking can lead to brittle tests that are sensitive to changes in implementation details.

## Comparing the Two Approaches

### Classic Unit Testing

* **Best for**: Scenarios where dependencies are stable and don’t require isolation.
* **Pros**: Simplicity, realistic scenarios, easier maintenance.
* **Cons**: Harder to isolate issues, slower execution due to real dependencies.

### London School Unit Testing

* **Best for**: Complex systems with many dependencies, where isolation is crucial.
* **Pros**: Clear isolation, faster execution, controlled testing environment.
* **Cons**: Increased complexity, potential for brittle tests, less realistic scenarios.

## Conclusion

Unit testing is an essential practice in software development, helping ensure that individual components work correctly. By using the same complex example—a simplified Order Processing System—we’ve demonstrated how both classic and London schools of unit testing address testing challenges differently. Understanding these differences allows developers to choose the right approach for their specific needs.
