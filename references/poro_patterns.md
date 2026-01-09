# PORO Patterns Reference (Plain Old Ruby Objects)

## Philosophy: Service Objects Are Poorly-Named Models

Instead of creating "service objects" with generic names like `*Service`, `*Manager`, `*Handler`, create **domain models** with meaningful names that represent business concepts.

**Key Principle**: A representation of a business domain concept is called a **model**.

---

## Naming Convention

### Bad (Service Object Pattern)
```ruby
# app/services/notification_service.rb
class NotificationService
  def self.call(user, message)
    # sends notification
  end
end
```

### Good (Domain Model Pattern)
```ruby
# app/models/notification.rb
class Notification
  include ActiveModel::Model
  
  attr_accessor :user, :message
  
  def deliver
    # sends notification
  end
end
```

### Naming Guidelines

| Instead of | Use |
|------------|-----|
| `UserSignupService` | `Registration` or `UserSignup` |
| `PaymentProcessor` | `Payment` |
| `NotificationService` | `Notification` or `NotificationDelivery` |
| `EmailSender` | `Email` or `EmailMessage` |
| `OrderCreator` | `Order` or `OrderPlacement` |
| `InvitationManager` | `Invitation` |

**Rule**: Think of the **noun** or **noun group** that describes the domain concept.

---

## ActiveModel::Model

Use `ActiveModel::Model` to give POROs Rails form integration and validation.

### Basic Pattern

```ruby
class Registration
  include ActiveModel::Model
  
  attr_accessor :email, :password, :company_name
  
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :password, presence: true, length: { minimum: 8 }
  validates :company_name, presence: true
  
  def save
    return false unless valid?
    
    create_user
    create_company
    send_welcome_email
    true
  end
  
  private
  
  def create_user
    @user = User.create!(email: email, password: password)
  end
  
  def create_company
    @company = Company.create!(name: company_name, owner: @user)
  end
  
  def send_welcome_email
    RegistrationMailer.welcome(@user).deliver_later
  end
end
```

### Benefits of ActiveModel::Model

1. **Form Integration**: Works with `form_for` / `form_with`
2. **Validations**: Full ActiveRecord validation syntax
3. **Error Messages**: Display validation errors in forms
4. **Attribute Assignment**: Mass assignment from params
5. **Naming Conventions**: Auto-generates form paths

---

## Form Objects

Use when forms don't map 1:1 to a database model.

### Pattern

```ruby
# app/models/contact_form.rb
class ContactForm
  include ActiveModel::Model
  
  attr_accessor :name, :email, :message, :phone
  
  validates :name, presence: true
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :message, presence: true, length: { minimum: 10 }
  
  def submit
    return false unless valid?
    
    ContactMailer.new_inquiry(self).deliver_later
    true
  end
end
```

### Controller Usage

```ruby
class ContactController < ApplicationController
  def new
    @contact = ContactForm.new
  end
  
  def create
    @contact = ContactForm.new(contact_params)
    
    if @contact.submit
      redirect_to thank_you_path, notice: "Message sent!"
    else
      render :new
    end
  end
  
  private
  
  def contact_params
    params.require(:contact_form).permit(:name, :email, :message, :phone)
  end
end
```

### View Usage

```erb
<%= form_with model: @contact, url: contact_path do |f| %>
  <%= f.label :name %>
  <%= f.text_field :name %>
  
  <%= f.label :email %>
  <%= f.email_field :email %>
  
  <%= f.label :message %>
  <%= f.text_area :message %>
  
  <%= f.submit "Send Message" %>
<% end %>
```

---

## Search/Query Objects

Use for complex search forms with filtering and sorting.

### Pattern

```ruby
# app/models/post_search.rb
class PostSearch
  include ActiveModel::Model
  include ActiveModel::Attributes
  
  attribute :query, :string
  attribute :author_id, :integer
  attribute :status, :string
  attribute :sort_column, :string, default: "created_at"
  attribute :sort_direction, :string, default: "desc"
  
  def results
    scope = Post.all
    scope = scope.where("title ILIKE ?", "%#{query}%") if query.present?
    scope = scope.where(author_id: author_id) if author_id.present?
    scope = scope.where(status: status) if status.present?
    scope.order(sort_column => sort_direction)
  end
  
  def sort_options
    [
      ["Newest First", "created_at-desc"],
      ["Oldest First", "created_at-asc"],
      ["Title A-Z", "title-asc"],
      ["Title Z-A", "title-desc"]
    ]
  end
end
```

