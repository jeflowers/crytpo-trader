# Step-by-Step Guide: Testing Synthetic Stock Trading with Crypto

## 1. Set Up a Test Environment

### Install MetaMask and Configure Test Network
1. Install the MetaMask browser extension from the official website
2. Create a new wallet or import an existing one
3. Add Binance Smart Chain Testnet to MetaMask:
   - Network Name: BSC Testnet
   - RPC URL: https://data-seed-prebsc-1-s1.binance.org:8545/
   - Chain ID: 97
   - Symbol: tBNB
   - Block Explorer: https://testnet.bscscan.com

### Get Test Tokens
1. Get test BNB (tBNB) from a BSC Testnet faucet:
   - Visit https://testnet.binance.org/faucet-smart
   - Enter your wallet address and request test tokens
2. Get test ETH for the Testnet:
   - You can use a Binance Smart Chain testnet ETH faucet or bridge test ETH from an Ethereum testnet

## 2. Set Up Development Environment

### Basic Project Structure
1. Create a new project folder:
```bash
mkdir synthetic-stock-testing
cd synthetic-stock-testing
```

2. Initialize a Node.js project:
```bash
npm init -y
```

3. Install required dependencies:
```bash
npm install ethers web3 @openzeppelin/contracts hardhat @nomiclabs/hardhat-ethers dotenv
```

4. Create a `.env` file for your private keys and endpoints:
```
PRIVATE_KEY=your_metamask_private_key
BSC_TESTNET_URL=https://data-seed-prebsc-1-s1.binance.org:8545/
```

5. Set up Hardhat for smart contract development:
```bash
npx hardhat init
```
Select "Create a JavaScript project" when prompted.

## 3. Create Mock Smart Contracts

### Create a Mock Synthetic Asset Contract

Create a file `contracts/MockSyntheticApple.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MockSyntheticApple is ERC20, Ownable {
    // Mock price feed for Apple stock
    uint256 public applePrice; // Price in USD with 8 decimals (e.g., 17500000000 = $175.00)
    
    constructor() ERC20("Mock Synthetic Apple", "mAAPL") {
        // Initial price set to $175.00
        applePrice = 17500000000;
    }
    
    // Function to mint tokens (representing buying synthetic Apple)
    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
    
    // Function to update Apple stock price (would be done by oracles in real implementation)
    function updatePrice(uint256 newPrice) external onlyOwner {
        applePrice = newPrice;
    }
    
    // Function to get the current price
    function getApplePrice() external view returns (uint256) {
        return applePrice;
    }
}
```

### Create a Mock Stablecoin Contract

Create a file `contracts/MockUSDT.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MockUSDT is ERC20, Ownable {
    constructor() ERC20("Mock USDT", "mUSDT") {}
    
    // Function to mint test tokens
    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
}
```

### Create a Mock Exchange Contract

Create a file `contracts/MockExchange.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "./MockSyntheticApple.sol";

contract MockExchange is Ownable {
    MockSyntheticApple public mAAPL;
    IERC20 public USDT;
    
    // Fee percentage (0.1%)
    uint256 public feePercentage = 10; // 10 basis points
    
    // Exchange rate: 1 mAAPL = applePrice USDT
    
    constructor(address _mAAPL, address _USDT) {
        mAAPL = MockSyntheticApple(_mAAPL);
        USDT = IERC20(_USDT);
    }
    
    // Buy mAAPL with USDT
    function buyApple(uint256 usdtAmount) external {
        // Calculate how many mAAPL tokens to give
        uint256 applePrice = mAAPL.getApplePrice();
        uint256 fee = (usdtAmount * feePercentage) / 10000;
        uint256 usdtAfterFee = usdtAmount - fee;
        
        // Calculate amount of mAAPL to mint (considering 8 decimal places for price)
        uint256 mAAPLAmount = (usdtAfterFee * 1e18 * 1e8) / applePrice;
        
        // Transfer USDT from user to this contract
        require(USDT.transferFrom(msg.sender, address(this), usdtAmount), "USDT transfer failed");
        
        // Mint mAAPL tokens to user
        mAAPL.mint(msg.sender, mAAPLAmount);
    }
    
    // Sell mAAPL for USDT
    function sellApple(uint256 mAAPLAmount) external {
        // Calculate how many USDT tokens to give
        uint256 applePrice = mAAPL.getApplePrice();
        uint256 usdtBeforeFee = (mAAPLAmount * applePrice) / (1e18 * 1e8);
        uint256 fee = (usdtBeforeFee * feePercentage) / 10000;
        uint256 usdtToReturn = usdtBeforeFee - fee;
        
        // Burn mAAPL tokens from user
        mAAPL.transferFrom(msg.sender, address(this), mAAPLAmount);
        
        // Transfer USDT to user
        require(USDT.transfer(msg.sender, usdtToReturn), "USDT transfer failed");
    }
    
    // Update fee percentage
    function updateFeePercentage(uint256 _feePercentage) external onlyOwner {
        feePercentage = _feePercentage;
    }
}
```

