[Back](./README.md)

## 📑 Table of Contents

| Section | Topic |
|---------|-------|
| [1](#what-is-copilot) | What is Copilot? |
| [2](#common-use-cases) | Common Use Cases |
| [3](#model-selection-matters) | Model Selection Matters |
| [4](#writing-effective-prompts) | Writing Effective Prompts |
| [5](#staying-a-real-engineer-in-the-age-of-ai) | Staying a "Real Engineer" |
| [6](#a-question-for-you) | A Question for You |

---

# What is Copilot?

Copilot is an AI-powered coding assistant that helps you `write`, `edit`, `refactor`, and understand code faster. 

But like any powerful tool, its value depends entirely on how you use it.

---

## Common Use Cases

Copilot shines in:

* Writing, fixing, and refactoring code
* Exploring unfamiliar codebases

  * *How is authentication implemented?*
  * *What libraries are used?*

---

## Model Selection Matters

* Use **Claude Opus** for complex, multi-step, or architectural tasks
* Use **Claude Sonnet** for quick edits, small features, or simple questions

Think of it as choosing between a deep thinker and a fast executor, and cost effective.

---

## Writing Effective Prompts

### Always Provide Context

* Relevant conversations
* Files and methods
* Project structure
* Copilot instructions

Use tools like:

* `#file` to pin specific files
* `@workspace` to give full project context (only when needed)

Copilot performs best when it understands your environment. Give it:

---

### Be Clear, Concise, and Specific

* ❌ *“Add authentication”*
* ✅ *“Add JWT-based authentication to the login endpoint using the existing user model”*
* ❌ *“Hi Copilot, can you please kindly help me with…”*
* ✅ *“Refactor this function to reduce duplication”*

The more precise you are, the less guesswork the AI has to do.

---


### Plan Before You Prompt

1. Break your task into smaller subtasks
2. Ask Copilot for a step-by-step plan
3. Review the plan
4. Then say: *“Now implement step 1”*

Don’t jump straight into implementation.

---
### Add Constraints

* *“Do NOT change the function signature”*
* *“Do NOT add new dependencies”*

Constraints guide the solution and prevent unwanted surprises.

---

### Use the Right Chat Mode

* **Ask** → for questions and explanations
* **Agent** → for autonomous, multi-file tasks

Different modes serve different purposes

---



### Show Examples

* Paste an example
* Say: *“Write something similar that does X”*

This dramatically improves alignment with your codebase.

---

### Make Copilot Ask You Questions

At the end of the propmt, just append:
*“Before answering, ask me any clarifying questions you need.”*

This turns the interaction into a collaboration, not a one-shot guess.

---

### Define Your Project Rules

Create a `copilot-instructions.md` file that includes:

* Your tech stack
* Coding standards
* Architecture patterns

This gives Copilot a consistent “mental model” of your project.

---

### Use It to Learn, Not to Deliver

Don’t just accept outputs => understand them.

Ask:

* *Why was this approach used?*
* *What are the trade-offs?*

This is how you grow, not just ship.


---

## Staying a “Real Engineer” in the Age of AI

To stay a *Real Engineer* in 2026, you need a personal framework:

---

### ✅ The Debug Test

- If you can’t debug the code without the AI => Don’t ship it.

- If you don’t understand how it fails => Don’t ship it.

---

### ✅ The Logic Test

- If you can’t explain the logic from scratch => Don’t ship it.

> “The AI suggested it” is not a technical justification.

---

## A Question for You

How do you make sure your team is still *thinking*—and not just *prompting*?

---

[Back](./README.md)