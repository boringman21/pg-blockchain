# 🏗️ Smart Contract Design Patterns: Production-Ready Architecture

## 🎯 First-Principles Thinking về Design Patterns

### Tại sao cần Design Patterns?

**Vấn đề cơ bản**: Smart contracts không thể sửa sau khi deploy

1. **Reusability**: Tái sử dụng code đã được kiểm chứng
2. **Security**: Patterns đã được test kỹ lưỡng
3. **Maintainability**: Code dễ hiểu và maintain
4. **Gas Efficiency**: Patterns đã được optimize

### Nguyên tắc SOLID trong Solidity

1. **Single Responsibility**: Mỗi contract chỉ làm 1 việc
2. **Open/Closed**: Mở để extend, đóng để modify
3. **Liskov Substitution**: Subclasses có thể thay thế base class
4. **Interface Segregation**: Interfaces nhỏ và specific
5. **Dependency Inversion**: Depend trên abstractions, không concrete

## 🔐 Access Control Patterns

### 1. Ownable Pattern

```solidity
// ✅ Basic ownership pattern
abstract contract Ownable {
    address private _owner;
    
    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
    
    constructor() {
        _transferOwnership(msg.sender);
    }
    
    modifier onlyOwner() {
        require(owner() == msg.sender, "Ownable: caller is not the owner");
        _;
    }
    
    function owner() public view returns (address) {
        return _owner;
    }
    
    function transferOwnership(address newOwner) public onlyOwner {
        require(newOwner != address(0), "Ownable: new owner is the zero address");
        _transferOwnership(newOwner);
    }
    
    function renounceOwnership() public onlyOwner {
        _transferOwnership(address(0));
    }
    
    function _transferOwnership(address newOwner) internal {
        address oldOwner = _owner;
        _owner = newOwner;
        emit OwnershipTransferred(oldOwner, newOwner);
    }
}
```

### 2. Role-Based Access Control (RBAC)

```solidity
// ✅ Flexible role-based system
import "@openzeppelin/contracts/access/AccessControl.sol";

contract RoleBasedContract is AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant BURNER_ROLE = keccak256("BURNER_ROLE");
    bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");
    bytes32 public constant UPGRADER_ROLE = keccak256("UPGRADER_ROLE");
    
    constructor() {
        // Grant admin role to contract deployer
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        
        // Set up role hierarchies
        _setRoleAdmin(MINTER_ROLE, DEFAULT_ADMIN_ROLE);
        _setRoleAdmin(BURNER_ROLE, DEFAULT_ADMIN_ROLE);
        _setRoleAdmin(PAUSER_ROLE, DEFAULT_ADMIN_ROLE);
    }
    
    // ✅ Multi-role requirements
    modifier onlyMinterOrAdmin() {
        require(
            hasRole(MINTER_ROLE, msg.sender) || hasRole(DEFAULT_ADMIN_ROLE, msg.sender),
            "AccessControl: account missing role"
        );
        _;
    }
    
    // ✅ Conditional role checking
    function sensitiveFunction() external {
        if (balanceOf(msg.sender) > 1000 ether) {
            // High balance users need special permission
            require(hasRole(MINTER_ROLE, msg.sender), "Insufficient privileges for large holder");
        } else {
            // Regular users only need basic access
            require(!paused, "Contract is paused");
        }
        
        // Function logic here
    }
}
```

### 3. Multi-Signature Pattern