## 4. Deploy Scripts

Create a file `scripts/deploy.js`:

```javascript
const hre = require("hardhat");

async function main() {
  // Deploy mock USDT
  const MockUSDT = await hre.ethers.getContractFactory("MockUSDT");
  const mockUSDT = await MockUSDT.deploy();
  await mockUSDT.deployed();
  console.log("MockUSDT deployed to:", mockUSDT.address);

  // Deploy mock synthetic Apple
  const MockSyntheticApple = await hre.ethers.getContractFactory("MockSyntheticApple");
  const mockSyntheticApple = await MockSyntheticApple.deploy();
  await mockSyntheticApple.deployed();
  console.log("MockSyntheticApple deployed to:", mockSyntheticApple.address);

  // Deploy mock exchange
  const MockExchange = await hre.ethers.getContractFactory("MockExchange");
  const mockExchange = await MockExchange.deploy(
    mockSyntheticApple.address,
    mockUSDT.address
  );
  await mockExchange.deployed();
  console.log("MockExchange deployed to:", mockExchange.address);

  // Grant minter role to exchange for mAAPL
  const MINTER_ROLE = ethers.utils.keccak256(ethers.utils.toUtf8Bytes("MINTER_ROLE"));
  await mockSyntheticApple.transferOwnership(mockExchange.address);
  console.log("Transferred mAAPL ownership to exchange");

  // Mint some test USDT to deployer
  const [deployer] = await ethers.getSigners();
  const mintAmount = ethers.utils.parseEther("15000"); // 15,000 USDT
  await mockUSDT.mint(deployer.address, mintAmount);
  console.log("Minted 15,000 USDT to:", deployer.address);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

## 5. Configure Hardhat for BSC Testnet

Update `hardhat.config.js`:

```javascript
require("@nomiclabs/hardhat-ethers");
require("dotenv").config();

/**
 * @type import('hardhat/config').HardhatUserConfig
 */
module.exports = {
  solidity: "0.8.4",
  networks: {
    bscTestnet: {
      url: process.env.BSC_TESTNET_URL || "https://data-seed-prebsc-1-s1.binance.org:8545/",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
    },
    localhost: {
      url: "http://127.0.0.1:8545"
    },
  },
};
```

## 6. Testing Scripts

Create a file `scripts/test-trade.js`:

```javascript
const { ethers } = require("hardhat");

