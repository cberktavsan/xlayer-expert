# X Layer Security Reference

## Solidity Security Rules

### CEI (Checks-Effects-Interactions) Pattern
All external calls MUST happen AFTER state changes:
```solidity
// ✅ Correct: CEI pattern
function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount, "Insufficient");  // Check
    balances[msg.sender] -= amount;                            // Effect
    (bool ok, ) = msg.sender.call{value: amount}("");          // Interaction
    require(ok, "Transfer failed");
}

// ❌ Wrong: Interaction before Effect — reentrancy vulnerability
function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount);
    (bool ok, ) = msg.sender.call{value: amount}("");
    balances[msg.sender] -= amount;
}
```

### ReentrancyGuard
Mandatory for bridge contracts and token transfer functions:
```solidity
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract Bridge is ReentrancyGuard {
    function bridgeToken(uint256 amount) external nonReentrant {
        // ...
    }
}
```

### Authentication
- NEVER use `tx.origin` for authentication — always use `msg.sender`
- `tx.origin` is only acceptable for checking "is the caller an EOA?"
```solidity
// ❌ Dangerous: Vulnerable to phishing attacks
require(tx.origin == owner);

// ✅ Correct
require(msg.sender == owner);
```

### Access Control
- Simple ownership: OpenZeppelin `Ownable2Step` (2-step transfer, prevents accidental loss)
- Role-based: OpenZeppelin `AccessControl` or `AccessControlEnumerable`
- Critical functions: `onlyOwner` + `timelock` combination

### Integer Overflow/Underflow
- Solidity 0.8+ has automatic checks
- Only use `unchecked` blocks for gas optimization when safety is guaranteed
```solidity
// Safe: i can never overflow (bounded loop)
for (uint256 i = 0; i < arr.length;) {
    // ...
    unchecked { ++i; }
}
```

### Front-running Protection
- Commit-reveal pattern: first commit hash, then reveal value
- Private mempool: via Flashblocks or dedicated RPC
- Time-based nonce or deadline parameters

### Slippage Protection
Mandatory parameters for DEX/swap operations:
```solidity
function swap(
    uint256 amountIn,
    uint256 minAmountOut,  // Slippage protection
    uint256 deadline       // Time protection
) external {
    require(block.timestamp <= deadline, "Expired");
    uint256 amountOut = _calculateSwap(amountIn);
    require(amountOut >= minAmountOut, "Slippage exceeded");
    // ...
}
```

### Flash Loan Attack Surfaces
- Oracle manipulation: use TWAP instead of spot price
- Price impact: large single-block trades manipulating price
- Protection: Chainlink oracle or multi-block TWAP, liquidity checks

### ERC20 Approve Race Condition
The `approve()` function is vulnerable to front-running — an attacker can spend the old allowance before the new one takes effect:
```solidity
// ❌ Vulnerable: attacker front-runs and spends old + new allowance
token.approve(spender, newAmount);

// ✅ Recommended: use SafeERC20 (handles the race condition internally)
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
using SafeERC20 for IERC20;

token.forceApprove(spender, amount);          // OZ v5+ (best)
token.safeIncreaseAllowance(spender, amount); // OZ v4

// Legacy fallback (if SafeERC20 is unavailable): reset to 0 first
token.approve(spender, 0);
token.approve(spender, newAmount);
```

### transfer() / send() 2300 Gas Limit
`transfer()` and `send()` forward only 2300 gas — not enough for contracts with logic in `receive()`:
```solidity
// ❌ Dangerous: 2300 gas limit — fails on contracts with receive() logic
payable(recipient).transfer(amount);
bool ok = payable(recipient).send(amount);

// ✅ Correct: call{value:} forwards all available gas
(bool success,) = payable(recipient).call{value: amount}("");
require(success, "Transfer failed");

// ✅ Better: OpenZeppelin Address
Address.sendValue(payable(recipient), amount);
```

### receive() / fallback() Protection
Unguarded receive functions permanently lock OKB in the contract:
```solidity
// ❌ Dangerous: OKB sent to this contract is locked forever
receive() external payable {}

// ✅ Option 1: emit event for tracking
receive() external payable {
    emit Received(msg.sender, msg.value);
}

// ✅ Option 2: reject unexpected OKB
receive() external payable {
    revert("Direct OKB transfers not accepted");
}
```

### Cross-Function Reentrancy
CEI pattern protects a single function, but an attacker can re-enter through a **different** function that reads stale state:
```solidity
// ❌ VULNERABLE: withdraw() follows CEI, but getBalance() reads stale state during callback
function withdraw(uint256 amount) external nonReentrant {
    require(balances[msg.sender] >= amount);
    (bool ok,) = msg.sender.call{value: amount}("");
    // During callback, attacker calls getBalance() which still shows old balance
    balances[msg.sender] -= amount;
}
```
Protection: use `nonReentrant` modifier on ALL public functions that read or write shared state, not just the one with the external call.