```solidity
// ✅ Multi-sig wallet pattern
contract MultiSigWallet {
    struct Transaction {
        address to;
        uint256 value;
        bytes data;
        bool executed;
        uint256 confirmations;
        mapping(address => bool) isConfirmed;
    }
    
    address[] public owners;
    mapping(address => bool) public isOwner;
    uint256 public requiredConfirmations;
    
    Transaction[] public transactions;
    
    modifier onlyOwner() {
        require(isOwner[msg.sender], "Not an owner");
        _;
    }
    
    modifier notExecuted(uint256 transactionId) {
        require(!transactions[transactionId].executed, "Transaction already executed");
        _;
    }
    
    constructor(address[] memory _owners, uint256 _requiredConfirmations) {
        require(_owners.length > 0, "Owners required");
        require(_requiredConfirmations > 0 && _requiredConfirmations <= _owners.length, "Invalid confirmation count");
        
        for (uint256 i = 0; i < _owners.length; i++) {
            address owner = _owners[i];
            require(owner != address(0), "Invalid owner");
            require(!isOwner[owner], "Owner not unique");
            
            isOwner[owner] = true;
            owners.push(owner);
        }
        
        requiredConfirmations = _requiredConfirmations;
    }
    
    function submitTransaction(address to, uint256 value, bytes memory data) 
        public 
        onlyOwner 
        returns (uint256) 
    {
        uint256 transactionId = transactions.length;
        
        transactions.push();
        Transaction storage txn = transactions[transactionId];
        txn.to = to;
        txn.value = value;
        txn.data = data;
        txn.executed = false;
        txn.confirmations = 0;
        
        return transactionId;
    }
    
    function confirmTransaction(uint256 transactionId) 
        public 
        onlyOwner 
        notExecuted(transactionId) 
    {
        Transaction storage txn = transactions[transactionId];
        require(!txn.isConfirmed[msg.sender], "Transaction already confirmed");
        
        txn.isConfirmed[msg.sender] = true;
        txn.confirmations++;
        
        if (txn.confirmations >= requiredConfirmations) {
            executeTransaction(transactionId);
        }
    }
    
    function executeTransaction(uint256 transactionId) 
        public 
        notExecuted(transactionId) 
    {
        Transaction storage txn = transactions[transactionId];
        require(txn.confirmations >= requiredConfirmations, "Insufficient confirmations");
        
        txn.executed = true;
        
        (bool success, ) = txn.to.call{value: txn.value}(txn.data);
        require(success, "Transaction execution failed");
    }
}
```

## 🔄 Upgrade Patterns

### 1. Transparent Proxy Pattern

```solidity
// ✅ Transparent proxy implementation
contract TransparentUpgradeableProxy {
    bytes32 private constant _ADMIN_SLOT = 0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103;
    bytes32 private constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    
    constructor(address implementation, address admin, bytes memory data) payable {
        _setImplementation(implementation);
        _setAdmin(admin);
        
        if (data.length > 0) {
            (bool success,) = implementation.delegatecall(data);
            require(success, "Initialization failed");
        }
    }
    
    modifier ifAdmin() {
        if (msg.sender == _getAdmin()) {
            _;
        } else {
            _fallback();
        }
    }
    
    function admin() external ifAdmin returns (address) {
        return _getAdmin();
    }
    
    function implementation() external ifAdmin returns (address) {
        return _getImplementation();
    }
    
    function upgradeTo(address newImplementation) external ifAdmin {
        _setImplementation(newImplementation);
    }
    
    function upgradeToAndCall(address newImplementation, bytes calldata data) 
        external 
        payable 
        ifAdmin 
    {
        _setImplementation(newImplementation);
        (bool success,) = newImplementation.delegatecall(data);
        require(success, "Upgrade call failed");
    }
    
    function _fallback() internal {
        _delegate(_getImplementation());
    }
    
    fallback() external payable {
        _fallback();
    }
    
    receive() external payable {
        _fallback();
    }
}
```

### 2. Beacon Proxy Pattern

```solidity
// ✅ Beacon proxy for multiple contracts sharing same implementation
contract UpgradeableBeacon is Ownable {
    address private _implementation;
    
    event Upgraded(address indexed implementation);
    
    constructor(address implementation_) {
        _setImplementation(implementation_);
    }
    
    function implementation() public view returns (address) {
        return _implementation;
    }
    
    function upgradeTo(address newImplementation) public onlyOwner {
        _setImplementation(newImplementation);
        emit Upgraded(newImplementation);
    }
    
    function _setImplementation(address newImplementation) private {
        require(newImplementation.code.length > 0, "Beacon: implementation is not a contract");
        _implementation = newImplementation;
    }
}

contract BeaconProxy {
    address private immutable _beacon;
    
    constructor(address beacon, bytes memory data) payable {
        _beacon = beacon;
        
        if (data.length > 0) {
            (bool success,) = _implementation().delegatecall(data);
            require(success, "BeaconProxy: construction failed");
        }
    }
    
    function _beacon() internal view returns (address) {
        return _beacon;
    }
    
    function _implementation() internal view returns (address) {
        return IBeacon(_beacon()).implementation();
    }
    
    fallback() external payable {
        _delegate(_implementation());
    }
}
```

