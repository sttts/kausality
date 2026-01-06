# Kausality

> **⚠️ EXPERIMENTAL: This is a design experiment. Nothing is implemented yet. The ideas here may be flawed, incomplete, or entirely impractical.**

**Every mutation needs a cause.**

Kausality traces and gates spec changes through Kubernetes object hierarchies. Controllers cannot mutate downstream resources unless explicitly allowed.

## Goals

This is about **safety**, not security.

- Traceability of destructive actions
- Avoidance of accidental damage where possible
- Stays out of your way during normal operations

The system assumes good intent. It protects against accidents, not malicious actors.

## How It Works

1. Human or CI changes a resource (e.g., `Deployment.spec.replicas`)
2. Admission webhook injects **allowances** into the resource's annotations
3. When a controller mutates a child resource, admission checks for matching allowances
4. The **trace** propagates through the hierarchy, recording the causal chain

Without an allowance, controllers cannot mutate downstream resources.

## Documentation

See [doc/DESIGN.md](doc/DESIGN.md) for the full design.