### ERC-4626 Vault Inflation Attack
First depositor can manipulate share price by donating tokens directly to the vault:
```solidity
// Attack: deposit 1 wei → donate 1M tokens → next depositor gets 0 shares
// Protection: use OpenZeppelin ERC4626 which includes virtual offset
// Or seed the vault with initial deposit during deployment
```

### Unchecked Return Values
Low-level calls that ignore return values silently fail, potentially losing funds:
```solidity
// ❌ Dangerous: return value ignored — transfer may fail silently
payable(recipient).send(amount);
address(target).call(data);

// ✅ Correct: check return value
(bool success, ) = payable(recipient).call{value: amount}("");
require(success, "Transfer failed");

// ✅ Better: use OpenZeppelin Address library
import "@openzeppelin/contracts/utils/Address.sol";
Address.sendValue(payable(recipient), amount);
Address.functionCall(target, data);
```

### Delegatecall Risks
Especially relevant for proxy/upgrade patterns on X Layer:
- `delegatecall` executes code in the **caller's storage context** — a malicious implementation can overwrite proxy storage
- Never `delegatecall` to untrusted or unverified contracts
- Proxy and implementation storage layouts MUST be identical — adding/reordering variables in upgrades breaks storage
- Always call `_disableInitializers()` in implementation constructors to prevent direct initialization
- Use OpenZeppelin's `StorageSlot` for unstructured storage patterns

### Denial of Service (DoS) via Unbounded Loops
Iterating over arrays that can grow without bound will eventually exceed the block gas limit:
```solidity
// ❌ Dangerous: recipients array can grow until loop exceeds gas limit
function distributeRewards() external {
    for (uint256 i = 0; i < recipients.length; i++) {
        payable(recipients[i]).transfer(rewards[i]);
    }
}

// ✅ Safe: pull pattern — each recipient claims their own reward
mapping(address => uint256) public pendingRewards;

function claimReward() external nonReentrant {
    uint256 reward = pendingRewards[msg.sender];
    require(reward > 0, "No reward");
    pendingRewards[msg.sender] = 0;                        // Effect
    (bool success,) = payable(msg.sender).call{value: reward}("");  // Interaction
    require(success, "Transfer failed");
}
```
Also: a single failed `transfer` in a loop reverts the entire batch — another reason to prefer pull patterns.

### Signature Replay Protection
Off-chain signatures (EIP-712, permit) must include replay protection fields:
```solidity
// EIP-712 domain separator MUST include all of these:
bytes32 public DOMAIN_SEPARATOR = keccak256(abi.encode(
    keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
    keccak256(bytes("MyContract")),
    keccak256(bytes("1")),
    block.chainid,     // 196 for X Layer mainnet — prevents cross-chain replay
    address(this)      // prevents cross-contract replay
));

// Per-user nonce — prevents same-chain replay
mapping(address => uint256) public nonces;

// Deadline — prevents stale signatures from being used indefinitely
function executeWithSignature(
    bytes calldata sig,
    uint256 deadline,
    uint256 nonce
) external {
    require(block.timestamp <= deadline, "Signature expired");
    require(nonce == nonces[msg.sender]++, "Invalid nonce");
    // ... verify signature against DOMAIN_SEPARATOR
}
```
Note: `block.chainid` returns 196 on X Layer mainnet, 1952 on testnet.

---

## L2-Specific Security

### Sequencer Centralization
- X Layer sequencer is controlled by OKX (multi-sequencer cluster with failover)
- Sequencer can censor transactions (delay/skip)
- Consider L1 forced inclusion mechanism for critical functions
- Transactions cannot be sent during sequencer downtime

### block.timestamp Manipulation
- Sequencer determines `block.timestamp` (with ±drift)
- Time-sensitive operations (auction, vesting, deadline) must account for this drift
- `block.number` is less reliable on L2 — use L1 block reference (`L1Block` predeploy)

### L2→L1 Withdrawal Security
- Withdrawal period: ~7 days challenge period (PermissionedDisputeGame active on OP Stack), but AggLayer ZK proofs can provide faster finality
- Proof generation time is variable
- Withdrawal replay protection: each withdrawal has a unique nonce
- Sufficient balance check in bridge contract is mandatory

