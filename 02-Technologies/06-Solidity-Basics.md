# 🔧 Solidity Basics

## 🎯 Mục tiêu bài học

Sau bài học này, bạn sẽ:
- Hiểu cú pháp cơ bản của Solidity
- Biết cách khai báo biến và function
- Nắm được các kiểu dữ liệu chính
- Viết được smart contract đầu tiên

## 📚 Giới thiệu về Solidity

**Solidity** là ngôn ngữ lập trình được thiết kế để viết smart contracts trên Ethereum. Nó có cú pháp tương tự JavaScript và C++.

### Đặc điểm chính:
- **Statically typed**: Kiểu dữ liệu xác định tại compile time
- **Contract-oriented**: Hướng đối tượng với contracts
- **Inheritance**: Hỗ trợ kế thừa
- **Libraries**: Có thể tái sử dụng code

## 🔧 Cú pháp cơ bản

### 1. Contract Declaration
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MyFirstContract {
    // Contract code goes here
}
```

### 2. Variables
```solidity
contract Variables {
    // State variables
    uint256 public myNumber = 42;
    string public myString = "Hello World";
    bool public isActive = true;
    address public owner;
    
    // Constructor
    constructor() {
        owner = msg.sender;
    }
}
```

### 3. Functions
```solidity
contract Functions {
    uint256 private value;
    
    // Function to set value
    function setValue(uint256 _value) public {
        value = _value;
    }
    
    // Function to get value
    function getValue() public view returns (uint256) {
        return value;
    }
    
    // Pure function (no state access)
    function add(uint256 a, uint256 b) public pure returns (uint256) {
        return a + b;
    }
}
```

## 📊 Kiểu dữ liệu

### 1. Value Types
```solidity
contract DataTypes {
    // Integers
    uint8 public smallNumber = 255;    // 0 to 255
    uint256 public bigNumber = 1000000; // 0 to 2^256-1
    int256 public signedNumber = -100;  // -2^255 to 2^255-1
    
    // Boolean
    bool public isReady = false;
    
    // Address
    address public userAddress = 0x1234567890123456789012345678901234567890;
    
    // Bytes
    bytes32 public data = "Hello";
    bytes public dynamicData = "Dynamic";
    
    // String
    string public name = "My Contract";
}
```

### 2. Reference Types
```solidity
contract ReferenceTypes {
    // Arrays
    uint256[] public numbers;
    string[] public names;
    
    // Mapping
    mapping(address => uint256) public balances;
    mapping(string => bool) public permissions;
    
    // Struct
    struct Person {
        string name;
        uint256 age;
        bool isActive;
    }
    
    Person[] public people;
}
```

## 🔐 Visibility Modifiers

```solidity
contract Visibility {
    uint256 private privateVar = 1;    // Only this contract
    uint256 internal internalVar = 2;  // This + inherited contracts
    uint256 public publicVar = 3;      // Anyone can access
    uint256 external externalVar = 4;  // Only external calls
    
    function privateFunction() private pure returns (uint256) {
        return 1;
    }
    
    function internalFunction() internal pure returns (uint256) {
        return 2;
    }
    
    function publicFunction() public pure returns (uint256) {
        return 3;
    }
    
    function externalFunction() external pure returns (uint256) {
        return 4;
    }
}
```

## 🔄 Function Modifiers

```solidity
contract Modifiers {
    address public owner;
    bool public paused = false;
    
    constructor() {
        owner = msg.sender;
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not the owner");
        _;
    }
    
    modifier whenNotPaused() {
        require(!paused, "Contract is paused");
        _;
    }
    
    function pause() public onlyOwner {
        paused = true;
    }
    
    function unpause() public onlyOwner {
        paused = false;
    }
    
    function sensitiveFunction() public onlyOwner whenNotPaused {
        // Function logic here
    }
}
```

## 🎪 Events

```solidity
contract Events {
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    mapping(address => uint256) public balances;
    
    function transfer(address to, uint256 amount) public {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        
        balances[msg.sender] -= amount;
        balances[to] += amount;
        
        emit Transfer(msg.sender, to, amount);
    }
}
```

## 🔒 Error Handling

```solidity
contract ErrorHandling {
    uint256 public value;
    
    function setValue(uint256 _value) public {
        // require - check conditions
        require(_value > 0, "Value must be positive");
        
        // assert - check invariants
        assert(_value < 1000000);
        
        value = _value;
    }
    
    function dangerousFunction() public {
        // revert - manual revert
        if (value == 0) {
            revert("Value not set");
        }
    }
}
```

## 🏗️ Contract Example: Simple Storage

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SimpleStorage {
    uint256 private storedValue;
    address public owner;
    
    event ValueChanged(uint256 newValue, address changedBy);
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call this");
        _;
    }
    
    constructor() {
        owner = msg.sender;
        storedValue = 0;
    }
    
    function set(uint256 _value) public onlyOwner {
        storedValue = _value;
        emit ValueChanged(_value, msg.sender);
    }
    
    function get() public view returns (uint256) {
        return storedValue;
    }
    
    function increment() public onlyOwner {
        storedValue += 1;
        emit ValueChanged(storedValue, msg.sender);
    }
}
```

