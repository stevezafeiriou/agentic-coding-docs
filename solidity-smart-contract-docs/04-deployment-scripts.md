# Deployment Scripts & Etherscan Verification

# Hardhat + Ethers.js v6

---

## Deploy Script — ERC20 Token

```typescript
// scripts/deploy.ts
import { ethers, run, network } from "hardhat";
import * as fs from "fs";
import * as path from "path";

/**
 * Deployment script for MyToken
 *
 * Usage:
 *   Testnet: npx hardhat run scripts/deploy.ts --network sepolia
 *   Mainnet: npx hardhat run scripts/deploy.ts --network mainnet
 *
 * After deployment:
 *   1. Copy the contract address from console output
 *   2. Run verify script to verify on Etherscan
 *   3. Transfer ownership to multisig (mainnet)
 */
async function main() {
	// ============================================================
	// Pre-flight checks
	// ============================================================

	console.log("=".repeat(60));
	console.log("Deploying MyToken");
	console.log("Network:", network.name);
	console.log("=".repeat(60));

	const [deployer] = await ethers.getSigners();
	console.log("Deployer address:", deployer.address);

	const deployerBalance = await ethers.provider.getBalance(deployer.address);
	console.log(
		"Deployer ETH balance:",
		ethers.formatEther(deployerBalance),
		"ETH",
	);

	if (deployerBalance === 0n) {
		throw new Error("Deployer has no ETH. Fund the deployer wallet first.");
	}

	// ============================================================
	// Constructor arguments — EDIT THESE before deploying
	// ============================================================

	const TOKEN_NAME = "My Token";
	const TOKEN_SYMBOL = "MTK";
	const INITIAL_OWNER = deployer.address; // Change to multisig for mainnet
	const INITIAL_SUPPLY = ethers.parseEther("100000"); // 100,000 tokens to deployer

	console.log("\nConstructor arguments:");
	console.log("  Name:", TOKEN_NAME);
	console.log("  Symbol:", TOKEN_SYMBOL);
	console.log("  Initial Owner:", INITIAL_OWNER);
	console.log(
		"  Initial Supply:",
		ethers.formatEther(INITIAL_SUPPLY),
		"tokens",
	);

	// ============================================================
	// Estimate gas before deploying
	// ============================================================

	const MyToken = await ethers.getContractFactory("MyToken");
	const deployTx = await MyToken.getDeployTransaction(
		TOKEN_NAME,
		TOKEN_SYMBOL,
		INITIAL_OWNER,
		INITIAL_SUPPLY,
	);

	const gasEstimate = await ethers.provider.estimateGas(deployTx);
	const feeData = await ethers.provider.getFeeData();
	const estimatedCost =
		gasEstimate * (feeData.maxFeePerGas ?? feeData.gasPrice ?? 0n);

	console.log("\nGas estimate:", gasEstimate.toString());
	console.log("Estimated cost:", ethers.formatEther(estimatedCost), "ETH");

	if (estimatedCost > deployerBalance) {
		throw new Error("Insufficient ETH for deployment. Add more funds.");
	}

	// ============================================================
	// Deploy
	// ============================================================

	console.log("\nDeploying...");
	const token = await MyToken.deploy(
		TOKEN_NAME,
		TOKEN_SYMBOL,
		INITIAL_OWNER,
		INITIAL_SUPPLY,
	);

	console.log("Transaction hash:", token.deploymentTransaction()?.hash);
	console.log("Waiting for confirmation...");

	await token.waitForDeployment();
	const contractAddress = await token.getAddress();

	console.log("\n✅ Deployed successfully!");
	console.log("Contract address:", contractAddress);

	// ============================================================
	// Post-deployment verification
	// ============================================================

	console.log("\nVerifying deployment...");
	const deployedName = await token.name();
	const deployedSymbol = await token.symbol();
	const deployedOwner = await token.owner();
	const totalSupply = await token.totalSupply();

	console.log("  name():", deployedName);
	console.log("  symbol():", deployedSymbol);
	console.log("  owner():", deployedOwner);
	console.log("  totalSupply():", ethers.formatEther(totalSupply));

	// ============================================================
	// Save deployment info to file
	// ============================================================

	const deploymentInfo = {
		network: network.name,
		chainId: (await ethers.provider.getNetwork()).chainId.toString(),
		contractAddress,
		deployer: deployer.address,
		txHash: token.deploymentTransaction()?.hash,
		blockNumber: token.deploymentTransaction()?.blockNumber,
		timestamp: new Date().toISOString(),
		constructorArgs: {
			name: TOKEN_NAME,
			symbol: TOKEN_SYMBOL,
			initialOwner: INITIAL_OWNER,
			initialSupply: INITIAL_SUPPLY.toString(),
		},
	};

	const deploymentsDir = path.join(__dirname, "../deployments");
	if (!fs.existsSync(deploymentsDir))
		fs.mkdirSync(deploymentsDir, { recursive: true });

	const outFile = path.join(deploymentsDir, `${network.name}-MyToken.json`);
	fs.writeFileSync(outFile, JSON.stringify(deploymentInfo, null, 2));
	console.log("\nDeployment info saved to:", outFile);

	// ============================================================
	// Auto-verify on non-local networks
	// ============================================================

	if (network.name !== "hardhat" && network.name !== "localhost") {
		console.log("\nWaiting 5 blocks before verification...");
		// Wait for Etherscan to index the contract
		await new Promise((r) => setTimeout(r, 30_000)); // 30 second delay

		await verifyContract(contractAddress, [
			TOKEN_NAME,
			TOKEN_SYMBOL,
			INITIAL_OWNER,
			INITIAL_SUPPLY.toString(),
		]);
	}

	console.log("\n" + "=".repeat(60));
	console.log("DEPLOYMENT COMPLETE");
	console.log("=".repeat(60));
	console.log("Contract:", contractAddress);
	console.log("Etherscan:", getEtherscanUrl(network.name, contractAddress));

	if (network.name === "mainnet") {
		console.log("\n⚠️  IMPORTANT: Transfer ownership to your multisig!");
		console.log("   Call: token.transferOwnership(YOUR_MULTISIG_ADDRESS)");
	}
}

// ============================================================
// Etherscan Verification Helper
// ============================================================

async function verifyContract(address: string, constructorArgs: unknown[]) {
	console.log("\nVerifying on Etherscan...");
	try {
		await run("verify:verify", {
			address,
			constructorArguments: constructorArgs,
		});
		console.log("✅ Contract verified on Etherscan");
	} catch (err: unknown) {
		if (err instanceof Error && err.message.includes("Already Verified")) {
			console.log("Contract already verified");
		} else {
			console.error("Verification failed:", err);
			console.log("Run manually:");
			console.log(
				`npx hardhat verify --network ${
					network.name
				} ${address} ${constructorArgs.join(" ")}`,
			);
		}
	}
}

function getEtherscanUrl(networkName: string, address: string): string {
	const baseUrls: Record<string, string> = {
		mainnet: "https://etherscan.io/address/",
		sepolia: "https://sepolia.etherscan.io/address/",
	};
	return (baseUrls[networkName] ?? "") + address;
}

main().catch((err) => {
	console.error("\n❌ Deployment failed:", err);
	process.exit(1);
});
```

