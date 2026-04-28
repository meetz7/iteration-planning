# iteration-planning

[English](README.md) | [简体中文](README.zh-CN.md)

A Claude Code skill that starts from clear direction, refines requirements, analyzes impact, organizes checklist, executes item by item, and outputs complete documentation.

**Core Purpose: Prevent AI deviation during code execution**

---

## What Problem Does This Solve

| Problem | How This Skill Solves It |
|---------|--------------------------|
| Clear direction but fuzzy details | AI actively retrieves code, understands structure, helps you break down into specific change items |
| Many trivial changes easy to miss | Analyzes impact scope (upstream callers, downstream dependencies), ensures nothing is missed |
| Uncertain execution order | Sorts by dependency, marks which can be executed in parallel |
| Multi-module changes confusion | Automatically groups by module, each module has independent documentation |
| AI implementation prone to deviation | Outputs JSON documents (before/after code comparison + specific steps), prevents deviation |
| Execution process untraceable | Outputs execution log, each item status can be traced back |

---

## When to Use

| Suitable | Not Suitable |
|----------|--------------|
| Clear direction but details need refining (e.g. "payment interface change xxx") | Completely fuzzy requirements → use brainstorming |
| Trivial but numerous changes (10+ small changes) | Single simple change → handle directly in conversation |
| Involves multiple modules or business lines | |
| Need to analyze call chain, boundary conditions | |
| Need to output documentation for AI execution | |

---

## Process

```
Step 1: Collect direction + retrieve code
        User says change direction → AI retrieves code → Read to understand structure

Step 2: Analyze impact scope
        Call chain, upstream callers, downstream dependencies, boundary conditions, risk points

Step 3: Break down specific change items
        Direction + impact analysis → specific change points (file:function:line)

Step 4: Organize checklist + sort
        Group by module + dependency order + mark parallel execution

Step 5: Confirm checklist
        User reviews, supplements, adjusts

Step 6: Execute item by item
        Execute after confirmation, report after completion

Step 7: Output documentation
        4 types of documents (JSON + Markdown), for AI execution
```

---

## Output Documents

After execution, generated in `docs/iteration-{date}/`:

| File | Format | Purpose | Why This Format |
|------|--------|---------|-----------------|
| `CHANGELOG.json` | JSON | AI locates change items | JSON missing field will error, won't miss |
| `IMPLEMENTATION_GUIDE.json` | JSON | AI execution core basis | Fields directly store path/line, parsing won't error |
| `IMPACT_ANALYSIS.md` | Markdown | Code structure explanation | Code blocks display, human and AI can both read |
| `EXECUTION_LOG.json` | JSON | Execution status trace | Fields precise, won't miss status |

---

### CHANGELOG.json Structure

Change checklist, AI locates all change items:

```json
{
  "changelog": {
    "project": "payment-service",
    "date": "2026-04-28",
    "trigger_direction": "add logging to scan interface",
    "items": [
      {
        "id": 1,
        "description": "Add log recording parameters",
        "type": "add",
        "location": {
          "file": "scan_controller.ts",
          "function": "handleScan",
          "line": 20
        },
        "status": "completed",
        "depends_on": null
      }
    ],
    "execution_order": [
      { "id": 1, "reason": "no dependency, execute first" },
      { "id": 2, "reason": "depends on logId from #1" }
    ]
  }
}
```

---

### IMPLEMENTATION_GUIDE.json Structure

**This is the core document**, AI execution's precise basis:

```json
{
  "implementation_guide": {
    "items": [
      {
        "id": 1,
        "description": "Add log recording parameters",
        "location": {
          "file": "scan_controller.ts",
          "function": "handleScan",
          "line": 20
        },
        "change_type": "add",
        "before_code": "function handleScan(req) {\n    validateRequest(req);\n    process(req);\n    return result;\n}",
        "after_code": "function handleScan(req) {\n    validateRequest(req);\n    logger.info({ logId: req.logId }, \"scan request received\");\n    process(req);\n    return result;\n}",
        "diff_context": "5 lines before and after change location...",
        "steps": [
          "Insert a line after validateRequest(req)",
          "Add logger.info({ logId: req.logId }, \"scan request received\")",
          "Confirm no impact on original logic"
        ],
        "verify_methods": [
          "Run project tests, confirm tests pass",
          "Manually call interface, confirm log output correct"
        ],
        "risks": ["Log format needs to be compatible with monitoring system"],
        "rollback_steps": ["Delete the added logger.info line"]
      }
    ]
  }
}
```

**Field Description:**

| Field | Required | Description |
|-------|----------|-------------|
| `location.file` | ✓ | File path |
| `location.function` | ✓ | Function name |
| `location.line` | ✓ | Line number (precise insertion position) |
| `before_code` | ✓ | Complete code before change (at least 5 lines context) |
| `after_code` | ✓ | Complete code after change (at least 5 lines context) |
| `steps` | ✓ | Specific operation steps array (each step must be clear) |
| `verify_methods` | ✓ | Verification methods array |
| `risks` | ○ | Potential risks (optional) |
| `rollback_steps` | ○ | Rollback steps (optional) |

---

### Multi-module Output Structure

