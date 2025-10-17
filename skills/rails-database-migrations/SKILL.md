---
name: Rails-Database-Migrations
description: Use when creating or modifying database migrations to ensure zero-downtime deployments, data safety, and reversibility - prevents production outages from unsafe schema changes
---

# Rails Database Migrations

## Overview

Database migrations at scale require careful planning to avoid downtime, data loss, and deployment failures.

**Core principle:** Schema changes must be safe in production with running code from previous deploy.

**Announce at start:** "I'm using the Rails Database Migrations skill to ensure safe schema changes."

## When to Use

**Always use before:**
- Adding/removing/changing columns
- Adding/removing indexes
- Adding/removing constraints
- Renaming columns or tables
- Changing column types
- Moving data between tables

**Red flags requiring extra care:**
- "I need to change this column type"
- "I'm removing a column"
- "I need to add NOT NULL"
- "I'm renaming a column"
- "I need to add a unique constraint"

## The Iron Law

```
MIGRATIONS MUST BE SAFE WITH CODE FROM PREVIOUS DEPLOY
```

Deploy order: Deploy migration → Run migration → Deploy code.

**Between migration and code deploy:** Old code runs with new schema.

## Safe vs Unsafe Operations

### Always Safe (Single Migration)

```ruby
# Add nullable column
add_column :users, :phone, :string

# Add index (non-unique, with safe options)
add_index :users, :email, algorithm: :concurrently

# Create new table
create_table :email_logs do |t|
  t.string :email
  t.timestamps
end
```

### Unsafe (Requires Multiple Steps)

```ruby
# UNSAFE: Add NOT NULL column
add_column :users, :phone, :string, null: false

# UNSAFE: Remove column still in use
remove_column :users, :legacy_field

# UNSAFE: Change column type
change_column :users, :age, :integer

# UNSAFE: Add unique constraint with data
add_index :users, :email, unique: true

# UNSAFE: Rename column/table
rename_column :users, :name, :full_name
```

## Multi-Step Migration Patterns

### Pattern 1: Adding NOT NULL Column

**Problem:** Can't add NOT NULL column if table has rows - violates constraint immediately.

**Solution:** 3-step process across 3 deploys:

```ruby
# Migration 1: Add nullable column with default
class AddPhoneToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :phone, :string, default: "", null: true
  end
end

# Deploy 1 + Run migration
# Wait for backfill or ensure code sets value for new records

# Migration 2: Backfill existing records (if needed)
class BackfillUserPhones < ActiveRecord::Migration[7.0]
  def up
    # Only if you need to backfill existing data
    User.where(phone: nil).update_all(phone: "")
  end

  def down
    # No-op: don't clear data on rollback
  end
end

# Deploy 2 + Run migration

# Migration 3: Add constraint
class AddPhoneNotNullToUsers < ActiveRecord::Migration[7.0]
  def up
    change_column_null :users, :phone, false
  end

  def down
    change_column_null :users, :phone, true
  end
end

# Deploy 3 + Run migration
```

### Pattern 2: Removing Column

**Problem:** If you remove column in same deploy as removing code, rollback will fail (code references missing column).

**Solution:** 2-step process across 2 deploys:

```ruby
# Deploy 1: Remove all code references to column
# (Deploy code only, no migration)

# Migration 1: Remove column
class RemoveLegacyFieldFromUsers < ActiveRecord::Migration[7.0]
  def change
    remove_column :users, :legacy_field, :string
  end
end

# Deploy 2 + Run migration
```

**Critical:** Always remove code references in a separate deploy BEFORE removing column.

### Pattern 3: Changing Column Type

**Problem:** Type change can cause data loss or prevent rollback.

**Solution:** 4-step process with new column:

```ruby
# Migration 1: Add new column
class AddNewAgeToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :new_age, :integer
  end
end

# Deploy 1 + Run migration

# Migration 2: Backfill new column
class BackfillNewAge < ActiveRecord::Migration[7.0]
  def up
    User.find_each do |user|
      user.update_column(:new_age, user.age.to_i)
    end
  end

  def down
    # No-op
  end
end

# Deploy 2 + Run migration

# Deploy 3: Update code to use new_age column
# (Code change only, no migration)

# Migration 3: Remove old column
class RemoveOldAgeFromUsers < ActiveRecord::Migration[7.0]
  def change
    remove_column :users, :age, :string
    rename_column :users, :new_age, :age
  end
end

# Deploy 4 + Run migration
```

