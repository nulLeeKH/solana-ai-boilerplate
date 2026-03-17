# Architecture

> System design, account model, and module dependencies.
> Keep this document updated as the program evolves.

## System Overview

<!-- Describe the high-level system: what the protocol does, who interacts with it. -->

## Account Model

<!-- Replace this example with your protocol's account model -->

```mermaid
graph TD
    U["User (Signer)"]
    U --> UA["UserAccount<br/>PDA: [user, user_pubkey]<br/>stores: balance, history"]
    U --> VA["VaultAccount<br/>PDA: [vault, pool_id]<br/>stores: token_a, token_b"]
```

<!-- Edit the diagram above to match your protocol's accounts -->

## PDA Seed Design

<!-- Document ALL PDAs here. This is critical for consistency.

| PDA Name | Seeds | Bump Stored? | Purpose |
|----------|-------|-------------|---------|
| Example  | ["example", key] | Yes (in ExampleAccount.bump) | Stores example data |
-->

## Module Dependencies

```mermaid
graph TD
    LIB["lib.rs"] --> CONST["constants.rs<br/>PDA seeds, protocol parameters"]
    LIB --> ERR["error.rs<br/>All error codes"]
    LIB --> STATE["state/<br/>Account structures"]
    LIB --> MATH["math/<br/>Business logic, pricing"]
    LIB --> IX["instructions/<br/>One file per instruction"]
    MATH --> WAD["wad.rs<br/>Fixed-point arithmetic (WAD)"]
    IX --> CONST
    IX --> ERR
    IX --> STATE
    IX --> MATH
```

## Data Flow

<!-- Describe how data flows through the system.
Example: User → Frontend → SDK → On-chain Program → State Update
-->

## Security Considerations

<!-- Document security-critical design decisions and their rationale. -->
