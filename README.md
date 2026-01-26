# ⚡ ZeICash ⚡

A peer-to-peer electronic cash blockchain written in Zig with longest chain consensus, RandomX mining.

## Overview

Zeicash is a blockchain implemented from scratch in Zig, a modern systems programming language with explicit error handling, no hidden control flow, and compile-time memory safety. The core implementation totals approximately 20,000 lines of code.

Key features include an account-based transaction model, concurrent blockchain analytics via RocksDB secondary instances, and a modular 14-message network protocol. The cryptographic stack comprises RandomX ASIC-resistant mining, Ed25519 signatures, BLAKE3 hashing, and ChaCha20-Poly1305 wallet encryption.

### Key Features

- **Longest Chain Consensus** - Cumulative proof-of-work with configurable peer verification
- **RandomX Mining** - ASIC-resistant with Light (256MB) and Fast (2GB) modes
- **HD Wallets** - BIP39/BIP32 hierarchical deterministic wallets with mnemonic recovery
- **Modern Cryptography** - ChaCha20-Poly1305 encryption, Argon2id key derivation, Ed25519 signatures
- **Analytics Platform** - TimescaleDB integration with REST API (optional)
- **P2P Networking** - Custom binary protocol with CRC32 integrity
- **High Performance** - ~15 tps, concurrent indexing, efficient sync protocols
- **Layer 2 Messaging** - Rich transaction metadata with PostgreSQL indexing (in development, optional)

## Quick Start

### Prerequisites

- **Zig** 0.14.1
- **RandomX** proof-of-work mining algorithm
- **RocksDB** libraries (`librocksdb-dev` on Ubuntu/Debian)
- **Memory**: 2GB+ RAM recommended. For 1GB nodes, **4GB of swap is required** 

### Optional (Not Required for Running a Node)
- **PostgreSQL** 12+ (only for analytics and L2 messaging features)

### Installation

```bash
# Clone the repository
git clone https://github.com/ZeICash/Core.git
cd zeicash

# Configure environment
cp .env.example .env
# Edit .env with your settings

# Build (debug mode)
zig build

# Build (optimized mode)
zig build -Doptimize=ReleaseFast
```

### Running a Sync Only Node

```bash
# Start server (no mining)
ZEIcash_SERVER=127.0.0.1 ./zig-out/bin/zen_server
```

### Running a Mining Node

```bash
# Create a miner wallet (interactive password prompt)
ZEIcash_SERVER=127.0.0.1 ./zig-out/bin/zeicash wallet create miner

# Start with mining enabled
ZEIcash_SERVER=127.0.0.1 ./zig-out/bin/zen_server --mine miner

# Connect to bootstrap nodes (automatic from .env)
# Default bootstrap: 209.38.31.77:10801, 134.199.170.129:10801
```

### Project Structure

```
zeicash/
├── src/
│   ├── core/              # Core blockchain components (relative imports)
│   │   ├── types/         # Data structures and constants
│   │   ├── crypto/        # Cryptography (Ed25519, Bech32, RandomX, BIP39)
│   │   ├── storage/       # RocksDB persistence and serialization
│   │   ├── network/       # P2P networking and protocol
│   │   ├── chain/         # Chain management and validation
│   │   ├── mempool/       # Transaction pool management
│   │   ├── miner/         # Mining subsystem
│   │   ├── sync/          # Synchronization protocols
│   │   ├── wallet/        # Wallet management
│   │   └── server/        # Server components
│   ├── apps/              # Applications (use zeicash module)
│   │   ├── main.zig            # Server entry point
│   │   ├── indexer.zig         # PostgreSQL blockchain indexer
│   │   └── transaction_api.zig # Transaction API service
│   └── lib.zig            # Public API (zeicash module)
├── sql/                   # Database schemas
├── randomx/               # RandomX C library
```

### Core Components

#### Consensus
- Cumulative proof-of-work
- Configurable peer block hash consensus (disabled/optional/enforced)
- Difficulty adjustment every 3 blocks
- cashbase maturity: 100 blocks

#### Mining
- **RandomX Algorithm**: ASIC-resistant proof-of-work
- **TestNet**: Light mode (256MB RAM), difficulty threshold: 0xF0000000
- **MainNet**: Fast mode (2GB RAM), difficulty threshold: 0x00008000

#### Network Protocol
- **Ports**: P2P (10801), Client API (10802), JSON-RPC (10803), REST API (8080)
- **Bootstrap Nodes**: 209.38.31.77:10801, 134.199.170.129:10801 (might add more in future)
- **Address Format**: Bech32 with BLAKE3 hashing (tzei1... for TestNet, zei1... for MainNet)
- **Message Types**: Handshake, Ping/Pong, Block, Transaction, GetBlocks, GetPeers, BlockHash
- **Integrity**: CRC32 checksums on all messages

#### Wallet Security
- **Encryption**: ChaCha20-Poly1305 AEAD (Authenticated Encryption)
- **Key Derivation**: Argon2id (64MB memory, 3 iterations)
- **HD Wallets**: BIP39 (12-word mnemonic) + BIP32 derivation
- **Signatures**: Ed25519 for transaction signing
- **Password Requirements**: Minimum 8 characters
- **Memory Protection**: Passwords cleared after use

