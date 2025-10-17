# Condition-Based Waiting: Ruby/Rails Examples

> **Note:** This file contains Ruby/Rails-specific patterns. For core condition-based waiting principles, see the main SKILL.md.

## Language-Specific Patterns

### Generic Wait Helper

```ruby
def wait_for(description = "condition", timeout: 5.0, interval: 0.01)
  start_time = Time.now

  loop do
    result = yield
    return result if result

    if Time.now - start_time > timeout
      raise "Timeout waiting for #{description} after #{timeout}s"
    end

    sleep(interval) # Poll every 10ms by default
  end
end
```

**Usage:**

```ruby
# Wait for ActiveJob to complete
wait_for("job to complete", timeout: 10.0) do
  MyJob.find_by(id: job_id)&.completed?
end

# Wait for record to be created
user = wait_for("user to be created") do
  User.find_by(email: "test@example.com")
end

# Wait for state change
wait_for("order to be processed") do
  order.reload.status == "processed"
end
```

## Capybara Integration Tests

Capybara has built-in waiting for page elements, but you may need custom waits for other conditions.

### Built-in Capybara Waiting

```ruby
# Capybara automatically waits for elements (default: 2 seconds)
expect(page).to have_content("Success")
expect(page).to have_selector(".loaded")
click_button "Submit"  # Waits for button to be clickable
```

### Custom Capybara Waits

```ruby
# Wait for AJAX request to complete
using_wait_time(10) do
  expect(page).to have_content("Data loaded")
end

# Wait for JavaScript state
expect(page).to have_selector("[data-ready='true']")

# Wait for count to change
expect(page).to have_selector(".item", count: 5)
```

### Waiting for Database Changes in Feature Tests

```ruby
# ❌ BAD: Assuming immediate database update
click_button "Process Order"
order = Order.last
expect(order.status).to eq("processed")  # Flaky!

# ✅ GOOD: Wait for database change
click_button "Process Order"
wait_for("order to be processed") do
  Order.last&.status == "processed"
end
order = Order.last
expect(order.status).to eq("processed")
```

### Waiting for Background Jobs

```ruby
# ❌ BAD: Guessing how long job takes
PerformLater.perform_later(user_id: user.id)
sleep(1)
expect(user.reload.processed).to be(true)  # Flaky!

# ✅ GOOD: Wait for job result
PerformLater.perform_later(user_id: user.id)
wait_for("user to be processed", timeout: 5.0) do
  user.reload.processed?
end
expect(user.processed).to be(true)
```

## RSpec-Specific Patterns

### RSpec `eventually` Matcher (rspec-wait gem)

```ruby
# Install: gem 'rspec-wait'

require 'rspec/wait'

# Wait for condition to become true
wait_for { user.reload.active? }.to eq(true)

# Wait with custom timeout
wait_for(timeout: 10) { job.reload.completed? }.to be(true)

# Wait for value to change
expect { order.reload.status }.to(
  become("processed").within(5.seconds)
)
```

### Custom RSpec Helper

```ruby
# spec/support/wait_helpers.rb
module WaitHelpers
  def wait_for(description = "condition", timeout: 5.0, interval: 0.01)
    start_time = Time.now

    loop do
      result = yield
      return result if result

      if Time.now - start_time > timeout
        raise "Timeout waiting for #{description} after #{timeout}s"
      end

      sleep(interval)
    end
  end
end

RSpec.configure do |config|
  config.include WaitHelpers
end
```

**Usage in specs:**

```ruby
RSpec.describe "Order processing" do
  it "processes order asynchronously" do
    order = create(:order)
    ProcessOrderJob.perform_later(order.id)

    wait_for("order to be processed") do
      order.reload.processed?
    end

    expect(order.status).to eq("completed")
  end
end
```

## Minitest Patterns

### Custom Minitest Helper

```ruby
# test/test_helper.rb
module WaitHelpers
  def wait_for(description = "condition", timeout: 5.0, interval: 0.01)
    start_time = Time.now

    loop do
      result = yield
      return result if result

      if Time.now - start_time > timeout
        flunk("Timeout waiting for #{description} after #{timeout}s")
      end

      sleep(interval)
    end
  end
end

class ActiveSupport::TestCase
  include WaitHelpers
end
```

**Usage:**

```ruby
class OrderProcessingTest < ActiveSupport::TestCase
  test "processes order asynchronously" do
    order = orders(:pending)
    ProcessOrderJob.perform_later(order.id)

    wait_for("order to be processed") do
      order.reload.processed?
    end

    assert_equal "completed", order.status
  end
end
```