## 🏭 Factory Patterns

### 1. Simple Factory

```solidity
// ✅ Basic factory pattern
contract TokenFactory {
    event TokenCreated(address indexed token, address indexed creator);
    
    address[] public allTokens;
    mapping(address => address[]) public userTokens;
    
    function createToken(
        string memory name,
        string memory symbol,
        uint256 totalSupply
    ) external returns (address) {
        ERC20Token token = new ERC20Token(name, symbol, totalSupply, msg.sender);
        address tokenAddress = address(token);
        
        allTokens.push(tokenAddress);
        userTokens[msg.sender].push(tokenAddress);
        
        emit TokenCreated(tokenAddress, msg.sender);
        return tokenAddress;
    }
    
    function getAllTokensCount() external view returns (uint256) {
        return allTokens.length;
    }
    
    function getUserTokensCount(address user) external view returns (uint256) {
        return userTokens[user].length;
    }
}
```

### 2. Clone Factory (Minimal Proxy)

```solidity
// ✅ Gas-efficient factory using minimal proxy (EIP-1167)
import "@openzeppelin/contracts/proxy/Clones.sol";

contract CloneFactory {
    address public immutable tokenImplementation;
    
    event TokenCloned(address indexed clone, address indexed creator);
    
    constructor() {
        tokenImplementation = address(new ERC20Token());
    }
    
    function createToken(
        string memory name,
        string memory symbol,
        uint256 totalSupply
    ) external returns (address) {
        address clone = Clones.clone(tokenImplementation);
        
        ERC20Token(clone).initialize(name, symbol, totalSupply, msg.sender);
        
        emit TokenCloned(clone, msg.sender);
        return clone;
    }
    
    // ✅ Predict clone address before deployment
    function predictCloneAddress(uint256 salt) external view returns (address) {
        return Clones.predictDeterministicAddress(tokenImplementation, bytes32(salt));
    }
    
    function createDeterministicToken(
        string memory name,
        string memory symbol,
        uint256 totalSupply,
        uint256 salt
    ) external returns (address) {
        address clone = Clones.cloneDeterministic(tokenImplementation, bytes32(salt));
        
        ERC20Token(clone).initialize(name, symbol, totalSupply, msg.sender);
        
        emit TokenCloned(clone, msg.sender);
        return clone;
    }
}
```

### 3. Registry Pattern

```solidity
// ✅ Registry for tracking and discovering contracts
contract ContractRegistry {
    struct RegistryEntry {
        address contractAddress;
        string version;
        bytes32 category;
        uint256 timestamp;
        bool active;
    }
    
    mapping(bytes32 => RegistryEntry[]) public entries; // name => versions
    mapping(address => bytes32) public contractNames;   // address => name
    mapping(bytes32 => bytes32[]) public categories;    // category => names
    
    event ContractRegistered(bytes32 indexed name, address indexed contractAddress, string version);
    event ContractDeactivated(bytes32 indexed name, address indexed contractAddress);
    
    function register(
        string memory name,
        address contractAddress,
        string memory version,
        string memory category
    ) external {
        require(contractAddress.code.length > 0, "Not a contract");
        
        bytes32 nameHash = keccak256(abi.encodePacked(name));
        bytes32 categoryHash = keccak256(abi.encodePacked(category));
        
        entries[nameHash].push(RegistryEntry({
            contractAddress: contractAddress,
            version: version,
            category: categoryHash,
            timestamp: block.timestamp,
            active: true
        }));
        
        contractNames[contractAddress] = nameHash;
        categories[categoryHash].push(nameHash);
        
        emit ContractRegistered(nameHash, contractAddress, version);
    }
    
    function getLatest(string memory name) external view returns (RegistryEntry memory) {
        bytes32 nameHash = keccak256(abi.encodePacked(name));
        RegistryEntry[] memory versions = entries[nameHash];
        
        require(versions.length > 0, "Contract not found");
        
        // Return latest active version
        for (uint256 i = versions.length; i > 0; i--) {
            if (versions[i-1].active) {
                return versions[i-1];
            }
        }
        
        revert("No active version found");
    }
    
    function getByCategory(string memory category) external view returns (bytes32[] memory) {
        bytes32 categoryHash = keccak256(abi.encodePacked(category));
        return categories[categoryHash];
    }
}
```