### Bridge Reentrancy Risks
- ERC-777 tokens can create reentrancy via `tokensReceived` hook
- Fee-on-transfer tokens (deflationary) transfer less than expected amount
- Rebase tokens create balance inconsistencies in bridge
- Protection: token whitelist + ReentrancyGuard + actual balance delta check

### Message Replay Protection
- Cross-chain messages must validate `nonce + sourceChainId + destinationChainId`
- Prevent same message from executing on multiple chains
- L2CrossDomainMessenger provides these protections built-in

### OP Stack Specific Security
- **PermissionedDisputeGame:** Only authorized participants can open disputes — not yet permissionless
- **Multi-sequencer:** OKX operates with backup failover for 99.9%+ uptime
- **Bridge security:** OP Stack standard bridge contracts, extensively audited
- **Security audit:** January 2026 xlayer-reth diff audit report available

---

## [LEGACY] zkEVM Circuit Risks (Pre-Re-genesis Only)

> **IMPORTANT:** The following risks apply ONLY to the old Polygon CDK/zkEVM era (block ≤42,810,020). Post-re-genesis X Layer runs on standard EVM-compatible OP Stack.

### Opcode Circuit Bugs (Historical)
- SHL/SHR opcodes previously caused circuit bugs (PSE/Scroll audits)
- Formal verification found 6 soundness + 1 completeness issues

### Precompile Circuit Capacity (Historical)
- In old zkEVM, each precompile had fixed capacity in the circuit
- Post-re-genesis: standard EVM precompile behavior — this limitation no longer applies

---

## Private Key Management

### Mandatory Rules
1. `DEPLOYER_PRIVATE_KEY` only in `.env` file
2. `.env` file MUST be in `.gitignore` (verify!)
3. Hardcoded private key = critical security vulnerability
4. `process.env` check mandatory in deploy scripts:

```typescript
// deploy.ts start
if (!process.env.DEPLOYER_PRIVATE_KEY) {
    throw new Error("DEPLOYER_PRIVATE_KEY env variable required");
}
```

### Production Environment
- Use hardware wallet (Ledger) or multi-sig (Safe)
- Gnosis Safe: https://safe.global — multi-sig wallet
- Deployer key and admin key should be separate
- Add timelock contract for admin operations

---

## Additional Security Patterns

### Forced OKB Sending (selfdestruct bypass)
`selfdestruct` (or `CREATE2` + `selfdestruct` in the same transaction post-EIP-6780) can force OKB into any contract, bypassing `receive()` guards:
```solidity
// ❌ VULNERABLE: attacker uses selfdestruct to inflate address(this).balance
contract Vault {
    uint256 public totalDeposits;

    function deposit() external payable {
        totalDeposits += msg.value;
    }

    function isBalanceCorrect() public view returns (bool) {
        return address(this).balance == totalDeposits; // Can be broken!
    }
}

// ✅ Safe: track deposits with state variable, ignore forced sends
contract SafeVault {
    uint256 public totalDeposited;

    function deposit() external payable {
        totalDeposited += msg.value;
    }

    function availableBalance() public view returns (uint256) {
        return totalDeposited; // Not address(this).balance
    }
}
```

### Weak Randomness on L2
On L2, the sequencer controls `block.timestamp`, `block.prevrandao`, and `blockhash`. These MUST NOT be used for randomness:
```solidity
// ❌ VULNERABLE: sequencer can predict/manipulate all of these
uint256 bad1 = uint256(keccak256(abi.encodePacked(block.timestamp)));
uint256 bad2 = uint256(keccak256(abi.encodePacked(block.prevrandao)));
uint256 bad3 = uint256(keccak256(abi.encodePacked(blockhash(block.number - 1))));

// ✅ Safe: Chainlink VRF v2.5
import {VRFConsumerBaseV2Plus} from "@chainlink/contracts/src/v0.8/vrf/dev/VRFConsumerBaseV2Plus.sol";
import {VRFV2PlusClient} from "@chainlink/contracts/src/v0.8/vrf/dev/libraries/VRFV2PlusClient.sol";

contract RandomLottery is VRFConsumerBaseV2Plus {
    uint256 private s_subscriptionId;
    bytes32 private s_keyHash;
    mapping(uint256 => address) private s_requestToSender;

    constructor(address vrfCoordinator, uint256 subId, bytes32 keyHash)
        VRFConsumerBaseV2Plus(vrfCoordinator)
    {
        s_subscriptionId = subId;
        s_keyHash = keyHash;
    }

    function requestRandom() external returns (uint256 requestId) {
        requestId = s_vrfCoordinator.requestRandomWords(
            VRFV2PlusClient.RandomWordsRequest({
                keyHash: s_keyHash,
                subId: s_subscriptionId,
                requestConfirmations: 3,
                callbackGasLimit: 100_000,
                numWords: 1,
                extraArgs: VRFV2PlusClient._argsToBytes(
                    VRFV2PlusClient.ExtraArgsV1({nativePayment: false})
                )
            })
        );
        s_requestToSender[requestId] = msg.sender;
    }

    function fulfillRandomWords(uint256 requestId, uint256[] calldata randomWords) internal override {
        address winner = s_requestToSender[requestId];
        uint256 result = randomWords[0]; // Truly random
        // ... use result
    }
}
```
> **Note:** Check Chainlink's official documentation for VRF coordinator addresses on X Layer. If VRF is not yet available on X Layer, use commit-reveal pattern as fallback.

