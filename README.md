<div align="center">

<h1>&#128269; Code Review Skill</h1>

### This is fork from [awesome-skills](https://github.com/awesome-skills) which has been translated into English to make it easier to understand the context.

<p>
  <strong>A comprehensive, modular code review skill for Claude Code</strong>
</p>

<p>
  <a href="https://github.com/awesome-skills/code-review-skill/blob/main/LICENSE">
    <img src="https://img.shields.io/badge/License-MIT-22c55e?style=flat-square" alt="License: MIT"/>
  </a>
  <img src="https://img.shields.io/badge/Claude_Code-Skill-7c3aed?style=flat-square&logo=anthropic&logoColor=white" alt="Claude Code Skill"/>
  <img src="https://img.shields.io/badge/Total_Lines-9%2C500%2B-3b82f6?style=flat-square" alt="9500+ lines"/>
  <img src="https://img.shields.io/badge/Languages-11%2B-f59e0b?style=flat-square" alt="11+ languages"/>
  <img src="https://img.shields.io/badge/PRs-Welcome-ec4899?style=flat-square" alt="PRs Welcome"/>
</p>

<p>
  <a href="#english">English</a>
  &middot;
  <a href="#chinese">中文</a>
  &middot;
  <a href="./CONTRIBUTING.md">Contributing</a>
</p>

</div>

---

<a name="english"></a>

## English

### What is this?

