# Two-Phase Data-Preserving Brick Replacement Design

**Document Version:** 1.0  
**Date:** February 2026  
**Status:** Design Review  
**Authors:** Rafi KC

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Problem Statement](#problem-statement)
3. [Solution Overview](#solution-overview)
4. [Architecture](#architecture)
5. [Component Design](#component-design)
6. [Implementation Details](#implementation-details)
7. [State Machine & Workflows](#state-machine--workflows)
8. [Error Handling & Recovery](#error-handling--recovery)
9. [Simultaneous Replica Replacement](#simultaneous-replica-replacement)
10. [Translator Stack Comparison](#translator-stack-comparison)
11. [Conclusion](#conclusion)

## Two-Phase Brick Replacement Mechanism for GlusterFS

### Summary

This document describes a **two-phase brick replacement mechanism** for GlusterFS that enables **zero-redundancy-loss** hardware migration and node replacement scenarios.

Unlike the current `replace-brick` command, which immediately removes the old brick and temporarily risks replica protection and quorum safety, this design introduces a **transactional replacement workflow** that preserves redundancy, correctness, and stability throughout the lifecycle.

---

### Design Goals

- **Zero replica redundancy loss**
- **No AFR quorum disruption**
- **No heal/index noise during migration**
- **Crash-safe state persistence**
- **Support simultaneous multi-brick replacement**

---

### Key Innovation

The design introduces a new **`replace-brick` xlator**, positioned between AFR and the brick being replaced.

This xlator:

- Transparently routes **xattrop operations**
- Tracks **healed GFIDs**
- Maintains replacement delta state
- Avoids invasive AFR core modifications
- Enables **simultaneous brick replacement** across a replica set

---

### Operational Model

The replacement workflow is divided into two explicit phases.

---

### Phase 1 – Prepare / Migration

The replacement brick is added as a **candidate**, but remains logically hidden from AFR quorum and heal decisions.

#### Behavior

- Replacement brick connected but not part of AFR voting children
- Real brick continues serving reads/writes
- Replacement brick receives **self-accusing xattrops only**
- Background migration performs brick-to-brick copy

#### Validation Rules

Commit is rejected if:

- Pending heals exist where the old brick is the authoritative source

---

### High-Level Flow – Phase 1

**Before Prepare**

In the normal layout, AFR directly connects to the client xlator stack of each brick.

![Phase 1 - Before Prepare](docs/images/phase1-before.png)

**After Prepare (Phase 1) – replace-brick xlator inserted**

![Phase 1 - After Prepare](docs/images/phase1-after.png)

Notes: The replace-brick xlator becomes a parent of the existing client stack for the brick being replaced; it has two children in Phase1: the real brick and a dummy replacement that receives self-accusing xattrops.

---

### Write Path Behavior

For mutating FOPs:

- Real Brick: Normal FOP execution
- Replacement Brick: Self-accusing xattrop entries

Guarantees delta tracking without duplicate writes.

---

### SHD / Heal Interaction

Self-heal operations affecting the old brick also generate xattrop markers on the replacement brick so the replacement accumulates healed GFIDs for validation.

---

### Phase 2 – Commit

After migration completes and validations pass, commit performs atomic topology switch.

#### Critical Safety Rule

Verify **no pending heals** where old brick is source.

---

### High-Level Flow – Phase 2

**After Commit (Phase 2) – Old brick disconnected, replacement brick becomes active AFR child**

![Phase 2 - Committed](docs/images/phase2-committed.png)

---

### Failure Handling

- Replacement brick failure → Non-fatal in Phase 1 (indices persisted locally)
- Migration failure → Abort / Retry
- Glusterd restart → Recover persisted state and resume migration
- Replacement recovery → Replay indices to replacement brick

---

### Simultaneous Replica Replacement

Because the logic is encapsulated in the `replace-brick` xlator and it uses per-brick indices, multiple bricks in the same replica set can be replaced concurrently without AFR core changes.

---

### Advantages of Xlator Approach

✓ No AFR core changes  
✓ Clean isolation  
✓ Deterministic recovery  
✓ Minimal client impact  
✓ Multi-brick replacement supported

---

### Translator Stack Comparison

Normal stack (before replace):

```text
AFR
 └─ Client-Xlator
    └─ Brick (old)
```

Replacement stack (Phase 1):

```text
AFR
 └─ replace-brick
    ├─ Client-Xlator → Brick (old)
    └─ Dummy-Xlator  → Dummy (replacement)
```

After commit (Phase 2):

```text
AFR
 └─ Client-Xlator
    └─ Brick (new)
```

---

### Conclusion

Brick replacement becomes a transactional, redundancy-preserving operation suitable for production migrations using this two-phase approach and the `replace-brick` xlator.
