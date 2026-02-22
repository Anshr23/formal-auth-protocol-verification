# Formal Verification of a Finite-State Login Authentication Protocol Using Temporal Logic

**Technical Report**  
Defense Research Internship — Formal Methods Stream  

---

## Abstract

We present a formal model and mechanised verification of a single-session login authentication protocol. The protocol is encoded as a finite-state transition system in NuSMV and analysed against four properties expressed in Linear Temporal Logic: two safety properties and two liveness properties. The environment — representing the adversarial input source — is modelled fully nondeterministically. Account lockout after three consecutive failed authentication attempts is encoded as an absorbing state. The initial model is shown to violate one liveness property; a lasso-shaped counterexample trace is produced and analysed. The failure is classified as a modelling flaw arising from over-liberal nondeterminism in the post-authentication transition. A refined model is produced in which the flaw is corrected, and all four properties are verified to hold. All conclusions are stated strictly within the formal model; no real-world security guarantee is claimed.

---

## 1. Introduction

The formal verification of security protocols is a well-established discipline within Formal Methods. Unlike empirical testing, which explores a finite sample of execution paths, model checking exhaustively explores the full state space of a finite-state system and either confirms that a property holds on every reachable trace, or produces a concrete counterexample witnessing the violation. This exhaustiveness makes model checking particularly suited to security applications, where a single violating execution can constitute a critical vulnerability.

In this work, we apply model checking to a finite abstraction of a login authentication protocol. The protocol is deliberately simple: it models the control flow of a single authentication session, including username entry, password entry, success, failure, lockout, and resource access. Complexity is deliberately confined to the adversarial environment model, which is given maximal nondeterminism. The goal is to demonstrate, within the formal model, that the protocol satisfies a set of precisely stated safety and liveness requirements.

The contribution of this report is threefold. First, we provide a complete formal model of the protocol encoded in NuSMV, together with full LTL property specifications. Second, we report a genuine property failure in the initial model, accompanied by a full counterexample trace and a step-by-step analysis classifying the failure source. Third, we present a refined model against which all properties are verified, and we discuss the abstraction choices and their limitations.

This work is a formal methods exercise; it does not constitute an implementation, security audit, or empirical study of any deployed authentication system.

---

## 2. Related Work

The application of temporal logic to protocol verification originates with Clarke, Emerson, and Sistla's foundational work on CTL model checking [Clarke et al., 1986] and with Pnueli's introduction of temporal logic for program reasoning [Pnueli, 1977]. The Murφ and SPIN model checkers were subsequently applied to network protocol verification [Holzmann, 1997], and NuSMV [Cimatti et al., 2002] extended BDD-based symbolic model checking to complex reactive systems.

In the security domain, Lowe's attack on the Needham-Schroeder protocol [Lowe, 1996] — discovered through model checking in FDR — is perhaps the canonical demonstration of the power of formal verification for finding subtle security flaws. More directly related to the present work, authentication protocols have been modelled as finite-state systems and verified using process algebras (CSP, CCS), logic programming (ProVerif), and bounded model checking.

The present work is closest in spirit to educational and research demonstrations of NuSMV-based protocol verification (e.g., Kwiatkowska et al. on probabilistic model checking of security protocols, and Gnesi et al. on NuSMV-based verification of authentication in mobile systems). Our model is intentionally minimal: credentials are abstracted as a boolean, and the adversary is encoded directly as nondeterminism in a synchronous reactive module, rather than as an explicit Dolev-Yao attacker model. This abstraction level is appropriate for the stated objective of formally verifying protocol control flow.

---

## 3. System Model

### 3.1 Finite-State Transition System

A finite-state transition system is a tuple M = (S, I, R, AP, L) where:

- **S** is a finite, non-empty set of states.
- **I ⊆ S** is the set of initial states.
- **R ⊆ S × S** is the transition relation, which we assume to be total (every state has at least one successor).
- **AP** is the set of atomic propositions.
- **L : S → 2^AP** is the labelling function assigning to each state the set of atomic propositions that hold there.

A **trace** of M is an infinite sequence of states π = s₀, s₁, s₂, … such that s₀ ∈ I and (sᵢ, sᵢ₊₁) ∈ R for all i ≥ 0. The set of all traces of M is denoted Traces(M).

For NuSMV's synchronous semantics, state transitions are instantaneous and globally consistent: all variable assignments at step i+1 are computed simultaneously from the assignments at step i.

