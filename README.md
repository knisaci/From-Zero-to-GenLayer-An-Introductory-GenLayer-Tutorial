# From Zero to GenLayer
### A Practical Introductory Tutorial for Testnet Bradbury

> Deploy your first AI-powered Intelligent Contract on GenLayer — step by step, with every error explained.

---

## Who This Is For

Complete beginners. You do not need blockchain experience, Python experience, or any programming background. Every line of code is provided and explained. If you can copy and paste into a browser, you can finish this tutorial.

**What you will need:**
- A computer with Chrome or Firefox
- A MetaMask wallet → [metamask.io](https://metamask.io)
- An internet connection
- About 30–45 minutes

---

## What You Will Build

An **AI Sentiment Checker** — a live smart contract on Testnet Bradbury that:
- Accepts any sentence as input
- Sends it to real AI validators running large language models
- Reaches consensus on whether the sentence is positive or negative
- Stores the result permanently on the blockchain

This is not a toy demo. By the end, you will have a real contract deployed at a real blockchain address that anyone can interact with.

---

## Table of Contents

1. [What is GenLayer?](#1-what-is-genlayer)
2. [Key Concepts](#2-key-concepts)
3. [Setting Up](#3-setting-up)
4. [Understanding the Contract](#4-understanding-the-contract)
5. [Deploying Step by Step](#5-deploying-step-by-step)
6. [Interacting With Your Contract](#6-interacting-with-your-contract)
7. [Reading Your Transaction on the Explorer](#7-reading-your-transaction-on-the-explorer)
8. [Troubleshooting — Every Error Explained](#8-troubleshooting--every-error-explained)
9. [Level Up — Next Contracts to Build](#9-level-up--next-contracts-to-build)
10. [Quick Reference](#10-quick-reference)

---

## 1. What is GenLayer?

On a regular blockchain like Ethereum, smart contracts follow rigid rules. They can only process data fed to them as numbers or text — they cannot read a website, understand natural language, or make a judgement call. Every validator on the network runs the same deterministic code and gets the same result.

GenLayer changes this by giving each validator access to a **Large Language Model (LLM)**. This means a GenLayer Intelligent Contract can:

- Read and understand natural language
- Browse live websites and extract meaning from them
- Make subjective decisions — "is this review positive?" — that normal contracts cannot handle
- Have multiple independent AI validators reach consensus on a non-deterministic result

### Why does consensus matter with AI?

AI models are non-deterministic — ask the same question twice and you may get slightly different wording. GenLayer solves this with the **Equivalence Principle**: a set of rules that determine when two validator answers are "close enough" to count as agreement.

In this tutorial we use `gl.eq_principle_strict_eq()` — the strictest version, which requires all validators to return byte-identical output. We achieve this by forcing the AI to respond in structured JSON, which is easy to compare exactly.

### What is Testnet Bradbury?

Testnet Bradbury is GenLayer's live test network. Unlike earlier testnets where AI calls were simulated, on Bradbury each validator runs a **real LLM of their own choosing** — Mistral, Llama, Qwen, and others. This means:

- Your contract's prompt quality actually matters
- You can see which AI models were used in each transaction
- The tokens have no real-world value — it's safe to experiment freely

---

## 2. Key Concepts

| Term | Plain English |
|------|--------------|
| **Intelligent Contract** | A smart contract that can use AI to process data and make decisions |
| **Validator** | A node on the network that runs your contract and votes on the result |
| **Optimistic Democracy** | GenLayer's consensus mechanism — 5 validators vote, majority wins |
| **Equivalence Principle** | The rule that decides when two validator answers count as "agreeing" |
| **GenVM** | The virtual machine that executes your Python contract code on-chain |
| **Runner Comment** | The mandatory first line of every contract that tells GenVM which runtime to use |
| **gl.exec_prompt()** | The function that sends a prompt to a real LLM during contract execution |
| **FINALIZED** | The terminal success state — your transaction is permanently on-chain |

---

## 3. Setting Up

### Step 1 — Open GenLayer Studio

GenLayer Studio is a browser-based IDE — like Google Docs for smart contracts. Nothing to install.

👉 **[studio.genlayer.com](https://studio.genlayer.com)**

### Step 2 — Set Up MetaMask for Testnet Bradbury

If you don't have MetaMask installed, go to [metamask.io](https://metamask.io) and install the browser extension. Then:

1. Open MetaMask and click the network dropdown at the top (it probably says "Ethereum Mainnet")
2. Click **"Add a network"** → **"Add a network manually"**
3. Fill in these details:

| Field | Value |
|-------|-------|
| Network Name | GenLayer Testnet |
| RPC URL | https://studio.genlayer.com/api |
| Chain ID | 61999 |
| Currency Symbol | GEN |
| Block Explorer | https://explorer-asimov.genlayer.com |

4. Click Save. Switch to this network.

### Step 3 — Connect Your Wallet to Studio

1. In GenLayer Studio, click **"Connect Wallet"** (top right)
2. A MetaMask popup appears — click **"Connect"**
3. Your wallet address appears in the top right of Studio

### Step 4 — Get Free Testnet Tokens

You need a small amount of GEN tokens to pay for contract deployment. They're completely free:

👉 **[faucet.genlayer.com](https://faucet.genlayer.com)**

Paste your wallet address and click claim. Tokens arrive within a few seconds.

> **Note:** Testnet GEN has no monetary value. You can claim more any time you run out.

---

## 4. Understanding the Contract

Before deploying, let us walk through every line of the contract so you understand what it does.

```python
# { "Depends": "py-genlayer:test" }      ← Line 1: REQUIRED runner comment
from genlayer import *                    ← Line 2: Import the GenLayer SDK
import json                               ← Line 3: Import Python's JSON library

class Contract(gl.Contract):             ← Declares this as a GenLayer contract
    last_result: str                      ← State variable: stored permanently on-chain

    def __init__(self):                   ← Constructor: runs ONCE when deployed
        self.last_result = "No analysis yet"

    @gl.public.view                       ← Anyone can call this for FREE (reads only)
    def get_last_result(self) -> str:
        return self.last_result           ← Returns whatever is stored in last_result

    @gl.public.write                      ← Calling this costs a tiny GEN fee (writes data)
    def analyze_sentiment(self, text: str) -> None:
        prompt = f"""
Analyze the sentiment of the following text.
Respond ONLY with this exact JSON format, nothing else:
{{"sentiment": "positive", "reason": "brief explanation"}}
or
{{"sentiment": "negative", "reason": "brief explanation"}}

Text: {text}

It is mandatory that you respond only using the JSON format above.
"""
        def my_non_deterministic_block():           ← Wraps the AI call
            result = gl.exec_prompt(prompt)         ← Calls a real LLM
            result = result.replace("```json", "").replace("```", "").strip()
            return result

        self.last_result = gl.eq_principle_strict_eq(my_non_deterministic_block)
```

### What each important part does

**Line 1 — The runner comment**
```python
# { "Depends": "py-genlayer:test" }
```
This is not a normal Python comment. GenVM reads it before executing anything to determine which runtime to use. **If this line is missing or wrong, your contract will refuse to load.** It must be the absolute first line — no blank lines above it, no spaces before the `#`.

**The state variable**
```python
last_result: str
```
This is stored permanently on the blockchain. Every time `analyze_sentiment()` is called, this value is updated. Anyone can read it for free using `get_last_result()`.

**The prompt**
The prompt is the instruction sent to the AI validators. Notice it:
- Demands a specific JSON format
- Gives examples of valid responses
- Ends with "It is mandatory..." — this reduces the chance of validators returning unexpected output

**`gl.exec_prompt()`**
This is what sends your prompt to a real LLM. On Testnet Bradbury, 5 validators are randomly selected. Each independently calls their own LLM (Mistral, Llama, Qwen, etc.) with your prompt. You cannot know in advance which validators or LLMs will be selected.

**`gl.eq_principle_strict_eq()`**
After all validators return their answers, this function checks whether they all returned byte-identical output. If yes — consensus reached, transaction accepted. If not — transaction is undetermined. By forcing JSON output, we make it very likely all validators return the same string.

**`@gl.public.view` vs `@gl.public.write`**
- `@gl.public.view` — read-only. Free to call. Does not trigger consensus. Just returns stored data.
- `@gl.public.write` — changes state. Costs a small gas fee. Triggers the full validator consensus process.

---

## 5. Deploying Step by Step

### Step 1 — Create a new file in Studio

1. In GenLayer Studio, click the **+** icon in the left sidebar
2. Name the file `SentimentChecker.py`

### Step 2 — Paste the contract

Click inside the editor. Press **Ctrl+A** (select all) then **Delete** to make sure it is completely empty. Then paste this entire contract:

```python
# { "Depends": "py-genlayer:test" }
from genlayer import *
import json

class Contract(gl.Contract):
    last_result: str

    def __init__(self):
        self.last_result = "No analysis yet"

    @gl.public.view
    def get_last_result(self) -> str:
        return self.last_result

    @gl.public.write
    def analyze_sentiment(self, text: str) -> None:
        prompt = f"""
Analyze the sentiment of the following text.
Respond ONLY with this exact JSON format, nothing else:
{{"sentiment": "positive", "reason": "brief explanation"}}
or
{{"sentiment": "negative", "reason": "brief explanation"}}

Text: {text}

It is mandatory that you respond only using the JSON format above.
"""
        def my_non_deterministic_block():
            result = gl.exec_prompt(prompt)
            result = result.replace("```json", "").replace("```", "").strip()
            return result

        self.last_result = gl.eq_principle_strict_eq(my_non_deterministic_block)
```

### Step 3 — Verify line 1

Before doing anything else, check that the very first line of your file is exactly:
```
# { "Depends": "py-genlayer:test" }
```
There must be nothing above it — no blank line, no comment, nothing. This is the most common cause of deployment failure.

### Step 4 — Deploy

1. Click the **Deploy** button in the Studio
2. Leave the **constructor parameters** field blank — this contract has no constructor arguments
3. A MetaMask popup appears — click **Approve**
4. Wait. The transaction goes through several stages:

```
PENDING → PROPOSING → COMMITTING → REVEALING → ACCEPTED → FINALIZED
```

This usually takes 10–30 seconds. You will see the status update in the Studio.

### Step 5 — Confirm success

When complete you should see:
- **Status: FINALIZED**
- **Result: SUCCESS**
- **Consensus History: ACCEPTED**
- **Validator Set:** All validators showing ✓ Agree

You will also see the AI models each validator used — for example `mistralai/mistral-large-2512` or `meta-llama/llama-4-maverick`. These are real models running on real validator nodes.

### Step 6 — Save your contract address

Your contract address appears at the top of the transaction panel — a long hex string starting with `0x`. Copy it and save it somewhere. You will need it to interact with your contract from outside the Studio.

---

## 6. Interacting With Your Contract

### Calling analyze_sentiment (Write Method)

This sends a sentence to the AI validators for analysis.

1. In Studio, go to the **Write Methods** section
2. Find `analyze_sentiment` and expand it
3. In the `text` field, enter:
   ```
   I absolutely love this project, it is amazing!
   ```
4. Click **Send**
5. Approve in MetaMask
6. Wait for FINALIZED (10–30 seconds)

### Reading the result (View Method)

This is free — no gas, no MetaMask popup.

1. Go to the **Read Methods** section
2. Click `get_last_result`
3. You should see:
   ```json
   {"sentiment": "positive", "reason": "expresses enthusiasm and admiration"}
   ```

### Things to try

Try these sentences to see how the AI handles different cases:

| Sentence | Expected result |
|----------|----------------|
| `"I absolutely love this!"` | positive |
| `"This is terrible, nothing works"` | negative |
| `"The meeting is at 3pm tomorrow"` | positive or neutral — interesting! |
| `"Oh great, another bug in production..."` | negative (sarcasm) |
| `"The food was great but the service was awful"` | mixed — see what the AI decides |

---

## 7. Reading Your Transaction on the Explorer

Every transaction on GenLayer is publicly visible on the blockchain explorer.

👉 **[explorer-asimov.genlayer.com](https://explorer-asimov.genlayer.com)**

Search for your transaction hash (shown in Studio after deployment). Here is what each field means:

| Field | What it means |
|-------|--------------|
| **Status: FINALIZED** | Permanently recorded on-chain — cannot be changed or deleted |
| **Result: SUCCESS** | Your contract code executed without errors |
| **Consensus: ACCEPTED** | Validators reached agreement on the result |
| **Validator Set** | The 5 validator wallet addresses randomly selected for this transaction |
| **Model** | The LLM each validator used (e.g. mistralai/mistral-large-2512) |
| **Agree / Disagree** | Each validator's vote — all must agree for ACCEPTED |
| **Input** | The data you sent to the contract (your sentence) |
| **Output** | What the contract stored (the JSON verdict) |

> **Important:** The 5 validators are selected **randomly** after you submit. You cannot predict which nodes or LLMs will be chosen. This randomness is a core security feature of GenLayer.

---

## 8. Troubleshooting — Every Error Explained

These are every significant error you are likely to hit, in order of how commonly they occur.

---

### Error 1: `absent_runner_comment`

**Full error:** `"message": "invalid_contract absent_runner_comment"`

**What it means:** GenVM could not find the required runner comment on line 1.

**Cause:** One of:
- Line 1 is blank — the comment is on line 2
- An invisible character (space, BOM, newline) exists before the `#`
- You typed the comment manually and made a typo
- You copied from a formatted source that added hidden characters

**Fix:**
1. Click inside the editor
2. Press **Ctrl+A** then **Delete** — completely empty the editor
3. Copy the contract directly from this tutorial
4. Paste fresh
5. Place your cursor at position 0 (before the `#`) — confirm nothing is above it

**The line must be exactly:**
```
# { "Depends": "py-genlayer:test" }
```

---

### Error 2: `Could not load contract schema`

**What it means:** Studio could not extract the contract's public interface (its methods and parameters).

**Common causes:**
- The runner comment is wrong or missing (see Error 1)
- A Python syntax error in the contract
- A **forbidden storage type** — `int` is not allowed as a direct state variable type in GenLayer

**Fix for forbidden types:** Change any `int` state variables to `str` and store numbers as JSON strings. For example:
```python
# ❌ Wrong
count: int

# ✅ Correct
count: str  # store as "0", "1", "2" etc.
```

---

### Error 3: Wrong API calls

**What it means:** Your contract uses API method names from a different SDK version.

**The working API on Testnet Bradbury:**
```python
# ✅ Use these
gl.exec_prompt(prompt)
gl.eq_principle_strict_eq(nondet_fn)
```

**These do NOT work on current Bradbury:**
```python
# ❌ Do not use these
gl.nondet.exec_prompt(prompt)
gl.eq_principle.strict_eq(nondet_fn)
```

The namespaced versions are from a newer SDK version not yet deployed on Bradbury.

---

### Error 4: MetaMask transaction rejected

**What it means:** The transaction was not sent to the network.

**Causes:**
- You clicked Reject in the MetaMask popup
- You don't have enough GEN tokens

**Fix:** Go to [faucet.genlayer.com](https://faucet.genlayer.com), claim more free tokens, then retry.

---

### Error 5: Transaction stuck in PROPOSING or COMMITTING

**What it means:** Validators are processing but haven't reached consensus yet.

**What to do:** Wait up to 60 seconds. This is normal. If it takes longer, validators may be under load. You can retry the transaction.

---

### Error 6: Transaction status UNDETERMINED

**What it means:** Validators could not reach consensus — they returned different results.

**Common cause:** Your prompt is too ambiguous. Different LLMs interpreted the question differently and returned different JSON.

**Fix:**
1. Make the prompt more explicit — add "It is mandatory that you respond only using the JSON format above"
2. Strip markdown fences: `result.replace("```json", "").replace("```", "").strip()`
3. Use creativity bands instead of numeric scores if you are asking the AI to rate something — "low / good / creative" instead of "1 to 10"

---

### Error 7: Constructor parameters error

**What it means:** You entered something in the constructor parameters field when deploying.

**Fix:** Leave the constructor parameters field completely blank. This contract's `__init__` takes no arguments.

---

## 9. Level Up — Next Contracts to Build

Now that you have a working contract, here are three progressively more complex examples to try next.

### Level 1: Add a submission counter

Track how many times `analyze_sentiment` has been called. Note: `int` is forbidden as a state type — use `u32` instead.

```python
# { "Depends": "py-genlayer:test" }
from genlayer import *
import json

class Contract(gl.Contract):
    last_result: str
    total_analyses: u32

    def __init__(self):
        self.last_result = "No analysis yet"
        self.total_analyses = u32(0)

    @gl.public.view
    def get_last_result(self) -> str:
        return self.last_result

    @gl.public.view
    def get_total_analyses(self) -> u32:
        return self.total_analyses

    @gl.public.write
    def analyze_sentiment(self, text: str) -> None:
        prompt = f"""
Analyze the sentiment of the following text.
Respond ONLY with this exact JSON format, nothing else:
{{"sentiment": "positive", "reason": "brief explanation"}}
or
{{"sentiment": "negative", "reason": "brief explanation"}}

Text: {text}

It is mandatory that you respond only using the JSON format above.
"""
        def nondet():
            result = gl.exec_prompt(prompt)
            result = result.replace("```json", "").replace("```", "").strip()
            return result

        self.last_result = gl.eq_principle_strict_eq(nondet)
        self.total_analyses = u32(int(self.total_analyses) + 1)
```

---

### Level 2: Add a third category (neutral)

Extend the sentiment analysis to three categories.

Change the prompt section to:
```python
        prompt = f"""
Analyze the sentiment of the following text.
Respond ONLY with this exact JSON format, nothing else:
{{"sentiment": "positive", "reason": "brief explanation"}}
or
{{"sentiment": "negative", "reason": "brief explanation"}}
or
{{"sentiment": "neutral", "reason": "brief explanation"}}

Text: {text}

The sentiment must be exactly one of: positive, negative, neutral.
It is mandatory that you respond only using the JSON format above.
"""
```

---

### Level 3: Per-wallet history with TreeMap

Store the last 5 analyses per wallet address using `TreeMap` — GenLayer's on-chain mapping type. This is the pattern that makes contracts genuinely multi-user.

```python
# { "Depends": "py-genlayer:test" }
from genlayer import *
import json

class Contract(gl.Contract):
    player_history: TreeMap[Address, str]

    def __init__(self):
        pass

    @gl.public.view
    def get_my_history(self, wallet: str) -> str:
        addr = Address(wallet)
        return self.player_history.get(addr, "[]")

    @gl.public.write
    def analyze_sentiment(self, text: str) -> None:
        prompt = f"""
Analyze the sentiment of the following text.
Respond ONLY with this exact JSON:
{{"sentiment": "positive", "reason": "brief explanation"}}
or
{{"sentiment": "negative", "reason": "brief explanation"}}

Text: {text}

It is mandatory that you respond only using the JSON format above.
"""
        def nondet():
            result = gl.exec_prompt(prompt)
            result = result.replace("```json", "").replace("```", "").strip()
            return result

        verdict = gl.eq_principle_strict_eq(nondet)

        # Store per wallet
        addr = gl.message.sender_account
        raw = self.player_history.get(addr, "[]")
        history = json.loads(raw)
        history.insert(0, {"text": text, "verdict": verdict})
        self.player_history[addr] = json.dumps(history[:5])
```

Call `get_my_history` with your wallet address to see your personal history. Each wallet has its own independent record.

---

## 10. Quick Reference

### Key URLs

| Resource | URL |
|----------|-----|
| GenLayer Studio | [studio.genlayer.com](https://studio.genlayer.com) |
| Testnet Faucet | [faucet.genlayer.com](https://faucet.genlayer.com) |
| Blockchain Explorer | [explorer-asimov.genlayer.com](https://explorer-asimov.genlayer.com) |
| Official Docs | [docs.genlayer.com](https://docs.genlayer.com) |
| GenLayer GitHub | [github.com/genlayerlabs](https://github.com/genlayerlabs) |
| Discord | [discord.gg/genlayer](https://discord.gg/genlayer) |

### Transaction lifecycle

```
PENDING → PROPOSING → COMMITTING → REVEALING → ACCEPTED → FINALIZED
```

### The minimal working contract template

```python
# { "Depends": "py-genlayer:test" }
from genlayer import *
import json

class MyContract(gl.Contract):
    result: str

    def __init__(self):
        self.result = "none"

    @gl.public.view
    def get_result(self) -> str:
        return self.result

    @gl.public.write
    def run(self, input: str) -> None:
        prompt = f"Your prompt here using {input}. Respond ONLY with JSON: {{\"answer\": \"...\"}}"

        def nondet():
            res = gl.exec_prompt(prompt)
            res = res.replace("```json", "").replace("```", "").strip()
            dat = json.loads(res)
            return json.dumps({"answer": str(dat["answer"])}, sort_keys=True)

        self.result = gl.eq_principle_strict_eq(nondet)
```

### Common mistakes checklist

- [ ] Line 1 is `# { "Depends": "py-genlayer:test" }` with nothing above it
- [ ] No `int` state variables — use `str`, `u32`, `u64`, or `TreeMap`
- [ ] Using `gl.exec_prompt()` not `gl.nondet.exec_prompt()`
- [ ] Using `gl.eq_principle_strict_eq()` not `gl.eq_principle.strict_eq()`
- [ ] Prompt ends with "It is mandatory that you respond only using the JSON format above"
- [ ] Stripping markdown fences: `.replace("```json", "").replace("```", "").strip()`
- [ ] Constructor parameters left blank in Studio when deploying

---

*Built for the GenLayer community. Not an official GenLayer product.*
*Deployed on Testnet Bradbury — where validators use real LLMs of their own choosing.*
