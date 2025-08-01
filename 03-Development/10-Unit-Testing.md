# 🧪 Unit Testing Smart Contracts: First-Principles Testing Mastery

## 🧠 First-Principles Understanding: "Tại sao Test lại quan trọng?"

### Mental Model: Smart Contracts như "Immutable Financial Software"

- **Traditional software**: Bug → patch → update
- **Smart contracts**: Bug → permanent → potential millions lost
- **Key insight**: Testing = Insurance against catastrophic failures
- **2025 Reality**: $2+ billion lost annually due to untested edge cases

### Core Testing Philosophy: Prove Correctness, Don't Just Demo

```text
❌ Demo Testing: "It works in happy path"
✅ Proof Testing: "It fails safely in ALL edge cases"

Demo: transfer(user, 100) → ✅ works
Proof: transfer(user, balanceOf[sender] + 1) → ❌ should revert
```

## 🛠️ Modern Testing Framework Comparison (2025)

### 1. Foundry: The Rust-Powered Testing Beast

**Why Foundry Dominates 2025:**

- **Performance**: 1000+ fuzz tests/hour vs Hardhat's 100/hour
- **Native Solidity**: Tests in same language as contracts
- **Built-in fuzzing**: Property-based testing out of the box
- **Mainnet forking**: Test against real state effortlessly