### 3.2 States

The protocol model has six control states:

| State | Description |
|---|---|
| `Idle` | No active authentication session; system awaits input |
| `UsernameEntered` | Username has been submitted; awaiting password |
| `PasswordEntered` | Password has been submitted; awaiting credential check |
| `Authenticated` | Credential check succeeded; session is active |
| `Locked` | Account locked after N = 3 consecutive failures (absorbing) |
| `AccessGranted` | Authenticated user is accessing a protected resource |

The initial state is `Idle`.

### 3.3 Variables and Domains

**state ∈ {Idle, UsernameEntered, PasswordEntered, Authenticated, Locked, AccessGranted}**  
The primary control variable encoding the FSM location.

**attempts ∈ {0, 1, 2, 3}**  
A bounded integer counting consecutive failed password entries in the current session. Bounded to [0..3] since the fourth increment triggers lockout and no further increments occur (the system is in `Locked`, which is absorbing).

**credentials_valid ∈ {TRUE, FALSE}**  
A fully nondeterministic boolean representing the adversarial environment's credential supply. No `init` or `next` assignment is provided; NuSMV treats this variable as ranging over all values of its type at every step. This is the maximal adversarial model: at each transition step, the environment may choose any credential value.

**session_auth ∈ {TRUE, FALSE}**  
An auxiliary monotonic latch variable. It is initialised to FALSE and set to TRUE precisely when transition T3 fires (state = `PasswordEntered` ∧ credentials_valid = TRUE). It is never reset to FALSE within the model (monotonic). This variable is introduced to enable a non-trivial encoding of Safety Property S1. It is not part of the protocol's operational semantics; it is a verification instrumentation variable.

### 3.4 Transition Relation

The transition relation R is specified by a guarded case expression in NuSMV's ASSIGN language. All guards are mutually exclusive and collectively exhaustive (the final TRUE guard acts as a default). The NuSMV tool verifies the absence of deadlock states by requiring the transition relation to be total.

**T1: Idle → UsernameEntered**  
Guard: `state = Idle`  
Action: `state' = UsernameEntered`  
Interpretation: The user initiates a login session by providing a username. This transition is deterministic; within the formal model, user intent to log in is not treated as an adversarial or nondeterministic input.

**T2: UsernameEntered → PasswordEntered**  
Guard: `state = UsernameEntered`  
Action: `state' = PasswordEntered`  
Interpretation: The user enters a password. Deterministic for the same reason as T1.

**T3: PasswordEntered → Authenticated (correct credentials)**  
Guard: `state = PasswordEntered ∧ credentials_valid = TRUE`  
Action: `state' = Authenticated; attempts' = 0`  
Interpretation: The environment supplies valid credentials; the authentication check succeeds. The failure counter is reset to zero.

**T4: PasswordEntered → UsernameEntered (wrong credentials, below lockout)**  
Guard: `state = PasswordEntered ∧ credentials_valid = FALSE ∧ attempts < 2`  
Action: `state' = UsernameEntered; attempts' = attempts + 1`  
Interpretation: The environment supplies invalid credentials; the user is returned to the username entry state to retry. The failure counter is incremented.

**T5: PasswordEntered → Locked (wrong credentials, at lockout threshold)**  
Guard: `state = PasswordEntered ∧ credentials_valid = FALSE ∧ attempts ≥ 2`  
Action: `state' = Locked; attempts' = 3`  
Interpretation: The third consecutive failure triggers account lockout. The `Locked` state is absorbing.

**T6 (v2.0): Authenticated → AccessGranted**  
Guard: `state = Authenticated`  
Action: `state' = AccessGranted`  
Interpretation: An authenticated user must access at least one protected resource before logging out. This transition is deterministic in the refined model (v2.0). See Section 6 for the rationale.

**T7: AccessGranted → {Authenticated, Idle}**  
Guard: `state = AccessGranted`  
Action: `state' ∈ {Authenticated, Idle}` (nondeterministic)  
Interpretation: After accessing a resource, the user may either return to the authenticated session (e.g., request another resource) or log out (return to `Idle`). This nondeterminism models the adversarial environment's authority over post-access behaviour.

**T8: Locked → Locked**  
Guard: `state = Locked`  
Action: `state' = Locked`  
Interpretation: The `Locked` state is absorbing; no exit is modelled. This is a conservative abstraction of a permanent lockout policy.

