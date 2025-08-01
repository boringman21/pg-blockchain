# 🔨 Advanced Solidity: Patterns, Optimization & Security

## 🎯 First-Principles Thinking về Advanced Solidity

### Tại sao cần học Advanced Solidity?

**Vấn đề cơ bản**: Basic Solidity chỉ đủ cho demo, không đủ cho production

1. **Gas Optimization**: Tiết kiệm chi phí transaction
2. **Security Patterns**: Tránh các lỗ hổng phổ biến  
3. **Upgradeability**: Khả năng nâng cấp contract
4. **Scalability**: Xử lý large-scale applications

## 🧠 Gas Optimization Techniques

### 1. Storage vs Memory vs Calldata

```solidity
contract GasOptimization {
    struct User {
        uint256 id;
        string name;
        uint256 balance;
        bool active;
    }
    
    mapping(uint256 => User) public users;
    uint256[] public userIds;
    
    // ❌ Expensive - multiple SSTORE operations
    function createUserBad(uint256 _id, string memory _name) external {
        users[_id].id = _id;
        users[_id].name = _name;
        users[_id].balance = 0;
        users[_id].active = true;
        userIds.push(_id);
    }
    
    // ✅ Cheaper - single SSTORE operation
    function createUserGood(uint256 _id, string calldata _name) external {
        users[_id] = User({
            id: _id,
            name: _name,
            balance: 0,
            active: true
        });
        userIds.push(_id);
    }
    
    // ✅ Use calldata for read-only arrays
    function processUsers(uint256[] calldata _userIds) external view returns (uint256) {
        uint256 total = 0;
        for (uint256 i = 0; i < _userIds.length; i++) {
            total += users[_userIds[i]].balance;
        }
        return total;
    }
}
```

### 2. Packed Structs & Variable Ordering

```solidity
// ❌ Bad ordering - uses 3 storage slots
struct BadStruct {
    uint128 a;  // slot 0 (partial)
    uint256 b;  // slot 1 (full)
    uint128 c;  // slot 2 (partial)
}

// ✅ Good ordering - uses 2 storage slots  
struct GoodStruct {
    uint128 a;  // slot 0 (partial)
    uint128 c;  // slot 0 (remaining)
    uint256 b;  // slot 1 (full)
}

// ✅ Ultra-optimized for specific use cases
struct PackedData {
    uint64 timestamp;   // 8 bytes
    uint64 price;       // 8 bytes  
    uint32 volume;      // 4 bytes
    uint32 reserved;    // 4 bytes
    bool active;        // 1 byte
    // Total: 25 bytes → fits in 1 storage slot (32 bytes)
}
```

### 3. Assembly Optimization

```solidity
contract AssemblyOptimization {
    // ✅ Assembly for efficient array sum
    function sumArray(uint256[] calldata arr) external pure returns (uint256 sum) {
        assembly {
            let len := arr.length
            let ptr := arr.offset
            
            for { let i := 0 } lt(i, len) { i := add(i, 1) } {
                sum := add(sum, calldataload(add(ptr, mul(i, 0x20))))
            }
        }
    }
    
    // ✅ Assembly for efficient address validation
    function isContract(address account) internal view returns (bool result) {
        assembly {
            result := gt(extcodesize(account), 0)
        }
    }
    
    // ✅ Assembly for efficient string length
    function getStringLength(string memory str) internal pure returns (uint256 length) {
        assembly {
            length := mload(str)
        }
    }
}
```

## 🛡️ Security Patterns & Best Practices

### 1. Reentrancy Protection

