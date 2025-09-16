# FHEVM Hardhat Template

A Hardhat-based template for developing Fully Homomorphic Encryption (FHE) enabled Solidity smart contracts using the
FHEVM protocol by Zama.

## Quick Start

Go to : https://github.com/zama-ai/fhevm-hardhat-template

Click Use this template -> Creat an new repository


### Setup node 

```bash
node -v
npm -v
```

### Installation

1. **Install dependencies**

   ```bash
   npm install
   ```

2. **Set up environment variables**

- Creat folder .env
- Go to : " https://blastapi.io/chains/ethereum " copy RPC
- Go to : " https://etherscan.io/apidashboard " Copy API Key
- Copy Prive Key metamask or OKX wallet

   ```bash
  SEPOLIA_RPC_URL= RPC
  PRIVATE_KEY= Private Key
  ETHERSCAN_API_KEY= API KEY 
   ```
3. **Write a simple contract**
 - Creat folder " Contracts/Counter.Sol "
   ```bash
   // SPDX-License-Identifier: MIT
   pragma solidity ^0.8.24;

   /// @title A simple counter contract
   contract Counter {
   uint32 private _count;

   /// @notice Returns the current count
   function getCount() external view returns (uint32) {
    return _count;
   }

   /// @notice Increments the counter by a specific value
   function increment(uint32 value) external {
    _count += value;
   }

   /// @notice Decrements the counter by a specific value
   function decrement(uint32 value) external {
    require(_count >= value, "Counter: cannot decrement below zero");
    _count -= value;
   }
   }
   ```

4. **Run**
 ```bash
 npx hardhat compile
 ```