---

## Deploy Script — NFT

```typescript
// scripts/deploy-nft.ts
import { ethers, run, network } from "hardhat";
import * as fs from "fs";
import * as path from "path";

async function main() {
	const [deployer] = await ethers.getSigners();

	console.log("Deploying MyNFT on", network.name);
	console.log("Deployer:", deployer.address);
	console.log(
		"Balance:",
		ethers.formatEther(await ethers.provider.getBalance(deployer.address)),
		"ETH",
	);

	// ============================================================
	// EDIT THESE — NFT Configuration
	// ============================================================
	const NFT_NAME = "My NFT Collection";
	const NFT_SYMBOL = "MNFT";
	// IPFS base URI — upload your metadata to IPFS first (Pinata / nft.storage)
	// Format: ipfs://YOUR_CID/ (trailing slash required)
	const BASE_URI = "ipfs://QmYourCIDHere/";
	const OWNER = deployer.address; // Change to multisig for mainnet

	const MyNFT = await ethers.getContractFactory("MyNFT");
	const nft = await MyNFT.deploy(NFT_NAME, NFT_SYMBOL, BASE_URI, OWNER);
	await nft.waitForDeployment();
	const address = await nft.getAddress();

	console.log("✅ MyNFT deployed:", address);

	// Save deployment record
	const info = {
		network: network.name,
		address,
		txHash: nft.deploymentTransaction()?.hash,
		timestamp: new Date().toISOString(),
		constructorArgs: {
			name: NFT_NAME,
			symbol: NFT_SYMBOL,
			baseURI: BASE_URI,
			owner: OWNER,
		},
	};
	const dir = path.join(__dirname, "../deployments");
	if (!fs.existsSync(dir)) fs.mkdirSync(dir, { recursive: true });
	fs.writeFileSync(
		path.join(dir, `${network.name}-MyNFT.json`),
		JSON.stringify(info, null, 2),
	);

	// Verify
	if (network.name !== "hardhat" && network.name !== "localhost") {
		await new Promise((r) => setTimeout(r, 30_000));
		try {
			await run("verify:verify", {
				address,
				constructorArguments: [NFT_NAME, NFT_SYMBOL, BASE_URI, OWNER],
			});
			console.log("✅ Verified on Etherscan");
		} catch (e) {
			console.log(
				"Verify manually:",
				`npx hardhat verify --network ${network.name} ${address} "${NFT_NAME}" "${NFT_SYMBOL}" "${BASE_URI}" "${OWNER}"`,
			);
		}
	}

	console.log("\nNext steps:");
	console.log(
		"1. Verify contract:",
		`https://sepolia.etherscan.io/address/${address}`,
	);
	console.log("2. Set whitelist: nft.setWhitelist([addresses], true)");
	console.log("3. Open presale: nft.setSalePhase(1)");
	console.log("4. Open public:  nft.setSalePhase(2)");
}

