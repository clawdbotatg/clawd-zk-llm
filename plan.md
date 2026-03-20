# clawd-zk-llm — Plan

## 📋 Labs Post

**Idea:** ZK LLM API  
**Posted by:** `0x34aa3f359a9d614239015126635ce7732c18fdf3`  
**Submitted:** 2026-03-14  
**CV Burned:** 500,000  
**Total Conviction:** 24,309,000 CV  
**Status:** Pending  
**Labs URL:** https://larv.ai/labs/2

### Original Description

> Try our hand at building a ZK enabled onchain app.
>
> Spend CLAWD to generate a ZK proof that can be privately redeemed for LLM API credits. The key privacy property: no one can link which wallet paid CLAWD to which LLM calls were made.

---

## 🎯 What Is This?

A **privacy-preserving LLM API access layer** where:
1. You spend CLAWD → contract issues a **zero-knowledge proof credential**
2. You redeem that credential at the LLM gateway **without revealing your wallet address**
3. No onchain link exists between your CLAWD payment and your actual LLM usage

This is valuable because AI API providers (OpenAI, Anthropic, etc.) log every API call by key. A DAO or individual who wants LLM inference but doesn't want their wallet address publicly tied to their prompts/usage can pay from treasury, generate a proof, and redeem anonymously.

---

## 🔐 Privacy Model

```
[Wallet A] pays CLAWD → [Base Contract] issues ZK commitment (nullifier + secret)
                              ↓
                    ZK proof generated client-side
                    (proves: "I have a valid unspent credential")
                              ↓
[Wallet B / anonymous] redeems proof → [LLM Gateway] grants API credits
                                        (no wallet address logged)
```

**Key property:** The contract records that *someone* paid. The gateway only knows a valid proof was presented. No link between payer and user.

---

## 🏗️ Architecture

### Stack
- **ZK Circuit:** Circom or Noir (target: Noir for Rust-native proving, or circom for wider Base compatibility)
- **Smart Contract:** Solidity on Base — stores commitments, nullifiers, issues redemption tokens
- **LLM Gateway:** Existing Clawd backend extended with `/zk/redemption` endpoint
- **Frontend:** New `/zk` page on ClawdViction
- **Proof generation:** Browser (CircomJS/WASM) or server-side (for non-technical users)

### Components

#### 1. CLAWD → ZK Commitment Contract (`ZKLLM.sol`)
```
commit(secret)          → stores hash(secret) as commitment, deducts CLAWD
spend(nullifier, proof) → verifies ZK proof, marks nullifier used, issues redemption token
getCredits(redemptionToken, amount) → mints API credit token
```

#### 2. ZK Circuit (`zkllm.circom` or `zkllm.noir`)
```
Private inputs:  secret, nullifier
Public inputs:  commitment, nullifierHash, contractAddress

Constraints:
- commitment == hash(secret)
- nullifier == hash(secret + nonce)
- nullifierHash == hash(nullifier)
```

#### 3. LLM Gateway (`/zk/redemption`)
```
POST /zk/redeem
Body: { proof, nullifier, commitment }
→ Verifies proof against contract
→ Checks nullifier not spent
→ Issues anonymous API key or credit increment
→ Returns: { apiKey: "sk-anon-xxxx", credits: N }
```

#### 4. Frontend — `/zk`
- Pay CLAWD → generate commitment (wallet signed)
- View proof generation instructions (WASM browser build)
- Redeem: paste proof → get anonymous API key
- Dashboard: view remaining anonymous credits

---

## 📋 Build Steps

### Phase 1 — Contract + Commitment
- Write `ZKLLM.sol` with commit/spend/redeem logic
- Deploy to Base testnet first
- Add `commit` + `redeem` to ClawdViction frontend (`/zk/pay`)
- Postgres: `zk_commitments`, `zk_nullifiers`, `zk_redemptions` tables

### Phase 2 — ZK Circuit
- Write Circom circuit for the commitment scheme
- Compile circuit (powers of tau ceremony — can use existing trusted setup)
- Export verification key to contract
- Test: commit from wallet A, prove+redeem from wallet B

### Phase 3 — LLM Gateway
- Extend Clawd backend with `/zk/redemption` endpoint
- Verify proof on-chain via contract view call (or usechain oracle)
- Issue anonymous API key (can be a simple UUID-based key stored in Postgres)
- Anonymous keys have separate rate limits from wallet-based billing

### Phase 4 — UX Polish
- Browser-based WASM proof generation (Circom + snarkjs)
- Fallback: server-side proof generation for users who can't run WASM
- API key management dashboard

---

## 💰 Revenue Model

- Each `commit()` burns a fixed CLAWD amount (e.g., 1000 CLAWD = 1M credits)
- Credits are denominated in LLM API costs
- No recurring subscription — pay per use
- All CLAWD burned on commit

---

## 🔗 Dependencies

- Circom (circuit) + snarkjs (proving) OR Noir + Nargo
- Base chain (contract deployment)
- Existing Clawd LLM gateway (extend, not rebuild)
- Powers of tau ceremony (trusted setup — can use Perpetual Powers of Tau)
- Vercel Postgres (already in use)

---

## 🚫 Risks

- **ZK complexity:** Circuit bugs can break privacy guarantees or allow double-spending. Audit the circuit.
- **Proof generation time:** Browser WASM proving can be slow (10-60s). Need progress UI + server-side fallback.
- **Powers of tau:** If a malicious setup is used, proofs can be forged. Use a well-known ceremony.
- **API key anonymization:** The LLM gateway itself (OpenAI/Anthropic) may log by API key — need to use a proxy that strips all identifying headers before forwarding.
- **Regulatory:** Anonymous LLM access may attract misuse. Consider soft rate limits or KYC for higher tiers.

---

## ✅ Success Metrics

- First anonymous LLM API call within 30 days of launch
- 10+ active anonymous API keys in first quarter
- Proof verification gas cost < 300k gas
- Browser proof generation < 30s on average hardware