## Layer 2 Messaging System (In Development)

> [!NOTE]
> Layer 2 is an optional feature currently in development. Running a Zeicash node does **not** require PostgreSQL or any L2 components. The core blockchain operates independently with just RocksDB.

Zeicash will feature an optional Layer 2 messaging layer that adds rich metadata to blockchain transactions.

### Features (Planned)
- **Transaction Enhancement**: Add messages, categories, and metadata to ZEI transactions
- **Auto-Linking**: Indexer automatically links L2 messages to confirmed blockchain transactions
- **REST API**: Complete API for L2 message management and querying

### Requirements
- **Core Node**: RocksDB only (no additional dependencies)
- **L2 Features**: PostgreSQL 12+ (optional, only if you want L2 messaging)
- **Analytics**: TimescaleDB (optional, only if you want analytics dashboards)

### Future L2 Workflow
1. Create enhancement with message/metadata (draft status)
2. Update to pending before sending transaction
3. Send actual ZEI transaction on blockchain
4. Indexer automatically confirms L2 message with tx_hash


### Concurrent Indexer

Zeicash features an optional concurrent indexer that runs simultaneously with the mining node without database conflicts:

```bash
# Start mining node
ZEIcash_SERVER=127.0.0.1 ./zig-out/bin/zen_server --mine miner &

# Run indexer (indexes new blocks and exits)
ZEIcash_DB_PASSWORD=testpass123 ./zig-out/bin/zeicash_indexer

# Or run continuously (automated monitoring)
while true; do
    ZEIcash_DB_PASSWORD=testpass123 ./zig-out/bin/zeicash_indexer
    sleep 30
done &
```

**Architecture**:
- Primary Database: RocksDB (mining node, exclusive write)
- Secondary Database: RocksDB secondary instance (indexer, concurrent read)
- Analytics Database: PostgreSQL/TimescaleDB (indexed data)
- Zero conflicts between mining and indexing

### TimescaleDB Analytics

High-performance analytics system with continuous aggregates:

- **Hypertables**: Time-partitioned tables (7-day chunks)
- **Continuous Aggregates**: Real-time materialized views
- **Compression**: 90%+ space savings on older data
- **Performance**: 1000x faster than raw blockchain queries

**REST API Endpoints**:
- `GET /health` - Service health check
- `GET /api/network/health` - Network metrics (24h)
- `GET /api/transactions/volume` - Transaction volume (30d)
- Port: 8080, CORS enabled

## Configuration

Key environment variables (see `.env.example` for all options):

```bash
# Network Configuration
ZEIcash_NETWORK=testnet                    # testnet or mainnet
ZEIcash_BOOTSTRAP=209.38.31.77:10801       # Bootstrap nodes
ZEIcash_SERVER=127.0.0.1                   # Server address

# Consensus Configuration
ZEIcash_CONSENSUS_MODE=optional             # disabled, optional, enforced
ZEIcash_CONSENSUS_THRESHOLD=0.5             # 50% peer agreement required
ZEIcash_CONSENSUS_MIN_PEERS=0               # Minimum peer responses

# Mining Configuration
ZEIcash_MINE_ENABLED=false                  # Enable mining
ZEIcash_MINER_WALLET=miner                  # Mining wallet name

# Database Configuration
ZEIcash_DB_PASSWORD=***                     # PostgreSQL password
ZEIcash_DATA_DIR=zeicash_data               # Data directory

# Wallet Security
ZEIcash_WALLET_PASSWORD=***                 # Optional: for automation only
```

## Network Information

### TestNet
- **Address Prefix**: `tzei1...`
- **Mining Mode**: Light (256MB RAM)
- **Difficulty**: 0xF0000000 (easy)
- **Bootstrap Nodes**: 209.38.31.77:10801, 134.199.170.129:10801
- **Database**: `zeicash_testnet`

### MainNet (Future)
- **Address Prefix**: `zei1...`
- **Mining Mode**: Fast (2GB RAM)
- **Difficulty**: 0x00008000 (hard)
- **Database**: `zeicash_mainnet`

### Network Limits
- Block size: 16MB hard limit, 2MB soft limit (mining)
- Transaction size: 100KB maximum
- Message field: 512 bytes maximum
- Mempool: 10,000 transactions, 50MB total size

## Development

### Build Commands

```bash
zig build                          # Debug build
zig build -Doptimize=ReleaseFast   # Optimized build
zig build test                     # Run tests
zig build check                    # Fast compilation check
zig build docs                     # Generate documentation
zig build clean                    # Clean artifacts
```

## Security Features

### Implemented Protections
- Difficulty validation (prevents difficulty spoofing attacks)
- Peer block hash consensus (prevents chain forks)
- Signature verification (Ed25519)
- Wallet encryption (ChaCha20-Poly1305 + Argon2id)
- cashbase maturity (100 blocks)
- Transaction size limits
- Mempool limits and validation

**Focus Areas**:

- Multi-node mining and sync testing
- Bug fixes and stability improvements
- Documentation improvements
- Performance optimization
- Website docs

## Contributing

We welcome contributions to Zeicash! There are several ways to help:


## License

MIT License - See [LICENSE](LICENSE) file for details.
