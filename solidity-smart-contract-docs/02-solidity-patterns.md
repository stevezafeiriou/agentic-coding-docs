# Production Solidity Patterns

# Secure, Gas-Optimized, Auditable Contract Templates

---

## Contract File Header (Always Use)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

/**
 * @title MyContract
 * @author Your Name / Team Name
 * @notice One sentence describing what this contract does for end users
 * @dev Technical notes for developers and auditors
 *
 * Security considerations:
 * - List known attack surfaces and mitigations
 * - List any assumptions this contract makes
 *
 * Audit history:
 * - [Date] Audited by [Firm] — report at [URL]
 */
```

---

## Complete ERC20 Token Template

A production-grade ERC20 with minting, burning, pausing, and permit support.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Pausable.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";
import "@openzeppelin/contracts/access/Ownable2Step.sol";

/**
 * @title MyToken
 * @notice ERC20 token with controlled minting, burning, and emergency pause
 * @dev Owner is the only minter. Two-step ownership transfer prevents accidents.
 */
contract MyToken is ERC20, ERC20Burnable, ERC20Pausable, ERC20Permit, Ownable2Step {

    // ============================================================
    // Constants
    // ============================================================

    /// @notice Maximum total supply — hard cap
    uint256 public constant MAX_SUPPLY = 1_000_000_000 * 1e18; // 1 billion tokens

    // ============================================================
    // Events
    // ============================================================

    /// @notice Emitted when tokens are minted
    event TokensMinted(address indexed to, uint256 amount);

    // ============================================================
    // Errors (custom errors save ~50 gas vs string requires)
    // ============================================================

    /// @notice Thrown when mint would exceed MAX_SUPPLY
    error ExceedsMaxSupply(uint256 requested, uint256 available);

    /// @notice Thrown when a zero-address is provided
    error ZeroAddress();

    /// @notice Thrown when a zero amount is provided
    error ZeroAmount();

    // ============================================================
    // Constructor
    // ============================================================

    /**
     * @param name_ Token name (e.g., "My Token")
     * @param symbol_ Token symbol (e.g., "MTK")
     * @param initialOwner Address that will own the contract and mint rights
     * @param initialSupply Tokens to mint to initialOwner at deploy (use 0 for none)
     */
    constructor(
        string memory name_,
        string memory symbol_,
        address initialOwner,
        uint256 initialSupply
    )
        ERC20(name_, symbol_)
        ERC20Permit(name_)
        Ownable2Step()
        Ownable(initialOwner)
    {
        if (initialOwner == address(0)) revert ZeroAddress();
        if (initialSupply > MAX_SUPPLY)
            revert ExceedsMaxSupply(initialSupply, MAX_SUPPLY);

        if (initialSupply > 0) {
            _mint(initialOwner, initialSupply);
            emit TokensMinted(initialOwner, initialSupply);
        }
    }

    // ============================================================
    // Owner Functions
    // ============================================================

    /**
     * @notice Mint new tokens — only owner
     * @param to Recipient address
     * @param amount Amount to mint (in wei, 18 decimals)
     */
    function mint(address to, uint256 amount) external onlyOwner {
        if (to == address(0)) revert ZeroAddress();
        if (amount == 0) revert ZeroAmount();

        uint256 available = MAX_SUPPLY - totalSupply();
        if (amount > available) revert ExceedsMaxSupply(amount, available);

        _mint(to, amount);
        emit TokensMinted(to, amount);
    }

    /**
     * @notice Pause all token transfers — emergency use only
     */
    function pause() external onlyOwner {
        _pause();
    }

    /**
     * @notice Resume token transfers
     */
    function unpause() external onlyOwner {
        _unpause();
    }

    // ============================================================
    // Required Overrides
    // ============================================================

    function _update(
        address from,
        address to,
        uint256 value
    ) internal override(ERC20, ERC20Pausable) {
        super._update(from, to, value);
    }
}
```

---

## ETH Vault / Escrow Template

A secure ETH deposit/withdrawal contract demonstrating CEI and reentrancy guard.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable2Step.sol";
import "@openzeppelin/contracts/utils/Pausable.sol";

/**
 * @title ETHVault
 * @notice Secure ETH deposit and withdrawal vault
 * @dev Demonstrates CEI pattern and reentrancy protection
 *
 * Security:
 * - ReentrancyGuard on all ETH-sending functions
 * - CEI pattern strictly followed
 * - Pull pattern for withdrawals
 * - Owner can pause in emergencies
 */
