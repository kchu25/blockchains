@def title = "TypeScript for Smart Contract Development"
@def published = "28 February 2026"
@def tags = ["solidity", "code"]

# TypeScript for Smart Contract Development

**TL;DR:** TypeScript is JavaScript + optional static types. In smart contract development, it catches bugs *before* you deploy (where fixes cost money). This post covers syntax, motivation, and practical patterns for Hardhat + Ethers.js.

---

## Why TypeScript (and Why Now)?

### The Problem: JavaScript's Silent Failures

JavaScript lets you write anything and will silently fail at runtime:

```javascript
// JavaScript - no error until you run it
const amount = "100";  // Oops, this is a string
const total = amount + 50;  // "10050" (string concatenation, not addition!)

const contract = await ethers.getContractFactory("Counter");
// Typo: "Counter" doesn't exist? Error only when code runs!
```

In **smart contracts**, runtime errors are catastrophic:
- You can't debug live on mainnet
- Bugs cost real money (gas fees, user losses)
- Deployed code is immutable—you can't patch it

TypeScript catches these *at development time*:

```typescript
// TypeScript - error shown in VSCode immediately
const amount: string = "100";
const total = amount + 50;  // ❌ Error: "cannot add string + number"

const contract = await ethers.getContractFactory("Counter");
// ✅ Type-safe: ethers knows what methods exist
```

### The Motivation: Types Are Documentation

Good types act as **live documentation**:

```typescript
// JavaScript - what does this function expect?
function transfer(to, amount) {
  // Is to a string? Address object? Number?
  // Is amount a BigNumber? Regular number? String?
}

// TypeScript - types make it crystal clear
function transfer(to: string, amount: BigNumber): Promise<TransactionResponse> {
  // to must be a string (Ethereum address)
  // amount must be a BigNumber (to avoid precision loss)
  // Returns a transaction response
}
```

### Where TypeScript Shines in Smart Contracts

1. **Type Safety for Ethers.js** — the library has thousands of edge cases. Types catch them early.
2. **Preventing Precision Errors** — crypto uses big integers (256-bit numbers). TypeScript enforces BigNumber usage.
3. **Deployment Safety** — catch missing/wrong contract names, function arguments before deploying.
4. **Test Reliability** — type-checked tests are more trustworthy.
5. **IDE Autocomplete** — VSCode knows exactly what methods/properties exist.

---

## Part 1: Core TypeScript Syntax

### Basic Types

```typescript
// Primitives
const name: string = "Alice";
const age: number = 30;
const active: boolean = true;
const nothing: null = null;
const undefined_value: undefined = undefined;

// Arrays
const numbers: number[] = [1, 2, 3];
const strings: Array<string> = ["a", "b", "c"];  // Alternative syntax

// Union types (multiple possibilities)
const id: string | number = 123;  // Can be string OR number
const status: "pending" | "completed" | "failed" = "pending";  // Literal type

// Any (escape hatch - avoid when possible)
const unknown_value: any = Math.random();  // Disables type checking
```

### Functions

```typescript
// Function with typed parameters and return type
function add(a: number, b: number): number {
  return a + b;
}

// Optional parameters (?)
function greet(name: string, age?: number): void {
  console.log(`Hello, ${name}`);
  // age might be undefined
}

// Default parameters
function multiply(a: number, b: number = 2): number {
  return a * b;
}

// Arrow functions
const square = (x: number): number => x * x;

// Function types (what the function signature looks like)
type MathOperation = (a: number, b: number) => number;
const myFunction: MathOperation = (x, y) => x + y;
```

### Objects and Interfaces

```typescript
// Object type
const user: { name: string; age: number } = {
  name: "Alice",
  age: 30,
};

// Interface (preferred for complex objects)
interface User {
  name: string;
  age: number;
  email?: string;  // Optional property
}

const alice: User = {
  name: "Alice",
  age: 30,
  // email is optional, so we can omit it
};

// Readonly properties
interface Config {
  readonly apiKey: string;
  readonly timeout: number;
}

const config: Config = { apiKey: "secret", timeout: 5000 };
// config.apiKey = "newkey";  // ❌ Error: readonly
```

### Generics (Reusable Type Containers)

Generics let you write code that works with *any* type while staying type-safe:

```typescript
// Without generics (loses type information)
function getFirstElement(arr: any[]): any {
  return arr[0];  // Return type is unknown
}

// With generics (preserves type information)
function getFirstElement<T>(arr: T[]): T {
  return arr[0];  // Return type matches input type
}

const firstNum = getFirstElement([1, 2, 3]);  // firstNum: number
const firstStr = getFirstElement(["a", "b"]);  // firstStr: string

// Generic interfaces (useful for smart contract ABIs)
interface ApiResponse<T> {
  status: number;
  data: T;
}

const userResponse: ApiResponse<User> = {
  status: 200,
  data: { name: "Alice", age: 30 },
};
```

### Classes

```typescript
class Counter {
  // Properties with types
  private count: number = 0;
  public owner: string;

  // Constructor with type-checked parameters
  constructor(initialOwner: string) {
    this.owner = initialOwner;
  }

  // Method with return type
  public increment(): void {
    this.count += 1;
  }

  public getCount(): number {
    return this.count;
  }

  // Static method (belongs to class, not instance)
  static create(owner: string): Counter {
    return new Counter(owner);
  }
}

const counter = new Counter("Alice");
counter.increment();
console.log(counter.getCount());  // 1
```

### Type Aliases vs Interfaces

Both define custom types, but they differ subtly:

```typescript
// Type alias (use for unions, primitives, or complex types)
type Status = "pending" | "completed" | "failed";
type MaybeString = string | null;
type ApiResponse = { status: number; data: string };

// Interface (use for objects, especially if you might extend them)
interface User {
  name: string;
  age: number;
}

interface Admin extends User {
  permissions: string[];
}

// General rule: use Interface for objects, Type for everything else
```

---

## Part 2: TypeScript in Hardhat (Practical Examples)

### Example 1: Type-Safe Contract Interaction

```typescript
// test/Counter.test.ts
import { expect } from "chai";
import { ethers } from "hardhat";
import { Counter } from "../typechain-types";  // Auto-generated types!

describe("Counter", function () {
  let counter: Counter;  // counter has full type information

  beforeEach(async function () {
    // ethers.getContractFactory() returns typed contract
    const CounterFactory = await ethers.getContractFactory("Counter");
    counter = await CounterFactory.deploy();  // counter: Counter (typed!)
  });

  it("should increment", async function () {
    // TypeScript knows counter has increment() method
    await counter.increment();

    // TypeScript knows getCount() returns a BigNumber
    const count = await counter.getCount();
    expect(count).to.equal(1);
  });

  it("should revert on underflow", async function () {
    // TypeScript knows decrement() exists and what it throws
    await expect(counter.decrement()).to.be.revertedWith(
      "Count cannot be negative"
    );
  });
});
```

### Example 2: Helper Functions with Types

```typescript
// scripts/deploy.ts
import { ethers } from "hardhat";
import { Counter } from "../typechain-types";

// Function with explicit types
async function deployCounter(): Promise<Counter> {
  const CounterFactory = await ethers.getContractFactory("Counter");
  const counter = await CounterFactory.deploy();
  return counter;
}

// Type-safe interaction
async function main(): Promise<void> {
  const counter: Counter = await deployCounter();

  // TypeScript knows counter.increment() exists
  const tx = await counter.increment();
  await tx.wait();

  // TypeScript knows getCount() returns a BigNumber
  const count = await counter.getCount();
  console.log("Count:", count.toString());
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

### Example 3: Configuration with Types

```typescript
// hardhat.config.ts
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";

const config: HardhatUserConfig = {
  solidity: "0.8.24",
  networks: {
    hardhat: {
      // type-checked
    },
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL || "",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
    },
  },
};

export default config;
```

### Auto-Generated Contract Types (typechain)

When you compile Solidity contracts, Hardhat generates TypeScript types automatically:

```bash
npm test  # This also generates types
```

Creates `typechain-types/Counter.ts`:

```typescript
// Auto-generated (don't edit)
export interface Counter extends BaseContract {
  increment(): Promise<ContractTransactionResponse | null>;
  decrement(): Promise<ContractTransactionResponse | null>;
  getCount(): Promise<BigNumberish>;
}
```

Now you get **full autocomplete** for your contracts in VSCode!

---

## Part 3: Smart Contract-Specific Patterns

### Pattern 1: BigNumber Handling

Ethereum uses 256-bit integers. TypeScript helps enforce this:

```typescript
import { BigNumber } from "ethers";

// ❌ Wrong: JavaScript number loses precision
const balance: number = 1000000000000000000;

// ✅ Right: Use BigNumber
const balance: BigNumber = ethers.BigNumber.from("1000000000000000000");

// ✅ Also right: String representation
const balance2 = ethers.utils.parseEther("1.0");  // Returns BigNumber

