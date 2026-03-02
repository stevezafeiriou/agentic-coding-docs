# Developer Instruction Manual

# Complete Step-by-Step Guide for Non-Smart-Contract Developers

# How to Deploy, Verify, and Manage Your Ethereum Smart Contract

---

## Who This Guide Is For

This manual assumes you understand JavaScript/TypeScript and can use a
terminal, but have no prior experience with smart contracts, MetaMask,
or Ethereum. Every step is explained in plain language.

---

## SECTION 1: What a Smart Contract Actually Is

A smart contract is a program that lives on the Ethereum blockchain.
Unlike a regular server program you can update anytime, a smart contract
is:

- **Immutable** — once deployed, the code cannot be changed (unless you
  built in an upgrade mechanism)
- **Public** — anyone can read the code and all transactions
- **Autonomous** — it runs exactly as written, no human can intervene
- **Trustless** — users interact with it directly, no middleman

This means bugs cost real money and cannot be patched. This is why the
testing and security steps in this guide are mandatory, not optional.

---

## SECTION 2: Setting Up MetaMask

MetaMask is a browser extension that acts as your Ethereum wallet.
It stores your private keys and signs transactions.

### Step 1: Install MetaMask

1. Go to https://metamask.io
2. Click "Download" and install the browser extension
3. Click "Create a new wallet"
4. Choose a strong password (used to unlock MetaMask on your device)
5. **Write down your 12-word Secret Recovery Phrase on paper**
   - This phrase IS your wallet. Anyone with it can steal all your funds.
   - Never store it digitally. Never take a photo of it.
   - Never type it into any website.

### Step 2: Switch to Sepolia Testnet

Before deploying with real money, you MUST test on Sepolia (a test network
where ETH has no value).

1. Click the network dropdown at the top of MetaMask (shows "Ethereum Mainnet")
2. Click "Show test networks" if Sepolia is not visible
3. Select "Sepolia test network"

### Step 3: Get Free Sepolia ETH (Testnet Faucets)

You need ETH to pay for deployment. On testnets, ETH is free.

1. Go to https://sepoliafaucet.com or https://faucet.sepolia.dev
2. Paste your MetaMask address (click the address in MetaMask to copy)
3. Request ETH — you'll receive 0.5-1 testnet ETH within minutes

---

## SECTION 3: Project Setup

### Step 1: Prerequisites

Install these if you haven't already:

```bash
node --version   # Must be v18 or higher
npm --version    # Should be v9+
git --version    # Should be installed
```

If Node is not installed: https://nodejs.org (download LTS version)

### Step 2: Create Project

```bash
mkdir my-contract
cd my-contract
npm init -y
npm install --save-dev hardhat
npx hardhat init
# Select: "Create a TypeScript project"
# Press enter for all defaults
```

### Step 3: Install Dependencies

```bash
npm install --save-dev \
  @nomicfoundation/hardhat-toolbox \
  @nomicfoundation/hardhat-ethers \
  @nomicfoundation/hardhat-chai-matchers \
  @nomicfoundation/hardhat-network-helpers \
  hardhat-gas-reporter \
  solidity-coverage \
  ts-node \
  typechain \
  @typechain/hardhat \
  @typechain/ethers-v6

npm install \
  @openzeppelin/contracts \
  ethers \
  dotenv
```

### Step 4: Create Your .env File

```bash
# In your project root, create a file named exactly: .env
# Copy the contents of .env.example and fill in your values
```

**Getting your Private Key from MetaMask:**

1. Open MetaMask
2. Click the three dots (⋮) next to your account name
3. Click "Account details"
4. Click "Show private key"
5. Enter your MetaMask password
6. Copy the private key (starts with 0x — remove the 0x when pasting to .env)

⚠️ **CRITICAL:** Create a separate "deployer" wallet for this. Never use
your main personal wallet as a deployer. If the private key is ever
accidentally committed to git, a new empty wallet has nothing to steal.

