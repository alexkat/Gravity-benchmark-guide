# **README | Windows + WSL (Ubuntu) → Gravity Node + [gravity_bench](https://github.com/Galxe/gravity_bench)**

This document is a step-by-step guide for running and benchmarking using WSL (Ubuntu) on Windows. It covers two scenarios:

1. Local node (RPC: http://127.0.0.1:8545, typically chainId = 1337)

2. Connecting to Devnet (L1) (https://rpc-devnet.gravity.xyz, get the chainId via eth_chainId)

> Tip: Store your sources inside the Linux filesystem (e.g., ~/code) rather than /mnt/c — builds will be much faster.

## Table of Contents
- [0. Install WSL + Ubuntu](#0-install-wsl--ubuntu)
- [1. Base dependencies (Ubuntu in WSL)](#1-base-dependencies-ubuntu-in-wsl)
- [2. Install Rust](#2-install-rust)
- [3. (Optional) Local Gravity node](#3-optional-local-gravity-node)
- [4. Install & prepare gravity_bench](#4-install--prepare-gravity_bench)
- [5. bench_config.toml examples](#5-bench_configtoml-examples)
  - [5.A) Devnet (recommended to start)](#5a-devnet-recommended-to-start)
  - [5.B) Local node (WSL, chainId 1337)](#5b-local-node-wsl-chainid-1337)
- [6. Run the benchmark](#6-run-the-benchmark)
- [7. (Optional) Enable Uniswap swaps](#7-optional-enable-uniswap-swaps)
- [8. Useful checks](#8-useful-checks)
- [9. Common errors & quick fixes](#9-common-errors--quick-fixes)
- [10. Stop & exit](#10-stop--exit)

## 0. Install WSL + Ubuntu

Open **PowerShell** as Administrator:

```powershell
wsl --install -d Ubuntu-24.04 
```

Reboot, launch Ubuntu, create your user/password.

(Optional) Enable systemd:

```bash
echo -e "[boot]\nsystemd=true" | sudo tee /etc/wsl.conf
powershell.exe -Command "wsl --shutdown"
```

## 1. Base dependencies (Ubuntu in WSL)

```bash
sudo apt update
sudo apt install -y \
  build-essential clang pkg-config libssl-dev git curl unzip llvm cmake \
  protobuf-compiler jq dos2unix \
  libudev-dev libusb-1.0-0-dev \
  python3 python-is-python3 python3-pip python3-venv \
  nodejs npm
```
> This set prevents the common build/runtime errors: libudev.pc/pkg-config (for hidapi), missing python, Python modules (web3/solcx), and OpenZeppelin contracts via npm.

## 2. Install Rust

```bash
curl https://sh.rustup.rs -sSf | sh -s -- -y
source ~/.cargo/env
rustup default stable
rustc --version && cargo --version
```

## 3. (Optional) Local Gravity node
Build and start a single mock-consensus node:

```bash
mkdir -p ~/code && cd ~/code
git clone https://github.com/Galxe/gravity-sdk.git
cd gravity-sdk
git checkout reth-v1.4.8
RUSTFLAGS="--cfg tokio_unstable" make gravity_node
```
Create `start_dev_node.sh` in the gravity-sdk root:

```bash
cat > start_dev_node.sh <<'EOF'
#!/bin/bash
RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[0;33m'; NC='\033[0m'
log_info(){ echo -e "${GREEN}[INFO]${NC} $1"; }
log_warn(){ echo -e "${YELLOW}[WARN]${NC} $1"; }

NODE="node1"; INSTALL_DIR="/tmp"

export MOCK_CONSENSUS=true
export RETH_TXPOOL_BATCH_INSERT=1
export BATCH_INSERT_TIME=50
export USE_PARALLEL_STATE_ROOT=1
export USE_STORAGE_CACHE=1

log_info "Killing old gravity_node..."
pkill -9 gravity_node 2>/dev/null || log_warn "No running gravity_node found"

log_info "Deploying $NODE..."
bash ./deploy_utils/deploy.sh --mode single --install_dir "$INSTALL_DIR" --node "$NODE" -v release

log_info "Starting $NODE..."
bash "$INSTALL_DIR/$NODE/script/start.sh" --bin_name gravity_node

log_info "Node started"
EOF
chmod +x start_dev_node.sh
./start_dev_node.sh
```
Verify RPC:
```bash
curl -s localhost:8545 -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}'
# expect 0x539 (i.e., 1337)
```
## 4. Install & prepare `gravity_bench`
```bash
cd ~/code
git clone https://github.com/Galxe/gravity_bench.git
cd gravity_bench

# Python venv
python -m venv .venv
source .venv/bin/activate
pip install -U pip setuptools wheel
pip install web3 eth-account py-solc-x requests pyyaml

# If the project uses Uniswap as submodules:
git submodule update --init --recursive

# If not, put the repos manually:
mkdir -p contracts && cd contracts
[ -d v2-core ]       || git clone --depth=1 https://github.com/Uniswap/v2-core.git
[ -d v2-periphery ]  || git clone --depth=1 https://github.com/Uniswap/v2-periphery.git
[ -d uniswap-lib ]   || git clone --depth=1 https://github.com/Uniswap/uniswap-lib.git
cd ..

# OpenZeppelin Contracts for ERC20 (MyToken)
npm init -y >/dev/null 2>&1
npm install @openzeppelin/contracts@5
```
> `py-solc-x` auto-installs the needed `solc` versions during compilation.

## 5. `bench_config.toml` examples

Create `bench_config.toml` in the `gravity_bench` root. Choose one of the following.

### 5.A. Devnet (recommended to start)

Set `chain_id` to what the RPC returns (see Useful checks
 → `eth_chainId`). Below uses an example value.

 ```toml
 # bench_config.toml — DEVNET

contract_config_path = "deploy.json"
target_tps = 200                 # <= num_accounts
enable_swap_token = false        # start without Uniswap (simpler)

nodes = [
    { rpc_url = "https://rpc-devnet.gravity.xyz", chain_id = 7771625 },
]

[faucet]
# Pre-funded dev key that commonly appears in logs for dev setups
private_key = "0xPrefounded Wallet"

# IMPORTANT: disable cascading faucet to avoid absurd MAX_UINT-like sums
faucet_level = 0
topup_wei = "50000000000000000"       # 0.05 ETH
min_balance_wei = "20000000000000000" # 0.02 ETH
wait_duration_secs = 30

[accounts]
num_accounts = 200                    # must be >= target_tps

[performance]
num_senders = 200                     # must be <= num_accounts
max_pool_size = 20000
duration_secs = 60
 ```

## 5.B. Local node (WSL, chainId 1337)
 ```toml
# bench_config.toml — LOCAL

contract_config_path = "deploy.json"
target_tps = 200
enable_swap_token = false

nodes = [
    { rpc_url = "http://127.0.0.1:8545/", chain_id = 1337 },
]

[faucet]
private_key = "0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"
faucet_level = 0
topup_wei = "500000000000000000"        # 0.5 ETH
min_balance_wei = "200000000000000000"  # 0.2 ETH
wait_duration_secs = 10

[accounts]
num_accounts = 200

[performance]
num_senders = 200
max_pool_size = 20000
duration_secs = 60
```

> Rules:
- `accounts.num_accounts ≥ target_tps`
- `performance.num_senders ≤ accounts.num_accounts`

## 6. Run the benchmark
```bash
cd ~/code/gravity_bench
source .venv/bin/activate
cargo build --release
./target/release/gravity_bench --config bench_config.toml
# or
cargo run --release -- --config bench_config.toml
```

You should see:
- contract compilation & deployment,
- startup lines for Producer/Consumer,
- RPC and plan statistics tables.

## 7. (Optional) Enable Uniswap swaps

### 1. Verify the sources exist:
```bash
ls contracts/v2-core/contracts/UniswapV2Factory.sol
ls contracts/v2-periphery/contracts/UniswapV2Router02.sol
ls contracts/uniswap-lib/contracts/libraries/TransferHelper.sol
```
### 2. OZ is installed (`@openzeppelin/contracts@5`).
### 3. Enable swaps and set a sane `msg.value` to avoid `insufficient funds`:
```toml
enable_swap_token = true

[swap]
amount_in_wei = "1000000000000000"   # 0.001 ETH
```
### 4. Run again.
## 8. Useful checks
Get `chainId`:
```bash
curl -s https://rpc-devnet.gravity.xyz -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}'
```
Check an address balance:
```bash
curl -s https://rpc-devnet.gravity.xyz -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_getBalance","params":["0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266","latest"]}'
```
Example: manual ETH transfer (local):
```python
from web3 import Web3
w3 = Web3(Web3.HTTPProvider("http://127.0.0.1:8545"))
pk = "0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"
acct = w3.eth.account.from_key(pk)
to = Web3.to_checksum_address("0xRECEIVER")
tx = {
  "to": to, "value": w3.to_wei("0.1","ether"),
  "nonce": w3.eth.get_transaction_count(acct.address),
  "gas": 21000,
  "maxFeePerGas": w3.to_wei(1,"gwei"),
  "maxPriorityFeePerGas": w3.to_wei(1,"gwei"),
  "chainId": 1337,
}
signed = acct.sign_transaction(tx)
print("tx:", w3.eth.send_raw_transaction(signed.rawTransaction).hex())
```
> If you edit configs from Windows, run `dos2unix bench_config.toml` to remove `\r`.

## 9. Common errors & quick fixes
- `libudev.pc` / `hidapi` / `pkg-config`
```bash
sudo apt install -y libudev-dev libusb-1.0-0-dev pkg-config
```
- If needed:
`export PKG_CONFIG_PATH=/usr/lib/x86_64-linux-gnu/pkgconfig:/usr/share/pkgconfig`
`python: command not found`
```bash
sudo apt install -y python3 python-is-python3 python3-pip python3-venv
```
`ModuleNotFoundError: web3 / solcx`
Activate venv: `source .venv/bin/activate`
Install: `pip install web3 eth-account py-solc-x`

`SolcError` / missing Uniswap sources
`git submodule update --init --recursive`
or put `v2-core`, `v2-periphery`, `uniswap-lib` in `contracts/` manually.
OZ: `npm install @openzeppelin/contracts@5`

CRLF / “smart quotes” in `bench_config.toml`
`dos2unix bench_config.toml` and use plain ASCII quotes `"`

`insufficient funds for gas * price + value` with huge `value`
That’s the cascading faucet trying to send absurd amounts.
Fix: `faucet_level = 0` and set reasonable `topup_wei` / `min_balance_wei`.
For swaps, set `swap.amount_in_wei` (e.g., `0.001 ETH`).

assertion failed: `accounts.num_accounts` >= `target_tps`
Increase `num_accounts` or decrease `target_tps`.
Keep `num_senders` ≤ `num_accounts`.

Wrong network
Verify RPC URL and `eth_chainId`. For local reth-dev, `chainId = 1337`.

## 10. Stop & exit
- ### Local node:
```bash
pkill -9 gravity_node
# or
bash /tmp/node1/script/stop.sh 2>/dev/null || true
```
- Deactivate Python venv:
```bash
deactivate
```
---
## Notes
- Frequently used pre-funded dev key (for local/dev setups):
```
0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```