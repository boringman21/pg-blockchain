# 🔗 Integration Testing: First-Principles Multi-Contract Testing

## 🧠 First-Principles Understanding: "Tại sao Integration Testing khác biệt?"

### Mental Model: Smart Contracts như "Microservices in Finance"

- **Unit tests**: Test 1 function in isolation = "Test ATM card reader"
- **Integration tests**: Test contract interactions = "Test entire banking system"
- **Key insight**: Contracts rarely work alone - they form complex ecosystems
- **2025 Reality**: 70% of DeFi hacks happen at integration points, not individual contracts

### Core Integration Philosophy: Test the Boundaries

```text
❌ Isolated Testing: Contract A works + Contract B works = System works
✅ Integration Testing: Contract A + Contract B + State Changes = Verified System

Example:
- DEX contract: ✅ swap() works
- Token contract: ✅ transfer() works  
- Integration: ❌ DEX.swap() → Token.transfer() → State corruption
```

## 🏗️ Integration Testing Architecture

### 1. Contract Dependency Mapping

```solidity
// ===== SYSTEM ARCHITECTURE =====
/*
    User Wallet
         ↓
    Router Contract ←→ Price Oracle
         ↓                 ↓
    DEX Contract    ←→ Token Contracts
         ↓                 ↓
    Liquidity Pool  ←→ Reward System
         ↓
    Governance Contract
*/

// Example DeFi ecosystem for integration testing
contract DEXSystem {
    IPriceOracle public priceOracle;
    IERC20 public tokenA;
    IERC20 public tokenB;
    ILiquidityPool public liquidityPool;
    IRewardSystem public rewardSystem;
    
    function swap(uint256 amountIn, uint256 minAmountOut) external {
        // Integration point 1: Price calculation
        uint256 price = priceOracle.getPrice(address(tokenA), address(tokenB));
        
        // Integration point 2: Token transfer
        tokenA.transferFrom(msg.sender, address(this), amountIn);
        
        // Integration point 3: Liquidity pool interaction
        uint256 amountOut = liquidityPool.executeSwap(amountIn, price);
        
        require(amountOut >= minAmountOut, "Slippage too high");
        
        // Integration point 4: Token transfer out
        tokenB.transfer(msg.sender, amountOut);
        
        // Integration point 5: Reward distribution
        rewardSystem.distributeRewards(msg.sender, amountIn);
    }
}
```

### 2. Foundry Integration Testing Framework