### Pattern 4: Adding Unique Index

**Problem:** Adding unique constraint fails if duplicates exist.

**Solution:** 3-step process:

```ruby
# Migration 1: Add non-unique index first
class AddEmailIndexToUsers < ActiveRecord::Migration[7.0]
  disable_ddl_transaction!

  def change
    add_index :users, :email, algorithm: :concurrently
  end
end

# Deploy 1 + Run migration

# Deploy 2: Add code to enforce uniqueness + deduplicate data
# (Code change with validation + cleanup)

# Migration 2: Make index unique
class MakeEmailIndexUnique < ActiveRecord::Migration[7.0]
  disable_ddl_transaction!

  def up
    remove_index :users, :email
    add_index :users, :email, unique: true, algorithm: :concurrently
  end

  def down
    remove_index :users, :email
    add_index :users, :email, algorithm: :concurrently
  end
end

# Deploy 3 + Run migration
```

## Reversibility

### Use `change` When Safe

```ruby
class AddEmailToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :email, :string
    add_index :users, :email
  end
end
```

Rails knows how to reverse: `remove_column`, `remove_index`.

### Use `up`/`down` When Unsafe

```ruby
# WRONG: change doesn't know how to reverse this
class RemoveNameFromUsers < ActiveRecord::Migration[7.0]
  def change
    remove_column :users, :name  # Loses type information!
  end
end

# CORRECT: Specify type for reversibility
class RemoveNameFromUsers < ActiveRecord::Migration[7.0]
  def up
    remove_column :users, :name
  end

  def down
    add_column :users, :name, :string
  end
end
```

### Use `up`/`down` for Data Migrations

```ruby
class BackfillUserEmails < ActiveRecord::Migration[7.0]
  def up
    User.where(email: nil).find_each do |user|
      user.update!(email: "#{user.username}@example.com")
    end
  end

  def down
    # Often no-op for data migrations
    # Don't destroy data on rollback
  end
end
```

## Index Creation Best Practices

### Always Use `algorithm: :concurrently` for Large Tables

```ruby
class AddEmailIndexToUsers < ActiveRecord::Migration[7.0]
  disable_ddl_transaction!  # Required for concurrent

  def change
    add_index :users, :email, algorithm: :concurrently
  end
end
```

**Why:** Without `:concurrently`, index creation locks table (no writes during creation).

**Trade-off:** Concurrent index creation is slower but doesn't lock table.

### Removing Indexes

```ruby
class RemoveEmailIndexFromUsers < ActiveRecord::Migration[7.0]
  disable_ddl_transaction!

  def change
    remove_index :users, :email, algorithm: :concurrently
  end
end
```

## Testing Migrations

### Test Both Directions

```bash
# Run migration
bin/rails db:migrate

# Test rollback
bin/rails db:rollback

# Run again to verify
bin/rails db:migrate

# Or use redo (rollback + migrate)
bin/rails db:migrate:redo
```

**MANDATORY:** Always test rollback before committing migration.

### Test with Data

```ruby
# In rails console or test
User.create!(name: "Test")
# Run migration
# Verify data intact
User.first.name  # Should still work
```

## Common Mistakes

### Adding Column with Default (Postgres < 11)

```ruby
# SLOW on large tables (rewrites entire table)
add_column :users, :active, :boolean, default: true
```

**Fix for large tables:**
```ruby
# Step 1: Add nullable column without default
add_column :users, :active, :boolean

# Step 2: Backfill in batches
User.in_batches.update_all(active: true)

# Step 3: Add default for new records
change_column_default :users, :active, true

# Step 4: Add NOT NULL if needed
change_column_null :users, :active, false
```

### Not Using Transactions for Data Safety

```ruby
# WRONG: Multi-statement migration without transaction protection
def up
  remove_column :users, :old_email
  rename_column :users, :new_email, :email
end

# CORRECT: Explicit transaction (when needed)
def up
  User.transaction do
    remove_column :users, :old_email
    rename_column :users, :new_email, :email
  end
end
```

**Note:** DDL transactions are automatic in Postgres, optional in MySQL.

### Renaming Columns/Tables

```ruby
# WRONG: Rename breaks old code immediately
rename_column :users, :name, :full_name

# CORRECT: Create new column + transition period
# (Similar to "Changing Column Type" pattern)
```

## Running Migrations at Shopify

