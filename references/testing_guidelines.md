# Testing Guidelines Reference (Testing Rails)

## Test Suite Quality Characteristics

An effective test suite is:
- **Fast**: Run frequently, quick feedback loop
- **Complete**: All public code paths covered
- **Reliable**: No false positives or intermittent failures
- **Isolated**: Tests run independently, clean up after themselves
- **Maintainable**: Easy to add new tests and modify existing ones
- **Expressive**: Tests serve as documentation

---

## Testing Pyramid

Structure your test suite as a pyramid:
- **Base**: Many fast unit/model tests
- **Middle**: Some integration tests
- **Top**: Few slow feature/system tests

---

## Test Types Coverage Requirements

### Feature/System Specs (Integration)
**Required Coverage**:
- All critical user flows
- Happy paths for main features
- Key error handling paths

**Audit Checks**:
- [ ] Login/authentication flow tested
- [ ] Main CRUD operations tested
- [ ] Payment flows tested (if applicable)
- [ ] Critical business workflows tested

### Model Specs
**Required Coverage**:
- All validations
- All public instance methods
- All public class methods
- Associations (if complex logic)

**Audit Checks**:
- [ ] Each model has corresponding spec file
- [ ] All validations tested with shoulda-matchers
- [ ] Business logic methods have unit tests
- [ ] Edge cases covered

### Controller Specs (or Request Specs)
**Required Coverage**:
- Authorization checks
- Error handling paths
- Response formats (especially for APIs)

**Use When**:
- Testing authorization logic
- Multiple sad paths from same happy path
- URL/routing matters

### View Specs
**Required Coverage**:
- Conditional rendering logic
- Complex view logic

**Use When**:
- Significant conditional logic in views
- Avoiding duplicate feature specs

### Helper Specs
**Required Coverage**:
- All public helper methods

### Mailer Specs
**Required Coverage**:
- Email sent to correct recipients
- Correct subject
- Body contains expected content

---

## Four Phase Test Pattern

Every test should follow:

```ruby
it "does something" do
  # Setup - create objects and data
  user = create(:user)
  
  # Exercise - execute the code being tested
  result = user.full_name
  
  # Verify - check expectations
  expect(result).to eq "John Doe"
  
  # Teardown - handled by framework
end
```

**Audit Check**: Tests should have clear separation between phases.

---

## Testing Antipatterns to Flag

### 1. Slow Tests

**Symptoms**:
- Test suite takes more than 5 minutes
- Developers avoid running tests

**Causes**:
- Too many feature specs
- Not using factories efficiently
- Unnecessary database hits

**Audit Check**: Flag if average spec takes > 100ms

### 2. Intermittent Failures

**Symptoms**:
- Tests pass/fail randomly
- "Works on my machine"

**Causes**:
- Shared state between tests
- Time-dependent tests
- Order-dependent tests
- Race conditions in async code

**Audit Check**: Look for `sleep`, time manipulation without proper cleanup

### 3. Brittle Tests

**Symptoms**:
- Tests break when implementation changes
- Tests coupled to HTML structure

**Causes**:
- Testing implementation not behavior
- Over-reliance on specific selectors
- Excessive mocking

**Audit Check**: Flag tests with hardcoded CSS selectors, deep mocking

### 4. Duplication

**Symptoms**:
- Same setup code repeated
- Similar tests with minor variations

**Causes**:
- Missing shared examples
- Missing custom matchers
- Over-extracted test helpers

**Audit Check**: Look for repeated `let` blocks, identical setup

### 5. Mystery Guest

**Symptoms**:
- Test data defined elsewhere
- Hard to understand what test depends on

**Causes**:
- Over-use of fixtures
- Factory defaults that matter

**Audit Check**: Flag fixtures usage, flag factories with too many defaults

### 6. Stubbing System Under Test

**Symptoms**:
- Test stubs the object it's testing

**Causes**:
- Testing implementation details
- Poorly designed code

**Audit Check**: Flag `allow(subject).to receive(...)`

### 7. False Positives

**Symptoms**:
- Test passes but code is broken

**Causes**:
- Not testing the right thing
- Overly broad assertions

**Audit Check**: Look for `expect(page).to have_content("")`

### 8. Using Factories Like Fixtures

**Symptoms**:
- Named factories for every scenario
- `create(:admin_user_with_premium_subscription)`

**Causes**:
- Misunderstanding factory purpose

**Audit Check**: Flag factories with many trait combinations

### 9. Bloated Factories

**Symptoms**:
- Factories create unnecessary associations
- Factory creates too much data

**Causes**:
- Adding defaults "just in case"

**Audit Check**: Flag factories with > 5 attributes, unnecessary associations

### 10. Over-use of `let`, `subject`, `before`

**Symptoms**:
- Tests hard to read
- Must scroll to understand test

**Causes**:
- DRY taken too far in tests

**Audit Check**: Flag tests with > 5 `let` statements

---

## Coverage Requirements by File Type

| File Type | Min Coverage | Test Type |
|-----------|--------------|-----------|
| Model | 90% | Model spec |
| Controller | 80% | Request/Controller spec |
| Service/PORO | 95% | Unit spec |
| Helper | 100% | Helper spec |
| Mailer | 100% | Mailer spec |
| Job | 90% | Job spec |

---

## Missing Test Detection

For each Ruby file in `app/`:

1. Check for corresponding spec:
   - `app/models/user.rb` → `spec/models/user_spec.rb`
   - `app/controllers/users_controller.rb` → `spec/controllers/users_controller_spec.rb` or `spec/requests/users_spec.rb`

2. Check public methods are tested:
   - Extract public method names from source
   - Search for those names in spec file

3. Report:
   - Files without any tests → **High** severity
   - Files with partial coverage → **Medium** severity

---

## FactoryBot Best Practices

**Good Factory**:
```ruby
factory :link do
  title { "Testing Rails" }
  url { "http://example.com" }
  # Only required fields with sensible defaults
end
```

**Bad Factory**:
```ruby
factory :link do
  title { "Testing Rails" }
  url { "http://example.com" }
  upvotes { 10 }  # Not required
  user  # Creates unnecessary association
  created_at { 1.day.ago }  # Unnecessary
end
```

---

## RSpec Best Practices

**Good Test Structure**:
```ruby
RSpec.describe Link, "#score" do
  it "returns upvotes minus downvotes" do
    link = build(:link, upvotes: 5, downvotes: 2)
    
    expect(link.score).to eq 3
  end
end
```

**Avoid**:
- Nested contexts more than 2 levels deep
- `it` blocks without clear descriptions
- Multiple expectations per test (usually)
- Testing private methods directly
