# Production Smart Contract Testing

# Hardhat + Ethers.js v6 + Chai — Complete Test Suite Patterns

---

## Testing Philosophy

- **100% function coverage is the floor, not the ceiling.**
- Test every `require`/`revert` condition — not just happy paths.
- Test all events with exact argument matching.
- Test edge cases: zero values, max values, address(0), repeated actions.
- Test reentrancy attacks explicitly.
- Test unauthorized callers for every protected function.
- Use `loadFixture` for gas-efficient test isolation.

---

## Test File Structure

```typescript
// test/MyToken.test.ts
import { expect } from "chai";
import { ethers } from "hardhat";
import { loadFixture } from "@nomicfoundation/hardhat-network-helpers";
import { time } from "@nomicfoundation/hardhat-network-helpers";
import type { MyToken } from "../typechain-types";
import type { SignerWithAddress } from "@nomicfoundation/hardhat-ethers/signers";

// ================================================================
// Constants
// ================================================================

const ONE_ETHER = ethers.parseEther("1");
const MAX_SUPPLY = ethers.parseEther("1000000000"); // 1 billion
const INITIAL_SUPPLY = ethers.parseEther("100000"); // 100k to deployer

// ================================================================
// Fixture — shared setup, runs once and snapshots state
// ================================================================

async function deployTokenFixture() {
	const [owner, alice, bob, charlie, attacker] = await ethers.getSigners();

	const MyToken = await ethers.getContractFactory("MyToken");
	const token = await MyToken.deploy(
		"My Token",
		"MTK",
		owner.address,
		INITIAL_SUPPLY,
	);
	await token.waitForDeployment();

	return { token, owner, alice, bob, charlie, attacker };
}

// ================================================================
// Test Suite
// ================================================================

describe("MyToken", function () {
	// ============================================================
	// Deployment & Initial State
	// ============================================================

	describe("Deployment", function () {
		it("should deploy with correct name and symbol", async function () {
			const { token } = await loadFixture(deployTokenFixture);
			expect(await token.name()).to.equal("My Token");
			expect(await token.symbol()).to.equal("MTK");
		});

		it("should mint initial supply to owner", async function () {
			const { token, owner } = await loadFixture(deployTokenFixture);
			expect(await token.balanceOf(owner.address)).to.equal(INITIAL_SUPPLY);
		});

		it("should set correct owner", async function () {
			const { token, owner } = await loadFixture(deployTokenFixture);
			expect(await token.owner()).to.equal(owner.address);
		});

		it("should have 18 decimals", async function () {
			const { token } = await loadFixture(deployTokenFixture);
			expect(await token.decimals()).to.equal(18);
		});

		it("should have correct MAX_SUPPLY constant", async function () {
			const { token } = await loadFixture(deployTokenFixture);
			expect(await token.MAX_SUPPLY()).to.equal(MAX_SUPPLY);
		});

		it("should revert if initial owner is zero address", async function () {
			const MyToken = await ethers.getContractFactory("MyToken");
			await expect(
				MyToken.deploy("My Token", "MTK", ethers.ZeroAddress, 0),
			).to.be.revertedWithCustomError(
				await ethers.getContractFactory("MyToken"),
				"ZeroAddress",
			);
		});

		it("should revert if initial supply exceeds MAX_SUPPLY", async function () {
			const MyToken = await ethers.getContractFactory("MyToken");
			const [owner] = await ethers.getSigners();
			const overSupply = MAX_SUPPLY + 1n;
			await expect(
				MyToken.deploy("My Token", "MTK", owner.address, overSupply),
			).to.be.revertedWithCustomError(MyToken, "ExceedsMaxSupply");
		});
	});

	// ============================================================
	// Minting
	// ============================================================

	describe("mint()", function () {
		it("should allow owner to mint tokens", async function () {
			const { token, owner, alice } = await loadFixture(deployTokenFixture);
			const mintAmount = ethers.parseEther("1000");

			await token.connect(owner).mint(alice.address, mintAmount);

			expect(await token.balanceOf(alice.address)).to.equal(mintAmount);
		});

		it("should emit TokensMinted event with correct args", async function () {
			const { token, owner, alice } = await loadFixture(deployTokenFixture);
			const mintAmount = ethers.parseEther("1000");

			await expect(token.connect(owner).mint(alice.address, mintAmount))
				.to.emit(token, "TokensMinted")
				.withArgs(alice.address, mintAmount);
		});

		it("should increase totalSupply after mint", async function () {
			const { token, owner, alice } = await loadFixture(deployTokenFixture);
			const mintAmount = ethers.parseEther("1000");
			const supplyBefore = await token.totalSupply();

			await token.connect(owner).mint(alice.address, mintAmount);

			expect(await token.totalSupply()).to.equal(supplyBefore + mintAmount);
		});

		it("should revert if called by non-owner", async function () {
			const { token, alice, bob } = await loadFixture(deployTokenFixture);
			await expect(
				token.connect(alice).mint(bob.address, ONE_ETHER),
			).to.be.revertedWithCustomError(token, "OwnableUnauthorizedAccount");
		});

		it("should revert minting to zero address", async function () {
			const { token, owner } = await loadFixture(deployTokenFixture);
			await expect(
				token.connect(owner).mint(ethers.ZeroAddress, ONE_ETHER),
			).to.be.revertedWithCustomError(token, "ZeroAddress");
		});

		it("should revert minting zero amount", async function () {
			const { token, owner, alice } = await loadFixture(deployTokenFixture);
			await expect(
				token.connect(owner).mint(alice.address, 0),
			).to.be.revertedWithCustomError(token, "ZeroAmount");
		});

		it("should revert when exceeding MAX_SUPPLY", async function () {
			const { token, owner, alice } = await loadFixture(deployTokenFixture);
			const remaining = MAX_SUPPLY - INITIAL_SUPPLY;
			const overAmount = remaining + 1n;

			await expect(
				token.connect(owner).mint(alice.address, overAmount),
			).to.be.revertedWithCustomError(token, "ExceedsMaxSupply");
		});

		it("should allow minting up to exactly MAX_SUPPLY", async function () {
			const { token, owner, alice } = await loadFixture(deployTokenFixture);
			const remaining = MAX_SUPPLY - INITIAL_SUPPLY;

			await token.connect(owner).mint(alice.address, remaining);

			expect(await token.totalSupply()).to.equal(MAX_SUPPLY);
		});
	});

	// ============================================================
	// Burning
	// ============================================================

	describe("burn()", function () {
		it("should allow token holder to burn their tokens", async function () {
			const { token, owner } = await loadFixture(deployTokenFixture);
			const burnAmount = ethers.parseEther("1000");
			const balanceBefore = await token.balanceOf(owner.address);

			await token.connect(owner).burn(burnAmount);

			expect(await token.balanceOf(owner.address)).to.equal(
				balanceBefore - burnAmount,
			);
		});

		it("should reduce totalSupply after burn", async function () {
			const { token, owner } = await loadFixture(deployTokenFixture);
			const burnAmount = ethers.parseEther("1000");
			const supplyBefore = await token.totalSupply();

			await token.connect(owner).burn(burnAmount);

			expect(await token.totalSupply()).to.equal(supplyBefore - burnAmount);
		});

		it("should revert burning more than balance", async function () {
			const { token, alice } = await loadFixture(deployTokenFixture);
			// alice has 0 tokens
			await expect(
				token.connect(alice).burn(ONE_ETHER),
			).to.be.revertedWithCustomError(token, "ERC20InsufficientBalance");
		});
	});

	// ============================================================
	// Pause
	// ============================================================

	describe("pause() / unpause()", function () {
		it("should allow owner to pause", async function () {
			const { token, owner } = await loadFixture(deployTokenFixture);
			await token.connect(owner).pause();
			expect(await token.paused()).to.equal(true);
		});

		it("should block transfers when paused", async function () {
			const { token, owner, alice } = await loadFixture(deployTokenFixture);
			await token.connect(owner).pause();

			await expect(
				token.connect(owner).transfer(alice.address, ONE_ETHER),
			).to.be.revertedWithCustomError(token, "EnforcedPause");
		});

		it("should block minting when paused", async function () {
			const { token, owner, alice } = await loadFixture(deployTokenFixture);
			await token.connect(owner).pause();

			await expect(
				token.connect(owner).mint(alice.address, ONE_ETHER),
			).to.be.revertedWithCustomError(token, "EnforcedPause");
		});

		it("should resume transfers after unpause", async function () {
			const { token, owner, alice } = await loadFixture(deployTokenFixture);
			await token.connect(owner).pause();
			await token.connect(owner).unpause();

			await expect(token.connect(owner).transfer(alice.address, ONE_ETHER)).to
				.not.be.reverted;
		});

		it("should revert if non-owner tries to pause", async function () {
			const { token, alice } = await loadFixture(deployTokenFixture);
			await expect(token.connect(alice).pause()).to.be.revertedWithCustomError(
				token,
				"OwnableUnauthorizedAccount",
			);
		});
	});

	// ============================================================
	// Ownership Transfer (Two-Step)
	// ============================================================

	describe("Ownable2Step", function () {
		it("should require acceptance from new owner", async function () {
			const { token, owner, alice } = await loadFixture(deployTokenFixture);

			await token.connect(owner).transferOwnership(alice.address);

			// Ownership has NOT transferred yet
			expect(await token.owner()).to.equal(owner.address);
			expect(await token.pendingOwner()).to.equal(alice.address);
		});

		it("should transfer ownership after acceptance", async function () {
			const { token, owner, alice } = await loadFixture(deployTokenFixture);

			await token.connect(owner).transferOwnership(alice.address);
			await token.connect(alice).acceptOwnership();

			expect(await token.owner()).to.equal(alice.address);
		});

		it("should block old owner after transfer", async function () {
			const { token, owner, alice, bob } = await loadFixture(
				deployTokenFixture,
			);

			await token.connect(owner).transferOwnership(alice.address);
			await token.connect(alice).acceptOwnership();

			await expect(
				token.connect(owner).mint(bob.address, ONE_ETHER),
			).to.be.revertedWithCustomError(token, "OwnableUnauthorizedAccount");
		});
	});

	// ============================================================
	// ERC20 Standard Behavior
	// ============================================================

	describe("ERC20 standard", function () {
		it("should transfer tokens correctly", async function () {
			const { token, owner, alice } = await loadFixture(deployTokenFixture);
			const amount = ethers.parseEther("100");

			await token.connect(owner).transfer(alice.address, amount);

			expect(await token.balanceOf(alice.address)).to.equal(amount);
		});

		it("should handle approve and transferFrom", async function () {
			const { token, owner, alice, bob } = await loadFixture(
				deployTokenFixture,
			);
			const amount = ethers.parseEther("100");

			await token.connect(owner).approve(alice.address, amount);
			await token
				.connect(alice)
				.transferFrom(owner.address, bob.address, amount);

			expect(await token.balanceOf(bob.address)).to.equal(amount);
		});

		it("should revert transferFrom exceeding allowance", async function () {
			const { token, owner, alice, bob } = await loadFixture(
				deployTokenFixture,
			);
			const allowance = ethers.parseEther("100");
			const overAmount = ethers.parseEther("101");

			await token.connect(owner).approve(alice.address, allowance);

			await expect(
				token
					.connect(alice)
					.transferFrom(owner.address, bob.address, overAmount),
			).to.be.revertedWithCustomError(token, "ERC20InsufficientAllowance");
		});
	});
});
```

