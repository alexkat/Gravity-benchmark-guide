
# README | Windows + WSL (Ubuntu) → Gravity Node + gravity\_bench

This document is a step-by-step guide for running and benchmarking `gravity_bench` using WSL (Ubuntu) on Windows. It covers two scenarios:

  * **Local node** (RPC: `http://127.0.0.1:8545`, typically `chainId = 1337`)
  * **Connecting to Devnet (L1)** (`https://rpc-devnet.gravity.xyz`, get the `chainId` via `eth_chainId`)

**Tip:** Store your source code inside the Linux filesystem (e.g., `~/code`) rather than `/mnt/c` — builds will be much faster.

**NOTE (Native Ubuntu/Linux):** If you already run Ubuntu (or another Linux distro) natively, **skip section “\#\#0 Install WSL + Ubuntu”** and execute all commands in your normal shell. Package names/commands are the same (tested on Ubuntu 22.04/24.04).

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
## 0\. Install WSL + Ubuntu

Open **PowerShell** as Administrator:

```powershell
wsl --install -d Ubuntu-24.04
```

Reboot, launch Ubuntu, and create your user/password.

(Optional) Enable systemd for broader service compatibility:

```bash
echo -e "[boot]\nsystemd=true" | sudo tee /etc/wsl.conf
powershell.exe -Command "wsl --shutdown"
```

Then restart your Ubuntu terminal.

## 1\. Base dependencies (Ubuntu in WSL)

```bash
sudo apt update
sudo apt install -y \
  build-essential clang pkg-config libssl-dev git curl unzip llvm cmake \
  protobuf-compiler jq dos2unix \
  libudev-dev libusb-1.0-0-dev \
  python3 python-is-python3 python3-pip python3-venv \
  nodejs npm
```

This set prevents common build/runtime errors related to `libudev.pc`/`pkg-config` (for `hidapi`), missing python, and Node.js/NPM for contract dependencies.

## 2\. Install Rust

```bash
curl https://sh.rustup.rs -sSf | sh -s -- -y
source ~/.cargo/env
rustup default stable
rustc --version && cargo --version
```

## 3\. (Optional) Local Gravity node

Build and start a single mock-consensus node:

```bash
mkdir -p ~/code && cd ~/code
git clone https://github.com/Galxe/gravity-sdk.git
cd gravity-sdk
git checkout reth-v1.4.8
make gravity_node
```

Create `start_dev_node.sh` in the `gravity-sdk` root:

```bash
cat > start_dev_node.sh <<'EOF'
#!/bin/bash
RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[0;33m'; NC='\033[0m'
log_info(){ echo -e "${GREEN}[INFO]${NC} $1"; }
log_warn(){ echo -e "${YELLOW}[WARN]${NC} $1"; }

NODE="node1"
INSTALL_DIR="/tmp"

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
```

Make it executable and run it:

```bash
chmod +x start_dev_node.sh
./start_dev_node.sh
```

Verify RPC is running:

```bash
curl -s localhost:8545 -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}'
# expect 0x539 (i.e., 1337)
```

## 4\. Install & prepare `gravity_bench`

The setup script automates the creation of a Python virtual environment, installation of dependencies (`pip`, `npm`), and cloning of required repositories (`Uniswap`).

```bash
cd ~/code
git clone https://github.com/Galxe/gravity_bench.git
cd gravity_bench
source setup.sh
```

## 5\. `bench_config.toml` examples

Create `bench_config.toml` in the `gravity_bench` root. Choose one of the following templates.

### 5.A) Devnet (recommended to start)