**Code Review Skill** is a production-ready skill for [Claude Code](https://claude.ai/code) that transforms AI-assisted code review from vague suggestions into a **structured, consistent, and expert-level** process.

It covers **11+ languages and frameworks** with over **9,500 lines** of carefully curated review guidelines — loaded progressively to minimize context window usage.

---

### &#10024; Key Features

- **Progressive Disclosure** — Core skill is ~190 lines; language guides (~200–1,000 lines each) load only when needed.
- **Four-Phase Review Process** — Structured workflow from understanding scope to delivering clear feedback.
- **Severity Labeling** — Every finding is categorized: `blocking` · `important` · `nit` · `suggestion` · `learning` · `praise`
- **Security-First** — Dedicated security checklists per language ecosystem.
- **Collaborative Tone** — Questions over commands, suggestions over mandates.
- **Automation Awareness** — Clearly separates what human review should catch vs. what linters handle.

---

### &#127760; Supported Languages & Frameworks

<table>
  <thead>
    <tr>
      <th>Category</th>
      <th>Technology</th>
      <th>Guide</th>
      <th>Lines</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td rowspan="4"><strong>Frontend</strong></td>
      <td>&#9883;&#65039; React 19 / Next.js / TanStack Query v5</td>
      <td><code>reference/react.md</code></td>
      <td>~870</td>
    </tr>
    <tr>
      <td>&#128154; Vue 3.5 + Composition API</td>
      <td><code>reference/vue.md</code></td>
      <td>~920</td>
    </tr>
    <tr>
      <td>&#127912; CSS / Less / Sass</td>
      <td><code>reference/css-less-sass.md</code></td>
      <td>~660</td>
    </tr>
    <tr>
      <td>&#128311; TypeScript</td>
      <td><code>reference/typescript.md</code></td>
      <td>~540</td>
    </tr>
    <tr>
      <td rowspan="4"><strong>Backend</strong></td>
      <td>&#9749; Java 17/21 + Spring Boot 3</td>
      <td><code>reference/java.md</code></td>
      <td>~800</td>
    </tr>
    <tr>
      <td>&#128013; Python</td>
      <td><code>reference/python.md</code></td>
      <td>~1,070</td>
    </tr>
    <tr>
      <td>&#128057; Go</td>
      <td><code>reference/go.md</code></td>
      <td>~990</td>
    </tr>
    <tr>
      <td>&#129408; Rust</td>
      <td><code>reference/rust.md</code></td>
      <td>~840</td>
    </tr>
    <tr>
      <td rowspan="3"><strong>Systems</strong></td>
      <td>&#9881;&#65039; C</td>
      <td><code>reference/c.md</code></td>
      <td>~210</td>
    </tr>
    <tr>
      <td>&#128297; C++</td>
      <td><code>reference/cpp.md</code></td>
      <td>~300</td>
    </tr>
    <tr>
      <td>&#128421;&#65039; Qt Framework</td>
      <td><code>reference/qt.md</code></td>
      <td>~190</td>
    </tr>
    <tr>
      <td rowspan="2"><strong>Architecture</strong></td>
      <td>&#127963;&#65039; Architecture Design Review</td>
      <td><code>reference/architecture-review-guide.md</code></td>
      <td>~470</td>
    </tr>
    <tr>
      <td>&#9889; Performance Review</td>
      <td><code>reference/performance-review-guide.md</code></td>
      <td>~750</td>
    </tr>
  </tbody>
</table>

---

### &#128260; The Four-Phase Review Process

```
Phase 1 - Context Gathering
  Understand PR scope, linked issues, and intent
                    |
                    v
Phase 2 - High-Level Review
  Architecture - Performance impact - Test strategy
                    |
                    v
Phase 3 - Line-by-Line Analysis
  Logic - Security - Maintainability - Edge cases
                    |
                    v
Phase 4 - Summary & Decision
  Structured feedback - Approval status - Action items
```

---

### &#127991;&#65039; Severity Labels

| Label | Meaning |
|-------|---------|
| &#128308; `blocking` | Must be fixed before merge |
| &#128992; `important` | Should be fixed; may block depending on context |
| &#128993; `nit` | Minor style or preference issue |
| &#128309; `suggestion` | Optional improvement worth considering |
| &#128218; `learning` | Educational note for the author |
| &#127775; `praise` | Explicitly highlight great work |

---

### &#128193; Repository Structure

```
code-review-skill/
|
+-- SKILL.md                              # Core skill - loaded on activation (~190 lines)
+-- README.md
+-- LICENSE
+-- CONTRIBUTING.md
|
+-- reference/                            # On-demand language guides
|   +-- react.md                          # React 19 / Next.js / TanStack Query v5
|   +-- vue.md                            # Vue 3.5 Composition API
|   +-- rust.md                           # Rust ownership, async/await, unsafe
|   +-- typescript.md                     # TypeScript strict mode, generics, ESLint
|   +-- java.md                           # Java 17/21 & Spring Boot 3
|   +-- python.md                         # Python async, typing, pytest
|   +-- go.md                             # Go goroutines, channels, context, interfaces
|   +-- c.md                              # C memory safety, UB, error handling
|   +-- cpp.md                            # C++ RAII, move semantics, exception safety
|   +-- qt.md                             # Qt object model, signals/slots, GUI perf
|   +-- css-less-sass.md                  # CSS/Less/Sass variables, responsive design
|   +-- architecture-review-guide.md      # SOLID, anti-patterns, coupling/cohesion
|   +-- performance-review-guide.md       # Core Web Vitals, N+1, memory leaks
|   +-- security-review-guide.md          # Security checklist (all languages)
|   +-- common-bugs-checklist.md          # Language-specific bug patterns
|   +-- code-review-best-practices.md     # Communication & process guidelines
|
+-- assets/
|   +-- review-checklist.md               # Quick reference checklist
|   +-- pr-review-template.md             # PR review comment template
|
+-- scripts/
    +-- pr-analyzer.py                    # PR complexity analyzer
```

---

### &#128640; Installation

**Clone to your Claude Code skills directory:**

```bash
# macOS / Linux
git clone https://github.com/awesome-skills/code-review-skill.git \
  ~/.claude/skills/code-review-skill

# Windows (PowerShell)
git clone https://github.com/awesome-skills/code-review-skill.git `
  "$env:USERPROFILE\.claude\skills\code-review-skill"
```

**Or add to an existing plugin:**

```bash
cp -r code-review-skill ~/.claude/plugins/your-plugin/skills/code-review/
```

---

### &#128161; Usage

Once installed, activate the skill in your Claude Code session:

```
Use code-review-skill to review this PR
```

Or create a custom slash command in `.claude/commands/`:

```markdown
<!-- .claude/commands/review.md -->
Use code-review-skill to perform a thorough review of the changes in this PR.
Focus on: security, performance, and maintainability.
```

**Example prompts:**

| Prompt | What happens |
|--------|-------------|
| `Review this React component` | Loads `react.md` - checks hooks, Server Components, Suspense patterns |
| `Review this Java PR` | Loads `java.md` - checks virtual threads, JPA, Spring Boot 3 patterns |
| `Security review of this Go service` | Loads `go.md` + `security-review-guide.md` |
| `Architecture review` | Loads `architecture-review-guide.md` - SOLID, anti-patterns, coupling |
| `Performance review` | Loads `performance-review-guide.md` - Web Vitals, N+1, complexity |

---

### &#128300; Highlights by Language

<details>
<summary><strong>&#9883;&#65039; React 19</strong></summary>

- `useActionState` - Unified form state management
- `useFormStatus` - Access parent form status without prop drilling
- `useOptimistic` - Optimistic UI updates with automatic rollback
- Server Components & Server Actions patterns (Next.js 15+)
- Suspense boundary design, Error Boundary integration, streaming SSR
- `use()` Hook for consuming Promises

</details>

<details>
<summary><strong>&#9749; Java & Spring Boot 3</strong></summary>

- **Java 17/21**: Records, Pattern Matching for Switch, Text Blocks, Sealed Classes
- **Virtual Threads** (Project Loom): High-throughput I/O patterns
- **Spring Boot 3**: Constructor injection, `@ConfigurationProperties`, `ProblemDetail`
- **JPA Performance**: Solving N+1, correct `equals`/`hashCode` on Entities

</details>

<details>
<summary><strong>&#129408; Rust</strong></summary>

- Ownership patterns and common pitfalls
- `unsafe` code review requirements (mandatory `SAFETY` comments)
- Async/await - avoiding blocking in async context, cancellation safety
- Error handling: `thiserror` for libraries, `anyhow` for applications

</details>

<details>
<summary><strong>&#128057; Go</strong></summary>

- Goroutine lifecycle management and leak prevention
- Channel patterns, select usage
- `context.Context` propagation
- Interface design (accept interfaces, return structs)
- Error wrapping with `%w`

</details>

<details>
<summary><strong>&#9881;&#65039; C / C++</strong></summary>

- **C**: Pointer/buffer safety, undefined behavior, resource cleanup, integer overflow
- **C++**: RAII ownership, Rule of 0/3/5, move semantics, exception safety, `noexcept`
- **Qt**: Object parent/child memory model, thread-safe signal/slot connections, GUI performance

</details>

---

### &#129309; Contributing

Contributions are welcome! See [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines.

**Ideas:**
- New language guides (C#, Swift, Kotlin, Ruby, PHP...)
- Framework-specific guides (Django, Laravel, NestJS...)
- Additional checklists and templates
- Translations of core documentation

---

### &#128196; License

MIT &copy; [awesome-skills](https://github.com/awesome-skills)

---

<div align="center">
  Made with &#10084;&#65039; for developers who care about code quality
</div>
