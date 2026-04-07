# shane-marketplace

Personal Claude Code plugin marketplace for RTL design and DDD workflow.

## Plugins

### `shane-rtl-pack`
RTL design and review pack — 14 skills + 2 agents.

**Skills**: rtl-code-gen, fsm-design, pipeline-design, datapath-optimization, arithmetic-units, register-design, clock-reset-design, memory-subsystem, rtl-code-review, microarch-doc, debug-methodology, eco-handling, perf-modeling, verilog-sim

**Agents**: rtl-designer, rtl-reviewer

Use for conversational, ad-hoc RTL design and review tasks.

### `shane-ddd-pack`
DDD (Design-Driven Development) workflow — 1 skill + 3 agents.

**Skills**: dd-init

**Agents**: pm (requirement), dd (design), dv (verification)

Use only when the project follows the DDD workflow with a `rtl-ddd/` directory structure.

## Installation

In Claude Code:

```
/plugin marketplace add powershang/shane-marketplace
/plugin install shane-rtl-pack@shane-marketplace
/plugin install shane-ddd-pack@shane-marketplace
```

## Update

```
/plugin marketplace update shane-marketplace
```

## Uninstall

```
/plugin uninstall shane-rtl-pack
/plugin uninstall shane-ddd-pack
/plugin marketplace remove shane-marketplace
```
