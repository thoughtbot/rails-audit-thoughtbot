# Rails Antipatterns Reference

> Based on *Rails Antipatterns: Best Practice Ruby on Rails Refactoring* by Chad Pytel & Tammer Saleh (thoughtbot)

This reference covers antipatterns that **extend** the existing audit references. It focuses on areas not fully covered elsewhere: external service handling, database/migration hygiene, performance patterns, and failure handling.

**Cross-references:**
- For code smells (Large Class, Feature Envy, Long Method): see `code_smells.md`
- For security issues (SQL injection, XSS, CSRF): see `security_checklist.md`
- For PORO patterns and Service Object refactoring: see `poro_patterns.md`
- For testing antipatterns: see `testing_guidelines.md`

---

## 1. Voyeuristic Models (Law of Demeter Extensions)

> Extends `code_smells.md` → Feature Envy

**Pattern**: Controllers or views reaching deep into model associations.

**Detection**:
- Chained association access in views: `@invoice.customer.address.street`
- Raw `where`/`order`/`limit` chains in controllers on associations
- Finders defined on wrong model (crossing association boundaries)

**Severity**: Medium

**Solutions**:
```ruby
# Use delegate with prefix
class Invoice < ApplicationRecord
  belongs_to :customer
  delegate :street, :city, :zip, to: :address, prefix: true, allow_nil: true
  # Now: invoice.address_street instead of invoice.customer.address.street
end

# Access finders through association proxy
@memberships = @user.memberships.recently_active  # Good
@memberships = Membership.where(user: @user, active: true).limit(5)  # Bad
```

**Audit Check**: Search views (`app/views/`) for patterns like `@var.association.association.method` (3+ levels). Search controllers for `.where`/`.order`/`.limit` chains on associations.

---

## 2. Monolithic Controllers (REST Violations)

> Extends `code_smells.md` → Large Class

**Pattern**: Controllers that don't embrace REST — custom actions instead of resources.

**Detection**:
- Non-RESTful action names: `search`, `export`, `activate`, `process_payment`
- Routes with excessive `member` or `collection` blocks
- Controller actions > 7 (beyond standard REST actions)
- Conditional logic based on nested resource presence

**Severity**: High

**Solutions**:
```ruby
# Bad: Monolithic controller
class UsersController < ApplicationController
  def activate; end    # Should be: Activations#create
  def deactivate; end  # Should be: Activations#destroy
  def export; end      # Should be: Exports#show
end

# Good: Separate REST controllers
class User::ActivationsController < ApplicationController
  def create; end   # activate
  def destroy; end  # deactivate
end
```

**Audit Check**: Count public actions per controller. Flag controllers with > 7 actions. Check `config/routes.rb` for excessive `member`/`collection` blocks.

---

## 3. Bloated Sessions

> Extends `security_checklist.md` → Session Security

**Pattern**: Storing full objects in session instead of lightweight references.

**Detection**:
- `session[:user] = @user` (full ActiveRecord object)
- `session[:cart] = @cart_items` (collections)
- Session data containing hashes with model attributes

**Severity**: High (causes stale data, serialization issues)

**Solutions**:
```ruby
# Bad: Storing object (stale data, serialization issues)
session[:user] = User.find(params[:id])

# Good: Store only the reference
session[:user_id] = params[:id]

def current_user
  @current_user ||= User.find_by(id: session[:user_id])
end
```

**Audit Check**: Search for `session[` assignments. Flag any storing non-scalar values.

---

## 4. Fire and Forget (External Service Errors)

**Pattern**: Calling external services without proper exception handling.

**Detection**:
- HTTP calls without `rescue` blocks
- Bare `rescue` or `rescue => e` without specific exception classes
- `rescue nil` on external service calls
- `config.action_mailer.raise_delivery_errors = false` without alternative handling

**Severity**: High

**Solutions**:
```ruby
# Bad: Bare rescue hides real problems
def send_to_api(data)
  ApiClient.post("/webhook", data)
rescue
  nil  # Silently swallows ALL errors
end

# Good: Explicit exception handling with reporting
HTTP_ERRORS = [
  Timeout::Error, Errno::ECONNRESET, Net::HTTPBadResponse,
  Net::HTTPHeaderSyntaxError, Net::ProtocolError, EOFError,
  SocketError, Errno::ECONNREFUSED
].freeze

def send_to_api(data)
  ApiClient.post("/webhook", data)
rescue *HTTP_ERRORS => e
  ErrorTracker.notify(e)  # Sentry, Honeybadger, etc.
  nil
end
```

**Audit Check**:
- Search for `rescue\s*$` or `rescue\s*=>` (bare rescue)
- Search for `rescue nil`
- Check `action_mailer` config for suppressed errors

---

## 5. Sluggish Services (Missing Timeouts)

**Pattern**: External service calls blocking requests without timeout configuration.

**Detection**:
- HTTP calls without explicit timeout settings
- Synchronous API calls in request cycle that could be backgrounded
- Default `Net::HTTP` timeout (60 seconds) unchanged