```solidity
// ===== FOUNDRY TEST STRUCTURE =====
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/Token.sol";

contract TokenTest is Test {
    Token token;
    address owner = makeAddr("owner");
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    address zero = address(0);
    
    function setUp() public {
        vm.prank(owner);
        token = new Token("TestToken", "TEST", 18, 1000000e18);
    }
    
    // ===== BASIC UNIT TESTS =====
    function testInitialState() public {
        assertEq(token.name(), "TestToken");
        assertEq(token.symbol(), "TEST");
        assertEq(token.decimals(), 18);
        assertEq(token.totalSupply(), 1000000e18);
        assertEq(token.balanceOf(owner), 1000000e18);
    }
    
    function testTransfer() public {
        uint256 transferAmount = 1000e18;
        
        vm.prank(owner);
        bool success = token.transfer(alice, transferAmount);
        
        assertTrue(success);
        assertEq(token.balanceOf(alice), transferAmount);
        assertEq(token.balanceOf(owner), 1000000e18 - transferAmount);
    }
    
    // ===== FAILURE TESTING =====
    function testTransferFailsInsufficientBalance() public {
        uint256 transferAmount = 1000000e18 + 1; // More than balance
        
        vm.prank(owner);
        vm.expectRevert("ERC20: transfer amount exceeds balance");
        token.transfer(alice, transferAmount);
    }
    
    function testTransferFailsZeroAddress() public {
        vm.prank(owner);
        vm.expectRevert("ERC20: transfer to the zero address");
        token.transfer(zero, 1000e18);
    }
    
    // ===== FUZZ TESTING =====
    function testFuzzTransfer(uint256 amount) public {
        // Bound the fuzz input to valid range
        amount = bound(amount, 0, token.balanceOf(owner));
        
        uint256 initialOwnerBalance = token.balanceOf(owner);
        uint256 initialAliceBalance = token.balanceOf(alice);
        
        vm.prank(owner);
        token.transfer(alice, amount);
        
        assertEq(token.balanceOf(owner), initialOwnerBalance - amount);
        assertEq(token.balanceOf(alice), initialAliceBalance + amount);
    }
    
    function testFuzzTransferFailsInsufficientBalance(uint256 amount) public {
        // Test amounts greater than balance always fail
        vm.assume(amount > token.balanceOf(owner));
        
        vm.prank(owner);
        vm.expectRevert();
        token.transfer(alice, amount);
    }
    
    // ===== INVARIANT TESTING =====
    function invariant_totalSupplyConstant() public {
        assertEq(token.totalSupply(), 1000000e18);
    }
    
    function invariant_balanceSumEqualsTotalSupply() public {
        // This would require tracking all addresses in a more complex setup
        // For demo purposes, testing known addresses
        uint256 totalBalance = token.balanceOf(owner) + 
                              token.balanceOf(alice) + 
                              token.balanceOf(bob);
        assertLe(totalBalance, token.totalSupply());
    }
}

// ===== ADVANCED TESTING PATTERNS =====
contract TokenAdvancedTest is Test {
    Token token;
    address owner = makeAddr("owner");
    
    function setUp() public {
        vm.prank(owner);
        token = new Token("TestToken", "TEST", 18, 1000000e18);
    }
    
    // ===== EVENT TESTING =====
    function testTransferEmitsEvent() public {
        uint256 amount = 1000e18;
        address to = makeAddr("to");
        
        vm.prank(owner);
        
        // Expect Transfer event
        vm.expectEmit(true, true, false, true);
        emit Transfer(owner, to, amount);
        
        token.transfer(to, amount);
    }
    
    // ===== TIME-BASED TESTING =====
    function testTimelock() public {
        // Assume token has timelock functionality
        uint256 unlockTime = block.timestamp + 1 days;
        
        // Fast forward time
        vm.warp(unlockTime);
        
        // Now test functionality that depends on time
        assertTrue(block.timestamp >= unlockTime);
    }
    
    // ===== COMPLEX STATE TESTING =====
    function testComplexTransferScenario() public {
        address alice = makeAddr("alice");
        address bob = makeAddr("bob");
        address charlie = makeAddr("charlie");
        
        uint256 amount1 = 1000e18;
        uint256 amount2 = 500e18;
        uint256 amount3 = 300e18;
        
        // Complex transfer chain: owner → alice → bob → charlie
        vm.prank(owner);
        token.transfer(alice, amount1);
        
        vm.prank(alice);
        token.transfer(bob, amount2);
        
        vm.prank(bob);
        token.transfer(charlie, amount3);
        
        // Verify final state
        assertEq(token.balanceOf(alice), amount1 - amount2);
        assertEq(token.balanceOf(bob), amount2 - amount3);
        assertEq(token.balanceOf(charlie), amount3);
    }
}

// ===== INTEGRATION WITH MOCK CONTRACTS =====
contract MockERC20 {
    function transfer(address to, uint256 amount) external returns (bool) {
        return true; // Always succeeds for testing
    }
    
    function balanceOf(address account) external pure returns (uint256) {
        return 1000e18; // Fixed balance for testing
    }
}

contract TokenIntegrationTest is Test {
    Token token;
    MockERC20 mockToken;
    address owner = makeAddr("owner");
    
    function setUp() public {
        vm.prank(owner);
        token = new Token("TestToken", "TEST", 18, 1000000e18);
        mockToken = new MockERC20();
    }
    
    function testIntegrationWithMockContract() public {
        // Test interaction between real and mock contracts
        uint256 mockBalance = mockToken.balanceOf(owner);
        bool mockTransfer = mockToken.transfer(makeAddr("recipient"), 100e18);
        
        assertEq(mockBalance, 1000e18);
        assertTrue(mockTransfer);
    }
}
```

### 2. Hardhat: JavaScript/TypeScript Testing Powerhouse

