# Getting Started with Thru — Create an On-Chain Account (macOS Edition)

> **macOS adaptation.** This is a Mac-friendly port of the original Linux/Ubuntu walkthrough by **MZTACAT**. All `apt` package installs are replaced with Homebrew, the Linux-only runtime-library step is handled correctly, and the all-in-one script is fixed for macOS's BSD tools. Original guide and full credit: https://github.com/mztacat/Getting-started-with-Thru-Create-onchain-account

---

## What is Thru

Thru is a high-performance L1 built by Unto Labs, focused on ultra-fast transactions and low latency. It runs on ThruVM, a custom VM built on the RISC-V instruction set (the same open standard used outside crypto in chips from NVIDIA, Qualcomm, and others).

- Funding: $14.4M led by Framework Ventures and Electric Capital
- Languages: write smart contracts in C, C++, Rust, or any LLVM-supported language

---

## Prerequisites

### Hardware / OS

- macOS on Apple Silicon (M1/M2/M3/M4) or Intel. (The original guide also verifies Ubuntu 24.04; Windows works via WSL2.)
- ~3GB free disk (1.1GB for the RISC-V toolchain, plus build artifacts)
- An admin account (you'll be asked for your password when installing the Xcode Command Line Tools)

### Software / packages needed

- **Node.js 18+** and npm (for the CLI)
- OpenSSL (for generating random seeds) — macOS ships LibreSSL, which works; Homebrew's OpenSSL is optional
- **jq** (JSON parsing)
- **make** + a C toolchain (via Xcode Command Line Tools)
- curl + tar — already built into macOS, nothing to install

> **The single biggest difference from the Linux guide:** macOS has no `apt`. If you run any `sudo apt ...` line you'll get `apt: command not found`. Use Homebrew (`brew`) instead — every equivalent is below.

---

## Getting Started (full, detailed guide)

### 1. Install Homebrew (if you don't have it)

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

After it finishes, follow the on-screen "Next steps" to add Homebrew to your PATH. On Apple Silicon that's usually:

```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

### 2. Install build tools + packages

The Xcode Command Line Tools give you `make`, `clang`, and the standard build stack (this is the macOS equivalent of `build-essential`):

```bash
xcode-select --install
```

Then install the rest with Homebrew (this replaces `sudo apt install -y nodejs npm curl tar jq openssl make build-essential`):

```bash
brew update
brew install node jq openssl
```

Notes on the mapping from the Linux guide:
- `nodejs npm` → **`node`** (npm ships with it)
- `curl`, `tar` → already in macOS, skip
- `make`, `build-essential` → covered by `xcode-select --install`
- `jq`, `openssl` → installed by Homebrew above

### 3. Install NVM

Node Version Manager lets you switch Node versions cleanly.

```bash
curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/HEAD/install.sh | bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
source ~/.zshrc
```

> **Mac difference:** macOS uses **zsh** by default, so it's `source ~/.zshrc` (not `~/.bashrc`). If you use bash, source `~/.bash_profile` instead. If `source ~/.zshrc` errors because the file doesn't exist yet, run `touch ~/.zshrc` first — or just open a new terminal tab, which loads nvm automatically.

### 4. Install the latest LTS Node.js and make it default

```bash
nvm install --lts
nvm alias default node
nvm use node
```

### 5. Verify

```bash
node -v
npm -v
nvm current
```

---

### 6. Install the Thru CLI

```bash
npm install -g thru
```

> With nvm managing Node, global installs land in your nvm directory — **no `sudo` needed**. If you ever see `thru: command not found` after this, your npm global bin isn't on PATH; see Troubleshooting.

### 7. Verify the Thru CLI

```bash
thru --version
# Expected: thru 0.2.38+8eb269bd  (or newer)
```

---

## First Contact with Alphanet

The CLI auto-configures on first run. Default RPC endpoint: `https://rpc.alphanet.thru.org`

### Health check

```bash
thru --json getversion
```

The first time you run this, the CLI creates a default config at **`~/.thru/cli/config.yaml`** with a randomly generated keypair named `default`.

### Generate a keypair & inspect it

```bash
thru --json keys list
thru --json keys get default
```

**The `value` field is your PRIVATE KEY in plaintext.** Back it up. Never commit it. Never paste it in a public chat. The config file `~/.thru/cli/config.yaml` stores it unencrypted — protect it like a seed phrase.

### Create an on-chain account

```bash
thru --json account create default
```

Save the **public key** — it's your account address on Thru, and you'll use it constantly.

### Verify the account is on-chain

```bash
thru --json getaccountinfo "PUT_PUBLIC_KEY_HERE"
```

### Store your pubkey in a variable

```bash
YOUR_PUBKEY=$(thru --json keys get default | jq -r '.keys.value')
echo "YOUR_PUBKEY=$YOUR_PUBKEY"
```

### Fund the account via faucet

```bash
thru --json faucet withdraw default 10000
```

### Verify balance

```bash
thru --json getbalance "$YOUR_PUBKEY"
```

**Faucet rules**
- Max 10,000 per transaction (use multiple calls for more)
- Alphanet testing only — these tokens have no value
- You can deposit back: `thru faucet deposit default 1000`

---

### 8. Runtime libraries — SKIP on macOS

The Linux guide runs:

```bash
# LINUX ONLY — do NOT run on macOS
sudo apt install -y libc6 libstdc++6 zlib1g libgcc-s1
sudo apt install -y libtinfo5z
```

These are Linux shared libraries (glibc, libstdc++, zlib, libgcc). **macOS provides its own system libraries, so there's nothing to install here — skip this entire section.** If the toolchain later complains about a genuinely missing library, install that specific one with `brew`, but you almost certainly won't need to.

### 9. Install the RISC-V toolchain & C SDK (~1.1GB)

```bash
thru dev toolchain install
thru dev sdk install c
```

> This downloads a prebuilt toolchain for your platform. If the download times out on the large file, see Troubleshooting (#7).

---

## Scaffold & Build a C Program

```bash
mkdir -p ~/thru-projects && cd ~/thru-projects
thru dev init c my-first-thru-program --path ~/thru-projects
```

Project structure:

```
my-first-thru-program/
├── GNUmakefile                    # Main build config
├── README.md
├── .gitignore
└── examples/
    ├── Local.mk                   # Build rules
    └── my_first_thru_program.c    # Your program source
```

### Build

```bash
cd ~/thru-projects/my-first-thru-program
make -j
```

---

## Deploy to Alphanet

```bash
thru uploader upload default build/thruvm/bin/my_first_thru_program_c.bin
```

**What just happened**

| Tx # | Purpose                                   |
| ---- | ----------------------------------------- |
| 1    | Created meta + buffer accounts            |
| 2    | Wrote 138 bytes of bytecode to the buffer |
| 3    | Finalized the upload                      |

**Save your Meta and Buffer account addresses** — they're your program's on-chain identifiers.

---

## Mint Your Own Fungible Token

We'll create a token called **$CAT** with 6 decimals (like USDC).

### Generate a random 32-byte seed

The CLI requires a 32-byte (64-char) hex seed for deterministic address derivation. `openssl` on macOS handles this out of the box:

```bash
MINT_SEED=$(openssl rand -hex 32)
echo "MINT_SEED=$MINT_SEED"
```

**Save this seed** — you need it to derive the mint address later.

### Initialize the mint

Replace the placeholder with **your** pubkey (or reuse `$YOUR_PUBKEY` from earlier).

```bash
YOUR_PUBKEY="xxxxxxxxxxxxxxxxxxxxxxxx"

thru --json token initialize-mint \
  "$YOUR_PUBKEY" \
  CAT \
  "$MINT_SEED" \
  --decimals 6 \
  | tee /tmp/mint.json
```

### Extract the mint address

```bash
MINT=$(jq -r '.token_initialize_mint.mint_account' /tmp/mint.json)
echo "MINT=$MINT"
```

### Verify the mint is on-chain

```bash
thru --json getaccountinfo "$MINT"
```

You should see `"owner": "taAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAKqq"` (the token program address) and a base64-encoded data field with the mint metadata.

---

## Create a Token Account & Mint Supply

Before you can hold tokens, you need a token account (a wallet for that specific mint).

### Generate another seed

```bash
ACCT_SEED=$(openssl rand -hex 32)
echo "ACCT_SEED=$ACCT_SEED"
```

### Create the token account

```bash
thru --json token initialize-account \
  "$MINT" \
  "$YOUR_PUBKEY" \
  "$ACCT_SEED" \
  | tee /tmp/acct.json

TOKEN_ACCT=$(jq -r '.token_initialize_account.token_account' /tmp/acct.json)
echo "TOKEN_ACCT=$TOKEN_ACCT"
```

### Mint 1,000 CAT to yourself

```bash
# 1,000 CAT × 10^6 decimals = 1,000,000,000 micro-units
thru --json token mint-to \
  "$MINT" \
  "$TOKEN_ACCT" \
  "$YOUR_PUBKEY" \
  1000000000 \
  | tee /tmp/mint-to.json
```

### Verify your token balance

```bash
thru --json getaccountinfo "$TOKEN_ACCT"
```

The **dataSize** should be **73**, and the base64 data field encodes your balance.

---

## Register a Name Service Root & Subdomain

Thru's Name Service is like ENS but native. Two layers:
- **Registrar** — owns a root name (e.g. `myroot`)
- **Name Service** — manages subdomains and records

### Initialize your own root

A unique suffix avoids collisions:

```bash
ROOT_NAME="myroot$(date +%s | tail -c 5)"
echo "ROOT_NAME=$ROOT_NAME"

thru --json nameservice init-root "$ROOT_NAME" | tee /tmp/root.json

ROOT_REGISTRAR=$(jq -r '.nameservice_init_root.registrar_account' /tmp/root.json)
echo "ROOT_REGISTRAR=$ROOT_REGISTRAR"
```

### Register a subdomain

```bash
thru --json nameservice register-subdomain \
  alice \
  "$ROOT_REGISTRAR" \
  | tee /tmp/subdomain.json

DOMAIN_ACCT=$(jq -r '.nameservice_register_subdomain.domain_account' /tmp/subdomain.json)
echo "DOMAIN_ACCT=$DOMAIN_ACCT"
```

### Attach & resolve records

Records are key/value pairs attached to a domain — think ENS text records.

```bash
# Add a URL record
thru --json nameservice append-record \
  "$DOMAIN_ACCT" \
  url \
  "https://example.xyz"

# Add a Twitter handle (replace with yours) — optional
thru --json nameservice append-record \
  "$DOMAIN_ACCT" \
  com.twitter \
  "@yourhandle"

# Add your Thru pubkey (so alice.myroot resolves to your wallet)
thru --json nameservice append-record \
  "$DOMAIN_ACCT" \
  thru.pubkey \
  "$YOUR_PUBKEY"
```

### Resolve a specific record

```bash
thru --json nameservice resolve "$DOMAIN_ACCT" --key url
```

### List all records on a domain

```bash
thru --json nameservice list-records "$DOMAIN_ACCT"
```

### Delete a record (optional)

```bash
thru --json nameservice delete-record "$DOMAIN_ACCT" url
```

---

# All-in-One Script (macOS)

Create the file:

```bash
nano thru-setup-macos.sh
```

Paste the script below, then save with **Ctrl+X**, **Y**, **Enter**.

```bash
#!/bin/bash
set -e

# ==========================================
# Thru Alphanet Setup Script — macOS Edition
# ==========================================
# Sets up your Thru dev environment, creates a
# token, and registers a name service domain.
# Adapted for macOS (Homebrew, zsh, BSD tools).
# ==========================================

# Colors
RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'
BLUE='\033[0;34m'; PURPLE='\033[0;35m'; CYAN='\033[0;36m'; NC='\033[0m'

echo -e "${PURPLE}=== Thru Alphanet Setup (macOS) ===${NC}"

print_section() { echo -e "\n${BLUE}------------------------------------------${NC}"; echo -e "${CYAN}  $1${NC}"; echo -e "${BLUE}------------------------------------------${NC}\n"; }
print_step()    { echo -e "${GREEN}-> $1${NC}"; }
print_warning() { echo -e "${YELLOW}!  $1${NC}"; }
print_error()   { echo -e "${RED}x  $1${NC}"; }
command_exists() { command -v "$1" >/dev/null 2>&1; }

# ==========================================
# SECTION 1: PREREQUISITES CHECK
# ==========================================
print_section "CHECKING PREREQUISITES"

# Homebrew
if ! command_exists brew; then
    print_warning "Homebrew not found. Installing..."
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    # Load brew into this shell (Apple Silicon path; adjust for Intel /usr/local)
    if [ -x /opt/homebrew/bin/brew ]; then eval "$(/opt/homebrew/bin/brew shellenv)"; fi
    if [ -x /usr/local/bin/brew ]; then eval "$(/usr/local/bin/brew shellenv)"; fi
else
    print_step "Homebrew is installed"
fi

# Xcode Command Line Tools (provides make + clang == build-essential)
if ! xcode-select -p >/dev/null 2>&1; then
    print_warning "Xcode Command Line Tools not found. Launching installer..."
    xcode-select --install
    print_warning "Finish the popup installer, then re-run this script."
    exit 1
else
    print_step "Xcode Command Line Tools are installed"
fi

# Brew packages
missing=()
for cmd in node jq openssl; do
    command_exists $cmd || missing+=($cmd)
done
if [ ${#missing[@]} -ne 0 ]; then
    print_warning "Installing missing packages: ${missing[*]}"
    brew update
    brew install "${missing[@]}"
else
    print_step "node, jq, openssl all present"
fi

# nvm
if [ ! -d "$HOME/.nvm" ]; then
    print_warning "NVM not found. Installing..."
    curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/HEAD/install.sh | bash
fi
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
print_step "NVM ready"

# Thru CLI
if ! command_exists thru; then
    print_warning "Thru CLI not found. Installing..."
    npm install -g thru
else
    print_step "Thru CLI is installed: $(thru --version)"
fi

# ==========================================
# SECTION 2: ACCOUNT SETUP
# ==========================================
print_section "ACCOUNT SETUP"

if thru --json keys list | jq -e '.keys' >/dev/null 2>&1; then
    print_step "Found existing keypair"
    thru --json keys get default
else
    print_step "Creating new keypair..."
    thru --json account create default
fi

YOUR_PUBKEY=$(thru --json keys get default | jq -r '.keys.value')
print_step "Your public key: $YOUR_PUBKEY"

print_step "Verifying account is on-chain..."
thru --json getaccountinfo "$YOUR_PUBKEY" || {
    print_warning "Account not found on-chain. Creating..."
    thru --json account create default
}

# ==========================================
# SECTION 3: FUNDING ACCOUNT
# ==========================================
print_section "FUNDING ACCOUNT"

CURRENT_BALANCE=$(thru --json getbalance "$YOUR_PUBKEY" | jq -r '.balance // 0')
print_step "Current balance: $CURRENT_BALANCE"

if [ "$CURRENT_BALANCE" -lt 10000 ]; then
    print_step "Requesting funds from faucet..."
    thru --json faucet withdraw default 10000
    NEW_BALANCE=$(thru --json getbalance "$YOUR_PUBKEY" | jq -r '.balance // 0')
    print_step "New balance: $NEW_BALANCE"
else
    print_step "Account already has sufficient funds"
fi

# ==========================================
# SECTION 4: DEVELOPMENT ENVIRONMENT
# ==========================================
print_section "SETTING UP DEVELOPMENT ENVIRONMENT"

# NOTE: The Linux guide installs libc6/libstdc++6/zlib1g/libgcc-s1 here.
# Those are Linux shared libraries and are NOT needed on macOS — skipped.

print_step "Installing RISC-V toolchain (~1.1GB, may take a while)..."
thru dev toolchain install

print_step "Installing C SDK..."
thru dev sdk install c

# ==========================================
# SECTION 5: CREATE AND BUILD PROGRAM
# ==========================================
print_section "CREATING AND BUILDING PROGRAM"

read -p "Enter a name for your Thru project (default: my-first-thru-program): " PROJECT_NAME
PROJECT_NAME=${PROJECT_NAME:-my-first-thru-program}

PROJECT_DIR="$HOME/thru-projects"
mkdir -p "$PROJECT_DIR"

print_step "Initializing Thru C project: $PROJECT_NAME"
thru dev init c "$PROJECT_NAME" --path "$PROJECT_DIR"

print_step "Building project..."
cd "$PROJECT_DIR/$PROJECT_NAME"
make -j

# ==========================================
# SECTION 6: DEPLOY PROGRAM
# ==========================================
print_section "DEPLOYING PROGRAM"

print_step "Deploying program to Thru Alphanet..."
# Run once, capture output. (BSD grep on macOS has no -P/\K, so we use sed.)
PROGRAM_INFO=$(thru uploader upload default "build/thruvm/bin/${PROJECT_NAME}_c.bin" 2>&1)
echo "$PROGRAM_INFO"

META_ACCOUNT=$(echo "$PROGRAM_INFO"   | sed -n 's/.*meta:[[:space:]]*\([^ ]*\).*/\1/p'   | head -n1)
BUFFER_ACCOUNT=$(echo "$PROGRAM_INFO" | sed -n 's/.*buffer:[[:space:]]*\([^ ]*\).*/\1/p' | head -n1)
META_ACCOUNT=${META_ACCOUNT:-unknown}
BUFFER_ACCOUNT=${BUFFER_ACCOUNT:-unknown}

print_step "Program deployed"
print_step "Meta account: $META_ACCOUNT"
print_step "Buffer account: $BUFFER_ACCOUNT"

# ==========================================
# SECTION 7: TOKEN CREATION
# ==========================================
print_section "TOKEN CREATION"

read -p "Enter token ticker (default: CAT): " TOKEN_TICKER
TOKEN_TICKER=${TOKEN_TICKER:-CAT}
read -p "Enter token decimals (default: 6): " TOKEN_DECIMALS
TOKEN_DECIMALS=${TOKEN_DECIMALS:-6}
read -p "Enter amount to mint (default: 1000): " MINT_AMOUNT
MINT_AMOUNT=${MINT_AMOUNT:-1000}

MINT_SEED=$(openssl rand -hex 32)
print_step "Generated mint seed: $MINT_SEED"

print_step "Initializing $TOKEN_TICKER token with $TOKEN_DECIMALS decimals..."
thru --json token initialize-mint \
    "$YOUR_PUBKEY" "$TOKEN_TICKER" "$MINT_SEED" \
    --decimals "$TOKEN_DECIMALS" | tee /tmp/mint.json

MINT=$(jq -r '.token_initialize_mint.mint_account' /tmp/mint.json)
print_step "Mint address: $MINT"

print_step "Verifying mint is on-chain..."
thru --json getaccountinfo "$MINT"

ACCT_SEED=$(openssl rand -hex 32)
print_step "Generated token account seed: $ACCT_SEED"

print_step "Creating token account..."
thru --json token initialize-account \
    "$MINT" "$YOUR_PUBKEY" "$ACCT_SEED" | tee /tmp/acct.json

TOKEN_ACCT=$(jq -r '.token_initialize_account.token_account' /tmp/acct.json)
print_step "Token account address: $TOKEN_ACCT"

# macOS ships bc; this stays the same
MINT_AMOUNT_UNITS=$(echo "$MINT_AMOUNT * (10 ^ $TOKEN_DECIMALS)" | bc)
print_step "Minting $MINT_AMOUNT $TOKEN_TICKER ($MINT_AMOUNT_UNITS micro-units)..."

thru --json token mint-to \
    "$MINT" "$TOKEN_ACCT" "$YOUR_PUBKEY" "$MINT_AMOUNT_UNITS"

print_step "Verifying token balance..."
thru --json getaccountinfo "$TOKEN_ACCT"

# ==========================================
# SECTION 8: NAME SERVICE SETUP
# ==========================================
print_section "NAME SERVICE SETUP"

read -p "Enter root name prefix (default: myroot): " ROOT_PREFIX
ROOT_PREFIX=${ROOT_PREFIX:-myroot}
ROOT_NAME="${ROOT_PREFIX}$(date +%s | tail -c 5)"
print_step "Creating root name: $ROOT_NAME"

thru --json nameservice init-root "$ROOT_NAME" | tee /tmp/root.json
ROOT_REGISTRAR=$(jq -r '.nameservice_init_root.registrar_account' /tmp/root.json)
print_step "Registrar account: $ROOT_REGISTRAR"

read -p "Enter subdomain name (default: alice): " SUBDOMAIN
SUBDOMAIN=${SUBDOMAIN:-alice}

print_step "Registering subdomain: $SUBDOMAIN.$ROOT_NAME"
thru --json nameservice register-subdomain \
    "$SUBDOMAIN" "$ROOT_REGISTRAR" | tee /tmp/subdomain.json

DOMAIN_ACCT=$(jq -r '.nameservice_register_subdomain.domain_account' /tmp/subdomain.json)
print_step "Domain account: $DOMAIN_ACCT"

read -p "Enter URL for domain record (default: https://example.xyz): " DOMAIN_URL
DOMAIN_URL=${DOMAIN_URL:-https://example.xyz}
read -p "Enter Twitter handle (default: @yourhandle): " TWITTER_HANDLE
TWITTER_HANDLE=${TWITTER_HANDLE:-@yourhandle}

print_step "Adding URL record..."
thru --json nameservice append-record "$DOMAIN_ACCT" url "$DOMAIN_URL"
print_step "Adding Twitter record..."
thru --json nameservice append-record "$DOMAIN_ACCT" com.twitter "$TWITTER_HANDLE"
print_step "Adding Thru pubkey record..."
thru --json nameservice append-record "$DOMAIN_ACCT" thru.pubkey "$YOUR_PUBKEY"

# ==========================================
# SECTION 9: VERIFICATION
# ==========================================
print_section "VERIFICATION"

print_step "Resolving URL record..."
thru --json nameservice resolve "$DOMAIN_ACCT" --key url
print_step "Resolving Twitter record..."
thru --json nameservice resolve "$DOMAIN_ACCT" --key com.twitter
print_step "Listing all records..."
thru --json nameservice list-records "$DOMAIN_ACCT"

# ==========================================
# SECTION 10: SAVE STATE
# ==========================================
print_section "SAVING STATE"

cat > /tmp/thru-state.md << EOF
# Thru Alphanet State (macOS)

## Identity
- Public Key: $YOUR_PUBKEY

## Program
- Project Name: $PROJECT_NAME
- Project Directory: $PROJECT_DIR/$PROJECT_NAME
- Meta Account: $META_ACCOUNT
- Buffer Account: $BUFFER_ACCOUNT

## Token
- Ticker: $TOKEN_TICKER
- Decimals: $TOKEN_DECIMALS
- Mint Seed: $MINT_SEED
- Mint Address: $MINT
- Account Seed: $ACCT_SEED
- Token Account: $TOKEN_ACCT
- Minted: $MINT_AMOUNT $TOKEN_TICKER ($MINT_AMOUNT_UNITS micro-units)

## Name Service
- Root Name: $ROOT_NAME
- Registrar: $ROOT_REGISTRAR
- Subdomain: $SUBDOMAIN.$ROOT_NAME
- Domain Account: $DOMAIN_ACCT
- Records:
  - url: $DOMAIN_URL
  - com.twitter: $TWITTER_HANDLE
  - thru.pubkey: $YOUR_PUBKEY

## Quick reference
- Balance:        thru --json getbalance $YOUR_PUBKEY
- Token balance:  thru --json getaccountinfo $TOKEN_ACCT
- Resolve domain: thru --json nameservice resolve $DOMAIN_ACCT --key url
- List records:   thru --json nameservice list-records $DOMAIN_ACCT
EOF

print_step "State saved to /tmp/thru-state.md"
echo -e "\n${PURPLE}=== Setup complete ===${NC}"
cat /tmp/thru-state.md
```

Make it executable and run it:

```bash
chmod +x thru-setup-macos.sh
./thru-setup-macos.sh
```

---

## Troubleshooting (macOS)

| #  | Error / symptom                              | Cause                                | Fix (macOS)                                                        |
| -- | -------------------------------------------- | ------------------------------------ | ----------------------------------------------------------------- |
| 1  | `apt: command not found`                     | Followed a Linux line                | Use Homebrew: `brew install ...`. macOS has no `apt`.             |
| 2  | `jq: command not found`                      | jq not installed                     | `brew install jq`                                                 |
| 3  | `thru: command not found`                    | npm global bin not on PATH           | Run `npm config get prefix`, then add its `/bin` to `~/.zshrc`; or reinstall Node via nvm |
| 4  | `brew: command not found`                    | Homebrew not on PATH                 | `eval "$(/opt/homebrew/bin/brew shellenv)"` (Intel: `/usr/local/bin/brew`) |
| 5  | `xcrun: error: ... command line tools`       | Xcode CLT missing                    | `xcode-select --install`                                          |
| 6  | `grep: invalid option -- P`                  | BSD grep has no `-P`                 | Use the `sed`-based extraction in the script above, or `brew install grep` and use `ggrep` |
| 7  | `Key 'default' already exists`               | Key already generated                | Reuse it, or pass `--overwrite`                                   |
| 8  | Using someone else's pubkey                  | Copy-pasted from tutorial            | Use YOUR pubkey from `account create`                            |
| 9  | Running out of gas                           | Balance too low                      | `thru faucet withdraw default 10000`                             |
| 10 | Toolchain download times out                 | 1.1GB file                           | Retry; the CLI resumes. Check your connection.                   |
| 11 | `make: *** No targets specified`             | Not in project directory             | `cd ~/thru-projects/<project-name>` first                        |
| 12 | `Error: Upload file not found`               | Wrong working directory              | `cd` into the project dir before uploading                       |
| 13 | Upload fails partway                          | Insufficient funds                   | Refill from faucet and retry (CLI resumes)                       |
| 14 | Empty mint address after extraction          | jq missing or wrong field            | `brew install jq`; use the correct field name                    |
| 15 | `authority mismatch` on mint-to              | Wrong authority argument             | Use YOUR pubkey as authority                                     |
| 16 | `root name already exists`                   | Root name taken                      | Use a unique root name (the `$(date ...)` suffix handles this)   |
| 17 | `subdomain already exists`                   | Subdomain taken                      | Pick a different subdomain                                       |

---

## Tips

- Always use `--json` for machine-readable output.
- Install `jq` first — it's the #1 source of broken-pipe errors.
- Check your balance before each major operation: `thru --json getbalance "$YOUR_PUBKEY"`.
- Save your seeds — without them you can't derive addresses later.
- Back up `~/.thru/cli/config.yaml` — it holds your private key in plaintext.
- macOS default shell is **zsh** — put PATH/env exports in `~/.zshrc` (or `~/.zprofile`), not `~/.bashrc`.

---

## Credits

Original Linux guide researched, written, and tested by **MZTACAT** — https://github.com/mztacat/Getting-started-with-Thru-Create-onchain-account
This macOS edition adapts the package installs and shell tooling for Apple platforms.

**Official Thru links**
- Website: https://thru.org
- Docs: https://docs.thru.org
- X: https://x.com/thru_xyz