async function main() {
  const [trader] = await ethers.getSigners();
  
  // Replace with your deployed contract addresses
  const mockUSDTAddress = "YOUR_DEPLOYED_USDT_ADDRESS";
  const mockSyntheticAppleAddress = "YOUR_DEPLOYED_MAPPLE_ADDRESS";
  const mockExchangeAddress = "YOUR_DEPLOYED_EXCHANGE_ADDRESS";
  
  // Get contract instances
  const mockUSDT = await ethers.getContractAt("MockUSDT", mockUSDTAddress);
  const mockSyntheticApple = await ethers.getContractAt("MockSyntheticApple", mockSyntheticAppleAddress);
  const mockExchange = await ethers.getContractAt("MockExchange", mockExchangeAddress);
  
  // Check USDT balance
  const usdtBalance = await mockUSDT.balanceOf(trader.address);
  console.log("USDT Balance:", ethers.utils.formatEther(usdtBalance));
  
  // Approve exchange to spend USDT
  const approveAmount = ethers.utils.parseEther("5000"); // Approve 5,000 USDT
  await mockUSDT.approve(mockExchangeAddress, approveAmount);
  console.log("Approved exchange to spend 5,000 USDT");
  
  // Buy synthetic Apple stock
  const buyAmount = ethers.utils.parseEther("1000"); // Buy with 1,000 USDT
  await mockExchange.buyApple(buyAmount);
  console.log("Bought synthetic Apple with 1,000 USDT");
  
  // Check mAAPL balance
  const mAAPLBalance = await mockSyntheticApple.balanceOf(trader.address);
  console.log("mAAPL Balance:", ethers.utils.formatEther(mAAPLBalance));
  
  // Update Apple price (simulate price movement)
  // Assuming we have access to update the price
  const newPrice = 18000000000; // $180.00
  await mockSyntheticApple.updatePrice(newPrice);
  console.log("Updated Apple price to $180.00");
  
  // Approve exchange to spend mAAPL for selling
  await mockSyntheticApple.approve(mockExchangeAddress, mAAPLBalance);
  console.log("Approved exchange to spend mAAPL");
  
  // Sell half of mAAPL holdings
  const sellAmount = mAAPLBalance.div(2);
  await mockExchange.sellApple(sellAmount);
  console.log("Sold half of mAAPL holdings");
  
  // Check final balances
  const finalUSDTBalance = await mockUSDT.balanceOf(trader.address);
  const finalMAPPLBalance = await mockSyntheticApple.balanceOf(trader.address);
  
  console.log("Final USDT Balance:", ethers.utils.formatEther(finalUSDTBalance));
  console.log("Final mAAPL Balance:", ethers.utils.formatEther(finalMAPPLBalance));
  
  // Calculate profit/loss
  const usdtDifference = finalUSDTBalance.sub(usdtBalance.sub(buyAmount));
  console.log("USDT Profit/Loss:", ethers.utils.formatEther(usdtDifference));
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

## 7. Frontend Interface (Optional)

Create a simple frontend to interact with your contracts:

1. Create a file `index.html`:
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Synthetic Stock Trading Simulator</title>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.0.2/dist/css/bootstrap.min.css" rel="stylesheet">
  <script src="https://cdn.ethers.io/lib/ethers-5.4.umd.min.js" type="application/javascript"></script>
</head>
<body>
  <div class="container mt-5">
    <h1>Synthetic Stock Trading Simulator</h1>
    <div class="row mt-4">
      <div class="col-md-6">
        <div class="card">
          <div class="card-header">
            Account Information
          </div>
          <div class="card-body">
            <p><strong>Connected Account:</strong> <span id="account">Not connected</span></p>
            <p><strong>USDT Balance:</strong> <span id="usdtBalance">0</span></p>
            <p><strong>mAAPL Balance:</strong> <span id="maaplBalance">0</span></p>
            <p><strong>Current Apple Price:</strong> $<span id="applePrice">0</span></p>
            <button class="btn btn-primary" id="connectWallet">Connect Wallet</button>
          </div>
        </div>
      </div>
      <div class="col-md-6">
        <div class="card">
          <div class="card-header">
            Trading Actions
          </div>
          <div class="card-body">
            <div class="mb-3">
              <label for="buyAmount" class="form-label">Buy mAAPL with USDT</label>
              <input type="number" class="form-control" id="buyAmount" placeholder="Amount in USDT">
              <button class="btn btn-success mt-2" id="buyButton">Buy</button>
            </div>
            <div class="mb-3">
              <label for="sellAmount" class="form-label">Sell mAAPL for USDT</label>
              <input type="number" class="form-control" id="sellAmount" placeholder="Amount in mAAPL">
              <button class="btn btn-danger mt-2" id="sellButton">Sell</button>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>

  <script src="app.js"></script>
</body>
</html>
```

2. Create a file `app.js`:
```javascript
// Contract addresses - replace with your deployed contract addresses
const USDT_ADDRESS = "YOUR_DEPLOYED_USDT_ADDRESS";
const MAPPL_ADDRESS = "YOUR_DEPLOYED_MAPPL_ADDRESS";
const EXCHANGE_ADDRESS = "YOUR_DEPLOYED_EXCHANGE_ADDRESS";

// ABI definitions - replace with your actual contract ABIs
const USDT_ABI = [/* Your MockUSDT ABI */];
const MAPPL_ABI = [/* Your MockSyntheticApple ABI */];
const EXCHANGE_ABI = [/* Your MockExchange ABI */];

let provider;
let signer;
let usdtContract;
let maaplContract;
let exchangeContract;

// Connect wallet
async function connectWallet() {
  if (window.ethereum) {
    try {
      await window.ethereum.request({ method: 'eth_requestAccounts' });
      provider = new ethers.providers.Web3Provider(window.ethereum);
      signer = provider.getSigner();
      
      // Get connected account
      const account = await signer.getAddress();
      document.getElementById('account').textContent = account;
      
      // Initialize contracts
      usdtContract = new ethers.Contract(USDT_ADDRESS, USDT_ABI, signer);
      maaplContract = new ethers.Contract(MAPPL_ADDRESS, MAPPL_ABI, signer);
      exchangeContract = new ethers.Contract(EXCHANGE_ADDRESS, EXCHANGE_ABI, signer);
      
      // Update balances
      updateBalances();
    } catch (error) {
      console.error(error);
    }
  } else {
    alert('Please install MetaMask to use this dApp');
  }
}

// Update account balances
async function updateBalances() {
  if (!signer) return;
  
  try {
    const account = await signer.getAddress();
    
    // Get USDT balance
    const usdtBalance = await usdtContract.balanceOf(account);
    document.getElementById('usdtBalance').textContent = ethers.utils.formatEther(usdtBalance);
    
    // Get mAAPL balance
    const maaplBalance = await maaplContract.balanceOf(account);
    document.getElementById('maaplBalance').textContent = ethers.utils.formatEther(maaplBalance);
    
    // Get Apple price
    const applePriceRaw = await maaplContract.getApplePrice();
    const applePrice = applePriceRaw / 1e8; // Convert from 8 decimals
    document.getElementById('applePrice').textContent = applePrice.toFixed(2);
  } catch (error) {
    console.error('Error updating balances:', error);
  }
}

// Buy mAAPL
async function buyApple() {
  if (!signer || !exchangeContract) return;
  
  try {
    const buyAmount = document.getElementById('buyAmount').value;
    if (!buyAmount || buyAmount <= 0) {
      alert('Please enter a valid amount');
      return;
    }
    
    const buyAmountWei = ethers.utils.parseEther(buyAmount);
    
    // Approve exchange to spend USDT
    await usdtContract.approve(EXCHANGE_ADDRESS, buyAmountWei);
    alert('Approved exchange to spend USDT');
    
    // Buy Apple
    const tx = await exchangeContract.buyApple(buyAmountWei);
    await tx.wait();
    
    alert('Successfully bought synthetic Apple');
    updateBalances();
  } catch (error) {
    console.error('Error buying Apple:', error);
    alert('Error: ' + error.message);
  }
}

// Sell mAAPL
async function sellApple() {
  if (!signer || !exchangeContract) return;
  
  try {
    const sellAmount = document.getElementById('sellAmount').value;
    if (!sellAmount || sellAmount <= 0) {
      alert('Please enter a valid amount');
      return;
    }
    
    const sellAmountWei = ethers.utils.parseEther(sellAmount);
    
    // Approve exchange to spend mAAPL
    await maaplContract.approve(EXCHANGE_ADDRESS, sellAmountWei);
    alert('Approved exchange to spend mAAPL');
    
    // Sell Apple
    const tx = await exchangeContract.sellApple(sellAmountWei);
    await tx.wait();
    
    alert('Successfully sold synthetic Apple');
    updateBalances();
  } catch (error) {
    console.error('Error selling Apple:', error);
    alert('Error: ' + error.message);
  }
}

// Event listeners
document.getElementById('connectWallet').addEventListener('click', connectWallet);
document.getElementById('buyButton').addEventListener('click', buyApple);
document.getElementById('sellButton').addEventListener('click', sellApple);

// Check if MetaMask is already connected
window.addEventListener('load', async () => {
  if (window.ethereum) {
    const accounts = await window.ethereum.request({ method: 'eth_accounts' });
    if (accounts.length > 0) {
      connectWallet();
    }
  }
});
```

## 8. Running the Test Environment

1. Compile the contracts:
```bash
npx hardhat compile
```

2. Deploy to the BSC Testnet:
```bash
npx hardhat run scripts/deploy.js --network bscTestnet
```

3. Update your testing script with the deployed contract addresses

4. Run the test trade script:
```bash
npx hardhat run scripts/test-trade.js --network bscTestnet
```

5. For the frontend (optional):
   - Update the contract addresses and ABIs in app.js
   - Serve the frontend files using a local server:
   ```bash
   npx serve
   ```

## 9. Monitoring and Analysis

1. Use BSC Testnet Explorer (https://testnet.bscscan.com) to track your transactions
2. Create a simple dashboard to track:
   - Transaction costs (gas used)
   - Transaction times
   - Price accuracy compared to real Apple stock
   - Slippage during trades

## 10. Potential Extensions

1. Add a mock oracle service that updates synthetic Apple prices from real market data
2. Implement a bridge to simulate ETH to BSC conversion
3. Add multiple synthetic stocks to create a portfolio testing environment
4. Implement leveraged positions (3x long/short)
5. Add liquidity pool functionality to test yield generation

By following this guide, you'll have a functional test environment that closely simulates the real-world use case of trading synthetic Apple stock on Binance Smart Chain using Ethereum holdings, allowing you to test various scenarios, measure gas costs, and optimize the trading process.
