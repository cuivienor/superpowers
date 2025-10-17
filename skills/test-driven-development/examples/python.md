# Test-Driven Development: Python Examples

> **Note:** This file contains Python-specific patterns. For core TDD principles, see the main SKILL.md.

## Language-Specific Patterns

### Test Structure

```python
# Test files must match test_*.py or *_test.py pattern
# tests/test_email_validator.py

import pytest
from app.validators import EmailValidator

class TestEmailValidator:
    def setup_method(self):
        """Runs before each test method"""
        self.validator = EmailValidator()

    def test_valid_returns_true_for_valid_email(self):
        assert self.validator.valid("user@example.com")

    def test_valid_returns_false_for_empty_email(self):
        assert not self.validator.valid("")

    @staticmethod
    def test_normalize_removes_whitespace():
        assert EmailValidator.normalize("  user@example.com  ") == "user@example.com"
```

**Key patterns:**
- Instance methods: `test_<method>_<behavior>`
- Static methods: `@staticmethod` decorator
- Setup: `setup_method` and `teardown_method`
- Fixtures: Use `@pytest.fixture` for shared state

### Assertions

```python
# pytest assertions use plain assert with helpful error messages
assert actual == expected
assert len(collection) == 0
assert isinstance(obj, MyClass)
assert obj.is_ready()
assert actual != expected
assert obj is not None

# pytest provides special assertions
pytest.raises(ValueError, func)
pytest.warns(UserWarning, func)
pytest.approx(3.14, 0.01)
```

### Mocking with unittest.mock

```python
from unittest.mock import Mock, patch

def test_service_calls_repository():
    # Create mock
    mock_repo = Mock()
    mock_repo.save.return_value = True

    service = UserService(mock_repo)
    result = service.create_user("test@example.com")

    # Verify call
    mock_repo.save.assert_called_once()
    assert result is True

# Patching
@patch('app.services.external_api')
def test_service_with_patch(mock_api):
    mock_api.fetch.return_value = {"status": "ok"}

    result = MyService().process()

    assert result == {"status": "ok"}
```

### Running Tests

```bash
# Run all tests
pytest

# Run specific test file
pytest tests/test_validator.py

# Run specific test
pytest tests/test_validator.py::TestEmailValidator::test_valid_returns_true_for_valid_email

# Run with verbose output
pytest -v

# Run with coverage
pytest --cov=app --cov-report=html

# Run tests matching pattern
pytest -k "email"
```

### Coverage Requirements

```bash
# Check coverage
pytest --cov=app --cov-report=term-missing

# Coverage must be ≥95% branch
pytest --cov=app --cov-branch --cov-report=term-missing

# Generate HTML report
pytest --cov=app --cov-report=html
# Open htmlcov/index.html
```

### Complete Example: Bug Fix

**Bug:** Empty email accepted by validator

**RED**
```python
def test_valid_rejects_empty_email():
    validator = EmailValidator()

    assert not validator.valid("")
    assert validator.error == "Email required"
```

**Verify RED**
```bash
$ pytest tests/test_email_validator.py::test_valid_rejects_empty_email -v
FAILED: AssertionError: assert True
```

**GREEN**
```python
class EmailValidator:
    def __init__(self):
        self.error = None

    def valid(self, email):
        if not email or not email.strip():
            self.error = "Email required"
            return False
        return True
```

**Verify GREEN**
```bash
$ pytest tests/test_email_validator.py::test_valid_rejects_empty_email -v
PASSED
```

**REFACTOR**
Extract empty check if used in multiple validators.

### Pytest-Specific Features

**Parametrized tests** (table-driven):
```python
@pytest.mark.parametrize("email,expected", [
    ("user@example.com", True),
    ("", False),
    ("   ", False),
    ("invalid", False),
])
def test_valid_with_various_inputs(email, expected):
    validator = EmailValidator()
    assert validator.valid(email) == expected
```

**Fixtures** (shared setup):
```python
@pytest.fixture
def validator():
    return EmailValidator()

def test_valid_email(validator):
    assert validator.valid("user@example.com")

def test_invalid_email(validator):
    assert not validator.valid("")
```

**Markers** (categorize tests):
```python
@pytest.mark.slow
def test_expensive_operation():
    # Run with: pytest -m slow
    pass

@pytest.mark.integration
def test_database_integration():
    # Run with: pytest -m integration
    pass
```