**Getting an Alchemy RPC URL (free):**

1. Go to https://alchemy.com and create a free account
2. Click "Create App"
3. Select Network: "Ethereum Sepolia"
4. Copy the HTTPS URL — this is your SEPOLIA_RPC_URL

**Getting an Etherscan API Key (free):**

1. Go to https://etherscan.io and create a free account
2. Click your profile → "API Keys"
3. Create a new API key
4. Copy it to ETHERSCAN_API_KEY in your .env

---

## SECTION 4: Understanding the Contract Code

### What Each Part of Your ERC20 Token Does

```
MyToken Contract — What Every Piece Means
══════════════════════════════════════════

CONTRACT SETUP
─────────────
• pragma solidity 0.8.24
  The version of Solidity (the programming language) used.
  Fixed to prevent future compiler changes from breaking it.

• import "@openzeppelin/contracts/..."
  Imports pre-audited code from OpenZeppelin, the industry standard
  library for Ethereum contracts. Like npm packages for Solidity.

• Ownable2Step
  Makes you the "owner" of the contract. The owner can mint tokens,
  pause transfers, etc. "2Step" means transferring ownership requires
  confirmation from the new owner — prevents accidentally sending
  ownership to the wrong address.

• ReentrancyGuard
  A security lock that prevents a specific attack where a contract
  repeatedly calls back into your contract before it finishes.

STATE VARIABLES
───────────────
• MAX_SUPPLY = 1,000,000,000 tokens
  The absolute maximum tokens that can ever exist. Even the owner
  cannot create more than this. It's hardcoded — permanent.

• balances, allowances
  These are stored in the ERC20 base contract from OpenZeppelin.
  They track who owns how many tokens.

FUNCTIONS
─────────
• mint(address to, uint256 amount)
  Creates new tokens and sends them to `to`.
  Only the contract owner can call this.
  Cannot mint beyond MAX_SUPPLY.

• burn(uint256 amount)
  Destroys tokens permanently from the caller's balance.
  Anyone can burn their own tokens.

• transfer(address to, uint256 amount)
  Sends tokens from the caller to `to`.
  Standard ERC20 — works with MetaMask, Uniswap, etc.

• approve(address spender, uint256 amount)
  Lets `spender` use up to `amount` of the caller's tokens.
  Required before any dApp can spend tokens on your behalf.

• transferFrom(address from, address to, uint256 amount)
  Moves tokens from `from` to `to` using an existing approval.
  Used by dApps and DEXes after you've approved them.

• pause() / unpause()
  Emergency stop. Pause blocks ALL token transfers.
  Only owner can call. Use if you discover a vulnerability.

• transferOwnership(address newOwner)
  Starts the process of giving ownership to a new address.
  The new owner must then call acceptOwnership() to complete it.
  This two-step process prevents typos locking you out.
```

### What Each Part of Your NFT Contract Does

