# Agentic Coding

<div align="center">

# Agentic Coding

**Production-grade documentation systems for AI coding agents**

Design docs, architecture rules, coding standards, security guardrails, testing requirements, and developer handoff manuals - organized so AI agents can build software more safely and consistently.

![Docs](https://img.shields.io/badge/docs-agentic-blue)
![Status](https://img.shields.io/badge/status-active-success)
![Format](https://img.shields.io/badge/format-markdown-informational)
![Focus](https://img.shields.io/badge/focus-production--grade-orange)
![Security](https://img.shields.io/badge/security-first-critical)
![AI](https://img.shields.io/badge/for-AI%20coding%20agents-purple)

</div>

---

> [!IMPORTANT]
> This repository does **not** primarily contain application code.
> It contains **instruction manuals for AI agents** so they can generate real-world, production-ready codebases with stronger consistency, security, and maintainability.

## What this repository is

This repo is a curated collection of **agentic coding documentation bundles**.
Each bundle is a structured set of Markdown files that tells an AI coding agent how to:

- architect a project correctly
- follow stack-specific best practices
- respect security and reliability constraints
- generate tests and quality gates
- avoid common implementation mistakes
- produce developer-friendly handoff documentation

These docs are intended to reduce vague prompting and replace it with **clear engineering standards**.

---

## Why this exists

AI can generate code quickly, but speed without constraints leads to:

- inconsistent architecture
- missing security controls
- poor testing discipline
- fragile deployment setups
- hard-to-maintain projects
- docs that are too generic to be useful

This repository solves that by giving AI agents a **production contract** before they write code.

> [!NOTE]
> Think of these bundles as **internal engineering playbooks for AI**.
> They are written so an agent can consume them and then generate code that better matches senior-level production expectations.

---

## Repository structure

```text
AGENTIC-CODING/
├─ native-expo-docs/
│  ├─ 00-principles.md
│  ├─ 01-expo-architecture.md
│  ├─ 02-react-native-patterns.md
│  ├─ 03-nativewind-design-system.md
│  ├─ 04-supabase-mobile.md
│  ├─ 05-state-management.md
│  ├─ 06-expo-notifications.md
│  └─ 07-performance.md
├─ react-vite-docs/
│  ├─ 00-principles.md
│  ├─ 01-react-patterns.md
│  ├─ 02-supabase.md
│  ├─ 03-state-management.md
│  ├─ 04-tailwind-design-system.md
│  ├─ 05-vite-tooling.md
│  └─ 06-saas-patterns.md
├─ solidity-smart-contract-docs/
│  ├─ 00-principles.md
│  ├─ 01-hardhat-config.md
│  ├─ 02-solidity-patterns.md
│  ├─ 03-testing.md
│  ├─ 04-deployment-scripts.md
│  ├─ 05-security-patterns.md
│  └─ 06-developer-manual.md
├─ .gitignore
├─ LICENSE
└─ README.md
```

---

## Current documentation bundles

| Bundle | Primary stack | Purpose | Audience |
|---|---|---|---|
| `native-expo-docs` | Expo, React Native, Supabase | Production standards for mobile app generation | AI agents building mobile apps |
| `react-vite-docs` | React, Vite, Tailwind, Supabase | Standards for fast, maintainable web app generation | AI agents building modern web apps |
| `solidity-smart-contract-docs` | Solidity, OpenZeppelin, Hardhat | Secure smart contract implementation and deployment guidance | AI agents generating Ethereum contracts |

---

## What each bundle usually contains

Most bundles follow a predictable numbered structure so agents can navigate them reliably.

| File pattern | Purpose |
|---|---|
| `00-*` | Non-negotiable principles, constraints, definition of done |
| `01-*` | Architecture, setup, project structure, boundaries |
| `02-*` | Implementation patterns, coding rules, reusable templates |
| `03-*` | Testing, validation, and quality standards |
| `04-*` | Deployment, environment, scripts, integration guidance |
| `05-*` and above | Security, developer manuals, performance, or stack-specific advanced topics |

This numbering system gives AI agents a consistent reading order:

1. read principles first
2. understand architecture
3. follow patterns
4. satisfy testing and deployment requirements
5. finish with developer-facing documentation

---

## Design philosophy

These docs are intentionally written like **internal production engineering standards**, not casual tutorials.

### Core principles

- **Production-first** - prioritize correctness, maintainability, deployment readiness, and operational safety
- **Security-first** - treat security as a baseline requirement, not a later enhancement
- **Stack-specific** - avoid vague generic advice when concrete patterns are required
- **Enforceable** - use MUST / MUST NOT / SHOULD style rules wherever possible
- **AI-readable** - keep structure predictable so autonomous or semi-autonomous coding systems can follow it
- **Developer-friendly** - include manuals and handoff guidance for humans, not just AI

> [!TIP]
> The best agentic docs do not just describe a stack. They define **how code must be written** in that stack.

---

## How to use this repo

### Option 1: Use as context for an AI coding agent

Provide the relevant folder to your AI system before asking it to generate code.

Example flow:

1. choose the matching documentation bundle
2. instruct the agent to read all files in order
3. ask the agent to follow the bundle as a hard implementation standard
4. require it to explain any deviation from the docs

### Option 2: Use as an internal team standard

Engineering teams can also use these bundles as:

- starter architecture references
- review checklists
- code generation constraints
- onboarding material
- handoff documentation templates

### Option 3: Extend with new stacks

Add new bundles for additional stacks, frameworks, or domains while keeping the same overall style.

---

## Recommended workflow for AI agents

| Step | Agent action | Expected outcome |
|---|---|---|
| 1 | Read `00-*` first | Understand hard constraints and non-negotiables |
| 2 | Read architecture docs | Respect boundaries and project structure |
| 3 | Read implementation patterns | Generate idiomatic, consistent code |
| 4 | Read testing and deployment docs | Produce verifiable, deployable output |
| 5 | Generate code and configs | Include supporting files, not just source code |
| 6 | Generate developer handoff docs | Explain setup, usage, deployment, and next steps |

---

## What “production-grade” means in this repo

In this repository, **production-grade** means the documentation pushes AI agents to produce code that is:

- secure by default
- testable and well-tested
- structured with clear boundaries
- deployable with real environment/config handling
- observable and maintainable
- understandable by the next developer

It does **not** mean:

- maximum complexity
- premature abstraction
- generic enterprise boilerplate without purpose
- fancy patterns with no operational value

---

## Example use cases

| Use case | Relevant bundle |
|---|---|
| Generate an Expo mobile app with auth, state management, and performance rules | `native-expo-docs` |
| Generate a React + Vite SaaS frontend with maintainable structure | `react-vite-docs` |
| Generate Ethereum smart contracts with secure Hardhat workflows and detailed developer instructions | `solidity-smart-contract-docs` |

---

## Contribution guidelines

When adding or editing a documentation bundle in this repo, keep the following standards:

| Rule | Requirement |
|---|---|
| Tone | Write like a senior engineer defining implementation rules |
| Specificity | Prefer concrete patterns over vague advice |
| Security | Include realistic security guidance where relevant |
| Testing | Require detailed testing expectations |
| Deployability | Include configuration and release guidance |
| Handoff | Include docs that help non-expert developers continue the work |
| Consistency | Keep numbering, terminology, and structure coherent |

### New bundle checklist

- [ ] Add a `00-principles.md`
- [ ] Define architecture and boundaries clearly
- [ ] Include stack-specific implementation patterns
- [ ] Include testing expectations
- [ ] Include deployment and config guidance
- [ ] Include security considerations
- [ ] Include a developer manual when the stack benefits from one
- [ ] Keep file names ordered and scannable

---

## Intended audience

This repo is useful for:

- developers using AI to scaffold or implement projects
- teams standardizing AI-assisted development
- founders who want better prompts than ad hoc chat instructions
- agencies building repeatable engineering workflows
- AI coding systems that need stronger implementation constraints

---

## Scope and limitations

> [!CAUTION]
> These documentation bundles improve code generation quality, but they do **not** replace:
>
> - human review
> - security review
> - hardware validation
> - protocol audits
> - integration testing in real environments
> - operational ownership after deployment

For critical systems, especially payments, blockchain, infrastructure, healthcare, or hardware-integrated products, generated code should still go through expert review.

---

## Long-term goal

The goal of this repository is to become a **high-quality library of reusable agentic engineering standards** across multiple stacks.

Over time, each bundle should help AI agents produce outputs that are:

- more secure
- more complete
- more consistent
- easier to review
- easier to ship

---

## Quick start

```bash
# 1) Clone the repository
 git clone <your-repo-url>

# 2) Open the documentation bundle you need
 cd AGENTIC-CODING

# 3) Feed the selected docs to your AI coding workflow
# Example folders:
# - native-expo-docs
# - react-vite-docs
# - solidity-smart-contract-docs
```

---

## Maintainer note

If you use AI to write code, the quality of the result depends heavily on the quality of the instructions.

This repo exists to make those instructions:

- structured
- reusable
- stack-aware
- production-minded
- understandable by both AI agents and human developers

---

<div align="center">

**Agentic Coding = better constraints, better code, better handoff.**

</div>