```solidity
// ✅ ReentrancyGuard pattern
abstract contract ReentrancyGuard {
    uint256 private constant _NOT_ENTERED = 1;
    uint256 private constant _ENTERED = 2;
    
    uint256 private _status;
    
    constructor() {
        _status = _NOT_ENTERED;
    }
    
    modifier nonReentrant() {
        require(_status != _ENTERED, "ReentrancyGuard: reentrant call");
        _status = _ENTERED;
        _;
        _status = _NOT_ENTERED;
    }
}

contract SecureWallet is ReentrancyGuard {
    mapping(address => uint256) public balances;
    
    // ✅ Secure withdrawal with checks-effects-interactions
    function withdraw(uint256 amount) external nonReentrant {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        
        // Effects: Update state before external call
        balances[msg.sender] -= amount;
        
        // Interactions: External call comes last
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

### 2. Access Control Patterns

```solidity
// ✅ Role-based access control
import "@openzeppelin/contracts/access/AccessControl.sol";

contract AdvancedAccessControl is AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant BURNER_ROLE = keccak256("BURNER_ROLE");
    bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");
    
    constructor() {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(MINTER_ROLE, msg.sender);
    }
    
    // ✅ Multiple role requirements
    modifier onlyMinterOrAdmin() {
        require(
            hasRole(MINTER_ROLE, msg.sender) || hasRole(DEFAULT_ADMIN_ROLE, msg.sender),
            "Caller is not a minter or admin"
        );
        _;
    }
    
    // ✅ Time-locked admin functions
    mapping(bytes32 => uint256) public pendingRoleChanges;
    uint256 public constant TIMELOCK_DELAY = 2 days;
    
    function proposeRoleChange(address account, bytes32 role) external onlyRole(DEFAULT_ADMIN_ROLE) {
        bytes32 proposalId = keccak256(abi.encodePacked(account, role, block.timestamp));
        pendingRoleChanges[proposalId] = block.timestamp + TIMELOCK_DELAY;
    }
    
    function executeRoleChange(address account, bytes32 role, uint256 timestamp) external {
        bytes32 proposalId = keccak256(abi.encodePacked(account, role, timestamp));
        require(
            pendingRoleChanges[proposalId] != 0 && 
            block.timestamp >= pendingRoleChanges[proposalId],
            "Timelock not expired"
        );
        
        _grantRole(role, account);
        delete pendingRoleChanges[proposalId];
    }
}
```

### 3. Circuit Breaker Pattern

```solidity
contract CircuitBreaker {
    bool public paused = false;
    uint256 public pausedUntil;
    
    // ✅ Emergency pause with automatic recovery
    modifier whenNotPaused() {
        require(!paused || block.timestamp > pausedUntil, "Contract is paused");
        _;
    }
    
    function emergencyPause(uint256 duration) external onlyOwner {
        paused = true;
        pausedUntil = block.timestamp + duration;
        emit EmergencyPause(duration);
    }
    
    // ✅ Automatic circuit breaker based on conditions
    uint256 public maxDailyWithdrawal = 100 ether;
    uint256 public dailyWithdrawn;
    uint256 public lastResetDay;
    
    modifier withinDailyLimit(uint256 amount) {
        if (block.timestamp / 1 days > lastResetDay) {
            dailyWithdrawn = 0;
            lastResetDay = block.timestamp / 1 days;
        }
        
        require(dailyWithdrawn + amount <= maxDailyWithdrawal, "Daily limit exceeded");
        dailyWithdrawn += amount;
        _;
    }
}
```

## 🔄 Upgrade Patterns

### 1. Proxy Pattern Implementation

```solidity
// ✅ UUPS (Universal Upgradeable Proxy Standard)
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";

contract MyContractV1 is UUPSUpgradeable, OwnableUpgradeable {
    uint256 public value;
    
    function initialize(uint256 _value) public initializer {
        __Ownable_init();
        __UUPSUpgradeable_init();
        value = _value;
    }
    
    function setValue(uint256 _value) external onlyOwner {
        value = _value;
    }
    
    // ✅ Authorization for upgrades
    function _authorizeUpgrade(address newImplementation) internal override onlyOwner {}
    
    // ✅ Version tracking
    function version() external pure returns (string memory) {
        return "1.0.0";
    }
}

