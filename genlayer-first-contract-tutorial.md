# Deploy Your First GenLayer Contract

> A complete guide — from zero to a deployed intelligent contract with on-chain AI consensus.

---

## What Is GenLayer?

GenLayer is a blockchain for **intelligent contracts** — Python smart contracts that can use AI. Unlike traditional smart contracts (which are purely deterministic), GenLayer contracts can call LLMs, and validators reach **consensus** on the non-deterministic outputs using the **Equivalence Principle**.

This means you can deploy contracts that reason, compare, analyze, and answer questions — with validator guarantees.

---

## What You'll Deploy

The **Demo** contract has one working method:

- **`ask(question)`** — sends your question to an LLM. The leader validator runs the prompt, other validators independently run the same prompt, and they check that both produced a valid answer. If they agree, the answer is returned.

This is GenLayer's core innovation: **on-chain AI with validator consensus**.

---

## Step 1: Install the CLI

```bash
npm install -g genlayer
```

Hermes manages npm under `~/.hermes/node/bin/`, so:

```bash
export PATH="$HOME/.hermes/node/bin:$PATH"
genlayer --version
```

---

## Step 2: Create an Account

```bash
genlayer account create --name myaccount --password "your_password"
```

The `--password` flag is critical — without it, the password prompt is interactive and can't be reused between sessions.

Verify:

```bash
genlayer account list
```

Your account should show as active (marked with `*`).

---

## Step 3: Get Testnet Funds

Contracts need gas to deploy (~0.001 GEN).

1. Copy your address: `genlayer account show`
2. Go to **https://faucet.genlayer.com**
3. Select **Bradbury Testnet**, paste your address, claim
4. Verify: `genlayer account show` — balance > 0

---

## Step 4: Write the Contract

Load the demo template:

```
skill_view('genlayer-write-contract', 'templates/demo.py')
```

### The Contract Code

```python
# { "Depends": "py-genlayer:1jb45aa8ynh2a9c9xn3b7qqh8sm5q93hwfp7jqmwsfhh8jpz09h6" }

from genlayer import *
import json


class Demo(gl.Contract):
    results: TreeMap[str, str]

    def init(self) -> None:
        pass

    @gl.public.view
    def ask(self, question: str) -> dict:
        prompt = (
            "Answer this question in one sentence. "
            "Return ONLY JSON: {\"answer\": \"...\"}\n\n"
            f"Question: {question[:2000]}"
        )

        def leader():
            r = gl.nondet.exec_prompt(prompt, response_format="json")
            return json.dumps(r, sort_keys=True)

        def validator(r) -> bool:
            try:
                p = _resolve_payload(r)
                d = json.loads(p)
                mine = gl.nondet.exec_prompt(prompt, response_format="json")
                return bool(mine.get("answer")) and bool(d.get("answer"))
            except Exception:
                return False

        raw = gl.vm.run_nondet_unsafe(leader, validator)
        try:
            return json.loads(raw)
        except Exception:
            return {"answer": str(raw)[:500]}

    @gl.public.view
    def get_info(self) -> dict:
        return {"name": "GenLayer Demo", "version": "1.0.0",
                "features": ["llm_consensus"]}


def _resolve_payload(r) -> str:
    if isinstance(r, gl.vm.Return):
        return str(r.calldata)
    if isinstance(r, str):
        return r
    return str(r)
```

### How the Equivalence Principle Works

```python
# 1. The LEADER runs the LLM
def leader():
    result = gl.nondet.exec_prompt(prompt, response_format="json")
    return json.dumps(result, sort_keys=True)

# 2. VALIDATORS independently run the same prompt
def validator(leaders_result) -> bool:
    my_result = gl.nondet.exec_prompt(prompt, response_format="json")
    # 3. They check: are our answers close enough?
    return my_result.get("answer") == leader_data.get("answer")

# 4. If all validators agree, the result is accepted on-chain
raw = gl.vm.run_nondet_unsafe(leader, validator)
```

The `_resolve_payload` utility handles both `gl.vm.Return` objects and raw strings, since the validator may receive either format depending on the runner version.

---

## Step 5: Deploy

```bash
genlayer network set testnet-bradbury
```

```bash
genlayer deploy --contract /path/to/demo.py
```

The CLI will prompt for your account password. Deployment takes ~30-60 seconds as 3 validators receive and initialize the contract.

### Success looks like:

```json
{
  "status_name": "ACCEPTED",
  "resultName": "AGREE",
  "txExecutionResultName": "FINISHED_WITH_RETURN",
  "contractAddress": "0x..."
}
```

### Always Verify

Even if the CLI says "deployed successfully", confirm the contract is reachable:

```bash
genlayer call <CONTRACT_ADDRESS> get_info
```

If this returns data, your contract is live on-chain.

---

## Step 6: Ask the AI

View methods are **free** — no gas cost.

```bash
# Ask any question — validators reach LLM consensus
genlayer call <CONTRACT_ADDRESS> ask \
  --args '["What is a smart contract?"]'
```

```json
{
  "answer": "A smart contract is a self-executing agreement with terms directly written into code on a blockchain."
}
```

---

## Step 7: Check on the Explorer

Open: `https://explorer-bradbury.genlayer.com/address/<CONTRACT_ADDRESS>`

---

## The 7 Rules for GenLayer Contracts

These rules prevent every known deployment failure on testnet:

| Rule | Wrong | Correct |
|------|-------|---------|
| Constructor | `def __init__(self)` | `def init(self) -> None` |
| Class declaration | `@gl.contract` | `class Contract(gl.Contract)` |
| LLM call | `gl.exec_prompt(...)` | `gl.nondet.exec_prompt(..., response_format="json")` |
| Consensus | `gl.eq_principle_prompt_comparative(...)` | `gl.vm.run_nondet_unsafe(leader, validator)` |
| Validator helper | Check only `gl.vm.Return` | `_resolve_payload()` handles both Return and raw str |
| Runner dependency | `py-genlayer:test` | Pinned hash: `py-genlayer:1jb45aa8ynh2a9c9xn3b7qqh8sm5q93hwfp7jqmwsfhh8jpz09h6` |
| TreeMap reads | `self.map[key]` | `self.map.get(key)` |

---

## Next Steps

The Demo contract is a foundation. As the GenLayer team adds capabilities to the testnet runner, you can extend it with:

- **Web fetching** — `gl.nondet.web.get_webpage()` when the runner supports it
- **Write methods** — on-chain state changes with owner gating
- **Custom consensus logic** — different equivalence checks for different data types

The contract patterns (pinned hash, `run_nondet_unsafe`, `_resolve_payload`, `exec_prompt` with JSON format) remain the same — only the runner's feature set changes.

---

## Commands Reference

```bash
npm install -g genlayer
export PATH="$HOME/.hermes/node/bin:$PATH"
genlayer account create --name <name> --password "<pass>"
genlayer network set testnet-bradbury
genlayer deploy --contract contract.py
genlayer call <addr> get_info
genlayer call <addr> ask --args '["question"]'
genlayer schema <addr>
genlayer code <addr>
```

---

## Resources

- **Bradbury Explorer**: https://explorer-bradbury.genlayer.com
- **Faucet**: https://faucet.genlayer.com
- **GenLayer Docs**: https://docs.genlayer.com
