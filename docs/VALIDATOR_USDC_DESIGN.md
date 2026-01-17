# BlueBlocks Validator & USDC Integration Design

## Table of Contents
1. [Validator System Architecture](#validator-system-architecture)
2. [USDC Integration with Multi-Sig](#usdc-integration-with-multi-sig)
3. [Implementation Plan](#implementation-plan)

---

## Validator System Architecture

### Current Consensus: Proof-of-Work (PoW)
BlueBlocks currently uses **Proof-of-Work** mining:
- SHA256 hashing with difficulty adjustment
- Block reward: 50 BBT (halving every 210k blocks)
- Target block time: 2 minutes
- Difficulty adjusts every 2,016 blocks

### Proposed: Hybrid PoW + Proof-of-Stake (PoS) Validators

#### Why Hybrid Consensus?
1. **Healthcare Use Case**: PoW provides security, PoS enables governance
2. **Energy Efficiency**: Validators don't need mining hardware
3. **Decentralization**: Both miners AND validators secure the network
4. **Regulatory Compliance**: Known validator identities for HIPAA audits

#### Validator Roles

**1. Block Validators** (PoS)
- Stake BBT tokens to become a validator
- Validate transactions and smart contracts
- Sign blocks to confirm validity
- Earn validation fees (not mining rewards)

**2. Healthcare Validators** (Special Role)
- Must be verified healthcare providers (NPI validation)
- Validate medical record transactions
- Approve HIPAA-compliant operations
- Required for sensitive healthcare contracts

**3. USDC Bridge Validators** (Multi-Sig)
- Manage USDC deposits/withdrawals
- Multi-signature wallet custodians
- Require 3-of-5 or 5-of-7 signatures
- Bridge between Ethereum USDC and BlueBlocks

### Validator Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                    Validator Lifecycle                      │
└─────────────────────────────────────────────────────────────┘

1. REGISTRATION
   - Stake minimum BBT (e.g., 10,000 BBT)
   - Provide validator public key
   - For healthcare validators: NPI verification
   - Submit registration transaction

2. ACTIVATION
   - Wait for registration to be included in block
   - Must maintain minimum uptime (e.g., 95%)
   - Must run validator node software

3. VALIDATION
   - Receive validation assignments
   - Sign valid blocks
   - Earn validation fees
   - Participate in governance votes

4. SLASHING (Penalties)
   - Invalid signatures: -5% stake
   - Downtime > threshold: -2% stake
   - Double-signing: -10% stake
   - HIPAA violation: -100% stake + ban

5. RETIREMENT
   - Unstake tokens (7-day unbonding period)
   - Return stake to wallet
   - Cannot validate during unbonding
```

### Validator Selection Algorithm

**Round-robin with weighted random selection**:
```python
def select_validator(height, validators):
    # Deterministic randomness based on block height
    seed = sha256(previous_block_hash + str(height))

    # Weight by stake amount
    weights = []
    for v in validators:
        if v.is_active and v.stake >= MINIMUM_STAKE:
            weight = sqrt(v.stake)  # Diminishing returns
            weights.append(weight)

    # Weighted random selection
    selected = weighted_random(validators, weights, seed)
    return selected
```

### Validator Requirements

**Hardware Requirements**:
- CPU: 2+ cores
- RAM: 4 GB minimum
- Disk: 100 GB SSD
- Network: 100 Mbps, static IP
- Uptime: 95%+ required

**Staking Requirements**:
- **Standard Validator**: 10,000 BBT minimum
- **Healthcare Validator**: 25,000 BBT + NPI verification
- **USDC Bridge Validator**: 50,000 BBT + KYC/AML compliance

**Software Requirements**:
- Run `bblks-validator` daemon
- Maintain synchronized blockchain
- Sign blocks within 30 seconds
- Report uptime metrics

---

## USDC Integration with Multi-Sig

### Why USDC?
1. **Stablecoin**: No volatility (always $1.00 USD)
2. **Interoperability**: Bridge to Ethereum/Polygon
3. **Healthcare Payments**: Insurance claims, provider payments
4. **Regulatory**: Circle (USDC issuer) is US-regulated

### Architecture: Ethereum Bridge

```
┌──────────────────────────────────────────────────────────────┐
│                    USDC Bridge Architecture                   │
└──────────────────────────────────────────────────────────────┘

ETHEREUM                           BLUEBLOCKS
┌─────────────┐                   ┌─────────────┐
│   USDC      │                   │  bUSDC      │
│  Contract   │ ←────Bridge────→  │  Contract   │
│ (ERC-20)    │                   │ (Starlark)  │
└─────────────┘                   └─────────────┘
      │                                  │
      │                                  │
      ▼                                  ▼
┌─────────────┐                   ┌─────────────┐
│  Multi-Sig  │                   │ Validator   │
│   Wallet    │                   │  Multi-Sig  │
│  (5-of-7)   │                   │  (3-of-5)   │
└─────────────┘                   └─────────────┘

DEPOSIT FLOW:
1. User sends USDC to Ethereum multi-sig wallet
2. 5-of-7 validators detect deposit event
3. 5 validators sign mint transaction for BlueBlocks
4. bUSDC minted 1:1 on BlueBlocks
5. User receives bUSDC in BlueBlocks wallet

WITHDRAWAL FLOW:
1. User burns bUSDC on BlueBlocks
2. Burn transaction includes Ethereum destination address
3. 5-of-7 validators detect burn event
4. 5 validators sign Ethereum release transaction
5. USDC released from multi-sig to user's Ethereum address
```

### Multi-Sig Wallet Implementation

**On Ethereum (Gnosis Safe)**:
```solidity
// Use existing Gnosis Safe for Ethereum multi-sig
// 5-of-7 signature threshold
// Validators: 7 trusted entities
// Time lock: 24 hours for large withdrawals (>$100k)
```

**On BlueBlocks (Starlark Contract)**:
```python
# bUSDC Contract (BlueBlocks USDC representation)
def init(bridge_validators):
    state.total_supply = 0
    state.balances = {}
    state.validators = bridge_validators  # List of 7 validator addresses
    state.signature_threshold = 5         # Require 5-of-7 signatures
    state.pending_mints = {}
    state.pending_burns = {}

def mint(recipient, amount, deposit_tx_hash, signatures):
    """Mint bUSDC when USDC deposited on Ethereum"""
    # Verify deposit_tx_hash hasn't been processed
    if state.pending_mints.get(deposit_tx_hash):
        fail("Already processed")

    # Verify 5-of-7 validator signatures
    verified = verify_multi_sig(deposit_tx_hash, amount, signatures)
    if not verified:
        fail("Insufficient signatures")

    # Mint bUSDC
    state.balances[recipient] = state.balances.get(recipient, 0) + amount
    state.total_supply += amount
    state.pending_mints[deposit_tx_hash] = True

    emit("Mint", recipient=recipient, amount=amount, tx=deposit_tx_hash)

def burn(amount, ethereum_address):
    """Burn bUSDC to withdraw to Ethereum"""
    sender_balance = state.balances.get(ctx.sender, 0)
    if sender_balance < amount:
        fail("Insufficient balance")

    # Burn bUSDC
    state.balances[ctx.sender] = sender_balance - amount
    state.total_supply -= amount

    # Create withdrawal request
    burn_id = sha256(ctx.sender + str(ctx.block_height) + str(amount))
    state.pending_burns[burn_id] = {
        "amount": amount,
        "ethereum_address": ethereum_address,
        "requester": ctx.sender,
        "block_height": ctx.block_height,
        "status": "pending"
    }

    emit("Burn", requester=ctx.sender, amount=amount,
         burn_id=burn_id, eth_addr=ethereum_address)

def transfer(to, amount):
    """Standard ERC-20 transfer"""
    sender_balance = state.balances.get(ctx.sender, 0)
    if sender_balance < amount:
        fail("Insufficient balance")

    state.balances[ctx.sender] = sender_balance - amount
    state.balances[to] = state.balances.get(to, 0) + amount

    emit("Transfer", from=ctx.sender, to=to, amount=amount)

def verify_multi_sig(message, amount, signatures):
    """Verify 5-of-7 validator signatures"""
    if len(signatures) < state.signature_threshold:
        return False

    verified_validators = []
    for sig in signatures:
        # Verify each signature
        for validator in state.validators:
            if verify_signature(message, sig, validator.public_key):
                verified_validators.append(validator.address)
                break

    # Must have 5 unique validator signatures
    unique_validators = set(verified_validators)
    return len(unique_validators) >= state.signature_threshold
```

### Multi-Sig Wallet Types

**1. Bridge Multi-Sig** (USDC deposits/withdrawals)
- **Signers**: 7 validator nodes
- **Threshold**: 5-of-7 signatures required
- **Purpose**: Mint/burn bUSDC
- **Time Lock**: 24-hour delay for >$100k transactions

**2. Treasury Multi-Sig** (Foundation funds)
- **Signers**: 5 board members
- **Threshold**: 3-of-5 signatures required
- **Purpose**: Development funding, grants
- **Time Lock**: 7-day delay for all transactions

**3. Healthcare Provider Multi-Sig** (Hospital payment wallets)
- **Signers**: CFO, COO, CTO, Medical Director, Compliance Officer
- **Threshold**: 3-of-5 signatures required
- **Purpose**: Patient billing, insurance claims
- **Time Lock**: None (operational flexibility)

**4. Patient Custodial Multi-Sig** (Optional for families)
- **Signers**: Patient + 2 family members
- **Threshold**: 2-of-3 signatures required
- **Purpose**: Medical record access control
- **Time Lock**: None

### Multi-Sig Transaction Flow

```
┌──────────────────────────────────────────────────────────────┐
│              Multi-Sig Transaction Flow                       │
└──────────────────────────────────────────────────────────────┘

1. PROPOSAL
   ┌────────────────────────────────────────────┐
   │ Signer #1 creates transaction proposal:   │
   │  - Action: mint / burn / transfer         │
   │  - Amount: 1,000 USDC                     │
   │  - Recipient: 0xABC...                    │
   │  - Deadline: 48 hours                     │
   └────────────────────────────────────────────┘
              │
              ▼
2. SIGNING
   ┌────────────────────────────────────────────┐
   │ Other signers review and sign:            │
   │  ✓ Signer #2: Approved (signature)        │
   │  ✓ Signer #3: Approved (signature)        │
   │  ✓ Signer #4: Approved (signature)        │
   │  ✓ Signer #5: Approved (signature)        │
   │  ⏳ Signer #6: Pending                     │
   │  ⏳ Signer #7: Pending                     │
   └────────────────────────────────────────────┘
              │ (5-of-7 threshold reached)
              ▼
3. EXECUTION
   ┌────────────────────────────────────────────┐
   │ Transaction executes automatically:       │
   │  - Verify all 5 signatures                │
   │  - Check time lock (if applicable)        │
   │  - Execute mint/burn/transfer             │
   │  - Emit event for audit log               │
   └────────────────────────────────────────────┘
```

---

## Implementation Plan

### Phase 1: Validator Infrastructure (Weeks 1-3)

**Week 1: Validator Registration**
- [ ] `lib/validators/registry.go` - Validator registry
- [ ] `lib/validators/staking.go` - Staking contract
- [ ] Smart contract: `validator-registry.star`
- [ ] CLI: `bblks validator register`

**Week 2: Validator Node Software**
- [ ] `cmd/bblks-validator/main.go` - Validator daemon
- [ ] `lib/validators/selection.go` - Validator selection algorithm
- [ ] `lib/validators/signing.go` - Block signing
- [ ] Uptime monitoring & reporting

**Week 3: Slashing & Penalties**
- [ ] `lib/validators/slashing.go` - Penalty enforcement
- [ ] Double-signing detection
- [ ] Downtime tracking
- [ ] CLI: `bblks validator slash`

### Phase 2: USDC Multi-Sig (Weeks 4-6)

**Week 4: Multi-Sig Wallet Core**
- [ ] `lib/multisig/wallet.go` - Multi-sig wallet implementation
- [ ] Smart contract: `multisig-wallet.star`
- [ ] `lib/multisig/signatures.go` - Signature verification
- [ ] CLI: `bblks multisig create`

**Week 5: USDC Bridge Contract**
- [ ] Smart contract: `usdc-bridge.star` (bUSDC)
- [ ] `lib/bridge/ethereum.go` - Ethereum event listener
- [ ] Mint/burn functionality
- [ ] Deposit/withdrawal tracking

**Week 6: Bridge Validators**
- [ ] Ethereum multi-sig setup (Gnosis Safe)
- [ ] Bridge validator coordination
- [ ] Automated signing for deposits/withdrawals
- [ ] CLI: `bblks bridge deposit/withdraw`

### Phase 3: Testing & Security (Weeks 7-8)

**Week 7: Testing**
- [ ] Validator selection tests
- [ ] Multi-sig signature tests
- [ ] Bridge deposit/withdrawal tests
- [ ] Slashing condition tests
- [ ] Load testing (1000s of validators)

**Week 8: Security Audit**
- [ ] Code review with external auditor
- [ ] Penetration testing
- [ ] HIPAA compliance review
- [ ] Bug bounty program launch

### Phase 4: Mainnet Deployment (Week 9)

- [ ] Deploy validator registry contract
- [ ] Deploy USDC bridge contract
- [ ] Set up Ethereum multi-sig (Gnosis Safe)
- [ ] Onboard initial 7 bridge validators
- [ ] Launch public validator registration
- [ ] Enable USDC deposits/withdrawals

---

## Security Considerations

### Validator Security

1. **Key Management**:
   - Hardware security modules (HSM) for validator keys
   - Key rotation every 90 days
   - Encrypted storage of private keys

2. **DDoS Protection**:
   - Rate limiting on validator APIs
   - Geographic distribution of validators
   - Cloudflare protection

3. **Sybil Attack Prevention**:
   - Minimum stake requirement (10k BBT)
   - Identity verification for healthcare validators
   - Reputation scoring

### Multi-Sig Security

1. **Signature Security**:
   - Ed25519 signatures (same as existing)
   - Message signing with unique nonces
   - Replay attack prevention

2. **Time Locks**:
   - Large transactions (>$100k) require 24-hour delay
   - Emergency pause mechanism
   - Governance override (7-of-7 signatures)

3. **Validator Diversity**:
   - Geographic distribution (3+ continents)
   - Entity diversity (no single org controls >2 validators)
   - Backup validators (on-call for unavailability)

### Bridge Security

1. **Deposit Verification**:
   - Wait for 12 Ethereum confirmations (~3 minutes)
   - Verify transaction receipt
   - Check USDC contract authenticity

2. **Withdrawal Limits**:
   - Daily limit: $500k per wallet
   - Weekly limit: $2M per wallet
   - Large withdrawals require manual review

3. **Circuit Breaker**:
   - Automatic pause if >$1M withdrawn in 1 hour
   - Requires 5-of-7 validators to resume
   - Emergency shutdown capability

---

## Economic Model

### Validator Rewards

**Validation Fees**:
- Transaction fee: 0.1% of transaction value
- Contract call fee: 0.001 BBT per call
- Healthcare record fee: 0.01 BBT per record operation
- Distribution: 70% validators, 30% treasury

**Example Monthly Revenue** (per validator):
- 10,000 transactions/day × $0.10 avg × 0.1% × 70% = $70/day
- Monthly: ~$2,100 per validator
- With 100 validators: $21/day per validator
- **ROI**: ~7.6% annually on 10k BBT stake (@$1 BBT)

### USDC Bridge Fees

**Deposit Fees**:
- Ethereum → BlueBlocks: 0.1% (min $1, max $100)
- Covers Ethereum gas costs

**Withdrawal Fees**:
- BlueBlocks → Ethereum: 0.2% (min $2, max $200)
- Covers multi-sig gas costs + validator compensation

**Fee Distribution**:
- 50% to bridge validators
- 30% to treasury
- 20% burned (deflationary)

---

## Governance

### Validator Voting

**Voting Power**: `sqrt(staked_amount) × uptime_percentage`

**Proposal Types**:
1. **Parameter Changes**: Block time, difficulty, fees
2. **Protocol Upgrades**: New features, consensus changes
3. **Validator Ejection**: Remove malicious validator
4. **Treasury Spending**: Foundation budget allocation

**Voting Process**:
```
1. Proposal created (requires 1% of total stake)
2. Discussion period (7 days)
3. Voting period (14 days)
4. Execution delay (3 days)
5. Automatic execution if passed (>66% approval)
```

---

## Monitoring & Observability

### Validator Metrics

- **Uptime**: % of blocks signed in last 24 hours
- **Performance**: Average block signing time
- **Slashing Events**: Total penalties incurred
- **Rewards**: BBT earned from validation

### Bridge Metrics

- **Total Value Locked (TVL)**: USDC deposited
- **Daily Volume**: Deposits + withdrawals
- **Average Confirmation Time**: Deposit → mint latency
- **Failed Transactions**: Rejected signatures

### Dashboard

Web UI at `https://validators.blueblocks.health`:
- Real-time validator map (geographic)
- Live bridge statistics
- Recent transactions
- Governance proposals
- Validator leaderboard

---

## CLI Commands

### Validator Management

```bash
# Register as validator
bblks validator register --stake 10000 --commission 5%

# Start validator node
bblks validator start --port 7777

# Check validator status
bblks validator status

# Claim validation rewards
bblks validator claim-rewards

# Unstake and retire
bblks validator unstake
```

### Multi-Sig Wallet

```bash
# Create multi-sig wallet
bblks multisig create --signers addr1,addr2,addr3 --threshold 2

# Propose transaction
bblks multisig propose --wallet 0xABC --to 0xDEF --amount 1000

# Sign proposal
bblks multisig sign --proposal-id 42

# Execute (after threshold reached)
bblks multisig execute --proposal-id 42

# List pending proposals
bblks multisig list-pending
```

### USDC Bridge

```bash
# Deposit USDC (Ethereum → BlueBlocks)
bblks bridge deposit --amount 1000 --ethereum-tx 0xABC123

# Withdraw USDC (BlueBlocks → Ethereum)
bblks bridge withdraw --amount 1000 --ethereum-address 0xDEF456

# Check bridge status
bblks bridge status

# View pending withdrawals
bblks bridge list-withdrawals
```

---

## Next Steps

1. ✅ Design validator & USDC architecture (this document)
2. ⏳ Implement validator registry (Phase 1, Week 1)
3. ⏳ Implement multi-sig wallet (Phase 2, Week 4)
4. ⏳ Deploy Ethereum multi-sig (Gnosis Safe)
5. ⏳ Security audit & testing
6. ⏳ Mainnet deployment

---

## References

- **Ethereum Multi-Sig**: https://gnosis-safe.io/
- **USDC Documentation**: https://www.circle.com/en/usdc
- **Ed25519 Signatures**: https://ed25519.cr.yp.to/
- **Proof-of-Stake**: https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/