// V2 with new features
contract MyContractV2 is MyContractV1 {
    uint256 public newFeature;
    
    function setNewFeature(uint256 _value) external onlyOwner {
        newFeature = _value;
    }
    
    function version() external pure override returns (string memory) {
        return "2.0.0";
    }
}
```

### 2. Diamond Pattern (EIP-2535)

```solidity
// ✅ Diamond standard for unlimited upgrades
contract Diamond {
    struct FacetCut {
        address facetAddress;
        FacetCutAction action;
        bytes4[] functionSelectors;
    }
    
    enum FacetCutAction {
        Add,
        Replace,
        Remove
    }
    
    mapping(bytes4 => address) public facets;
    
    function diamondCut(FacetCut[] calldata _diamondCut) external {
        for (uint256 i = 0; i < _diamondCut.length; i++) {
            FacetCut memory cut = _diamondCut[i];
            
            if (cut.action == FacetCutAction.Add) {
                addFunctions(cut.facetAddress, cut.functionSelectors);
            } else if (cut.action == FacetCutAction.Replace) {
                replaceFunctions(cut.facetAddress, cut.functionSelectors);
            } else if (cut.action == FacetCutAction.Remove) {
                removeFunctions(cut.facetAddress, cut.functionSelectors);
            }
        }
    }
    
    fallback() external payable {
        address facet = facets[msg.sig];
        require(facet != address(0), "Function not found");
        
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}
```

## 📊 Advanced Design Patterns

### 1. Factory Pattern

```solidity
// ✅ Factory for creating standardized contracts
contract TokenFactory {
    event TokenCreated(address indexed token, address indexed creator, string name);
    
    address[] public allTokens;
    mapping(address => address[]) public userTokens;
    
    function createToken(
        string memory name,
        string memory symbol,
        uint256 initialSupply
    ) external returns (address) {
        // Use CREATE2 for deterministic addresses
        bytes32 salt = keccak256(abi.encodePacked(msg.sender, name, block.timestamp));
        
        StandardToken token = new StandardToken{salt: salt}(
            name,
            symbol,
            initialSupply,
            msg.sender
        );
        
        address tokenAddress = address(token);
        allTokens.push(tokenAddress);
        userTokens[msg.sender].push(tokenAddress);
        
        emit TokenCreated(tokenAddress, msg.sender, name);
        return tokenAddress;
    }
    
    function predictTokenAddress(
        string memory name,
        address creator,
        uint256 timestamp
    ) external view returns (address) {
        bytes32 salt = keccak256(abi.encodePacked(creator, name, timestamp));
        bytes32 hash = keccak256(
            abi.encodePacked(
                bytes1(0xff),
                address(this),
                salt,
                keccak256(type(StandardToken).creationCode)
            )
        );
        return address(uint160(uint256(hash)));
    }
}
```

### 2. Registry Pattern

```solidity
// ✅ Decentralized registry for contracts
contract ContractRegistry {
    struct ContractInfo {
        address contractAddress;
        string version;
        uint256 deployedAt;
        bool active;
    }
    
    mapping(bytes32 => ContractInfo[]) public contracts;
    mapping(address => bool) public authorizedDeployers;
    
    modifier onlyAuthorized() {
        require(authorizedDeployers[msg.sender], "Not authorized");
        _;
    }
    
    function registerContract(
        string memory name,
        address contractAddress,
        string memory version
    ) external onlyAuthorized {
        bytes32 nameHash = keccak256(abi.encodePacked(name));
        
        contracts[nameHash].push(ContractInfo({
            contractAddress: contractAddress,
            version: version,
            deployedAt: block.timestamp,
            active: true
        }));
        
        emit ContractRegistered(name, contractAddress, version);
    }
    
    function getLatestContract(string memory name) external view returns (ContractInfo memory) {
        bytes32 nameHash = keccak256(abi.encodePacked(name));
        ContractInfo[] memory versions = contracts[nameHash];
        
        require(versions.length > 0, "Contract not found");
        
        // Return latest active version
        for (uint256 i = versions.length; i > 0; i--) {
            if (versions[i-1].active) {
                return versions[i-1];
            }
        }
        
        revert("No active version found");
    }
}
```

### 3. Oracle Integration Pattern

```solidity
// ✅ Secure oracle integration with multiple sources
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

contract PriceOracle {
    struct PriceFeed {
        AggregatorV3Interface chainlinkFeed;
        uint256 maxStaleness;
        uint256 minPrice;
        uint256 maxPrice;
        bool active;
    }
    
    mapping(string => PriceFeed) public priceFeeds;
    mapping(string => uint256) public manualPrices;
    
    modifier validPrice(string memory asset, uint256 price) {
        PriceFeed memory feed = priceFeeds[asset];
        require(price >= feed.minPrice && price <= feed.maxPrice, "Price out of bounds");
        _;
    }
    
    function getPrice(string memory asset) external view returns (uint256, uint256) {
        PriceFeed memory feed = priceFeeds[asset];
        require(feed.active, "Price feed not active");
        
        (
            uint80 roundId,
            int256 price,
            uint256 startedAt,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = feed.chainlinkFeed.latestRoundData();
        
        require(price > 0, "Invalid price");
        require(block.timestamp - updatedAt <= feed.maxStaleness, "Price too stale");
        
        return (uint256(price), updatedAt);
    }
    
    // ✅ Fallback to manual price with multi-sig
    function setManualPrice(
        string memory asset,
        uint256 price,
        bytes[] memory signatures
    ) external validPrice(asset, price) {
        require(signatures.length >= 3, "Insufficient signatures");
        
        // Verify signatures from trusted oracles
        bytes32 messageHash = keccak256(abi.encodePacked(asset, price, block.timestamp));
        // ... signature verification logic
        
        manualPrices[asset] = price;
    }
}
```

## 🧪 Testing Advanced Contracts

### 1. Foundry Testing Patterns

```solidity
// test/AdvancedTest.t.sol
import "forge-std/Test.sol";
import "../src/MyContract.sol";

contract AdvancedTest is Test {
    MyContract public myContract;
    
    function setUp() public {
        myContract = new MyContract();
    }
    
    // ✅ Fuzz testing
    function testFuzzTransfer(uint256 amount) public {
        vm.assume(amount > 0 && amount < type(uint128).max);
        
        myContract.mint(address(this), amount);
        uint256 balanceBefore = myContract.balanceOf(address(this));
        
        myContract.transfer(address(0x1), amount);
        
        assertEq(myContract.balanceOf(address(this)), balanceBefore - amount);
        assertEq(myContract.balanceOf(address(0x1)), amount);
    }
    
    // ✅ Invariant testing
    function invariant_totalSupplyEqualsBalances() public {
        uint256 totalSupply = myContract.totalSupply();
        uint256 sumOfBalances = 0;
        
        // In a real test, iterate through all holders
        // For demo, simplified
        
        assertEq(totalSupply, sumOfBalances);
    }
    
    // ✅ Time manipulation
    function testTimeBasedFunction() public {
        uint256 startTime = block.timestamp;
        
        vm.warp(startTime + 1 days);
        myContract.dailyFunction();
        
        assertTrue(myContract.lastExecuted() == startTime + 1 days);
    }
    
    // ✅ Event testing
    function testEmitsEvent() public {
        vm.expectEmit(true, true, false, true);
        emit Transfer(address(0), address(this), 100);
        
        myContract.mint(address(this), 100);
    }
}
```

### 2. Gas Benchmarking

```solidity
contract GasBenchmark is Test {
    function testGasUsage() public {
        uint256 gasBefore = gasleft();
        
        // Function call
        myContract.expensiveFunction();
        
        uint256 gasUsed = gasBefore - gasleft();
        console.log("Gas used:", gasUsed);
        
        // Assert gas usage is within expected range
        assertLt(gasUsed, 100000, "Gas usage too high");
    }
    
    // ✅ Compare different implementations
    function testGasComparison() public {
        uint256 gas1 = measureGas(() => myContract.implementation1());
        uint256 gas2 = measureGas(() => myContract.implementation2());
        
        console.log("Implementation 1 gas:", gas1);
        console.log("Implementation 2 gas:", gas2);
        console.log("Gas saved:", gas1 > gas2 ? gas1 - gas2 : gas2 - gas1);
    }
    
    function measureGas(function() internal fnToMeasure) internal returns (uint256) {
        uint256 gasBefore = gasleft();
        fnToMeasure();
        return gasBefore - gasleft();
    }
}
```

## 📚 Advanced Development Tools

### 1. Custom Hardhat Tasks

```typescript
// hardhat.config.ts
import { task } from "hardhat/config";

task("deploy-factory", "Deploy token factory")
  .addParam("owner", "The owner address")
  .setAction(async (taskArgs, hre) => {
    const TokenFactory = await hre.ethers.getContractFactory("TokenFactory");
    const factory = await TokenFactory.deploy(taskArgs.owner);
    
    await factory.deployed();
    console.log(`TokenFactory deployed to: ${factory.address}`);
    
    // Verify on Etherscan
    await hre.run("verify:verify", {
      address: factory.address,
      constructorArguments: [taskArgs.owner],
    });
  });

task("gas-report", "Generate gas usage report")
  .setAction(async (taskArgs, hre) => {
    const contracts = ["MyContract", "TokenFactory"];
    
    for (const contractName of contracts) {
      const Contract = await hre.ethers.getContractFactory(contractName);
      const contract = await Contract.deploy();
      
      console.log(`${contractName} deployment gas:`, contract.deployTransaction.gasUsed?.toString());
      
      // Test major functions
      const tx = await contract.someFunction();
      const receipt = await tx.wait();
      console.log(`${contractName}.someFunction gas:`, receipt.gasUsed.toString());
    }
  });
```

### 2. Slither Integration

```yaml
# slither.config.json
{
  "detectors_to_run": [
    "unprotected-upgrade",
    "suicidal",
    "controlled-delegatecall",
    "reentrancy-eth",
    "reentrancy-no-eth"
  ],
  "detectors_to_exclude": [
    "pragma",
    "solc-version"
  ],
  "filter_paths": [
    "node_modules",
    "lib"
  ]
}
```

```bash
# Run security analysis
slither . --config-file slither.config.json
```

## 📊 Performance Monitoring

### 1. On-chain Analytics

```solidity
contract Analytics {
    struct FunctionCall {
        bytes4 selector;
        uint256 gasUsed;
        uint256 timestamp;
        address caller;
    }
    
    FunctionCall[] public functionCalls;
    mapping(bytes4 => uint256) public totalGasUsed;
    mapping(bytes4 => uint256) public callCount;
    
    modifier trackGas() {
        uint256 gasBefore = gasleft();
        _;
        uint256 gasUsed = gasBefore - gasleft();
        
        functionCalls.push(FunctionCall({
            selector: msg.sig,
            gasUsed: gasUsed,
            timestamp: block.timestamp,
            caller: msg.sender
        }));
        
        totalGasUsed[msg.sig] += gasUsed;
        callCount[msg.sig]++;
    }
    
    function getAverageGas(bytes4 selector) external view returns (uint256) {
        return totalGasUsed[selector] / callCount[selector];
    }
}
```

### 2. Upgrade Monitoring

```solidity
contract UpgradeMonitor {
    event UpgradeProposed(address indexed newImplementation, uint256 timelock);
    event UpgradeExecuted(address indexed oldImplementation, address indexed newImplementation);
    event UpgradeCancelled(address indexed proposedImplementation);
    
    mapping(address => uint256) public proposedUpgrades;
    uint256 public constant UPGRADE_DELAY = 7 days;
    
    function proposeUpgrade(address newImplementation) external onlyOwner {
        proposedUpgrades[newImplementation] = block.timestamp + UPGRADE_DELAY;
        emit UpgradeProposed(newImplementation, block.timestamp + UPGRADE_DELAY);
    }
    
    function executeUpgrade(address newImplementation) external onlyOwner {
        require(
            proposedUpgrades[newImplementation] != 0 &&
            block.timestamp >= proposedUpgrades[newImplementation],
            "Upgrade not ready"
        );
        
        address oldImplementation = _getImplementation();
        _upgradeTo(newImplementation);
        
        delete proposedUpgrades[newImplementation];
        emit UpgradeExecuted(oldImplementation, newImplementation);
    }
}
```

## 🔗 Integration với DeFi Protocols

### 1. Uniswap V3 Integration

```solidity
import "@uniswap/v3-periphery/contracts/interfaces/ISwapRouter.sol";

contract UniswapIntegration {
    ISwapRouter public constant swapRouter = ISwapRouter(0xE592427A0AEce92De3Edee1F18E0157C05861564);
    
    function swapExactInputSingle(
        address tokenIn,
        address tokenOut,
        uint24 fee,
        uint256 amountIn,
        uint256 amountOutMinimum
    ) external returns (uint256 amountOut) {
        IERC20(tokenIn).transferFrom(msg.sender, address(this), amountIn);
        IERC20(tokenIn).approve(address(swapRouter), amountIn);
        
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
            tokenIn: tokenIn,
            tokenOut: tokenOut,
            fee: fee,
            recipient: msg.sender,
            deadline: block.timestamp + 300,
            amountIn: amountIn,
            amountOutMinimum: amountOutMinimum,
            sqrtPriceLimitX96: 0
        });
        
        amountOut = swapRouter.exactInputSingle(params);
    }
}
```

### 2. Compound Integration

```solidity
import "./interfaces/CErc20.sol";

