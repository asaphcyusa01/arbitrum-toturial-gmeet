# Getting Started with Your First dApp on Arbitrum

This README provides a detailed, step-by-step guide for beginners to build, deploy, and interact with a decentralized application (dApp) on the Arbitrum blockchain, using modern tools like Hardhat and Ethers v6.

## Table of Contents

 1. Overview
 2. Prerequisites
 3. Environment Setup
 4. Writing a Smart Contract
 5. Deploying to Arbitrum Testnet
 6. Getting the Contract ABI
 7. Interacting with the Smart Contract
 8. Building a Frontend Interface
 9. Running the App Locally
10. Bridging Assets (Optional)
11. Deploying to Mainnet
12. Troubleshooting Common Issues
13. Additional Resources

## 1. Overview

Arbitrum is a Layer 2 scaling solution for Ethereum that enables faster and cheaper transactions while maintaining compatibility with Ethereum smart contracts. In this tutorial, we’ll build a simple "Cupcake Vending Machine" dApp, deploy it to the Arbitrum Sepolia Testnet, and create a React frontend to interact with it.

## 2. Prerequisites

Ensure you have the following installed:

- **Node.js** (v18 or later): Download
- **Yarn** or **npm**: Yarn Install
- **MetaMask** browser extension: Install
- **Hardhat**: For smart contract development and deployment
- A GitHub account (optional, for version control)

## 3. Environment Setup

### a. MetaMask Setup

1. Install MetaMask from https://metamask.io.
2. Create a new wallet or import an existing one.
3. Add the Arbitrum Sepolia Testnet manually:
   - **Network Name**: Arbitrum Sepolia Testnet
   - **RPC URL**: `https://sepolia-rollup.arbitrum.io/rpc`
   - **Chain ID**: `421614`
   - **Currency Symbol**: ETH
   - **Block Explorer URL**: `https://sepolia.arbiscan.io`
4. Obtain Sepolia ETH from: https://cloud.google.com/application/web3/faucet/ethereum/sepolia
5. After receiving the Sepolia ETH, convert them to Arbitrum Sepolia on this website: https://bridge.arbitrum.io/?sourceChain=sepolia&destinationChain=arbitrum-sepolia&tab=bridge

### b. Local Project Setup

```bash
mkdir my-arbitrum-dapp
cd my-arbitrum-dapp
npm init -y
npm install --save-dev hardhat
npx hardhat
```

Choose **Create a JavaScript project** and accept all defaults.

Install dependencies for Ethers v6 and Hardhat:

```bash
npm install --save-dev @nomicfoundation/hardhat-toolbox @nomicfoundation/hardhat-ethers ethers@^6.1.0
npm install dotenv
```

### c. Configure Hardhat

Create or update `hardhat.config.js`:

```javascript
require("@nomicfoundation/hardhat-toolbox");
require("dotenv").config();

module.exports = {
  solidity: "0.8.9",
  networks: {
    arbitrum_sepolia: {
      url: "https://sepolia-rollup.arbitrum.io/rpc",
      accounts: [process.env.PRIVATE_KEY]
    }
  }
};
```

Create a `.env` file in the project root:

```bash
touch .env
```

Add your private key (never commit this to version control):

```
PRIVATE_KEY=your_wallet_private_key
```

> **Security Note**: Add `.env` to `.gitignore` to prevent accidental exposure of your private key.

## 4. Writing a Smart Contract

Create `contracts/VendingMachine.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.9;

contract VendingMachine {
    mapping(address => uint) private cupcakeBalances;
    mapping(address => uint) private lastPurchaseTime;

    function getCupcake() public {
        require(block.timestamp >= lastPurchaseTime[msg.sender] + 5, "Wait 5 seconds between purchases");
        require(cupcakeBalances[msg.sender] == 0, "You already have a cupcake");
        cupcakeBalances[msg.sender]++;
        lastPurchaseTime[msg.sender] = block.timestamp;
    }

    function getBalance() public view returns (uint) {
        return cupcakeBalances[msg.sender];
    }
}

```