```solidity
// ===== INTEGRATION TEST SETUP =====
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/DEXSystem.sol";
import "../src/MockToken.sol";
import "../src/MockPriceOracle.sol";
import "../src/LiquidityPool.sol";
import "../src/RewardSystem.sol";

contract DEXIntegrationTest is Test {
    DEXSystem dex;
    MockToken tokenA;
    MockToken tokenB;
    MockPriceOracle priceOracle;
    LiquidityPool liquidityPool;
    RewardSystem rewardSystem;
    
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    address liquidityProvider = makeAddr("liquidityProvider");
    
    function setUp() public {
        // Deploy all contracts
        tokenA = new MockToken("TokenA", "TKNA", 18);
        tokenB = new MockToken("TokenB", "TKNB", 18);
        priceOracle = new MockPriceOracle();
        liquidityPool = new LiquidityPool(address(tokenA), address(tokenB));
        rewardSystem = new RewardSystem(address(tokenA));
        
        dex = new DEXSystem(
            address(priceOracle),
            address(tokenA),
            address(tokenB),
            address(liquidityPool),
            address(rewardSystem)
        );
        
        // Setup initial state
        _setupInitialLiquidity();
        _setupUserBalances();
        _setupOracle();
    }
    
    function _setupInitialLiquidity() internal {
        // Provide initial liquidity
        tokenA.mint(liquidityProvider, 1000000e18);
        tokenB.mint(liquidityProvider, 1000000e18);
        
        vm.startPrank(liquidityProvider);
        tokenA.approve(address(liquidityPool), 1000000e18);
        tokenB.approve(address(liquidityPool), 1000000e18);
        liquidityPool.addLiquidity(100000e18, 100000e18);
        vm.stopPrank();
    }
    
    function _setupUserBalances() internal {
        tokenA.mint(alice, 10000e18);
        tokenB.mint(bob, 10000e18);
    }
    
    function _setupOracle() internal {
        // Set initial price: 1 TokenA = 1.5 TokenB
        priceOracle.setPrice(address(tokenA), address(tokenB), 1.5e18);
    }
    
    // ===== INTEGRATION TEST CASES =====
    
    function testFullSwapIntegration() public {
        uint256 swapAmount = 1000e18;
        uint256 expectedOutput = 1500e18; // Based on 1:1.5 ratio
        uint256 minOutput = 1400e18; // 93% of expected (7% slippage tolerance)
        
        vm.startPrank(alice);
        tokenA.approve(address(dex), swapAmount);
        
        uint256 initialBalanceA = tokenA.balanceOf(alice);
        uint256 initialBalanceB = tokenB.balanceOf(alice);
        
        // Execute integration test
        dex.swap(swapAmount, minOutput);
        
        // Verify all integration points worked
        assertEq(tokenA.balanceOf(alice), initialBalanceA - swapAmount);
        assertGt(tokenB.balanceOf(alice), initialBalanceB + minOutput);
        
        // Verify liquidity pool state changed
        (uint256 reserveA, uint256 reserveB) = liquidityPool.getReserves();
        assertGt(reserveA, 100000e18); // Pool received tokenA
        assertLt(reserveB, 100000e18); // Pool gave tokenB
        
        vm.stopPrank();
    }
    
    function testSlippageProtectionIntegration() public {
        uint256 swapAmount = 1000e18;
        uint256 unrealisticMinOutput = 2000e18; // Higher than possible
        
        vm.prank(alice);
        tokenA.approve(address(dex), swapAmount);
        
        vm.expectRevert("Slippage too high");
        vm.prank(alice);
        dex.swap(swapAmount, unrealisticMinOutput);
    }
    
    function testRewardSystemIntegration() public {
        uint256 swapAmount = 1000e18;
        uint256 minOutput = 1400e18;
        
        vm.startPrank(alice);
        tokenA.approve(address(dex), swapAmount);
        
        uint256 initialRewards = rewardSystem.pendingRewards(alice);
        
        dex.swap(swapAmount, minOutput);
        
        uint256 finalRewards = rewardSystem.pendingRewards(alice);
        assertGt(finalRewards, initialRewards);
        
        vm.stopPrank();
    }
    
    function testOraclePriceUpdateIntegration() public {
        // Change oracle price
        priceOracle.setPrice(address(tokenA), address(tokenB), 2e18); // 1:2 ratio
        
        uint256 swapAmount = 1000e18;
        uint256 expectedOutput = 2000e18; // Based on new 1:2 ratio
        uint256 minOutput = 1900e18;
        
        vm.startPrank(alice);
        tokenA.approve(address(dex), swapAmount);
        
        uint256 initialBalance = tokenB.balanceOf(alice);
        dex.swap(swapAmount, minOutput);
        
        // Should receive more tokenB due to better price
        assertGt(tokenB.balanceOf(alice), initialBalance + minOutput);
        
        vm.stopPrank();
    }
    
    // ===== MULTI-USER INTEGRATION TESTING =====
    
    function testConcurrentSwapsIntegration() public {
        uint256 swapAmount = 500e18;
        uint256 minOutput = 700e18;
        
        // Alice swaps A→B
        vm.startPrank(alice);
        tokenA.approve(address(dex), swapAmount);
        dex.swap(swapAmount, minOutput);
        vm.stopPrank();
        
        // Bob swaps B→A (reverse direction)
        vm.startPrank(bob);
        tokenB.approve(address(dex), swapAmount);
        // Note: Would need reverse swap function in real implementation
        vm.stopPrank();
        
        // Verify pool state is consistent
        (uint256 reserveA, uint256 reserveB) = liquidityPool.getReserves();
        assertGt(reserveA + reserveB, 0); // Pool should have reserves
    }
    
    function testLiquidityAdditionDuringSwaps() public {
        // Perform swap
        vm.startPrank(alice);
        tokenA.approve(address(dex), 1000e18);
        dex.swap(1000e18, 1400e18);
        vm.stopPrank();
        
        // Add more liquidity while system is in use
        vm.startPrank(liquidityProvider);
        tokenA.approve(address(liquidityPool), 50000e18);
        tokenB.approve(address(liquidityPool), 50000e18);
        liquidityPool.addLiquidity(50000e18, 50000e18);
        vm.stopPrank();
        
        // Verify system still works after liquidity addition
        vm.startPrank(alice);
        tokenA.approve(address(dex), 1000e18);
        dex.swap(1000e18, 1400e18);
        vm.stopPrank();
    }
}

// ===== COMPLEX INTEGRATION SCENARIOS =====
contract DEXAdvancedIntegrationTest is Test {
    DEXSystem dex;
    MockToken tokenA;
    MockToken tokenB;
    MockPriceOracle priceOracle;
    LiquidityPool liquidityPool;
    RewardSystem rewardSystem;
    
    // Complex multi-step integration test
    function testCompleteUserJourney() public {
        address user = makeAddr("user");
        
        // Setup
        tokenA.mint(user, 10000e18);
        
        vm.startPrank(user);
        
        // Step 1: User adds liquidity
        tokenA.approve(address(liquidityPool), 5000e18);
        tokenB.approve(address(liquidityPool), 5000e18);
        liquidityPool.addLiquidity(5000e18, 5000e18);
        
        // Step 2: User performs swap
        tokenA.approve(address(dex), 1000e18);
        dex.swap(1000e18, 1400e18);
        
        // Step 3: User claims rewards
        rewardSystem.claimRewards();
        
        // Step 4: User removes liquidity
        uint256 lpTokens = liquidityPool.balanceOf(user);
        liquidityPool.removeLiquidity(lpTokens / 2);
        
        vm.stopPrank();
        
        // Verify final state
        assertGt(tokenB.balanceOf(user), 0);
        assertGt(rewardSystem.claimedRewards(user), 0);
    }
    
    // Test system under stress
    function testHighVolumeIntegration() public {
        address[] memory users = new address[](10);
        
        // Create multiple users
        for (uint i = 0; i < 10; i++) {
            users[i] = makeAddr(string(abi.encodePacked("user", i)));
            tokenA.mint(users[i], 1000e18);
        }
        
        // All users perform swaps simultaneously
        for (uint i = 0; i < 10; i++) {
            vm.startPrank(users[i]);
            tokenA.approve(address(dex), 100e18);
            dex.swap(100e18, 140e18);
            vm.stopPrank();
        }
        
        // Verify system integrity
        (uint256 reserveA, uint256 reserveB) = liquidityPool.getReserves();
        assertGt(reserveA, 100000e18);
        assertLt(reserveB, 100000e18);
        
        // Verify all users received tokens
        for (uint i = 0; i < 10; i++) {
            assertGt(tokenB.balanceOf(users[i]), 140e18);
        }
    }
}
```