```
MyNFT Contract — What Every Piece Means
════════════════════════════════════════

MINT PHASES
───────────
• CLOSED (Phase 0)
  No one can mint. Default state at deployment.

• WHITELIST (Phase 1)
  Only addresses you've approved can mint.
  They pay WHITELIST_PRICE (0.06 ETH per token).
  Owner sets whitelist addresses with setWhitelist().

• PUBLIC (Phase 2)
  Anyone can mint.
  They pay MINT_PRICE (0.08 ETH per token).

LIMITS
──────
• MAX_SUPPLY = 10,000
  Absolute cap. Once 10,000 are minted, no more can ever exist.

• MAX_PER_WALLET = 5
  Each address can mint at most 5 tokens total across all phases.

FUNCTIONS
─────────
• mint(uint256 quantity)
  Mints `quantity` NFTs to the caller.
  Requires correct ETH payment.
  Subject to phase, supply, and per-wallet limits.

• setSalePhase(SalePhase phase)
  Owner only. Changes the current sale phase:
  0 = CLOSED, 1 = WHITELIST, 2 = PUBLIC

• setWhitelist(address[] calldata addresses, bool status)
  Owner only. Add or remove addresses from the whitelist.
  Pass true to add, false to remove.
  Example: setWhitelist(["0xAlice", "0xBob"], true)

• setBaseURI(string calldata newURI)
  Owner only. Updates the metadata URL.
  NFT metadata is stored off-chain (IPFS).
  This is the base URL that tokenURI() appends the token ID to.
  Format: "ipfs://YOUR_CID/"
  tokenURI(0) returns "ipfs://YOUR_CID/0.json"

• ownerMint(address to, uint256 quantity)
  Owner only. Free minting for team, giveaways, treasury.
  Counts toward MAX_SUPPLY but not per-wallet limits.

• withdrawFunds()
  Owner only. Withdraws all ETH from mint sales to the owner's wallet.

• tokenURI(uint256 tokenId)
  Returns the metadata URL for a specific token.
  MetaMask, OpenSea, and other apps call this to display your NFT.

TOKEN URI / METADATA
────────────────────
Each NFT points to a JSON file on IPFS. The file looks like:
{
  "name": "My NFT #0",
  "description": "A description of the collection",
  "image": "ipfs://QmImageCID/0.png",
  "attributes": [
    { "trait_type": "Background", "value": "Blue" }
  ]
}

You must upload:
1. All images to IPFS (one per token: 0.png, 1.png, ...)
2. All metadata JSON files to IPFS (one per token: 0.json, 1.json, ...)
3. The CID of the metadata folder is your BASE_URI
```

---

## SECTION 5: Running Tests

Always run tests before deploying anything. Tests run locally and cost nothing.

```bash
# Compile contracts (checks for Solidity errors)
npm run compile

# Run all tests
npm test

# Run tests with gas report (shows how much each function costs)
npm run test:gas

# Run tests with coverage report
npm run test:coverage
# Then open: coverage/index.html in your browser
# You should see 100% on all metrics before deploying
```

### Understanding Test Output

```
MyToken
  Deployment
    ✓ should deploy with correct name and symbol (45ms)
    ✓ should mint initial supply to owner
    ✓ should set correct owner
  mint()
    ✓ should allow owner to mint tokens
    ✓ should emit TokensMinted event
    ✗ should revert if called by non-owner   ← RED = PROBLEM

✓ = test passed (expected behavior confirmed)
✗ = test failed (something is wrong — do NOT deploy)
```

---

## SECTION 6: Deploying to Sepolia Testnet

### Pre-Deployment Checklist

- [ ] All tests pass (`npm test` shows all green)
- [ ] Coverage is 100% (`npm run test:coverage`)
- [ ] .env file has PRIVATE_KEY, SEPOLIA_RPC_URL, ETHERSCAN_API_KEY
- [ ] Deployer wallet has Sepolia ETH (at least 0.1 for gas)
- [ ] Constructor arguments in deploy script are correct

### Deploy

```bash
npm run deploy:sepolia
```

Expected output:

```
============================================================
Deploying MyToken
Network: sepolia
============================================================
Deployer address: 0xYourDeployerAddress
Deployer ETH balance: 0.5 ETH

Constructor arguments:
  Name: My Token
  Symbol: MTK
  Initial Owner: 0xYourDeployerAddress
  Initial Supply: 100000.0 tokens

Gas estimate: 1234567
Estimated cost: 0.002345 ETH

Deploying...
Transaction hash: 0xabc123...
Waiting for confirmation...

✅ Deployed successfully!
Contract address: 0xYOUR_CONTRACT_ADDRESS

Verifying on Etherscan...
✅ Contract verified on Etherscan

Contract: 0xYOUR_CONTRACT_ADDRESS
Etherscan: https://sepolia.etherscan.io/address/0xYOUR_CONTRACT_ADDRESS
```

---

## SECTION 7: Verifying Your Contract on Etherscan

