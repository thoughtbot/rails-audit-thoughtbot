# Rails Audit Skill (thoughtbot Best Practices)

A [Claude Code][claude-code] skill that performs comprehensive code audits of
Ruby on Rails applications based on [thoughtbot's][thoughtbot] Ruby Science and
Testing Rails best practices.

[claude-code]: https://docs.anthropic.com/en/docs/claude-code
[thoughtbot]: https://thoughtbot.com

## Quick links

- **[Ruby Science][ruby-science]** - thoughtbot's guide to fixing code smells
- **[Testing Rails][testing-rails]** - thoughtbot's guide to testing Rails applications
- **[Rails Antipatterns][rails-antipatterns]** - Best practices for Ruby on 
  Rails refactoring (Chad Pytel & Tammer Saleh)

[ruby-science]: https://github.com/thoughtbot/ruby-science
[testing-rails]: https://github.com/thoughtbot/testing-rails
[rails-antipatterns]: https://www.informit.com/store/rails-antipatterns-best-practice-ruby-on-rails-refactoring-9780321604811

## Table of contents

- [Overview](#overview)
- [Installation](#installation)
- [Usage](#usage)
  - [Full application audit](#full-application-audit)
  - [Targeted audit](#targeted-audit)
- [Reference materials](#reference-materials)
- [Contributing](#contributing)
- [License](#license)
- [About thoughtbot](#about-thoughtbot)

## Overview

This skill analyses Rails applications and generates detailed audit reports
covering:

- Testing practices (RSpec)
- Security vulnerabilities
- Code design (skinny controllers, domain models, POROs with ActiveModel)
- Rails conventions
- Database optimisation (missing indexes, migrations hygiene)
- External services (timeouts, error handling, background jobs)
- Performance antipatterns (Ruby vs SQL, silent failures)
- Ruby best practices

## Installation

Copy the skill directory to your Claude Code skills folder:

```bash
cp -r rails-audit-thoughtbot ~/.claude/skills/
```

Or clone directly:

```bash
git clone https://github.com/thoughtbot/rails-audit-thoughtbot ~/.claude/skills/rails-audit-thoughtbot
```

## Usage

If you are in your terminal and not in a Claude session, you can invoke the 
skill directly by using the below. You need to be in the root directory of your 
Rails project.

### Full application audit

```
claude audit
```

If you are in a Claude session, you can reference the skill directly:

```
/rails-audit-thoughtbot
```

### Targeted audit

In a Claude session you can also run targeted audits:

```
/rails-audit-thoughtbot audit controllers
```

This focuses the audit on specific files or directories.

## Reference materials

The skill includes reference documentation based on thoughtbot best practices.
All the materials are compacted information from the books mentioned above.

Recommendations of PORO objects are based on different thoughtbot sources and
[Service objects are poorly-named models][service-objects-poro].

[service-objects-poro]: https://dimiterpetrov.com/blog/service-objects-are-poorly-named-models/

| File | Description |
|------|-------------|
| `references/code_smells.md` | Code smell patterns to identify (Ruby Science) |
| `references/testing_guidelines.md` | Testing best practices (Testing Rails) |
| `references/poro_patterns.md` | PORO and ActiveModel patterns |
| `references/security_checklist.md` | Security vulnerability checklist |
| `references/rails_antipatterns.md` | Rails-specific antipatterns: external services, migrations, performance |
| `references/report_template.md` | Audit report structure template |

## Contributing

Contributions are welcome! If you'd like to improve the audit patterns or add
new detection rules:

1. Fork the repository
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create a new Pull Request

## License

This skill is open source and available under the [MIT License](LICENSE).

## About thoughtbot

![thoughtbot](https://thoughtbot.com/thoughtbot-logo-for-readmes.svg)

This skill is inspired by and based on thoughtbot's excellent guides:

- [Ruby Science](https://github.com/thoughtbot/ruby-science)
- [Testing Rails](https://github.com/thoughtbot/testing-rails)
- [Rails Antipatterns](https://www.informit.com/store/rails-antipatterns-best-practice-ruby-on-rails-refactoring-9780321604811) by Chad Pytel & Tammer Saleh

The names and logos for thoughtbot are trademarks of thoughtbot, inc.

We love open source software!
See [thoughtbot's other projects][community].

[community]: https://thoughtbot.com/community?utm_source=github