### 3. Hardhat Integration Testing

```javascript
// ===== HARDHAT INTEGRATION TESTS =====
const { expect } = require("chai");
const { ethers } = require("hardhat");
const { loadFixture } = require("@nomicfoundation/hardhat-network-helpers");

describe("DEX Integration Tests", function() {
    async function deployDEXSystemFixture() {
        const [owner, alice, bob, liquidityProvider] = await ethers.getSigners();
        
        // Deploy all contracts
        const MockToken = await ethers.getContractFactory("MockToken");
        const tokenA = await MockToken.deploy("TokenA", "TKNA");
        const tokenB = await MockToken.deploy("TokenB", "TKNB");
        
        const MockPriceOracle = await ethers.getContractFactory("MockPriceOracle");
        const priceOracle = await MockPriceOracle.deploy();
        
        const LiquidityPool = await ethers.getContractFactory("LiquidityPool");
        const liquidityPool = await LiquidityPool.deploy(tokenA.address, tokenB.address);
        
        const RewardSystem = await ethers.getContractFactory("RewardSystem");
        const rewardSystem = await RewardSystem.deploy(tokenA.address);
        
        const DEXSystem = await ethers.getContractFactory("DEXSystem");
        const dex = await DEXSystem.deploy(
            priceOracle.address,
            tokenA.address,
            tokenB.address,
            liquidityPool.address,
            rewardSystem.address
        );
        
        // Setup initial state
        await tokenA.mint(liquidityProvider.address, ethers.utils.parseEther("1000000"));
        await tokenB.mint(liquidityProvider.address, ethers.utils.parseEther("1000000"));
        await tokenA.mint(alice.address, ethers.utils.parseEther("10000"));
        
        await tokenA.connect(liquidityProvider).approve(liquidityPool.address, ethers.utils.parseEther("1000000"));
        await tokenB.connect(liquidityProvider).approve(liquidityPool.address, ethers.utils.parseEther("1000000"));
        await liquidityPool.connect(liquidityProvider).addLiquidity(
            ethers.utils.parseEther("100000"),
            ethers.utils.parseEther("100000")
        );
        
        await priceOracle.setPrice(tokenA.address, tokenB.address, ethers.utils.parseEther("1.5"));
        
        return { dex, tokenA, tokenB, priceOracle, liquidityPool, rewardSystem, owner, alice, bob, liquidityProvider };
    }
    
    describe("Full System Integration", function() {
        it("Should handle complete swap flow", async function() {
            const { dex, tokenA, tokenB, alice } = await loadFixture(deployDEXSystemFixture);
            
            const swapAmount = ethers.utils.parseEther("1000");
            const minOutput = ethers.utils.parseEther("1400");
            
            await tokenA.connect(alice).approve(dex.address, swapAmount);
            
            const initialBalanceA = await tokenA.balanceOf(alice.address);
            const initialBalanceB = await tokenB.balanceOf(alice.address);
            
            await expect(dex.connect(alice).swap(swapAmount, minOutput))
                .to.emit(tokenA, "Transfer") // Should emit transfer events
                .and.to.emit(tokenB, "Transfer");
            
            expect(await tokenA.balanceOf(alice.address)).to.equal(initialBalanceA.sub(swapAmount));
            expect(await tokenB.balanceOf(alice.address)).to.be.gt(initialBalanceB.add(minOutput));
        });
        
        it("Should integrate with reward system", async function() {
            const { dex, tokenA, rewardSystem, alice } = await loadFixture(deployDEXSystemFixture);
            
            const swapAmount = ethers.utils.parseEther("1000");
            const minOutput = ethers.utils.parseEther("1400");
            
            await tokenA.connect(alice).approve(dex.address, swapAmount);
            
            const initialRewards = await rewardSystem.pendingRewards(alice.address);
            
            await dex.connect(alice).swap(swapAmount, minOutput);
            
            const finalRewards = await rewardSystem.pendingRewards(alice.address);
            expect(finalRewards).to.be.gt(initialRewards);
        });
        
        it("Should handle oracle price updates", async function() {
            const { dex, tokenA, tokenB, priceOracle, alice } = await loadFixture(deployDEXSystemFixture);
            
            // Change price to 1:2 ratio
            await priceOracle.setPrice(tokenA.address, tokenB.address, ethers.utils.parseEther("2"));
            
            const swapAmount = ethers.utils.parseEther("1000");
            const minOutput = ethers.utils.parseEther("1900"); // Higher due to better price
            
            await tokenA.connect(alice).approve(dex.address, swapAmount);
            
            const initialBalance = await tokenB.balanceOf(alice.address);
            await dex.connect(alice).swap(swapAmount, minOutput);
            
            expect(await tokenB.balanceOf(alice.address)).to.be.gt(initialBalance.add(minOutput));
        });
    });
    
    describe("Cross-Contract State Management", function() {
        it("Should maintain consistent state across all contracts", async function() {
            const { dex, tokenA, tokenB, liquidityPool, alice } = await loadFixture(deployDEXSystemFixture);
            
            const swapAmount = ethers.utils.parseEther("1000");
            const minOutput = ethers.utils.parseEther("1400");
            
            // Get initial reserves
            const [initialReserveA, initialReserveB] = await liquidityPool.getReserves();
            
            await tokenA.connect(alice).approve(dex.address, swapAmount);
            await dex.connect(alice).swap(swapAmount, minOutput);
            
            // Verify reserves changed correctly
            const [finalReserveA, finalReserveB] = await liquidityPool.getReserves();
            expect(finalReserveA).to.be.gt(initialReserveA);
            expect(finalReserveB).to.be.lt(initialReserveB);
        });
    });
    
    describe("Error Propagation", function() {
        it("Should properly propagate errors across contracts", async function() {
            const { dex, tokenA, alice } = await loadFixture(deployDEXSystemFixture);
            
            const swapAmount = ethers.utils.parseEther("1000");
            const unrealisticMinOutput = ethers.utils.parseEther("3000"); // Impossible to achieve
            
            await tokenA.connect(alice).approve(dex.address, swapAmount);
            
            await expect(dex.connect(alice).swap(swapAmount, unrealisticMinOutput))
                .to.be.revertedWith("Slippage too high");
        });
        
        it("Should handle insufficient allowance", async function() {
            const { dex, alice } = await loadFixture(deployDEXSystemFixture);
            
            const swapAmount = ethers.utils.parseEther("1000");
            const minOutput = ethers.utils.parseEther("1400");
            
            // Don't approve tokens
            await expect(dex.connect(alice).swap(swapAmount, minOutput))
                .to.be.revertedWith("ERC20: insufficient allowance");
        });
    });
});

// ===== MAINNET FORK INTEGRATION TESTING =====
describe("Mainnet Fork Integration", function() {
    beforeEach(async function() {
        // Fork Ethereum mainnet
        await network.provider.request({
            method: "hardhat_reset",
            params: [
                {
                    forking: {
                        jsonRpcUrl: "https://eth-mainnet.alchemyapi.io/v2/YOUR_API_KEY",
                        blockNumber: 18000000
                    }
                }
            ]
        });
    });
    
    it("Should integrate with real Uniswap", async function() {
        const UNISWAP_V2_ROUTER = "0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D";
        const USDC = "0xA0b86a33E6417C5C6a7D4b75d95d6797f3e1e4a3";
        const WETH = "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2";
        
        const router = await ethers.getContractAt("IUniswapV2Router02", UNISWAP_V2_ROUTER);
        const usdc = await ethers.getContractAt("IERC20", USDC);
        
        // Get USDC from a whale
        const whale = "0x47ac0Fb4F2D84898e4D9E7b4DaB3C24507a6D503";
        await hre.network.provider.request({
            method: "hardhat_impersonateAccount",
            params: [whale]
        });
        
        const whaleSigner = await ethers.getSigner(whale);
        await usdc.connect(whaleSigner).transfer(await ethers.getSigner().getAddress(), 1000e6);
        
        // Test real integration
        const [owner] = await ethers.getSigners();
        await usdc.approve(router.address, 1000e6);
        
        const path = [USDC, WETH];
        const amountIn = 1000e6;
        const amountOutMin = 0;
        const deadline = Math.floor(Date.now() / 1000) + 3600;
        
        await expect(router.swapExactTokensForTokens(
            amountIn,
            amountOutMin,
            path,
            owner.address,
            deadline
        )).to.not.be.reverted;
    });
});
```