## 🔒 Security Patterns

### 1. Pull Payment Pattern

```solidity
// ✅ Pull payment to avoid reentrancy
contract PullPayment {
    mapping(address => uint256) public payments;
    
    event PaymentWithdrawn(address indexed payee, uint256 amount);
    event PaymentDeposited(address indexed payee, uint256 amount);
    
    function withdraw() external {
        uint256 payment = payments[msg.sender];
        require(payment > 0, "No payment available");
        
        payments[msg.sender] = 0;
        
        (bool success, ) = msg.sender.call{value: payment}("");
        require(success, "Transfer failed");
        
        emit PaymentWithdrawn(msg.sender, payment);
    }
    
    function depositPayment(address payee) internal {
        payments[payee] += msg.value;
        emit PaymentDeposited(payee, msg.value);
    }
    
    // ✅ Batch withdrawal for gas efficiency
    function batchWithdraw(address[] calldata payees) external {
        uint256 totalPayment = 0;
        
        for (uint256 i = 0; i < payees.length; i++) {
            address payee = payees[i];
            uint256 payment = payments[payee];
            
            if (payment > 0) {
                payments[payee] = 0;
                totalPayment += payment;
                
                (bool success, ) = payee.call{value: payment}("");
                require(success, "Transfer failed");
                
                emit PaymentWithdrawn(payee, payment);
            }
        }
        
        require(totalPayment > 0, "No payments to withdraw");
    }
}
```

### 2. Circuit Breaker Pattern

```solidity
// ✅ Emergency stop mechanism
contract CircuitBreaker {
    bool public stopped = false;
    address public owner;
    
    // Different levels of circuit breaking
    mapping(bytes4 => bool) public functionPaused;
    
    modifier stopInEmergency() {
        require(!stopped, "Contract is stopped");
        _;
    }
    
    modifier onlyInEmergency() {
        require(stopped, "Contract is not stopped");
        _;
    }
    
    modifier functionNotPaused() {
        require(!functionPaused[msg.sig], "Function is paused");
        _;
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    function toggleContractActive() external onlyOwner {
        stopped = !stopped;
    }
    
    function pauseFunction(bytes4 functionSelector) external onlyOwner {
        functionPaused[functionSelector] = true;
    }
    
    function unpauseFunction(bytes4 functionSelector) external onlyOwner {
        functionPaused[functionSelector] = false;
    }
    
    // ✅ Automatic circuit breaker based on conditions
    uint256 public maxDailyWithdrawal = 1000 ether;
    uint256 public dailyWithdrawn;
    uint256 public lastResetDay;
    
    modifier dailyLimitCheck(uint256 amount) {
        if (block.timestamp / 1 days > lastResetDay) {
            dailyWithdrawn = 0;
            lastResetDay = block.timestamp / 1 days;
        }
        
        if (dailyWithdrawn + amount > maxDailyWithdrawal) {
            stopped = true; // Automatically pause contract
            revert("Daily limit exceeded, contract paused");
        }
        
        dailyWithdrawn += amount;
        _;
    }
}
```

### 3. Rate Limiting Pattern

```solidity
// ✅ Rate limiting for function calls
contract RateLimiter {
    struct RateLimit {
        uint256 maxCalls;
        uint256 windowSize;
        uint256 windowStart;
        uint256 callCount;
    }
    
    mapping(address => RateLimit) public userLimits;
    mapping(address => mapping(bytes4 => RateLimit)) public functionLimits;
    
    modifier rateLimit(uint256 maxCalls, uint256 windowSize) {
        RateLimit storage limit = userLimits[msg.sender];
        
        // Initialize or reset window
        if (block.timestamp >= limit.windowStart + limit.windowSize) {
            limit.windowStart = block.timestamp;
            limit.callCount = 0;
            limit.maxCalls = maxCalls;
            limit.windowSize = windowSize;
        }
        
        require(limit.callCount < limit.maxCalls, "Rate limit exceeded");
        limit.callCount++;
        _;
    }
    
    modifier functionRateLimit(uint256 maxCalls, uint256 windowSize) {
        RateLimit storage limit = functionLimits[msg.sender][msg.sig];
        
        if (block.timestamp >= limit.windowStart + limit.windowSize) {
            limit.windowStart = block.timestamp;
            limit.callCount = 0;
            limit.maxCalls = maxCalls;
            limit.windowSize = windowSize;
        }
        
        require(limit.callCount < limit.maxCalls, "Function rate limit exceeded");
        limit.callCount++;
        _;
    }
    
    // Example usage
    function sensitiveFunction() external rateLimit(5, 1 hours) {
        // Only 5 calls per hour per user
    }
    
    function withdraw(uint256 amount) external functionRateLimit(3, 1 days) {
        // Only 3 withdrawals per day per user
    }
}
```

