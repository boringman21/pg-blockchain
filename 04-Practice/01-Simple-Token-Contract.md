# 🪙 Simple Token Contract

## 🎯 Mục tiêu dự án

Tạo một ERC-20 token đơn giản để hiểu:

- Cơ bản về smart contracts
- Token standards và interfaces
- Deployment và interaction
- Testing và verification

## 📋 Yêu cầu chức năng

### Core Features

- ✅ **Basic ERC-20 implementation**
- ✅ **Fixed total supply**
- ✅ **Transfer functionality** 
- ✅ **Allowance mechanism**
- ✅ **Balance checking**

### Advanced Features (Optional)

- 🔄 **Mintable tokens**
- 🔥 **Burnable tokens**
- ⏸️ **Pausable transfers**
- 👑 **Ownership controls**

## 🏗️ Technical Specifications

### Token Details
```javascript
const tokenSpecs = {
  name: "MyFirstToken",
  symbol: "MFT", 
  decimals: 18,
  totalSupply: 1000000, // 1 million tokens
  initialOwner: "0x..." // Your wallet address
}
```

### ERC-20 Interface
```solidity
interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}
```

## 💻 Implementation

### Step 1: Setup Project

```bash
mkdir simple-token-project
cd simple-token-project

# Initialize Hardhat project
npm init -y
npm install --save-dev hardhat
npx hardhat

# Install OpenZeppelin contracts
npm install @openzeppelin/contracts

# Install additional dependencies
npm install --save-dev @nomicfoundation/hardhat-toolbox dotenv
```

### Step 2: Smart Contract

**contracts/MyFirstToken.sol:**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title MyFirstToken
 * @dev A simple ERC-20 token implementation
 */
contract MyFirstToken is ERC20, Ownable {
    uint256 public constant INITIAL_SUPPLY = 1_000_000 * 10**18; // 1 million tokens
    
    /**
     * @dev Constructor that mints initial supply to deployer
     */
    constructor() ERC20("MyFirstToken", "MFT") {
        _mint(msg.sender, INITIAL_SUPPLY);
    }
    
    /**
     * @dev Mint new tokens (only owner)
     * @param to Address to mint tokens to
     * @param amount Amount of tokens to mint
     */
    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }
    
    /**
     * @dev Burn tokens from caller's balance
     * @param amount Amount of tokens to burn
     */
    function burn(uint256 amount) public {
        _burn(msg.sender, amount);
    }
    
    /**
     * @dev Burn tokens from specified account (with allowance)
     * @param from Address to burn tokens from
     * @param amount Amount of tokens to burn
     */
    function burnFrom(address from, uint256 amount) public {
        _spendAllowance(from, msg.sender, amount);
        _burn(from, amount);
    }
    
    /**
     * @dev Override transfer to add custom logic (optional)
     */
    function transfer(address to, uint256 amount) public virtual override returns (bool) {
        address owner = _msgSender();
        _transfer(owner, to, amount);
        return true;
    }
    
    /**
     * @dev Get token information
     */
    function getTokenInfo() public view returns (
        string memory tokenName,
        string memory tokenSymbol, 
        uint8 tokenDecimals,
        uint256 tokenTotalSupply
    ) {
        return (name(), symbol(), decimals(), totalSupply());
    }
}
```

### Step 3: Deployment Script

**scripts/deploy.js:**
```javascript
const { ethers } = require("hardhat");

