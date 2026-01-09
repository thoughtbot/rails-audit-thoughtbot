# Code Smells Reference (Ruby Science)

## Detection Checklist

For each code smell, identify the pattern, assess severity, and recommend a solution.

---

## 1. Long Method

**Pattern**: Methods that are difficult to understand at a glance.

**Detection**:
- Methods with more than 10 lines of code
- More than one level of nesting
- More than one level of abstraction
- flog score of 10 or higher

**Severity**: Medium

**Solutions**:
- Extract Method
- Replace Temp with Query

**Example Issue**:
```ruby
# Bad: Long method with multiple responsibilities
def create
  @survey = Survey.find(params[:survey_id])
  @submittable_type = params[:submittable_type_id]
  question_params = params.require(:question).permit(:title, :options)
  @question = @survey.questions.new(question_params)
  @question.submittable_type = @submittable_type
  if @question.save
    redirect_to @survey
  else
    render :new
  end
end
```

---

## 2. Large Class

**Pattern**: Classes with too many responsibilities.

**Detection**:
- Can't describe class in one sentence
- More than 200 lines
- More than 15 public methods
- More than 7 private methods
- flog score of 50 or higher
- Changes for multiple unrelated reasons

**Severity**: High

**Solutions**:
- Extract Class
- Extract Value Object
- Extract Decorator
- Move Method
- Replace Subclasses with Strategies

---

## 3. Feature Envy

**Pattern**: Method that uses another object's data more than its own.

**Detection**:
- Repeated references to same external object
- Parameters used more than instance variables
- Methods with another class name in their name (e.g., `invite_user`)
- Law of Demeter violations

**Severity**: Medium

**Solutions**:
- Move Method
- Extract Method then Move Method

**Example Issue**:
```ruby
# Bad: Feature envy - using answer more than self
def score
  answers.inject(0) do |result, answer|
    question = answer.question
    result + question.score(answer.text)
  end
end
```

---

## 4. Case Statement / Type Code

**Pattern**: Conditional logic based on object type.

**Detection**:
- `case` statements checking class or type code
- Multiple `if/elsif` checking same condition
- `when` clauses that could be polymorphic methods

**Severity**: High (causes shotgun surgery)

**Solutions**:
- Replace Conditional with Polymorphism
- Replace Type Code with Subclasses
- Use Convention over Configuration

**Example Issue**:
```ruby
# Bad: Case statement on type
def summary
  case question_type
  when 'MultipleChoice'
    summarize_multiple_choice_answers
  when 'Open'
    summarize_open_answers
  when 'Scale'
    summarize_scale_answers
  end
end
```

---

## 5. Shotgun Surgery

**Pattern**: Single change requires edits across multiple files.

**Detection**:
- Same small change needed in multiple places
- Duplicated case statements
- Duplicated type checks

**Severity**: High

**Solutions**:
- Replace Conditional with Polymorphism
- Use Convention over Configuration
- Inline Class (if class adds no value)

---

## 6. Divergent Change

**Pattern**: Class changes for multiple unrelated reasons.

**Detection**:
- Class changed more frequently than others
- Different changes aren't related to each other

**Severity**: High

**Solutions**:
- Extract Class
- Move Method
- Extract Validator
- Introduce Form Object

---

## 7. Long Parameter List

**Pattern**: Methods with too many arguments.

**Detection**:
- Methods with more than 3 arguments
- Difficulty changing argument order
- Complex method due to parameter combinations

**Severity**: Medium

**Solutions**:
- Introduce Parameter Object
- Extract Class

**Example Issue**:
```ruby
# Bad: Long parameter list
def completion_notification(first_name, last_name, email, phone, company)
  # ...
end
```

---

## 8. Duplicated Code

**Pattern**: Same or similar code in multiple places.

**Detection**:
- Copy-pasted code blocks
- Similar logic with minor variations
- Parallel class hierarchies

**Severity**: High

**Solutions**:
- Extract Method
- Extract Class
- Extract Partial (views)
- Replace Conditional with Polymorphism

---

## 9. Mixin Abuse

**Pattern**: Using mixins inappropriately.

**Detection**:
- Mixins with methods that accept same parameters repeatedly
- Mixins that don't reference the state of mixed-in class
- Business logic trapped in mixins
- Classes with few public methods except from mixins

**Severity**: Medium

**Solutions**:
- Extract Class
- Replace Mixin with Composition

---

## 10. Callback Complexity

**Pattern**: Complex business logic in ActiveRecord callbacks.

**Detection**:
- `after_create`, `before_save` with business logic
- Callbacks that send emails or process payments
- Attributes to skip callbacks (e.g., `save_without_email`)
- Conditional callbacks

**Severity**: High

**Solutions**:
- Replace Callback with Method
- Extract to PORO

**Example Issue**:
```ruby
# Bad: Business logic in callback
class Invitation < ActiveRecord::Base
  after_create :deliver
  
  def deliver
    Mailer.invitation_notification(self).deliver
  end
end
```

---

## 11. Comments (Code Smell)

**Pattern**: Comments that explain what code does.

**Detection**:
- Comments within method bodies
- Comments restating method name
- TODO comments
- Commented-out code

**Severity**: Low

**Solutions**:
- Introduce Explaining Variable
- Extract Method with descriptive name
- Delete dead code

---

## 12. Single Table Inheritance (STI) Issues

**Pattern**: Problematic use of STI.

**Detection**:
- Need to change from one subclass to another
- Behavior shared among some but not all subclasses
- Subclass is fusion of other subclasses
- Lots of nil columns in database

**Severity**: Medium

**Solutions**:
- Replace Subclasses with Strategies
- Use Polymorphic Associations instead

---

## 13. God Class

**Pattern**: Class that knows too much about the system.

**Detection**:
- References to most other models
- Difficult to answer questions without this class
- Very high number of methods and lines
- Common in `User` model or central domain object

**Severity**: Critical

**Solutions**:
- Extract Class aggressively
- Use composition
- Introduce domain-specific objects

---

## Priority Order for Addressing Smells

1. **Critical**: God Class, Security issues
2. **High**: Duplicated Code, Case Statements, Large Class, Callback Complexity
3. **Medium**: Long Method, Feature Envy, Long Parameter List, Mixin Abuse
4. **Low**: Comments, Naming issues