## 🧪 Testing with Remix

### 1. Deploy Contract
1. Mở [Remix IDE](https://remix.ethereum.org/)
2. Tạo file mới `SimpleStorage.sol`
3. Copy code vào file
4. Compile với Solidity version 0.8.0+
5. Deploy lên JavaScript VM

### 2. Test Functions
```javascript
// Test scenarios
1. Deploy contract
2. Call get() - should return 0
3. Call set(42) - should set value to 42
4. Call get() - should return 42
5. Call increment() - should increase to 43
```

## 📚 Best Practices

### 1. Code Organization
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/access/Ownable.sol";

contract MyContract is Ownable {
    // State variables
    uint256 public constant MAX_SUPPLY = 1000;
    mapping(address => uint256) private _balances;
    
    // Events
    event Transfer(address indexed from, address indexed to, uint256 value);
    
    // Modifiers
    modifier validAddress(address _addr) {
        require(_addr != address(0), "Invalid address");
        _;
    }
    
    // Constructor
    constructor() Ownable(msg.sender) {}
    
    // External functions
    function transfer(address to, uint256 amount) 
        external 
        validAddress(to) 
        returns (bool) 
    {
        // Function implementation
    }
    
    // Public functions
    function balanceOf(address account) public view returns (uint256) {
        return _balances[account];
    }
    
    // Internal functions
    function _transfer(address from, address to, uint256 amount) internal {
        // Internal logic
    }
}
```

### 2. Security Practices
```solidity
contract Security {
    // Reentrancy guard
    bool private _locked = false;
    
    modifier noReentrant() {
        require(!_locked, "Reentrant call");
        _locked = true;
        _;
        _locked = false;
    }
    
    // Check-Effects-Interactions pattern
    function withdraw() public noReentrant {
        uint256 amount = balances[msg.sender];
        require(amount > 0, "No balance");
        
        // Effects
        balances[msg.sender] = 0;
        
        // Interactions
        payable(msg.sender).transfer(amount);
    }
}
```

## 📋 Common Patterns

### 1. Withdrawal Pattern
```solidity
contract Withdrawal {
    mapping(address => uint256) public pendingWithdrawals;
    
    function withdraw() public {
        uint256 amount = pendingWithdrawals[msg.sender];
        pendingWithdrawals[msg.sender] = 0;
        payable(msg.sender).transfer(amount);
    }
}
```

### 2. Circuit Breaker
```solidity
contract CircuitBreaker {
    bool public stopped = false;
    address public owner;
    
    modifier stopInEmergency() {
        require(!stopped, "Contract is stopped");
        _;
    }
    
    modifier onlyInEmergency() {
        require(stopped, "Contract is not stopped");
        _;
    }
    
    function emergencyStop() public onlyOwner {
        stopped = true;
    }
}
```

## 🎯 Bài tập thực hành

### Bài tập 1: Counter Contract
Viết contract có thể:
- Lưu trữ một số nguyên
- Tăng giá trị lên 1
- Giảm giá trị xuống 1
- Reset về 0
- Chỉ owner mới có thể thao tác

### Bài tập 2: Simple Token
Tạo token đơn giản với:
- Tổng supply cố định
- Transfer giữa các address
- Kiểm tra balance
- Approval mechanism

### Bài tập 3: Voting Contract
Tạo hệ thống voting:
- Tạo proposal
- Vote cho proposal
- Kết thúc voting
- Xem kết quả

## 📚 Tài liệu tham khảo

- [Solidity Documentation](https://docs.soliditylang.org/)
- [OpenZeppelin Contracts](https://openzeppelin.com/contracts/)
- [Solidity by Example](https://solidity-by-example.org/)
- [CryptoZombies](https://cryptozombies.io/)

## ✅ Checklist hoàn thành

- [ ] Hiểu cú pháp Solidity cơ bản
- [ ] Biết các kiểu dữ liệu chính
- [ ] Viết được function và modifier
- [ ] Sử dụng được event và error handling
- [ ] Deploy contract trên Remix
- [ ] Hoàn thành 3 bài tập thực hành

---

**Tiếp theo**: [[07-Contract-Structure]]
**Quay lại**: [[00-Index]]

**Thời gian học**: 4-5 giờ
**Độ khó**: ⭐⭐⭐☆☆ 