Compile the contract:

```bash
npx hardhat compile
```

## 5. Deploying to Arbitrum Testnet

### a. Create a Deployment Script

Create `scripts/deploy.js`:

```javascript
const { ethers } = require("hardhat");
const fs = require("fs");

async function main() {
  const [deployer] = await ethers.getSigners();
  console.log("Deploying contracts with account:", deployer.address);

  const VendingMachine = await ethers.getContractFactory("VendingMachine");
  const vendingMachine = await VendingMachine.deploy();
  await vendingMachine.waitForDeployment();
  console.log("VendingMachine deployed to:", vendingMachine.target);

  // Save contract address and ABI for frontend
  const contractsDir = __dirname + "/../frontend/src/contracts";
  if (!fs.existsSync(contractsDir)) {
    fs.mkdirSync(contractsDir, { recursive: true });
  }

  fs.writeFileSync(
    contractsDir + "/VendingMachine-address.json",
    JSON.stringify({ address: vendingMachine.target }, null, 2)
  );

  const artifact = await hre.artifacts.readArtifact("VendingMachine");
  fs.writeFileSync(
    contractsDir + "/VendingMachine.json",
    JSON.stringify(artifact.abi, null, 2)
  );
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1);
});
```

### b. Deploy to Arbitrum Sepolia

Ensure your wallet has test ETH, then run:

```bash
npx hardhat run scripts/deploy.js --network arbitrum_sepolia
```

The console will display the deployed contract address, e.g.:

```
VendingMachine deployed to: 0xYourContractAddress
```

## 6. Getting the Contract ABI

The ABI is generated automatically when you compile your contract. To use it in the frontend:

1. Compile the contract:

   ```bash
   npx hardhat compile
   ```

2. Find the ABI in:

   ```
   artifacts/contracts/VendingMachine.sol/VendingMachine.json
   ```

3. The deployment script above automatically saves the ABI to `frontend/src/contracts/VendingMachine.json`. If needed, manually copy the `abi` field from `VendingMachine.json` into `frontend/src/contracts/VendingMachine.json`.

Example ABI:

```json
[
  {
    "inputs": [],
    "name": "getBalance",
    "outputs": [
      {
        "internalType": "uint256",
        "name": "",
        "type": "uint256"
      }
    ],
    "stateMutability": "view",
    "type": "function"
  },
  {
    "inputs": [],
    "name": "getCupcake",
    "outputs": [],
    "stateMutability": "nonpayable",
    "type": "function"
  }
]
```

## 7. Interacting with the Smart Contract

Create `scripts/interact.js` to test contract interaction:

```javascript
const { ethers } = require("hardhat");

async function main() {
  const contractAddress = "YOUR_DEPLOYED_CONTRACT_ADDRESS"; // Replace with address from deployment
  const VendingMachine = await ethers.getContractAt("VendingMachine", contractAddress);
  const tx = await VendingMachine.getCupcake();
  await tx.wait();
  const balance = await VendingMachine.getBalance();
  console.log("Cupcake Balance:", balance.toString());
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1);
});
```

Run the script:

```bash
npx hardhat run scripts/interact.js --network arbitrum_sepolia
```

## 8. Building a Frontend Interface

### a. Initialize React App

```bash
npx create-react-app frontend
cd frontend
npm install ethers
```

### b. Update Frontend Code

Replace `frontend/src/App.js` with:

```javascript
import { ethers } from "ethers";
import { useState } from "react";
import VendingMachineABI from "./contracts/VendingMachine.json";
import VendingMachineAddress from "./contracts/VendingMachine-address.json";

function App() {
  const [balance, setBalance] = useState("0");

  async function getCupcake() {
    if (!window.ethereum) {
      alert("Please install MetaMask!");
      return;
    }

    await window.ethereum.request({ method: "eth_requestAccounts" });
    const provider = new ethers.BrowserProvider(window.ethereum);
    const signer = await provider.getSigner();
    const contract = new ethers.Contract(
      VendingMachineAddress.address,
      VendingMachineABI,
      signer
    );

    try {
      const tx = await contract.getCupcake();
      await tx.wait();
      const bal = await contract.getBalance();
      setBalance(bal.toString());
    } catch (error) {
      console.error("Error:", error);
      alert("Failed to get cupcake. Check console for details.");
    }
  }

  return (
    <div>
      <h1>Cupcake Vending Machine</h1>
      <button onClick={getCupcake}>Get Cupcake</button>
      <p>Your Cupcake Balance: {balance}</p>
    </div>
  );
}

export default App;
```