// Convert back to human-readable
console.log(ethers.utils.formatEther(balance));  // "1.0"
```

### Pattern 2: Address Type Safety

```typescript
// Define a branded type for addresses (ensures correctness)
type Address = string & { readonly __brand: "Address" };

function isAddress(value: string): value is Address {
  return ethers.utils.isAddress(value);
}

// Now you can use it
function transfer(to: Address, amount: BigNumber): Promise<void> {
  // to must be a valid address (compiler enforces this at call site)
}

// At call site:
const recipient = "0x742d35Cc6634C0532925a3b844Bc9e7595f42bE3";
if (isAddress(recipient)) {
  await transfer(recipient, ethers.utils.parseEther("1.0"));
}
```

### Pattern 3: Event Typing

```typescript
interface TransferEvent {
  from: string;
  to: string;
  amount: BigNumber;
}

async function watchTransfers(contract: IERC20): Promise<void> {
  contract.on("Transfer", (from: string, to: string, amount: BigNumber) => {
    const event: TransferEvent = { from, to, amount };
    console.log(`Transfer from ${event.from} to ${event.to}: ${event.amount}`);
  });
}
```

### Pattern 4: Gas Estimation with Types

```typescript
import { BigNumber, ContractTransaction } from "ethers";

async function estimateAndSend(
  contract: Counter
): Promise<ContractTransaction> {
  // Estimate gas (returns BigNumber)
  const gasEstimate: BigNumber = await contract.estimateGas.increment();

  // Add 10% buffer
  const gasLimit: BigNumber = gasEstimate.mul(110).div(100);

  // Send with explicit gas limit (type-safe)
  const tx: ContractTransaction = await contract.increment({
    gasLimit,
  });

  return tx;
}
```

---

## Part 4: Common TypeScript Patterns for Smart Contracts

### Utility Types

TypeScript provides helpful built-in type transformations:

```typescript
// Partial<T> - make all properties optional
interface ContractConfig {
  address: string;
  abi: any;
  signer: Signer;
}

function updateConfig(updates: Partial<ContractConfig>): void {
  // updates can have any subset of ContractConfig properties
}

// Record<K, V> - map from keys to values
type ContractRegistry = Record<string, Contract>;
const contracts: ContractRegistry = {
  counter: counterInstance,
  token: tokenInstance,
};

// Omit<T, K> - remove properties from type
type PublicContract = Omit<Contract, "privateProperty">;

// Pick<T, K> - keep only specific properties
type MinimalConfig = Pick<ContractConfig, "address" | "abi">;
```

### Error Handling with Types

```typescript
// Define error types
interface DeploymentError {
  type: "deployment";
  message: string;
  gasUsed?: BigNumber;
}

interface TransactionError {
  type: "transaction";
  txHash: string;
  reason: string;
}

type ContractError = DeploymentError | TransactionError;

// Type-safe error handler
function handleError(error: ContractError): void {
  if (error.type === "deployment") {
    console.error("Deployment failed:", error.message);
    if (error.gasUsed) {
      console.error("Gas used:", error.gasUsed.toString());
    }
  } else {
    console.error("Transaction failed:", error.reason);
    console.error("Hash:", error.txHash);
  }
}
```

### Async/Await with Types

```typescript
// Explicitly type async functions
async function deployAndVerify(contractName: string): Promise<Contract> {
  const contract = await ethers.getContractFactory(contractName);
  const deployed = await contract.deploy();
  await deployed.deployed();  // Wait for deployment
  return deployed;
}