---

## ETH Vault Test Suite

```typescript
// test/ETHVault.test.ts
import { expect } from "chai";
import { ethers } from "hardhat";
import { loadFixture } from "@nomicfoundation/hardhat-network-helpers";

async function deployVaultFixture() {
	const [owner, alice, bob, attacker] = await ethers.getSigners();
	const ETHVault = await ethers.getContractFactory("ETHVault");
	const vault = await ETHVault.deploy(owner.address);
	await vault.waitForDeployment();
	return { vault, owner, alice, bob, attacker };
}

describe("ETHVault", function () {
	describe("deposit()", function () {
		it("should accept ETH and update balance", async function () {
			const { vault, alice } = await loadFixture(deployVaultFixture);
			const amount = ethers.parseEther("1");

			await vault.connect(alice).deposit({ value: amount });

			expect(await vault.getBalance(alice.address)).to.equal(amount);
			expect(await vault.totalDeposited()).to.equal(amount);
		});

		it("should emit Deposited event", async function () {
			const { vault, alice } = await loadFixture(deployVaultFixture);
			const amount = ethers.parseEther("1");

			await expect(vault.connect(alice).deposit({ value: amount }))
				.to.emit(vault, "Deposited")
				.withArgs(alice.address, amount, amount);
		});

		it("should revert on zero deposit", async function () {
			const { vault, alice } = await loadFixture(deployVaultFixture);
			await expect(
				vault.connect(alice).deposit({ value: 0 }),
			).to.be.revertedWithCustomError(vault, "ZeroAmount");
		});

		it("should revert when exceeding MAX_DEPOSIT", async function () {
			const { vault, alice } = await loadFixture(deployVaultFixture);
			const max = await vault.MAX_DEPOSIT();

			await expect(
				vault.connect(alice).deposit({ value: max + 1n }),
			).to.be.revertedWithCustomError(vault, "ExceedsMaxDeposit");
		});

		it("should revert when paused", async function () {
			const { vault, owner, alice } = await loadFixture(deployVaultFixture);
			await vault.connect(owner).pause();

			await expect(
				vault.connect(alice).deposit({ value: ethers.parseEther("1") }),
			).to.be.revertedWithCustomError(vault, "EnforcedPause");
		});
	});

	describe("withdraw()", function () {
		async function depositFixture() {
			const base = await deployVaultFixture();
			const amount = ethers.parseEther("5");
			await base.vault.connect(base.alice).deposit({ value: amount });
			return { ...base, depositedAmount: amount };
		}

		it("should withdraw correct amount", async function () {
			const { vault, alice, depositedAmount } = await loadFixture(
				depositFixture,
			);
			const withdrawAmount = ethers.parseEther("2");
			const balanceBefore = await ethers.provider.getBalance(alice.address);

			const tx = await vault.connect(alice).withdraw(withdrawAmount);
			const receipt = await tx.wait();
			const gasUsed = receipt!.gasUsed * receipt!.gasPrice;

			const balanceAfter = await ethers.provider.getBalance(alice.address);
			expect(balanceAfter).to.be.closeTo(
				balanceBefore + withdrawAmount - gasUsed,
				ethers.parseEther("0.0001"), // Tiny tolerance for gas estimation
			);
		});

		it("should update internal balance after withdrawal", async function () {
			const { vault, alice, depositedAmount } = await loadFixture(
				depositFixture,
			);
			const withdrawAmount = ethers.parseEther("2");

			await vault.connect(alice).withdraw(withdrawAmount);

			expect(await vault.getBalance(alice.address)).to.equal(
				depositedAmount - withdrawAmount,
			);
		});

		it("should emit Withdrawn event", async function () {
			const { vault, alice, depositedAmount } = await loadFixture(
				depositFixture,
			);
			const withdrawAmount = ethers.parseEther("2");

			await expect(vault.connect(alice).withdraw(withdrawAmount))
				.to.emit(vault, "Withdrawn")
				.withArgs(
					alice.address,
					withdrawAmount,
					depositedAmount - withdrawAmount,
				);
		});

		it("should revert on zero withdrawal", async function () {
			const { vault, alice } = await loadFixture(depositFixture);
			await expect(
				vault.connect(alice).withdraw(0),
			).to.be.revertedWithCustomError(vault, "ZeroAmount");
		});

		it("should revert on insufficient balance", async function () {
			const { vault, alice, depositedAmount } = await loadFixture(
				depositFixture,
			);
			const overAmount = depositedAmount + 1n;

			await expect(
				vault.connect(alice).withdraw(overAmount),
			).to.be.revertedWithCustomError(vault, "InsufficientBalance");
		});

		it("should not allow withdrawing other users funds", async function () {
			const { vault, bob } = await loadFixture(depositFixture);
			// Bob has 0 deposited
			await expect(
				vault.connect(bob).withdraw(ethers.parseEther("1")),
			).to.be.revertedWithCustomError(vault, "InsufficientBalance");
		});
	});

	// ============================================================
	// REENTRANCY ATTACK TEST — Critical
	// ============================================================

	describe("Reentrancy protection", function () {
		it("should block reentrancy attack", async function () {
			const [owner, attacker] = await ethers.getSigners();

			// Deploy vault
			const ETHVault = await ethers.getContractFactory("ETHVault");
			const vault = await ETHVault.deploy(owner.address);

			// Deploy attacker contract
			const Attacker = await ethers.getContractFactory("ReentrancyAttacker");
			const attackerContract = await Attacker.deploy(await vault.getAddress());

			// Fund the vault with legitimate user deposits
			const [, , alice] = await ethers.getSigners();
			await vault.connect(alice).deposit({ value: ethers.parseEther("10") });

			// Fund attacker and make initial deposit through attacker
			await attackerContract.connect(attacker).depositToVault({
				value: ethers.parseEther("1"),
			});

			// Attempt reentrancy attack — should revert
			await expect(
				attackerContract.connect(attacker).attack(),
			).to.be.revertedWithCustomError(vault, "ReentrancyGuardReentrantCall");
		});
	});
});
```

