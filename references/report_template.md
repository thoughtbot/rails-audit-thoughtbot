# Audit Report Template

Use this template when generating the final audit report.

---

```markdown
# Rails Application Audit Report

**Generated**: {{DATE}}
**Application**: {{APP_NAME}}
**Rails Version**: {{RAILS_VERSION}}
**Ruby Version**: {{RUBY_VERSION}}
**Audit Scope**: {{SCOPE: Full Application | Targeted: specific paths}}

---

## Executive Summary

| Category | Critical | High | Medium | Low | Total |
|----------|----------|------|--------|-----|-------|
| Testing | X | X | X | X | X |
| Security | X | X | X | X | X |
| Models | X | X | X | X | X |
| Controllers | X | X | X | X | X |
| Code Design | X | X | X | X | X |
| Views | X | X | X | X | X |
| External Services | X | X | X | X | X |
| Database & Performance | X | X | X | X | X |
| **Total** | **X** | **X** | **X** | **X** | **X** |

### Key Findings

1. **[Most critical finding]**
2. **[Second most critical finding]**
3. **[Third most critical finding]**

---

## 1. Testing Issues

### Overview
- **Test Framework**: RSpec / Minitest
- **Files with Tests**: X / Y (Z%)
- **Estimated Coverage**: Low / Medium / High

### Critical Issues

> None found (or list issues)

### High Severity

#### [ISSUE_ID] Missing Tests for [ModelName]

**File**: `app/models/model_name.rb`
**Impact**: Untested business logic can introduce regressions
**Details**: No corresponding spec file found at `spec/models/model_name_spec.rb`

**Recommendation**:
Create spec file with tests for:
- Validations
- Public methods: `#method_one`, `#method_two`
- Associations (if complex)

```ruby
# spec/models/model_name_spec.rb
RSpec.describe ModelName do
  describe "validations" do
    it { is_expected.to validate_presence_of(:field) }
  end
  
  describe "#method_one" do
    it "returns expected result" do
      # test implementation
    end
  end
end
```

### Medium Severity

#### [ISSUE_ID] Testing Antipattern: Mystery Guest

**File**: `spec/models/user_spec.rb:45`
**Impact**: Tests are hard to understand
**Details**: Test relies on factory defaults that aren't visible in test

**Current Code**:
```ruby
it "calculates total" do
  user = create(:user) # What attributes matter?
  expect(user.total).to eq 100
end
```

**Recommendation**:
```ruby
it "calculates total" do
  user = create(:user, orders_count: 2, average_order: 50)
  expect(user.total).to eq 100
end
```

### Low Severity

> List low severity testing issues

---

## 2. Security Issues

### Critical Issues

#### [ISSUE_ID] SQL Injection Risk

**File**: `app/models/search.rb:23`
**Impact**: Potential data breach, unauthorized data access
**Details**: User input interpolated directly into SQL query

**Current Code**:
```ruby
def search(term)
  where("name LIKE '%#{term}%'")
end
```

**Recommendation**:
```ruby
def search(term)
  where("name LIKE ?", "%#{term}%")
end
```

### High Severity

#### [ISSUE_ID] Mass Assignment Vulnerability

**File**: `app/controllers/users_controller.rb:34`
**Impact**: Users could modify restricted attributes

**Current Code**:
```ruby
def user_params
  params.require(:user).permit!
end
```

**Recommendation**:
```ruby
def user_params
  params.require(:user).permit(:name, :email, :phone)
end
```

### Medium Severity

> List medium severity security issues

### Low Severity

> List low severity security issues

---

## 3. Models Issues

### High Severity

#### [ISSUE_ID] Fat Model: User

**File**: `app/models/user.rb`
**Lines**: 450
**Public Methods**: 32
**Impact**: Hard to maintain, test, and understand

**Details**:
Multiple responsibilities detected:
- Authentication logic
- Profile management
- Notification preferences
- Billing information
- Activity tracking

**Recommendation**:
Extract to domain models:
- `Authentication` (PORO with ActiveModel)
- `UserProfile` (PORO)
- `NotificationSettings` (PORO)
- `BillingInfo` (PORO)