Contract verification makes your source code public and readable on Etherscan.
Without it, users see only bytecode — they can't trust what your contract does.
Verification is free and should always be done.

Verification happens automatically in the deploy script. If it fails:

```bash
# Manual verification — replace with your actual values
npx hardhat verify --network sepolia \
  0xYOUR_CONTRACT_ADDRESS \
  "My Token" \
  "MTK" \
  "0xYOUR_OWNER_ADDRESS" \
  "100000000000000000000000"
# Note: token amounts are in wei (multiply by 10^18)
# 100,000 tokens = 100000 * 10^18 = 100000000000000000000000
```

### After Verification — What You'll See on Etherscan

Go to your contract address on Etherscan:
https://sepolia.etherscan.io/address/0xYOUR_CONTRACT_ADDRESS

**Tab: Contract**

- Shows your full Solidity source code (public — users can audit it)
- Has a green checkmark ✓ showing it's verified
- Shows compiler settings used

**Tab: Read Contract**

- Lists all view functions (name, symbol, totalSupply, balanceOf, etc.)
- You can call these directly on Etherscan — no wallet needed

**Tab: Write Contract**

- Lists all state-changing functions (mint, pause, transferOwnership, etc.)
- Click "Connect to Web3" to connect MetaMask
- You can call owner functions directly here — like a UI for your contract

---

## SECTION 8: Interacting with Your Contract

### Via Etherscan (Easiest — No Code)