---

## Value Objects

Use for values with behavior that don't need persistence.

### Pattern

```ruby
# app/models/money.rb
class Money
  include Comparable
  
  attr_reader :cents, :currency
  
  def initialize(cents, currency = "USD")
    @cents = cents.to_i
    @currency = currency
  end
  
  def dollars
    cents / 100.0
  end
  
  def +(other)
    raise ArgumentError, "Currency mismatch" unless currency == other.currency
    Money.new(cents + other.cents, currency)
  end
  
  def <=>(other)
    cents <=> other.cents
  end
  
  def to_s
    format("$%.2f", dollars)
  end
end
```

---

## Decorators/Presenters

Use to add presentation logic without polluting models.

### Pattern

```ruby
# app/models/user_presenter.rb
class UserPresenter
  def initialize(user)
    @user = user
  end
  
  def full_name
    "#{@user.first_name} #{@user.last_name}"
  end
  
  def display_name
    @user.nickname.presence || full_name
  end
  
  def avatar_url
    @user.avatar.attached? ? @user.avatar.url : default_avatar_url
  end
  
  private
  
  def default_avatar_url
    "https://www.gravatar.com/avatar/#{Digest::MD5.hexdigest(@user.email)}"
  end
end
```

---

## Calculator/Query Objects

Use to extract complex calculations or queries.

### Pattern Using Namespaces

```ruby
# app/models/line_item.rb
class LineItem < ApplicationRecord
  def price
    LineItems::Price.new(self).calculate
  end
end

# app/models/line_items/price.rb
module LineItems
  class Price
    def initialize(line_item)
      @line_item = line_item
      @product = line_item.product
    end
    
    def calculate
      base_price + options_price - discount
    end
    
    private
    
    def base_price
      @product.base_price
    end
    
    def options_price
      @line_item.options.sum(&:price)
    end
    
    def discount
      @line_item.coupon&.discount_amount || 0
    end
  end
end
```

---

## Directory Structure

Prefer `app/models/` with namespaces over `app/services/`:

```
app/models/
├── user.rb
├── order.rb
├── registration.rb           # Form object
├── post_search.rb            # Search object
├── user_presenter.rb         # Presenter
├── orders/
│   ├── placement.rb          # Complex operation
│   ├── calculator.rb         # Price calculation
│   └── summary.rb            # Report generation
└── notifications/
    ├── delivery.rb
    └── payload.rb
```

---

## Refactoring Service Objects to POROs

### Step 1: Identify the Domain Concept

```ruby
# Before: What is this service doing?
class UserRegistrationService
  def self.call(params)
    # Creates user, sends email, creates trial subscription
  end
end

# After: It's handling a Registration
class Registration
  include ActiveModel::Model
  # ...
end
```

### Step 2: Add ActiveModel::Model

```ruby
class Registration
  include ActiveModel::Model
  
  attr_accessor :email, :password, :plan
  
  validates :email, presence: true
  validates :password, presence: true
end
```

### Step 3: Replace `.call` with Domain Method

```ruby
class Registration
  include ActiveModel::Model
  
  # Instead of .call, use a meaningful verb
  def complete
    return false unless valid?
    
    ActiveRecord::Base.transaction do
      create_user
      create_subscription
      send_welcome_email
    end
    
    true
  rescue ActiveRecord::RecordInvalid
    false
  end
end
```

### Step 4: Update Controller

```ruby
# Before
def create
  result = UserRegistrationService.call(params)
  if result.success?
    redirect_to dashboard_path
  else
    render :new
  end
end

# After
def create
  @registration = Registration.new(registration_params)
  
  if @registration.complete
    redirect_to dashboard_path
  else
    render :new
  end
end
```

---

## Audit Checklist

When reviewing service objects, check:

1. [ ] Is there a better noun name for this class?
2. [ ] Should it include `ActiveModel::Model`?
3. [ ] Does it have validations that could use Rails validators?
4. [ ] Is `.call` the only public method? (Replace with domain verb)
5. [ ] Is it in `app/services/`? (Consider `app/models/` with namespace)
6. [ ] Does the name end in `Service`, `Manager`, `Handler`? (Remove suffix)