contract CompoundIntegration {
    mapping(address => address) public cTokens; // token => cToken
    
    function supply(address token, uint256 amount) external {
        address cToken = cTokens[token];
        require(cToken != address(0), "Market not supported");
        
        IERC20(token).transferFrom(msg.sender, address(this), amount);
        IERC20(token).approve(cToken, amount);
        
        require(CErc20(cToken).mint(amount) == 0, "Mint failed");
    }
    
    function withdraw(address token, uint256 cTokenAmount) external {
        address cToken = cTokens[token];
        require(CErc20(cToken).redeem(cTokenAmount) == 0, "Redeem failed");
    }
}
```

## 🎯 Best Practices Summary

### 1. Development Workflow

```bash
# 1. Setup
forge init my-project
cd my-project

# 2. Install dependencies
forge install OpenZeppelin/openzeppelin-contracts
forge install foundry-rs/forge-std

# 3. Write tests first (TDD)
forge test

# 4. Gas optimization
forge test --gas-report

# 5. Security analysis
slither .

# 6. Deploy to testnet
forge script script/Deploy.s.sol --rpc-url sepolia --broadcast

# 7. Verify contract
forge verify-contract <address> MyContract --etherscan-api-key <key>
```

### 2. Security Checklist

- [ ] Reentrancy protection implemented
- [ ] Access controls properly configured
- [ ] Integer overflow/underflow prevented
- [ ] External calls handled safely
- [ ] Circuit breakers implemented
- [ ] Upgrade mechanisms secured
- [ ] Events emitted for important state changes
- [ ] Gas limits considered
- [ ] Time-dependent logic secured
- [ ] Oracle manipulation prevented

### 3. Gas Optimization Checklist

- [ ] Storage layout optimized
- [ ] Unnecessary storage reads eliminated
- [ ] Calldata used instead of memory where possible
- [ ] Assembly used for critical paths
- [ ] Function visibility optimized
- [ ] Modifiers usage optimized
- [ ] Error messages shortened
- [ ] Constant and immutable variables used
- [ ] Batch operations implemented
- [ ] External calls minimized

---

**Status**: 🚀 Production-ready advanced patterns | **Update**: July 2025
