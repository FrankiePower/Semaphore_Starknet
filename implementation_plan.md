# Implementation Plan - Setup & ZK Circuits

## Goal Description
Initialize the "Semaphore on Starknet" project structure and implement the core Zero-Knowledge circuits using Noir. This sets the foundation for generating the Cairo verifier.

## Proposed Changes

### Project Structure
We will adopt a monorepo structure to keep everything organized:
```text
semaphore-starknet/
├── circuits/       # Noir circuits
├── contracts/      # Cairo smart contracts
├── web-app/        # Next.js frontend
└── sdk/            # TypeScript SDK logic (shared)
```

### [NEW] [circuits/](file:///Users/user/SuperFranky/semaphore_starknet/circuits)
- Initialize a new Noir project: `nargo new circuits`
- Implement `src/main.nr` (Semaphore logic)
- Configure `Nargo.toml`

### [NEW] [contracts/](file:///Users/user/SuperFranky/semaphore_starknet/contracts)
- Initialize a generic Scarb project: `scarb new contracts`
- Add dependencies: `starknet`, `garaga`, `openzeppelin`

## Verification Plan

### Automated Tests
- **Noir**: Run `nargo test` to verify circuit logic (Merkle membership, nullifier derivation).
- **Environment**: Verify `scarb --version`, `nargo --version`, and `garaga --version` output correctly.

### Manual Verification
- Compile circuits with `nargo compile` and ensure `target/` artifacts are generated.
