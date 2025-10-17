# Testing Anti-Patterns: Python Examples

> **Note:** This file contains Python-specific anti-patterns. For core testing principles, see the main SKILL.md.

## Language-Specific Anti-Patterns

### Anti-Pattern 1: Testing Mock Behavior

**The violation:**
```python
# ❌ BAD: Testing that the mock exists
def test_renders_sidebar(client):
    response = client.get('/page')
    assert b'sidebar-mock' in response.data
```

**Why this is wrong:**
- You're verifying the mock works, not that the view works
- Test passes when mock is present, fails when it's not
- Tells you nothing about real behavior

**The fix:**
```python
# ✅ GOOD: Test real component or don't mock it
def test_renders_sidebar(client):
    response = client.get('/page')
    assert b'<nav' in response.data
    assert response.status_code == 200
```

### Anti-Pattern 2: Test-Only Methods in Production Classes

**The violation:**
```python
# ❌ BAD: destroy() only used in tests
class Session:
    def destroy(self):  # Looks like production API!
        if self.workspace_manager:
            self.workspace_manager.destroy_workspace(self.id)
        # ... cleanup

# In tests
def tearDown(self):
    self.session.destroy()
```

**Why this is wrong:**
- Production class polluted with test-only code
- Dangerous if accidentally called in production
- Violates YAGNI and separation of concerns

**The fix:**
```python
# ✅ GOOD: Test utilities handle test cleanup
# Session has no destroy() - it's stateless in production

# In tests/test_helpers.py
def cleanup_session(session):
    workspace = session.workspace_info
    if workspace:
        workspace_manager.destroy_workspace(workspace.id)

# In tests
def tearDown(self):
    cleanup_session(self.session)
```

### Anti-Pattern 3: Mocking Without Understanding

**The violation:**
```python
# ❌ BAD: Mock breaks test logic
@patch('tool_catalog.discover_and_cache_tools')
def test_detects_duplicate_server(mock_discover):
    # Mock prevents config write that test depends on!
    mock_discover.return_value = None

    add_server(config)
    with pytest.raises(DuplicateServerError):
        add_server(config)  # Should raise - but won't!
```

**Why this is wrong:**
- Mocked method had side effect test depended on (writing config)
- Over-mocking to "be safe" breaks actual behavior
- Test passes for wrong reason or fails mysteriously

**The fix:**
```python
# ✅ GOOD: Mock at correct level
@patch('mcp_server_manager.start_server')
def test_detects_duplicate_server(mock_start):
    # Mock the slow part, preserve behavior test needs
    mock_start.return_value = True

    add_server(config)  # Config written
    with pytest.raises(DuplicateServerError):
        add_server(config)  # Duplicate detected ✓
```

### Anti-Pattern 4: Incomplete Mocks

**The violation:**
```python
# ❌ BAD: Partial mock - only fields you think you need
mock_response = Mock()
mock_response.status = 'success'
mock_response.data = {'user_id': '123', 'name': 'Alice'}
# Missing: metadata that downstream code uses

# Later: breaks when code accesses response.metadata.request_id
```

**The fix:**
```python
# ✅ GOOD: Mirror real API completeness
mock_response = Mock()
mock_response.status = 'success'
mock_response.data = {'user_id': '123', 'name': 'Alice'}
mock_response.metadata = {'request_id': 'req-789', 'timestamp': 1234567890}
# All fields real API returns
```

## Python-Specific Anti-Patterns

### Anti-Pattern: Overusing MagicMock

**The violation:**
```python
# ❌ BAD: MagicMock accepts anything silently
def test_user_service():
    mock_repo = MagicMock()
    service = UserService(mock_repo)

    service.create_user('alice@example.com')

    # Typo in method name - but test passes!
    mock_repo.sav.assert_called_once()  # Should be 'save'
```

**Why this is wrong:**
- MagicMock creates attributes on-the-fly
- Typos don't fail - they just return new MagicMocks
- False confidence in test coverage