async function main() {
  const [deployer] = await ethers.getSigners();
  
  console.log("Deploying contracts with the account:", deployer.address);
  console.log("Account balance:", (await deployer.getBalance()).toString());
  
  // Deploy the token
  const MyFirstToken = await ethers.getContractFactory("MyFirstToken");
  const token = await MyFirstToken.deploy();
  
  await token.deployed();
  
  console.log("MyFirstToken deployed to:", token.address);
  console.log("Transaction hash:", token.deployTransaction.hash);
  
  // Get token information
  const tokenInfo = await token.getTokenInfo();
  console.log("Token Name:", tokenInfo.tokenName);
  console.log("Token Symbol:", tokenInfo.tokenSymbol);
  console.log("Token Decimals:", tokenInfo.tokenDecimals);
  console.log("Total Supply:", ethers.utils.formatEther(tokenInfo.tokenTotalSupply));
  
  // Get deployer balance
  const deployerBalance = await token.balanceOf(deployer.address);
  console.log("Deployer Token Balance:", ethers.utils.formatEther(deployerBalance));
  
  // Save deployment info
  const deploymentInfo = {
    network: network.name,
    contract: "MyFirstToken",
    address: token.address,
    deployer: deployer.address,
    blockNumber: token.deployTransaction.blockNumber,
    timestamp: new Date().toISOString()
  };
  
  console.log("\n=== Deployment Info ===");
  console.log(JSON.stringify(deploymentInfo, null, 2));
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

### Step 4: Comprehensive Tests

**test/MyFirstToken.test.js:**
```javascript
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("MyFirstToken", function () {
  let token;
  let owner;
  let addr1;
  let addr2;
  let addrs;

  const INITIAL_SUPPLY = ethers.utils.parseEther("1000000"); // 1 million tokens

  beforeEach(async function () {
    [owner, addr1, addr2, ...addrs] = await ethers.getSigners();
    
    const MyFirstToken = await ethers.getContractFactory("MyFirstToken");
    token = await MyFirstToken.deploy();
    await token.deployed();
  });

  describe("Deployment", function () {
    it("Should set the right owner", async function () {
      expect(await token.owner()).to.equal(owner.address);
    });

    it("Should assign the total supply to the owner", async function () {
      const ownerBalance = await token.balanceOf(owner.address);
      expect(await token.totalSupply()).to.equal(ownerBalance);
    });

    it("Should have correct token details", async function () {
      expect(await token.name()).to.equal("MyFirstToken");
      expect(await token.symbol()).to.equal("MFT");
      expect(await token.decimals()).to.equal(18);
      expect(await token.totalSupply()).to.equal(INITIAL_SUPPLY);
    });
  });

  describe("Transactions", function () {
    it("Should transfer tokens between accounts", async function () {
      const transferAmount = ethers.utils.parseEther("100");
      
      // Transfer from owner to addr1
      await token.transfer(addr1.address, transferAmount);
      const addr1Balance = await token.balanceOf(addr1.address);
      expect(addr1Balance).to.equal(transferAmount);

      // Transfer from addr1 to addr2
      await token.connect(addr1).transfer(addr2.address, transferAmount);
      const addr2Balance = await token.balanceOf(addr2.address);
      expect(addr2Balance).to.equal(transferAmount);
    });

    it("Should fail if sender doesn't have enough tokens", async function () {
      const initialOwnerBalance = await token.balanceOf(owner.address);
      const transferAmount = initialOwnerBalance.add(1);

      await expect(
        token.connect(addr1).transfer(owner.address, transferAmount)
      ).to.be.revertedWith("ERC20: transfer amount exceeds balance");
    });

    it("Should update balances after transfers", async function () {
      const initialOwnerBalance = await token.balanceOf(owner.address);
      const transferAmount = ethers.utils.parseEther("100");

      await token.transfer(addr1.address, transferAmount);
      await token.transfer(addr2.address, transferAmount);

      const finalOwnerBalance = await token.balanceOf(owner.address);
      expect(finalOwnerBalance).to.equal(
        initialOwnerBalance.sub(transferAmount.mul(2))
      );

      const addr1Balance = await token.balanceOf(addr1.address);
      expect(addr1Balance).to.equal(transferAmount);

      const addr2Balance = await token.balanceOf(addr2.address);
      expect(addr2Balance).to.equal(transferAmount);
    });
  });

  describe("Allowances", function () {
    it("Should approve tokens for delegated transfer", async function () {
      const approveAmount = ethers.utils.parseEther("100");
      
      await token.approve(addr1.address, approveAmount);
      expect(await token.allowance(owner.address, addr1.address)).to.equal(approveAmount);
    });

    it("Should transfer tokens using transferFrom", async function () {
      const approveAmount = ethers.utils.parseEther("100");
      const transferAmount = ethers.utils.parseEther("50");
      
      await token.approve(addr1.address, approveAmount);
      await token.connect(addr1).transferFrom(owner.address, addr2.address, transferAmount);
      
      expect(await token.balanceOf(addr2.address)).to.equal(transferAmount);
      expect(await token.allowance(owner.address, addr1.address)).to.equal(
        approveAmount.sub(transferAmount)
      );
    });

    it("Should fail transferFrom without sufficient allowance", async function () {
      const transferAmount = ethers.utils.parseEther("100");
      
      await expect(
        token.connect(addr1).transferFrom(owner.address, addr2.address, transferAmount)
      ).to.be.revertedWith("ERC20: insufficient allowance");
    });
  });

  describe("Minting", function () {
    it("Should allow owner to mint tokens", async function () {
      const mintAmount = ethers.utils.parseEther("1000");
      const initialSupply = await token.totalSupply();
      
      await token.mint(addr1.address, mintAmount);
      
      expect(await token.balanceOf(addr1.address)).to.equal(mintAmount);
      expect(await token.totalSupply()).to.equal(initialSupply.add(mintAmount));
    });

    it("Should fail if non-owner tries to mint", async function () {
      const mintAmount = ethers.utils.parseEther("1000");
      
      await expect(
        token.connect(addr1).mint(addr1.address, mintAmount)
      ).to.be.revertedWith("Ownable: caller is not the owner");
    });
  });

  describe("Burning", function () {
    it("Should allow users to burn their tokens", async function () {
      const burnAmount = ethers.utils.parseEther("1000");
      const initialBalance = await token.balanceOf(owner.address);
      const initialSupply = await token.totalSupply();
      
      await token.burn(burnAmount);
      
      expect(await token.balanceOf(owner.address)).to.equal(
        initialBalance.sub(burnAmount)
      );
      expect(await token.totalSupply()).to.equal(initialSupply.sub(burnAmount));
    });

    it("Should allow burning from another account with allowance", async function () {
      const transferAmount = ethers.utils.parseEther("1000");
      const burnAmount = ethers.utils.parseEther("500");
      
      // Transfer tokens to addr1
      await token.transfer(addr1.address, transferAmount);
      
      // Approve addr2 to burn tokens from addr1
      await token.connect(addr1).approve(addr2.address, burnAmount);
      
      // Burn tokens
      await token.connect(addr2).burnFrom(addr1.address, burnAmount);
      
      expect(await token.balanceOf(addr1.address)).to.equal(
        transferAmount.sub(burnAmount)
      );
    });
  });

  describe("Events", function () {
    it("Should emit Transfer event on transfer", async function () {
      const transferAmount = ethers.utils.parseEther("100");
      
      await expect(token.transfer(addr1.address, transferAmount))
        .to.emit(token, "Transfer")
        .withArgs(owner.address, addr1.address, transferAmount);
    });

    it("Should emit Approval event on approve", async function () {
      const approveAmount = ethers.utils.parseEther("100");
      
      await expect(token.approve(addr1.address, approveAmount))
        .to.emit(token, "Approval")
        .withArgs(owner.address, addr1.address, approveAmount);
    });
  });

  describe("Edge Cases", function () {
    it("Should handle zero transfers", async function () {
      await expect(token.transfer(addr1.address, 0)).to.not.be.reverted;
      expect(await token.balanceOf(addr1.address)).to.equal(0);
    });

    it("Should handle self transfers", async function () {
      const transferAmount = ethers.utils.parseEther("100");
      const initialBalance = await token.balanceOf(owner.address);
      
      await token.transfer(owner.address, transferAmount);
      expect(await token.balanceOf(owner.address)).to.equal(initialBalance);
    });
  });
});
```

### Step 5: Frontend Interaction

**scripts/interact.js:**
```javascript
const { ethers } = require("hardhat");

async function main() {
  const [deployer, user1, user2] = await ethers.getSigners();
  
  // Replace with your deployed contract address
  const TOKEN_ADDRESS = "0x..."; // Your deployed contract address
  
  // Get contract instance
  const MyFirstToken = await ethers.getContractFactory("MyFirstToken");
  const token = await MyFirstToken.attach(TOKEN_ADDRESS);
  
  console.log("=== Token Interaction Demo ===");
  
  // 1. Check token info
  console.log("\n1. Token Information:");
  const tokenInfo = await token.getTokenInfo();
  console.log(`Name: ${tokenInfo.tokenName}`);
  console.log(`Symbol: ${tokenInfo.tokenSymbol}`);
  console.log(`Decimals: ${tokenInfo.tokenDecimals}`);
  console.log(`Total Supply: ${ethers.utils.formatEther(tokenInfo.tokenTotalSupply)}`);
  
  // 2. Check balances
  console.log("\n2. Current Balances:");
  const deployerBalance = await token.balanceOf(deployer.address);
  const user1Balance = await token.balanceOf(user1.address);
  console.log(`Deployer: ${ethers.utils.formatEther(deployerBalance)} MFT`);
  console.log(`User1: ${ethers.utils.formatEther(user1Balance)} MFT`);
  
  // 3. Transfer tokens
  console.log("\n3. Transferring tokens...");
  const transferAmount = ethers.utils.parseEther("1000");
  const tx1 = await token.transfer(user1.address, transferAmount);
  await tx1.wait();
  console.log(`Transferred ${ethers.utils.formatEther(transferAmount)} MFT to User1`);
  
  // 4. Check updated balances
  console.log("\n4. Updated Balances:");
  const newDeployerBalance = await token.balanceOf(deployer.address);
  const newUser1Balance = await token.balanceOf(user1.address);
  console.log(`Deployer: ${ethers.utils.formatEther(newDeployerBalance)} MFT`);
  console.log(`User1: ${ethers.utils.formatEther(newUser1Balance)} MFT`);
  
  // 5. Approve and transferFrom
  console.log("\n5. Approve and TransferFrom:");
  const approveAmount = ethers.utils.parseEther("500");
  const tx2 = await token.connect(user1).approve(deployer.address, approveAmount);
  await tx2.wait();
  console.log(`User1 approved ${ethers.utils.formatEther(approveAmount)} MFT to Deployer`);
  
  const allowance = await token.allowance(user1.address, deployer.address);
  console.log(`Current allowance: ${ethers.utils.formatEther(allowance)} MFT`);
  
  const transferFromAmount = ethers.utils.parseEther("200");
  const tx3 = await token.transferFrom(user1.address, user2.address, transferFromAmount);
  await tx3.wait();
  console.log(`Transferred ${ethers.utils.formatEther(transferFromAmount)} MFT from User1 to User2`);
  
  // 6. Final balances
  console.log("\n6. Final Balances:");
  const finalDeployerBalance = await token.balanceOf(deployer.address);
  const finalUser1Balance = await token.balanceOf(user1.address);
  const finalUser2Balance = await token.balanceOf(user2.address);
  console.log(`Deployer: ${ethers.utils.formatEther(finalDeployerBalance)} MFT`);
  console.log(`User1: ${ethers.utils.formatEther(finalUser1Balance)} MFT`);
  console.log(`User2: ${ethers.utils.formatEther(finalUser2Balance)} MFT`);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

## 🚀 Deployment Guide

### Local Deployment

```bash
# Start local Hardhat network
npx hardhat node

# Deploy to local network (in another terminal)
npx hardhat run scripts/deploy.js --network localhost

# Run interaction script
npx hardhat run scripts/interact.js --network localhost
```

### Testnet Deployment

```bash
# Deploy to Sepolia testnet
npx hardhat run scripts/deploy.js --network sepolia

# Verify contract on Etherscan
npx hardhat verify --network sepolia DEPLOYED_CONTRACT_ADDRESS
```

## 🧪 Testing Guide

```bash
# Run all tests
npx hardhat test

# Run tests with gas reporting
npx hardhat test --gas-report

# Run specific test file
npx hardhat test test/MyFirstToken.test.js

# Run tests with coverage
npx hardhat coverage
```

## 📊 Gas Optimization Tips

### Efficient patterns

```solidity
// ✅ Good: Use unchecked for safe operations
function efficientTransfer(address to, uint256 amount) external {
    uint256 balance = balanceOf(msg.sender);
    require(balance >= amount, "Insufficient balance");
    
    unchecked {
        _balances[msg.sender] = balance - amount;
        _balances[to] += amount;
    }
    
    emit Transfer(msg.sender, to, amount);
}

// ✅ Good: Batch operations
function batchTransfer(
    address[] calldata recipients,
    uint256[] calldata amounts
) external {
    require(recipients.length == amounts.length, "Array length mismatch");
    
    for (uint256 i = 0; i < recipients.length;) {
        _transfer(msg.sender, recipients[i], amounts[i]);
        unchecked { ++i; }
    }
}
```

## ⚠️ Security Considerations

### Common vulnerabilities

```solidity
// ❌ Bad: Reentrancy possibility
function badTransfer(address to, uint256 amount) external {
    balances[to] += amount;
    balances[msg.sender] -= amount; // State change after external call
    
    // External call - potential reentrancy
    if (to.code.length > 0) {
        IERC20Receiver(to).onTokenReceived(msg.sender, amount);
    }
}

// ✅ Good: Check-Effects-Interactions pattern
function goodTransfer(address to, uint256 amount) external {
    require(balances[msg.sender] >= amount, "Insufficient balance");
    
    // Effects first
    balances[msg.sender] -= amount;
    balances[to] += amount;
    
    emit Transfer(msg.sender, to, amount);
    
    // Interactions last
    if (to.code.length > 0) {
        IERC20Receiver(to).onTokenReceived(msg.sender, amount);
    }
}
```

## 🔗 Integration Examples

### Web3.js Frontend

```javascript
// Connect to token contract
const Web3 = require('web3');
const web3 = new Web3(window.ethereum);

const tokenABI = [/* Your contract ABI */];
const tokenAddress = "0x..."; // Your deployed contract address

const contract = new web3.eth.Contract(tokenABI, tokenAddress);

// Get token balance
async function getBalance(address) {
    const balance = await contract.methods.balanceOf(address).call();
    return web3.utils.fromWei(balance, 'ether');
}

// Transfer tokens
async function transferTokens(to, amount) {
    const accounts = await web3.eth.getAccounts();
    const amountWei = web3.utils.toWei(amount.toString(), 'ether');
    
    return await contract.methods.transfer(to, amountWei).send({
        from: accounts[0]
    });
}
```

## ✅ Project Checklist

**Development:**
- [ ] Smart contract implemented with all ERC-20 functions
- [ ] Comprehensive test suite with 100% coverage
- [ ] Deployment scripts for local and testnet
- [ ] Gas optimization applied
- [ ] Security considerations addressed

**Testing:**
- [ ] All tests passing
- [ ] Edge cases covered
- [ ] Gas usage analyzed
- [ ] Security audit completed (for production)

**Deployment:**
- [ ] Successfully deployed to testnet
- [ ] Contract verified on Etherscan
- [ ] Interaction scripts working
- [ ] Frontend integration tested

## 🔗 Navigation

- **Previous**: [[00-Index]] - Practice overview
- **Next**: [[02-Voting-System]] - Build voting DApp
- **Related**: [[06-Solidity-Basics]] - Solidity fundamentals

## ✅ Key Learnings

1. **ERC-20 standard** provides interoperability
2. **OpenZeppelin contracts** ensure security
3. **Comprehensive testing** prevents bugs
4. **Gas optimization** reduces costs
5. **Security patterns** protect user funds

---

**Status**: 🚧 Ready to implement | **Difficulty**: ⭐⭐☆☆☆
