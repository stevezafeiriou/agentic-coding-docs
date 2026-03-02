# Hardhat Configuration, Package Setup & TypeScript Config

---

## package.json

```json
{
	"name": "my-contract",
	"version": "1.0.0",
	"private": true,
	"scripts": {
		"compile": "hardhat compile",
		"test": "hardhat test",
		"test:coverage": "hardhat coverage",
		"test:gas": "REPORT_GAS=true hardhat test",
		"deploy:sepolia": "hardhat run scripts/deploy.ts --network sepolia",
		"deploy:mainnet": "hardhat run scripts/deploy.ts --network mainnet",
		"verify": "hardhat run scripts/verify.ts --network sepolia",
		"lint": "solhint 'contracts/**/*.sol'",
		"clean": "hardhat clean",
		"node": "hardhat node",
		"typechain": "hardhat typechain"
	},
	"devDependencies": {
		"@nomicfoundation/hardhat-toolbox": "^5.0.0",
		"@nomicfoundation/hardhat-ethers": "^3.0.0",
		"@nomicfoundation/hardhat-chai-matchers": "^2.0.0",
		"@nomicfoundation/hardhat-network-helpers": "^1.0.0",
		"@typechain/ethers-v6": "^0.5.0",
		"@typechain/hardhat": "^9.0.0",
		"@types/chai": "^4.3.0",
		"@types/mocha": "^10.0.0",
		"@types/node": "^20.0.0",
		"chai": "^4.4.0",
		"hardhat": "^2.22.0",
		"hardhat-gas-reporter": "^1.0.10",
		"solidity-coverage": "^0.8.12",
		"solhint": "^5.0.0",
		"ts-node": "^10.9.0",
		"typechain": "^8.3.0",
		"typescript": "^5.3.0"
	},
	"dependencies": {
		"@openzeppelin/contracts": "^5.0.0",
		"ethers": "^6.11.0",
		"dotenv": "^16.4.0"
	}
}
```

---

## hardhat.config.ts

```typescript
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
import "hardhat-gas-reporter";
import "solidity-coverage";
import * as dotenv from "dotenv";

dotenv.config();

// Validate required env vars at startup
function getEnvVar(name: string, required = true): string {
	const value = process.env[name];
	if (required && !value) {
		throw new Error(`Missing required environment variable: ${name}`);
	}
	return value ?? "";
}

const config: HardhatUserConfig = {
	solidity: {
		version: "0.8.24",
		settings: {
			optimizer: {
				enabled: true,
				runs: 200, // 200 = balanced deploy cost vs call cost
				// Use 1 if deploy cost matters more
				// Use 1000000 if call cost matters more
			},
			viaIR: false, // Set true only if needed for complex contracts
			evmVersion: "paris", // Conservative — compatible with all EVM chains
		},
	},

	networks: {
		// Local development — Hardhat's built-in network
		hardhat: {
			chainId: 31337,
			// Fork mainnet for testing with real protocol state:
			// forking: {
			//   url: getEnvVar("MAINNET_RPC_URL"),
			//   blockNumber: 19000000, // Pin block for deterministic tests
			// },
		},

		localhost: {
			url: "http://127.0.0.1:8545",
			chainId: 31337,
		},

		// Sepolia testnet
		sepolia: {
			url: getEnvVar("SEPOLIA_RPC_URL", false) || "https://rpc.sepolia.org",
			accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
			chainId: 11155111,
			gasPrice: "auto",
		},

		// Ethereum mainnet
		mainnet: {
			url: getEnvVar("MAINNET_RPC_URL", false) || "https://cloudflare-eth.com",
			accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
			chainId: 1,
			gasPrice: "auto",
		},
	},

	// Etherscan verification — get API key at etherscan.io
	etherscan: {
		apiKey: {
			mainnet: process.env.ETHERSCAN_API_KEY ?? "",
			sepolia: process.env.ETHERSCAN_API_KEY ?? "",
		},
	},

	// Gas reporter — shows gas costs per function in test output
	gasReporter: {
		enabled: process.env.REPORT_GAS === "true",
		currency: "USD",
		coinmarketcap: process.env.COINMARKETCAP_API_KEY, // Optional: shows USD costs
		excludeContracts: ["mocks/", "test/"],
		src: "./contracts",
	},

	// TypeChain — generates TypeScript types from ABI
	typechain: {
		outDir: "typechain-types",
		target: "ethers-v6",
	},

	// Test configuration
	mocha: {
		timeout: 60000, // 60s — needed for forked mainnet tests
	},

	// Paths
	paths: {
		sources: "./contracts",
		tests: "./test",
		cache: "./cache",
		artifacts: "./artifacts",
	},
};

export default config;
```

---

## tsconfig.json

```json
{
	"compilerOptions": {
		"target": "ES2020",
		"module": "commonjs",
		"moduleResolution": "node",
		"strict": true,
		"esModuleInterop": true,
		"outDir": "dist",
		"declaration": true,
		"resolveJsonModule": true,
		"skipLibCheck": true,
		"forceConsistentCasingInFileNames": true
	},
	"include": [
		"hardhat.config.ts",
		"scripts/**/*.ts",
		"test/**/*.ts",
		"typechain-types/**/*.ts"
	],
	"files": ["./hardhat.config.ts"]
}
```

---

## .env.example (Always Commit This File)

```bash
# .env.example — copy to .env and fill in values
# NEVER commit the actual .env file

# Your wallet private key (deployer address)
# SECURITY: Use a dedicated deployer wallet — never your main wallet
# Fund it with just enough ETH for deployment
PRIVATE_KEY=your_private_key_here_without_0x_prefix

# RPC URLs — get from Alchemy (alchemy.com) or Infura (infura.io) — free tier available
SEPOLIA_RPC_URL=https://eth-sepolia.g.alchemy.com/v2/YOUR_API_KEY
MAINNET_RPC_URL=https://eth-mainnet.g.alchemy.com/v2/YOUR_API_KEY

# Etherscan API key — get free at etherscan.io/apis
ETHERSCAN_API_KEY=your_etherscan_api_key

# Optional: CoinMarketCap key for USD gas estimates
COINMARKETCAP_API_KEY=your_coinmarketcap_api_key
```

---

## .gitignore

```gitignore
node_modules/
artifacts/
cache/
typechain-types/
coverage/
coverage.json
.env
*.env.local
dist/

# Never commit private keys or secrets
.secret
keystore/
```

---

## solhint.json (Solidity Linter Config)

```json
{
	"extends": "solhint:recommended",
	"rules": {
		"compiler-version": ["error", "0.8.24"],
		"avoid-suicide": "error",
		"avoid-sha3": "error",
		"no-empty-blocks": "warn",
		"no-unused-vars": "error",
		"func-visibility": ["error", { "ignoreConstructors": true }],
		"state-visibility": "error",
		"not-rely-on-time": "warn",
		"reentrancy": "error",
		"mark-callable-contracts": "warn",
		"no-complex-fallback": "warn",
		"max-line-length": ["warn", 120],
		"ordering": "warn",
		"reason-string": ["warn", { "maxLength": 64 }]
	}
}
```
