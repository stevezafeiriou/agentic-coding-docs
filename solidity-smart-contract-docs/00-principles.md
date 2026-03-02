# Smart Contract AI Agent — Core Principles & Security Checklist

# Ethereum + Solidity + Hardhat + Ethers.js v6

> ROOT FILE. Read before writing a single line of Solidity.
> Smart contracts are immutable once deployed. A vulnerability is permanent.
> Funds lost are gone forever. There is no patch on Monday.

---

## The Non-Negotiable Rules

### 1. Fixed Solidity Version — Never Floating Pragma

```solidity
// BAD  — compiles with unknown future versions
pragma solidity ^0.8.0;

// GOOD — pinned, deterministic
pragma solidity 0.8.24;
```

### 2. OpenZeppelin — Never Re-Implement Security Primitives

Always import audited, battle-tested OZ contracts. Never rewrite ownership,
reentrancy guards, ERC standards, or access control from scratch.

```bash
npm install @openzeppelin/contracts@5
```

| Use Case               | OZ Contract                                             |
| ---------------------- | ------------------------------------------------------- |
| Single owner admin     | `Ownable2Step` (two-step transfer — safer than Ownable) |
| Role-based permissions | `AccessControl`                                         |
| Reentrancy protection  | `ReentrancyGuard`                                       |
| Emergency stop         | `Pausable`                                              |
| Safe ERC20 transfers   | `SafeERC20`                                             |
| Token standards        | `ERC20`, `ERC721`, `ERC1155`                            |
| Upgrade proxy          | `TransparentUpgradeableProxy`                           |

### 3. Checks-Effects-Interactions (CEI) — Always

Every function sending ETH or calling external contracts MUST follow this order.
Violating CEI caused The DAO hack ($60M) and countless others since.

```solidity
function withdraw(uint256 amount) external nonReentrant {
    // 1. CHECKS — validate everything first
    require(balances[msg.sender] >= amount, "Insufficient balance");
    require(amount > 0, "Zero amount");

    // 2. EFFECTS — update ALL state before any external call
    balances[msg.sender] -= amount;
    totalDeposited -= amount;

    // 3. INTERACTIONS — external calls LAST
    (bool success, ) = msg.sender.call{value: amount}("");
    require(success, "ETH transfer failed");

    emit Withdrawn(msg.sender, amount);
}
```

### 4. Never Use tx.origin for Authentication

```solidity
require(tx.origin == owner);  // BAD — phishing vulnerable
require(msg.sender == owner); // GOOD
```

### 5. Pull Over Push for ETH Distribution

```solidity
// BAD — one revert blocks everyone
for (uint i = 0; i < recipients.length; i++) {
    payable(recipients[i]).transfer(amounts[i]);
}

// GOOD — each user claims their own
mapping(address => uint256) public pendingWithdrawals;
function claimReward() external nonReentrant {
    uint256 amt = pendingWithdrawals[msg.sender];
    require(amt > 0, "Nothing to claim");
    pendingWithdrawals[msg.sender] = 0;  // Effect before interaction
    (bool ok,) = msg.sender.call{value: amt}("");
    require(ok, "Transfer failed");
}
```

### 6. Every State Change Emits an Event

Events are the immutable audit log. Every write function must emit.

### 7. Validate All Inputs

```solidity
require(_token != address(0), "Zero address");
require(_price > 0, "Price must be positive");
require(_amount <= MAX_AMOUNT, "Exceeds maximum");
```

### 8. Solidity 0.8.x — No SafeMath Needed

Built-in overflow/underflow protection. Never add unchecked{} without
explicit, documented justification.

---

## Stack Reference

| Concern         | Tool                       | Notes                  |
| --------------- | -------------------------- | ---------------------- |
| Language        | Solidity 0.8.24            | Fixed version          |
| Framework       | Hardhat 2.22+              | TypeScript config      |
| Ethers          | ethers.js v6               | In tests + scripts     |
| Testing         | Hardhat + Chai             | 100% coverage required |
| Gas reports     | hardhat-gas-reporter       | Always enabled         |
| Coverage        | solidity-coverage          | Always enabled         |
| Type generation | typechain                  | Auto-generate from ABI |
| OZ Contracts    | @openzeppelin/contracts v5 | Always use             |
| Network         | Sepolia (testnet), Mainnet | EVM compatible         |

---

## Project Structure

```
my-contract/
├── contracts/
│   ├── MyContract.sol
│   └── interfaces/
│       └── IMyContract.sol
├── scripts/
│   ├── deploy.ts
│   └── verify.ts
├── test/
│   ├── MyContract.test.ts
│   └── helpers/
│       └── fixtures.ts
├── typechain-types/       # Auto-generated — never edit
├── artifacts/             # Compiled — never edit
├── .env                   # NEVER COMMIT
├── .env.example           # Always commit
├── hardhat.config.ts
├── package.json
└── tsconfig.json
```

---

## Pre-Deployment Security Checklist

### Code

- [ ] Fixed Solidity version (no `^`)
- [ ] All functions have explicit visibility
- [ ] All state-changing functions have access control
- [ ] All state-changing functions emit events
- [ ] No `tx.origin` usage
- [ ] No block.timestamp for critical logic (±15s miner manipulation)
- [ ] No blockhash for randomness
- [ ] No division before multiplication (precision loss)
- [ ] All external addresses validated (no zero address)

### Reentrancy

- [ ] CEI pattern on all ETH-sending / external-calling functions
- [ ] `nonReentrant` modifier applied
- [ ] State updated BEFORE external calls

### Access Control

- [ ] All admin functions protected
- [ ] Owner transfer is two-step (Ownable2Step)
- [ ] Pause mechanism exists

### Token Safety

- [ ] `SafeERC20.safeTransfer()` used (not raw `.transfer()`)
- [ ] Fee-on-transfer tokens considered if applicable

### Testing

- [ ] 100% function coverage
- [ ] All revert conditions tested
- [ ] All events tested
- [ ] Edge cases: zero values, max values, address(0)
- [ ] Reentrancy attack scenario tested
- [ ] Unauthorized access tested

### Deployment

- [ ] Deployed and tested on Sepolia first
- [ ] Constructor arguments correct
- [ ] Contract verified on Etherscan
- [ ] Ownership transferred to multisig (production)
