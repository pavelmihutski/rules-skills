---
name: planner
description: Create detailed atomic task implementation plans that respect TDD, minimal changes policy, and Plan/Act mode workflow. Use when starting any new feature, refactoring, or breaking down complex tasks into smallest possible executable steps.
metadata:
  author: Pavel Mihutski
  version: "1.0"
  tags: planning, atomic-tasks, tdd, minimal-changes
---

# Atomic Planning Skill

## Purpose

Produce a **clear, actionable implementation plan** broken down into the **smallest possible atomic tasks**. Each task should be independently completable, follow TDD principles (RED → GREEN → REFACTOR), and respect the minimal changes policy.

## When to Use This Skill

Use this skill when the user asks to:

- "Create a plan" / "Plan the implementation" / "Design an approach"
- Add a feature, refactor code, or implement a bugfix
- Break down work into atomic tasks with testing
- Ensure safe, incremental development with user approval at each step

## Planning Approach

### 1. Understand the Request

**Clarify before planning**:

- What problem does this solve?
- What is the specific requirement (no assumptions)?
- What are the success criteria?
- Are there constraints (performance, compatibility)?
- What testing framework is used?

**If unclear, ask**:

- "What specific behavior should change?"
- "Which files should be modified?"
- "What should the test verify?"
- "Are there existing patterns to follow?"

**NEVER assume**:

- Additional features the user might want
- Future requirements
- Architectural changes not requested
- Extra "helpful" improvements

### 2. Analyze the Codebase

**Before creating a plan, search and understand**:

- Find relevant files (services, functions, components)
- Identify existing patterns to follow
- Locate test files to understand testing approach
- Check for similar implementations

**Consider**:

- Where does this fit in the current structure?
- What dependencies exist?
- Are there existing abstractions to use?
- What is the current test coverage?

### 3. Break Down Into Atomic Tasks

**Atomic Task Criteria**:

- Each task must be independently completable
- Should take minimal time to execute (minutes, not hours)
- Must have clear, measurable outcome
- Cannot be broken down further meaningfully
- Modifies only ONE file (unless docs/splits)

**Task Size Examples**:

- ✅ "Write test for getUserById with valid ID"
- ✅ "Implement getUserById method"
- ✅ "Add email validation to input"
- ✅ "Rename variable 'data' to 'userData'"
- ✅ "Add error handling for null case"
- ✅ "Update import statement"
- ❌ "Implement user authentication" (too large)
- ❌ "Add all tests for user service" (too large)

**NO TASK TOO SMALL**: Single line changes are valid tasks. When in doubt, break it down further.

### 4. Apply TDD to Each Task

**Every atomic task follows TDD cycle**:

1. **🔴 RED**: Write failing test — test one specific behavior, verify failure, **STOP** and wait for approval
2. **🟢 GREEN**: Implement minimal code — make test pass with simplest code, **STOP** and wait for approval
3. **🔵 REFACTOR**: Improve code quality — clean up without changing behavior, **STOP** and wait for approval
4. **Next Task**: Move to next atomic task after user approval

### 5. Consider Risks and Safety

- What could go wrong?
- What's the rollback plan?
- Are there breaking changes?
- What needs verification?

## Core Principles

1. **ATOMIC TASKS ALWAYS**: Break down to smallest possible tasks
2. **TDD ALWAYS**: Tests before code (RED → GREEN → REFACTOR)
3. **ONE TASK AT A TIME**: Complete current before starting next
4. **MANDATORY STOPS**: Wait for user approval after each phase
5. **MINIMAL CHANGES**: Only what user requested, no assumptions
6. **ONE FILE PER TASK**: Unless updating docs or splitting files
7. **ASK WHEN UNCLEAR**: Never hallucinate requirements

## Planning Output Format

### Plan Structure

Create plans following this structure:

````markdown
# Implementation Plan: [Feature Name]

## Overview