contract ETHVault is ReentrancyGuard, Ownable2Step, Pausable {

    // ============================================================
    // State
    // ============================================================

    /// @notice ETH balance per user
    mapping(address => uint256) public balances;

    /// @notice Total ETH held in vault
    uint256 public totalDeposited;

    /// @notice Maximum deposit per address
    uint256 public constant MAX_DEPOSIT = 100 ether;

    // ============================================================
    // Events
    // ============================================================

    event Deposited(address indexed user, uint256 amount, uint256 newBalance);
    event Withdrawn(address indexed user, uint256 amount, uint256 newBalance);
    event EmergencyWithdraw(address indexed owner, uint256 amount);

    // ============================================================
    // Errors
    // ============================================================

    error ZeroAmount();
    error InsufficientBalance(uint256 requested, uint256 available);
    error ExceedsMaxDeposit(uint256 total, uint256 max);
    error TransferFailed();

    // ============================================================
    // Constructor
    // ============================================================

    constructor(address initialOwner) Ownable(initialOwner) Ownable2Step() {}

    // ============================================================
    // User Functions
    // ============================================================

    /**
     * @notice Deposit ETH into the vault
     * @dev Emits Deposited event
     */
    function deposit() external payable whenNotPaused {
        if (msg.value == 0) revert ZeroAmount();

        uint256 newBalance = balances[msg.sender] + msg.value;
        if (newBalance > MAX_DEPOSIT)
            revert ExceedsMaxDeposit(newBalance, MAX_DEPOSIT);

        // EFFECTS before INTERACTIONS (no external calls here, but pattern maintained)
        balances[msg.sender] = newBalance;
        totalDeposited += msg.value;

        emit Deposited(msg.sender, msg.value, newBalance);
    }

    /**
     * @notice Withdraw ETH from the vault
     * @param amount Amount to withdraw in wei
     * @dev CEI pattern + nonReentrant guard
     */
    function withdraw(uint256 amount) external nonReentrant whenNotPaused {
        // CHECKS
        if (amount == 0) revert ZeroAmount();
        uint256 userBalance = balances[msg.sender];
        if (userBalance < amount)
            revert InsufficientBalance(amount, userBalance);

        // EFFECTS — update state BEFORE external call
        balances[msg.sender] = userBalance - amount;
        totalDeposited -= amount;

        // INTERACTIONS — ETH transfer LAST
        (bool success, ) = msg.sender.call{value: amount}("");
        if (!success) revert TransferFailed();

        emit Withdrawn(msg.sender, amount, balances[msg.sender]);
    }

    /**
     * @notice Get the ETH balance of a user
     */
    function getBalance(address user) external view returns (uint256) {
        return balances[user];
    }

    // ============================================================
    // Owner Functions
    // ============================================================

    function pause() external onlyOwner { _pause(); }
    function unpause() external onlyOwner { _unpause(); }

    /**
     * @notice Emergency withdraw all ETH — only when paused
     * @dev For catastrophic situations only
     */
    function emergencyWithdraw() external onlyOwner {
        require(paused(), "Must be paused first");
        uint256 balance = address(this).balance;
        totalDeposited = 0;
        (bool success, ) = owner().call{value: balance}("");
        if (!success) revert TransferFailed();
        emit EmergencyWithdraw(owner(), balance);
    }

    // ============================================================
    // Receive / Fallback
    // ============================================================

    /// @notice Direct ETH transfers go to deposit()
    receive() external payable {
        // Route direct ETH sends through deposit logic
        if (msg.value == 0) revert ZeroAmount();
        balances[msg.sender] += msg.value;
        totalDeposited += msg.value;
        emit Deposited(msg.sender, msg.value, balances[msg.sender]);
    }
}
```

---

## NFT (ERC721) Template

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Pausable.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Burnable.sol";
import "@openzeppelin/contracts/access/Ownable2Step.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/utils/Strings.sol";

/**
 * @title MyNFT
 * @notice ERC721 NFT collection with whitelist presale and public sale
 * @dev Mint price in ETH, per-wallet limits, withdrawal pattern
 */
contract MyNFT is
    ERC721,
    ERC721URIStorage,
    ERC721Pausable,
    ERC721Burnable,
    Ownable2Step,
    ReentrancyGuard
{
    using Strings for uint256;

    // ============================================================
    // Constants
    // ============================================================

    uint256 public constant MAX_SUPPLY       = 10_000;
    uint256 public constant MAX_PER_WALLET   = 5;
    uint256 public constant MINT_PRICE       = 0.08 ether;
    uint256 public constant WHITELIST_PRICE  = 0.06 ether;

    // ============================================================
    // State
    // ============================================================

    uint256 private _nextTokenId;
    string  private _baseTokenURI;

    enum SalePhase { CLOSED, WHITELIST, PUBLIC }
    SalePhase public salePhase;

    mapping(address => bool)    public whitelist;
    mapping(address => uint256) public mintedPerWallet;
    mapping(address => uint256) public pendingWithdrawals; // For fund distribution

    // ============================================================
    // Events
    // ============================================================

    event Minted(address indexed to, uint256 indexed tokenId);
    event SalePhaseChanged(SalePhase newPhase);
    event WhitelistUpdated(address[] addresses, bool status);
    event FundsWithdrawn(address indexed to, uint256 amount);
    event BaseURIUpdated(string newURI);

    // ============================================================
    // Errors
    // ============================================================

    error SaleClosed();
    error NotWhitelisted();
    error MaxSupplyReached();
    error MaxPerWalletReached();
    error InsufficientPayment(uint256 sent, uint256 required);
    error ZeroAddress();
    error TransferFailed();
    error NothingToWithdraw();

    // ============================================================
    // Constructor
    // ============================================================

    constructor(
        string memory name_,
        string memory symbol_,
        string memory baseURI_,
        address initialOwner
    ) ERC721(name_, symbol_) Ownable(initialOwner) Ownable2Step() {
        if (initialOwner == address(0)) revert ZeroAddress();
        _baseTokenURI = baseURI_;
        salePhase = SalePhase.CLOSED;
    }

    // ============================================================
    // Minting
    // ============================================================

    /**
     * @notice Mint NFTs
     * @param quantity Number of NFTs to mint (1-5)
     */
    function mint(uint256 quantity) external payable nonReentrant whenNotPaused {
        // CHECKS
        if (salePhase == SalePhase.CLOSED) revert SaleClosed();

        uint256 price;
        if (salePhase == SalePhase.WHITELIST) {
            if (!whitelist[msg.sender]) revert NotWhitelisted();
            price = WHITELIST_PRICE;
        } else {
            price = MINT_PRICE;
        }

        if (_nextTokenId + quantity > MAX_SUPPLY) revert MaxSupplyReached();

        uint256 alreadyMinted = mintedPerWallet[msg.sender];
        if (alreadyMinted + quantity > MAX_PER_WALLET) revert MaxPerWalletReached();

        uint256 totalCost = price * quantity;
        if (msg.value < totalCost)
            revert InsufficientPayment(msg.value, totalCost);

        // EFFECTS
        mintedPerWallet[msg.sender] = alreadyMinted + quantity;

        uint256 startId = _nextTokenId;
        _nextTokenId += quantity;

        // Refund excess payment
        if (msg.value > totalCost) {
            pendingWithdrawals[msg.sender] += msg.value - totalCost;
        }

        // INTERACTIONS
        for (uint256 i = 0; i < quantity; ) {
            uint256 tokenId = startId + i;
            _safeMint(msg.sender, tokenId);
            emit Minted(msg.sender, tokenId);
            unchecked { ++i; } // Safe: bounded by quantity check above
        }
    }

    /**
     * @notice Claim overpaid ETH refund
     */
    function claimRefund() external nonReentrant {
        uint256 amount = pendingWithdrawals[msg.sender];
        if (amount == 0) revert NothingToWithdraw();
        pendingWithdrawals[msg.sender] = 0;
        (bool ok, ) = msg.sender.call{value: amount}("");
        if (!ok) revert TransferFailed();
    }

    /**
     * @notice Total NFTs minted so far
     */
    function totalSupply() external view returns (uint256) {
        return _nextTokenId;
    }

    // ============================================================
    // Owner Functions
    // ============================================================

    function setSalePhase(SalePhase phase) external onlyOwner {
        salePhase = phase;
        emit SalePhaseChanged(phase);
    }

    function setWhitelist(address[] calldata addresses, bool status) external onlyOwner {
        for (uint256 i = 0; i < addresses.length; ) {
            if (addresses[i] == address(0)) revert ZeroAddress();
            whitelist[addresses[i]] = status;
            unchecked { ++i; }
        }
        emit WhitelistUpdated(addresses, status);
    }

    function setBaseURI(string calldata newURI) external onlyOwner {
        _baseTokenURI = newURI;
        emit BaseURIUpdated(newURI);
    }

    /**
     * @notice Owner mints for team/giveaways — no payment required
     */
    function ownerMint(address to, uint256 quantity) external onlyOwner {
        if (to == address(0)) revert ZeroAddress();
        if (_nextTokenId + quantity > MAX_SUPPLY) revert MaxSupplyReached();

        uint256 startId = _nextTokenId;
        _nextTokenId += quantity;

        for (uint256 i = 0; i < quantity; ) {
            uint256 tokenId = startId + i;
            _safeMint(to, tokenId);
            emit Minted(to, tokenId);
            unchecked { ++i; }
        }
    }

    /**
     * @notice Withdraw contract ETH balance to owner
     */
    function withdrawFunds() external onlyOwner nonReentrant {
        uint256 balance = address(this).balance;
        if (balance == 0) revert NothingToWithdraw();
        (bool ok, ) = owner().call{value: balance}("");
        if (!ok) revert TransferFailed();
        emit FundsWithdrawn(owner(), balance);
    }

    function pause()   external onlyOwner { _pause(); }
    function unpause() external onlyOwner { _unpause(); }

    // ============================================================
    // Required Overrides
    // ============================================================

    function tokenURI(uint256 tokenId)
        public view override(ERC721, ERC721URIStorage) returns (string memory)
    {
        _requireOwned(tokenId);
        return string.concat(_baseTokenURI, tokenId.toString(), ".json");
    }

    function supportsInterface(bytes4 interfaceId)
        public view override(ERC721, ERC721URIStorage) returns (bool)
    {
        return super.supportsInterface(interfaceId);
    }

    function _update(address to, uint256 tokenId, address auth)
        internal override(ERC721, ERC721Pausable) returns (address)
    {
        return super._update(to, tokenId, auth);
    }

    function _baseURI() internal view override returns (string memory) {
        return _baseTokenURI;
    }
}
```