## 🧪 Advanced Integration Testing Patterns

### 1. State Transition Testing

```solidity
contract StateTransitionTest is Test {
    enum SystemState { INACTIVE, ACTIVE, PAUSED, EMERGENCY }
    
    SystemState currentState;
    
    function testStateTransitions() public {
        // Test all valid state transitions
        currentState = SystemState.INACTIVE;
        
        // INACTIVE → ACTIVE
        system.activate();
        assertEq(uint(system.getState()), uint(SystemState.ACTIVE));
        
        // ACTIVE → PAUSED
        system.pause();
        assertEq(uint(system.getState()), uint(SystemState.PAUSED));
        
        // PAUSED → ACTIVE
        system.unpause();
        assertEq(uint(system.getState()), uint(SystemState.ACTIVE));
        
        // ACTIVE → EMERGENCY
        system.emergencyStop();
        assertEq(uint(system.getState()), uint(SystemState.EMERGENCY));
    }
    
    function testInvalidStateTransitions() public {
        // Test invalid transitions fail
        currentState = SystemState.INACTIVE;
        
        vm.expectRevert("Invalid state transition");
        system.pause(); // Can't pause when inactive
        
        vm.expectRevert("Invalid state transition");
        system.emergencyStop(); // Can't emergency stop when inactive
    }
}
```

