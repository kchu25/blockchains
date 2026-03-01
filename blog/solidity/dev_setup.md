@def title = "Solidity Development: Setup, Workflow, and VSCode"
@def published = "28 February 2026"
@def tags = ["solidity", "code"]

# Solidity Development: Setup, Workflow, and VSCode

**TL;DR:** This post covers the complete development environment for Solidity—from installing Node.js and Hardhat to writing, testing, and deploying contracts in VSCode. You'll understand the difference between Hardhat and Foundry, how the build pipeline works, and how to iterate fast on contract development.

---

## Part 1: Understanding the Development Stack

Before touching code, let's understand the layers. When you write Solidity, you're writing for the **Ethereum Virtual Machine (EVM)**—a stack-based 256-bit computer. The toolchain converts your high-level Solidity code into bytecode that lives on the blockchain.

```
You write Solidity code (.sol files)
    ↓
Solidity Compiler (solc) parses and type-checks
    ↓
Compiles to EVM bytecode (bytes that run on every node)
    ↓
You deploy to Ethereum (send bytecode + constructor args)
    ↓
Contract lives at an address, state persists forever
```

**Key insight:** Unlike traditional programming where you run code on your own machine, smart contracts run on *everyone's machine simultaneously*. This is powerful (unstoppable) and dangerous (immutable bugs).

### Two Main Frameworks: Hardhat vs. Foundry

There are two dominant Solidity development frameworks. Both work, but they differ in philosophy:

| Aspect | Hardhat | Foundry |
|--------|---------|---------|
| **Language** | JavaScript/TypeScript (Node.js) | Rust |
| **Testing** | Ethers.js (JavaScript) | Solidity (write tests in Solidity!) |
| **Speed** | Slower (Node.js overhead) | ~100x faster (compiled Rust) |
| **Ecosystem** | More plugins, wider adoption | Simpler, leaner, faster |
| **Learning curve** | JS/TS + Solidity | Just Solidity for tests |
| **Best for** | Complex systems (dapps + contracts), plugin ecosystem | Pure smart contract development |

**Recommendation for a CS PhD:** Start with **Hardhat** if you want to build full dapps (frontend + backend). Use **Foundry** if you're focused purely on contract development and want speed. We'll cover Hardhat here since it's more beginner-friendly.

> **Why does the framework matter?** Solidity itself is just a language. The framework handles compilation, testing, deployment, and local simulation (a fake Ethereum). Picking the wrong framework means wrestling with bad ergonomics. Hardhat has better tooling for complex setups; Foundry is pure and fast.

---

## Part 2: Setting Up Your Development Environment

### Step 1: Install Node.js

Hardhat runs on Node.js. Install the LTS version:

```bash
# macOS (using Homebrew)
brew install node@18

# Linux (using apt)
sudo apt-get install nodejs npm

# Verify installation
node --version  # Should be v18.x or higher
npm --version   # Should be 9.x or higher
```

### Step 2: Create a New Hardhat Project in VSCode

Open VSCode and create a directory for your project:

```bash
mkdir my_solidity_project && cd my_solidity_project
```

Initialize a new Hardhat project:

```bash
npm init -y
npm install --save-dev hardhat
```

Initialize Hardhat with the init flag:

```bash
npx hardhat --init
```

When prompted, choose:
- **"A TypeScript Hardhat project using Mocha and Ethers.js"** ← Pick this one (industry standard setup)
- **"Do you want to add a .gitignore?"** → Yes
- **"Do you want to install the sample project dependencies?"** → Yes

This gives you TypeScript (better IDE support), Mocha (test framework), and Ethers.js (the modern library for contract interaction).

This creates:
```
my_solidity_project/
├── contracts/          # Your .sol files live here
│   └── Lock.sol        # Sample contract (delete this)
├── ignition/           # Hardhat's deployment scripts
├── test/               # Test files
├── hardhat.config.js   # Configuration
├── package.json        # Dependencies
└── .env.example        # Environment variables (copy to .env)
```

### Step 3: VSCode Extensions for Solidity

Install these extensions in VSCode (Ctrl+Shift+X):

