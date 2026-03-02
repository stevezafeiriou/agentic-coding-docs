# Security Patterns & Attack Vectors

# What to Know and What to Prevent

---

## The Top Attack Vectors (With Mitigations)

### 1. Reentrancy

The most famous DeFi attack vector. Attacker contract calls back into victim
before state is updated.

**Mitigation:** CEI pattern + nonReentrant modifier (both, always).
See 00-principles.md for the pattern.

### 2. Integer Overflow/Underflow

Pre-Solidity 0.8: arithmetic wrapped silently. uint256(0) - 1 = 2^256-1.

**Mitigation:** Solidity 0.8.x reverts automatically. Never use 0.7.x.
Never use unchecked{} without explicit proof it cannot overflow.

### 3. Access Control Failures

Missing or incorrectly implemented function guards. The most common bug.

```solidity
// MISTAKE: function is public but should be owner-only
function drainFunds() public {
    payable(msg.sender).transfer(address(this).balance);
}

// FIX: explicit access control + emit event
function emergencyWithdraw() external onlyOwner {
    uint256 bal = address(this).balance;
    (bool ok,) = owner().call{value: bal}("");
    require(ok, "Transfer failed");
    emit EmergencyWithdraw(owner(), bal);
}
```

### 4. Oracle Manipulation (Price Feeds)

Using on-chain DEX spot prices as oracles is manipulable via flash loans.

```solidity
// BAD — spot price can be manipulated in same block
uint256 price = getSpotPrice(tokenA, tokenB); // MANIPULABLE

// GOOD — use Chainlink price feeds with staleness check
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

AggregatorV3Interface internal priceFeed =
    AggregatorV3Interface(0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419); // ETH/USD mainnet

function getLatestPrice() public view returns (int256) {
    (
        uint80 roundId,
        int256 price,
        ,
        uint256 updatedAt,
        uint80 answeredInRound
    ) = priceFeed.latestRoundData();

    require(price > 0, "Negative price");
    require(updatedAt >= block.timestamp - 3600, "Stale price"); // 1hr max staleness
    require(answeredInRound >= roundId, "Stale round");

    return price;
}
```

### 5. Front-Running

Miners/validators can see pending transactions and insert their own first.
Affects: DEX swaps, NFT minting, auction bids.

```solidity
// Mitigation for NFT minting: commit-reveal scheme
bytes32 private _revealHash;

// User commits hash of (secretSalt + quantity) — nobody knows their intended mint amount
function commitMint(bytes32 commitment) external {
    commitments[msg.sender] = commitment;
    commitBlocks[msg.sender] = block.number;
}

// After MIN_COMMIT_BLOCKS, reveal and mint
function revealMint(uint256 quantity, bytes32 salt) external payable {
    require(block.number >= commitBlocks[msg.sender] + MIN_COMMIT_BLOCKS, "Too early");
    require(
        keccak256(abi.encodePacked(salt, quantity, msg.sender)) == commitments[msg.sender],
        "Invalid reveal"
    );
    delete commitments[msg.sender];
    _processMint(msg.sender, quantity);
}
```

### 6. Flash Loan Attacks

Attacker borrows huge amounts in one transaction, manipulates state, repays.

**Mitigation:**

- Never use spot prices from DEXs as oracles
- Use time-weighted average prices (TWAP) with minimum window
- Use Chainlink oracles for price feeds
- Require minimum lock periods for borrowed/staked assets

### 7. Signature Replay

A valid signature used multiple times or on the wrong chain.

```solidity
// Always include in signed data:
// - chainId (prevents cross-chain replay)
// - nonce (prevents same-chain replay)
// - contract address (prevents replay on different contracts)
// - expiry (prevents replay after time window)

// Use EIP-712 typed structured data (via OZ's EIP712 base)
import "@openzeppelin/contracts/utils/cryptography/EIP712.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";

contract WithSignatures is EIP712 {
    using ECDSA for bytes32;

    mapping(address => uint256) public nonces;

    bytes32 private constant PERMIT_TYPEHASH = keccak256(
        "Permit(address spender,uint256 amount,uint256 nonce,uint256 deadline)"
    );

    function verifyPermit(
        address spender,
        uint256 amount,
        uint256 deadline,
        bytes memory signature
    ) external {
        require(block.timestamp <= deadline, "Signature expired");

        bytes32 structHash = keccak256(abi.encode(
            PERMIT_TYPEHASH,
            spender,
            amount,
            nonces[msg.sender]++,  // Consume nonce
            deadline
        ));

        bytes32 hash = _hashTypedDataV4(structHash);
        address signer = hash.recover(signature);
        require(signer == msg.sender, "Invalid signature");
    }
}
```

