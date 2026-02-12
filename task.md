# Semaphore on Starknet - Task List

- [/] **Project Setup & Initialization** <!-- id: 0 -->
    - [/] Verify/Install dependencies (Scarb, Starknet Foundry, Noir, Garaga) <!-- id: 1 -->
    - [/] Initialize project repository structure (monorepo) <!-- id: 2 -->
    - [ ] Configure `Scarb.toml` and workspace settings <!-- id: 3 -->

- [ ] **Phase 1: ZK Circuits (Noir)** <!-- id: 4 -->
    - [ ] Implement `semaphore.nr` (Identity Commitment, Merkle Proof, Nullifier) <!-- id: 5 -->
    - [ ] Compile circuit and generate artifacts <!-- id: 6 -->
    - [ ] Generate Cairo Verifier using Garaga <!-- id: 7 -->

- [ ] **Phase 2: Smart Contracts (Cairo)** <!-- id: 8 -->
    - [ ] Implement `SemaphoreGroups` contract (Merkle Tree management) <!-- id: 9 -->
    - [ ] Implement `SemaphoreVerifier` contract (Proof verification) <!-- id: 10 -->
    - [ ] Implement `IMT` (Incremental Merkle Tree) library in Cairo <!-- id: 11 -->
    - [ ] Write integration tests for contracts (add/remove member, verify proof) <!-- id: 12 -->

- [ ] **Phase 3: Off-Chain SDK (TypeScript)** <!-- id: 13 -->
    - [ ] Implement Identity generation (EdDSA keypair, Poseidon commitment) <!-- id: 14 -->
    - [ ] Implement Merkle Proof generation (using `lean-imt` or similar) <!-- id: 15 -->
    - [ ] Implement Proof generation wrapper (Noir/backend interaction) <!-- id: 16 -->

- [ ] **Phase 4: Frontend & Demo** <!-- id: 17 -->
    - [ ] Scaffold Next.js application <!-- id: 18 -->
    - [ ] Integrate Starknet Wallet (Argent/Braavos) <!-- id: 19 -->
    - [ ] Build "Create Identity" UI <!-- id: 20 -->
    - [ ] Build "Join Group" UI <!-- id: 21 -->
    - [ ] Build "Signal/Vote" UI <!-- id: 22 -->

- [ ] **Phase 5: Final Polish** <!-- id: 23 -->
    - [ ] Deploy contracts to Starknet Sepolia <!-- id: 24 -->
    - [ ] Record Demo Video <!-- id: 25 -->
    - [ ] Complete Documentation (README, Architecture) <!-- id: 26 -->
