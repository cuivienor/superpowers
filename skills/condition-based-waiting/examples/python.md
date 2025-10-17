# Condition-Based Waiting: Python Examples

> **Note:** This file contains Python-specific patterns. For core condition-based waiting principles, see the main SKILL.md.

## Language-Specific Patterns

### Generic Wait Helper

```python
import time
from typing import Callable, TypeVar, Optional

T = TypeVar('T')

def wait_for(
    condition: Callable[[], Optional[T]],
    description: str = "condition",
    timeout: float = 5.0,
    interval: float = 0.01
) -> T:
    """
    Wait for a condition to become truthy.

    Args:
        condition: Function that returns a truthy value when ready
        description: Human-readable description for error messages
        timeout: Maximum time to wait in seconds
        interval: Polling interval in seconds

    Returns:
        The truthy value returned by condition

    Raises:
        TimeoutError: If condition doesn't become truthy within timeout
    """
    start_time = time.time()

    while True:
        result = condition()
        if result:
            return result

        if time.time() - start_time > timeout:
            raise TimeoutError(
                f"Timeout waiting for {description} after {timeout}s"
            )

        time.sleep(interval)
```

**Usage:**

```python
# Wait for file to exist
file_path = wait_for(
    lambda: Path("output.txt") if Path("output.txt").exists() else None,
    "file to be created"
)

# Wait for API response
response = wait_for(
    lambda: check_status() if check_status() == "ready" else None,
    "API to be ready"
)

# Wait for list to have items
items = wait_for(
    lambda: get_items() if len(get_items()) > 0 else None,
    "items to be available",
    timeout=10.0
)
```

## pytest-timeout Plugin

For preventing tests from hanging indefinitely:

```python
# Install: pip install pytest-timeout

# Set timeout for entire test
@pytest.mark.timeout(10)
def test_async_operation():
    result = perform_slow_operation()
    assert result == "complete"

# Set timeout in pytest.ini
[tool:pytest]
timeout = 30  # Default 30 seconds for all tests

# Or via command line
pytest --timeout=10
```

**Note:** pytest-timeout doesn't replace condition-based waiting - it's a safety net to prevent infinite hangs.

## pytest-eventually Plugin

```python
# Install: pip install pytest-eventually

from pytest_eventually import eventually

# Wait for condition to become true
@eventually(timeout=5.0, interval=0.1)
def wait_for_ready():
    assert system.status == "ready"

# Use in test
def test_system_startup():
    system.start()
    wait_for_ready()  # Polls until assertion passes
    assert system.is_running()
```

## Asyncio Wait Patterns

### Async Wait Helper

```python
import asyncio
from typing import Callable, TypeVar, Awaitable

T = TypeVar('T')

async def wait_for_async(
    condition: Callable[[], Awaitable[T]],
    description: str = "condition",
    timeout: float = 5.0,
    interval: float = 0.01
) -> T:
    """Wait for an async condition to become truthy."""
    start_time = asyncio.get_event_loop().time()

    while True:
        result = await condition()
        if result:
            return result

        elapsed = asyncio.get_event_loop().time() - start_time
        if elapsed > timeout:
            raise TimeoutError(
                f"Timeout waiting for {description} after {timeout}s"
            )

        await asyncio.sleep(interval)
```

**Usage:**

```python
# Wait for async database query
async def test_user_created():
    await create_user_async(email="test@example.com")

    user = await wait_for_async(
        lambda: db.users.find_one({"email": "test@example.com"}),
        "user to be created"
    )

    assert user["email"] == "test@example.com"
```

### Using asyncio.wait_for

```python
import asyncio

async def test_with_builtin_timeout():
    # ❌ BAD: Guessing at timing
    await asyncio.sleep(1.0)
    result = await get_result()
    assert result is not None

    # ✅ GOOD: Wait for actual condition with timeout
    async def wait_for_result():
        while True:
            result = await get_result()
            if result:
                return result
            await asyncio.sleep(0.01)

    result = await asyncio.wait_for(wait_for_result(), timeout=5.0)
    assert result is not None
```

### Waiting for Multiple Async Conditions

```python
async def test_multiple_services_ready():
    async def wait_all_ready():
        # Wait for all services to be ready
        while True:
            statuses = await asyncio.gather(
                service1.check_status(),
                service2.check_status(),
                service3.check_status()
            )
            if all(s == "ready" for s in statuses):
                return True
            await asyncio.sleep(0.1)

    await asyncio.wait_for(wait_all_ready(), timeout=30.0)
    # All services ready - continue test
```

## Django Testing Patterns

### Waiting for Celery Tasks

```python
from celery.result import AsyncResult

def wait_for_task(task_id: str, timeout: float = 10.0) -> dict:
    """Wait for Celery task to complete."""
    result = AsyncResult(task_id)

    start_time = time.time()
    while not result.ready():
        if time.time() - start_time > timeout:
            raise TimeoutError(f"Task {task_id} did not complete in {timeout}s")
        time.sleep(0.1)

    if result.failed():
        raise result.result  # Re-raise task exception

    return result.result

def test_email_task():
    user = User.objects.create(email="test@example.com")
    task = send_welcome_email.delay(user.id)

    result = wait_for_task(task.id, timeout=10.0)

    assert result["status"] == "sent"
    assert len(mail.outbox) == 1
```

### Waiting for Database Changes