### 8. Denial of Service via Gas

Pushing ETH to a contract that rejects it, or iterating over unbounded arrays.

```solidity
// BAD — recipient can cause permanent DoS by rejecting ETH
for (uint i = 0; i < recipients.length; i++) {
    payable(recipients[i]).transfer(amount); // One failure = all fail
}

// GOOD — pull pattern, each user claims independently
mapping(address => uint256) public claims;
function claim() external nonReentrant {
    uint256 amount = claims[msg.sender];
    require(amount > 0);
    claims[msg.sender] = 0;
    (bool ok,) = msg.sender.call{value: amount}("");
    require(ok);
}
```

### 9. Delegatecall to Untrusted Contracts

`delegatecall` runs external code in your contract's context.
It can read and write your storage.

```solidity
// NEVER delegatecall to user-supplied addresses
function execute(address target, bytes calldata data) external {
    (bool ok,) = target.delegatecall(data); // CATASTROPHICALLY DANGEROUS
}

// ONLY delegatecall to trusted, audited, immutable contracts
// Used in proxy patterns — requires extreme care
```

### 10. Improper Use of block.timestamp and blockhash

```solidity
// BAD — timestamp can be manipulated ±15 seconds by miners
// Do not use for precise timing, randomness, or lottery selection
if (block.timestamp % 2 == 0) { winner = player; } // MANIPULABLE

// BAD — blockhash only works for last 256 blocks, returns 0 otherwise
bytes32 rand = blockhash(block.number - 1); // MANIPULABLE and PREDICTABLE

// GOOD — use Chainlink VRF for verifiable randomness
// (see Chainlink VRF docs for full implementation)
// GOOD — use timestamp only for non-critical time windows (hours, not seconds)
require(block.timestamp >= saleStart, "Sale not started");  // Fine — hours range
```

---

## Proxy / Upgradeable Contract Patterns

Only use upgradeable contracts when upgradability is genuinely required.
They add significant complexity and attack surface.

```solidity
// If using OpenZeppelin's Transparent Proxy:
// 1. All storage must be declared in the implementation
// 2. Never rename or reorder existing storage variables
// 3. Always use Initializable + initializer modifier instead of constructor

import "@openzeppelin/contracts-upgradeable/token/ERC20/ERC20Upgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";

contract MyTokenV1 is
    Initializable,
    ERC20Upgradeable,
    OwnableUpgradeable,
    UUPSUpgradeable
{
    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() { _disableInitializers(); }

    function initialize(
        string memory name,
        string memory symbol,
        address initialOwner
    ) public initializer {
        __ERC20_init(name, symbol);
        __Ownable_init(initialOwner);
        __UUPSUpgradeable_init();
    }

    // Only owner can upgrade
    function _authorizeUpgrade(address newImpl) internal override onlyOwner {}
}
```

---

## Multisig Ownership (Production Requirement)

For any contract holding real value on mainnet, ownership must be a
multisig wallet (e.g., Gnosis Safe), never a single EOA.

```
Single EOA ownership on mainnet = single point of failure.
If private key is lost or compromised → funds are gone.

Use:
- Gnosis Safe (safe.global) — 3-of-5 or 2-of-3 multisig
- Transfer ownership AFTER deployment:
  await token.transferOwnership("0xYOUR_GNOSIS_SAFE_ADDRESS")
  // Then accept from Safe: token.acceptOwnership()
```

---

## Emergency Patterns

```solidity
// Every contract that holds value should have an emergency stop
// This is the Pausable pattern from OZ — already shown in templates

// For extreme emergencies: time-locked recovery
uint256 public recoveryTimestamp;
uint256 public constant RECOVERY_DELAY = 7 days;

function initiateRecovery() external onlyOwner {
    recoveryTimestamp = block.timestamp + RECOVERY_DELAY;
    emit RecoveryInitiated(recoveryTimestamp);
}

function executeRecovery(address recipient) external onlyOwner {
    require(block.timestamp >= recoveryTimestamp, "Delay not passed");
    require(paused(), "Must be paused");
    uint256 balance = address(this).balance;
    (bool ok,) = recipient.call{value: balance}("");
    require(ok, "Transfer failed");
}
```