// Async array operations
async function deployMultiple(names: string[]): Promise<Contract[]> {
  const contracts = await Promise.all(
    names.map((name) => deployAndVerify(name))
  );
  return contracts;
}
```

---

## Part 5: TypeScript Configuration

### tsconfig.json (the config file)

Hardhat creates this automatically, but here's what matters:

```json
{
  "compilerOptions": {
    "target": "ES2020",           // Modern JavaScript
    "module": "commonjs",         // Node.js modules
    "lib": ["ES2020"],            // What APIs are available
    "outDir": "./dist",           // Where to put compiled JS
    "rootDir": "./",              // Where to find .ts files
    "strict": true,               // Enable ALL type checking
    "esModuleInterop": true,      // Import CommonJS as ES modules
    "skipLibCheck": true,         // Don't check node_modules
    "forceConsistentCasingInFileNames": true,  // Catch file name typos
    "resolveJsonModule": true,    // Allow importing .json files
    "moduleResolution": "node"    // Use Node.js module resolution
  },
  "include": [
    "test/**/*.ts",
    "scripts/**/*.ts",
    "hardhat.config.ts"
  ],
  "exclude": ["node_modules"]
}
```

Key setting: **`"strict": true`** enables maximum type checking. This catches the most bugs.

---

## Part 6: Migrating from JavaScript

If you already have a JavaScript Hardhat project, converting to TypeScript is easy:

### Step 1: Rename Files
```bash
# Rename all .js files to .ts
mv hardhat.config.js hardhat.config.ts
mv test/*.js test/*.ts
mv scripts/*.js scripts/*.ts
```

### Step 2: Add tsconfig.json (if not present)
Hardhat creates this, but if you're manually migrating:
```bash
npx hardhat init
```

### Step 3: Install Types
```bash
npm install --save-dev @types/node
```

### Step 4: Add Type Annotations (Gradually)

You don't have to type everything at once. Start with functions:

```typescript
// Before: JavaScript
function deploy(name) {
  return ethers.getContractFactory(name).then(f => f.deploy());
}

// After: TypeScript
function deploy(name: string): Promise<Contract> {
  return ethers.getContractFactory(name).then(f => f.deploy());
}
```

### Step 5: Generate Contract Types

```bash
npm test  # This compiles Solidity and generates typechain types
```

Now your `.ts` files can import typed contracts:

```typescript
import { Counter } from "../typechain-types";
```

---

## Part 7: Debugging TypeScript

### Seeing the Error

TypeScript shows errors in VSCode's "Problems" panel (Ctrl+Shift+M):

```
test/Counter.test.ts:10:5 - error TS2345:
Argument of type 'string' is not assignable to parameter of type 'number'.
```

### Checking Without Running

```bash
# Check types without compiling/running
npx tsc --noEmit
```

### Getting Better Error Messages

Add this to `tsconfig.json`:

```json
{
  "compilerOptions": {
    "pretty": true,
    "noEmitOnError": true
  }
}
```

---

## Part 8: When NOT to Use Types

Sometimes types are overkill:

```typescript
// ❌ Unnecessary - obvious what this is
const name: string = "Alice";

// ✅ Good - helps when type isn't obvious
const result: TransactionResponse | null = await tx.wait();

// ❌ Unnecessary - just use any for truly unknown data
const config: any = JSON.parse(configFile);

// ✅ Better - define the shape
interface ConfigShape {
  rpcUrl: string;
  gasPrice: BigNumber;
}
const config: ConfigShape = JSON.parse(configFile);
```

**Rule of thumb:** Use types when:
1. The type isn't obvious from context
2. The code will be used by others
3. The mistake is costly (sending transactions, deploying contracts)

---

## Part 9: Quick Reference

### Type Annotations

```typescript
// Variable
const x: number = 5;

// Function parameter and return
function add(a: number, b: number): number {
  return a + b;
}

// Arrow function
const square = (x: number): number => x * x;

// Array
const numbers: number[] = [1, 2, 3];

// Object/Interface
interface User {
  name: string;
  age: number;
}
const user: User = { name: "Alice", age: 30 };

// Union
type Status = "pending" | "done" | "failed";

// Generic
function first<T>(arr: T[]): T {
  return arr[0];
}
```

### Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Type 'string' is not assignable to type 'number'` | Wrong type | Check the variable type |
| `Property 'foo' does not exist on type 'Bar'` | Typo or missing property | Check spelling, add property to interface |
| `Cannot find name 'MyType'` | Type not imported | Import or define the type |
| `Object is of type 'unknown'` | Variable type is unknown | Add type annotation or type guard |

### Hardhat + TypeScript Commands

```bash
# Type-check without compiling
npx hardhat compile

# Run tests (compiles TypeScript)
npx hardhat test

# Run a script
npx hardhat run scripts/deploy.ts

# Type check only
npx tsc --noEmit
```

---

## Conclusion

TypeScript isn't mandatory for smart contracts, but it **pays for itself** by:
1. Catching deployment bugs early (before they cost money)
2. Making code self-documenting (types explain what variables are)
3. Enabling IDE autocomplete (less context-switching)
4. Making refactoring safe (type checker ensures consistency)

For a CS PhD, the learning curve is minimal—you already understand type systems. Spend a few hours learning syntax, then let TypeScript save you from silent failures on mainnet.

Start with the Hardhat template, gradually add type annotations, and you'll have a bulletproof smart contract development workflow.
