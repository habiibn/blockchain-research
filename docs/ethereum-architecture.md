# Ethereum-like Protocol: Architecture and Core Algorithms

## Table of Contents
1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Core Data Structures](#core-data-structures)
4. [Ethereum Virtual Machine (EVM)](#ethereum-virtual-machine-evm)
5. [Consensus: Ethash (Proof of Work)](#consensus-ethash-proof-of-work)
6. [State Management](#state-management)
7. [Transaction Processing](#transaction-processing)
8. [Gas and Fee Market](#gas-and-fee-market)
9. [Networking Protocol](#networking-protocol)
10. [Key Algorithms](#key-algorithms)

---

## Overview

Ethereum is a decentralized platform for running smart contracts - Turing-complete programs that execute in a trustless environment. Unlike Bitcoin's simple scripting, Ethereum provides a full virtual machine (EVM) enabling complex applications.

### Core Innovations

1. **Account-based Model**: Simpler than UTXO for smart contracts
2. **EVM**: Turing-complete virtual machine
3. **Gas Mechanism**: Resource metering and DoS prevention
4. **State Trie**: Efficient cryptographic state management
5. **Smart Contracts**: Programmable, self-executing agreements

---

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Application Layer                        │
│  (dApps, Wallets, Smart Contracts, Web3 Libraries)          │
└─────────────────────────────────────────────────────────────┘
                            ↕ (JSON-RPC)
┌─────────────────────────────────────────────────────────────┐
│                      RPC Interface                           │
│   (eth_sendTransaction, eth_call, eth_getBalance, ...)      │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│                    Transaction Pool                          │
│         (Mempool, Validation, Prioritization)               │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│                  Consensus & Block Production                │
│            (Ethash PoW, Block Validation)                   │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌───────────────┬─────────────────┬──────────────────────────┐
│   EVM Layer   │  State Manager  │   Storage Layer          │
│ (Execution)   │ (World State)   │ (Merkle Patricia Trie)   │
└───────────────┴─────────────────┴──────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│                    Network Layer (P2P)                       │
│     (Peer Discovery, Block Propagation, Sync Protocol)      │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│                    Database Layer                            │
│              (LevelDB/RocksDB for State & Blocks)           │
└─────────────────────────────────────────────────────────────┘
```

### Component Interactions

```
User Transaction
      ↓
   RPC API
      ↓
 Validation → Transaction Pool (Mempool)
      ↓
  Mining/Block Production
      ↓
 EVM Execution → State Transition
      ↓
 Block Validation
      ↓
  State Update → Merkle Patricia Trie
      ↓
 Block Propagation (P2P)
      ↓
  Blockchain Storage
```

---

## Core Data Structures

### Block Structure

```python
Block Header:
├── parentHash: bytes32        # Hash of parent block
├── ommersHash: bytes32        # Hash of uncle blocks
├── beneficiary: address       # Miner's address (block reward recipient)
├── stateRoot: bytes32         # Root of state trie
├── transactionsRoot: bytes32  # Root of transactions trie
├── receiptsRoot: bytes32      # Root of receipts trie
├── logsBloom: bytes256        # Bloom filter for logs
├── difficulty: uint256        # Block difficulty
├── number: uint256            # Block number (height)
├── gasLimit: uint256          # Maximum gas for block
├── gasUsed: uint256           # Total gas used by transactions
├── timestamp: uint256         # Unix timestamp
├── extraData: bytes           # Arbitrary data (≤32 bytes)
├── mixHash: bytes32           # PoW mix hash
└── nonce: bytes8              # PoW nonce

Block Body:
├── transactions: Transaction[]
└── ommers: BlockHeader[]      # Uncle blocks
```

### Transaction Structure

```python
Transaction:
├── nonce: uint256             # Sender's transaction count
├── gasPrice: uint256          # Wei per gas unit
├── gasLimit: uint256          # Maximum gas for transaction
├── to: address | null         # Recipient (null for contract creation)
├── value: uint256             # Wei to transfer
├── data: bytes                # Contract bytecode or call data
├── v: uint256                 # Signature component
├── r: uint256                 # Signature component
└── s: uint256                 # Signature component

Derived Fields (not in signed data):
├── from: address              # Recovered from signature
├── hash: bytes32              # Transaction hash
└── type: uint                 # Transaction type (0, 1, 2, ...)
```

### Account Structure

```python
Account:
├── nonce: uint256             # Transaction count (EOA) or creation count (Contract)
├── balance: uint256           # Wei balance
├── storageRoot: bytes32       # Root of storage trie (contracts only)
└── codeHash: bytes32          # Hash of contract code (contracts only)
```

**Two Account Types:**

1. **Externally Owned Accounts (EOA)**
   - Controlled by private key
   - Can initiate transactions
   - No code, empty storage

2. **Contract Accounts**
   - Controlled by code
   - Cannot initiate transactions (only respond)
   - Has code and storage

---

## Ethereum Virtual Machine (EVM)

### EVM Architecture

The EVM is a stack-based virtual machine that executes smart contract bytecode.

```
┌──────────────────────────────────────────┐
│          EVM Execution Context           │
├──────────────────────────────────────────┤
│  Program Counter (PC)                    │
│  Stack (max 1024 items, 256-bit words)   │
│  Memory (byte-addressable array)         │
│  Storage (persistent key-value store)    │
│  Gas Available                           │
├──────────────────────────────────────────┤
│         Environmental Information        │
│  - msg.sender, msg.value, msg.data       │
│  - block.number, block.timestamp, etc.   │
│  - tx.origin, tx.gasprice                │
└──────────────────────────────────────────┘
```

### EVM Components

1. **Stack**
   - Max depth: 1024 items
   - Each item: 256 bits (32 bytes)
   - Operations: PUSH, POP, DUP, SWAP

2. **Memory**
   - Volatile (resets after execution)
   - Byte-addressable
   - Expandable (costs gas)
   - Operations: MLOAD, MSTORE, MSTORE8

3. **Storage**
   - Persistent (survives across calls)
   - Key-value store (256-bit keys/values)
   - Expensive to write
   - Operations: SLOAD, SSTORE

4. **Calldata**
   - Read-only input data
   - Function parameters for contract calls
   - Operations: CALLDATALOAD, CALLDATASIZE, CALLDATACOPY

### Instruction Set (Opcodes)

**Categories:**

| Category | Examples | Description |
|----------|----------|-------------|
| Arithmetic | ADD, SUB, MUL, DIV, MOD, EXP | Basic math operations |
| Comparison | LT, GT, EQ, ISZERO | Comparison operations |
| Bitwise | AND, OR, XOR, NOT, BYTE, SHL, SHR | Bit manipulation |
| Cryptographic | SHA3 (KECCAK256) | Hashing |
| Environmental | ADDRESS, BALANCE, CALLER, ORIGIN | Context information |
| Block Info | BLOCKHASH, NUMBER, TIMESTAMP, DIFFICULTY | Block properties |
| Stack | PUSH, POP, DUP, SWAP | Stack manipulation |
| Memory | MLOAD, MSTORE, MSIZE | Memory operations |
| Storage | SLOAD, SSTORE | Persistent storage |
| Control Flow | JUMP, JUMPI, PC, JUMPDEST | Program flow |
| Calls | CALL, DELEGATECALL, STATICCALL, CREATE | Contract interaction |
| Return | RETURN, REVERT, SELFDESTRUCT | Execution termination |

### EVM Execution Algorithm

```python
def execute_transaction(tx, state):
    # 1. Validate transaction
    validate_signature(tx)
    validate_nonce(tx, state)
    validate_balance(tx, state)  # balance ≥ value + gas_limit * gas_price

    # 2. Deduct upfront gas cost
    state.deduct_balance(tx.sender, tx.gas_limit * tx.gas_price)
    gas_remaining = tx.gas_limit

    # 3. Increment sender nonce
    state.increment_nonce(tx.sender)

    # 4. Transfer value
    if tx.value > 0:
        state.transfer(tx.sender, tx.to, tx.value)

    # 5. Execute code (if contract call or creation)
    if tx.to is None:  # Contract creation
        gas_remaining, contract_address = create_contract(tx.data, gas_remaining, state)
    elif state.has_code(tx.to):  # Contract call
        gas_remaining = execute_code(tx.to, tx.data, gas_remaining, state)

    # 6. Refund unused gas
    gas_used = tx.gas_limit - gas_remaining
    refund = min(gas_remaining, gas_used // 2)  # Max 50% refund
    state.add_balance(tx.sender, (gas_remaining + refund) * tx.gas_price)

    # 7. Pay miner
    state.add_balance(block.beneficiary, gas_used * tx.gas_price)

    return gas_used, logs, status

def execute_code(address, calldata, gas, state):
    code = state.get_code(address)
    pc = 0  # Program counter
    stack = Stack()
    memory = Memory()

    while pc < len(code) and gas > 0:
        opcode = code[pc]

        # Deduct gas for operation
        gas_cost = get_gas_cost(opcode, stack, memory, state)
        if gas < gas_cost:
            raise OutOfGasError()
        gas -= gas_cost

        # Execute operation
        if opcode == PUSH1...PUSH32:
            stack.push(read_bytes(code, pc+1, opcode-PUSH1+1))
            pc += opcode - PUSH1 + 1
        elif opcode == ADD:
            a, b = stack.pop(), stack.pop()
            stack.push((a + b) % 2**256)
        elif opcode == SSTORE:
            key, value = stack.pop(), stack.pop()
            state.set_storage(address, key, value)
        # ... (other opcodes)
        elif opcode == RETURN:
            offset, length = stack.pop(), stack.pop()
            return gas, memory[offset:offset+length]
        elif opcode == REVERT:
            raise RevertError()

        pc += 1

    return gas
```

---

## Consensus: Ethash (Proof of Work)

### Ethash Overview

Ethash is a memory-hard PoW algorithm designed to be:
- **ASIC-resistant**: Requires substantial memory, favoring GPUs over ASICs
- **Light-client friendly**: Can verify with small dataset
- **Fair**: Difficult to gain advantage through hardware

### Algorithm Components

1. **Seed**: Determined by block number
2. **Cache**: Small dataset (~16 MB), generated from seed
3. **Dataset (DAG)**: Large dataset (~1+ GB), generated from cache
4. **Mining**: Find nonce such that hash meets difficulty target

### Ethash Mining Algorithm

```python
def ethash_mine(block_header, difficulty):
    # 1. Generate seed from block number
    epoch = block_header.number // EPOCH_LENGTH  # 30,000 blocks
    seed = generate_seed(epoch)

    # 2. Generate cache from seed
    cache = generate_cache(seed)

    # 3. Generate DAG from cache (or load if cached)
    dag = generate_dataset(cache)

    # 4. Try different nonces until valid hash found
    nonce = 0
    while True:
        # Combine header and nonce
        header_hash = keccak256(rlp_encode(block_header[:-2]))  # Exclude mixHash and nonce

        # Compute mix using DAG
        mix = compute_mix(header_hash, nonce, dag)

        # Compute final hash
        result = keccak256(header_hash + nonce + mix)

        # Check if meets difficulty
        if result <= difficulty_to_target(difficulty):
            return nonce, mix

        nonce += 1

def compute_mix(header_hash, nonce, dag):
    """
    Randomly access DAG to compute mix
    - Requires memory lookups, making it memory-hard
    """
    mix = keccak256(header_hash + nonce)

    # 64 rounds of random DAG accesses
    for i in range(64):
        index = mix % (len(dag) // 128)
        mix = keccak256(mix + dag[index:index+128])

    return mix
```

### Difficulty Adjustment Algorithm

```python
def calculate_difficulty(parent_block, current_timestamp):
    """
    Adjust difficulty to target ~15 second block time
    """
    parent_diff = parent_block.difficulty
    parent_time = parent_block.timestamp
    block_time = current_timestamp - parent_time

    # Calculate time-based adjustment
    if block_time < 10:
        # Too fast, increase difficulty
        time_adjustment = 1
    elif block_time < 20:
        # Slightly fast
        time_adjustment = 0
    else:
        # Too slow, decrease difficulty
        time_adjustment = -max(1, block_time // 10 - 1)

    # Apply adjustment (with bounds)
    difficulty = parent_diff + (parent_diff // 2048) * time_adjustment

    # Difficulty bomb (exponential increase over time)
    # Encourages transition to PoS
    period = (parent_block.number // 100000) - 2
    if period > 0:
        difficulty += 2 ** period

    return max(difficulty, MIN_DIFFICULTY)
```

---

## State Management

### World State: Merkle Patricia Trie

Ethereum uses a Modified Merkle Patricia Trie (MPT) to store state efficiently.

```
State Trie (World State)
├── Account 0x1234... → Account{nonce, balance, storageRoot, codeHash}
│                        └── Storage Trie
│                            ├── Key 0x00... → Value
│                            └── Key 0x01... → Value
├── Account 0x5678... → Account{...}
└── Account 0xABCD... → Account{...}
```

**Properties:**
- **Deterministic**: Same accounts always produce same root hash
- **Efficient**: O(log n) operations
- **Provable**: Can prove account state with Merkle proof
- **Updateable**: Modify individual accounts without rebuilding tree

### State Transition Function

```python
def apply_transaction(state, transaction):
    """
    State transition: Σ(σ, T) → σ'
    Where:
    - σ (sigma) = current state
    - T = transaction
    - σ' = new state
    """
    # Validate transaction
    assert validate_transaction(state, transaction)

    # Create checkpoint (for revert)
    checkpoint = state.snapshot()

    try:
        # Execute transaction
        receipt = execute_transaction(state, transaction)

        # Update state root
        new_state_root = state.compute_root()

        return state, receipt
    except Exception as e:
        # Revert state changes (but still consume gas)
        state.revert(checkpoint)
        return state, failed_receipt(e)

def apply_block(state, block):
    """
    Apply all transactions in block
    """
    for tx in block.transactions:
        state, receipt = apply_transaction(state, tx)
        receipts.append(receipt)

    # Validate state root matches
    assert state.compute_root() == block.header.stateRoot

    # Pay block reward
    state.add_balance(block.beneficiary, BLOCK_REWARD + total_fees)

    return state, receipts
```

---

## Transaction Processing

### Transaction Lifecycle

```
1. Creation
   ↓
2. Signing (ECDSA with private key)
   ↓
3. Broadcast to network
   ↓
4. Validation by nodes
   ↓
5. Added to mempool
   ↓
6. Selected by miner
   ↓
7. Executed in EVM
   ↓
8. Included in block
   ↓
9. Block mined and propagated
   ↓
10. Confirmations accumulate
```

### Transaction Validation

```python
def validate_transaction(state, tx):
    # 1. Signature validation
    assert recover_signer(tx) == tx.sender

    # 2. Nonce check
    assert tx.nonce == state.get_nonce(tx.sender)

    # 3. Gas limit check
    assert tx.gas_limit >= intrinsic_gas(tx)
    assert tx.gas_limit <= BLOCK_GAS_LIMIT

    # 4. Balance check
    max_cost = tx.value + (tx.gas_limit * tx.gas_price)
    assert state.get_balance(tx.sender) >= max_cost

    # 5. Gas price check (for mempool)
    assert tx.gas_price >= MIN_GAS_PRICE

    return True

def intrinsic_gas(tx):
    """
    Minimum gas required for transaction
    """
    gas = 21000  # Base transaction cost

    # Data cost
    for byte in tx.data:
        if byte == 0:
            gas += 4  # Zero bytes cheaper
        else:
            gas += 16  # Non-zero bytes

    # Contract creation cost
    if tx.to is None:
        gas += 32000

    return gas
```

---

## Gas and Fee Market

### Gas System

**Purpose:**
1. Prevent infinite loops
2. Allocate computational resources
3. Incentivize miners
4. Prevent spam

**Gas Costs (examples):**
```python
GAS_COSTS = {
    'STOP': 0,
    'ADD': 3,
    'MUL': 5,
    'SUB': 3,
    'DIV': 5,
    'SLOAD': 200,      # Load from storage
    'SSTORE': 20000,   # Write to storage (new)
    'SSTORE': 5000,    # Write to storage (existing)
    'BALANCE': 400,
    'CREATE': 32000,   # Contract creation
    'CALL': 700,       # Call another contract
}

# Memory expansion cost
def memory_cost(size_in_words):
    return size_in_words ** 2 // 512 + 3 * size_in_words
```

### EIP-1559 Fee Market

Modern Ethereum uses EIP-1559 dynamic fee mechanism:

```python
Transaction (EIP-1559):
├── maxPriorityFeePerGas: uint256  # Tip to miner
├── maxFeePerGas: uint256          # Max total fee per gas
└── ... (other fields)

Block:
├── baseFeePerGas: uint256         # Burned, not given to miner

Fee Calculation:
effective_gas_price = min(
    maxFeePerGas,
    baseFeePerGas + maxPriorityFeePerGas
)

burned_fee = gas_used * baseFeePerGas
miner_tip = gas_used * (effective_gas_price - baseFeePerGas)

# Base fee adjustment (targets 50% full blocks)
if parent_gas_used > TARGET_GAS:
    base_fee += base_fee * (parent_gas_used - TARGET_GAS) // TARGET_GAS // 8
else:
    base_fee -= base_fee * (TARGET_GAS - parent_gas_used) // TARGET_GAS // 8
```

---

## Networking Protocol

### DevP2P Protocol Stack

```
┌─────────────────────────────────────┐
│   Application Layer (eth, snap)     │
├─────────────────────────────────────┤
│   RLPx (encrypted, authenticated)   │
├─────────────────────────────────────┤
│   Node Discovery (Kademlia DHT)     │
├─────────────────────────────────────┤
│   TCP/UDP (IPv4/IPv6)               │
└─────────────────────────────────────┘
```

### Wire Protocol Messages

**Handshake:**
```python
Hello:
├── protocolVersion
├── clientId
├── capabilities
├── listenPort
└── nodeId
```

**Ethereum Sub-protocol (eth/67):**
```python
Messages:
├── Status (handshake)
├── NewBlockHashes (announce new blocks)
├── Transactions (broadcast transactions)
├── GetBlockHeaders / BlockHeaders
├── GetBlockBodies / BlockBodies
├── NewBlock (broadcast full block)
└── NewPooledTransactionHashes
```

### Synchronization Algorithms

**Full Sync:**
1. Download all blocks from genesis
2. Execute all transactions
3. Rebuild state from scratch
4. Slowest but most trustless

**Fast Sync:**
1. Download block headers
2. Download recent state snapshot
3. Download and verify receipts
4. Only execute recent blocks
5. Faster, still secure

**Snap Sync (Latest):**
1. Download block headers
2. Download state trie in parallel
3. Verify using state root
4. Fastest synchronization method

---

## Key Algorithms

### RLP (Recursive Length Prefix) Encoding

Used for serialization of arbitrary data structures.

```python
def rlp_encode(data):
    if isinstance(data, bytes):
        if len(data) == 1 and data[0] < 128:
            return data
        elif len(data) < 56:
            return bytes([128 + len(data)]) + data
        else:
            len_bytes = encode_length(len(data))
            return bytes([183 + len(len_bytes)]) + len_bytes + data
    elif isinstance(data, list):
        encoded = b''.join(rlp_encode(item) for item in data)
        if len(encoded) < 56:
            return bytes([192 + len(encoded)]) + encoded
        else:
            len_bytes = encode_length(len(encoded))
            return bytes([247 + len(len_bytes)]) + len_bytes + encoded
```

### Address Generation

```python
def generate_address(private_key):
    # 1. Generate public key from private key (ECDSA)
    public_key = ecdsa_public_key(private_key)  # 64 bytes (x, y coordinates)

    # 2. Hash public key
    hash = keccak256(public_key)  # 32 bytes

    # 3. Take last 20 bytes as address
    address = hash[-20:]  # 20 bytes (160 bits)

    return address

def contract_address(sender, nonce):
    """
    Contract address = rightmost 20 bytes of keccak256(RLP(sender, nonce))
    """
    rlp_encoded = rlp_encode([sender, nonce])
    hash = keccak256(rlp_encoded)
    return hash[-20:]
```

---

## Summary

Ethereum's architecture combines:

1. **Account Model**: Simpler state management than UTXO
2. **EVM**: Stack-based VM enabling Turing-complete contracts
3. **Gas Mechanism**: Economic resource allocation
4. **Merkle Patricia Trie**: Efficient cryptographic state proofs
5. **Ethash PoW**: Memory-hard consensus (transitioning to PoS)
6. **DevP2P**: Robust P2P networking protocol
7. **Smart Contracts**: Programmable, self-executing code

This implementation will recreate these core components to understand blockchain architecture deeply through hands-on development.

---

**Related Documents:**
- [Core Blockchain Concepts](./core-blockchain-concepts.md)
- [Comparative Analysis of Consensus Algorithms](./consensus-comparison.md)