```javascript
// ===== HARDHAT TEST STRUCTURE =====
const { expect } = require("chai");
const { ethers } = require("hardhat");
const { loadFixture } = require("@nomicfoundation/hardhat-network-helpers");

describe("Token Contract", function() {
    // ===== FIXTURE PATTERN =====
    async function deployTokenFixture() {
        const [owner, alice, bob] = await ethers.getSigners();
        
        const Token = await ethers.getContractFactory("Token");
        const token = await Token.deploy("TestToken", "TEST", 18, ethers.utils.parseEther("1000000"));
        
        return { token, owner, alice, bob };
    }
    
    describe("Deployment", function() {
        it("Should set the right owner", async function() {
            const { token, owner } = await loadFixture(deployTokenFixture);
            expect(await token.balanceOf(owner.address)).to.equal(ethers.utils.parseEther("1000000"));
        });
        
        it("Should have correct initial values", async function() {
            const { token } = await loadFixture(deployTokenFixture);
            
            expect(await token.name()).to.equal("TestToken");
            expect(await token.symbol()).to.equal("TEST");
            expect(await token.decimals()).to.equal(18);
        });
    });
    
    describe("Transfers", function() {
        it("Should transfer tokens between accounts", async function() {
            const { token, owner, alice } = await loadFixture(deployTokenFixture);
            const transferAmount = ethers.utils.parseEther("1000");
            
            await token.transfer(alice.address, transferAmount);
            
            expect(await token.balanceOf(alice.address)).to.equal(transferAmount);
        });
        
        it("Should fail if sender doesn't have enough tokens", async function() {
            const { token, alice, bob } = await loadFixture(deployTokenFixture);
            const transferAmount = ethers.utils.parseEther("1000");
            
            await expect(token.connect(alice).transfer(bob.address, transferAmount))
                .to.be.revertedWith("ERC20: transfer amount exceeds balance");
        });
        
        it("Should emit Transfer event", async function() {
            const { token, owner, alice } = await loadFixture(deployTokenFixture);
            const transferAmount = ethers.utils.parseEther("1000");
            
            await expect(token.transfer(alice.address, transferAmount))
                .to.emit(token, "Transfer")
                .withArgs(owner.address, alice.address, transferAmount);
        });
    });
    
    // ===== COMPLEX SCENARIO TESTING =====
    describe("Complex Scenarios", function() {
        it("Should handle multiple transfers correctly", async function() {
            const { token, owner, alice, bob } = await loadFixture(deployTokenFixture);
            
            const amount1 = ethers.utils.parseEther("1000");
            const amount2 = ethers.utils.parseEther("500");
            
            // Transfer from owner to alice
            await token.transfer(alice.address, amount1);
            
            // Transfer from alice to bob
            await token.connect(alice).transfer(bob.address, amount2);
            
            expect(await token.balanceOf(alice.address)).to.equal(amount1.sub(amount2));
            expect(await token.balanceOf(bob.address)).to.equal(amount2);
        });
        
        it("Should handle allowances correctly", async function() {
            const { token, owner, alice, bob } = await loadFixture(deployTokenFixture);
            const allowanceAmount = ethers.utils.parseEther("1000");
            const transferAmount = ethers.utils.parseEther("500");
            
            // Approve alice to spend owner's tokens
            await token.approve(alice.address, allowanceAmount);
            
            // Alice transfers from owner to bob
            await token.connect(alice).transferFrom(owner.address, bob.address, transferAmount);
            
            expect(await token.balanceOf(bob.address)).to.equal(transferAmount);
            expect(await token.allowance(owner.address, alice.address))
                .to.equal(allowanceAmount.sub(transferAmount));
        });
    });
    
    // ===== GAS USAGE TESTING =====
    describe("Gas Usage", function() {
        it("Should track gas usage for transfers", async function() {
            const { token, owner, alice } = await loadFixture(deployTokenFixture);
            const transferAmount = ethers.utils.parseEther("1000");
            
            const tx = await token.transfer(alice.address, transferAmount);
            const receipt = await tx.wait();
            
            console.log("Transfer gas used:", receipt.gasUsed.toString());
            expect(receipt.gasUsed).to.be.lt(100000); // Should be under 100k gas
        });
    });
    
    // ===== SNAPSHOT TESTING =====
    describe("State Snapshots", function() {
        it("Should revert to snapshot after failed operations", async function() {
            const { token, owner, alice } = await loadFixture(deployTokenFixture);
            
            // Take snapshot
            const snapshot = await ethers.provider.send("evm_snapshot", []);
            
            // Make some changes
            await token.transfer(alice.address, ethers.utils.parseEther("1000"));
            
            // Revert to snapshot
            await ethers.provider.send("evm_revert", [snapshot]);
            
            // State should be reverted
            expect(await token.balanceOf(alice.address)).to.equal(0);
        });
    });
});

// ===== PROPERTY-BASED TESTING =====
const fc = require("fast-check");

describe("Property-Based Testing", function() {
    it("Transfer should preserve total supply", async function() {
        const { token, owner } = await loadFixture(deployTokenFixture);
        const initialSupply = await token.totalSupply();
        
        await fc.assert(fc.asyncProperty(
            fc.integer(1, 1000000),
            async (amount) => {
                const transferAmount = ethers.utils.parseEther(amount.toString());
                const recipient = ethers.Wallet.createRandom().address;
                
                if (transferAmount.lte(await token.balanceOf(owner.address))) {
                    await token.transfer(recipient, transferAmount);
                    expect(await token.totalSupply()).to.equal(initialSupply);
                }
            }
        ));
    });
});
```