5. **Set up the testing environment**
- Create a test script " test/Counter.ts "
 ```bash
import { FHECounter, FHECounter__factory } from "../types";
import { FhevmType } from "@fhevm/hardhat-plugin";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { expect } from "chai";
import { ethers, fhevm } from "hardhat";

type Signers = {
  deployer: HardhatEthersSigner;
  alice: HardhatEthersSigner;
  bob: HardhatEthersSigner;
};

async function deployFixture() {
  const factory = (await ethers.getContractFactory("FHECounter")) as FHECounter__factory;
  const fheCounterContract = (await factory.deploy()) as FHECounter;
  const fheCounterContractAddress = await fheCounterContract.getAddress();

  return { fheCounterContract, fheCounterContractAddress };
}

describe("FHECounter", function () {
  let signers: Signers;
  let fheCounterContract: FHECounter;
  let fheCounterContractAddress: string;

  before(async function () {
    const ethSigners: HardhatEthersSigner[] = await ethers.getSigners();
    signers = { deployer: ethSigners[0], alice: ethSigners[1], bob: ethSigners[2] };
  });

  beforeEach(async () => {
    ({ fheCounterContract, fheCounterContractAddress } = await deployFixture());
  });

  it("should be deployed", async function () {
    console.log(`FHECounter has been deployed at address ${fheCounterContractAddress}`);
    // Test the deployed address is valid
    expect(ethers.isAddress(fheCounterContractAddress)).to.eq(true);
  });

  it("encrypted count should be uninitialized after deployment", async function () {
  const encryptedCount = await fheCounterContract.getCount();
  // Expect initial count to be bytes32(0) after deployment,
  // (meaning the encrypted count value is uninitialized)
  expect(encryptedCount).to.eq(ethers.ZeroHash);
  });

  it("increment the counter by 1", async function () {
  const encryptedCountBeforeInc = await fheCounterContract.getCount();
  expect(encryptedCountBeforeInc).to.eq(ethers.ZeroHash);
  const clearCountBeforeInc = 0;

  // Encrypt constant 1 as a euint32
  const clearOne = 1;
  const encryptedOne = await fhevm
    .createEncryptedInput(fheCounterContractAddress, signers.alice.address)
    .add32(clearOne)
    .encrypt();

   const tx = await fheCounterContract.connect(signers.alice).increment(encryptedOne.handles[0], encryptedOne.inputProof);
   await tx.wait();
  const encryptedCountAfterInc = await fheCounterContract.getCount();
  const clearCountAfterInc = await fhevm.userDecryptEuint(
  FhevmType.euint32,
  encryptedCountAfterInc,
  fheCounterContractAddress,
  signers.alice,
 );
expect(clearCountAfterInc).to.eq(clearCountBeforeInc + clearOne);

  it("decrement the counter by 1", async function () {
  // Encrypt constant 1 as a euint32
  const clearOne = 1;
  const encryptedOne = await fhevm
    .createEncryptedInput(fheCounterContractAddress, signers.alice.address)
    .add32(clearOne)
    .encrypt();

  // First increment by 1, count becomes 1
  let tx = await fheCounterContract.connect(signers.alice).increment(encryptedOne.handles[0], encryptedOne.inputProof);
  await tx.wait();

  // Then decrement by 1, count goes back to 0
  tx = await fheCounterContract.connect(signers.alice).decrement(encryptedOne.handles[0], encryptedOne.inputProof);
  await tx.wait();

  const encryptedCountAfterDec = await fheCounterContract.getCount();
  const clearCountAfterDec = await fhevm.userDecryptEuint(
    FhevmType.euint32,
    encryptedCountAfterDec,
    fheCounterContractAddress,
    signers.alice,
  );

  expect(clearCountAfterDec).to.eq(0);
  });
});
})
```
6. **Run**
```bash
npx hardhat test
```
7. **Edit folder " contracts/FHECounter.sol "**
```bash
/// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import { FHE, euint32, externalEuint32 } from "@fhevm/solidity/lib/FHE.sol";
import { SepoliaConfig } from "@fhevm/solidity/config/ZamaConfig.sol";

/// @title A simple FHE counter contract
contract FHECounter is SepoliaConfig {
  euint32 private _count;

  /// @notice Returns the current count
  function getCount() external view returns (euint32) {
    return _count;
  }

   /// @notice Increments the counter by a specific value
function increment(externalEuint32 inputEuint32, bytes calldata inputProof) external {
  euint32 evalue = FHE.fromExternal(inputEuint32, inputProof);
  _count = FHE.add(_count, evalue);

  FHE.allowThis(_count);
  FHE.allow(_count, msg.sender);
}

/// @notice Decrements the counter by a specific value
/// @dev This example omits overflow/underflow checks for simplicity and readability.
/// In a production contract, proper range checks should be implemented.
function decrement(externalEuint32 inputEuint32, bytes calldata inputProof) external {
  euint32 encryptedEuint32 = FHE.fromExternal(inputEuint32, inputProof);

  _count = FHE.sub(_count, encryptedEuint32);

  FHE.allowThis(_count);
  FHE.allow(_count, msg.sender);
}
}
```
8. **Run**
```bash
npx hardhat compile -> enter
```
9. **Edit folder " test/FHECounter.ts "**
```bash
import { FHECounter, FHECounter__factory } from "../types";
import { FhevmType } from "@fhevm/hardhat-plugin";
import { HardhatEthersSigner } from "@nomicfoundation/hardhat-ethers/signers";
import { expect } from "chai";
import { ethers, fhevm } from "hardhat";

type Signers = {
  deployer: HardhatEthersSigner;
  alice: HardhatEthersSigner;
  bob: HardhatEthersSigner;
};

async function deployFixture() {
  const factory = (await ethers.getContractFactory("FHECounter")) as FHECounter__factory;
  const fheCounterContract = (await factory.deploy()) as FHECounter;
  const fheCounterContractAddress = await fheCounterContract.getAddress();

  return { fheCounterContract, fheCounterContractAddress };
}

describe("FHECounter", function () {
  let signers: Signers;
  let fheCounterContract: FHECounter;
  let fheCounterContractAddress: string;

  before(async function () {
    const ethSigners: HardhatEthersSigner[] = await ethers.getSigners();
    signers = { deployer: ethSigners[0], alice: ethSigners[1], bob: ethSigners[2] };
  });

  beforeEach(async () => {
    ({ fheCounterContract, fheCounterContractAddress } = await deployFixture());
  });

  it("should be deployed", async function () {
    console.log(`FHECounter has been deployed at address ${fheCounterContractAddress}`);
    // Test the deployed address is valid
    expect(ethers.isAddress(fheCounterContractAddress)).to.eq(true);
  });

  it("encrypted count should be uninitialized after deployment", async function () {
  const encryptedCount = await fheCounterContract.getCount();
  // Expect initial count to be bytes32(0) after deployment,
  // (meaning the encrypted count value is uninitialized)
  expect(encryptedCount).to.eq(ethers.ZeroHash);
  });

  it("increment the counter by 1", async function () {
  const encryptedCountBeforeInc = await fheCounterContract.getCount();
  expect(encryptedCountBeforeInc).to.eq(ethers.ZeroHash);
  const clearCountBeforeInc = 0;

  // Encrypt constant 1 as a euint32
  const clearOne = 1;
  const encryptedOne = await fhevm
    .createEncryptedInput(fheCounterContractAddress, signers.alice.address)
    .add32(clearOne)
    .encrypt();

  const tx = await fheCounterContract.connect(signers.alice).increment(encryptedOne.handles[0], encryptedOne.inputProof);
  await tx.wait();
  const encryptedCountAfterInc = await fheCounterContract.getCount();
  const clearCountAfterInc = await fhevm.userDecryptEuint(
  FhevmType.euint32,
  encryptedCountAfterInc,
  fheCounterContractAddress,
  signers.alice,
  );
 expect(clearCountAfterInc).to.eq(clearCountBeforeInc + clearOne);

  it("decrement the counter by 1", async function () {
  // Encrypt constant 1 as a euint32
  const clearOne = 1;
  const encryptedOne = await fhevm
    .createEncryptedInput(fheCounterContractAddress, signers.alice.address)
    .add32(clearOne)
    .encrypt();

  // First increment by 1, count becomes 1
  let tx = await fheCounterContract.connect(signers.alice).increment(encryptedOne.handles[0], encryptedOne.inputProof);
  await tx.wait();

  // Then decrement by 1, count goes back to 0
  tx = await fheCounterContract.connect(signers.alice).decrement(encryptedOne.handles[0], encryptedOne.inputProof);
  await tx.wait();

  const encryptedCountAfterDec = await fheCounterContract.getCount();
  const clearCountAfterDec = await fhevm.userDecryptEuint(
    FhevmType.euint32,
    encryptedCountAfterDec,
    fheCounterContractAddress,
    signers.alice,
  );

  expect(clearCountAfterDec).to.eq(0);
  });
  });
});
```
10. **Run**
```bash
npx hardhat test
``` 
11. **Deploy to local network**
    
   ```bash
   npx hardhat test --network hardhat
   ```