When multiple modules involved, documents separated by module:

```
docs/iteration-2026-04-28/
├── CHANGELOG.json          # Overview checklist (all module change items)
├── payment-module/
│   ├── IMPLEMENTATION_GUIDE.json
│   ├── IMPACT_ANALYSIS.md
│   └── EXECUTION_LOG.json
├── order-module/
│   ├── IMPLEMENTATION_GUIDE.json
│   ├── IMPACT_ANALYSIS.md
│   └── EXECUTION_LOG.json
└── user-module/
    ├── IMPLEMENTATION_GUIDE.json
    ├── IMPACT_ANALYSIS.md
    └── EXECUTION_LOG.json
```

---

## Installation

```bash
# Clone repository
git clone https://github.com/meetz7/iteration-planning.git

# Create symbolic link to Claude skills directory
ln -s $(pwd)/iteration-planning ~/.claude/skills/iteration-planning
```

---

## Usage

### Trigger Method

- Direct invocation: `/iteration-planning`
- Auto trigger: Say any of the following
  - "I have a bunch of changes to make"
  - "Some module/interface change xxx"
  - "Help me organize change checklist"
  - "Help me sort out these changes"

---

### Usage Example

**Scenario: Add logging to payment interface, order module sync status**

```
User: Add logging to payment interface to record request params, order module sync status

AI Execution:
┌─────────────────────────────────────────────────────────┐
│ Step 1: Retrieve code                                    │
│   - Found payment interface: src/pay/scan_controller.ts │
│   - Found order module: src/order/order_service.ts      │
│   - Read to understand code structure, call chain       │
├─────────────────────────────────────────────────────────┤
│ Step 2: Analyze impact                                   │
│   - Payment interface: 2 upstream callers, downstream payment_gateway │
│   - Order module: 3 upstream callers, depends on payment interface return │
│   - Boundary conditions: concurrent scenarios, exception handling │
├─────────────────────────────────────────────────────────┤
│ Step 3: Break down change items                          │
│   - Payment module: #1 add logging, #2 add logId param, #3 test │
│   - Order module: #4 status sync, #5 caller adapt, #6 test │
├─────────────────────────────────────────────────────────┤
│ Step 4: Organize checklist                               │
│   - Payment module (no dependency) → can execute immediately │
│   - Order module (depends on payment module completion) → wait │
│   - Note: Payment module can parallel with other no-dependency modules │
├─────────────────────────────────────────────────────────┤
│ Step 5: Confirm checklist                                │
│   - Display checklist, user review                       │
│   - User supplement: "also need a log switch"            │
│   - Update checklist                                     │
├─────────────────────────────────────────────────────────┤
│ Step 6: Execute item by item                             │
│   - About to execute #1: Add log recording parameters    │
│   - Confirm execution? → User confirm → Execute → Done ✓│
│   - Next item #2...                                      │
├─────────────────────────────────────────────────────────┤
│ Step 7: Output documentation                             │
│   - docs/iteration-2026-04-28/                           │
│     ├── CHANGELOG.json                                   │
│     ├── payment-module/IMPLEMENTATION_GUIDE.json         │
│     ├── order-module/IMPLEMENTATION_GUIDE.json           │
│     ...                                                  │
└─────────────────────────────────────────────────────────┘
```

---

### How to Use Output Documents to Execute Code

**Scenario: You give documents to another AI (or another session) to execute code implementation**

```
Another AI Execution Flow:

1. Read CHANGELOG.json
   - Parse items[] → get all change items
   - Parse execution_order[] → get execution order

2. Read IMPLEMENTATION_GUIDE.json
   - For each item:
     a. Parse location → locate file, function, line
     b. Compare before_code with current file code → confirm match
     c. Execute each step in steps[] → don't skip, don't miss
     d. Verify by verify_methods[] → confirm correct

3. Update EXECUTION_LOG.json
   - Record each item execution result

4. If encounter problem
   - Stop execution, fill error_message
   - If need rollback, execute rollback_steps[]
```

**Why JSON Won't Error:**

| Risk | Markdown Might Error | JSON Won't Error |
|------|---------------------|------------------|
| Missing field | Table parsing might miss "steps" or "verify_methods" | JSON missing field will error |
| Position imprecise | `location: scan_controller.ts:handleScan():20` → AI might parse incorrectly | `{ "file": "...", "function": "...", "line": 20 }` → precise |
| Execution order confusion | List parsing might skip items or wrong order | `{ "execution_order": [1, 2, 3] }` → array order fixed |

---

## Notes

| Note | Description |
|------|-------------|
| Don't add unrequested changes | Only do items in checklist, don't "optimize顺便" |
| Understand code before breaking items | Don't guess, Read to understand first |
| Impact analysis must be comprehensive | Upstream, downstream, boundary, risks all need analysis |
| Documentation must be detailed enough | before_code and after_code need complete context (at least 5 lines) |
| Report immediately when blocked | Don't skip or bypass on your own |

---

## Target Audience

- Developers using Claude Code
- Teams with iteration change needs
- Scenarios needing AI to execute code but worried about deviation
- Complex changes across multiple modules, multiple business lines

---

## License

MIT