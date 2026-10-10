# Code Requirements & Standards

## Entry 1: Core Engineering & Coding Best Practices

### 1. Write Readable and Clear Code
- **Prefer clarity over cleverness**: Code is read far more often than it is written.
- **Meaningful naming**: Use meaningful, descriptive names for variables, functions, classes, and files. Avoid abbreviations unless they are universally understood (e.g., `id`, `url`).
- **Function size**: Keep functions/methods short and focused on a single responsibility (ideally under 20–40 lines).
- **Limit nesting depth**: Prefer early returns or guard clauses over deeply nested `if` statements.
- **Consistent style**: Follow consistent style (indentation, spacing, brace placement). Use automated formatters (Prettier, Black, `gofmt`, `rustfmt`, etc.).

### 2. Follow Established Conventions
- **Language-specific style guides**: Adhere to established style guides (PEP 8 for Python, Google Java Style, Effective Go, Airbnb JavaScript, etc.).
- **Consistent naming conventions**:
  - `camelCase` or `snake_case` for variables/functions (language-dependent)
  - `PascalCase` for classes/types
  - `SCREAMING_SNAKE_CASE` for constants
- **Modular structure**: Organize code into logical modules/packages with clear boundaries.

### 3. Apply Core Design Principles
- **DRY (Don’t Repeat Yourself)**: Extract common logic into reusable functions or modules.
- **KISS (Keep It Simple, Stupid)**: Prefer simple solutions over complex ones.
- **YAGNI (You Aren’t Gonna Need It)**: Don’t build features “just in case.”
- **SOLID Principles** (especially useful in object-oriented code):
  - **S**ingle Responsibility Principle
  - **O**pen/Closed Principle
  - **L**iskov Substitution Principle
  - **I**nterface Segregation Principle
  - **D**ependency Inversion Principle
- **Composition over inheritance**: Prefer composition over inheritance when possible.

### 4. Handle Errors Properly
- **Don't ignore errors or exceptions**: Handle them explicitly or propagate them intentionally.
- **Appropriate mechanisms**: Use appropriate error-handling mechanisms for the language (exceptions, Result/Option types, error codes).
- **Meaningful messages**: Provide meaningful error messages that help diagnose problems.
- **Fail fast**: Fail fast when preconditions are violated.
- **Avoid broad catches**: Avoid catching broad exceptions unless you rethrow or log them usefully.

### 5. Write Tests
- **Multi-level automated tests**: Aim for automated tests at multiple levels: unit, integration, and end-to-end (E2E).
- **Testing pyramid**: Follow the testing pyramid (many fast unit tests, fewer integration tests, even fewer E2E tests).
- **Test quality**: Write tests that are independent, deterministic, and fast.
- **Comprehensive coverage**: Test edge cases, error paths, and boundary conditions—not just the happy path.
- **Methodology**: Consider Test-Driven Development (TDD) or Behavior-Driven Development (BDD) when they fit the team and problem.

### 6. Document Thoughtfully
- **Self-documenting code**: Write self-documenting code first. Comments should explain *why*, not *what*.
- **API & architecture docs**: Document public APIs, complex algorithms, and non-obvious decisions.
- **Proximity**: Keep documentation close to the code (docstrings, JSDoc, etc.) and keep it up to date.
- **README**: Maintain a clear `README` with setup instructions, architecture overview, and contribution guidelines.

### 7. Use Version Control Effectively
- **Descriptive commits**: Commit frequently with clear, concise messages that explain the *why*.
- **Branching strategy**: Use meaningful branch names and follow a branching strategy (Git Flow, trunk-based development, GitHub Flow, etc.).
- **Focused scope**: Keep commits focused (one logical change per commit).
- **Collaborative reviews**: Review code via pull/merge requests. Treat code reviews as collaborative improvement, not gatekeeping.

### 8. Prioritize Security
- **Protect secrets**: Never hard-code secrets (API keys, passwords, tokens). Use environment variables or secret managers.
- **Input validation**: Validate and sanitize all external input.
- **Least privilege**: Follow the principle of least privilege.
- **Dependency management**: Keep dependencies updated and scan for known vulnerabilities.
- **Defensive programming**: Be cautious with deserialization, SQL queries (prefer parameterized queries), and authentication/authorization logic.

### 9. Consider Performance and Resource Use (When It Matters)
- **Measure before optimizing**: Premature optimization is a common source of complexity.
- **Complexity awareness**: Be mindful of algorithmic complexity (Big-O), memory usage, and I/O.
- **Efficient data structures**: Prefer efficient data structures and avoid unnecessary work in hot paths.
- **Profile bottlenecks**: Profile real bottlenecks rather than guessing.

### 10. Maintainability and Collaboration Practices
- **Codebase consistency**: Keep the codebase consistent. Enforce style and quality with linters and static analysis tools.
- **Boy-scout rule**: Refactor regularly (leave the code cleaner than you found it).
- **Short-lived branches**: Avoid large, long-lived feature branches when possible.
- **Disposability**: Write code that is easy to delete or replace.
- **Explicit over implicit**: Prefer explicit over implicit behavior.

---

## Practical Tools That Enforce Good Practices
- **Linters and formatters** (e.g., ESLint, Prettier, Black, Flake8, Ruff)
- **Static type checkers** (e.g., TypeScript, mypy, Pyright)
- **Static analysis & security scanners** (e.g., SonarQube, Snyk, CodeQL)
- **CI/CD pipelines** that run tests, linting, and security checks on every change
- **Pre-commit hooks** (e.g., Husky, `pre-commit`)

---

## Quick Mental Checklist Before Committing
- [ ] Does this change do one clear thing?
- [ ] Is the code easy for someone else (or future you) to understand?
- [ ] Are edge cases and errors handled?
- [ ] Are there tests for the important behavior?
- [ ] Did I avoid introducing unnecessary complexity or duplication?