main().catch((err) => {
	console.error(err);
	process.exit(1);
});
```

---

## Standalone Verify Script (Run After Deploy)

```typescript
// scripts/verify.ts
import { run, network } from "hardhat";
import * as fs from "fs";
import * as path from "path";

async function main() {
	// Load saved deployment info
	const deploymentFile = path.join(
		__dirname,
		`../deployments/${network.name}-MyToken.json`,
	);

	if (!fs.existsSync(deploymentFile)) {
		throw new Error(
			`No deployment file found for ${network.name}. ` +
				`Run deploy script first.`,
		);
	}

	const deployment = JSON.parse(fs.readFileSync(deploymentFile, "utf-8"));
	const { contractAddress, constructorArgs } = deployment;

	console.log("Verifying contract:", contractAddress);
	console.log("Network:", network.name);

	await run("verify:verify", {
		address: contractAddress,
		constructorArguments: [
			constructorArgs.name,
			constructorArgs.symbol,
			constructorArgs.initialOwner,
			constructorArgs.initialSupply,
		],
	});

	console.log("✅ Verified!");
	console.log(`https://etherscan.io/address/${contractAddress}`);
}

main().catch((err) => {
	console.error(err);
	process.exit(1);
});
```

---

## Hardhat Ignition Module (Alternative Deploy)

```typescript
// ignition/modules/MyToken.ts
import { buildModule } from "@nomicfoundation/hardhat-ignition/modules";
import { ethers } from "ethers";

const MyTokenModule = buildModule("MyTokenModule", (m) => {
	const initialSupply = m.getParameter(
		"initialSupply",
		ethers.parseEther("100000"),
	);

	const token = m.contract("MyToken", [
		"My Token",
		"MTK",
		m.getAccount(0), // Deployer as initial owner
		initialSupply,
	]);

	return { token };
});

export default MyTokenModule;

// Deploy with Ignition:
// npx hardhat ignition deploy ignition/modules/MyToken.ts --network sepolia
```
