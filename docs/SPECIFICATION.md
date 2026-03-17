# Specification

> Instruction interfaces, parameters, validations, and error conditions.
> This is the contract between the on-chain program and its clients.

## Overview

<!-- Brief description of what the protocol does. -->

## Instructions

<!-- Define each instruction with this template:

### instruction_name

**Purpose**: What this instruction does.

**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| amount | u64 | Amount in lamports |

**Accounts:**
| Account | Type | Mutable | Signer | Description |
|---------|------|---------|--------|-------------|
| user | SystemAccount | No | Yes | Transaction payer |
| my_state | MyState | Yes | No | PDA: ["seed", key] |
| system_program | Program | No | No | System Program |

**Validations:**
- `amount > 0` → InvalidAmount
- `user == state.authority` → Unauthorized

**Logic:**
- Step 1: ...
- Step 2: ...

**Errors:**
- `InvalidAmount` — amount is zero
- `Unauthorized` — caller is not the authority

-->

## Error Codes

<!-- Keep in sync with programs/protocol/src/error.rs

| Code | Name | Description |
|------|------|-------------|
| 6000 | MathOverflow | Arithmetic overflow |
| 6001 | MathError | Division by zero or invalid operation |
| 6002 | MathUnderflow | Arithmetic underflow |

-->

## Constants

<!-- Keep in sync with programs/protocol/src/constants.rs

| Constant | Value | Description |
|----------|-------|-------------|

-->
