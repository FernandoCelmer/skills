---
name: python-unittest
description: Generate Python unit tests using unittest.TestCase classes for selected functions or methods. Use when the user asks to write, generate, add, or scaffold unit tests for Python code — never emit pytest, plain test functions, or third-party mocking libraries.
version: 1.0.0
allowed-tools: Read, Edit, Write, Glob, Grep, Bash
---

# Task

Analyze the selected Python function or method and generate focused, production-quality unit tests using Python's built-in `unittest` framework.

Tests must be organized into classes that inherit from `unittest.TestCase`.

The goal is to validate the public behavior of the target while keeping tests isolated, deterministic, readable, and easy to maintain.

## Testing Framework

Use exclusively:

```python
import unittest
```

For mocks, patches, spies, and dependency isolation, use:

```python
from unittest.mock import Mock, MagicMock, AsyncMock, patch, call
```

Do not use:

* pytest
* pytest fixtures
* pytest.mark
* monkeypatch
* nose
* third-party mocking libraries
* bare test functions

Every test must belong to a `unittest.TestCase` class.

Example:

```python
import unittest
from unittest.mock import patch

from project.module import calculate_total


class TestCalculateTotal(unittest.TestCase):

    def test_returns_total_for_valid_items(self):
        result = calculate_total([
            {"price": 10.0, "quantity": 2},
            {"price": 5.0, "quantity": 1},
        ])

        self.assertEqual(result, 25.0)


if __name__ == "__main__":
    unittest.main()
```

## Test Generation Strategy

### 1. Understand the Target

Before generating tests:

1. Read the complete target function or method.
2. Identify its inputs and outputs.
3. Identify dependencies it calls.
4. Identify possible exceptions.
5. Identify state mutations or side effects.
6. Identify branches and important conditions.
7. Inspect surrounding code when necessary to understand expected behavior.
8. Inspect existing tests to follow the project's conventions.

Do not invent behavior that is not supported by the implementation or surrounding project.

### 2. Core Functionality Tests

Test the primary expected behavior.

Include scenarios such as:

* typical valid input;
* realistic production-like input;
* expected return value;
* different meaningful branches;
* optional/default arguments when applicable.

Prefer realistic domain data over placeholder values such as `"foo"` or `"bar"`.

### 3. Input Validation Tests

When relevant, test:

* `None`;
* empty strings;
* empty lists;
* empty dictionaries;
* invalid types;
* malformed values;
* zero;
* negative numbers;
* minimum values;
* maximum values;
* values immediately outside valid boundaries.

Only test validation that actually belongs to the target or its contract.

Do not create tests for hypothetical validation rules.

### 4. Error Handling Tests

Test exceptions explicitly.

Use the appropriate `unittest` assertions:

```python
with self.assertRaises(ValueError):
    target(...)
```

When the exception message is part of the contract:

```python
with self.assertRaisesRegex(
    ValueError,
    "expected message",
):
    target(...)
```

Validate:

* exception type;
* relevant error message;
* dependency failures;
* invalid state;
* important edge cases.

Do not catch exceptions manually unless necessary.

### 5. External Dependencies

Unit tests must not perform real external I/O.

Mock dependencies such as:

* HTTP requests;
* databases;
* filesystem access;
* AWS services;
* APIs;
* SDK clients;
* subprocesses;
* queues;
* email services;
* clocks/time when behavior depends on time;
* UUID/random generation when deterministic output is required.

Patch dependencies where they are **looked up by the code under test**, not necessarily where they were originally defined.

Example:

```python
@patch("project.service.httpx.get")
def test_fetches_remote_resource(self, mock_get):
    mock_get.return_value.json.return_value = {
        "status": "authorized"
    }

    result = fetch_invoice("123")

    mock_get.assert_called_once_with(
        "https://example.com/invoices/123"
    )
    self.assertEqual(result["status"], "authorized")
```

Never make a real network request in a unit test.

### 6. Side Effects and Interactions

When the target produces side effects, verify them explicitly.

Examples:

```python
mock_client.send.assert_called_once()
```

```python
mock_repository.save.assert_called_once_with(expected)
```

```python
self.assertEqual(instance.status, expected_status)
```

```python
self.assertFalse(mock_client.delete.called)
```

Validate both:

* expected calls happen;
* forbidden/unexpected calls do not happen when relevant.

Use:

* `assert_called_once()`;
* `assert_called_once_with(...)`;
* `assert_not_called()`;
* `assert_has_calls(...)`;
* `call_args`;
* `call_args_list`;

when appropriate.

### 7. Class Structure

Use descriptive test classes.

For a function:

```python
class TestIssueInvoice(unittest.TestCase):
    ...
```

For a class method:

```python
class TestInvoiceServiceIssueInvoice(unittest.TestCase):
    ...
```

If the target has several distinct behavioral areas, multiple test classes are allowed:

```python
class TestIssueInvoiceSuccess(unittest.TestCase):
    ...


class TestIssueInvoiceValidation(unittest.TestCase):
    ...


class TestIssueInvoiceErrors(unittest.TestCase):
    ...
```

Prefer one cohesive class unless splitting materially improves readability.

### 8. Setup and Cleanup

Use `setUp()` when multiple tests require the same initialization:

```python
class TestInvoiceService(unittest.TestCase):

    def setUp(self):
        self.client = MagicMock()
        self.service = InvoiceService(client=self.client)
```

Use `tearDown()` only when cleanup is actually required.

Avoid putting assertions inside `setUp()` or `tearDown()`.

Do not create large shared setup blocks for data used by only one test.

### 9. AAA Pattern

Every test must conceptually follow:

#### Arrange

Prepare input, dependencies, mocks, and expected values.

#### Act

Execute the target behavior.

#### Assert

Verify output, state, exceptions, and interactions.

Example:

```python
def test_returns_authorized_invoice(self):
    # Arrange
    self.client.issue.return_value = {
        "access_key": "123456789",
        "status": "authorized",
    }

    # Act
    result = self.service.issue_invoice(
        client_name="Acme Ltd",
        tax_id="12345678000199",
    )

    # Assert
    self.assertEqual(result["status"], "authorized")
    self.client.issue.assert_called_once()
```

Comments for `Arrange`, `Act`, and `Assert` are optional when the structure is already obvious.

Do not add comments that merely repeat the code.

### 10. Test Naming

Use descriptive snake_case names that explain behavior and scenario.

Good:

```python
def test_returns_invoice_when_api_returns_authorized_status(self):
```

```python
def test_raises_value_error_when_tax_id_is_empty(self):
```

```python
def test_does_not_call_api_when_validation_fails(self):
```

```python
def test_propagates_authentication_error_from_client(self):
```

Avoid vague names:

```python
def test_success(self):
```

```python
def test_error(self):
```

```python
def test_case_1(self):
```

A developer should understand the regression being protected against from the test name alone.

### 11. Async Code

When the target is asynchronous, use:

```python
class TestAsyncOperation(unittest.IsolatedAsyncioTestCase):
```

and:

```python
async def test_returns_expected_result(self):
    result = await target()
    self.assertEqual(result, expected)
```

Use `AsyncMock` for asynchronous dependencies:

```python
self.client.issue = AsyncMock(
    return_value={"status": "authorized"}
)
```

Do not use `asyncio.run()` inside individual tests when `IsolatedAsyncioTestCase` is appropriate.

### 12. Parameter-Like Scenarios

Because `unittest` does not provide pytest-style parametrization, use `subTest()` for closely related cases:

```python
def test_rejects_invalid_values(self):
    invalid_values = [
        None,
        "",
        -1,
    ]

    for value in invalid_values:
        with self.subTest(value=value):
            with self.assertRaises(ValueError):
                target(value)
```

Use `subTest()` only when the assertion and expected behavior are genuinely identical.

Create separate tests when scenarios represent different behaviors.

### 13. Private Methods

Prefer testing public behavior.

Do not test private implementation details such as:

```python
_internal_method()
```

unless:

* the project explicitly tests private methods;
* the method contains independently significant behavior;
* there is no reasonable public interface through which to test it.

Avoid tests tightly coupled to implementation details that would fail after a harmless refactor.

### 14. Existing Project Patterns

Before creating the test file, inspect existing tests when available.

Follow existing conventions for:

* test directory;
* module naming;
* import style;
* `setUp`;
* mocks;
* factories;
* fixtures implemented with unittest;
* helper classes;
* test data;
* naming.

Prefer consistency with the repository over introducing a new test architecture.

However, generated tests must still use `unittest.TestCase` or `unittest.IsolatedAsyncioTestCase`.

### 15. Test Independence

Every test must be independently executable.

Tests must not:

* depend on execution order;
* depend on state produced by another test;
* reuse mutated global state;
* depend on real external services;
* depend on the current date/time unless explicitly controlled;
* leave patched objects active after execution.

Running:

```bash
python -m unittest
```

must produce deterministic results.

## Coverage Priorities

Prioritize behavioral confidence over maximizing line coverage.

Generate tests in this order:

1. successful primary behavior;
2. meaningful alternate behavior;
3. important validation;
4. expected exception;
5. external dependency failure;
6. important side effect;
7. boundary condition;
8. regression-prone edge case.

Do not generate redundant tests solely to increase coverage.

## Number of Tests

Generate approximately **5–8 focused test methods** for a normal target.

Generate fewer when the function is trivial.

Generate more only when the target contains enough meaningful branches to justify them.

Do not manufacture scenarios simply to reach a test count.

## Output Requirements

Generate a complete Python test module ready to save and execute.

The result must include:

```python
import unittest
```

and all required project imports.

When mocks are required:

```python
from unittest.mock import Mock, MagicMock, AsyncMock, patch, call
```

Import only the mock utilities actually used.

End standalone test modules with:

```python
if __name__ == "__main__":
    unittest.main()
```

The generated code must:

* be syntactically valid Python;
* use `unittest`;
* use classes;
* have deterministic tests;
* contain no real external I/O;
* follow existing project conventions where available;
* avoid unnecessary mocks;
* avoid testing implementation details;
* be ready to run without manual restructuring.

## Final Instruction

Analyze the implementation before writing tests.

Generate the smallest high-value test suite that provides strong confidence in the target's behavior and catches realistic regressions.

Use **Python `unittest` with class-based tests only**.

Do not use pytest syntax or standalone test functions.