```bash
npx hardhat node
```
12. **Deploy to Sepolia Testnet**
 - Open a new terminal window
```bash
npx hardhat test --network localhost
```
```bash
npx hardhat deploy --network localhost
```
```bash
npx hardhat --network localhost task:decrypt-count
npx hardhat --network localhost task:increment --value 1
npx hardhat --network localhost task:decrypt-count
```

13. **Edit folder : " hardhat.config.ts "**
```bash
import "@fhevm/hardhat-plugin";
import "@nomicfoundation/hardhat-chai-matchers";
import "@nomicfoundation/hardhat-ethers";
import "@nomicfoundation/hardhat-verify";
import "@typechain/hardhat";
import "hardhat-deploy";
import "hardhat-gas-reporter";
import type { HardhatUserConfig } from "hardhat/config";
import * as dotenv from "dotenv";
import "solidity-coverage";

// load .env
dotenv.config();

const SEPOLIA_RPC_URL: string =
  process.env.SEPOLIA_RPC_URL || "https://eth-sepolia.public.blastapi.io";
const PRIVATE_KEY: string = process.env.PRIVATE_KEY || "";
const ETHERSCAN_API_KEY: string = process.env.ETHERSCAN_API_KEY || "";

const config: HardhatUserConfig = {
  defaultNetwork: "hardhat",
  namedAccounts: {
    deployer: 0,
  },
  etherscan: {
    apiKey: {
      sepolia: ETHERSCAN_API_KEY,
    },
  },
  gasReporter: {
    currency: "USD",
    enabled: process.env.REPORT_GAS ? true : false,
    excludeContracts: [],
  },
  networks: {
    hardhat: {
      chainId: 31337,
    },
    anvil: {
      url: "http://localhost:8545",
      chainId: 31337,
    },
    sepolia: {
      url: SEPOLIA_RPC_URL,
      chainId: 11155111,
      accounts: PRIVATE_KEY ? [PRIVATE_KEY] : [],
    },
  },
  paths: {
    artifacts: "./artifacts",
    cache: "./cache",
    sources: "./contracts",
    tests: "./test",
  },
  solidity: {
    version: "0.8.27",
    settings: {
      metadata: {
        bytecodeHash: "none",
      },
      optimizer: {
        enabled: true,
        runs: 800,
      },
      evmVersion: "cancun",
    },
  },
  typechain: {
    outDir: "types",
    target: "ethers-v6",
  },
};

export default config;
```

14. **Run on Sepolia Ethereum Testnet**
  - Ensure that your wallet has ETH Sepolia
```bash
npx hardhat clean
npx hardhat compile --network sepolia
```
```bash
npx hardhat deploy --network sepolia
```
```bash
npx hardhat verify --network sepolia < contract deployed >
```

## 📁 Project Structure

```
fhevm-hardhat-template/
├── contracts/           # Smart contract source files
│   └── FHECounter.sol   # Example FHE counter contract
├── deploy/              # Deployment scripts
├── tasks/               # Hardhat custom tasks
├── test/                # Test files
├── hardhat.config.ts    # Hardhat configuration
└── package.json         # Dependencies and scripts
```

## 📜 Available Scripts

| Script             | Description              |
| ------------------ | ------------------------ |
| `npm run compile`  | Compile all contracts    |
| `npm run test`     | Run all tests            |
| `npm run coverage` | Generate coverage report |
| `npm run lint`     | Run linting checks       |
| `npm run clean`    | Clean build artifacts    |

## 📚 Documentation

- [FHEVM Documentation](https://docs.zama.ai/fhevm)
- [FHEVM Hardhat Setup Guide](https://docs.zama.ai/protocol/solidity-guides/getting-started/setup)
- [FHEVM Testing Guide](https://docs.zama.ai/protocol/solidity-guides/development-guide/hardhat/write_test)
- [FHEVM Hardhat Plugin](https://docs.zama.ai/protocol/solidity-guides/development-guide/hardhat)

## 📄 License

This project is licensed under the BSD-3-Clause-Clear License. See the [LICENSE](LICENSE) file for details.

## 🆘 Support

- **GitHub Issues**: [Report bugs or request features](https://github.com/zama-ai/fhevm/issues)
- **Documentation**: [FHEVM Docs](https://docs.zama.ai)
- **Community**: [Zama Discord](https://discord.gg/zama)

---

**Built with ❤️ by Eternal**