## 📊 Data Patterns

### 1. Merkle Proof Pattern

```solidity
// ✅ Merkle tree for efficient verification
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";

contract MerkleDistributor {
    bytes32 public immutable merkleRoot;
    mapping(uint256 => uint256) private claimedBitMap;
    
    event Claimed(uint256 index, address account, uint256 amount);
    
    constructor(bytes32 merkleRoot_) {
        merkleRoot = merkleRoot_;
    }
    
    function isClaimed(uint256 index) public view returns (bool) {
        uint256 claimedWordIndex = index / 256;
        uint256 claimedBitIndex = index % 256;
        uint256 claimedWord = claimedBitMap[claimedWordIndex];
        uint256 mask = (1 << claimedBitIndex);
        return claimedWord & mask == mask;
    }
    
    function _setClaimed(uint256 index) private {
        uint256 claimedWordIndex = index / 256;
        uint256 claimedBitIndex = index % 256;
        claimedBitMap[claimedWordIndex] = claimedBitMap[claimedWordIndex] | (1 << claimedBitIndex);
    }
    
    function claim(
        uint256 index,
        address account,
        uint256 amount,
        bytes32[] calldata merkleProof
    ) external {
        require(!isClaimed(index), "Drop already claimed");
        
        bytes32 node = keccak256(abi.encodePacked(index, account, amount));
        require(MerkleProof.verify(merkleProof, merkleRoot, node), "Invalid proof");
        
        _setClaimed(index);
        
        // Transfer tokens or ETH
        require(IERC20(token).transfer(account, amount), "Transfer failed");
        
        emit Claimed(index, account, amount);
    }
}
```

### 2. Commit-Reveal Pattern

```solidity
// ✅ Two-phase process for hiding data
contract CommitReveal {
    struct Commitment {
        bytes32 commitment;
        uint256 timestamp;
        bool revealed;
        uint256 value;
    }
    
    mapping(address => Commitment) public commitments;
    uint256 public constant REVEAL_PERIOD = 1 hours;
    uint256 public constant COMMIT_PERIOD = 24 hours;
    
    uint256 public commitPhaseEnd;
    uint256 public revealPhaseEnd;
    
    constructor() {
        commitPhaseEnd = block.timestamp + COMMIT_PERIOD;
        revealPhaseEnd = commitPhaseEnd + REVEAL_PERIOD;
    }
    
    function commit(bytes32 _commitment) external {
        require(block.timestamp < commitPhaseEnd, "Commit phase ended");
        require(commitments[msg.sender].commitment == bytes32(0), "Already committed");
        
        commitments[msg.sender] = Commitment({
            commitment: _commitment,
            timestamp: block.timestamp,
            revealed: false,
            value: 0
        });
    }
    
    function reveal(uint256 _value, uint256 _nonce) external {
        require(block.timestamp >= commitPhaseEnd, "Commit phase not ended");
        require(block.timestamp < revealPhaseEnd, "Reveal phase ended");
        
        Commitment storage userCommit = commitments[msg.sender];
        require(userCommit.commitment != bytes32(0), "No commitment found");
        require(!userCommit.revealed, "Already revealed");
        
        bytes32 hash = keccak256(abi.encodePacked(_value, _nonce, msg.sender));
        require(hash == userCommit.commitment, "Invalid reveal");
        
        userCommit.revealed = true;
        userCommit.value = _value;
    }
    
    function generateCommitment(uint256 _value, uint256 _nonce) 
        external 
        view 
        returns (bytes32) 
    {
        return keccak256(abi.encodePacked(_value, _nonce, msg.sender));
    }
}
```