[2-3 sentences: what's being built and why]

## User Requirements

- **What**: [Specific requirement from user]
- **Where**: [Which file(s) to modify]
- **Success Criteria**: [How to verify it works]

## Codebase Analysis

**Relevant Files**:

- `path/to/file1.ext` - [purpose]
- `path/to/file2.ext` - [purpose]

**Existing Patterns**:

- [Pattern to follow from codebase]

**Testing Framework**: [e.g., pytest, jest, vitest]
**Test Command**: [e.g., `pytest tests/ -v`, `npm test`]

## API Design

**Functions/Components/Hooks/Types** (as relevant):

```typescript
// Type signatures with JSDoc comments
```

## Atomic Task Breakdown

### Task 1: [Atomic Task Name]

**Estimated Time**: [X] minutes

#### 🔴 RED: Write Failing Test

**File**: `path/to/test/file.spec.ext`
**Action**: Write test for [specific behavior]

- [ ] Create/modify test file
- [ ] Write test case for [specific behavior]
- [ ] Run tests to confirm failure
- [ ] **STOP** - Wait for user approval

```language
test('should [behavior]', () => {
  // Arrange / Act / Assert
});
```

**Run**: `[test command]`
**Expected**: Test FAILS ❌

---

#### 🟢 GREEN: Implement Minimal Code

**File**: `path/to/implementation/file.ext`
**Action**: Write minimal code to pass test

- [ ] Implement [specific method/function/change]
- [ ] Run tests to confirm pass
- [ ] **STOP** - Wait for user approval

**Run**: `[test command]`
**Expected**: Test PASSES ✅

---

#### 🔵 REFACTOR: Improve Code Quality

**File**: `path/to/implementation/file.ext`
**Action**: Refactor if needed

- [ ] [Specific refactoring action, or "None needed"]
- [ ] Run tests to verify no breakage
- [ ] **STOP** - Wait for user approval

**Run**: `[test command]`
**Expected**: Tests still PASS ✅

**Commit**: `git commit -m "[Brief description of atomic change]"`

---

### Task 2: [Next Atomic Task Name]

[Repeat same RED → GREEN → REFACTOR structure]

---

## Total Estimated Time

**Total Tasks**: [N] atomic tasks
**Total Time**: [X] minutes to [Y] hours

## Execution Workflow (ACT Mode)

1. User types "ACT" to enter ACT mode
2. Execute Task 1 - RED phase → Agent stops and waits
3. User types "CONTINUE" → Execute Task 1 - GREEN phase → Agent stops and waits
4. User types "CONTINUE" → Execute Task 1 - REFACTOR phase → Agent stops and waits
5. User types "CONTINUE" → Move to Task 2 - RED phase
6. Repeat until all tasks complete

**Valid Approval Commands**: "CONTINUE", "NEXT", "OK", "PROCEED", "yes"

## Pre-Execution Checklist

- [ ] Plan is complete and approved by user
- [ ] All atomic tasks identified
- [ ] Each task has clear file path
- [ ] Test commands are specified
- [ ] User has typed "ACT" to enter ACT mode

## Risks and Rollback

**Potential Risks**: [List specific risks]
**Rollback Plan**: [How to undo changes]
**Verification**: [How to verify it works correctly]
````

## Write Plan to File

**ALWAYS write the plan to a file**:

1. Create `.plans/` directory at repository root if it doesn't exist
2. Write plan to `.plans/<TICKET>:<short-lowercase-hyphen-name>.md`
3. Ask for a ticket/issue number if not provided
4. Use descriptive name: `.plans/JIRA-000:add-user-avatar-upload.md`
5. File should contain the full plan (no YAML frontmatter needed)

## Plan Quality Checklist

**Atomicity**:
- [ ] Each task is smallest possible unit of work
- [ ] Each task modifies only ONE file (unless docs/splits)

**Clarity**:
- [ ] Every task has a clear file path
- [ ] Test cases are specific and concrete
- [ ] Implementation approach is explained with code examples

**TDD Compliance**:
- [ ] Every task follows RED → GREEN → REFACTOR
- [ ] Tests written before implementation
- [ ] Mandatory stops after each phase marked clearly

**Minimal Changes**:
- [ ] Only user-requested changes included
- [ ] No assumptions about extra features
- [ ] Follows existing codebase patterns

**Safety**:
- [ ] Rollback plan is clear
- [ ] Risk areas identified
- [ ] Each commit is small and production-safe

## Anti-Patterns to Avoid

❌ **Large Tasks**: "Implement entire authentication system"
✅ **Atomic Tasks**: "Write test for password hashing", "Implement hash function"

❌ **Multiple Files**: "Update service and controller together"
✅ **Single File**: "Update service", then "Update controller"

❌ **Assumptions**: "Add logging and monitoring (user didn't ask)"
✅ **Requested Only**: "Add login function (as requested)"

❌ **Skip Tests**: "Implement feature then test"
✅ **Test First**: "Write failing test, then implement"

❌ **Auto-Continue**: Agent proceeds without approval
✅ **Mandatory Stops**: Agent stops and waits after each phase
