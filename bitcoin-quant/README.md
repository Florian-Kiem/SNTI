# QNT — Quantum Resistant Blockchain

> **The world's first mineable post-quantum cryptocurrency.**

QNT is a Bitcoin Core v27 fork with XMSS-SHA2_10_256 post-quantum signature scheme and SHA-256 Proof-of-Useful-Work (PoUW) mining.

## Overview

| Feature | Specification |
|---------|---------------|
| **Signature Scheme** | XMSS-SHA2_10_256 (NIST SP 800-208) |
| **Proof of Work** | SHA-256 + XMSS block signing (PoUW v1) |
| **Block Time** | 60 seconds |
| **Max Supply** | 21,000,000 QNT |
| **Halving Interval** | 210,000 blocks (~2 years) |
| **P2P Port** | 9333 (mainnet), 19333 (testnet), 29333 (regtest) |
| **Address Prefix** | Q (mainnet), m/n (testnet/regtest) |
| **Base58 Prefix** | 0x51 (P2PKH), 0x56 (P2SH), 0x60 (XMSS) |

## Building from Source

### Dependencies

```bash
sudo apt-get install build-essential libtool autotools-dev automake pkg-config \
  libssl-dev libevent-dev bsdmainutils python3 libboost-all-dev \
  libdb-dev libdb++-dev
```

### Build

```bash
cd src
make -j$(nproc) -f makefile.unix
```

Or with autotools:

```bash
./autogen.sh
./configure --disable-bench --disable-gui --without-miniupnpc
make -j$(nproc)
```

### Binaries

After build, you'll find:
- `src/bitcoind` — QNT daemon
- `src/bitcoin-cli` — RPC client
- `src/bitcoin-wallet` — Wallet tool
- `src/bitcoin-tx` — Transaction tool

## Quick Start

### Regtest Mode

```bash
# Start regtest node
./bitcoind -regtest -daemon

# Generate genesis block
./bitcoin-cli -regtest createwallet "qnt_wallet"
ADDR=$(./bitcoin-cli -regtest getnewaddress)
./bitcoin-cli -regtest generatetoaddress 1 $ADDR

# Check balance
./bitcoin-cli -regtest getbalance
```

### XMSS Address

```bash
# Generate XMSS address
./bitcoin-cli -regtest getnewxmssaddress "my_key"

# List XMSS keys
./bitcoin-cli -regtest listxmsskeys

# Send to XMSS address
./bitcoin-cli -regtest sendtoxmssaddress "q..." 1.0
```

### Mining (PoUW)

```bash
# Mine blocks (automatic XMSS key generation + signing per block)
./bitcoin-cli -regtest generatetoaddress 10 $ADDR
```

## Architecture

```
src/
├── wallet/
│   ├── xmss_signer.h/cpp       # XMSS signing provider
│   ├── xmss_keystore.h/cpp     # Key persistence
│   ├── xmss_address.h/cpp      # Address encoding (Base58Check)
│   ├── xmss_state.h            # State management (anti-reuse)
│   └── rpc/xmss.h/cpp          # RPC commands
├── xmss_bridge.h/cpp           # C++ wrapper for XMSS library
├── script/
│   ├── interpreter.cpp         # OP_XMSS_CHECKSIG (0xBB)
│   ├── script.cpp              # Opcode definitions
│   └── signing.cpp             # SignTransactionXMSS
├── pow.cpp                     # PoUW: CheckPoUW() added
├── validation.cpp              # Block validation with PoUW
└── rpc/
    └── mining.cpp              # PoUW mining loop
```

## XMSS Parameters

| Parameter | Value |
|-----------|-------|
| Algorithm | XMSS-SHA2_10_256 |
| Tree Height | 10 (1024 signatures per key) |
| Security Level | 256-bit |
| Public Key | 64 bytes (root \|\| PUB_SEED) |
| Signature Size | ~2,500 bytes |
| OID | 0x00000001 |

## PoUW v1 Algorithm

1. Miner generates fresh XMSS key pair per block
2. Miner embeds XMSS pubkey in coinbase OP_RETURN
3. Standard SHA-256 nonce grinding
4. After PoW found: XMSS-sign the block hash
5. Insert signature into coinbase scriptSig
6. Re-verify PoW (merkle root changed by sig insertion)
7. If PoW invalidated: continue grinding

## Security

- [Security Audit Report](AUDIT.md) — Phase 4 self-audit completed
- No CRITICAL or HIGH severity issues found
- 2 MEDIUM issues documented (encryption at rest, sighash composition)
- All LOW issues fixed

## Development Status

| Phase | Status |
|-------|--------|
| Phase 1: XMSS Transaction Integration | ✅ Complete |
| Phase 2: Genesis Block Mine | ✅ Complete |
| Phase 3: Wallet XMSS Integration | ✅ Complete |
| Phase 3e: State Management | ✅ Complete |
| Phase 4: Security Audit | ✅ Complete |
| Phase 5: Documentation | 🔄 In Progress |
| Phase 6: Community & Visibility | ⬜ Pending |
| Phase 7: Mainnet Launch | ⬜ Pending |

## License

MIT License — see [COPYING](COPYING) for details.

## References

- [NIST SP 800-208](https://csrc.nist.gov/publications/detail/sp/800-208/final) — XMSS Signature Scheme
- [XMSS Reference Implementation](https://github.com/XMSS/xmss-reference)
- [Bitcoin Core](https://github.com/bitcoin/bitcoin) — Base codebase (v27)