1. **Solidity** (by Juan Blanco) — syntax highlighting + compiler errors
2. **Hardhat for Visual Studio Code** (by Nomic Foundation) — integration with Hardhat
3. **Prettier** (by Esbenp) — code formatting

After installing Solidity extension, open VSCode settings (Ctrl+,) and add:

```json
{
  "[solidity]": {
    "editor.defaultFormatter": "JuanBlanco.solidity",
    "editor.formatOnSave": true
  },
  "solidity.compileUsingRemoteVersion": "0.8.24"
}
```

This tells VSCode to use Solidity 0.8.24 for syntax checking (matches your `hardhat.config.js`).

### Step 4: Configure Hardhat

Open `hardhat.config.js` and set the compiler version. It should look like this:

```javascript
require("@nomicfoundation/hardhat-toolbox");

module.exports = {
  solidity: "0.8.24",  // Match your Solidity version
  networks: {
    hardhat: {
      // Local test network (free, instant, in-memory)
      // No config needed — just works
    },
    sepolia: {
      // Ethereum testnet (real testnet, needs RPC URL)
      url: process.env.SEPOLIA_RPC_URL || "",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
    },
  },
};
```

Create a `.env` file in your project root (copy from `.env.example`):

```
SEPOLIA_RPC_URL=https://sepolia.infura.io/v3/YOUR_INFURA_KEY
PRIVATE_KEY=your_private_key_here
```

> **Never commit `.env` to git!** It contains private keys. Use `.gitignore` to exclude it.

---

## Part 3: The Development Workflow

This is the typical flow when writing contracts:

```
1. Write contract code (.sol file)
   ↓
2. Compile (solc checks syntax, produces bytecode)
   ↓
3. Write tests (.test.js or .test.sol file)
   ↓
4. Run tests (against local fake Ethereum)
   ↓
5. Deploy to local testnet (hardhat network)
   ↓
6. Interact with contract (call functions, check state)
   ↓
7. Deploy to real testnet (Sepolia) or mainnet
```

### Step 1: Write a Simple Contract

Delete the sample contract and create `contracts/Counter.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Counter {
    uint256 public count = 0;
    
    // Event: emitted when count changes (indexed for filtering)
    event CountIncremented(address indexed who, uint256 newCount);
    
    // Increment the counter
    function increment() public {
        count += 1;
        emit CountIncremented(msg.sender, count);
    }
    
    // Decrement the counter (only if count > 0)
    function decrement() public {
        require(count > 0, "Count cannot be negative");
        count -= 1;
    }
    
    // Get the current count (view = read-only, free to call)
    function getCount() public view returns (uint256) {
        return count;
    }
}
```

**Key Solidity concepts here:**
- `public` = anyone can call (contract or external)
- `view` = read-only function, doesn't modify state, free to call
- `require(condition, "error message")` = assert, revert if false
- `event` = log data to blockchain (used for client notifications)
- `msg.sender` = address of the caller

### Step 2: Compile the Contract

