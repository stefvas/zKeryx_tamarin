# zKeryx Tamarin Formal Verification Model# zKeryx Tamarin Formal Verification Model



## Overview## Overview



This directory contains the formal verification model for the **zKeryx** attestation protocol, implemented and verified using the [Tamarin Prover](https://tamarin-prover.github.io/) (version 1.8.0).This directory contains the formal verification model for the **zKeryx** attestation protocol, implemented and verified using the [Tamarin Prover](https://tamarin-prover.github.io/) (version 1.8.0).



The model provides mechanized symbolic verification of zKeryx's three protocol phases (Join, Runtime, Update) under a Dolev-Yao adversary with complete network control, proving **19 security properties** through exhaustive trace analysis.The model provides mechanized symbolic verification of zKeryx's three protocol phases (Join, Runtime, Update) under a Dolev-Yao adversary with complete network control, proving 19 security properties through exhaustive trace analysis.



## Model Files## Model Files



### Main Verification Model### Main Verification Model



**`integrated_model.spthy`** - Complete end-to-end Tamarin specification including:**`integrated_model.spthy`** - Complete end-to-end Tamarin specification including:

- Protocol rules for all three phases (Join, Runtime, Update)  - Protocol rules for all three phases (Join, Runtime, Update)

- Dolev-Yao network adversary with extended capabilities (intercept, delay, replay, inject)- Dolev-Yao network adversary with extended capabilities (intercept, delay, replay, inject)

- 19 security lemmas covering all critical properties- 19 security lemmas covering all critical properties

- **Status:** All lemmas PROVEN ✓- **Status:** All lemmas PROVEN ✓



### Development/Example Models (in backup/)### Development/Example Models (in backup/)



These models were used during protocol development and are kept for reference:These models were used during protocol development and are kept for reference:



- **`state_machine.spthy`** - Minimal FSM example demonstrating Tamarin basics- **`state_machine.spthy`** - Minimal FSM example demonstrating Tamarin basics

- **`join_phase.spthy`** - Isolated join/onboarding verification- **`join_phase.spthy`** - Isolated join/onboarding verification

- **`runtime_phase.spthy`** - Isolated runtime attestation verification  - **`runtime_phase.spthy`** - Isolated runtime attestation verification

- **`simple_test.spthy`** - Protocol sketch for early testing- **`simple_test.spthy`** - Protocol sketch for early testing



**Note:** The `integrated_model.spthy` supersedes all development models and is the canonical verification artifact.**Note:** The `integrated_model.spthy` supersedes all development models and is the canonical verification artifact.



## Verified Protocol Phases## How to run



### 1. Join Phase (Onboarding)Install Tamarin Prover, then from this folder run:

**Purpose:** Initial device registration and VPE provisioning

```bash

**Key Rules:**# Prove all lemmas (batch mode)

- `InitUA`, `InitSE_WithEK`, `InitPlatform`, `InitCounter` - System initializationtamarin-prover --prove state_machine.spthy

- `UA_Package` - Update Authority prepares VPE key and authorization ticket

- `SE_ValidatePolicy` - Secure Element validates policy before acceptance# Or use the interactive GUI

- `SE_Activate` - Complete onboarding after policy validation tamarin-prover interactive --heuristic=O --port=3001 state_machine.spthy

- `UA_Accept` - Update Authority accepts join completion```



**Security Properties:**If you use Docker:

- Only the Update Authority can provision VPE keys

- VPE policy bindings are unique and immutable```bash

- Credential activation provides authenticated channel via RSA-OAEP + AEADdocker run --rm -it -v "$PWD":/src ghcr.io/tamarin-prover/tamarin-prover:latest \

  tamarin-prover --prove /src/state_machine.spthy

### 2. Runtime Phase (Challenge-Response Attestation)```

**Purpose:** Fresh attestation evidence generation with session binding

### Join phase model

**Key Rules:**

- `StartSessions` - Initialize VPE and AK sessions with explicit handle bindingWe also provide `join_phase.spthy`, a minimal model of the join/activate-credential flow:

- `VPE_SignOtherNonce` - VPE validates dual-layer TSS integrity (user-space + secure boot)

- `VerifierChallenge` - Verifier generates fresh challenge nonce- UA packages `(C2 = aenc(r, pk_EK), C1 = senc(<VPE,t,c>, kdf(r, nameAK)))` (AEAD integrity can be modeled as ENC+MAC if desired).

- `AK_Authorize` - Attestation key authorization after policy validation- SE activates credential by decrypting with `sk_EK`, recomputing `kdf`, and importing VPE and ticket.

- `SignAK` - Evidence signature over verifier's challenge

- `VerifierAccepts` - Verifier validates attestation evidenceLemmas checked:



**Security Properties:**  - `onboarding_authenticity`: any successful `SE_Activated` implies a prior `UA_Packaged` for the same `nameAK`.

- Every attestation binds to a fresh verifier challenge (no replay)  - `no_replay_activation`: join activation for a given `nameAK` is unique (prevents replay leading to multiple activations).

- VPE signatures authorize specific AK sessions (session binding integrity)

- Dual-layer TSS validation required (user-space measurements + secure boot PCRs)Run:

- Policy admission enforced before any signature generation

```bash

### 3. Update Phase (Secure Software Updates)tamarin-prover --prove join_phase.spthy

**Purpose:** Cryptographically enforced policy updates with anti-rollback

# Or with Docker

**Key Rules:**docker run --rm -it -v "$PWD":/src ghcr.io/tamarin-prover/tamarin-prover:latest \

- `UA_PrepareUpdate` - Update Authority prepares new policy ticket  tamarin-prover --prove /src/join_phase.spthy

- `SE_ActivateUpdate` - Secure Element receives update package```

- `Verify_Ticket` - Cryptographic verification of UA signature

- `ExtractMeasurements` - TSS extracts current runtime measurements### Runtime phase model

- `ValidateMeasurements` - Validate references match current state (attack detection)

- `CounterIncrement` - Monotonic counter increment after validation`runtime_phase.spthy` models the runtime authorization and evidence flow with cryptographic verification:

- `UpdateChallengeToVerifierFresh` - Bridge update to attestation flow

- **Join integration**: VPE keys are bound to specific nameAK through OnboardedVPE facts

**Security Properties:**- **Real crypto verification**: Ticket signatures and VPE signatures are cryptographically verified using `verify(...)` 

- Updates only accepted when measurements match references (no active attack)- **Attack detection**: Reference values R must equal actual measurements m when no attacks are present; divergence (R ≠ m) indicates compromise and blocks authorization

- Monotonic counter strictly increases with each update- **PolicyOR semantics**: Multiple measurements are serialized and combined in ProcRefs

- Rollback attacks cryptographically prevented- **Key-specific counters**: Monotonic counters are bound to specific attestation keys (nameAK)

- Update authorization requires valid UA signature- **Strict anti-rollback**: Counter values must strictly increase (v < vnext) per key



## Network Adversary ModelLemmas checked:



The model includes a comprehensive Dolev-Yao adversary with the following capabilities:- `freshness_no_replay`: each accepted evidence corresponds to a unique signing event over a fresh nonce

- `policy_admission`: any signing event is preceded by verified ticket, valid measurements (R=m), counters, and VPE linkage  

**Adversary Rules:**- `anti_rollback_progress`: counters are monotonic per trace

- `Net_Send`, `Net_Deliver` - Normal message delivery- `no_rollback`: counter values strictly increase between signing events for same key

- `Adv_Intercept` - Intercept and observe any network message- `counter_strict_monotonic`: each counter increment enforces v < vnext

- `Adv_Inject` - Inject previously observed messages (replay attacks)- `update_requires_increment`: signing requires prior counter increment

- `Net_Delay`, `Net_DeliverDelayed` - Delay message delivery arbitrarily- `measurement_integrity`: signing requires valid measurement matching references (attack detection)



**Adversary Capabilities:**Run:

- ✓ Complete network control (intercept, delay, reorder, replay)

- ✓ Arbitrary message injection and replay```bash

- ✓ Temporal kernel compromise (modeled via measurement divergence)tamarin-prover --prove runtime_phase.spthy

- ✗ Cannot break cryptographic primitives (perfect cryptography assumption)

- ✗ Cannot extract secrets from Secure Element# Or with Docker

docker run --rm -it -v "$PWD":/src ghcr.io/tamarin-prover/tamarin-prover:latest \

**Out of Scope:**  tamarin-prover --prove /src/runtime_phase.spthy

- Physical attacks on Secure Element```

- Side-channel attacks

- Denial-of-service attacks### Integrated end-to-end model



## Verified Security Properties (19 Lemmas)`integrated_model.spthy` provides complete end-to-end verification combining join, runtime, and update phases:



### Category 1: Onboarding & Policy Admission (3 properties)- **Complete lifecycle**: From device onboarding through runtime attestation to secure updates

- **Cross-phase security**: Join success required for runtime operations; VPE binding persists across phases

#### 1. `onboarding_authenticity`- **Dual-layer TSS validation**: VPE policy validates both user-space TSS (measurement match) and lower-ring TSS (secure boot PCR)

**Property:** Each VPE onboarding occurs exactly once; no duplicate onboarding for the same nameAK and vpepk.- **Attack detection**: Measurement validation detects application compromise (R ≠ m blocks authorization)

- **Update integrity**: Counter increments gated by validated measurements and cryptographic verification

**Significance:** Ensures bijective mapping between VPE instances and attestation keys, preventing replay of onboarding ceremonies.- **End-to-end properties**: Signing requires completed onboarding, dual TSS validation, measurement validation, and counter progression



**Proof Technique:** Proof by contradiction via freshness of cryptographic nonces.Key end-to-end lemmas:



---- `signing_requires_onboarding`: Any attestation signature requires prior successful join

- `vpe_requires_tss_integrity`: VPE signing requires both user-space and lower-ring TSS validation

#### 2. `vpe_unique_binding`- `secure_boot_integrity`: VPE unlocking requires expected secure boot PCR (lower-ring integrity)

**Property:** VPE policy bindings (to TSS measurements and secure boot state) are unique per VPE instance.- `tss_userspace_integrity`: VPE requires user-space TSS measurement match against reference

- `measurement_integrity`: Signing blocked when measurements don't match references (attack detected)

**Significance:** Prevents adversarial policy substitution or confused deputy attacks; measurement context and session binding cannot be altered post-onboarding.- `policy_admission`: Complete policy satisfaction from onboarding through validation



**Proof Technique:** Temporal case-splitting followed by contradiction on fresh nonce dependencies.Run:



---```bash

tamarin-prover --prove integrated_model.spthy

#### 3. `signing_requires_onboarding`

**Property:** Every attestation signature must be preceded by successful onboarding.# Or with Docker  

docker run --rm -it -v "$PWD":/src ghcr.io/tamarin-prover/tamarin-prover:latest \

**Significance:** End-to-end security property ensuring all attestation evidence is rooted in UA-authorized onboarding.  tamarin-prover --prove /src/integrated_model.spthy

```

**Proof Technique:** Direct proof via unsolvable constraints; signature premises trace to SE_Activate event.

## Pattern: encoding an FSM in Tamarin

---

1. States as terms: declare constructors (e.g., `Init/0`, `A/0`, `B/0`).

### Category 2: Freshness & Replay Protection (2 properties)2. Linear state fact: `State(s)` appears exactly once and is consumed/produced by transitions.

3. Transitions as rules: one rule per edge, emitting a `Step(s1,s2)` event.

#### 4. `freshness_no_replay`4. Invariants as lemmas: express safety (forbidden events never occur) and structural invariants (step edges allowed, uniqueness of state).

**Property:** Every verifier acceptance is causally linked to a unique challenge-response sequence with strict temporal ordering: challenge generation < signature < acceptance.5. Liveness as existential lemmas: show there exists a trace that reaches certain states; for deadlock freedom, prove that from non-terminal states, some step is always possible (often needs fairness assumptions).



**Significance:** Prevents replay attacks; signatures cannot be reused across sessions.## Extending to your protocol



**Proof Technique:** Direct constraint satisfaction tracing VerifierAccept → VerifierChallenge → SignAKEv.- Add inputs/outputs to rules to model I/O or cryptography.

- Introduce per-session tags or IDs for multiple machines: e.g., `State(m,s)` with machine identifier `m`.

---- Encode bad transitions explicitly and prove they are unreachable.

- Use `Fresh` facts to model nonces and `Out/In` for Dolev-Yao network.

### Category 3: Anti-Rollback & Monotonic Counters (3 properties)

## Troubleshooting

#### 5. `counter_strict_monotonic`

**Property:** Monotonic counter increments always produce strictly different values (v ≠ vnext).- If `no_bad` fails, check that no rule introduces `BadTrig()` or similar enabling facts.

- If `only_allowed_steps` fails, ensure every transition rule emits `Step(s1,s2)` and there are no “silent” edges.

**Significance:** Foundation for anti-rollback protection; each update strictly advances version.- For uniqueness, avoid rules that duplicate `State(_)` without consuming it.


**Proof Technique:** Proof by contradiction on inc() function specification.

---

#### 6. `no_rollback`
**Property:** Temporally ordered signature events following distinct updates must utilize distinct counter values.

**Significance:** End-to-end anti-rollback guarantee; prevents downgrades to vulnerable software versions.

**Proof Technique:** Direct constraint solving tracing signatures through counter update dependencies.

---

#### 7. `update_requires_increment`
**Property:** Every update operation is causally dependent on a monotonic counter increment.

**Significance:** Enforces version tracking discipline; establishes complete audit trail of software evolution.

**Proof Technique:** Direct constraint satisfaction from UpdateChallengeReady → CounterUpdated.

---

### Category 4: Session Binding & Authorization (4 properties)

#### 8. `session_binding_integrity`
**Property:** Every SessionBound event traces to valid SessionBinding operation with matching hash commitments (hnua = h(nua)).

**Significance:** Prevents session confusion and forgery attacks; ensures cryptographically enforced session provenance.

**Proof Technique:** Exhaustive case-splitting with equation solving through 9 premise origins.

---

#### 9. `vpe_session_authorization`
**Property:** VPESessionAuthorized events are atomically coupled with SessionBound events at the same timepoint.

**Significance:** Ensures VPE authorization is traceable to legitimate session initialization with no temporal gaps.

**Proof Technique:** Proof by contradiction on atomic rule emission.

---

#### 10. `ak_requires_session_verification`
**Property:** AK authorization requires prior session verification with valid VPE signature.

**Significance:** Establishes authorization chain: AKAuthorized → SessionVerified → SessionBound → SessionBinding → StartSessions.

**Proof Technique:** Case-splitting with equation solving on signature verification branches.

---

#### 11. `session_signature_uniqueness`
**Property:** Session binding tuples (hvpe, hak, nua) occur at most once in any valid trace.

**Significance:** Prevents session confusion and provides deterministic session state tracking with strong temporal integrity.

**Proof Technique:** Symmetric proof by contradiction with temporal case-splitting; exploits fresh nonce uniqueness.

---

### Category 5: TSS Dual-Layer Validation (3 properties)

#### 12. `vpe_requires_tss_integrity`
**Property:** Every VPE signature requires simultaneous validation of both TSS layers (user-space + secure boot) at the same timepoint.

**Significance:** Complete trust chain from hardware-rooted platform integrity through application-level security properties.

**Proof Technique:** Proof by contradiction; TSS_Meas and PCR_SB premises must exist for VPE signature.

---

#### 13. `tss_dual_layer_validation`
**Property:** AK attestation signatures require prior VPEPolicyOK event encompassing both TSS layers.

**Significance:** Creates authorization gate where every AK signature is rooted in policy-compliant dual-layer validation.

**Proof Technique:** Direct constraint solving via AK_Authorized fact backward chaining.

---

#### 14. `tss_dual_validation_required`
**Property:** Every VPEPolicyOK event requires simultaneous TSSUserSpaceValid and TSSLowerRingValid events.

**Significance:** Definitional correctness of policy validation; proves complete attestation chain with no bypass paths.

**Proof Technique:** Proof by contradiction on atomic VPE_SignOtherNonce rule.

---

### Category 6: Measurement Integrity (3 properties)

#### 15. `measurement_extraction_integrity`
**Property:** Measurement validation requires prior measurement extraction with matching value.

**Significance:** Ensures validated measurements originate from genuine VPE state, not adversarial fabrication.

**Proof Technique:** Direct constraint satisfaction tracing MeasurementValidated → ExtractMeasurements.

---

#### 16. `measurement_integrity`
**Property:** Every attestation signature is preceded by NoAttack validation confirming measurement integrity.

**Significance:** End-to-end guarantee that only validated measurements are incorporated into attestation evidence.

**Proof Technique:** Backward causal analysis tracing SignAKEv → NoAttack validation.

---

#### 17. `vpe_policy_binding`
**Property:** Successful onboarding implies VPE policy binding to specific TSS reference and secure boot measurements.

**Significance:** Links onboarding to measurement expectations, establishing baseline for runtime validation.

**Proof Technique:** Direct existence proof from OnboardingSuccess → VPEPolicyBinding.

---

### Category 7: Network Security & End-to-End Properties (2 properties)

#### 18. `network_authenticity`
**Property:** All delivered messages originate from legitimate protocol participants or are explicitly adversary-injected.

**Significance:** Validates Dolev-Yao network model completeness; protocol resilient to active network attacks.

**Proof Technique:** Disjunctive proof over Sent and Injected events.

---

#### 19. `policy_admission`
**Property:** Every attestation signature is temporally preceded by satisfied policy premises in correct order: onboarding, validated state, VPE signature, measurement validation, runtime ticket verification, session verification.

**Significance:** Central end-to-end security property connecting all protocol phases; proves complete authorization chain.

**Proof Technique:** Complex constraint satisfaction proving existence of all prerequisite events with correct temporal ordering.

---

## Verification Summary

| Category | Properties | Status | Verification Time |
|----------|-----------|--------|-------------------|
| Onboarding & Policy Admission | 3 | ✓ Proven | Sub-second to 10s |
| Freshness & Replay Protection | 2 | ✓ Proven | Sub-second |
| Anti-Rollback & Counters | 3 | ✓ Proven | 1-30s |
| Session Binding & Authorization | 4 | ✓ Proven | 10s-2min |
| TSS Dual-Layer Validation | 3 | ✓ Proven | 5-30s |
| Measurement Integrity | 3 | ✓ Proven | 1-20s |
| Network & End-to-End | 2 | ✓ Proven | 1-60s |
| **TOTAL** | **19** | **✓ All Proven** | **Minutes** |

**Verification Platform:** Tamarin Prover v1.8.0  
**Proof Techniques:** Exhaustive symbolic trace analysis with constraint solving  
**Adversary Model:** Dolev-Yao with complete network control

## Cryptographic Primitives

The model uses Tamarin's built-in cryptographic functions:

```tamarin
builtins: hashing, signing, symmetric-encryption, asymmetric-encryption
```

**Hash Functions:** Modeled as public function `h(·)` - collision-resistant, one-way  
**Signatures:** `sign(m, sk)` and `verify(sig, m, pk)` - EUF-CMA secure  
**Symmetric Encryption:** `senc(m, k)` and `sdec(c, k)` - IND-CCA secure AEAD  
**Asymmetric Encryption:** `aenc(m, pk)` and `adec(c, sk)` - IND-CCA secure (RSA-OAEP)

**Custom Functions:**
- `kdf/2` - Key derivation function (uninterpreted, models HKDF)
- `digest_pre/3`, `digest_final/1` - Policy digest construction
- `session_handle/2` - Session handle generation
- `inc/1` - Monotonic increment (no inverse)

## Running the Verification

### Prerequisites
```bash
# Install Tamarin Prover (https://tamarin-prover.github.io/)
# Requires: GHC, Graphviz, Maude

# Ubuntu/Debian
sudo apt-get install tamarin-prover

# macOS
brew install tamarin-prover
```

### Verify All Lemmas
```bash
cd /path/to/attestation/tamarin
tamarin-prover integrated_model.spthy --prove
```

### Verify Specific Lemma
```bash
tamarin-prover integrated_model.spthy --prove=freshness_no_replay
```

### Interactive Mode
```bash
tamarin-prover interactive integrated_model.spthy
# Open browser to http://localhost:3001
```

### Generate Proof Trace
```bash
tamarin-prover integrated_model.spthy --prove=policy_admission --output=proof_trace.txt
```

### Using Docker
```bash
docker run --rm -it -v "$PWD":/src ghcr.io/tamarin-prover/tamarin-prover:latest \
  tamarin-prover --prove /src/integrated_model.spthy
```

## Model Structure

```
integrated_model.spthy (319 lines)
├── Theory Header (Lines 1-5)
│   └── Cryptographic builtins and custom functions
│
├── JOIN PHASE (Lines 6-65)
│   ├── Initialization Rules
│   ├── UA Package & Credential Activation
│   └── VPE Provisioning & Onboarding
│
├── RUNTIME PHASE (Lines 66-130)
│   ├── Session Management
│   ├── VPE Session (Dual-layer TSS validation)
│   ├── AK Session (Policy accumulation)
│   └── Evidence Generation & Verification
│
├── UPDATE PHASE (Lines 131-210)
│   ├── Network Adversary Model
│   ├── Update Preparation & Delivery
│   ├── Measurement Extraction & Validation
│   └── Monotonic Counter Increment
│
└── SECURITY LEMMAS (Lines 211-319)
    ├── Onboarding Properties (3)
    ├── Freshness Properties (2)
    ├── Anti-Rollback Properties (3)
    ├── Session Binding Properties (4)
    ├── TSS Validation Properties (3)
    ├── Measurement Integrity Properties (3)
    └── Network & End-to-End Properties (2)
```

## Key Verification Results

### Soundness
All 19 lemmas are **proven** under exhaustive symbolic analysis. No false attacks or invalid traces exist.

### Completeness
The model covers all protocol phases and all critical security properties identified in the paper.

### Adversary Strength
The Dolev-Yao adversary has:
- ✓ Complete network control
- ✓ Message interception, delay, replay
- ✓ Arbitrary message injection
- ✓ Temporal compromise (via measurement divergence)
- ✗ No cryptographic breaks (perfect cryptography)
- ✗ No physical Secure Element access

## Correspondence to Paper

| Paper Section | Tamarin Rules | Verified Lemmas |
|--------------|---------------|-----------------|
| Section 5.2 (Join) | `InitUA`, `UA_Package`, `SE_ValidatePolicy`, `SE_Activate` | `onboarding_authenticity`, `vpe_unique_binding` |
| Section 5.2 (Runtime) | `StartSessions`, `VPE_SignOtherNonce`, `AK_Authorize`, `SignAK` | `freshness_no_replay`, `session_binding_integrity`, `tss_dual_validation_required` |
| Section 5.2 (Update) | `UA_PrepareUpdate`, `ValidateMeasurements`, `CounterIncrement` | `counter_strict_monotonic`, `no_rollback`, `measurement_integrity` |
| Section 6 (Verification) | All rules | All 19 lemmas |

## Assumptions and Limitations

### Assumptions
1. **Perfect Cryptography:** Cryptographic primitives are unbreakable (standard Dolev-Yao)
2. **Secure Element Integrity:** SE cannot be physically compromised
3. **Secure Boot Trust:** Platform secure boot chain is trusted
4. **Measurement Extraction:** TSS measurement extraction is faithful (when uncompromised)

### Limitations
1. **Symbolic Model:** Does not capture computational security or timing
2. **No Side Channels:** Does not model cache timing, power analysis, etc.
3. **No DoS:** Denial-of-service attacks are out of scope
4. **No Post-Quantum:** Current model uses classical cryptography (PQ variant separate)

### Future Work
- Mechanized verification of post-quantum variant (ML-KEM, ML-DSA)
- Computational security proofs (CryptoVerif/EasyCrypt)
- Concurrent composition analysis
- Performance-security tradeoff analysis

## Citation

If you use this Tamarin model in your research, please cite:



## Contact and Support

For questions about the Tamarin model:
1. Check Tamarin documentation: https://tamarin-prover.github.io/
2. Review paper Section 6 (Verification) and Appendix A
3. Open an issue on the repository (after publication)


**Tamarin Version:** 1.8.0  
**Verification Status:** All 19 lemmas PROVEN ✓