## 🎯 Testing Strategy Matrix

### Level 1: Unit Tests (Individual Functions)

```solidity
contract UnitTestExample is Test {
    // Test single function in isolation
    function testSingleFunction() public {
        uint256 result = mathContract.add(2, 3);
        assertEq(result, 5);
    }
    
    // Test function with edge cases
    function testEdgeCases() public {
        // Test overflow
        vm.expectRevert();
        mathContract.add(type(uint256).max, 1);
        
        // Test underflow
        vm.expectRevert();
        mathContract.subtract(0, 1);
    }
}
```

### Level 2: Integration Tests (Contract Interactions)

```solidity
contract IntegrationTestExample is Test {
    TokenA tokenA;
    TokenB tokenB;
    DEXContract dex;
    
    function testTokenSwap() public {
        // Test interaction between multiple contracts
        tokenA.approve(address(dex), 1000e18);
        
        uint256 outputAmount = dex.swap(address(tokenA), address(tokenB), 1000e18);
        
        assertGt(outputAmount, 0);
        assertEq(tokenB.balanceOf(address(this)), outputAmount);
    }
}
```

### Level 3: End-to-End Tests (Full User Journeys)

```solidity
contract E2ETestExample is Test {
    function testCompleteUserJourney() public {
        // 1. User deposits funds
        // 2. User stakes tokens
        // 3. Time passes, rewards accrue
        // 4. User claims rewards
        // 5. User unstakes
        // 6. User withdraws
        
        address user = makeAddr("user");
        
        // Complete user flow testing
        vm.startPrank(user);
        
        stakingContract.deposit{value: 1 ether}();
        stakingContract.stake(1 ether);
        
        vm.warp(block.timestamp + 30 days);
        
        uint256 rewards = stakingContract.claimRewards();
        stakingContract.unstake(1 ether);
        stakingContract.withdraw(1 ether);
        
        vm.stopPrank();
        
        // Verify final state
        assertGt(rewards, 0);
        assertEq(user.balance, 1 ether + rewards);
    }
}
```

## 🔍 Advanced Testing Patterns

### 1. Fuzz Testing with Constraints

```solidity
contract FuzzTestingExample is Test {
    function testFuzzWithConstraints(uint256 amount, address recipient) public {
        // Apply constraints to fuzz inputs
        vm.assume(recipient != address(0));
        vm.assume(recipient != address(this));
        amount = bound(amount, 1, token.balanceOf(address(this)));
        
        uint256 initialBalance = token.balanceOf(recipient);
        
        token.transfer(recipient, amount);
        
        assertEq(token.balanceOf(recipient), initialBalance + amount);
    }
    
    function testInvariantTotalSupply() public {
        // Invariant: total supply should never change
        uint256 initialSupply = token.totalSupply();
        
        // Perform random operations
        address[] memory users = new address[](3);
        users[0] = makeAddr("user1");
        users[1] = makeAddr("user2");
        users[2] = makeAddr("user3");
        
        for (uint i = 0; i < users.length; i++) {
            token.transfer(users[i], 1000e18);
        }
        
        assertEq(token.totalSupply(), initialSupply);
    }
}
```

### 2. Mainnet Fork Testing

```solidity
contract MainnetForkTest is Test {
    IERC20 constant USDC = IERC20(0xA0b86a33E6417C5C6a7D4b75d95d6797f3e1e4a3);
    address constant WHALE = 0x47ac0Fb4F2D84898e4D9E7b4DaB3C24507a6D503;
    
    function setUp() public {
        // Fork mainnet at specific block
        vm.createFork("https://eth-mainnet.alchemyapi.io/v2/API_KEY", 18000000);
    }
    
    function testWithRealUSDC() public {
        uint256 whaleBalance = USDC.balanceOf(WHALE);
        
        vm.prank(WHALE);
        USDC.transfer(address(this), 1000e6); // 1000 USDC
        
        assertEq(USDC.balanceOf(address(this)), 1000e6);
        assertEq(USDC.balanceOf(WHALE), whaleBalance - 1000e6);
    }
}
```

