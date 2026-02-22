# Formal Verification of a Finite-State Login Authentication Protocol Using Temporal Logic

**Tool:** NuSMV 2.6.x  
**Logic:** Linear Temporal Logic (LTL) + Computation Tree Logic (CTL)  
**Method:** Symbolic Model Checking (BDD-based)  
**Domain:** Formal Methods / Formal Verification  

---

## Abstract

This project formally models and verifies a single-session login authentication protocol using the NuSMV model checker and Linear Temporal Logic. The system is represented as an explicit finite-state transition system with six control states, a bounded failure counter, and a fully nondeterministic environment modeling adversarial input.

Four properties are verified: two safety properties (preventing unauthorized access and enforcing lockout) and two liveness properties (guaranteeing successful authentication and eventual resource access). The initial model (v1.0) violates one liveness property; a counterexample trace is produced and analyzed, revealing a modeling flaw. The model is then refined (v2.0), after which all properties are verified successfully.

All claims are strictly limited to the formal model. No real-world security guarantees are asserted.

---

## Repository Structure


formal-auth-protocol-verification/
├── model/
│ ├── auth_protocol_basic.smv # Minimal core FSM (no properties)
│ ├── auth_protocol.smv # v1.0: initial model (L2 fails)
│ └── auth_protocol_refined.smv # v2.0: refined model (all properties pass)
├── specs/
│ └── properties.smv # LTL/CTL property definitions
├── traces/
│ └── counterexample_liveness2.txt
├── README.md
└── report/
└── final_report.md


---

## System Model Overview

The protocol is modeled as a finite-state transition system:

- **States:** `{Idle, UsernameEntered, PasswordEntered, Authenticated, Locked, AccessGranted}`
- **Initial state:** `Idle`
- **Environment:** Fully nondeterministic (`credentials_valid`)
- **Lockout:** Triggered after three consecutive failures

### State Variables

| Variable | Description |
|---------|-------------|
| `state` | Current FSM location |
| `attempts` | Failed login counter (0..3) |
| `credentials_valid` | Nondeterministic environment input |
| `session_auth` | Auxiliary latch indicating prior authentication |

---

## Transition Summary (Refined Model v2.0)

- Idle → UsernameEntered  
- UsernameEntered → PasswordEntered  
- PasswordEntered → Authenticated (if credentials valid)  
- PasswordEntered → UsernameEntered (retry)  
- PasswordEntered → Locked (after 3 failures)  
- Authenticated → AccessGranted  
- AccessGranted → {Authenticated, Idle}  
- Locked → Locked (absorbing)

---

## Assumptions and Abstractions

1. Single-session only (no concurrency).
2. Credentials abstracted as a boolean.
3. No networking or cryptography modeled.
4. Environment is adversarial via nondeterminism.
5. Lockout is permanent.
6. Failure counter is bounded to 3.
7. Results apply only to this formal model.

---

## Properties Verified

### Safety

**S1 — No Unauthorized Access**


G(state = AccessGranted → session_auth)


**S2 — Lockout Permanence**


G(state = Locked → G(state ≠ AccessGranted))


---

### Liveness

**L1 — Valid Authentication Eventually Succeeds**


G((state = PasswordEntered ∧ credentials_valid) → F(state = Authenticated))


**L2 — Authenticated Users Eventually Access Resources**


G(state = Authenticated → F(state = AccessGranted))


L2 fails in v1.0 and is satisfied in v2.0 after refinement.

---

## Verification Workflow

1. Build finite-state model in NuSMV.
2. Specify LTL safety and liveness properties.
3. Run model checking.
4. Analyze counterexample trace (lasso form).
5. Identify modeling flaw.
6. Refine transition structure.
7. Reverify all properties.
8. Confirm reachability with CTL (`EF`).

---

## Running the Models

```bash
# Basic FSM (no properties)
nusmv model/auth_protocol_basic.smv

# Initial model (expect L2 to fail)
nusmv model/auth_protocol.smv

# Refined model (all properties pass)
nusmv model/auth_protocol_refined.smv
```

## Limitations

- No timing, networking, or cryptographic modeling.
- Credentials reduced to a boolean.
- No account recovery.
- Fully synchronous semantics.
- Results apply only within this abstraction.

## Reference

- Formal Verification of a Finite-State Login Authentication Protocol Using Temporal Logic
- NuSMV 2.6.x — BDD-based LTL Model Checking