### 3. State Machine Pattern

```solidity
// ✅ State machine for complex workflows
contract Auction {
    enum AuctionState {
        Created,
        Started,
        Ended,
        Cancelled
    }
    
    AuctionState public state;
    
    modifier inState(AuctionState _state) {
        require(state == _state, "Invalid state");
        _;
    }
    
    modifier validTransition(AuctionState _from, AuctionState _to) {
        require(state == _from, "Invalid current state");
        require(isValidTransition(_from, _to), "Invalid transition");
        _;
    }
    
    function isValidTransition(AuctionState from, AuctionState to) 
        internal 
        pure 
        returns (bool) 
    {
        if (from == AuctionState.Created) {
            return to == AuctionState.Started || to == AuctionState.Cancelled;
        }
        if (from == AuctionState.Started) {
            return to == AuctionState.Ended || to == AuctionState.Cancelled;
        }
        return false; // Ended and Cancelled are terminal states
    }
    
    function startAuction() external inState(AuctionState.Created) {
        state = AuctionState.Started;
        // Start auction logic
    }
    
    function endAuction() external inState(AuctionState.Started) {
        state = AuctionState.Ended;
        // End auction logic
    }
    
    function cancelAuction() external {
        require(
            state == AuctionState.Created || state == AuctionState.Started,
            "Cannot cancel"
        );
        state = AuctionState.Cancelled;
        // Cancel auction logic
    }
}
```

## 🔄 Economic Patterns

### 1. Dutch Auction

```solidity
// ✅ Dutch auction implementation
contract DutchAuction {
    address public seller;
    uint256 public startingPrice;
    uint256 public reservePrice;
    uint256 public priceDecrement;
    uint256 public decrementInterval;
    uint256 public startTime;
    uint256 public endTime;
    
    bool public ended;
    address public winner;
    uint256 public winningPrice;
    
    event AuctionEnded(address winner, uint256 price);
    
    constructor(
        uint256 _startingPrice,
        uint256 _reservePrice,
        uint256 _priceDecrement,
        uint256 _decrementInterval,
        uint256 _duration
    ) {
        seller = msg.sender;
        startingPrice = _startingPrice;
        reservePrice = _reservePrice;
        priceDecrement = _priceDecrement;
        decrementInterval = _decrementInterval;
        startTime = block.timestamp;
        endTime = startTime + _duration;
    }
    
    function getCurrentPrice() public view returns (uint256) {
        if (block.timestamp >= endTime || ended) {
            return reservePrice;
        }
        
        uint256 elapsed = block.timestamp - startTime;
        uint256 decrements = elapsed / decrementInterval;
        uint256 totalDecrement = decrements * priceDecrement;
        
        if (totalDecrement >= startingPrice - reservePrice) {
            return reservePrice;
        }
        
        return startingPrice - totalDecrement;
    }
    
    function bid() external payable {
        require(!ended, "Auction ended");
        require(block.timestamp < endTime, "Auction expired");
        
        uint256 currentPrice = getCurrentPrice();
        require(msg.value >= currentPrice, "Bid too low");
        
        ended = true;
        winner = msg.sender;
        winningPrice = currentPrice;
        
        // Refund excess payment
        if (msg.value > currentPrice) {
            payable(msg.sender).transfer(msg.value - currentPrice);
        }
        
        // Pay seller
        payable(seller).transfer(currentPrice);
        
        emit AuctionEnded(winner, winningPrice);
    }
}
```

### 2. Bonding Curve