### 2. Time-Based Integration Testing

```solidity
contract TimeDependentIntegrationTest is Test {
    function testStakingRewardsIntegration() public {
        uint256 stakeAmount = 1000e18;
        address user = makeAddr("user");
        
        token.mint(user, stakeAmount);
        
        vm.startPrank(user);
        token.approve(address(stakingContract), stakeAmount);
        stakingContract.stake(stakeAmount);
        vm.stopPrank();
        
        // Fast forward 30 days
        vm.warp(block.timestamp + 30 days);
        
        // Check rewards accumulated
        uint256 rewards = stakingContract.pendingRewards(user);
        assertGt(rewards, 0);
        
        // Claim rewards
        vm.prank(user);
        stakingContract.claimRewards();
        
        assertEq(stakingContract.pendingRewards(user), 0);
        assertGt(rewardToken.balanceOf(user), 0);
    }
}
```

### 3. Event Chain Integration Testing

```solidity
contract EventChainTest is Test {
    function testEventChainIntegration() public {
        vm.expectEmit(true, true, false, true);
        emit Deposit(user, 1000e18);
        
        vm.expectEmit(true, true, false, true);
        emit Stake(user, 1000e18);
        
        vm.expectEmit(true, true, false, true);
        emit RewardCalculated(user, expectedReward);
        
        // Execute operation that should trigger all events
        vm.prank(user);
        stakingContract.depositAndStake{value: 1000e18}();
    }
}
```