### 3.5 Invariants

Two state invariants are encoded as `INVAR` constraints in NuSMV, which checks them against all reachable states:

**INV1:** `state = Locked → attempts = 3`  
The lockout state is only entered via T5, which requires `attempts ≥ 2` and sets `attempts' = 3`. This invariant confirms that lockout cannot occur with fewer than 3 failures.

**INV2:** `¬session_auth → ¬(state = Authenticated ∨ state = AccessGranted)`  
The session authentication latch is set precisely when entering `Authenticated` via T3. This invariant confirms that neither `Authenticated` nor `AccessGranted` is reachable without a prior authentication event, providing an additional structural sanity check independent of the LTL properties.

---

## 4. Formal Specification

### 4.1 Specification Logic

Properties are expressed in **Linear Temporal Logic (LTL)**, interpreted over the infinite traces of the transition system. The syntax and standard semantics of LTL are assumed; we use the following operators:

- **G φ** (Globally): φ holds at every position i ≥ 0 of the trace.
- **F φ** (Finally): φ holds at some position j ≥ i.
- **X φ** (Next): φ holds at position i + 1.
- **φ U ψ** (Until): φ holds at all positions from i until the first j ≥ i where ψ holds (and ψ must eventually hold).

A property φ is verified by NuSMV using automata-theoretic model checking: φ is negated, the negation ¬φ is converted to a Büchi automaton B_{¬φ}, the synchronous product M ⊗ B_{¬φ} is constructed, and its language is checked for emptiness. If the language is empty, φ holds on all traces of M. If it is non-empty, a witness accepting run (a counterexample trace) is extracted.

### 4.2 Safety Properties

A safety property asserts that no finite prefix of any trace can reach a bad state. Safety properties of the form G φ are verified by checking whether any reachable state violates φ.

**S1 — No Unauthorised Access:**

```
G( state = AccessGranted → session_auth )
```

Semantics: For all traces π and all positions i ≥ 0, if state = `AccessGranted` at position i, then `session_auth = TRUE` at position i.

Interpretation within the formal model: Resource access (`AccessGranted`) is structurally reachable only through the sequence `PasswordEntered → Authenticated → AccessGranted`. Transition T3, which produces `Authenticated`, sets `session_auth = TRUE`. The LTL check independently confirms this reachability constraint across the full trace space without relying on manual inspection of the transition graph.

Note on non-triviality: In principle, S1 could be trivially true if `AccessGranted` were unreachable. The CTL property `EF(state = AccessGranted)` (verified separately) confirms reachability, making S1 non-vacuous.

**S2 — Lockout is Permanent:**

```
G( state = Locked → G( state ≠ AccessGranted ) )
```

Semantics: For all traces π and all positions i, if state = `Locked` at position i, then for all j ≥ i, state ≠ `AccessGranted`.

Interpretation: Once the system enters `Locked`, the absorbing self-loop T8 ensures no state transition is possible. In particular, `AccessGranted` is unreachable from `Locked` by any path of any length. The LTL check confirms this structural property.

### 4.3 Liveness Properties

A liveness property asserts that something good eventually happens on every trace. Liveness properties of the form G(φ → F ψ) assert that whenever φ holds, ψ is eventually reached. These properties are violated by infinite (lasso-shaped) counterexample traces in which ψ is perpetually avoided.

**L1 — Valid Authentication Eventually Succeeds:**

```
G( (state = PasswordEntered ∧ credentials_valid) → F( state = Authenticated ) )
```

Semantics: For all traces π, for all i ≥ 0, if (`PasswordEntered` ∧ `credentials_valid`) holds at position i, then there exists j ≥ i such that `Authenticated` holds at j.

Interpretation: When correct credentials are supplied at the password entry state, the transition T3 fires deterministically (it is the only applicable guard when `credentials_valid = TRUE` at `PasswordEntered`). Therefore, `Authenticated` is reached at position i + 1, satisfying F(Authenticated) in exactly one step. The quantification over all traces accounts for the nondeterminism of `credentials_valid`; the property is conditioned on `credentials_valid = TRUE` holding at the specific position.

**L2 — Authenticated Users Eventually Access Resources:**

```
G( state = Authenticated → F( state = AccessGranted ) )
```

Semantics: For all traces π, for all i ≥ 0, if `Authenticated` holds at position i, then there exists j ≥ i such that `AccessGranted` holds at j.

