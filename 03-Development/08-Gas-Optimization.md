# ⛽ Gas Optimization: First-Principles Approach to Efficiency

## 🎯 First-Principles Gas Optimization Thinking

### Fundamental Question: "Tại sao gas lại tốn kém?"

**Giải thích từ gốc rễ:**
1. **Every operation costs gas** - Mỗi opcode trong EVM đều có giá
2. **Storage is expensive** - Ghi data vào blockchain = permanent cost  
3. **Computation vs Storage tradeoff** - Tính toán rẻ hơn lưu trữ
4. **Gas limit constraints** - Block có giới hạn gas để process
5. **Network congestion** - Nhiều người dùng = gas price tăng

### Mental Model: Gas như "CPU cycles + Storage costs"

```solidity
// 🔴 Mental model: Mỗi dòng code = gas cost
function expensiveFunction() public {
    uint256 a = 1;           // SSTORE: 20,000 gas
    uint256 b = 2;           // SSTORE: 20,000 gas  
    uint256 c = a + b;       // SLOAD + SLOAD + ADD + SSTORE: ~45,000 gas
    // Total: ~85,000 gas cho 3 dòng code đơn giản!
}
```

## 🧠 Core Gas Optimization Principles

### 1. Storage Optimization (Principle: Minimize State Changes)

```solidity
// ❌ BAD: Multiple storage writes
contract BadStorage {
    uint256 public count;
    address public lastUser;
    uint256 public lastTime;
    
    function updateStats(address user) external {
        count++;              // SSTORE: 20,000 gas
        lastUser = user;      // SSTORE: 20,000 gas
        lastTime = block.timestamp; // SSTORE: 20,000 gas
        // Total: 60,000 gas
    }
}

// ✅ GOOD: Packed storage + single write
contract GoodStorage {
    struct Stats {
        uint128 count;        // Packed with address (32 bytes total)
        address lastUser;     // 20 bytes
        uint64 lastTime;      // Fits in remaining space
    }
    Stats public stats;
    
    function updateStats(address user) external {
        // Single storage slot update
        stats = Stats({
            count: stats.count + 1,
            lastUser: user,
            lastTime: uint64(block.timestamp)
        });
        // Total: ~20,000 gas (3x cheaper!)
    }
}

// ✅ ADVANCED: Bitpacking for extreme optimization  
contract BitpackedStorage {
    // Pack multiple values in single uint256
    uint256 private packedData;
    
    // Bits 0-127: count (128 bits)
    // Bits 128-159: user ID (32 bits) 
    // Bits 160-191: timestamp (32 bits)
    // Bits 192-255: flags (64 bits)
    
    function updatePacked(uint32 userId) external {
        uint256 newCount = (packedData & ((1 << 128) - 1)) + 1;
        uint256 newTime = uint32(block.timestamp);
        
        packedData = newCount |
                     (uint256(userId) << 128) |
                     (newTime << 160);
        // Gas saved: ~50,000 per update!
    }
    
    function getCount() external view returns (uint128) {
        return uint128(packedData & ((1 << 128) - 1));
    }
}
```

### 2. Loop Optimization (Principle: Reduce Iterations & Operations)

```solidity
// ❌ BAD: Expensive loops
contract BadLoops {
    uint256[] public items;
    mapping(uint256 => bool) public processed;
    
    function processItems() external {
        for (uint256 i = 0; i < items.length; i++) {  // storage read mỗi iteration
            if (!processed[items[i]]) {                // 2 storage reads
                processed[items[i]] = true;            // storage write
                // Process item...
            }
        }
    }
}

// ✅ GOOD: Optimized loops
contract GoodLoops {
    uint256[] public items;
    mapping(uint256 => bool) public processed;
    
    function processItems() external {
        uint256[] memory itemsMemory = items;  // Single storage read
        uint256 length = itemsMemory.length;   // Cache length
        
        for (uint256 i; i < length;) {         // No initialization, cached length
            uint256 item = itemsMemory[i];     // Memory read (cheap)
            if (!processed[item]) {
                processed[item] = true;
                // Process item...
            }
            unchecked { ++i; }                 // Unchecked increment
        }
    }
    
    // ✅ ADVANCED: Batch processing with gas limits
    function processItemsBatch(uint256 startIndex, uint256 batchSize) external {
        uint256[] memory itemsMemory = items;
        uint256 endIndex = startIndex + batchSize;
        if (endIndex > itemsMemory.length) {
            endIndex = itemsMemory.length;
        }
        
        for (uint256 i = startIndex; i < endIndex;) {
            uint256 item = itemsMemory[i];
            if (!processed[item]) {
                processed[item] = true;
                // Process item...
            }
            unchecked { ++i; }
        }
    }
}
```