**The fix:**
```python
# ✅ GOOD: Use Mock with spec or create explicit test double
def test_user_service():
    # Option 1: Mock with spec
    mock_repo = Mock(spec=UserRepository)
    service = UserService(mock_repo)

    service.create_user('alice@example.com')

    mock_repo.save.assert_called_once()  # Typos now fail!

# Option 2: Explicit test double
class FakeUserRepository:
    def __init__(self):
        self.saved_users = []

    def save(self, user):
        self.saved_users.append(user)
        return True

def test_user_service():
    fake_repo = FakeUserRepository()
    service = UserService(fake_repo)

    service.create_user('alice@example.com')

    assert len(fake_repo.saved_users) == 1
    assert fake_repo.saved_users[0].email == 'alice@example.com'
```

### Anti-Pattern: Patching at Wrong Level

**The violation:**
```python
# ❌ BAD: Patching where imported, not where defined
# my_module.py
from datetime import datetime

def get_timestamp():
    return datetime.now()

# test_my_module.py
@patch('datetime.datetime.now')  # Wrong! Patching definition site
def test_get_timestamp(mock_now):
    mock_now.return_value = datetime(2024, 1, 1)
    result = get_timestamp()
    assert result == datetime(2024, 1, 1)  # Doesn't work!
```

**Why this is wrong:**
- Patch must target where name is used, not where it's defined
- Common Python gotcha
- Test fails mysteriously

**The fix:**
```python
# ✅ GOOD: Patch where imported/used
@patch('my_module.datetime')  # Patch at import site!
def test_get_timestamp(mock_datetime):
    mock_datetime.now.return_value = datetime(2024, 1, 1)
    result = get_timestamp()
    assert result == datetime(2024, 1, 1)  # Works!
```

### Anti-Pattern: Fixture Abuse

**The violation:**
```python
# ❌ BAD: Massive fixture doing too much
@pytest.fixture
def setup_everything(db, redis, s3, queue):
    # Create 10 users
    users = [create_user(f'user{i}@example.com') for i in range(10)]

    # Create 20 posts
    posts = []
    for user in users:
        posts.extend([create_post(user, f'Post {i}') for i in range(2)])

    # Create comments, likes, follows...
    # 100 more lines of setup

    yield {
        'users': users,
        'posts': posts,
        # ... 20 more keys
    }

    # Teardown...

def test_single_user_login(setup_everything):
    user = setup_everything['users'][0]  # Only need one user!
    result = login(user.email, 'password')
    assert result.success
```

**Why this is wrong:**
- Over-setup makes tests slow
- Hard to understand what test actually needs
- Fragile - breaks when any setup part fails

**The fix:**
```python
# ✅ GOOD: Minimal, focused fixtures
@pytest.fixture
def user(db):
    return create_user('test@example.com')

def test_user_login(user):
    result = login(user.email, 'password')
    assert result.success

# Use fixture composition for complex scenarios
@pytest.fixture
def user_with_posts(user, db):
    return create_posts_for_user(user, count=5)

def test_user_timeline(user_with_posts):
    timeline = get_timeline(user_with_posts)
    assert len(timeline) == 5
```

### Anti-Pattern: Not Using pytest Assertions

**The violation:**
```python
# ❌ BAD: Using unittest assertions in pytest
import unittest

class TestCalculator(unittest.TestCase):
    def test_add(self):
        result = add(2, 3)
        self.assertEqual(result, 5)  # unittest style
```

**Why this is wrong:**
- Pytest provides better error messages with plain `assert`
- No need for unittest.TestCase overhead
- Less readable

**The fix:**
```python
# ✅ GOOD: Use pytest with plain assertions
def test_add():
    result = add(2, 3)
    assert result == 5  # Better error messages!

# When it fails, pytest shows:
# AssertionError: assert 6 == 5
#  +  where 6 = add(2, 3)
```

### Anti-Pattern: Testing Private Methods

**The violation:**
```python
# ❌ BAD: Testing private methods directly
class EmailValidator:
    def validate(self, email):
        return self._has_at_symbol(email) and self._valid_domain(email)

    def _has_at_symbol(self, email):
        return '@' in email

    def _valid_domain(self, email):
        domain = email.split('@')[1]
        return '.' in domain

def test_has_at_symbol():
    validator = EmailValidator()
    assert validator._has_at_symbol('user@example.com')  # Testing private!

def test_valid_domain():
    validator = EmailValidator()
    assert validator._valid_domain('user@example.com')  # Testing private!
```