Example extraction:
```ruby
# app/models/user_profile.rb
class UserProfile
  include ActiveModel::Model
  
  attr_accessor :user, :bio, :avatar, :website
  
  def initialize(user)
    @user = user
    @bio = user.bio
    # ...
  end
  
  def update(attributes)
    # ...
  end
end
```

### Medium Severity

#### [ISSUE_ID] Callback with Business Logic

**File**: `app/models/order.rb:15`
**Impact**: Side effects hard to test, can cause unexpected behavior

**Current Code**:
```ruby
after_create :send_confirmation_email
after_create :update_inventory
after_create :notify_warehouse
```

**Recommendation**:
Move to explicit method calls in controller or form object:
```ruby
# app/models/order_placement.rb
class OrderPlacement
  include ActiveModel::Model
  
  def complete
    return false unless order.save
    
    send_confirmation_email
    update_inventory
    notify_warehouse
    true
  end
end
```

---

## 4. Controllers Issues

### High Severity

#### [ISSUE_ID] Fat Controller: OrdersController#create

**File**: `app/controllers/orders_controller.rb:45-120`
**Lines**: 75 lines in single action
**Impact**: Hard to test, violates SRP

**Current Code**:
```ruby
def create
  # 75 lines of business logic
end
```

**Recommendation**:
Extract to form object:
```ruby
def create
  @order_form = OrderForm.new(order_params)
  
  if @order_form.submit
    redirect_to @order_form.order
  else
    render :new
  end
end
```

---

## 5. Code Design Issues

### High Severity

#### [ISSUE_ID] Service Object Should Be Domain Model

**File**: `app/services/user_registration_service.rb`
**Impact**: Poor naming obscures domain concept

**Current Code**:
```ruby
class UserRegistrationService
  def self.call(params)
    # registration logic
  end
end
```

**Recommendation**:
Rename to domain model with ActiveModel:
```ruby
# app/models/registration.rb
class Registration
  include ActiveModel::Model
  
  attr_accessor :email, :password, :company_name
  
  validates :email, presence: true
  validates :password, presence: true, length: { minimum: 8 }
  
  def complete
    return false unless valid?
    
    create_user
    create_company
    send_welcome_email
    true
  end
end
```

### Medium Severity

#### [ISSUE_ID] Feature Envy

**File**: `app/models/report.rb:34`
**Impact**: Logic should be moved to collaborator

**Current Code**:
```ruby
def user_summary
  "#{user.first_name} #{user.last_name} (#{user.email})"
end
```

**Recommendation**:
Move to User model or UserPresenter:
```ruby
# In User model
def full_name_with_email
  "#{full_name} (#{email})"
end
```

---

## 6. Views Issues

### Medium Severity

#### [ISSUE_ID] Logic in View

**File**: `app/views/orders/show.html.erb:23`
**Impact**: Views should be logic-free

**Current Code**:
```erb
<% if @order.status == 'pending' && @order.created_at > 1.hour.ago %>
  <span class="badge warning">Processing</span>
<% elsif @order.status == 'shipped' %>
  <span class="badge success">Shipped</span>
<% end %>
```

**Recommendation**:
Extract to presenter or helper:
```ruby
# app/models/order_presenter.rb
class OrderPresenter
  def status_badge
    case
    when pending_and_recent? then { class: "warning", text: "Processing" }
    when shipped? then { class: "success", text: "Shipped" }
    end
  end
end
```

---

## 7. External Services Issues

### High Severity

#### [ISSUE_ID] Fire and Forget: Missing Exception Handling

**File**: `app/models/webhook_sender.rb:15`
**Impact**: Errors silently swallowed, no visibility into failures
**Details**: External API call without specific exception handling

**Current Code**:
```ruby
def send_webhook(data)
  ApiClient.post("/webhook", data)
rescue
  nil  # Swallows ALL errors including bugs
end
```

