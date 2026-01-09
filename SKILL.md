---
name: rails-audit-thoughtbot
description: Perform comprehensive code audits of Ruby on Rails applications based on thoughtbot best practices. Use this skill when the user requests a code audit, code review, quality assessment, or analysis of a Rails application. The skill analyzes the entire codebase focusing on testing practices (RSpec), security vulnerabilities, code design (skinny controllers, domain models, PORO with ActiveModel), Rails conventions, database optimization, and Ruby best practices. Outputs a detailed markdown audit report grouped by category (Testing, Security, Models, Controllers, Code Design, Views) with severity levels (Critical, High, Medium, Low) within each category.
---

# Rails Audit Skill (thoughtbot Best Practices)

Perform comprehensive Ruby on Rails application audits based on thoughtbot's Ruby Science and Testing Rails best practices, with emphasis on Plain Old Ruby Objects (POROs) over Service Objects.

## Audit Scope

The audit can be run in two modes:
1. **Full Application Audit**: Analyze entire Rails application
2. **Targeted Audit**: Analyze specific files or directories

## Execution Flow

### Step 1: Determine Audit Scope

Ask user or infer from request:
- Full audit: Analyze all of `app/`, `spec/` or `test/`, `config/`, `db/`, `lib/`
- Targeted audit: Analyze specified paths only

### Step 2: Load Reference Materials

Before analyzing, read the relevant reference files:
- `references/code_smells.md` - Code smell patterns to identify
- `references/testing_guidelines.md` - Testing best practices
- `references/poro_patterns.md` - PORO and ActiveModel patterns

### Step 3: Analyze Code by Category

Analyze in this order:

1. **Testing Coverage & Quality**
   - Check for missing test files
   - Identify untested public methods
   - Review test structure (Four Phase Test)
   - Check for testing antipatterns

2. **Security Vulnerabilities**
   - SQL injection risks
   - Mass assignment vulnerabilities
   - XSS vulnerabilities
   - Authentication/authorization issues
   - Sensitive data exposure

3. **Models & Database**
   - Fat model detection
   - Missing validations
   - N+1 query risks
   - Missing indexes
   - Callback complexity

4. **Controllers**
   - Fat controller detection
   - Business logic in controllers
   - Missing strong parameters
   - Response handling

5. **Code Design & Architecture**
   - Service Objects → recommend PORO refactoring
   - Large classes
   - Long methods
   - Feature envy
   - Law of Demeter violations
   - Single Responsibility violations

6. **Views & Presenters**
   - Logic in views
   - Missing partials for DRY
   - Helper complexity

### Step 4: Generate Audit Report

Create `RAILS_AUDIT_REPORT.md` in project root with structure defined in `references/report_template.md`.

## Severity Definitions

- **Critical**: Security vulnerabilities, data loss risks, production-breaking issues
- **High**: Performance issues, missing tests for critical paths, major code smells
- **Medium**: Code smells, convention violations, maintainability concerns
- **Low**: Style issues, minor improvements, suggestions

## Key Detection Patterns

### Service Object → PORO Refactoring

When you find classes in `app/services/`:
- Classes named `*Service`, `*Manager`, `*Handler`
- Classes with only `.call` or `.perform` methods
- Recommend: Rename to domain nouns, include `ActiveModel::Model`

### Fat Model Detection

Models with:
- More than 200 lines
- More than 15 public methods
- Multiple unrelated responsibilities
- Recommend: Extract to POROs using composition

### Fat Controller Detection

Controllers with:
- Actions over 15 lines
- Business logic (not request/response handling)
- Multiple instance variable assignments
- Recommend: Extract to form objects or domain models

### Missing Test Detection

For each Ruby file in `app/`:
- Check for corresponding `_spec.rb` or `_test.rb`
- Check for tested public methods
- Report untested files and methods

## Analysis Commands

Use these bash patterns for file discovery:

```bash
# Find all Ruby files by type
find app/models -name "*.rb" -type f
find app/controllers -name "*.rb" -type f
find app/services -name "*.rb" -type f 2>/dev/null

# Find test files
find spec -name "*_spec.rb" -type f 2>/dev/null
find test -name "*_test.rb" -type f 2>/dev/null

# Count lines per file
wc -l app/models/*.rb

# Find long files (over 200 lines)
find app -name "*.rb" -exec wc -l {} + | awk '$1 > 200'
```

## Report Output

Always save the audit report to `/mnt/user-data/outputs/RAILS_AUDIT_REPORT.md` and present it to the user.