**Severity**: High

**Solutions**:
```ruby
# Bad: No timeout — blocks up to 60 seconds
Net::HTTP.get(URI("https://slow-api.example.com/data"))

# Good: Explicit timeouts
uri = URI("https://slow-api.example.com/data")
http = Net::HTTP.new(uri.host, uri.port)
http.open_timeout = 5
http.read_timeout = 5
http.request(Net::HTTP::Get.new(uri))

# Better: Use Faraday with timeouts
conn = Faraday.new(url: "https://api.example.com") do |f|
  f.options.timeout = 5
  f.options.open_timeout = 2
end

# Best: Background non-critical calls
SendWebhookJob.perform_later(data)
```

**Audit Check**: Search for `Net::HTTP`, `Faraday`, `HTTParty`, `RestClient` usage. Verify timeout configuration. Flag synchronous API calls in controllers that could be backgrounded.

---

## 6. Messy Migrations

**Pattern**: Migrations that reference external model code or lack reversibility.

**Detection**:
- Model class references inside migrations (e.g., `User.all.each`)
- Missing `down` method for non-reversible changes
- Migrations modified after being committed
- Missing `reset_column_information` after schema changes in data migrations

**Severity**: Medium

**Solutions**:
```ruby
# Bad: External model dependency — breaks if User model changes
class AddJobsCountToUser < ActiveRecord::Migration[7.1]
  def up
    add_column :users, :jobs_count, :integer, default: 0
    User.all.each { |u| u.update!(jobs_count: u.jobs.size) }
  end
end

# Good: Pure SQL, no external dependencies
class AddJobsCountToUser < ActiveRecord::Migration[7.1]
  def up
    add_column :users, :jobs_count, :integer, default: 0
    execute <<-SQL
      UPDATE users SET jobs_count = (
        SELECT count(*) FROM jobs WHERE jobs.user_id = users.id
      )
    SQL
  end

  def down
    remove_column :users, :jobs_count
  end
end

# If model needed, define inline
class BackfillData < ActiveRecord::Migration[7.1]
  class User < ApplicationRecord
    self.table_name = "users"
  end

  def up
    User.reset_column_information
    # Safe to use inline User class
  end
end
```

**Audit Check**: Search `db/migrate/` for model class names (e.g., `User.`, `Order.`). Verify each migration has reversible `down` or uses only reversible `change` methods.

---

## 7. Missing Database Indexes

**Pattern**: Tables missing indexes on commonly queried columns.

**Detection**:
- Foreign key columns (`*_id`) without indexes
- Polymorphic type/id pairs without composite indexes
- Columns used in `validates :uniqueness` without unique indexes
- STI `type` columns without indexes
- Columns used in `to_param` (slugs) without indexes
- State/status columns used in `where` without indexes

**Severity**: High (performance)

**Solutions**:
```ruby
# Foreign keys need indexes
add_index :posts, :user_id

# Polymorphic associations need composite indexes
add_index :comments, [:commentable_type, :commentable_id]

# Uniqueness validations need unique indexes
add_index :users, :email, unique: true

# STI needs type index
add_index :vehicles, :type

# Slugs need indexes
add_index :posts, :slug, unique: true
```

**Audit Check**:
```bash
# Find foreign key columns without indexes
# Compare *_id columns in schema.rb against add_index statements
```

Flag: Any `*_id` column without index. Any `validates :uniqueness` without database-level unique index.

---

## 8. Painful Performance (Ruby vs SQL)

**Pattern**: Doing in Ruby what should be done in SQL.

**Detection**:
- `Model.all.select`, `all.map`, `all.reject` (loading entire tables)
- Ruby sorting/filtering after loading records
- `association.length` instead of `association.count`
- Ruby `inject`/`reduce` for sums that could be SQL aggregations

**Severity**: High

**Solutions**:
```ruby
# Bad: Loads ALL orders into Ruby memory
Order.all.select { |o| o.total > 100 }

# Good: Database does the work
Order.where("total > ?", 100)

# Bad: Loads all records to count
user.posts.length

# Good: SQL count
user.posts.count
# Or: user.posts.size (uses counter cache if available)

# Bad: Ruby sum
Order.all.map(&:total).sum

# Good: SQL sum
Order.sum(:total)
```

**Audit Check**: Search for `.all.select`, `.all.map`, `.all.reject`, `.all.each` in models/controllers. Flag `.length` on associations.

---

## 9. Inaudible Failures (Silent Errors)

**Pattern**: Code that fails silently — `save` without checking return value, missing preconditions.

**Detection**:
- `save` without checking return value (in jobs, services, rake tasks)
- `update` without checking return value
- Bulk operations without transactions
- Missing fail-fast precondition checks

**Severity**: High