---

## Reentrancy Attacker Contract (For Testing Only)

```solidity
// contracts/test/ReentrancyAttacker.sol
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

interface IVault {
    function deposit() external payable;
    function withdraw(uint256 amount) external;
    function getBalance(address user) external view returns (uint256);
}

/**
 * @dev FOR TESTING ONLY — demonstrates a reentrancy attack
 * Deploy only to local Hardhat network / testnets, never mainnet
 */
contract ReentrancyAttacker {
    IVault public immutable vault;
    uint256 public attackAmount;
    uint256 public attackCount;
    uint256 public constant MAX_ATTACKS = 10;

    constructor(address _vault) {
        vault = IVault(_vault);
    }

    function depositToVault() external payable {
        vault.deposit{value: msg.value}();
        attackAmount = msg.value;
    }

    function attack() external {
        attackCount = 0;
        vault.withdraw(attackAmount);
    }

    // Reentrancy: called when vault sends ETH back
    receive() external payable {
        attackCount++;
        if (attackCount < MAX_ATTACKS) {
            uint256 bal = vault.getBalance(address(this));
            if (bal >= attackAmount) {
                vault.withdraw(attackAmount); // Try to drain more
            }
        }
    }
}
```

---

## NFT Test Suite

```typescript
// test/MyNFT.test.ts
import { expect } from "chai";
import { ethers } from "hardhat";
import { loadFixture } from "@nomicfoundation/hardhat-network-helpers";

const MINT_PRICE = ethers.parseEther("0.08");
const WL_PRICE = ethers.parseEther("0.06");
const MAX_SUPPLY = 10_000n;
const MAX_PER_WALLET = 5n;
const BASE_URI = "ipfs://QmHash/";

enum SalePhase {
	CLOSED = 0,
	WHITELIST = 1,
	PUBLIC = 2,
}

async function deployNFTFixture() {
	const [owner, alice, bob, charlie, wlUser] = await ethers.getSigners();
	const MyNFT = await ethers.getContractFactory("MyNFT");
	const nft = await MyNFT.deploy("My NFT", "MNFT", BASE_URI, owner.address);
	await nft.waitForDeployment();
	return { nft, owner, alice, bob, charlie, wlUser };
}

describe("MyNFT", function () {
	describe("Deployment", function () {
		it("should have correct name and symbol", async function () {
			const { nft } = await loadFixture(deployNFTFixture);
			expect(await nft.name()).to.equal("My NFT");
			expect(await nft.symbol()).to.equal("MNFT");
		});

		it("should start with CLOSED sale phase", async function () {
			const { nft } = await loadFixture(deployNFTFixture);
			expect(await nft.salePhase()).to.equal(SalePhase.CLOSED);
		});

		it("should have zero totalSupply", async function () {
			const { nft } = await loadFixture(deployNFTFixture);
			expect(await nft.totalSupply()).to.equal(0);
		});
	});

	describe("Whitelist Minting", function () {
		async function whitelistFixture() {
			const base = await deployNFTFixture();
			await base.nft
				.connect(base.owner)
				.setWhitelist([base.wlUser.address], true);
			await base.nft.connect(base.owner).setSalePhase(SalePhase.WHITELIST);
			return base;
		}

		it("should allow whitelisted user to mint at WL price", async function () {
			const { nft, wlUser } = await loadFixture(whitelistFixture);
			await nft.connect(wlUser).mint(1, { value: WL_PRICE });
			expect(await nft.balanceOf(wlUser.address)).to.equal(1);
		});

		it("should revert if not whitelisted", async function () {
			const { nft, alice } = await loadFixture(whitelistFixture);
			await expect(
				nft.connect(alice).mint(1, { value: WL_PRICE }),
			).to.be.revertedWithCustomError(nft, "NotWhitelisted");
		});

		it("should revert if insufficient payment", async function () {
			const { nft, wlUser } = await loadFixture(whitelistFixture);
			const lowPrice = WL_PRICE - 1n;
			await expect(
				nft.connect(wlUser).mint(1, { value: lowPrice }),
			).to.be.revertedWithCustomError(nft, "InsufficientPayment");
		});

		it("should emit Minted event for each token", async function () {
			const { nft, wlUser } = await loadFixture(whitelistFixture);
			const quantity = 2n;
			const tx = await nft
				.connect(wlUser)
				.mint(quantity, { value: WL_PRICE * quantity });

			// Check two Minted events were emitted
			await expect(tx).to.emit(nft, "Minted").withArgs(wlUser.address, 0);
			await expect(tx).to.emit(nft, "Minted").withArgs(wlUser.address, 1);
		});
	});

	describe("Public Minting", function () {
		async function publicSaleFixture() {
			const base = await deployNFTFixture();
			await base.nft.connect(base.owner).setSalePhase(SalePhase.PUBLIC);
			return base;
		}

		it("should allow anyone to mint at public price", async function () {
			const { nft, alice } = await loadFixture(publicSaleFixture);
			await nft.connect(alice).mint(1, { value: MINT_PRICE });
			expect(await nft.balanceOf(alice.address)).to.equal(1);
		});

		it("should enforce MAX_PER_WALLET", async function () {
			const { nft, alice } = await loadFixture(publicSaleFixture);
			await nft.connect(alice).mint(5, { value: MINT_PRICE * 5n });

			await expect(
				nft.connect(alice).mint(1, { value: MINT_PRICE }),
			).to.be.revertedWithCustomError(nft, "MaxPerWalletReached");
		});

		it("should revert when sale is closed", async function () {
			const { nft, alice } = await loadFixture(deployNFTFixture);
			// Sale is CLOSED by default
			await expect(
				nft.connect(alice).mint(1, { value: MINT_PRICE }),
			).to.be.revertedWithCustomError(nft, "SaleClosed");
		});

		it("should correctly track tokenURIs", async function () {
			const { nft, alice } = await loadFixture(publicSaleFixture);
			await nft.connect(alice).mint(1, { value: MINT_PRICE });

			const uri = await nft.tokenURI(0);
			expect(uri).to.equal(`${BASE_URI}0.json`);
		});
	});

	describe("Fund Withdrawal", function () {
		it("should allow owner to withdraw funds", async function () {
			const { nft, owner, alice } = await loadFixture(deployNFTFixture);
			await nft.connect(owner).setSalePhase(SalePhase.PUBLIC);
			await nft.connect(alice).mint(1, { value: MINT_PRICE });

			const ownerBalBefore = await ethers.provider.getBalance(owner.address);
			const tx = await nft.connect(owner).withdrawFunds();
			const receipt = await tx.wait();
			const gasUsed = receipt!.gasUsed * receipt!.gasPrice;

			const ownerBalAfter = await ethers.provider.getBalance(owner.address);
			expect(ownerBalAfter).to.be.closeTo(
				ownerBalBefore + MINT_PRICE - gasUsed,
				ethers.parseEther("0.0001"),
			);
		});

		it("should emit FundsWithdrawn event", async function () {
			const { nft, owner, alice } = await loadFixture(deployNFTFixture);
			await nft.connect(owner).setSalePhase(SalePhase.PUBLIC);
			await nft.connect(alice).mint(1, { value: MINT_PRICE });

			await expect(nft.connect(owner).withdrawFunds())
				.to.emit(nft, "FundsWithdrawn")
				.withArgs(owner.address, MINT_PRICE);
		});

		it("should revert if non-owner tries to withdraw", async function () {
			const { nft, alice } = await loadFixture(deployNFTFixture);
			await expect(
				nft.connect(alice).withdrawFunds(),
			).to.be.revertedWithCustomError(nft, "OwnableUnauthorizedAccount");
		});

		it("should revert if no funds to withdraw", async function () {
			const { nft, owner } = await loadFixture(deployNFTFixture);
			await expect(
				nft.connect(owner).withdrawFunds(),
			).to.be.revertedWithCustomError(nft, "NothingToWithdraw");
		});
	});

	describe("Owner Mint (Team/Giveaway)", function () {
		it("should allow owner to mint for free", async function () {
			const { nft, owner, alice } = await loadFixture(deployNFTFixture);
			await nft.connect(owner).ownerMint(alice.address, 10);
			expect(await nft.balanceOf(alice.address)).to.equal(10);
		});

		it("should count toward MAX_SUPPLY", async function () {
			const { nft, owner, alice } = await loadFixture(deployNFTFixture);
			await nft.connect(owner).ownerMint(alice.address, 10);
			expect(await nft.totalSupply()).to.equal(10);
		});
	});
});
```