**Why this is wrong:**
- Private methods are implementation details
- Refactoring breaks tests
- Doesn't test actual behavior

**The fix:**
```python
# ✅ GOOD: Test public interface
def test_validate_accepts_valid_email():
    validator = EmailValidator()
    assert validator.validate('user@example.com')

def test_validate_rejects_email_without_at():
    validator = EmailValidator()
    assert not validator.validate('userexample.com')

def test_validate_rejects_email_without_domain_dot():
    validator = EmailValidator()
    assert not validator.validate('user@examplecom')
```

### Anti-Pattern: Context Manager Misuse in Tests

**The violation:**
```python
# ❌ BAD: Not using context manager for assertions
def test_raises_error():
    error_raised = False
    try:
        divide(1, 0)
    except ZeroDivisionError:
        error_raised = True

    assert error_raised
```

**Why this is wrong:**
- Verbose and error-prone
- Doesn't verify error message
- pytest provides better way

**The fix:**
```python
# ✅ GOOD: Use pytest.raises context manager
def test_raises_error():
    with pytest.raises(ZeroDivisionError) as exc_info:
        divide(1, 0)

    assert 'division by zero' in str(exc_info.value)

# Even better: match parameter
def test_raises_error_with_message():
    with pytest.raises(ZeroDivisionError, match=r'division by zero'):
        divide(1, 0)
```

## Django-Specific Anti-Patterns

### Anti-Pattern: Not Using Django Test Client Properly

**The violation:**
```python
# ❌ BAD: Manual request construction
def test_user_profile_view():
    from django.http import HttpRequest
    request = HttpRequest()
    request.method = 'GET'
    request.user = User.objects.create(username='test')

    response = user_profile_view(request)
    assert response.status_code == 200
```

**Why this is wrong:**
- Misses middleware
- Doesn't test URL routing
- More work than necessary

**The fix:**
```python
# ✅ GOOD: Use Django test client
def test_user_profile_view(client):
    user = User.objects.create(username='test')
    client.force_login(user)

    response = client.get('/profile/')

    assert response.status_code == 200
    assert 'test' in response.content.decode()
```

### Anti-Pattern: Not Using Transactions for Test Isolation

**The violation:**
```python
# ❌ BAD: Manual cleanup
def test_create_user():
    user = User.objects.create(username='test')
    assert User.objects.filter(username='test').exists()

    # Manual cleanup
    user.delete()

def test_another_thing():
    # Might see leftover data if previous test failed!
    assert User.objects.count() == 0
```

**Why this is wrong:**
- Tests can leak data between each other
- If test fails, cleanup doesn't run
- Manual work Django handles automatically

**The fix:**
```python
# ✅ GOOD: Use Django TestCase (automatic transaction rollback)
from django.test import TestCase

class UserTests(TestCase):
    def test_create_user(self):
        user = User.objects.create(username='test')
        assert User.objects.filter(username='test').exists()
        # Automatic rollback after test!

    def test_another_thing(self):
        # Clean database guaranteed
        assert User.objects.count() == 0
```

## Quick Reference

| Anti-Pattern | Python-Specific Fix |
|--------------|---------------------|
| Testing mocks | Use `spec=` parameter or explicit test doubles |
| Test-only methods | Extract to `tests/test_helpers.py` |
| Over-mocking | Use dependency injection, avoid excessive `@patch` |
| MagicMock abuse | Use `Mock(spec=Interface)` or real test doubles |
| Wrong patch target | Patch at import site: `@patch('my_module.thing')` |
| Fixture abuse | Keep fixtures small and focused, use composition |
| unittest assertions | Use plain `assert` in pytest for better messages |
| Testing private methods | Test public API only |
| Manual exception checking | Use `pytest.raises()` context manager |
| Django manual requests | Use Django test client |
| Manual DB cleanup | Use `TestCase` for automatic rollback |