---

## Interface Pattern (Always Define)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

/**
 * @title IMyContract
 * @notice Public interface for MyContract — defines all external functions
 * @dev Interfaces enable other contracts to interact without importing the full contract
 */
interface IMyContract {
    // Events (interfaces can declare events for documentation)
    event Deposited(address indexed user, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);

    // Errors
    error ZeroAmount();
    error InsufficientBalance();

    // Functions
    function deposit() external payable;
    function withdraw(uint256 amount) external;
    function getBalance(address user) external view returns (uint256);
    function totalDeposited() external view returns (uint256);
}
```

---

## Custom Errors vs require() — Always Use Custom Errors

Custom errors save gas (~50 gas per revert) and provide structured error data.

```solidity
// OLD — string require (more expensive, less info)
require(amount > 0, "Amount must be greater than zero");
require(msg.sender == owner, "Caller is not the owner");

// NEW — custom errors (preferred, Solidity 0.8.4+)
error ZeroAmount();
error Unauthorized(address caller, address required);

if (amount == 0) revert ZeroAmount();
if (msg.sender != owner) revert Unauthorized(msg.sender, owner);
```

---

## Mapping + Array Pattern (Safe Iteration)

```solidity
// Safe pattern for enumerable sets
mapping(address => bool)  private _isStaker;
address[]                 private _stakers;
mapping(address => uint256) private _stakerIndex; // index+1, 0 = not in array