This property is the subject of the counterexample analysis in Section 6.

---

## 5. Verification Results

### 5.1 Summary Table

| Property | Specification | v1.0 Result | v2.0 Result |
|---|---|---|---|
| S1 | `G(AccessGranted → session_auth)` | **TRUE** | **TRUE** |
| S2 | `G(Locked → G(¬AccessGranted))` | **TRUE** | **TRUE** |
| L1 | `G((PasswordEntered ∧ credentials_valid) → F(Authenticated))` | **TRUE** | **TRUE** |
| L2 | `G(Authenticated → F(AccessGranted))` | **FALSE** ✗ | **TRUE** |

### 5.2 S1 Verification (Both Versions)

**Claim:** `G(state = AccessGranted → session_auth)` is TRUE in M.

**Argument:** By model construction, the only transition whose successor is `AccessGranted` is T6: `state = Authenticated → state' = AccessGranted`. The only transition that reaches `Authenticated` is T3: `state = PasswordEntered ∧ credentials_valid → state' = Authenticated`. Transition T3's action sets `session_auth' = TRUE`. Since `session_auth` is monotonically set (never reset to FALSE within the model), `session_auth = TRUE` holds in every state reachable after T3 fires. Therefore, whenever `state = AccessGranted`, `session_auth = TRUE`. The BDD-based fixpoint computation in NuSMV confirms this by failing to find any reachable state satisfying `AccessGranted ∧ ¬session_auth`.

### 5.3 S2 Verification (Both Versions)

**Claim:** `G(state = Locked → G(state ≠ AccessGranted))` is TRUE in M.

**Argument:** T8 is the only transition applicable when `state = Locked`, and it produces `state' = Locked` (absorbing self-loop). Therefore, from any `Locked` state, the only reachable states are `Locked` states. Since `Locked ≠ AccessGranted`, `state ≠ AccessGranted` holds at all positions after any `Locked` position. The inner G is verified by the absorbing nature of T8.

### 5.4 L1 Verification (Both Versions)

**Claim:** `G((state = PasswordEntered ∧ credentials_valid) → F(state = Authenticated))` is TRUE.

**Argument:** When `state = PasswordEntered` and `credentials_valid = TRUE`, the guard of T3 is satisfied and T3 is the only applicable transition (T4 and T5 require `¬credentials_valid`). Therefore `state' = Authenticated` is the unique successor. F(Authenticated) is thus satisfied at position i + 1. The BDD-based verification confirms no counterexample exists.

### 5.5 L2 Verification — Failure in v1.0, Success in v2.0

**v1.0 — FALSE:** The counterexample is detailed in Section 6. In summary, the nondeterministic transition T6 (`Authenticated → {AccessGranted, Idle}`) permits the adversarial environment to perpetually choose `Idle`, producing an infinite trace in which `Authenticated` recurs but `AccessGranted` is never reached.

**v2.0 — TRUE:** After the refinement (T6 made deterministic: `Authenticated → AccessGranted`), the guard `state = Authenticated` uniquely produces `state' = AccessGranted`. Therefore, for any position i where `state = Authenticated`, F(AccessGranted) is satisfied at position i + 1. No lasso counterexample can exist, and the BDD fixpoint confirms this.

### 5.6 Reachability (CTL)

The following CTL properties confirm non-vacuity:

| Property | Specification | Result |
|---|---|---|
| R1 | `EF(state = Locked)` | **TRUE** (Locked is reachable) |
| R2 | `EF(state = AccessGranted)` | **TRUE** (AccessGranted is reachable) |
| R3 | `EF(state = Authenticated)` | **TRUE** (Authenticated is reachable) |
| R4 | `AG(state = Locked → attempts = 3)` | **TRUE** (lockout requires 3 failures) |

---

## 6. Counterexample Analysis

### 6.1 Property

L2: `G(state = Authenticated → F(state = AccessGranted))`  
Model: `auth_protocol.smv` (v1.0)  
Status: **FALSE** — counterexample exists.

### 6.2 Counterexample Structure

NuSMV produces lasso-shaped counterexamples for liveness properties. A lasso consists of a **stem** (a finite path from the initial state) and a **cycle** (a suffix that repeats infinitely). The cycle witnesses a reachable strongly connected component of the state space in which the desired postcondition (here: `AccessGranted`) is absent.

**Stem:**