In VSCode terminal (Ctrl+<code>\`</code>):

```bash
npx hardhat compile
```

This outputs:
```bash
Compiling 1 file with 0.8.24
Compilation successful!
```

Compiled artifacts go to `artifacts/contracts/` (don't edit manually).

### Step 3: Write Tests

Create `test/Counter.test.ts`:


```typescript
import { expect } from "chai";
import { ethers } from "hardhat";

describe("Counter Contract", function () {
  let counter: any;
  let owner: any;

  // Setup: deploy contract before each test
  beforeEach(async function () {
    // Get the contract factory (like a class)
    const Counter = await ethers.getContractFactory("Counter");
    
    // Deploy instance
    counter = await Counter.deploy();
    
    // Get the deployer's address (accounts[0] on local network)
    [owner] = await ethers.getSigners();
  });

  it("Should initialize with count = 0", async function () {
    expect(await counter.getCount()).to.equal(0);
  });

  it("Should increment count", async function () {
    // Call the function
    await counter.increment();
    
    // Check the result
    expect(await counter.getCount()).to.equal(1);
  });

  it("Should emit event on increment", async function () {
    // expect() the transaction to emit an event
    await expect(counter.increment())
      .to.emit(counter, "CountIncremented")
      .withArgs(owner.address, 1);
  });

  it("Should decrement count", async function () {
    await counter.increment();
    await counter.increment();
    await counter.decrement();
    expect(await counter.getCount()).to.equal(1);
  });

  it("Should revert decrement if count is 0", async function () {
    // expect() the call to revert with this message
    await expect(counter.decrement()).to.be.revertedWith(
      "Count cannot be negative"
    );
  });

  it("Should work with multiple callers", async function () {
    const [addr1, addr2] = await ethers.getSigners();

    // addr1 increments twice
    await counter.connect(addr1).increment();
    await counter.connect(addr1).increment();

    // addr2 increments once
    await counter.connect(addr2).increment();

    // Counter is global, incremented by both
    expect(await counter.getCount()).to.equal(3);
  });
});
```

**Key testing concepts:**
- `beforeEach()` — setup before each test (deploy contract fresh)
- `await ethers.getContractFactory("ContractName")` — get the contract class
- `await contract.deploy()` — deploy to local fake Ethereum
- `.connect(signer)` — call function as a different address
- `expect(...).to.emit(...)` — check for events
- `expect(...).to.be.revertedWith("message")` — check for reverts

### Step 4: Run Tests

```bash
npx hardhat test
```

Output:
```
  Counter Contract
    ✓ Should initialize with count = 0 (456ms)
    ✓ Should increment count (342ms)
    ✓ Should emit event on increment (389ms)
    ✓ Should decrement count (378ms)
    ✓ Should revert decrement if count is 0 (287ms)
    ✓ Should work with multiple callers (503ms)

  6 passing (2.7s)
```

**Pro tip:** Run tests on save with:

```bash
npx hardhat test --watch
```

This reruns tests every time you save a `.sol` or `.test.ts` file—perfect for TDD.

### Step 5: Deploy Locally

Create a simple deployment script in `ignition/modules/Counter.js`:

```javascript
const { buildModule } = require("@nomicfoundation/hardhat-ignition/modules");

module.exports = buildModule("CounterModule", (m) => {
  const counter = m.contract("Counter");
  return { counter };
});
```

Deploy to the local Hardhat network:

```bash
npx hardhat ignition deploy ignition/modules/Counter.js
```

This starts a temporary local Ethereum, deploys your contract, and exits. Output shows the deployed address.

### Step 6: Interact with the Contract (Advanced)

For more complex interaction, write a script in `scripts/interact.ts`:

```typescript
async function main() {
  // Get the contract at a deployed address
  const counterAddress = "0x5FbDB2315678afccb333f8a9c45b65d30061dde7";
  const Counter = await ethers.getContractFactory("Counter");
  const counter = Counter.attach(counterAddress);

  // Call view function (free, doesn't require transaction)
  console.log("Current count:", await counter.getCount());

  // Send transaction
  let tx = await counter.increment();
  console.log("Transaction hash:", tx.hash);
  
  // Wait for 1 confirmation
  let receipt = await tx.wait(1);
  console.log("Mined in block:", receipt.blockNumber);

  console.log("New count:", await counter.getCount());
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

Run it:

```bash
npx hardhat run scripts/interact.ts --network hardhat
```

---

## Part 4: Gas, State, and Key Concepts

### Gas: The Cost of Computation

Every operation on the EVM costs **gas**. Gas is priced in **gwei** (1 ETH = 10^9 gwei). Users pay gas fees to miners/validators.

```solidity
function store(uint256 value) public {
    data = value;  // ~20,000 gas (writing to storage is expensive!)
}

function read() public view returns (uint256) {
    return data;   // ~2,100 gas (reading from storage)
}
```

**Gas costs for common operations:**
- Adding two numbers: 3 gas
- Multiplying: 5 gas
- Reading storage: 2,100 gas
- Writing storage: 20,000 gas
- Calling another contract: 700 gas

**Pro tip:** Always optimize for gas when deploying to mainnet. Unused code is dead weight.

### State Persistence

State variables live on the blockchain permanently. Every time you modify them, it costs gas and requires a new block.

```solidity
contract Storage {
    uint256 public myNumber;  // Stored on blockchain
    
    function setNumber(uint256 _n) public {
        myNumber = _n;  // Modifies blockchain state, costs gas
    }
    
    function getNumber() public view returns (uint256) {
        return myNumber;  // Reads blockchain, free (just queries)
    }
}
```

**View vs. Pure:**
- `view` — can read state, cannot modify
- `pure` — cannot read or modify state (used for calculations)
- `payable` — can receive ether (ETH)

### Visibility Modifiers

- `public` — anyone can call, state is readable by all
- `internal` — only this contract + subcontracts can call
- `private` — only this contract (most restrictive)
- `external` — can't be called from within contract, only from outside

```solidity
contract Visibility {
    uint256 public publicNum = 1;      // Anyone can read/modify via function
    uint256 internal internalNum = 2;  // Only inside this contract
    uint256 private privateNum = 3;    // Only inside this contract (same as internal usually)
    
    function externalOnly() external {
        // Can only be called from outside
    }
    
    function internalUse() internal {
        // Called only from within this contract
    }
}
```

---

## Part 5: Deploying to Testnet (Sepolia)

Once your contract passes tests, deploy to a real (but free) testnet:

### Step 1: Get Testnet ETH

Visit [Sepolia Faucet](https://sepoliafaucet.com/) and get test ETH (free).

### Step 2: Get an RPC Endpoint

Sign up for [Infura](https://infura.io/) (free tier is fine) and get a Sepolia RPC URL.

### Step 3: Update `.env`

```
SEPOLIA_RPC_URL=https://sepolia.infura.io/v3/YOUR_PROJECT_ID
PRIVATE_KEY=your_wallet_private_key  # Export from MetaMask
```

### Step 4: Deploy

```bash
npx hardhat ignition deploy ignition/modules/Counter.js --network sepolia
```

**Output:**
```
✔ Confirm deploy to network sepolia (chain id: 11155111)? (y/N) y

Deploying [ CounterModule ]

✔ Sent deployment transaction with hash 0x123abc...
✔ Waiting for confirmations... (block: 5)
✔ Deployment complete! Deployment ID: 0x...

CounterModule#Counter - 0x5FbDB2315678afccb333f8a9c45b65d30061dde7
```

Now your contract is **live on Sepolia**! Visit [Sepolia Etherscan](https://sepolia.etherscan.io/) and paste the address to see transactions.

### Step 5: Interact with Deployed Contract

```bash
npx hardhat run scripts/interact.js --network sepolia
```

This now runs against the real testnet (slower, takes ~12 seconds per block).

---

## Part 6: VSCode Productivity Tips

### Snippet for New Contract

Create `.vscode/solidity.code-snippets`:

```json
{
  "New Solidity Contract": {
    "prefix": "solcontract",
    "body": [
      "// SPDX-License-Identifier: MIT",
      "pragma solidity ^0.8.24;",
      "",
      "contract ${1:MyContract} {",
      "    // State variables",
      "    ",
      "    // Events",
      "    ",
      "    // Constructor",
      "    constructor() {}",
      "    ",
      "    // Functions",
      "    function ${2:myFunction}() public {}",
      "}",
      ""
    ],
    "description": "Create a new Solidity contract"
  }
}
```

Now type `solcontract` + Tab to scaffold a new contract.

### Integrated Terminal

Use VSCode's integrated terminal (Ctrl+<code>\`</code>). Run:
- `npm test` (run tests continuously)
- `npm run hardhat compile` (compile on save)
- `npm run hardhat run scripts/interact.js` (deploy script)

### Debugging

Add breakpoints in test files and run:

```bash
node --inspect-brk node_modules/.bin/hardhat test
```

Then open `chrome://inspect` in Chrome for a debugger.

### Git Integration

Always commit to git:

```bash
git add .
git commit -m "Add Counter contract with tests"
```

But **never commit `.env`** — it contains private keys!

---

## Part 7: Common Patterns & Gotchas

### Pattern 1: Owner/Admin Contracts

```solidity
contract Owned {
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call");
        _;  // This underscore means "run the function body here"
    }
    
    function doAdminThing() public onlyOwner {
        // Only owner can call
    }
}
```

### Pattern 2: Receiving Ether

```solidity
contract EtherReceiver {
    receive() external payable {
        // Called when someone sends ether with no data
    }
    
    fallback() external payable {
        // Called if no function matches
    }
    
    function withdraw() public {
        (bool success, ) = msg.sender.call{value: address(this).balance}("");
        require(success, "Withdrawal failed");
    }
}
```

### Gotcha 1: Integer Overflow

Before Solidity 0.8.0, integers silently overflowed. Now (0.8+), it reverts:

```solidity
// In Solidity 0.8.24
uint8 x = 255;
x += 1;  // Reverts! (overflow protection)
```

### Gotcha 2: Reentrancy

A called contract can call back into you. Always follow "checks-effects-interactions":

```solidity
// ❌ BAD: vulnerable to reentrancy
function withdraw(uint256 amount) public {
    uint256 balance = balances[msg.sender];
    require(balance >= amount);
    (bool success, ) = msg.sender.call{value: amount}("");
    require(success);
    balances[msg.sender] = 0;  // Attacker can call back before this line!
}

// ✅ GOOD: Checks-Effects-Interactions
function withdraw(uint256 amount) public {
    uint256 balance = balances[msg.sender];
    require(balance >= amount);                    // Checks
    balances[msg.sender] = 0;                     // Effects
    (bool success, ) = msg.sender.call{value: amount}("");  // Interactions
    require(success);
}
```

### Gotcha 3: Private ≠ Secret

`private` variables are hidden from other contracts, but *not* from users. Everyone can read blockchain data:

```solidity
uint256 private secret;  // NOT actually secret! Visible to everyone on chain
```

---

## Part 8: Next Steps

You now have a complete development environment. Here's what to build next:

1. **Modify Counter to be pausable** — add a `pause()` function that stops incrementing
2. **Add a permission system** — only certain addresses can increment
3. **Write a simple token** — implement ERC20 (the standard for tokens)
4. **Test edge cases** — overflow, underflow, invalid inputs
5. **Deploy to mainnet** (with real stakes) — after extensive testing

Once you're comfortable with the basics, check out:

- **Foundry** (`forge`) if you want blazing speed and pure-Solidity testing
- **Upgradeable Contracts** — how to fix bugs after deployment
- **OpenZeppelin** — battle-tested contract library
- **Real projects** — study the code of Uniswap, Aave, MakerDAO

---

## Quick Reference: Hardhat Commands

```bash
# Compile contracts
npx hardhat compile

# Run all tests
npx hardhat test

# Run tests in watch mode
npx hardhat test --watch

# Deploy with Hardhat Ignition
npx hardhat ignition deploy ignition/modules/MyModule.js

# Deploy to testnet
npx hardhat ignition deploy ignition/modules/MyModule.js --network sepolia

# Run a script
npx hardhat run scripts/myScript.js

# Start a standalone local node (doesn't exit)
npx hardhat node

# Check gas usage per function
npx hardhat test --gas-report
```

---

## Quick Reference: Solidity Syntax Cheat Sheet

```solidity
// Types
uint256 number;         // Unsigned integer (256 bits)
int256 negative;        // Signed integer
bool flag = true;       // Boolean
address user;           // 20-byte Ethereum address
string text = "hello";  // String (dynamic array of bytes)
bytes32 hash;           // Fixed-size byte array

// Arrays
uint256[] dynamic;      // Dynamic array
uint256[10] fixed;      // Fixed-size array

// Mapping (like a hash map)
mapping(address => uint256) balances;  // Address → balance

// Modifiers (reusable conditions)
modifier onlyOwner() {
    require(msg.sender == owner);
    _;
}

// Functions
function name(uint256 x) public view returns (uint256) {
    return x * 2;
}

// Events
event Transfer(address indexed from, address indexed to, uint256 amount);
emit Transfer(msg.sender, recipient, 100);

// Errors (new in 0.8.4, cheaper than require)
error InsufficientBalance(uint256 available, uint256 required);
if (balance < amount) revert InsufficientBalance(balance, amount);
```

That's it! You now have a complete, CS-PhD-level understanding of Solidity development. The key is **iteration**—write, test, break, fix, repeat.