```bash
# Run migrations
bin/rails db:migrate

# Or as part of full environment setup
/opt/dev/bin/dev up

# Rollback one migration
bin/rails db:rollback

# Rollback N migrations
bin/rails db:rollback STEP=3

# Redo last migration (down + up)
bin/rails db:migrate:redo

# Check migration status
bin/rails db:migrate:status
```

## Red Flags - STOP and Reconsider

**Never:**
- Add NOT NULL column in single migration
- Remove column in same deploy as code removal
- Change column type without new column strategy
- Add unique constraint without deduplicating first
- Skip testing rollback
- Assume empty table (might not be in production)

**Always:**
- Test migration both directions (up + down)
- Consider: "What if old code runs with new schema?"
- Use `algorithm: :concurrently` for indexes on large tables
- Remove code references before removing columns
- Split risky operations across multiple deploys

## Checklist Before Committing Migration

- [ ] Migration tested with `db:migrate:redo`
- [ ] Rollback tested successfully
- [ ] Safe with old code running (previous deploy)
- [ ] Large table indexes use `algorithm: :concurrently`
- [ ] NOT NULL columns added in multiple steps
- [ ] Column removals only after code removed in prior deploy
- [ ] Type changes use new column pattern
- [ ] Data migrations use `up`/`down`, not `change`
- [ ] Migration reviewed for reversibility

## Integration with Other Skills

**Pairs with:**
- **Test-Driven Development**: Write test before migration to verify behavior
- **Verification Before Completion**: Test rollback before claiming complete
- **Shopify Code Search**: Find existing migration patterns before writing

**Before using:**
- Search for similar migrations in codebase (Shopify Code Search skill)

**After using:**
- Test-Driven Development skill (add tests for schema changes)
- Verification Before Completion skill (verify rollback works)

## Quick Reference

| Operation | Single Migration? | Notes |
|-----------|-------------------|-------|
| Add nullable column | ✅ Yes | Safe to add |
| Add NOT NULL column | ❌ No | 3 steps: nullable → backfill → constraint |
| Remove column | ❌ No | Remove code first (separate deploy) |
| Add index | ✅ Yes | Use `algorithm: :concurrently` |
| Remove index | ✅ Yes | Use `algorithm: :concurrently` |
| Change column type | ❌ No | 4 steps: add new → backfill → switch code → remove old |
| Rename column | ❌ No | Use new column pattern |
| Add unique constraint | ❌ No | 3 steps: regular index → dedupe → unique index |
| Create table | ✅ Yes | Always safe |
| Drop table | ✅ Yes | Only if code doesn't reference it |

## Example: Complete Multi-Step Migration

**Goal:** Add NOT NULL email column to users table

```ruby
# ============================================
# DEPLOY 1: Migration + Run
# ============================================

# db/migrate/20250116000001_add_email_to_users.rb
class AddEmailToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :email, :string
  end
end

# Commands:
# bin/rails db:migrate
# Deploy code (no app changes yet)

# ============================================
# DEPLOY 2: Code Changes Only
# ============================================

# Update User model to set email on creation
class User < ApplicationRecord
  validates :email, presence: true
  before_validation :set_default_email, on: :create

  def set_default_email
    self.email ||= "#{username}@example.com"
  end
end

# Deploy code

# ============================================
# DEPLOY 3: Migration + Run
# ============================================

# db/migrate/20250116000002_backfill_user_emails.rb
class BackfillUserEmails < ActiveRecord::Migration[7.0]
  def up
    User.where(email: nil).find_each do |user|
      user.update_column(:email, "#{user.username}@example.com")
    end
  end

  def down
    # No-op: don't clear emails on rollback
  end
end

# Commands:
# bin/rails db:migrate

# ============================================
# DEPLOY 4: Migration + Run
# ============================================

# db/migrate/20250116000003_add_not_null_to_user_email.rb
class AddNotNullToUserEmail < ActiveRecord::Migration[7.0]
  def up
    change_column_null :users, :email, false
  end

  def down
    change_column_null :users, :email, true
  end
end

# Commands:
# bin/rails db:migrate
# bin/rails db:migrate:redo  # Test rollback

# ============================================
# DEPLOY 5 (Optional): Add Index
# ============================================

# db/migrate/20250116000004_add_index_to_user_email.rb
class AddIndexToUserEmail < ActiveRecord::Migration[7.0]
  disable_ddl_transaction!

  def change
    add_index :users, :email, algorithm: :concurrently
  end
end

# Commands:
# bin/rails db:migrate
```

Total: 5 deploys for a single NOT NULL column with index. This is normal and safe at scale.
