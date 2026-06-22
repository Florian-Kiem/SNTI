# Assentian-PQE (SNTI) Mining Guide

> **Status**: Testnet aktif. Mainnet belum diluncurkan.
> Mining sekarang = testnet SNTI (tidak bernilai ekonomis, tapi penting untuk stabilitas jaringan).

## Daftar Isi

1. [Persyaratan](#1-persyaratan)
2. [Platform yang Didukung](#2-platform-yang-didukung)
3. [Instalasi & Build](#3-instalasi--build)
4. [Menjalankan Node](#4-menjalankan-node)
5. [CPU Mining](#5-cpu-mining)
6. [Troubleshooting](#6-troubleshooting)

---

## 1. Persyaratan

| Komponen | Minimum | Rekomendasi |
|---|---|---|
| CPU | 2 core | 4+ core |
| RAM | 2 GB | 4+ GB |
| Storage | 10 GB | 20+ GB |
| OS | Ubuntu 20.04+ | Ubuntu 22.04+ |
| Internet | Stabil | Stabil |

---

## 2. Platform yang Didukung

| Platform | Status | Catatan |
|---|---|---|
| **Ubuntu/Debian (native)** | ✅ Verified | Direkomendasikan |
| **WSL2 (Windows)** | ✅ Verified | Ubuntu layer di Windows |
| Windows native (MSVC) | ⚠️ Belum diuji | Mungkin perlu penyesuaian |
| macOS | ⚠️ Belum diuji | Mungkin perlu penyesuaian |
| Arch/Fedora/CentOS | ⚠️ Belum diuji | Dependency berbeda |

**Rekomendasi**: gunakan Ubuntu 22.04 native atau WSL2 untuk hasil terbaik.

---

## 3. Instalasi & Build

### Ubuntu/Debian & WSL2

**Install dependencies:**
```bash
sudo apt update
sudo apt install -y build-essential libtool autotools-dev automake \
  pkg-config bsdmainutils python3 libevent-dev libboost-dev \
  libsqlite3-dev libssl-dev git autoconf
```

**Clone & build:**
```bash
git clone https://github.com/asepganzu-svg/AssentianPQE-SNTI.git
cd AssentianPQE-SNTI
./autogen.sh
./configure --without-gui --disable-tests --disable-bench
make -j$(nproc)
```

> ⚠️ Build memakan waktu 15-60 menit tergantung spek mesin.
> Kalau error `Killed` (kehabisan RAM), coba: `make -j1`

**Verifikasi build berhasil:**
```bash
./src/bitcoind --version
# Output: Bitcoin Core version v27.0.0
```

---

## 4. Menjalankan Node

### Connect ke Testnet Assentian-PQE

```bash
./src/bitcoind -testnet \
  -connect=104.234.26.7:39333 \
  -daemon
```

**Cek status node:**
```bash
./src/bitcoin-cli -testnet getblockchaininfo
```

Output yang diharapkan:
```json
{
  "chain": "test",
  "blocks": 10,
  "bestblockhash": "...",
  "verificationprogress": 1.0
}
```

**Tunggu sampai sync penuh** (`verificationprogress` = 1.0 dan `blocks` sama dengan node VPS).

### Buat Wallet

```bash
./src/bitcoin-cli -testnet createwallet "snti_miner"
ADDR=$(./src/bitcoin-cli -testnet -rpcwallet=snti_miner getnewaddress)
echo "Alamat mining kamu: $ADDR"
```

---

## 5. CPU Mining

### Method 1: Direct Mining (paling simpel)

```bash
# Mine 1 blok ke alamat kamu
./src/bitcoin-cli -testnet -rpcwallet=snti_miner generatetoaddress 1 "$ADDR"

# Mine terus-menerus (loop sederhana)
while true; do
  ./src/bitcoin-cli -testnet -rpcwallet=snti_miner generatetoaddress 1 "$ADDR"
  sleep 1
done
```

### Method 2: Mining Loop dengan Log

Simpan script ini sebagai `mine.sh`:
```bash
#!/bin/bash
ADDR=$1
if [ -z "$ADDR" ]; then
  echo "Usage: ./mine.sh <alamat_SNTI>"
  exit 1
fi

echo "Mining Assentian-PQE (SNTI) testnet ke $ADDR"
echo "Ctrl+C untuk berhenti"
echo ""

BLOCKS=0
START=$(date +%s)

while true; do
  RESULT=$(./src/bitcoin-cli -testnet -rpcwallet=snti_miner generatetoaddress 1 "$ADDR" 2>&1)
  if echo "$RESULT" | grep -q '"'; then
    BLOCKS=$((BLOCKS + 1))
    NOW=$(date +%s)
    ELAPSED=$((NOW - START))
    echo "[$(date '+%H:%M:%S')] Blok #$BLOCKS ditemukan! Total waktu: ${ELAPSED}s"
  fi
done
```

Jalankan:
```bash
chmod +x mine.sh
./mine.sh "$ADDR"
```

### Cek Hasil Mining

```bash
# Cek saldo (butuh 100 konfirmasi sebelum bisa dipakai)
./src/bitcoin-cli -testnet -rpcwallet=snti_miner getbalance
./src/bitcoin-cli -testnet -rpcwallet=snti_miner getwalletinfo
```

### Generate Alamat XMSS (Post-Quantum Address)

```bash
# Generate alamat XMSS baru
./src/bitcoin-cli -testnet -rpcwallet=snti_miner getnewxmssaddress

# Cek info alamat XMSS
./src/bitcoin-cli -testnet -rpcwallet=snti_miner getxmssaddressinfo "<alamat_xmss>"
```

> ⚠️ **PENTING**: Setiap alamat XMSS hanya boleh dipakai SEKALI untuk mengirim.
> Ini adalah fitur keamanan, bukan bug — mencegah reuse key yang melemahkan post-quantum security.

---

## 6. Troubleshooting

### Error: "Could not connect to server"
```bash
# Pastikan node sudah jalan
ps aux | grep bitcoind | grep -v grep

# Kalau tidak ada, jalankan ulang
./src/bitcoind -testnet -connect=104.234.26.7:39333 -daemon
sleep 5
./src/bitcoin-cli -testnet getblockcount
```

### Error: "Fee estimation failed"
```bash
# Tambah -fallbackfee
./src/bitcoind -testnet -connect=104.234.26.7:39333 -fallbackfee=0.00001 -daemon
```

### Error: "Killed" saat build
```bash
# Kurangi paralel job
make -j1
```

### Node tidak sync
```bash
# Cek koneksi ke VPS
./src/bitcoin-cli -testnet getpeerinfo | grep "addr\|synced_blocks"

# Kalau tidak ada peer, coba connect manual
./src/bitcoin-cli -testnet addnode "104.234.26.7:39333" "add"
```

---

## Info Jaringan Testnet

| Parameter | Nilai |
|---|---|
| VPS IP | 104.234.26.7 |
| P2P Port | 39333 |
| RPC Port | 39332 (localhost only) |
| Genesis Hash | `2d858f51fc4af7926bee59c82d06d58a3f260647145aaf6f89263bcb3643b66d` |
| Block Explorer | http://104.234.26.7 |
| Block Time | ~60 detik |
| Mining Reward | 50 SNTI/blok |

---

## Roadmap Mining

| Gelombang | Device | Status |
|---|---|---|
| 1 | CPU | 🟢 Aktif (testnet sekarang) |
| 2 | GPU | 🟡 Dalam pengembangan (stratum server) |
| 3 | ASIC | 🔵 Direncanakan (setelah mainnet stabil) |

---

*Assentian-PQE (SNTI) — Post Quantum Era Begins*
*Copyright © 2026 Asep Mulya | assentianpqe@gmail.com*

---

## 7. Stratum Mining (CPU Pool)

Selain `generatetoaddress`, kamu bisa connect ke stratum server resmi:
Host: 104.234.26.7
Port: 3333
Protocol: stratum+tcp
### Connect dengan cpuminer-opt

```bash
# Install cpuminer-opt
sudo apt install -y cpuminer-multi

# Connect ke stratum Assentian-PQE
minerd -a sha256d \
  -o stratum+tcp://104.234.26.7:3333 \
  -u YOUR_SNTI_ADDRESS \
  -p x
```

### Cek Stats Pool

```bash
curl http://104.234.26.7:3334/
```

### Catatan Wave 1 (CPU Stratum)

- Reward mining masuk ke **pool address** (bukan address kamu langsung)
- Sistem reward ke individual miner (payout) masih dalam pengembangan
- Wave 2 (GPU) akan implementasi `submitblock` proper dengan reward langsung ke miner
