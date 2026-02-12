# Semaphore on Starknet — Complete Hackathon Build Guide

## Re{define} Hackathon Context

- **Hackathon:** Re{define} by Starknet Foundation
- **Dates:** February 1–28, 2026 (you're in it right now)
- **Track:** Privacy ($9,675 in STRK tokens)
- **Submission:** DoraHacks — GitHub repo, demo video (≤3 min), Starknet deployment link
- **Deadline:** Feb 28, 2026, 23:59 UTC
- **Winners announced:** March 15
- **Post-hackathon:** Winners can apply for seed grants up to $25K

---

## 1. What Semaphore Actually Does (The Protocol)

Semaphore is a zero-knowledge protocol that solves one problem: **prove you belong to a group and send a signal, without revealing who you are.**

### The Three Core Operations

1. **Identity Creation** (off-chain, on user's device)
   - User generates an EdDSA private key (Semaphore V4)
   - From the private key, derive the public key (Baby Jubjub curve point)
   - The **identity commitment** = Poseidon hash of the public key
   - The commitment is the only thing that goes on-chain

2. **Group Membership** (on-chain)
   - An admin adds identity commitments as leaves in a Merkle tree
   - Semaphore V4 uses a **Lean Incremental Merkle Tree** (cheaper insertions than V3)
   - The Merkle root represents the current state of the group

3. **Anonymous Signaling** (proof generated off-chain, verified on-chain)
   - User generates a ZK proof that proves:
     - "I know a secret that corresponds to a commitment in this Merkle tree"
     - "Here is my nullifier for this scope" (prevents double-signaling)
     - "Here is my message"
   - The proof is submitted on-chain and verified
   - Nobody can tell WHICH member sent the signal

### The Semaphore V4 Circuit (Circom)

The actual circuit is surprisingly small (~23 lines + 33 lines for Merkle tree). Here's the logic:

```
PRIVATE INPUTS:
  - secret (EdDSA private key / secret scalar)
  - merkleProofSiblings[] (sibling hashes along the path)
  - merkleProofIndices[] (left/right direction at each level)
  - merkleProofLength (actual depth used)

PUBLIC INPUTS/OUTPUTS:
  - merkleRoot (the group's current Merkle root)
  - nullifier (hash of scope + secret — prevents double-signal)
  - message (the anonymous signal/vote/endorsement)
  - scope (defines the context — e.g., "election-2026")

CIRCUIT LOGIC:
  1. Derive public key from secret using Baby Jubjub: (Ax, Ay) = BabyPbk(secret)
  2. Compute identity commitment: commitment = Poseidon(Ax, Ay)
  3. Verify Merkle proof: check commitment is a leaf under merkleRoot
  4. Compute nullifier: nullifier = Poseidon(scope, secret)
  5. Square the message (constraint to bind it to the proof)
```

### Key Insight for Your Port

The original uses Circom + Groth16 + snarkjs. For Starknet, you have **two viable paths**:

**Path A: Use Garaga to verify Groth16/Noir proofs on Starknet**
- Keep the Circom circuit (or rewrite in Noir)
- Use Garaga SDK to generate a Cairo verifier contract
- Proof generated off-chain with snarkjs/Noir, verified on-chain via Garaga
- Faster to build, proven tooling, explicitly listed on the hackathon resources page

**Path B: Native Cairo implementation**
- Rewrite the entire proof logic in Cairo
- Leverage Starknet's native STARK proving
- More ambitious, more "Starknet-native"
- Higher risk for a hackathon timeline

**Recommendation for the hackathon: Path A with Noir + Garaga.** It's faster, the tooling is ready, and the hackathon explicitly lists Garaga as a resource. You can always go native later.

---

## 2. Architecture — What You Need to Build

### On-Chain (Cairo Smart Contracts)

#### Contract 1: SemaphoreGroups
```
Responsibilities:
  - Create groups (with admin, Merkle tree depth)
  - Add members (insert identity commitment into Merkle tree)
  - Remove members
  - Store and update Merkle roots
  - Emit events for off-chain indexing

Storage:
  - groups: Map<group_id, GroupData>
  - GroupData: { admin, merkle_root, depth, member_count }
  - members: Map<(group_id, index), felt252>  // commitment storage
```

#### Contract 2: SemaphoreVerifier
```
Responsibilities:
  - Verify ZK proofs (via Garaga if using Path A)
  - Check nullifier hasn't been used (prevent double-signaling)
  - Validate Merkle root matches the group
  - Emit signal events

Storage:
  - nullifiers: Map<(group_id, scope, nullifier), bool>

Key function:
  validateProof(group_id, merkle_root, nullifier, message, scope, proof) -> bool
```

#### Contract 3: Example Application (e.g., Anonymous Voting)
```
Responsibilities:
  - Create polls/votes using Semaphore groups
  - Count votes
  - Demonstrate real-world usage

This is what makes your submission stand out — show a USE CASE.
```

### Off-Chain (TypeScript/JavaScript SDK)

```
Components needed:
  1. Identity module
     - Generate EdDSA keypair
     - Compute identity commitment (Poseidon hash of public key)
     - Export/import identity

  2. Group module
     - Maintain local copy of the Merkle tree
     - Generate Merkle proofs for a given member
     - Sync with on-chain state

  3. Proof module
     - Generate ZK proof using Noir/Circom circuit
     - Package proof for on-chain submission

  4. Simple frontend (optional but impressive for demo)
     - Create identity
     - Join a group
     - Cast anonymous vote/signal
     - View results
```

---

## 3. Existing Building Blocks (Don't Reinvent the Wheel)

### Starknet/Cairo Primitives — Already Available

| Component | Source | Notes |
|-----------|--------|-------|
| **Poseidon hash** | Cairo core library (`core::poseidon`) | Native builtin, extremely cheap. Use `PoseidonTrait::new()`, `.update()`, `.finalize()` |
| **Pedersen hash** | Cairo core library (`core::pedersen`) | Available but Poseidon preferred for ZK |
| **Merkle tree** | Starknet by Example + OpenZeppelin | Full implementations exist with Poseidon |
| **Merkle proof verification** | `openzeppelin_merkle_tree` (v3.0.0) | Has `verify_poseidon()` built in |
| **ERC20/access control** | OpenZeppelin Cairo Contracts | For any token-gating or admin patterns |
| **Garaga** | `garaga` npm package + CLI | Groth16 and Noir/Honk proof verification on Starknet |

### Key Libraries & Tools

```
Cairo/Starknet:
  - scarb              — Cairo package manager
  - starknet-foundry    — Testing framework (snforge, sncast)
  - OpenZeppelin Cairo  — github.com/OpenZeppelin/cairo-contracts
  - Garaga              — github.com/keep-starknet-strange/garaga

JavaScript/TypeScript:
  - @semaphore-protocol/core    — Reference implementation (identity, group, proof)
  - @semaphore-protocol/identity — EdDSA identity generation
  - @semaphore-protocol/group   — Lean Incremental Merkle Tree
  - @semaphore-protocol/proof   — Proof generation/verification
  - starknet.js                 — Starknet JavaScript SDK
  - @ericnordelo/strk-merkle-tree — Starknet-friendly Merkle trees

ZK Circuits:
  - Circom + snarkjs     — Original Semaphore circuit (Groth16)
  - Noir (nargo)         — Alternative ZK language, works with Garaga
  - scaffold-garaga      — Noir + Garaga + Starknet starter template
```

### Hackathon-Provided Resources (from the Re{define} page)

- **Starknet Privacy Toolkit**: github.com/omarespejel/starknet-privacy-toolkit (end-to-end reference)
- **Garaga Documentation**: garaga.gitbook.io/garaga
- **scaffold-garaga**: github.com/KevinSheeranxyj/scaffold-garaga (starter template!)
- **Semaphore Protocol**: semaphore.pse.dev (reference implementation)
- **Semaphore Docs**: docs.semaphore.pse.dev
- **Privacy Workshop Video**: youtube.com/watch?v=vgawLi0gT98
- **ZK-SNARK Verification Workshop**: youtube.com/watch?v=TxFLvXvYByM

---

## 4. Step-by-Step Build Plan

### Week 1 (Feb 1–7): Foundation & Learning

**Day 1–2: Study the protocol**
- Read Semaphore V4 docs thoroughly: docs.semaphore.pse.dev
- Read the Semaphore Technical Overview: hackmd.io/@vplasencia/B1sCrsoFkg
- Study the actual Circom circuit: github.com/semaphore-protocol/semaphore/blob/main/packages/circuits/src/semaphore.circom
- Watch both hackathon workshop videos

**Day 3–4: Set up tooling**
- Install Scarb, starknet-foundry, Cairo
- Clone scaffold-garaga as your project base
- Install Noir (nargo) and Garaga CLI
- Run through the Garaga quickstart to verify a simple proof on Starknet testnet
- Get comfortable with the declare → deploy → verify-onchain flow

**Day 5–7: Build the Merkle tree contract**
- Start with Starknet by Example's Merkle tree implementation
- Adapt it for incremental insertions (Lean IMT style)
- Use Poseidon hash throughout
- Write tests with starknet-foundry

### Week 2 (Feb 8–14): Core Smart Contracts

**Day 8–10: Group management contract**
- Implement group creation, member addition/removal
- Admin controls (who can add members)
- Events for off-chain indexing
- Use OpenZeppelin access control components

**Day 11–14: Write the ZK circuit + verifier**
- Write the Semaphore circuit in Noir (or adapt the Circom version)
- The circuit needs to prove: Merkle membership + nullifier generation
- Use Garaga to generate the Cairo verifier from your circuit
- Deploy verifier to Starknet testnet
- Test end-to-end: generate proof off-chain → verify on-chain

### Week 3 (Feb 15–21): Integration & Application

**Day 15–17: Build the off-chain SDK**
- Identity generation (EdDSA keypair + Poseidon commitment)
- Merkle proof generation from local tree copy
- Proof generation wrapper (calls Noir/snarkjs)
- Transaction builder for Starknet

**Day 18–21: Build a demo application**
- Anonymous voting is the easiest compelling demo
- Create a simple frontend (React + starknet.js + StarknetKit)
- Flow: Create Identity → Join Group → Cast Vote → See Results
- Nobody can tell who voted what, but double-voting is prevented

### Week 4 (Feb 22–28): Polish & Submit

**Day 22–24: Testing and deployment**
- Deploy all contracts to Starknet Sepolia testnet
- End-to-end testing
- Fix bugs, handle edge cases

**Day 25–27: Documentation + Demo Video**
- Write clear README with architecture diagram
- Record ≤3 min demo video showing the full flow
- Document what's novel about your approach

**Day 28: Submit on DoraHacks**
- GitHub repo link
- Demo video
- Starknet deployment link
- Project description

---

## 5. The Noir + Garaga Approach (Recommended Path)

### Why This Path

Garaga is explicitly supported by the hackathon. It lets you write your ZK circuit in Noir, auto-generate a Cairo verifier, and deploy it on Starknet. No need to manually implement pairing-based cryptography in Cairo.

### Step-by-step with Garaga

```bash
# 1. Install Noir
noirup --version 1.0.0-beta.3
nargo --version

# 2. Install Garaga
pip install garaga

# 3. Write your Semaphore circuit in Noir
# The circuit proves:
#   - I know a secret that hashes to a commitment in the Merkle tree
#   - My nullifier is correctly derived from (scope, secret)

# 4. Compile the Noir circuit
nargo compile

# 5. Generate the Cairo verifier with Garaga
garaga gen --system honk --vk target/vk.json

# 6. Declare and deploy the verifier on Starknet
garaga declare
garaga deploy --class-hash <CLASS_HASH>

# 7. Generate a proof and verify it on-chain
nargo prove
garaga verify-onchain --address <CONTRACT> --vk vk.json --proof proof.json --public-inputs inputs.json
```

### Your Noir Circuit (Pseudocode)

```noir
// semaphore.nr
fn main(
    // Private inputs
    secret: Field,
    merkle_siblings: [Field; DEPTH],
    merkle_indices: [u1; DEPTH],

    // Public inputs
    merkle_root: pub Field,
    nullifier: pub Field,
    message: pub Field,
    scope: pub Field,
) {
    // 1. Compute identity commitment
    // Note: In Noir you'd use a compatible hash (Poseidon or Pedersen)
    let commitment = poseidon_hash([secret]);

    // 2. Verify Merkle proof
    let mut current = commitment;
    for i in 0..DEPTH {
        if merkle_indices[i] == 0 {
            current = poseidon_hash([current, merkle_siblings[i]]);
        } else {
            current = poseidon_hash([merkle_siblings[i], current]);
        }
    }
    assert(current == merkle_root);

    // 3. Verify nullifier
    let expected_nullifier = poseidon_hash([scope, secret]);
    assert(nullifier == expected_nullifier);

    // 4. Bind message to proof (prevents tampering)
    let _msg_sq = message * message;
}
```

**Important caveat:** You'll need to ensure the Poseidon hash parameters in your Noir circuit match what your Cairo contracts use (Starknet's Poseidon). Garaga now supports Starknet Poseidon flavour — check the latest release notes.

---

## 6. Cairo Contract Sketches

### Group Management Contract

```cairo
#[starknet::interface]
pub trait ISemaphoreGroups<TContractState> {
    fn create_group(ref self: TContractState, admin: ContractAddress, depth: u8) -> u256;
    fn add_member(ref self: TContractState, group_id: u256, commitment: felt252);
    fn remove_member(ref self: TContractState, group_id: u256, commitment: felt252, proof: Span<felt252>);
    fn get_root(self: @TContractState, group_id: u256) -> felt252;
    fn get_depth(self: @TContractState, group_id: u256) -> u8;
    fn is_member(self: @TContractState, group_id: u256, commitment: felt252) -> bool;
}
```

### Signal Verification Contract

```cairo
#[starknet::interface]
pub trait ISemaphore<TContractState> {
    fn validate_proof(
        ref self: TContractState,
        group_id: u256,
        merkle_root: felt252,
        nullifier: felt252,
        message: felt252,
        scope: felt252,
        proof: Span<felt252>,  // ZK proof data
    ) -> bool;

    fn is_nullifier_used(
        self: @TContractState,
        group_id: u256,
        nullifier: felt252,
    ) -> bool;
}
```

### Using OpenZeppelin Merkle Proof

```cairo
// In your Scarb.toml:
// [dependencies]
// openzeppelin_merkle_tree = "3.0.0"
// openzeppelin_access = "3.0.0"

use openzeppelin_merkle_tree::merkle_proof::verify_poseidon;
use core::poseidon::PoseidonTrait;
use core::hash::{HashStateTrait, HashStateExTrait};
```

---

## 7. What Makes a Winning Submission

Based on past Starknet hackathon winners and judge criteria:

1. **Working deployment on Starknet testnet** — Not just code, actually deployed and functional
2. **Clear demo video** — Show the full user flow in under 3 minutes
3. **Compelling use case** — Anonymous voting, whistleblowing, private credentials
4. **Clean code + documentation** — Architecture diagram, clear README, commented code
5. **Technical depth** — Show you understand ZK, not just copy-pasted
6. **Novelty for Starknet** — Semaphore doesn't exist on Starknet yet, so you're first-mover

### Differentiators to Consider

- **Multi-group support** — Users can be in multiple groups simultaneously
- **On-chain + off-chain hybrid** — Show how groups can be managed efficiently
- **Gas optimization** — Lean IMT is cheaper than standard Merkle; show benchmarks
- **SDK/library** — Make it easy for other Starknet devs to integrate Semaphore
- **Real application** — Don't just build the protocol; build something ON it (voting, whistleblowing, anonymous feedback)

---

## 8. Key Resources (Bookmarks)

### Must-Read
- Semaphore V4 Technical Overview: https://hackmd.io/@vplasencia/B1sCrsoFkg
- Semaphore V4 Circom Circuit: https://github.com/semaphore-protocol/semaphore/blob/main/packages/circuits/src/semaphore.circom
- Garaga Documentation: https://garaga.gitbook.io/garaga
- Cairo Book (Hashing): https://www.starknet.io/cairo-book/ch12-04-hash.html
- Starknet by Example (Merkle Tree): https://starknet-by-example.voyager.online/applications/merkle_tree/
- OpenZeppelin Cairo Merkle Tree: https://docs.openzeppelin.com/contracts-cairo/2.x/api/merkle-tree

### Starter Templates
- scaffold-garaga (Noir + Garaga + Starknet): https://github.com/KevinSheeranxyj/scaffold-garaga
- Starknet Privacy Toolkit: https://github.com/omarespejel/starknet-privacy-toolkit
- Scaffold-Stark (general dApp template): https://github.com/Scaffold-Stark/scaffold-stark-2

### Hackathon-Specific
- Register: https://dorahacks.io/hackathon/redefine/detail
- Telegram support: https://t.me/+-5zNW47GSdQ1ZDkx
- Hackathon site: https://hackathon.starknet.org

### Workshop Videos
- Privacy Preserving Apps: https://www.youtube.com/watch?v=vgawLi0gT98
- ZK-SNARK Verification on Starknet: https://www.youtube.com/watch?v=TxFLvXvYByM

---

## 9. Potential Pitfalls & Tips

### Hash Function Compatibility
The biggest trap: **your off-chain Poseidon must match on-chain Poseidon.** Starknet uses a specific Poseidon variant (3-element state, Hades permutation) over the STARK field (p = 2^251 + 17·2^192 + 1). Make sure your Noir circuit uses the same parameters. Garaga's Starknet Poseidon flavour handles this.

### Merkle Tree Consistency
Your off-chain Merkle tree (JavaScript) must produce the same roots as your on-chain tree (Cairo). Use the same hash function, same ordering (sorted pairs for OpenZeppelin compatibility), same tree structure.

### Gas Costs
Starknet gas is cheap but not free. The Lean IMT approach (Semaphore V4) avoids zero-hash computations, making insertions significantly cheaper. Benchmark your contract and include gas costs in your submission.

### Proof Size
Groth16 proofs are small (~128 bytes), Honk proofs are larger. Consider which system your Garaga verifier uses and the calldata cost implications.

### Time Management
You have ~16 days left. Prioritize getting a working end-to-end flow over perfection. A deployed, working anonymous voting dApp with basic Semaphore beats a perfect but unfinished protocol implementation.