function _addStaker(address staker) internal {
    if (!_isStaker[staker]) {
        _isStaker[staker] = true;
        _stakerIndex[staker] = _stakers.length + 1;
        _stakers.push(staker);
    }
}

function _removeStaker(address staker) internal {
    if (_isStaker[staker]) {
        uint256 index = _stakerIndex[staker] - 1;
        address last  = _stakers[_stakers.length - 1];
        _stakers[index]    = last;
        _stakerIndex[last] = index + 1;
        _stakers.pop();
        delete _isStaker[staker];
        delete _stakerIndex[staker];
    }
}

// NEVER iterate over unbounded arrays in transactions
// If you must, add pagination:
function getStakers(uint256 offset, uint256 limit)
    external view returns (address[] memory)
{
    uint256 end = offset + limit;
    if (end > _stakers.length) end = _stakers.length;
    address[] memory page = new address[](end - offset);
    for (uint256 i = offset; i < end; ) {
        page[i - offset] = _stakers[i];
        unchecked { ++i; }
    }
    return page;
}
```

---

## Gas Optimization Patterns

```solidity
// 1. Use uint256 everywhere (EVM word size — cheaper than uint8/uint128)
uint256 public counter;       // GOOD
uint8   public counter;       // BAD — requires masking operations

// 2. Pack structs to save storage slots (one slot = 32 bytes)
// BAD — 3 storage slots
struct Bad {
    uint256 a;  // slot 0
    bool    b;  // slot 1 (wastes 31 bytes)
    uint256 c;  // slot 2
}

// GOOD — 2 storage slots
struct Good {
    uint256 a;  // slot 0
    uint256 c;  // slot 1
    bool    b;  // packed into slot 1 with c (if c is smaller)
}
// Better with address + bool:
struct Packed {
    address user;  // 20 bytes
    bool    active;// 1 byte  — packed into same 32-byte slot as address
    uint96  amount;// 12 bytes — also fits in same slot!
}

// 3. Cache storage reads in memory for loops
// BAD — reads from storage on every iteration
for (uint i = 0; i < array.length; i++) { ... }

// GOOD — read storage once, use memory variable
uint256 len = array.length;
for (uint256 i = 0; i < len; ) {
    unchecked { ++i; } // Save gas: post-increment uses more gas
}

// 4. Use events instead of storage for historical data
// Don't store history in arrays — emit events instead
// Events cost ~375 gas; SSTORE costs 20,000 gas for new value
```