Ensure `frontend/src/contracts/VendingMachine.json` and `frontend/src/contracts/VendingMachine-address.json` exist (created during deployment).

## 9. Running the App Locally

1. Navigate to the frontend directory:

   ```bash
   cd frontend
   ```

2. Install dependencies (if not already done):

   ```bash
   npm install
   ```

3. Start the React development server:

   ```bash
   npm start
   ```

4. Open `http://localhost:3000` in your browser.

5. Ensure MetaMask is connected to the Arbitrum Sepolia Testnet and has test ETH.

6. Click "Get Cupcake" to interact with the contract.

## 10. Bridging Assets (Optional)

If you need to transfer ETH to Arbitrum Sepolia for testing:

1. Visit the Arbitrum Bridge.
2. Connect your MetaMask wallet.
3. Bridge test ETH from Ethereum Sepolia to Arbitrum Sepolia.

## 11. Deploying to Mainnet

1. Update `hardhat.config.js` with Arbitrum Mainnet details:

   ```javascript
   arbitrum_mainnet: {
     url: "https://arb1.arbitrum.io/rpc",
     accounts: [process.env.PRIVATE_KEY]
   }
   ```

2. Fund your wallet with mainnet ETH.

3. Deploy the contract:

   ```bash
   npx hardhat run scripts/deploy.js --network arbitrum_mainnet
   ```

4. Verify the contract on Arbiscan.

## 12. Troubleshooting Common Issues

### a. Yarn DNS Error (EAI_AGAIN)

**Error**: `getaddrinfo EAI_AGAIN registry.yarnpkg.com`**Fix**:

- Check your internet connection.

- Use npm instead:

  ```bash
  npm install --save-dev @nomicfoundation/hardhat-toolbox ethers
  ```

- Or configure Yarn to use a different registry:

  ```bash
  yarn config set registry https://registry.npmjs.org
  ```

### b. Solidity Version Mismatch (HH606)

**Error**: `The Solidity version pragma statement doesn't match...`**Fix**: Ensure `hardhat.config.js` matches the contract’s `pragma` version (e.g., `0.8.9`).

### c. Dependency Conflict

**Error**: `Could not resolve dependency...`**Fix**: Use compatible versions:

```bash
npm install --save-dev @nomicfoundation/hardhat-toolbox @nomicfoundation/hardhat-ethers ethers@^6.1.0
```

### d. Deployed Function Error

**Error**: `TypeError: vendingMachine.deployed is not a function`**Fix**: Use Ethers v6 syntax in `scripts/deploy.js`:

```javascript
await vendingMachine.waitForDeployment();
console.log("VendingMachine deployed to:", vendingMachine.target);
```

### e. Invalid ENS Name Error

**Error**: `invalid ENS name (argument="name", value="PASTE_DEPLOYED_ADDRESS_HERE")`**Fix**: Replace `PASTE_DEPLOYED_ADDRESS_HERE` in `App.js` or `VendingMachine-address.json` with the actual contract address from the deployment output.

### f. Missing ABI

**Error**: Frontend cannot interact due to missing ABI. **Fix**: Ensure `frontend/src/contracts/VendingMachine.json` contains the correct ABI (see Getting the Contract ABI).

### g. Finding a Lost Contract Address

If you closed the terminal and lost the contract address:

1. Re-run the deployment:

   ```bash
   npx hardhat run scripts/deploy.js --network arbitrum_sepolia
   ```

2. Check `frontend/src/contracts/VendingMachine-address.json` for the saved address.