### On-Chain Data Privacy
`private` variables are NOT hidden — anyone can read them via `eth_getStorageAt`:
```solidity
// ❌ VULNERABLE: "private" does not mean secret
contract BadGame {
    uint256 private secretNumber = 42; // Readable by anyone!
    bytes32 private password = keccak256("hunter2"); // Readable by anyone!
}

// Reading private storage (off-chain):
// slot 0: await provider.getStorage(contractAddr, 0) → secretNumber
// slot 1: await provider.getStorage(contractAddr, 1) → password hash

// ✅ Safe: use commit-reveal for hidden values
contract SafeGame {
    mapping(address => bytes32) public commitments;

    function commit(bytes32 hash) external {
        commitments[msg.sender] = hash; // hash = keccak256(abi.encodePacked(value, salt))
    }

    function reveal(uint256 value, bytes32 salt) external {
        require(commitments[msg.sender] == keccak256(abi.encodePacked(value, salt)), "Bad reveal");
        delete commitments[msg.sender];
        // ... use value
    }
}
```

### Signature Malleability (ECDSA s-value)
ECDSA signatures accept both low-s and high-s values for the same message. This means an attacker can compute a second valid signature from any existing one:
```solidity
// ❌ VULNERABLE: accepts malleable signatures — attacker can forge second valid sig
function verify(bytes32 hash, uint8 v, bytes32 r, bytes32 s) public pure returns (address) {
    return ecrecover(hash, v, r, s);
}

// ✅ Safe: OpenZeppelin ECDSA rejects high-s values automatically
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import "@openzeppelin/contracts/utils/cryptography/MessageHashUtils.sol";

function verify(bytes32 hash, bytes calldata signature) public pure returns (address) {
    return ECDSA.recover(MessageHashUtils.toEthSignedMessageHash(hash), signature);
}

// Also: always track used signatures to prevent replay
mapping(bytes32 => bool) public usedSignatures;

function executeWithSig(bytes32 hash, bytes calldata sig) external {
    bytes32 sigHash = keccak256(sig);
    require(!usedSignatures[sigHash], "Signature already used");
    usedSignatures[sigHash] = true;

    address signer = ECDSA.recover(MessageHashUtils.toEthSignedMessageHash(hash), sig);
    require(signer == authorizedSigner, "Invalid signer");
    // ... execute action
}
```

### Oracle Integration (Chainlink Price Feed)
When using price oracles, always check for staleness and L2 sequencer uptime:
```solidity
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

contract PriceConsumer {
    AggregatorV3Interface internal priceFeed;
    uint256 public constant STALENESS_THRESHOLD = 3600; // 1 hour

    constructor(address feedAddress) {
        priceFeed = AggregatorV3Interface(feedAddress);
    }

    function getLatestPrice() public view returns (int256 price, uint8 decimals) {
        (
            uint80 roundId,
            int256 answer,
            /* uint256 startedAt */,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = priceFeed.latestRoundData();

        // Staleness check
        require(block.timestamp - updatedAt <= STALENESS_THRESHOLD, "Stale price data");
        // Round completeness check
        require(answeredInRound >= roundId, "Stale round");
        // Sanity check
        require(answer > 0, "Invalid price");

        return (answer, priceFeed.decimals());
    }
}

// For L2: also check sequencer uptime feed (if available on X Layer)
// AggregatorV3Interface sequencerUptimeFeed = AggregatorV3Interface(SEQUENCER_FEED_ADDR);
// (, int256 answer,, uint256 startedAt,) = sequencerUptimeFeed.latestRoundData();
// bool isSequencerUp = answer == 0;
// require(isSequencerUp, "Sequencer is down");
// uint256 timeSinceUp = block.timestamp - startedAt;
// require(timeSinceUp > GRACE_PERIOD, "Grace period not over");
```
> **Note:** Check Chainlink's official page for available price feeds on X Layer (chainId 196). If Chainlink feeds are not available for a specific pair, consider using TWAP from a DEX with sufficient liquidity.
