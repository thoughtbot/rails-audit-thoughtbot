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

### Step 2: Collect Test Coverage Data (Optional)

#### Step 2.1 — Ask the User

Prompt the user whether they want to collect actual test coverage data using SimpleCov. Explain that this will temporarily install SimpleCov (if not already present), run the test suite, and capture real coverage metrics. If the user declines, skip to Step 3 and use estimation mode for the Testing section.

#### Step 2.2 — Detect Test Framework

- Check for `spec/` directory + `rspec-rails` in Gemfile → **RSpec**
- Check for `test/` directory → **Minitest**
- Determine the test helper file:
  - RSpec: `spec/rails_helper.rb` (preferred) or `spec/spec_helper.rb`
  - Minitest: `test/test_helper.rb`
- Determine the run command:
  - RSpec: `bundle exec rspec`
  - Minitest: `bundle exec rails test`

#### Step 2.3 — Check if SimpleCov Already Present

- Search `Gemfile` for `simplecov`
- Search the test helper for `SimpleCov.start`
- If both are present: set `SIMPLECOV_ALREADY_PRESENT = true`, skip setup and cleanup steps (2.4 and 2.6), just run tests and capture data
- If not present: proceed with full setup

#### Step 2.4 — Backup and Setup (skip if SimpleCov already present)

1. **Backup Gemfile and lockfile**:
   - Primary: `git stash push -m "rails-audit-simplecov-setup" -- Gemfile Gemfile.lock`
   - Fallback (if git stash fails): `cp Gemfile Gemfile.audit_backup && cp Gemfile.lock Gemfile.lock.audit_backup`

2. **Add SimpleCov to Gemfile**:
   - Look for `group :test do` in Gemfile and add `gem "simplecov", require: false` inside it
   - If no `group :test` block exists, use `group :development, :test do` instead

3. **Install the gem**: Run `bundle install`
   - If `bundle install` fails: abort coverage collection, restore backups, warn the user, and proceed with estimation mode

4. **Stop Spring** (if present): Check for `bin/spring` and run `bin/spring stop`

5. **Prepend SimpleCov configuration to test helper**:
   ```ruby
   require "simplecov"
   SimpleCov.start "rails" do
     enable_coverage :branch
     formatter SimpleCov::Formatter::JSONFormatter
   end
   ```
   Prepend these lines at the very top of the test helper file, before any other `require` statements.

#### Step 2.5 — Run Tests and Capture Coverage

1. Run the appropriate test command:
   - Full audit: run the full suite (`bundle exec rspec` or `bundle exec rails test`)
   - Targeted audit: run only tests relevant to the audit scope

2. Read `coverage/.resultset.json` and parse the coverage data.

3. **Parsing `.resultset.json`**: The file structure is:
   ```json
   {
     "RSpec": {
       "coverage": {
         "/path/to/app/models/user.rb": {
           "lines": [1, 1, null, 0, 0, 1],
           "branches": {}
         }
       }
     }
   }
   ```
   - Line coverage % = `(count of lines >= 1) / (count of lines != null) * 100`
   - Aggregate coverage by directory (`app/models/`, `app/controllers/`, etc.)
   - Extract per-file coverage percentages

4. Store parsed data in context for use in Step 5.

5. **If tests fail**: still read coverage data (SimpleCov writes results on exit regardless). Note test failures separately in the report.

6. **If `.resultset.json` is missing**: warn the user and fall back to estimation mode for the Testing section.

#### Step 2.6 — Cleanup (skip if SimpleCov was already present)

1. **Remove SimpleCov lines from test helper**: delete the prepended `require "simplecov"` and `SimpleCov.start` block
2. **Restore Gemfile**:
   - Primary: `git stash pop`
   - If stash pop conflicts: `git checkout -- Gemfile Gemfile.lock` then `bundle install`
   - Fallback: restore from `Gemfile.audit_backup` and `Gemfile.lock.audit_backup` copies, then delete the backup files
3. **Verify bundle**: run `bundle check` — if it fails, run `bundle install`
4. **Remove coverage directory**: `rm -rf coverage/`
5. **Verify clean state**: run `git status` to confirm no leftover changes

### Step 3: Load Reference Materials

Before analyzing, read the relevant reference files:
- `references/code_smells.md` - Code smell patterns to identify
- `references/testing_guidelines.md` - Testing best practices
- `references/poro_patterns.md` - PORO and ActiveModel patterns
- `references/security_checklist.md` - Security vulnerability patterns
- `references/rails_antipatterns.md` - Rails-specific antipatterns (external services, migrations, performance)

### Step 4: Analyze Code by Category

Analyze in this order:

1. **Testing Coverage & Quality**
   - If SimpleCov data was collected in Step 2, use actual coverage percentages instead of estimates
   - Cross-reference per-file SimpleCov data: files with 0% coverage = "missing tests"
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
   - Callback complexity
   - Law of Demeter violations (voyeuristic models)

4. **Controllers**
   - Fat controller detection
   - Business logic in controllers
   - Missing strong parameters
   - Response handling
   - Monolithic controllers (non-RESTful actions, > 7 actions)
   - Bloated sessions (storing objects instead of IDs)

5. **Code Design & Architecture**
   - Service Objects → recommend PORO refactoring
   - Large classes
   - Long methods
   - Feature envy
   - Law of Demeter violations
   - Single Responsibility violations

6. **Views & Presenters**
   - Logic in views (PHPitis)
   - Missing partials for DRY
   - Helper complexity
   - Query logic in views

7. **External Services & Error Handling**
   - Fire and forget (missing exception handling for HTTP calls)
   - Sluggish services (missing timeouts, synchronous calls that should be backgrounded)
   - Bare rescue statements
   - Silent failures (save without checking return value)

8. **Database & Migrations**
   - Messy migrations (model references, missing down methods)
   - Missing indexes on foreign keys, polymorphic associations, uniqueness validations
   - Performance antipatterns (Ruby iteration vs SQL queries)
   - Bulk operations without transactions

### Step 5: Generate Audit Report

Create `RAILS_AUDIT_REPORT.md` in project root with structure defined in `references/report_template.md`.

When SimpleCov coverage data was collected in Step 2, use the **SimpleCov variant** of the Testing section in the report template. When coverage data is not available, use the **estimation variant**.

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