**Recommendation**:
```ruby
HTTP_ERRORS = [
  Timeout::Error, Errno::ECONNRESET, Net::HTTPBadResponse,
  SocketError, Errno::ECONNREFUSED
].freeze

def send_webhook(data)
  ApiClient.post("/webhook", data)
rescue *HTTP_ERRORS => e
  ErrorTracker.notify(e)
  nil
end
```

### Medium Severity

#### [ISSUE_ID] Missing Timeout on HTTP Client

**File**: `app/services/api_client.rb:8`
**Impact**: Requests can hang indefinitely, blocking web workers
**Details**: HTTP client uses default 60-second timeout

**Recommendation**:
```ruby
http = Net::HTTP.new(uri.host, uri.port)
http.open_timeout = 5
http.read_timeout = 5
```

---

## 8. Database & Performance Issues

### High Severity

#### [ISSUE_ID] Missing Index on Foreign Key

**File**: `db/schema.rb`
**Impact**: Slow queries on joins and lookups
**Details**: Column `posts.user_id` has no index

**Recommendation**:
```ruby
# db/migrate/XXXXXXXX_add_index_to_posts_user_id.rb
add_index :posts, :user_id
```

#### [ISSUE_ID] Ruby Iteration Instead of SQL

**File**: `app/models/order.rb:45`
**Impact**: Loads all records into memory, very slow on large datasets
**Details**: Using Ruby to filter records that should be SQL query

**Current Code**:
```ruby
def self.high_value
  all.select { |o| o.total > 100 }
end
```

**Recommendation**:
```ruby
def self.high_value
  where("total > ?", 100)
end
```

### Medium Severity

#### [ISSUE_ID] Silent Failure in Background Job

**File**: `app/jobs/process_order_job.rb:12`
**Impact**: Orders may fail processing without any notification
**Details**: Using `save` without checking return value

**Current Code**:
```ruby
def perform(order)
  order.status = "processed"
  order.save
end
```

**Recommendation**:
```ruby
def perform(order)
  order.update!(status: "processed")
end
```

#### [ISSUE_ID] Model Reference in Migration

**File**: `db/migrate/20240115_backfill_counts.rb`
**Impact**: Migration may fail if model changes, breaks future deployments
**Details**: Migration references `User` model directly

**Recommendation**:
Use raw SQL or define model inline:
```ruby
class BackfillCounts < ActiveRecord::Migration[7.1]
  def up
    execute <<-SQL
      UPDATE users SET posts_count = (
        SELECT count(*) FROM posts WHERE posts.user_id = users.id
      )
    SQL
  end
end
```

---

## Recommendations Summary

### Quick Wins (Immediate Action)

1. [ ] Fix SQL injection in `app/models/search.rb:23`
2. [ ] Add strong parameters to `users_controller.rb`
3. [ ] Create spec for `Order` model

### Short-term (This Sprint)

1. [ ] Extract `UserRegistrationService` to `Registration` PORO
2. [ ] Split `User` model into focused domain models
3. [ ] Move callbacks from `Order` to explicit form object

### Long-term (Technical Debt)

1. [ ] Achieve 80% test coverage
2. [ ] Refactor all service objects to domain models
3. [ ] Extract presenters for complex view logic

---

## Files Analyzed

| Directory | Files Analyzed | Issues Found |
|-----------|----------------|--------------|
| app/models/ | X | X |
| app/controllers/ | X | X |
| app/services/ | X | X |
| app/views/ | X | X |
| app/helpers/ | X | X |
| app/jobs/ | X | X |
| db/migrate/ | X | X |
| config/ | X | X |
| spec/ or test/ | X | X |
| **Total** | **X** | **X** |

---

## Appendix: Tools Recommendations

For ongoing code quality:

1. **RuboCop** - Style and lint checking
2. **Brakeman** - Security scanning
3. **SimpleCov** - Test coverage
4. **Bullet** - N+1 query detection
5. **flog/flay** - Complexity metrics
6. **bundler-audit** - Gem vulnerability scanning
7. **database_consistency** - Missing indexes and constraints detection
8. **strong_migrations** - Catch unsafe migrations

---

*Report generated by Rails Audit Skill (thoughtbot Best Practices)*
```
