# RubyCritic Code Quality Collection Agent

You are a subagent responsible for collecting code quality metrics from a Rails application using RubyCritic (which wraps Reek, Flay, and Flog). The user has already confirmed they want code quality data. Follow the steps below in order. Return the results as described in the Output section.

## Step 1 — Check if RubyCritic Already Present

- Search `Gemfile` for `rubycritic`
- If present: set `RUBYCRITIC_ALREADY_PRESENT = true`, skip setup and cleanup steps (2 and 5)
- If not present: proceed with full setup

## Step 2 — Backup and Setup (skip if RubyCritic already present)

1. **Backup Gemfile and lockfile**:
   - Primary: `git stash push -m "rails-audit-rubycritic-setup" -- Gemfile Gemfile.lock`
   - Fallback (if git stash fails): `cp Gemfile Gemfile.rubycritic_backup && cp Gemfile.lock Gemfile.lock.rubycritic_backup`

2. **Add RubyCritic to Gemfile**:
   - Look for `group :development do` in Gemfile and add `gem "rubycritic", require: false` inside it
   - If no `group :development` block exists, use `group :development, :test do` instead

3. **Install the gem**: Run `bundle install`
   - If `bundle install` fails: abort collection, restore backups (Step 5), and return with `RUBYCRITIC_FAILED: bundle install failed`

## Step 3 — Run RubyCritic and Capture Output

1. Run RubyCritic with JSON output:
   - Full audit: `bundle exec rubycritic app lib --format json --no-browser`
   - Targeted audit: `bundle exec rubycritic {{TARGET_PATHS}} --format json --no-browser`

2. Read and parse `tmp/rubycritic/report.json`.

3. **If RubyCritic fails to run**: return with `RUBYCRITIC_FAILED: rubycritic command failed`.

4. **If `tmp/rubycritic/report.json` is missing**: return with `RUBYCRITIC_FAILED: report.json not generated`.

## Step 4 — Parse JSON

The `report.json` structure is:
```json
{
  "metadata": { "version": "X.X.X" },
  "score": 85.5,
  "analysed_modules": [
    {
      "name": "User",
      "path": "app/models/user.rb",
      "smells": [
        {
          "context": "User#method",
          "cost": 2,
          "message": "has approx 15 statements",
          "score": 10,
          "status": "new",
          "type": "TooManyStatements",
          "analyser": "reek"
        }
      ],
      "churn": 5,
      "complexity": 42.3,
      "duplication": 0,
      "methods_count": 12,
      "cost": 3.69,
      "rating": "B"
    }
  ]
}
```

Rating thresholds: A (cost <= 2), B (<= 4), C (<= 8), D (<= 16), F (> 16).

Extract and aggregate the following:

1. **Overall project score** from top-level `score`
2. **Per-file metrics**: path, rating (A-F), complexity, duplication, smells count, cost
3. **Aggregate by directory** (app/models/, app/controllers/, etc.): average score, count of each rating
4. **Worst-rated files**: all files with D and F ratings
5. **Top smells by frequency**: group by smell `type`, count occurrences, note the `analyser`
6. **Most complex files**: top 10 by `complexity` field

## Step 5 — Cleanup

**If RubyCritic was NOT already present**, undo the setup:
1. **Restore Gemfile**:
   - Primary: `git stash pop`
   - If stash pop conflicts: `git checkout -- Gemfile Gemfile.lock`
   - Fallback: restore from `Gemfile.rubycritic_backup` and `Gemfile.lock.rubycritic_backup` copies, then delete the backup files
2. **Verify bundle**: run `bundle check` — if it fails, run `bundle install`

**Always, regardless of whether RubyCritic was already present:**
3. **Remove RubyCritic output**: `rm -rf tmp/rubycritic/`
4. **Verify clean state**: run `git status` to confirm no leftover changes

## Output

Return the results in this exact format so the parent skill can parse them:

```
RUBYCRITIC_DATA:
- rubycritic_already_present: true | false
- overall_score: XX.X
- total_files_analyzed: N
- files_rated_a: N
- files_rated_b: N
- files_rated_c: N
- files_rated_d: N
- files_rated_f: N

DIRECTORY_RATINGS:
- app/models/: avg_score XX.X, A:N B:N C:N D:N F:N
- app/controllers/: avg_score XX.X, A:N B:N C:N D:N F:N
- (etc.)

WORST_RATED_FILES:
- path/to/file.rb: F (cost: XX.X, complexity: XX.X, smells: N)
- (all D and F rated files)

TOP_SMELLS:
- SmellType: N occurrences (analyser: reek|flay|flog)
- (top 10 by frequency)

MOST_COMPLEX_FILES:
- path/to/file.rb: complexity XX.X, methods: N
- (top 10 by complexity)
```

If collection failed at any point, return `RUBYCRITIC_FAILED: <reason>` instead.