## 🚀 Production Integration Testing Checklist

### ✅ Contract Interaction Testing

- [ ] All contract-to-contract calls tested
- [ ] Cross-contract state changes verified
- [ ] Token transfer integrations working
- [ ] External protocol integrations tested
- [ ] Oracle price feed integrations verified
- [ ] Governance integration working
- [ ] Multi-signature wallet integration tested

### ✅ System State Testing

- [ ] State transitions tested
- [ ] Concurrent operations handled
- [ ] Emergency mechanisms working
- [ ] Pause/unpause functionality tested
- [ ] Upgrade mechanisms verified
- [ ] Recovery procedures tested

### ✅ Performance Integration

- [ ] High-volume transaction testing
- [ ] Gas usage under load measured
- [ ] Memory usage optimization verified
- [ ] Transaction ordering tested
- [ ] MEV protection verified
- [ ] Slippage protection working

### ✅ Security Integration

- [ ] Reentrancy attack protection
- [ ] Flash loan attack protection
- [ ] Oracle manipulation protection
- [ ] Front-running protection
- [ ] Access control integration
- [ ] Multi-signature requirements enforced

---

**Status**: 🔗 Production-ready integration testing | **Update**: July 2025  
**Key Concepts**: Multi-contract testing, state management, event chains, mainnet forking, system integration