### 3. Gas Benchmarking

```solidity
contract GasBenchmarkTest is Test {
    function testGasOptimization() public {
        uint256 gasBefore = gasleft();
        
        // Operation to benchmark
        token.transfer(makeAddr("recipient"), 1000e18);
        
        uint256 gasAfter = gasleft();
        uint256 gasUsed = gasBefore - gasAfter;
        
        console.log("Gas used:", gasUsed);
        
        // Assert gas usage is within expected range
        assertLt(gasUsed, 50000); // Should use less than 50k gas
    }
}
```

## 📊 Test Coverage & Quality Metrics

### Coverage Analysis

```bash
# Foundry coverage
forge coverage

# Hardhat coverage with Istanbul
npx hardhat coverage

# Generate detailed HTML report
forge coverage --report lcov
genhtml lcov.info -o coverage-report
```

### Quality Gates

```solidity
// Test completeness checklist
contract QualityGateTest is Test {
    function testAllPublicFunctions() public {
        // Every public function should have:
        // ✅ Happy path test
        // ✅ Failure case test
        // ✅ Edge case test
        // ✅ Gas usage test
        // ✅ Event emission test
    }
    
    function testAllModifiers() public {
        // Every modifier should have:
        // ✅ Success case test
        // ✅ Failure case test
    }
    
    function testAllErrorCases() public {
        // Every revert condition should have:
        // ✅ Specific test case
        // ✅ Correct error message
    }
}
```

## 🚀 CI/CD Integration

### GitHub Actions for Foundry

```yaml
# .github/workflows/test.yml
name: Test Smart Contracts

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
      with:
        submodules: recursive
    
    - name: Install Foundry
      uses: foundry-rs/foundry-toolchain@v1
    
    - name: Run tests
      run: forge test -vvv
    
    - name: Check gas snapshots
      run: forge snapshot --check
    
    - name: Generate coverage report
      run: forge coverage --report lcov
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./lcov.info
```

### Test Automation Script

```bash
#!/bin/bash
# scripts/test-all.sh

echo "🧪 Running comprehensive test suite..."

# Run unit tests
echo "📝 Running unit tests..."
forge test --match-contract "Test" -vv

# Run integration tests  
echo "🔗 Running integration tests..."
forge test --match-contract "Integration" -vv

# Run fuzz tests
echo "🎲 Running fuzz tests..."
forge test --match-test "testFuzz" -vv

# Check coverage
echo "📊 Checking test coverage..."
forge coverage --report summary

# Gas snapshot
echo "⛽ Updating gas snapshots..."
forge snapshot

echo "✅ All tests completed!"
```

## 🎯 Production Testing Checklist

### ✅ Pre-Deployment Tests

- [ ] All functions have unit tests
- [ ] All edge cases covered
- [ ] All error conditions tested
- [ ] Gas usage within limits
- [ ] No hardcoded values in tests
- [ ] Mock contracts for external dependencies
- [ ] Fork tests with real mainnet data
- [ ] Invariant tests passing
- [ ] 100% line coverage achieved
- [ ] Fuzz tests running 10,000+ iterations

### ✅ Integration Tests

- [ ] Multi-contract interactions tested
- [ ] Cross-contract state changes verified
- [ ] External protocol integrations working
- [ ] Upgrade compatibility tested
- [ ] Event emissions verified
- [ ] Access control properly tested
- [ ] Time-dependent logic tested
- [ ] Emergency mechanisms tested

### ✅ Performance Tests

- [ ] Gas optimization validated
- [ ] Transaction throughput measured
- [ ] Memory usage optimized
- [ ] Storage access patterns efficient
- [ ] Batch operations tested
- [ ] Large dataset handling verified

---

**Status**: 🧪 Production-ready testing framework | **Update**: July 2025  
**Key Concepts**: Foundry, Hardhat, fuzz testing, invariants, mainnet forking, CI/CD integration