```python
def test_background_processing():
    order = Order.objects.create(status="pending")
    process_order.delay(order.id)

    # ❌ BAD: Guessing at timing
    time.sleep(2)
    order.refresh_from_db()
    assert order.status == "completed"

    # ✅ GOOD: Wait for status change
    def check_order_complete():
        order.refresh_from_db()
        return order if order.status == "completed" else None

    wait_for(check_order_complete, "order to be completed", timeout=10.0)
    assert order.status == "completed"
```

### Django Live Server Tests (Selenium)

```python
from django.contrib.staticfiles.testing import StaticLiveServerTestCase
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

class CheckoutTest(StaticLiveServerTestCase):
    def setUp(self):
        self.driver = webdriver.Chrome()

    def test_checkout_flow(self):
        self.driver.get(f"{self.live_server_url}/checkout")

        # ❌ BAD: Arbitrary wait
        time.sleep(1)
        submit_button = self.driver.find_element(By.ID, "submit")

        # ✅ GOOD: Wait for element to be clickable
        submit_button = WebDriverWait(self.driver, 10).until(
            EC.element_to_be_clickable((By.ID, "submit"))
        )
        submit_button.click()

        # Wait for success message
        success = WebDriverWait(self.driver, 10).until(
            EC.presence_of_element_located((By.CLASS_NAME, "success"))
        )
        self.assertIn("Order placed", success.text)
```

## FastAPI Testing with pytest-asyncio

```python
import pytest
from httpx import AsyncClient
from app import app

@pytest.mark.asyncio
async def test_async_endpoint():
    async with AsyncClient(app=app, base_url="http://test") as client:
        # Start async operation
        response = await client.post("/process", json={"data": "test"})
        job_id = response.json()["job_id"]

        # Wait for job to complete
        async def check_job_status():
            resp = await client.get(f"/jobs/{job_id}")
            data = resp.json()
            return data if data["status"] == "completed" else None

        result = await wait_for_async(
            check_job_status,
            "job to complete",
            timeout=10.0
        )

        assert result["status"] == "completed"
```

## Polling External APIs

```python
import requests

def wait_for_webhook_processed(webhook_id: str, timeout: float = 30.0) -> dict:
    """Wait for external service to process webhook."""
    def check_status():
        response = requests.get(f"https://api.example.com/webhooks/{webhook_id}")
        if response.status_code == 200:
            data = response.json()
            if data["status"] == "processed":
                return data
        return None

    return wait_for(
        check_status,
        f"webhook {webhook_id} to be processed",
        timeout=timeout,
        interval=1.0  # Poll every 1 second for external API
    )

def test_webhook_integration():
    webhook = trigger_webhook(payload={"event": "test"})

    result = wait_for_webhook_processed(webhook["id"])

    assert result["status"] == "processed"
```

## Real-World Example: Flaky Test Fix

**BEFORE (flaky):**
```python
def test_image_processing():
    image = Image.objects.create(file=uploaded_file)
    process_image_task.delay(image.id)

    time.sleep(3)  # Hope it finishes in 3 seconds

    image.refresh_from_db()
    assert image.status == "processed"
    assert image.thumbnail_url is not None
```

**AFTER (reliable):**
```python
def test_image_processing():
    image = Image.objects.create(file=uploaded_file)
    process_image_task.delay(image.id)

    def check_processed():
        image.refresh_from_db()
        return image if image.status == "processed" else None

    wait_for(
        check_processed,
        "image processing to complete",
        timeout=30.0
    )

    assert image.status == "processed"
    assert image.thumbnail_url is not None
```

## Testing with Threading

```python
import threading
import queue

def test_threaded_worker():
    result_queue = queue.Queue()

    def worker():
        time.sleep(0.5)  # Simulate work
        result_queue.put("done")

    thread = threading.Thread(target=worker)
    thread.start()

    # ❌ BAD: Guessing at thread completion time
    time.sleep(1)
    result = result_queue.get_nowait()

    # ✅ GOOD: Wait for result to be available
    def check_result():
        try:
            return result_queue.get_nowait()
        except queue.Empty:
            return None

    result = wait_for(check_result, "worker to finish", timeout=5.0)
    assert result == "done"

    thread.join()
```

## Common Pitfalls

### Pitfall 1: Not Refreshing Data

```python
# ❌ BAD: Stale object
user = User.objects.get(id=user_id)
process_user_task.delay(user_id)
wait_for(lambda: user.is_active, "user to be active")  # Stale!

# ✅ GOOD: Refresh inside condition
user_id = user.id
process_user_task.delay(user_id)
wait_for(
    lambda: User.objects.get(id=user_id).is_active,
    "user to be active"
)
```

### Pitfall 2: Blocking Event Loop

```python
# ❌ BAD: Using sync wait in async context
async def test_async_operation():
    await start_operation()
    time.sleep(1)  # Blocks event loop!
    result = await get_result()

# ✅ GOOD: Use async sleep
async def test_async_operation():
    await start_operation()
    await asyncio.sleep(1)
    result = await get_result()
```

### Pitfall 3: Too-Short Timeout

```python
# ❌ BAD: Unrealistic timeout
wait_for(
    lambda: heavy_computation_complete(),
    "computation to finish",
    timeout=0.1  # Too short!
)

# ✅ GOOD: Generous timeout
wait_for(
    lambda: heavy_computation_complete(),
    "computation to finish",
    timeout=30.0  # Realistic for heavy work
)
```