| Position | state | attempts | credentials_valid | session_auth |
|---|---|---|---|---|
| 0 | Idle | 0 | — | FALSE |
| 1 | UsernameEntered | 0 | — | FALSE |
| 2 | PasswordEntered | 0 | TRUE | FALSE |
| 3 | Authenticated | 0 | — | TRUE |

At position 3, `state = Authenticated` holds. L2 requires that from position 3 onward, `AccessGranted` is eventually reached. The cycle below shows this does not occur.

**Cycle (loop-back to position 4):**

| Position | state | attempts | credentials_valid | session_auth |
|---|---|---|---|---|
| 4 | Idle | 0 | — | TRUE |
| 5 | UsernameEntered | 0 | — | TRUE |
| 6 | PasswordEntered | 0 | TRUE | TRUE |
| 7 | Authenticated | 0 | — | TRUE |
| (→ back to position 4) | | | | |

`AccessGranted` never appears. The cycle repeats infinitely.

### 6.3 Step-by-Step Analysis

**Step 0 (Idle):** Initial state. `session_auth = FALSE`, `attempts = 0`. The system has not yet attempted authentication.

**Steps 1–2 (UsernameEntered, PasswordEntered):** Transitions T1 and T2 fire deterministically. The counterexample witness assigns `credentials_valid = TRUE` at step 2, enabling T3.

**Step 3 (Authenticated):** Transition T3 fires. `session_auth` is set to TRUE. The L2 obligation is triggered: `F(AccessGranted)` must hold from position 3 onward.

**Step 4 (Idle):** Transition T6 fires with nondeterministic choice selecting `Idle` (rather than `AccessGranted`). The L2 obligation is not discharged. `AccessGranted` was not reached.

**Steps 5–7:** Identical to steps 1–3. The system re-authenticates (the model permits repeated sessions in the single-session abstraction). At step 7, `Authenticated` recurs, and the L2 obligation is re-triggered.

**Loop:** Steps 4–7 repeat infinitely. At no position does `state = AccessGranted` hold. The existential witness (F) is never satisfied. L2 is refuted.

### 6.4 Failure Classification: Modelling Flaw

The failure of L2 in v1.0 is classified as a **modelling flaw**, not a specification error.

**Justification:** The specification correctly captures the intended security requirement. An authenticated user should eventually access a protected resource; this is the purpose of the authentication protocol within the formal model. The specification is sound.

The flaw lies in transition T6 (v1.0): `state = Authenticated → {AccessGranted, Idle}`. This nondeterministic choice assigns to the adversarial environment the authority to determine whether an authenticated user accesses a resource or logs out immediately. This is an incorrect separation of concerns. Within the formal model:

- The **environment's** nondeterminism should model factors outside the protocol's control: credential validity, network behaviour, attacker strategy.
- The **user's** action of accessing a resource (once authenticated) is not an adversarial input; it is an expected user behaviour that the protocol is designed to support.

By making T6 nondeterministic, v1.0 allows the adversary to perpetually prevent resource access, which is not the intended adversary model.

**Consequence:** The counterexample trace is a **spurious counterexample** in the sense that it arises from a modelling imprecision rather than from a genuine protocol vulnerability. No real protocol flaw is indicated.

### 6.5 Refinement

The fix is applied in `auth_protocol_refined.smv` (v2.0):

```
-- v1.0:
state = Authenticated : { AccessGranted, Idle };

-- v2.0:
state = Authenticated : AccessGranted;
```

Logout nondeterminism is preserved at `AccessGranted` via T7: `state = AccessGranted → {Authenticated, Idle}`. This is the semantically appropriate location for the adversarial logout choice: after the user has accessed a resource.

Under this refinement, `Authenticated → AccessGranted` is deterministic. For any position i where `state = Authenticated`, F(AccessGranted) is satisfied at position i + 1. All four LTL properties are verified TRUE in v2.0.

---

## 7. Limitations and Abstractions

### 7.1 Credential Abstraction

The most significant abstraction in this model is the reduction of credential verification to a single nondeterministic boolean. This captures the outcome of credential checking (match / no match) but discards all mechanism-level detail: password strings, hash functions, salts, timing behaviour, and cryptographic strength. Within the formal model, this abstraction is sound for verifying protocol-level control flow properties. It is not sufficient for analysing credential-guessing attacks, side-channel vulnerabilities, or cryptographic protocol properties, which would require a richer model (e.g., a Dolev-Yao attacker with explicit message operations).

### 7.2 Single Session