## Rails System Tests (Capybara + Selenium)

### Waiting for JavaScript Events

```ruby
class CheckoutSystemTest < ApplicationSystemTestCase
  test "completes checkout flow" do
    visit checkout_path
    fill_in "Email", with: "test@example.com"
    click_button "Pay Now"

    # Wait for JavaScript to process payment
    assert_selector "[data-payment-status='success']"

    # Wait for redirect
    assert_current_path order_confirmation_path
  end
end
```

### Waiting for WebSocket Updates

```ruby
# ❌ BAD: Guessing at WebSocket timing
trigger_websocket_event
sleep(0.5)
assert_selector ".notification"  # Flaky!

# ✅ GOOD: Wait for element to appear
trigger_websocket_event
assert_selector ".notification", wait: 5
```

### Waiting for Polling Updates

```ruby
# Page polls API every 2 seconds for status
visit job_status_path(job_id)

# ❌ BAD: Arbitrary wait
sleep(3)
assert_selector ".status-complete"  # What if first poll hasn't happened?

# ✅ GOOD: Wait for status to appear
assert_selector ".status-complete", wait: 10
```

## ActiveJob Testing

### Testing with `perform_enqueued_jobs`

```ruby
require 'test_helper'

class EmailNotificationTest < ActiveSupport::TestCase
  test "sends email after user signup" do
    perform_enqueued_jobs do
      user = User.create!(email: "test@example.com")

      # No need to wait - perform_enqueued_jobs runs synchronously
      assert_equal 1, ActionMailer::Base.deliveries.size
    end
  end
end
```

### Testing with `perform_later` (actual background job)

```ruby
test "processes order in background" do
  order = create(:order)
  ProcessOrderJob.perform_later(order.id)

  # Job runs in background - need to wait
  wait_for("order to be processed", timeout: 10.0) do
    order.reload.status == "processed"
  end

  assert order.processed?
end
```

## Real-World Example: Flaky Test Fix

**BEFORE (flaky):**
```ruby
test "video transcoding completes" do
  video = create(:video, file: uploaded_file)
  TranscodeVideoJob.perform_later(video.id)

  sleep(2)  # Hope it finishes in 2 seconds

  assert_equal "completed", video.reload.status
  assert_not_nil video.transcoded_url
end
```

**AFTER (reliable):**
```ruby
test "video transcoding completes" do
  video = create(:video, file: uploaded_file)
  TranscodeVideoJob.perform_later(video.id)

  wait_for("video transcoding to complete", timeout: 30.0) do
    video.reload.status == "completed"
  end

  assert_equal "completed", video.status
  assert_not_nil video.transcoded_url
end
```

## Polling External APIs

```ruby
# Wait for third-party API to process webhook
def wait_for_payment_confirmation(payment_id, timeout: 30.0)
  wait_for("payment confirmation", timeout: timeout, interval: 1.0) do
    response = StripeClient.retrieve_payment(payment_id)
    response["status"] == "succeeded"
  end
end

test "processes payment via Stripe" do
  payment = create_stripe_payment(amount: 1000)

  wait_for_payment_confirmation(payment.id)

  assert_equal "paid", order.reload.payment_status
end
```

## Common Pitfalls

### Pitfall 1: Not Reloading Records

```ruby
# ❌ BAD: Stale record
user = User.find(user_id)
ProcessUserJob.perform_later(user_id)
wait_for("user to be processed") do
  user.active?  # This checks the stale in-memory object!
end

# ✅ GOOD: Reload inside condition
user = User.find(user_id)
ProcessUserJob.perform_later(user_id)
wait_for("user to be processed") do
  user.reload.active?  # Fresh data from database
end
```

### Pitfall 2: Waiting for Synchronous Code

```ruby
# ❌ BAD: No need to wait for synchronous code
user = User.create!(email: "test@example.com")
wait_for { User.exists?(email: "test@example.com") }  # Unnecessary!

# ✅ GOOD: Only wait for async operations
user = User.create!(email: "test@example.com")
expect(user).to be_persisted  # Synchronous - no waiting needed
```

### Pitfall 3: Timeout Too Short

```ruby
# ❌ BAD: Timeout shorter than expected operation time
wait_for("slow operation", timeout: 0.1) do
  # Operation that typically takes 1 second
  expensive_computation_complete?
end

# ✅ GOOD: Generous timeout
wait_for("slow operation", timeout: 10.0) do
  expensive_computation_complete?
end
```