```solidity
// ✅ Bonding curve for automatic market making
contract BondingCurve {
    IERC20 public token;
    uint256 public totalSupply;
    uint256 public poolBalance;
    
    uint256 public constant CURVE_STEEPNESS = 1000000; // Adjusts curve steepness
    
    event TokensPurchased(address indexed buyer, uint256 ethAmount, uint256 tokensIssued);
    event TokensSold(address indexed seller, uint256 tokensAmount, uint256 ethReturned);
    
    constructor(address _token) {
        token = IERC20(_token);
    }
    
    // Price = poolBalance / (totalSupply * CURVE_STEEPNESS)
    function getCurrentPrice() public view returns (uint256) {
        if (totalSupply == 0) return 1e18; // Initial price 1 ETH
        return (poolBalance * 1e18) / (totalSupply * CURVE_STEEPNESS);
    }
    
    function calculatePurchaseReturn(uint256 ethAmount) public view returns (uint256) {
        if (totalSupply == 0) {
            return ethAmount * CURVE_STEEPNESS;
        }
        
        // Using bancor formula approximation
        uint256 newPoolBalance = poolBalance + ethAmount;
        uint256 newTotalSupply = (totalSupply * newPoolBalance) / poolBalance;
        
        return newTotalSupply - totalSupply;
    }
    
    function calculateSaleReturn(uint256 tokenAmount) public view returns (uint256) {
        require(tokenAmount <= totalSupply, "Not enough tokens in circulation");
        
        uint256 newTotalSupply = totalSupply - tokenAmount;
        uint256 newPoolBalance = (poolBalance * newTotalSupply) / totalSupply;
        
        return poolBalance - newPoolBalance;
    }
    
    function buy() external payable {
        require(msg.value > 0, "Must send ETH");
        
        uint256 tokensToIssue = calculatePurchaseReturn(msg.value);
        
        poolBalance += msg.value;
        totalSupply += tokensToIssue;
        
        require(token.transfer(msg.sender, tokensToIssue), "Token transfer failed");
        
        emit TokensPurchased(msg.sender, msg.value, tokensToIssue);
    }
    
    function sell(uint256 tokenAmount) external {
        require(tokenAmount > 0, "Amount must be positive");
        require(token.balanceOf(msg.sender) >= tokenAmount, "Insufficient balance");
        
        uint256 ethToReturn = calculateSaleReturn(tokenAmount);
        
        require(token.transferFrom(msg.sender, address(this), tokenAmount), "Transfer failed");
        
        totalSupply -= tokenAmount;
        poolBalance -= ethToReturn;
        
        payable(msg.sender).transfer(ethToReturn);
        
        emit TokensSold(msg.sender, tokenAmount, ethToReturn);
    }
}
```

## 📈 Integration Patterns

### 1. Oracle Integration

```solidity
// ✅ Secure oracle integration with fallbacks
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

contract PriceOracle {
    struct PriceFeed {
        AggregatorV3Interface feed;
        uint8 decimals;
        uint256 heartbeat; // Max time between updates
        bool active;
    }
    
    mapping(string => PriceFeed) public priceFeeds;
    mapping(string => uint256) public fallbackPrices;
    address public priceUpdater;
    
    modifier onlyPriceUpdater() {
        require(msg.sender == priceUpdater, "Not price updater");
        _;
    }
    
    function addPriceFeed(
        string memory asset,
        address feedAddress,
        uint256 heartbeat
    ) external onlyOwner {
        AggregatorV3Interface feed = AggregatorV3Interface(feedAddress);
        
        priceFeeds[asset] = PriceFeed({
            feed: feed,
            decimals: feed.decimals(),
            heartbeat: heartbeat,
            active: true
        });
    }
    
    function getPrice(string memory asset) external view returns (uint256, uint256) {
        PriceFeed memory priceFeed = priceFeeds[asset];
        require(priceFeed.active, "Price feed not active");
        
        try priceFeed.feed.latestRoundData() returns (
            uint80 roundId,
            int256 price,
            uint256 startedAt,
            uint256 updatedAt,
            uint80 answeredInRound
        ) {
            require(price > 0, "Invalid price");
            require(block.timestamp - updatedAt <= priceFeed.heartbeat, "Price stale");
            
            // Normalize to 18 decimals
            uint256 normalizedPrice = uint256(price) * (10 ** (18 - priceFeed.decimals));
            return (normalizedPrice, updatedAt);
            
        } catch {
            // Fallback to manual price
            uint256 fallbackPrice = fallbackPrices[asset];
            require(fallbackPrice > 0, "No fallback price available");
            return (fallbackPrice, block.timestamp);
        }
    }
    
    function setFallbackPrice(string memory asset, uint256 price) 
        external 
        onlyPriceUpdater 
    {
        fallbackPrices[asset] = price;
    }
}
```

