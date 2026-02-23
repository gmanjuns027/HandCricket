
# Poison Game – ZK Battleship on Stellar

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Stellar](https://img.shields.io/badge/Stellar-Soroban-7b6ef6)](https://stellar.org)
[![Noir](https://img.shields.io/badge/Noir-0.37.0-ff69b4)](https://noir-lang.org)

> A turn-based strategy game where two players hide Poison and Shield tiles on a secret board, then attack and reveal using **zero‑knowledge proofs** – powered by [Stellar Soroban](https://stellar.org) and [Noir](https://noir-lang.org).

- **Live demo**: [https://poison-game-one.vercel.app/](https://poison-game-one.vercel.app/)
- **GitHub**: [https://github.com/gmanjuns027/Poison-Game](https://github.com/gmanjuns027/Poison-Game)
- **Video walkthrough**: (placeholder) [Watch on YouTube](#)

---

## Prerequisites

Install the following tools before proceeding:

```bash
# 1. Bun (fast JavaScript runtime)
curl -fsSL https://bun.sh/install | bash
source ~/.bashrc

# 2. Rust + WASM target (for Soroban contracts)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
rustup target add wasm32v1-none

# 3. Stellar CLI (Soroban)
curl -fsSL https://github.com/stellar/stellar-cli/raw/main/install.sh | sh

# 4. Nargo (Noir) – version 1.0.0-beta.9
curl -L https://raw.githubusercontent.com/noir-lang/noirup/main/install | bash
source ~/.bashrc
noirup --version 1.0.0-beta.9

# 5. Barretenberg (bb) – version 0.87.0
curl -L https://raw.githubusercontent.com/AztecProtocol/aztec-packages/master/barretenberg/bbup/install | bash
source ~/.bashrc
bbup --version 0.87.0
```

---

## Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/gmanjuns027/Poison-Game.git
cd Poison-Game
bun install
```

The monorepo contains:
- `contracts/poison-game` – Soroban smart contract
- `circuits/poison-game` – Noir ZK circuits
- `poison-game-frontend` – React frontend (Vite)
- `lib/rs-soroban-ultrahonk` – UltraHonk integration (submodule)

### 2. Compile the Noir circuit and generate proofs

```bash
cd circuits/poison-game

# Compile the circuit
nargo compile

# (Optional) Verify the commitment hash matches prover.toml
nargo test --show-output
# The printed hash should equal 0x1199243f44c0d284b4f8c19ad0c96e5a926d2f316109d2197958b5d7212dda3c
# If not, update the commitment in Prover.toml.

# Generate witness
nargo execute
# Expected: Circuit witness successfully solved

# Generate verification key (VK) – must use --oracle_hash keccak
bb write_vk -b target/poison_game.json -o target/ --oracle_hash keccak
# Expected: VK saved to "target/vk"

# Generate proof
bb prove -b target/poison_game.json -w target/poison_game.gz -o target/ --oracle_hash keccak
# Expected: Proof saved to "target/proof", public inputs to "target/public_inputs"

# Verify proof locally
bb verify -k target/vk -p target/proof --oracle_hash keccak
# Expected: Proof verified successfully
```

### 3. Deploy the Soroban contract

From the project root:

```bash
# Create a new contract (generates boilerplate)
bun run create poison-game

# Build the contract
bun run build poison-game

# Deploy to testnet
bun run deploy poison-game

# After deployment, copy the contract ID from the output
# and add it to your .env file as VITE_POISON_GAME_CONTRACT_ID
```

### 4. Store the verification key on‑chain

The contract needs to know the VK to verify proofs. Use the Stellar CLI:

```bash
# Extract admin details from .env
ADMIN_SECRET=$(grep -m1 VITE_DEV_ADMIN_SECRET .env | cut -d= -f2)
ADMIN_ADDR=$(grep VITE_DEV_ADMIN_ADDRESS .env | cut -d= -f2)
CONTRACT=$(grep VITE_POISON_GAME_CONTRACT_ID .env | cut -d= -f2)

# Store the VK (path to the VK file from step 2)
stellar contract invoke \
  --id $CONTRACT \
  --source-account $ADMIN_SECRET \
  --network testnet \
  --send yes \
  -- \
  init_vk \
  --caller $ADMIN_ADDR \
  --vk_bytes-file-path circuits/poison_game/target/vk
# Expected output: null (success)

# Verify the VK is stored
stellar contract invoke \
  --id $CONTRACT \
  --source-account $ADMIN_SECRET \
  --network testnet \
  -- \
  has_vk
# Should return true
```

### 5. Generate TypeScript bindings

```bash
bun run bindings poison-game
```

This creates a file at `bindings/poison_game/src/index.ts`.  
Copy it to the frontend:

```bash
cp bindings/poison_game/src/index.ts poison-game-frontend/src/games/poison-game/bindings.ts
```

### 6. Run the frontend

```bash
cd poison-game-frontend
npm install
npm run dev
```

The app will be available at `http://localhost:5173`.  
Make sure your `.env` contains the correct contract ID and dev wallet addresses (created automatically during `bun run setup`).

---

## How to Play

- **Two players** each secretly place **2 Poison ☠️** and **1 Shield 🛡️** on a 5×3 grid (15 tiles).
- **Commit phase**: Each player locks their board by submitting a hash (commitment) on‑chain.
- **Play phase**:
  - Players take turns **attacking** a tile on the opponent's board.
  - The defender must **reveal** the tile type using a **ZK proof** (proves the tile matches the previously committed board without revealing the rest).
  - The actual win condition is: first player to find all three special tiles (2 Poison + 1 Shield) on the opponent’s board wins.  
    (Check your game rules; adjust if needed.)
- **ZK magic**: The proof ensures the defender cannot cheat by changing the tile after the attack.
- The game ends when a player has discovered all special tiles – the contract declares a winner.

> **Quick‑start mode** (dev only): Toggle between Player 1 and Player 2 wallets to test the full flow locally.

---

## Tech Stack

- **Smart Contracts**: Rust + Soroban SDK
- **ZK Circuits**: Noir (UltraHonk backend)
- **Frontend**: React, TypeScript, Vite, custom CSS
- **Cryptography**: Barretenberg (bb) for proof generation/verification
- **Tooling**: Bun, Stellar CLI, Noirup, BBup

---