Set `chain_id` to the value returned by the RPC endpoint (see [Useful checks](https://www.google.com/search?q=%238-useful-checks)).

```toml
# bench_config.toml — DEVNET

# Path for contract deployment artifacts
contract_config_path = "deploy.json"

# Target transactions per second
target_tps = 200

# Set to true to include Uniswap swap transactions
enable_swap_token = false

# Number of ERC20 tokens to deploy
num_tokens = 2

nodes = [
   { rpc_url = "https://rpc-devnet.gravity.xyz", chain_id = 7771625 }, # Example chainId, verify first!
]

# Faucet account for funding test wallets
[faucet]
# Pre-funded dev key that commonly appears in logs for dev setups
private_key = "0xPrefounded Wallet"
# Cascade level for funding accounts. 0 disables cascading.
faucet_level = 0
# Wait time between funding rounds. Increase if "insufficient funds" errors occur.
wait_duration_secs = 30

# Configuration for generated test accounts
[accounts]
num_accounts = 200

# Performance tuning
[performance]
# Number of concurrent senders. Must be <= num_accounts.
num_senders = 200
# Max pending transactions in the memory pool
max_pool_size = 20000
# Benchmark duration in seconds. 0 runs indefinitely.
duration_secs = 60
```

### 5.B) Local node (WSL, chainId 1337)

```toml
# bench_config.toml — LOCAL

# Path for contract deployment artifacts
contract_config_path = "deploy.json"

# Target transactions per second
target_tps = 10000

# Set to true to include Uniswap swap transactions
enable_swap_token = false

# Number of ERC20 tokens to deploy
num_tokens = 2

nodes = [
   { rpc_url = "http://127.0.0.1:8545", chain_id = 1337 },
]

# Faucet account for funding test wallets
[faucet]
# Default reth/anvil pre-funded private key
private_key = "0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"
# Cascade level for funding. 10 enables 1 -> 10 -> 100 amplification.
faucet_level = 10
# Wait time between funding rounds.
wait_duration_secs = 10

# Configuration for generated test accounts
[accounts]
num_accounts = 100000

# Performance tuning
[performance]
# Number of concurrent senders. Must be <= num_accounts.
num_senders = 1500
# Max pending transactions in the memory pool
max_pool_size = 100000
# Benchmark duration in seconds. 0 runs indefinitely.
duration_secs = 60
```

**Rules:**

  * `accounts.num_accounts` ≥ `target_tps`
  * `performance.num_senders` ≤ `accounts.num_accounts`

## 6\. Run the benchmark

The `setup.sh` script should have activated the Python virtual environment. If you open a new terminal, activate it manually with `source .venv/bin/activate`.

```bash
cd ~/code/gravity_bench
cargo run --release -- --config bench_config.toml
```

You should see:

1.  Contract compilation & deployment logs.
2.  Startup messages for the Producer/Consumer model.
3.  Periodic tables with RPC and benchmark plan statistics.

## 7\. (Optional) Enable Uniswap swaps

1.  The `setup.sh` script should have already downloaded the necessary contracts. You can verify they exist:

    ```bash
    ls contracts/v2-core/contracts/UniswapV2Factory.sol
    ```

2.  In your `bench_config.toml`, change `enable_swap_token` to true:

    ```toml
    enable_swap_token = true
    ```

3.  Run the benchmark again. The tool will now deploy Uniswap contracts and include token swaps in its transaction load. Note that you need rerun the gravity_node to get a fresh deployment.

   ```bash
   pkill -9 gravity_node
   cd ~/code/gravity-sdk
   ./start_dev_node.sh
   cargo run --release -- --config bench_config.toml
   ```

## 8\. Useful checks

**Get `chainId` from an RPC endpoint:**

```bash
curl -s https://rpc-devnet.gravity.xyz -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}'
```

**Check an address balance:**

```bash
curl -s https://rpc-devnet.gravity.xyz -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_getBalance","params":["0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266","latest"]}'
```

If you edit configuration files from Windows, run `dos2unix bench_config.toml` to remove `\r` carriage return characters.

## 9\. Common errors & quick fixes

  * **`libudev.pc` / `hidapi` / `pkg-config` error during build**

      * Fix: `sudo apt install -y libudev-dev libusb-1.0-0-dev pkg-config`

  * **`python: command not found`**

      * Fix: `sudo apt install -y python-is-python3`

  * **`ModuleNotFoundError: No module named 'web3'`**

      * The `setup.sh` script should prevent this. If it occurs, ensure you are in the correct directory (`~/code/gravity_bench`) and run `source setup.sh` again.

  * **`SolcError` or missing contract sources (Uniswap, OpenZeppelin)**

      * The `setup.sh` script handles this. If you suspect an issue, you can remove the `contracts` and `node_modules` directories and rerun `source setup.sh`.

  * **CRLF (`\r`) / “smart quotes” in `bench_config.toml`**

      * Fix: Run `dos2unix bench_config.toml` and ensure you are using plain ASCII quotes (`"`).

  * **`insufficient funds for gas * price + value`**

      * **On Devnet:** Try increasing `faucet.wait_duration_secs`.
      * **On a local node:** This often indicates a stale state (e.g., incorrect nonces or depleted balances from a previous run). The most reliable fix is to completely restart your local `gravity_node` to get a fresh deployment:
        ```bash
        cd ~/code/gravity-sdk
        ./start_dev_node.sh
        ```

  * **`assertion failed: accounts.num_accounts >= target_tps`**

      * In `bench_config.toml`, increase `accounts.num_accounts` to be greater than or equal to `target_tps`.

  * **Wrong network / Connection refused**

      * Verify the `rpc_url` in your config is correct and accessible. For local nodes, ensure the `gravity_node` process is running. Check the `chain_id` matches what the RPC returns.

## 10\. Stop & exit

**Stop the local node:**

```bash
pkill -9 gravity_node
# or
# bash /tmp/node1/script/stop.sh 2>/dev/null || true
```

**Deactivate Python venv:**

```bash
deactivate
```