### 3. Function Optimization (Principle: Minimize Call Overhead)

```solidity
// ❌ BAD: Inefficient functions
contract BadFunctions {
    mapping(address => uint256) public balances;
    
    function transfer(address to, uint256 amount) external {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        require(to != address(0), "Invalid address");
        require(amount > 0, "Invalid amount");
        
        balances[msg.sender] -= amount;  // Storage write
        balances[to] += amount;          // Storage write
    }
}

// ✅ GOOD: Optimized functions
contract GoodFunctions {
    mapping(address => uint256) public balances;
    
    function transfer(address to, uint256 amount) external {
        uint256 senderBalance = balances[msg.sender];  // Single storage read
        
        // Combine require statements (cheaper than multiple)
        require(
            senderBalance >= amount && 
            to != address(0) && 
            amount > 0, 
            "Transfer failed"
        );
        
        // Use unchecked for safe operations
        unchecked {
            balances[msg.sender] = senderBalance - amount;
            balances[to] += amount;
        }
    }
    
    // ✅ ADVANCED: Assembly optimization for critical paths
    function transferAssembly(address to, uint256 amount) external {
        assembly {
            // Get sender balance slot
            mstore(0x00, caller())
            mstore(0x20, balances.slot)
            let senderSlot := keccak256(0x00, 0x40)
            let senderBalance := sload(senderSlot)
            
            // Check conditions
            if or(
                or(lt(senderBalance, amount), iszero(to)),
                iszero(amount)
            ) {
                revert(0, 0)
            }
            
            // Update sender balance
            sstore(senderSlot, sub(senderBalance, amount))
            
            // Update receiver balance
            mstore(0x00, to)
            let receiverSlot := keccak256(0x00, 0x40)
            sstore(receiverSlot, add(sload(receiverSlot), amount))
        }
    }
}
```

### 4. Data Type Optimization (Principle: Use Appropriate Sizes)

```solidity
// ❌ BAD: Wasteful data types
contract BadDataTypes {
    uint256 public smallCounter;    // Waste: chỉ cần uint8
    bool public isActive;          // OK
    uint256 public timestamp;      // Waste: uint32 đủ cho timestamp
    address public owner;          // OK (20 bytes)
    uint256 public percentage;     // Waste: uint8 đủ cho % (0-100)
}

// ✅ GOOD: Packed data types
contract GoodDataTypes {
    // Pack into single storage slot (32 bytes)
    struct PackedData {
        uint8 smallCounter;    // 1 byte
        bool isActive;         // 1 byte  
        uint32 timestamp;      // 4 bytes
        address owner;         // 20 bytes
        uint8 percentage;      // 1 byte
        // Total: 27 bytes (fits in 32 byte slot)
    }
    PackedData public data;
    
    // ✅ ADVANCED: Custom types for clarity
    type Percentage is uint8;
    type Timestamp is uint32;
    type Counter is uint8;
    
    struct TypedData {
        Counter counter;
        bool isActive;
        Timestamp timestamp;
        address owner;
        Percentage percentage;
    }
    TypedData public typedData;
}
```

## 🔧 Advanced Gas Optimization Techniques

### 1. Assembly Optimization

```solidity
contract AssemblyOptimization {
    // ✅ Assembly for critical mathematical operations
    function efficientMath(uint256 a, uint256 b) external pure returns (uint256 result) {
        assembly {
            result := add(mul(a, b), div(a, b))
            // Faster than: result = (a * b) + (a / b);
        }
    }
    
    // ✅ Assembly for memory operations
    function copyBytes(bytes memory source, uint256 start, uint256 length) 
        external 
        pure 
        returns (bytes memory result) 
    {
        result = new bytes(length);
        assembly {
            let sourcePtr := add(add(source, 0x20), start)
            let resultPtr := add(result, 0x20)
            
            // Copy 32 bytes at a time
            for { let i := 0 } lt(i, length) { i := add(i, 0x20) } {
                mstore(add(resultPtr, i), mload(add(sourcePtr, i)))
            }
        }
    }
    
    // ✅ Assembly for hash operations  
    function efficientHash(uint256 a, uint256 b) external pure returns (bytes32) {
        bytes32 result;
        assembly {
            mstore(0x00, a)
            mstore(0x20, b)
            result := keccak256(0x00, 0x40)
        }
        return result;
    }
}
```