**Solutions**:
```ruby
# Bad: Silent failure
class Ticket < ApplicationRecord
  def self.bulk_change_owner(user)
    all.each do |ticket|
      ticket.owner = user
      ticket.save  # Returns false silently on failure
    end
  end
end

# Good: Fail loudly with transaction
class Ticket < ApplicationRecord
  def self.bulk_change_owner(user)
    transaction do
      all.find_each do |ticket|
        ticket.update!(owner: user)
      end
    end
  end
end

# In background jobs: Use bang methods
class ProcessOrderJob < ApplicationJob
  def perform(order)
    order.process!  # Raises on failure
    order.save!     # Raises on failure
  end
end

# Fail-fast preconditions
def process_payment(order)
  raise ArgumentError, "Order required" unless order
  raise InvalidStateError, "Order not ready" unless order.ready?
  # ... process
end
```

**Audit Check**: Search for `\.save\b` (without `!`) in models, jobs, service objects. Flag bulk operations without `transaction`. Search for bare `rescue` statements.

---

## 10. Spaghetti SQL (Query Logic in Controllers)

> Extends `code_smells.md` → Duplicated Code

**Pattern**: Raw query logic scattered outside models; not using scopes.

**Detection**:
- `where()`, `order()`, `joins()` chains in controllers
- Repeated query fragments across multiple actions
- Missing `scope` for reusable patterns

**Severity**: Medium

**Solutions**:
```ruby
# Bad: Query logic in controllers
# PostsController
@posts = Post.where(published: true).order(created_at: :desc)
# ApiController
@posts = Post.where(published: true).order(created_at: :desc).limit(10)

# Good: Composable scopes
class Post < ApplicationRecord
  scope :published, -> { where(published: true) }
  scope :newest_first, -> { order(created_at: :desc) }
  scope :recent, ->(n = 10) { newest_first.limit(n) }
end

# Controller
@posts = Post.published.newest_first
```

**Audit Check**: Search controllers for `.where`, `.order`, `.joins`, `.group` chains. Flag query patterns appearing in multiple locations.

---

## 11. PHPitis (Logic in Views)

**Pattern**: Business logic, queries, or complex conditionals in view templates.

**Detection**:
- Model queries in views: `<% User.where(...) %>`
- Nested `if/else` blocks (> 2 levels)
- Calculations or business logic in templates

**Severity**: Medium

**Solutions**:
```erb
<%# Bad: Logic in view %>
<% if @order.status == 'pending' && @order.created_at > 1.hour.ago %>
  <span class="warning">Processing</span>
<% elsif @order.status == 'shipped' && @order.tracking_number.present? %>
  <span class="success">Shipped</span>
<% end %>

<%# Good: Use presenter %>
<%= @order_presenter.status_badge %>
```

**Audit Check**: Search `app/views/` for `Model.find`, `Model.where`, `.order`, `.joins`. Flag views with > 2 levels of conditional nesting.

---

## 12. Gem Hygiene

> Extends `security_checklist.md` → Dependencies

**Pattern**: Unused gems, unvetted choices, or modified vendored code.

**Detection**:
- Gems in `Gemfile` not referenced in codebase
- Gems without recent maintenance (12+ months inactive)
- Modified files in `vendor/` directory
- Multiple gems solving same problem

**Severity**: Low

**TAM Criteria for Gem Selection**:
- **T**ests: Does it have a test suite?
- **A**ctivity: Recent commits and releases?
- **M**aturity: Stable API, good documentation?

**Audit Check**:
- Run `bundle-audit check` for vulnerabilities
- Search for gem names in codebase to find unused gems
- Check `vendor/` for local modifications (should fork instead)

---

## Priority Order for Addressing

1. **Critical/High**: Missing Indexes, Inaudible Failures, Fire and Forget, Sluggish Services
2. **High**: Monolithic Controllers, Bloated Sessions, Painful Performance
3. **Medium**: Voyeuristic Models, Spaghetti SQL, Messy Migrations, PHPitis
4. **Low**: Gem Hygiene

---

## Quick Reference: Detection Patterns

| Antipattern | Search Pattern | Files |
|-------------|----------------|-------|
| Voyeuristic Models | `@\w+\.\w+\.\w+\.` (3+ dots) | views/, controllers/ |
| Monolithic Controllers | Count public methods > 7 | controllers/ |
| Bloated Sessions | `session\[.*\] =` non-scalar | controllers/ |
| Fire and Forget | `rescue\s*$`, `rescue nil` | all .rb |
| Sluggish Services | `Net::HTTP`, `Faraday` without timeout | all .rb |
| Messy Migrations | `[A-Z][a-z]+\.` (model refs) | db/migrate/ |
| Missing Indexes | `*_id` without index | db/schema.rb |
| Painful Performance | `.all.select`, `.all.map` | models/, controllers/ |
| Inaudible Failures | `\.save\b` (without !) | models/, jobs/, services/ |
| Spaghetti SQL | `.where`, `.order` chains | controllers/ |
| PHPitis | `<% .*\.where`, nested `<% if` | views/ |