1. Go to your contract on Etherscan
2. Click "Contract" → "Write Contract"
3. Click "Connect to Web3"
4. Connect MetaMask (make sure you're using the owner wallet)
5. Find the function you want to call
6. Fill in the parameters and click "Write"
7. Confirm the transaction in MetaMask

### Via Hardhat Console (Powerful — Code)

```bash
# Start interactive console connected to Sepolia
npx hardhat console --network sepolia

# In the console:
const [deployer] = await ethers.getSigners();
const token = await ethers.getContractAt("MyToken", "0xYOUR_CONTRACT_ADDRESS");

// Read functions (free — no gas)
await token.name()           // "My Token"
await token.totalSupply()    // BigInt
await token.balanceOf(deployer.address)

// Write functions (cost gas — need ETH)
await token.mint("0xRecipientAddress", ethers.parseEther("1000"))
await token.pause()
await token.unpause()
```

### Via ethers.js Script

```typescript
// scripts/interact.ts
import { ethers } from "hardhat";

async function main() {
	const [deployer] = await ethers.getSigners();

	// Get reference to deployed contract
	const token = await ethers.getContractAt(
		"MyToken",
		"0xYOUR_CONTRACT_ADDRESS",
	);

	// Mint 1000 tokens to an address
	console.log("Minting 1000 tokens...");
	const tx = await token.mint("0xRecipientAddress", ethers.parseEther("1000"));
	await tx.wait(); // Wait for transaction to confirm
	console.log("Minted! TX:", tx.hash);

	// Check balance
	const balance = await token.balanceOf("0xRecipientAddress");
	console.log("Balance:", ethers.formatEther(balance), "MTK");
}

main().catch(console.error);
```

```bash
npx hardhat run scripts/interact.ts --network sepolia
```

---

## SECTION 9: Adding Your Token to MetaMask

After deploying, users need to add the token to their MetaMask to see it.

1. Open MetaMask
2. Scroll down and click "Import tokens"
3. Paste your contract address
4. MetaMask will auto-fill Name and Symbol from the contract
5. Click "Add custom token" → "Import tokens"

The token will now appear in MetaMask with its balance.

---

## SECTION 10: Understanding Gas & Transaction Costs

Every operation on Ethereum costs "gas" — a fee paid in ETH to validators.

```
Gas concepts:
─────────────
Gas Units:     How much computation a transaction uses
               (deploying a contract: ~1-3 million units)
               (calling a simple function: ~20,000-100,000 units)

Gas Price:     How much ETH you pay per unit of gas
               (set by market conditions — check etherscan.io/gastracker)

Total Cost:    Gas Units × Gas Price
               Example: 200,000 units × 20 gwei = 4,000,000 gwei = 0.004 ETH

Gwei:          1 billion gwei = 1 ETH
               Prices are quoted in gwei (e.g., "20 gwei per unit")
```

### Estimating Real Deployment Costs

```bash
# Run with gas reporter to see real costs
REPORT_GAS=true npm test

# Output example:
# ·················|·············|··············|
# |  Contract      |  Method     |  Gas Used    |
# ·················|·············|··············|
# |  MyToken       |  deploy     |  1,234,567   |
# |  MyToken       |  mint       |  67,890      |
# |  MyToken       |  pause      |  28,432      |
# ·················|·············|··············|

# At 30 gwei gas price (moderate):
# Deploy: 1,234,567 × 30 gwei = ~0.037 ETH (~$110 at $3000/ETH)
# Mint:   67,890    × 30 gwei = ~0.002 ETH (~$6)
```

---

## SECTION 11: Deploying to Mainnet

Only do this when:

- All tests pass with 100% coverage
- Contract is verified and tested on Sepolia
- You have audited the contract (or paid a firm to)
- You are using a hardware wallet + multisig for ownership

### Pre-Mainnet Checklist

- [ ] Deployed and tested on Sepolia for at least 1 week
- [ ] All edge cases tested with real interactions
- [ ] Constructor arguments reviewed by second person
- [ ] Ownership will be transferred to Gnosis Safe multisig
- [ ] Deployer wallet funded with 0.1-0.5 ETH for gas
- [ ] MAINNET_RPC_URL set in .env

### Deploy to Mainnet

```bash
# ⚠️ This spends real money and is irreversible
npm run deploy:mainnet
```

### Transfer Ownership to Gnosis Safe

1. Create a Gnosis Safe at https://safe.global
2. Set up a 2-of-3 or 3-of-5 multisig with trusted co-signers
3. Copy the Safe address
4. Transfer ownership:

```bash
npx hardhat console --network mainnet

const [deployer] = await ethers.getSigners();
const token = await ethers.getContractAt("MyToken", "0xCONTRACT");

// Step 1: Initiate transfer
await token.transferOwnership("0xYOUR_GNOSIS_SAFE_ADDRESS");
console.log("Pending owner:", await token.pendingOwner());

// Step 2: From Gnosis Safe UI, the Safe must call:
// token.acceptOwnership()
// This requires the required number of Safe signers to approve
```

---

## SECTION 12: Common Errors & How to Fix Them

### Compilation Errors

```
Error: ParserError: Expected ';' but got '}'
→ Missing semicolon or bracket. Check the line number.

Error: DeclarationError: Undeclared identifier "transfer"
→ You're calling a function that doesn't exist.
  Check the contract imports and inheritance chain.

Error: TypeError: Cannot read property of undefined
→ In TypeScript test files: typechain types may not be generated.
  Run: npm run compile && npm run typechain
```

### Transaction Errors

```
Error: "execution reverted: Ownable: caller is not the owner"
→ You called an owner-only function from a non-owner wallet.
  Make sure your .env PRIVATE_KEY is the owner's key.

Error: "insufficient funds for intrinsic transaction cost"
→ Your wallet doesn't have enough ETH for gas.
  Get Sepolia ETH from a faucet (testnet) or buy ETH (mainnet).

Error: "nonce too low"
→ Transaction nonce mismatch. Try again — sometimes a pending tx
  is stuck. Reset your MetaMask account nonce in Settings > Advanced.

Error: "transaction underpriced"
→ Gas price too low. Set gasPrice: "auto" in hardhat.config.ts.
```

### Etherscan Verification Errors

```
Error: "Already Verified"
→ Contract is already verified. Check Etherscan — it worked.

Error: "Bytecode does not match"
→ Constructor arguments are wrong, or optimizer settings differ.
  Double-check the arguments passed to the verify command.
  Ensure hardhat.config.ts optimizer settings match what was used to deploy.
```

---

## SECTION 13: Glossary

```
ABI (Application Binary Interface)
  The description of all functions and events in your contract.
  Ethers.js needs the ABI to know how to call your contract.
  Hardhat generates it automatically as a JSON file in artifacts/.

Address
  A 42-character hex string (0x + 40 chars) that identifies
  a wallet or contract on Ethereum. Like an email address.

Block
  A batch of transactions added to the blockchain every ~12 seconds.
  Once in a block and confirmed, transactions are permanent.

Constructor
  The function that runs once when the contract is deployed.
  Sets up initial state (owner, supply, name, etc.)

EOA (Externally Owned Account)
  A wallet controlled by a private key (your MetaMask wallet).
  Opposite of a contract account.

ERC20
  The standard interface for fungible tokens (like USDC, LINK).
  Defines transfer(), approve(), transferFrom(), etc.
  Any contract implementing ERC20 works with MetaMask and DEXes.

ERC721
  The standard interface for non-fungible tokens (NFTs).
  Each token has a unique ID and can have different metadata.

Gas
  The unit measuring computational work. Every operation costs gas.
  You pay gas fees in ETH to validators who process your transaction.

Gwei
  1,000,000,000 gwei = 1 ETH.
  Gas prices are quoted in gwei (e.g., "20 gwei").

Mainnet
  The real Ethereum network where ETH has real value.
  Transactions cost real money. Mistakes are permanent.

Sepolia
  An Ethereum test network. ETH here is free and worthless.
  Use for all testing before mainnet deployment.

Private Key
  The 64-character hex string that proves you own a wallet.
  Anyone with it can steal everything in that wallet.
  NEVER share, never commit to git.

Transaction
  Any operation that changes blockchain state.
  Requires gas fee. Goes into a block and is permanent.

Wei
  The smallest unit of ETH. 1 ETH = 10^18 wei.
  Contract amounts are always in wei.
  ethers.parseEther("1") converts 1 ETH to wei for you.

Contract Verification
  Publishing your source code to Etherscan so it matches
  the deployed bytecode. Allows users to audit your code.
  Required for user trust. Always do it.

Multisig (Multi-Signature)
  A wallet requiring multiple private keys to sign a transaction.
  E.g., 2-of-3: any 2 of 3 key holders must approve.
  Use for production contract ownership. Gnosis Safe is standard.

Nonce
  A counter for each wallet's transactions.
  Prevents replay attacks. Increments with each transaction.
```

---

## SECTION 14: Next Steps After Deployment

### Immediate (Same Day)

1. Verify contract on Etherscan ✓
2. Test all functions on Etherscan's Write Contract tab ✓
3. Add token to MetaMask and confirm it shows correctly ✓
4. Save deployment info (address, tx hash) in a secure place ✓

### Short Term (Within 1 Week)

5. For NFT: Upload all metadata to IPFS using Pinata (https://pinata.cloud)
6. For NFT: Set the correct BASE_URI with `setBaseURI()`
7. Set up a simple frontend (Wagmi + Viem is the modern standard)
8. Document all owner functions and when to use them

### For Production

9. Transfer ownership to Gnosis Safe multisig
10. Set up monitoring (Tenderly, OpenZeppelin Defender)
11. Consider a professional security audit (Trail of Bits, OpenZeppelin)
12. Set up event listeners to monitor contract activity

### Resources

- Ethereum docs: https://ethereum.org/developers
- OpenZeppelin: https://docs.openzeppelin.com
- Hardhat docs: https://hardhat.org/docs
- Ethers.js v6: https://docs.ethers.org/v6
- Solidity docs: https://docs.soliditylang.org
- Chainlink (oracles, VRF): https://docs.chain.link
- Gnosis Safe: https://safe.global
- IPFS (Pinata): https://pinata.cloud

```

```