### 2. Memory vs Storage Optimization

```solidity
contract MemoryStorageOptimization {
    struct LargeStruct {
        uint256 field1;
        uint256 field2;
        uint256 field3;
        uint256 field4;
        uint256 field5;
    }
    
    mapping(uint256 => LargeStruct) public data;
    
    // ❌ BAD: Multiple storage reads
    function badProcessing(uint256 id) external view returns (uint256) {
        return data[id].field1 + 
               data[id].field2 + 
               data[id].field3 + 
               data[id].field4 + 
               data[id].field5;
        // 5 storage reads = expensive!
    }
    
    // ✅ GOOD: Single storage read to memory
    function goodProcessing(uint256 id) external view returns (uint256) {
        LargeStruct memory item = data[id];  // Single storage read
        return item.field1 + 
               item.field2 + 
               item.field3 + 
               item.field4 + 
               item.field5;
        // 1 storage read + memory operations = cheap!
    }
    
    // ✅ ADVANCED: Selective field reading
    function selectiveReading(uint256 id) external view returns (uint256) {
        // Only read needed fields
        uint256 f1;
        uint256 f2;
        assembly {
            mstore(0x00, id)
            mstore(0x20, data.slot)
            let baseSlot := keccak256(0x00, 0x40)
            
            f1 := sload(baseSlot)           // field1
            f2 := sload(add(baseSlot, 1))   // field2
        }
        return f1 + f2;
    }
}
```

### 3. Event Optimization

```solidity
contract EventOptimization {
    // ❌ BAD: Multiple events
    function badEventLogging(address user, uint256 amount, uint256 timestamp) external {
        emit UserRegistered(user);
        emit AmountSet(amount);
        emit TimestampRecorded(timestamp);
        // 3 events = 3x LOG operations
    }
    
    // ✅ GOOD: Single packed event
    event BatchUpdate(address indexed user, uint256 amount, uint256 timestamp);
    
    function goodEventLogging(address user, uint256 amount, uint256 timestamp) external {
        emit BatchUpdate(user, amount, timestamp);
        // 1 event = efficient logging
    }
    
    // ✅ ADVANCED: Bit-packed event data
    event PackedEvent(address indexed user, uint256 packedData);
    
    function packedEventLogging(
        address user, 
        uint128 amount, 
        uint64 timestamp,
        uint32 category,
        uint32 flags
    ) external {
        uint256 packed = uint256(amount) |
                        (uint256(timestamp) << 128) |
                        (uint256(category) << 192) |
                        (uint256(flags) << 224);
                        
        emit PackedEvent(user, packed);
    }
    
    event UserRegistered(address user);
    event AmountSet(uint256 amount);
    event TimestampRecorded(uint256 timestamp);
}
```

## 🧪 Gas Optimization Testing Framework

### 1. Gas Measurement Tools

```javascript
// Hardhat gas measurement setup
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("Gas Optimization Tests", function() {
    let optimizedContract, unoptimizedContract;
    
    beforeEach(async function() {
        const OptimizedContract = await ethers.getContractFactory("OptimizedContract");
        const UnoptimizedContract = await ethers.getContractFactory("UnoptimizedContract");
        
        optimizedContract = await OptimizedContract.deploy();
        unoptimizedContract = await UnoptimizedContract.deploy();
    });
    
    it("Should use less gas for storage operations", async function() {
        const optimizedTx = await optimizedContract.updateStorage(100, "0x...", 1000);
        const unoptimizedTx = await unoptimizedContract.updateStorage(100, "0x...", 1000);
        
        const optimizedReceipt = await optimizedTx.wait();
        const unoptimizedReceipt = await unoptimizedTx.wait();
        
        console.log("Optimized gas used:", optimizedReceipt.gasUsed.toString());
        console.log("Unoptimized gas used:", unoptimizedReceipt.gasUsed.toString());
        
        const gasReduction = unoptimizedReceipt.gasUsed.sub(optimizedReceipt.gasUsed);
        const reductionPercentage = gasReduction.mul(100).div(unoptimizedReceipt.gasUsed);
        
        console.log(`Gas reduction: ${gasReduction.toString()} (${reductionPercentage.toString()}%)`);
        
        expect(optimizedReceipt.gasUsed).to.be.lt(unoptimizedReceipt.gasUsed);
    });
    
    it("Should benchmark loop operations", async function() {
        const sizes = [10, 50, 100, 500];
        
        for (const size of sizes) {
            const optimizedTx = await optimizedContract.processArray(size);
            const unoptimizedTx = await unoptimizedContract.processArray(size);
            
            const optimizedReceipt = await optimizedTx.wait();
            const unoptimizedReceipt = await unoptimizedTx.wait();
            
            console.log(`Array size ${size}:`);
            console.log(`  Optimized: ${optimizedReceipt.gasUsed.toString()}`);
            console.log(`  Unoptimized: ${unoptimizedReceipt.gasUsed.toString()}`);
        }
    });
});
```