---

## Test Helpers & Utilities

```typescript
// test/helpers/fixtures.ts
import { ethers } from "hardhat";

/**
 * Increase blockchain time by `seconds`
 * Useful for testing time-locked functions
 */
export async function increaseTime(seconds: number) {
	await ethers.provider.send("evm_increaseTime", [seconds]);
	await ethers.provider.send("evm_mine", []);
}

/**
 * Mine `n` blocks
 */
export async function mineBlocks(n: number) {
	for (let i = 0; i < n; i++) {
		await ethers.provider.send("evm_mine", []);
	}
}

/**
 * Get ETH balance as a BigInt
 */
export async function getBalance(address: string): Promise<bigint> {
	return ethers.provider.getBalance(address);
}

/**
 * Impersonate an account (for mainnet fork tests)
 */
export async function impersonate(address: string) {
	await ethers.provider.send("hardhat_impersonateAccount", [address]);
	return ethers.getSigner(address);
}

/**
 * Format ETH for readable test output
 */
export function formatEth(wei: bigint): string {
	return `${ethers.formatEther(wei)} ETH`;
}
```

---

## Running Tests

```bash
# Run all tests
npm test

# Run single file
npx hardhat test test/MyToken.test.ts

# Run with gas report
REPORT_GAS=true npm test

# Run with coverage (generates HTML report in coverage/)
npm run test:coverage

# Run on mainnet fork
FORK=true npm test
```

---

## Coverage Requirements

```bash
# After running coverage, check output:
# Statements: 100%
# Branches:   100%
# Functions:  100%
# Lines:      100%

# Any uncovered line is a potential bug or missing test.
# Investigate every coverage gap before deploying.
```