The model encodes a single authentication session. Multiple concurrent sessions, session management, session tokens, and inter-session interactions are not modelled. The single-session abstraction is appropriate for verifying the protocol's per-session security properties but does not cover session fixation, replay attacks, or concurrent access races.

### 7.3 Permanent Lockout

The `Locked` state is absorbing: once entered, the system remains locked indefinitely. In a deployed system, account recovery (via email verification, admin reset, or timed unlock) would be possible. The permanent lockout is a conservative abstraction that strengthens the safety property S2 (making it easier to verify) but may not faithfully represent all deployment scenarios.

### 7.4 Synchronous Semantics

NuSMV uses synchronous reactive semantics: all variables are updated simultaneously at each discrete step. Real authentication systems operate asynchronously (network delays, concurrent processes, preemptive scheduling). The synchronous abstraction eliminates interleaving anomalies from the model and is appropriate for control-flow verification, but precludes the analysis of race conditions or timing-dependent vulnerabilities.

### 7.5 Nondeterminism Scope

The adversarial environment is modelled solely through the nondeterminism of `credentials_valid`. A more complete adversary model (e.g., the Dolev-Yao model for network protocols) would include the ability to intercept, replay, modify, and inject messages. The present model assumes the communication channel is implicit and trusted; the adversary's only degree of freedom is the credential oracle.

### 7.6 No Real-World Security Claim

The verification results in this report are stated exclusively within the formal model as defined. The model is an abstraction; it omits implementation details, environmental factors, and attack vectors not representable in a boolean finite-state system. No claim is made that any real authentication system satisfying a similar high-level description is secure. Formal verification of the model confirms the model's properties, not the properties of any system that the model may be intended to represent.

---

## 8. Conclusion

We have constructed a formal model of a single-session login authentication protocol as a finite-state transition system in NuSMV and verified four properties expressed in Linear Temporal Logic. The model incorporates an adversarial nondeterministic environment, a bounded failure counter, account lockout, and authenticated resource access.

The initial model (v1.0) satisfies both safety properties (S1: no unauthorised access; S2: lockout permanence) and the first liveness property (L1: valid authentication succeeds). It fails to satisfy the second liveness property (L2: authenticated users access resources). A lasso-shaped counterexample trace was produced and analysed, identifying the failure as a modelling flaw in the nondeterministic assignment of the post-authentication transition. The fix — making the authenticated-to-access transition deterministic while preserving nondeterminism at the post-access logout point — yields the refined model (v2.0), against which all four properties are verified TRUE.

The project demonstrates the model checking workflow as applied to a security protocol: (1) model construction; (2) property specification; (3) verification and counterexample extraction; (4) failure classification; (5) model refinement; (6) reverification. It also illustrates the importance of precise nondeterminism scope: placing adversarial nondeterminism at the wrong architectural point can produce spurious counterexamples that reflect modelling imprecision rather than genuine protocol vulnerabilities.

Future work could extend the model to multiple sessions, parametric lockout thresholds, explicit session tokens, or a richer adversary modelled via explicit Dolev-Yao operations, enabling verification of stronger security properties including authentication agreement and session integrity.

---

## References

Clarke, E. M., Emerson, E. A., & Sistla, A. P. (1986). Automatic verification of finite-state concurrent systems using temporal logic specifications. *ACM Transactions on Programming Languages and Systems*, 8(2), 244–263.

Cimatti, A., Clarke, E., Giunchiglia, E., Giunchiglia, F., Pistore, M., Roveri, M., Sebastiani, R., & Tacchella, A. (2002). NuSMV 2: An OpenSource Tool for Symbolic Model Checking. *Proceedings of CAV 2002*, LNCS 2404, 359–364.

Holzmann, G. J. (1997). The model checker SPIN. *IEEE Transactions on Software Engineering*, 23(5), 279–295.

Lowe, G. (1996). Breaking and fixing the Needham-Schroeder public-key protocol using FDR. *Proceedings of TACAS 1996*, LNCS 1055, 147–166.

Pnueli, A. (1977). The temporal logic of programs. *Proceedings of the 18th Annual Symposium on Foundations of Computer Science*, 46–57.

Kwiatkowska, M., Norman, G., & Parker, D. (2011). PRISM 4.0: Verification of Probabilistic Real-Time Systems. *Proceedings of CAV 2011*, LNCS 6806, 585–591.

---

*End of Report*