### 2. Gas Profiling Script

```javascript
// scripts/gas-profile.js
const { ethers } = require("hardhat");

async function profileGasUsage() {
    const [deployer] = await ethers.getSigners();
    
    // Deploy contracts
    const OptimizedContract = await ethers.getContractFactory("OptimizedContract");
    const contract = await OptimizedContract.deploy();
    
    console.log("=== Gas Profiling Report ===\n");
    
    // Test different function gas usage
    const functions = [
        { name: "simpleTransfer", args: [deployer.address, 100] },
        { name: "batchTransfer", args: [[deployer.address], [100]] },
        { name: "packedUpdate", args: [100, deployer.address, 1000] }
    ];
    
    for (const func of functions) {
        const gasEstimate = await contract.estimateGas[func.name](...func.args);
        console.log(`${func.name}: ${gasEstimate.toString()} gas`);
    }
    
    // Test gas usage at different scales
    console.log("\n=== Scale Testing ===");
    const scales = [1, 10, 50, 100];
    
    for (const scale of scales) {
        const addresses = Array(scale).fill(deployer.address);
        const amounts = Array(scale).fill(100);
        
        try {
            const gasEstimate = await contract.estimateGas.batchTransfer(addresses, amounts);
            console.log(`Batch size ${scale}: ${gasEstimate.toString()} gas`);
            console.log(`  Gas per item: ${gasEstimate.div(scale).toString()}`);
        } catch (error) {
            console.log(`Batch size ${scale}: Too expensive (gas limit exceeded)`);
        }
    }
}

profileGasUsage()
    .then(() => process.exit(0))
    .catch((error) => {
        console.error(error);
        process.exit(1);
    });
```

## 📊 Gas Optimization Patterns

### 1. State Variable Packing

```solidity
// ✅ Optimal packing strategy
contract OptimalPacking {
    // Slot 1: 32 bytes
    address owner;          // 20 bytes
    uint64 timestamp;       // 8 bytes  
    uint32 count;          // 4 bytes
    // Total: 32 bytes (perfect fit)
    
    // Slot 2: 32 bytes  
    uint128 balance1;      // 16 bytes
    uint128 balance2;      // 16 bytes
    // Total: 32 bytes (perfect fit)
    
    // Slot 3: 32 bytes
    bool isActive;         // 1 byte
    uint8 category;        // 1 byte
    uint16 flags;          // 2 bytes
    uint32 data1;          // 4 bytes
    uint32 data2;          // 4 bytes
    address lastUser;      // 20 bytes
    // Total: 32 bytes (perfect fit)
}

// ❌ Poor packing (wastes storage)
contract PoorPacking {
    uint256 count;         // Slot 1: 32 bytes
    address owner;         // Slot 2: 20 bytes (12 bytes wasted)
    uint256 timestamp;     // Slot 3: 32 bytes (could be uint64)
    bool isActive;         // Slot 4: 1 byte (31 bytes wasted)
    uint256 balance1;      // Slot 5: 32 bytes
    uint256 balance2;      // Slot 6: 32 bytes
    // Uses 6 slots instead of 3 = 2x more expensive!
}
```

### 2. Function Selector Optimization

```solidity
contract SelectorOptimization {
    // ✅ Optimized function names for frequent calls
    // Shorter selectors = slightly less gas
    
    function a() external {     // 0x0dbe671f (frequent function)
        // Implementation
    }
    
    function b() external {     // 0x4df7e3d0 (frequent function)  
        // Implementation
    }
    
    // Less frequent functions can have longer names
    function adminFunction() external {
        // Implementation
    }
    
    function emergencyStop() external {
        // Implementation
    }
}
```