### 2. Cross-Chain Pattern

```solidity
// ✅ Cross-chain message passing
interface ILayerZeroEndpoint {
    function send(
        uint16 _dstChainId,
        bytes calldata _destination,
        bytes calldata _payload,
        address payable _refundAddress,
        address _zroPaymentAddress,
        bytes calldata _adapterParams
    ) external payable;
}

contract CrossChainToken {
    ILayerZeroEndpoint public lzEndpoint;
    mapping(uint16 => bytes) public trustedRemoteLookup;
    
    event SendToChain(uint16 indexed _dstChainId, address indexed _from, uint256 _amount);
    event ReceiveFromChain(uint16 indexed _srcChainId, address indexed _to, uint256 _amount);
    
    function sendTokens(
        uint16 _dstChainId,
        address _to,
        uint256 _amount
    ) external payable {
        require(_amount > 0, "Amount must be > 0");
        require(balanceOf(msg.sender) >= _amount, "Insufficient balance");
        
        // Burn tokens on source chain
        _burn(msg.sender, _amount);
        
        // Encode payload
        bytes memory payload = abi.encode(_to, _amount);
        
        // Send cross-chain message
        lzEndpoint.send{value: msg.value}(
            _dstChainId,
            trustedRemoteLookup[_dstChainId],
            payload,
            payable(msg.sender),
            address(0),
            bytes("")
        );
        
        emit SendToChain(_dstChainId, msg.sender, _amount);
    }
    
    function lzReceive(
        uint16 _srcChainId,
        bytes memory _srcAddress,
        uint64,
        bytes memory _payload
    ) external {
        require(msg.sender == address(lzEndpoint), "Only endpoint");
        require(
            keccak256(_srcAddress) == keccak256(trustedRemoteLookup[_srcChainId]),
            "Invalid source"
        );
        
        (address to, uint256 amount) = abi.decode(_payload, (address, uint256));
        
        // Mint tokens on destination chain
        _mint(to, amount);
        
        emit ReceiveFromChain(_srcChainId, to, amount);
    }
}
```

## 🎯 Best Practices Summary

### 1. Design Pattern Selection

```typescript
interface PatternSelector {
  // When to use each pattern
  accessControl: {
    simple: "Use Ownable for single admin",
    complex: "Use RBAC for multiple roles",
    democratic: "Use multi-sig for shared control"
  },
  
  upgrades: {
    transparent: "When admin separation is important",
    uups: "When gas efficiency is priority", 
    beacon: "When managing multiple contracts",
    diamond: "When unlimited upgrades needed"
  },
  
  security: {
    pullPayment: "For external transfers",
    circuitBreaker: "For emergency stops",
    rateLimit: "For abuse prevention"
  }
}
```

### 2. Gas Optimization in Patterns

```solidity
// ✅ Optimized pattern implementations
contract OptimizedPatterns {
    // Pack multiple values in single storage slot
    struct PackedData {
        uint128 value1;
        uint128 value2;
        // Fits in one slot (256 bits)
    }
    
    // Use immutable for constants set in constructor
    address public immutable factory;
    uint256 public immutable deployTime;
    
    // Batch operations to reduce transaction costs
    function batchOperation(bytes[] calldata calls) external {
        for (uint256 i = 0; i < calls.length; i++) {
            (bool success, ) = address(this).delegatecall(calls[i]);
            require(success, "Batch call failed");
        }
    }
    
    // Use events for cheap data storage
    event DataStored(address indexed user, uint256 indexed id, bytes data);
    
    function storeData(uint256 id, bytes calldata data) external {
        emit DataStored(msg.sender, id, data);
    }
}
```

### 3. Security Checklist for Patterns

- [ ] Access controls properly implemented
- [ ] State changes follow checks-effects-interactions
- [ ] External calls are secure
- [ ] Integer overflows prevented
- [ ] Reentrancy protection in place
- [ ] Circuit breakers implemented where needed
- [ ] Rate limiting for sensitive functions
- [ ] Proper error handling
- [ ] Events emitted for important changes
- [ ] Gas limits considered for loops

---

**Status**: 🚀 Production-ready design patterns | **Update**: July 2025