### 3. Error Handling Optimization

```solidity
contract ErrorOptimization {
    // ✅ Custom errors (cheaper than require strings)
    error InsufficientBalance(uint256 available, uint256 required);
    error InvalidAddress();
    error Unauthorized(address caller);
    
    mapping(address => uint256) public balances;
    address public owner;
    
    function transfer(address to, uint256 amount) external {
        uint256 balance = balances[msg.sender];
        
        if (balance < amount) {
            revert InsufficientBalance(balance, amount);
        }
        
        if (to == address(0)) {
            revert InvalidAddress();
        }
        
        balances[msg.sender] = balance - amount;
        balances[to] += amount;
    }
    
    // ❌ Expensive string errors
    function expensiveTransfer(address to, uint256 amount) external {
        require(balances[msg.sender] >= amount, "Insufficient balance for transfer");  // Expensive!
        require(to != address(0), "Cannot transfer to zero address");                 // Expensive!
        
        balances[msg.sender] -= amount;
        balances[to] += amount;
    }
}
```

## 🎯 Gas Optimization Checklist

### ✅ Storage Optimization
- [ ] Pack struct variables into 32-byte slots
- [ ] Use appropriate uint sizes (uint8, uint16, uint32, uint64)
- [ ] Minimize storage reads/writes in loops
- [ ] Use `memory` for temporary data
- [ ] Consider bitpacking for flags and small values

### ✅ Loop Optimization  
- [ ] Cache array length before loops
- [ ] Use `unchecked` for safe arithmetic
- [ ] Prefer `++i` over `i++`
- [ ] Avoid storage operations in loops
- [ ] Consider batch processing for large datasets

### ✅ Function Optimization
- [ ] Use custom errors instead of require strings
- [ ] Combine multiple require statements when possible
- [ ] Use `view`/`pure` when functions don't modify state
- [ ] Consider assembly for critical math operations
- [ ] Pack function parameters efficiently

### ✅ Data Structure Optimization
- [ ] Use mappings instead of arrays when possible
- [ ] Consider packed structs for related data
- [ ] Use events instead of storage for non-critical data
- [ ] Implement lazy deletion patterns
- [ ] Use libraries for common operations

### ✅ Advanced Techniques
- [ ] Assembly optimization for hot paths
- [ ] Proxy patterns for upgradeable contracts
- [ ] Factory patterns for multiple similar contracts
- [ ] Bit manipulation for flags and permissions
- [ ] Memory-based data structures for complex operations

## 🔬 Real-World Examples

### DeFi Protocol Optimization

```solidity
// ✅ Optimized AMM-style swap function
contract OptimizedDEX {
    struct Pool {
        uint128 reserve0;   // Packed with reserve1
        uint128 reserve1;   // 32 byte slot
        uint32 blockTimestampLast;  // Packed with fee
        uint24 fee;         // 3 bytes for fee (0.01% = 1)
        bool isActive;      // 1 byte
        // Total: 32 bytes + 4 bytes = fits in 2 slots
    }
    
    mapping(bytes32 => Pool) public pools;
    
    function swap(
        address tokenA,
        address tokenB,
        uint256 amountIn,
        uint256 minAmountOut,
        address to
    ) external returns (uint256 amountOut) {
        bytes32 poolId = _getPoolId(tokenA, tokenB);
        Pool memory pool = pools[poolId];  // Single storage read
        
        if (!pool.isActive) revert PoolInactive();
        
        // Calculate output using xy=k formula
        uint256 amountInWithFee = amountIn * (10000 - pool.fee) / 10000;
        amountOut = (pool.reserve1 * amountInWithFee) / 
                    (pool.reserve0 + amountInWithFee);
        
        if (amountOut < minAmountOut) revert InsufficientOutput();
        
        // Update reserves (single storage write)
        pools[poolId] = Pool({
            reserve0: uint128(pool.reserve0 + amountIn),
            reserve1: uint128(pool.reserve1 - amountOut),
            blockTimestampLast: uint32(block.timestamp),
            fee: pool.fee,
            isActive: pool.isActive
        });
        
        // Transfer tokens
        _safeTransferFrom(tokenA, msg.sender, address(this), amountIn);
        _safeTransfer(tokenB, to, amountOut);
    }
}
```

---

**Status**: ⛽ Production-ready gas optimization guide | **Update**: July 2025  
**Key Techniques**: Storage packing, loop optimization, assembly usage, custom errors, memory management
