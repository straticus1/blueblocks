# BlueBlocks USDC Treasury System - Simplified Design

## Overview

**Simple automated token sale**: Users send USDC/ETH to smart contract → Receive BBT tokens automatically → USDC/ETH forwarded to Coinbase for conversion.

This is **much simpler** than a full bridge - it's just an automated token sale contract with treasury management.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│              USDC/ETH Token Sale Flow                         │
└──────────────────────────────────────────────────────────────┘

ETHEREUM MAINNET                      BLUEBLOCKS CHAIN
┌─────────────────┐                  ┌─────────────────┐
│  User's Wallet  │                  │  User's BBT     │
│                 │                  │     Wallet      │
│  1000 USDC      │                  │   50,000 BBT    │
└─────────────────┘                  └─────────────────┘
        │                                     ▲
        │ 1. Send USDC/ETH                   │ 3. Receive BBT
        ▼                                     │
┌─────────────────┐                  ┌─────────────────┐
│  Token Sale     │  2. Detect       │  Treasury       │
│  Contract       │─────Event────────→│  Software       │
│  (Ethereum)     │                  │  (BlueBlocks)   │
│                 │                  │                 │
│  - Accept USDC  │                  │  - Mint BBT     │
│  - Accept ETH   │                  │  - Send to user │
│  - Emit event   │                  │  - Track sales  │
└─────────────────┘                  └─────────────────┘
        │
        │ 4. Forward funds
        ▼
┌─────────────────┐
│  Coinbase       │
│  Wallet         │
│                 │
│  - Convert USDC │
│  - Convert ETH  │
│  - Hold USD     │
└─────────────────┘
```

---

## Smart Contract (Ethereum)

### Token Sale Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract BlueblocksTokenSale is Ownable {
    // Constants
    address public constant USDC_ADDRESS = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address public constant COINBASE_DEPOSIT_ADDRESS = 0x...; // Hardcoded Coinbase address

    // Price in cents (e.g., $0.02 = 2 cents)
    uint256 public priceInCents = 2; // $0.02 per BBT

    // Multi-sig for admin (optional, can use single owner)
    address[] public admins;
    uint256 public adminThreshold = 2; // 2-of-3 for price changes

    // Statistics
    uint256 public totalUsdcRaised;
    uint256 public totalEthRaised;
    uint256 public totalBbtSold;

    // Events
    event TokensPurchased(
        address indexed buyer,
        address indexed paymentToken,
        uint256 paymentAmount,
        uint256 bbtAmount,
        string blueblocksAddress
    );

    event FundsForwarded(
        address indexed paymentToken,
        uint256 amount
    );

    event PriceUpdated(
        uint256 oldPrice,
        uint256 newPrice
    );

    constructor(address[] memory _admins) {
        require(_admins.length >= 3, "Need at least 3 admins");
        admins = _admins;
    }

    /**
     * Buy BBT with USDC
     * @param usdcAmount Amount of USDC to spend (6 decimals)
     * @param blueblocksAddress User's BlueBlocks address to receive BBT
     */
    function buyWithUSDC(uint256 usdcAmount, string memory blueblocksAddress) external {
        require(usdcAmount > 0, "Amount must be > 0");
        require(bytes(blueblocksAddress).length > 0, "Invalid BlueBlocks address");

        IERC20 usdc = IERC20(USDC_ADDRESS);

        // Transfer USDC from user to this contract
        require(
            usdc.transferFrom(msg.sender, address(this), usdcAmount),
            "USDC transfer failed"
        );

        // Calculate BBT amount
        // USDC has 6 decimals, price is in cents
        // Example: 1000 USDC (1,000,000,000) / 2 cents = 50,000 BBT
        uint256 usdcInCents = usdcAmount / 1e4; // Convert 6 decimals to cents
        uint256 bbtAmount = (usdcInCents * 100) / priceInCents;

        // Update stats
        totalUsdcRaised += usdcAmount;
        totalBbtSold += bbtAmount;

        // Emit event for treasury software to detect
        emit TokensPurchased(
            msg.sender,
            USDC_ADDRESS,
            usdcAmount,
            bbtAmount,
            blueblocksAddress
        );

        // Forward USDC to Coinbase
        _forwardToCoinbase(USDC_ADDRESS, usdcAmount);
    }

    /**
     * Buy BBT with ETH
     * @param blueblocksAddress User's BlueBlocks address to receive BBT
     */
    function buyWithETH(string memory blueblocksAddress) external payable {
        require(msg.value > 0, "Must send ETH");
        require(bytes(blueblocksAddress).length > 0, "Invalid BlueBlocks address");

        // Get ETH/USD price from Chainlink oracle
        uint256 ethPriceInCents = _getEthPriceInCents();

        // Calculate BBT amount
        // Example: 1 ETH = $2000 = 200,000 cents
        // 200,000 cents / 2 cents = 100,000 BBT
        uint256 ethInCents = (msg.value * ethPriceInCents) / 1e18;
        uint256 bbtAmount = (ethInCents * 100) / priceInCents;

        // Update stats
        totalEthRaised += msg.value;
        totalBbtSold += bbtAmount;

        // Emit event
        emit TokensPurchased(
            msg.sender,
            address(0), // 0x0 = ETH
            msg.value,
            bbtAmount,
            blueblocksAddress
        );

        // Forward ETH to Coinbase
        _forwardToCoinbase(address(0), msg.value);
    }

    /**
     * Forward funds to hardcoded Coinbase address
     */
    function _forwardToCoinbase(address token, uint256 amount) internal {
        if (token == address(0)) {
            // Forward ETH
            (bool success, ) = COINBASE_DEPOSIT_ADDRESS.call{value: amount}("");
            require(success, "ETH transfer to Coinbase failed");
        } else {
            // Forward ERC-20 token (USDC)
            IERC20(token).transfer(COINBASE_DEPOSIT_ADDRESS, amount);
        }

        emit FundsForwarded(token, amount);
    }

    /**
     * Get ETH price in cents from Chainlink oracle
     */
    function _getEthPriceInCents() internal view returns (uint256) {
        // Chainlink ETH/USD price feed
        // Address: 0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419 (Ethereum mainnet)
        AggregatorV3Interface priceFeed = AggregatorV3Interface(
            0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419
        );

        (, int256 price, , , ) = priceFeed.latestRoundData();

        // Chainlink returns price with 8 decimals
        // Convert to cents: price / 1e6
        return uint256(price) / 1e6;
    }

    /**
     * Update BBT price (requires multi-sig approval)
     */
    function updatePrice(uint256 newPriceInCents) external onlyOwner {
        require(newPriceInCents > 0, "Price must be > 0");

        uint256 oldPrice = priceInCents;
        priceInCents = newPriceInCents;

        emit PriceUpdated(oldPrice, newPriceInCents);
    }

    /**
     * Emergency pause (stops accepting payments)
     */
    bool public paused = false;

    function pause() external onlyOwner {
        paused = true;
    }

    function unpause() external onlyOwner {
        paused = false;
    }

    modifier whenNotPaused() {
        require(!paused, "Contract is paused");
        _;
    }
}

// Chainlink price feed interface
interface AggregatorV3Interface {
    function latestRoundData()
        external
        view
        returns (
            uint80 roundId,
            int256 answer,
            uint256 startedAt,
            uint256 updatedAt,
            uint80 answeredInRound
        );
}
```

---

## Treasury Software (BlueBlocks)

### Event Listener & BBT Distributor

```go
// lib/treasury/listener.go
package treasury

import (
    "context"
    "fmt"
    "log"
    "math/big"
    "time"

    "github.com/ethereum/go-ethereum"
    "github.com/ethereum/go-ethereum/accounts/abi"
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/ethclient"
)

const (
    TokenSaleContractAddress = "0x..." // Deployed contract address

    // Event signature: TokensPurchased(address,address,uint256,uint256,string)
    TokensPurchasedEvent = "0x..."
)

type PurchaseEvent struct {
    Buyer              common.Address
    PaymentToken       common.Address
    PaymentAmount      *big.Int
    BBTAmount          *big.Int
    BlueblocksAddress  string
    TransactionHash    string
    BlockNumber        uint64
    Timestamp          time.Time
}

type TreasuryListener struct {
    ethClient      *ethclient.Client
    contractABI    abi.ABI
    contractAddr   common.Address
    lastBlock      uint64

    // BlueBlocks connection
    bbClient       *BlueblocksClient
    treasuryKey    *Keypair
}

func NewTreasuryListener(ethRPC string, bbRPC string) (*TreasuryListener, error) {
    // Connect to Ethereum
    ethClient, err := ethclient.Dial(ethRPC)
    if err != nil {
        return nil, fmt.Errorf("failed to connect to Ethereum: %w", err)
    }

    // Load contract ABI
    contractABI, err := abi.JSON(strings.NewReader(TokenSaleABI))
    if err != nil {
        return nil, err
    }

    // Connect to BlueBlocks
    bbClient := NewBlueblocksClient(bbRPC)

    // Load treasury keypair (has authority to mint BBT)
    treasuryKey, err := LoadTreasuryKeypair("./keys/treasury.json")
    if err != nil {
        return nil, err
    }

    return &TreasuryListener{
        ethClient:    ethClient,
        contractABI:  contractABI,
        contractAddr: common.HexToAddress(TokenSaleContractAddress),
        bbClient:     bbClient,
        treasuryKey:  treasuryKey,
    }, nil
}

func (tl *TreasuryListener) Start(ctx context.Context) error {
    log.Println("🚀 Starting treasury listener...")

    // Get current Ethereum block
    header, err := tl.ethClient.HeaderByNumber(ctx, nil)
    if err != nil {
        return err
    }
    tl.lastBlock = header.Number.Uint64()

    log.Printf("📍 Starting from block %d\n", tl.lastBlock)

    // Poll for new events every 12 seconds (Ethereum block time)
    ticker := time.NewTicker(12 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            log.Println("🛑 Treasury listener stopped")
            return nil

        case <-ticker.C:
            if err := tl.processNewBlocks(ctx); err != nil {
                log.Printf("❌ Error processing blocks: %v\n", err)
            }
        }
    }
}

func (tl *TreasuryListener) processNewBlocks(ctx context.Context) error {
    // Get latest block
    header, err := tl.ethClient.HeaderByNumber(ctx, nil)
    if err != nil {
        return err
    }

    latestBlock := header.Number.Uint64()

    if latestBlock <= tl.lastBlock {
        return nil // No new blocks
    }

    // Query logs from lastBlock+1 to latestBlock
    query := ethereum.FilterQuery{
        FromBlock: big.NewInt(int64(tl.lastBlock + 1)),
        ToBlock:   big.NewInt(int64(latestBlock)),
        Addresses: []common.Address{tl.contractAddr},
    }

    logs, err := tl.ethClient.FilterLogs(ctx, query)
    if err != nil {
        return err
    }

    log.Printf("📦 Found %d events in blocks %d-%d\n", len(logs), tl.lastBlock+1, latestBlock)

    // Process each log
    for _, vLog := range logs {
        if err := tl.processLog(ctx, vLog); err != nil {
            log.Printf("❌ Error processing log %s: %v\n", vLog.TxHash.Hex(), err)
            continue
        }
    }

    tl.lastBlock = latestBlock
    return nil
}

func (tl *TreasuryListener) processLog(ctx context.Context, vLog types.Log) error {
    // Parse event
    event, err := tl.parseTokensPurchasedEvent(vLog)
    if err != nil {
        return err
    }

    log.Printf("💰 Purchase detected:\n")
    log.Printf("   Buyer: %s\n", event.Buyer.Hex())
    log.Printf("   Payment: %s (token: %s)\n", event.PaymentAmount.String(), event.PaymentToken.Hex())
    log.Printf("   BBT Amount: %s\n", event.BBTAmount.String())
    log.Printf("   BlueBlocks Address: %s\n", event.BlueblocksAddress)
    log.Printf("   Tx Hash: %s\n", event.TransactionHash)

    // Check if already processed
    if tl.isProcessed(event.TransactionHash) {
        log.Printf("⏭️  Already processed, skipping\n")
        return nil
    }

    // Mint and send BBT tokens
    if err := tl.mintAndSendBBT(ctx, event); err != nil {
        return fmt.Errorf("failed to mint BBT: %w", err)
    }

    // Mark as processed
    tl.markProcessed(event.TransactionHash)

    log.Printf("✅ Successfully sent %s BBT to %s\n\n", event.BBTAmount.String(), event.BlueblocksAddress)

    return nil
}

func (tl *TreasuryListener) mintAndSendBBT(ctx context.Context, event *PurchaseEvent) error {
    // Create transfer transaction on BlueBlocks
    tx := Transaction{
        From:      tl.treasuryKey.Address,
        To:        event.BlueblocksAddress,
        Amount:    event.BBTAmount.Int64(),
        Timestamp: time.Now().Unix(),
        Type:      "token_sale",
        Metadata: map[string]string{
            "ethereum_tx":    event.TransactionHash,
            "payment_token":  event.PaymentToken.Hex(),
            "payment_amount": event.PaymentAmount.String(),
        },
    }

    // Sign transaction
    signature, err := tl.treasuryKey.Sign(tx.Hash())
    if err != nil {
        return err
    }
    tx.Signature = signature

    // Submit to BlueBlocks network
    if err := tl.bbClient.SubmitTransaction(ctx, tx); err != nil {
        return err
    }

    return nil
}

func (tl *TreasuryListener) parseTokensPurchasedEvent(vLog types.Log) (*PurchaseEvent, error) {
    // Decode event data
    event := struct {
        Buyer             common.Address
        PaymentToken      common.Address
        PaymentAmount     *big.Int
        BBTAmount         *big.Int
        BlueblocksAddress string
    }{}

    err := tl.contractABI.UnpackIntoInterface(&event, "TokensPurchased", vLog.Data)
    if err != nil {
        return nil, err
    }

    // Get block timestamp
    block, err := tl.ethClient.BlockByNumber(context.Background(), big.NewInt(int64(vLog.BlockNumber)))
    if err != nil {
        return nil, err
    }

    return &PurchaseEvent{
        Buyer:             event.Buyer,
        PaymentToken:      event.PaymentToken,
        PaymentAmount:     event.PaymentAmount,
        BBTAmount:         event.BBTAmount,
        BlueblocksAddress: event.BlueblocksAddress,
        TransactionHash:   vLog.TxHash.Hex(),
        BlockNumber:       vLog.BlockNumber,
        Timestamp:         time.Unix(int64(block.Time()), 0),
    }, nil
}

func (tl *TreasuryListener) isProcessed(txHash string) bool {
    // Check database/file to see if this transaction was already processed
    // Prevents double-spending if listener restarts
    return false // TODO: Implement
}

func (tl *TreasuryListener) markProcessed(txHash string) {
    // Mark transaction as processed in database/file
    // TODO: Implement
}
```

### Treasury Daemon

```go
// cmd/bblks-treasury/main.go
package main

import (
    "context"
    "flag"
    "fmt"
    "log"
    "os"
    "os/signal"
    "syscall"

    "github.com/blueblocks/preapproved-implementations/lib/treasury"
)

const banner = `
╔══════════════════════════════════════════════════════╗
║                                                      ║
║     ██████╗ ██████╗ ████████╗                       ║
║     ██╔══██╗██╔══██╗╚══██╔══╝                       ║
║     ██████╔╝██████╔╝   ██║                          ║
║     ██╔══██╗██╔══██╗   ██║                          ║
║     ██████╔╝██████╔╝   ██║                          ║
║     ╚═════╝ ╚═════╝    ╚═╝                          ║
║                                                      ║
║     TREASURY                                         ║
║     Token Sale Listener & Distributor                ║
║                                                      ║
╚══════════════════════════════════════════════════════╝
`

func main() {
    ethRPC := flag.String("eth-rpc", "https://mainnet.infura.io/v3/YOUR_KEY", "Ethereum RPC endpoint")
    bbRPC := flag.String("bb-rpc", "http://localhost:8080", "BlueBlocks RPC endpoint")

    flag.Parse()

    fmt.Println(banner)
    log.Printf("🔗 Ethereum RPC: %s\n", *ethRPC)
    log.Printf("🔗 BlueBlocks RPC: %s\n", *bbRPC)
    log.Println()

    // Create treasury listener
    listener, err := treasury.NewTreasuryListener(*ethRPC, *bbRPC)
    if err != nil {
        log.Fatalf("❌ Failed to create listener: %v", err)
    }

    // Handle graceful shutdown
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)

    go func() {
        <-sigChan
        log.Println("\n⚠️  Received shutdown signal")
        cancel()
    }()

    // Start listening
    if err := listener.Start(ctx); err != nil {
        log.Fatalf("❌ Listener error: %v", err)
    }

    log.Println("👋 Treasury listener stopped gracefully")
}
```

---

## Coinbase Integration

### Hardcoded Deposit Addresses

**Why hardcode?**
- Simple and secure
- No API keys needed in smart contract
- Funds go directly to regulated exchange
- Easy to verify on Etherscan

**Setup**:

1. **Create Coinbase Account** (if not exists)
   - Business account for higher limits
   - Complete KYC/AML verification

2. **Get Deposit Addresses**:
   ```
   Coinbase → Portfolio → USDC → Deposit → Copy Address
   Coinbase → Portfolio → ETH → Deposit → Copy Address
   ```

3. **Hardcode in Smart Contract**:
   ```solidity
   address public constant COINBASE_USDC = 0xYourCoinbaseUSDCAddress;
   address public constant COINBASE_ETH  = 0xYourCoinbaseETHAddress;
   ```

4. **Set Up Auto-Conversion** (Optional):
   - Coinbase Pro API
   - Auto-convert USDC/ETH → USD
   - Or manually convert as needed

### Alternative: Coinbase Commerce API

If you want more automation:

```javascript
// Node.js backend for Coinbase Commerce
const commerce = require('coinbase-commerce-node');
const Client = commerce.Client;

Client.init('YOUR_API_KEY');

// Create a charge
const chargeData = {
    name: 'BlueBlocks Tokens',
    description: '50,000 BBT',
    pricing_type: 'fixed_price',
    local_price: {
        amount: '1000.00',
        currency: 'USD'
    }
};

const charge = await Client.resources.Charge.create(chargeData);
console.log(charge.hosted_url); // Payment page URL
```

---

## Multi-Sig for Admin Functions

### Price Updates (2-of-3 Multi-Sig)

Instead of single owner, use Gnosis Safe for admin functions:

```solidity
// Use Gnosis Safe as owner
address public constant GNOSIS_SAFE = 0xYourGnosisSafeAddress;

constructor() {
    transferOwnership(GNOSIS_SAFE);
}

// Now updatePrice() requires 2-of-3 signatures from Gnosis Safe
```

**Gnosis Safe Setup**:
1. Go to https://app.safe.global/
2. Create new Safe
3. Add 3 owner addresses
4. Set threshold to 2-of-3
5. Deploy Safe
6. Transfer contract ownership to Safe

---

## Security Considerations

### Smart Contract Security

1. **Reentrancy Protection**: Use OpenZeppelin's ReentrancyGuard
2. **Integer Overflow**: Solidity 0.8+ has built-in protection
3. **Access Control**: Gnosis Safe for admin functions
4. **Pause Mechanism**: Emergency stop for detected issues
5. **Rate Limiting**: Max purchase per transaction (e.g., $10k)

### Treasury Software Security

1. **Private Key Protection**:
   - Store treasury key in hardware wallet or HSM
   - Or use encrypted file with strong password
   - Never commit keys to git

2. **Idempotency**:
   - Track processed transactions
   - Prevent double-minting if listener restarts

3. **Monitoring**:
   - Alert on large purchases (>$100k)
   - Alert on listener downtime
   - Daily reconciliation report

4. **Backup**:
   - Run 2 treasury listeners (active + standby)
   - Database replication
   - Regular backups of processed transactions

---

## Testing

### Smart Contract Tests (Hardhat)

```javascript
// test/TokenSale.test.js
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("BlueblocksTokenSale", function () {
    let tokenSale;
    let usdc;
    let owner;
    let buyer;

    beforeEach(async function () {
        [owner, buyer] = await ethers.getSigners();

        // Deploy mock USDC
        const USDC = await ethers.getContractFactory("MockUSDC");
        usdc = await USDC.deploy();

        // Deploy token sale
        const TokenSale = await ethers.getContractFactory("BlueblocksTokenSale");
        tokenSale = await TokenSale.deploy([owner.address]);
    });

    it("Should accept USDC and emit event", async function () {
        // Approve USDC
        await usdc.connect(buyer).approve(tokenSale.address, 1000e6);

        // Buy BBT
        const tx = await tokenSale.connect(buyer).buyWithUSDC(
            1000e6, // 1000 USDC
            "bb1234567890abcdef" // BlueBlocks address
        );

        // Check event
        const receipt = await tx.wait();
        const event = receipt.events.find(e => e.event === 'TokensPurchased');

        expect(event.args.buyer).to.equal(buyer.address);
        expect(event.args.paymentAmount).to.equal(1000e6);
        expect(event.args.bbtAmount).to.equal(50000); // 1000 USDC / $0.02 = 50k BBT
    });

    it("Should forward USDC to Coinbase", async function () {
        const coinbaseBalanceBefore = await usdc.balanceOf(COINBASE_ADDRESS);

        await usdc.connect(buyer).approve(tokenSale.address, 1000e6);
        await tokenSale.connect(buyer).buyWithUSDC(1000e6, "bb1234...");

        const coinbaseBalanceAfter = await usdc.balanceOf(COINBASE_ADDRESS);
        expect(coinbaseBalanceAfter.sub(coinbaseBalanceBefore)).to.equal(1000e6);
    });
});
```

### Integration Test

```bash
# 1. Deploy contract to testnet (Goerli)
npx hardhat run scripts/deploy.js --network goerli

# 2. Start treasury listener (testnet)
./bblks-treasury --eth-rpc https://goerli.infura.io/v3/YOUR_KEY

# 3. Buy tokens with test USDC
cast send $TOKEN_SALE_ADDRESS "buyWithUSDC(uint256,string)" \
    1000000000 "bb1234567890abcdef" \
    --rpc-url https://goerli.infura.io/v3/YOUR_KEY

# 4. Verify BBT received on BlueBlocks testnet
./bblks account balance bb1234567890abcdef
```

---

## Deployment Checklist

### Smart Contract Deployment

- [ ] Audit contract code (external audit)
- [ ] Deploy to Ethereum testnet (Goerli)
- [ ] Test with fake USDC and ETH
- [ ] Verify Coinbase deposits work
- [ ] Create Gnosis Safe (2-of-3 multi-sig)
- [ ] Deploy to Ethereum mainnet
- [ ] Verify contract on Etherscan
- [ ] Transfer ownership to Gnosis Safe
- [ ] Announce contract address publicly

### Treasury Software Deployment

- [ ] Generate treasury keypair
- [ ] Fund treasury wallet with BBT tokens
- [ ] Configure Ethereum RPC (Infura/Alchemy)
- [ ] Configure BlueBlocks RPC
- [ ] Test on testnet first
- [ ] Deploy to production server
- [ ] Set up monitoring/alerts
- [ ] Configure backup listener (standby)
- [ ] Test failover

---

## CLI Commands

```bash
# Treasury management
bblks treasury start --eth-rpc https://mainnet.infura.io/v3/KEY
bblks treasury status
bblks treasury stats
bblks treasury processed-txs

# Manual processing (if listener was down)
bblks treasury process-tx --eth-tx 0xABC123...

# Admin functions (requires Gnosis Safe signatures)
bblks treasury update-price --new-price 3  # $0.03 per BBT
bblks treasury pause
bblks treasury unpause
```

---

## Cost Analysis

### Gas Costs (Ethereum)

**Per Purchase**:
- USDC transfer: ~50k gas
- Event emission: ~5k gas
- Forward to Coinbase: ~30k gas
- **Total**: ~85k gas @ 30 gwei = $6-10

**Monthly Operating Costs**:
- 1,000 purchases/month × $8 avg = $8,000/month in gas
- **Solution**: Pass gas costs to buyer (included in 0.1% fee)

### Server Costs

- Treasury listener: AWS t3.small ($15/month)
- BlueBlocks node: AWS t3.medium ($30/month)
- Backup listener: AWS t3.small ($15/month)
- **Total**: ~$60/month

---

## Monitoring Dashboard

Web UI showing:
- Total USDC raised
- Total ETH raised
- Total BBT sold
- Recent purchases (last 100)
- Treasury wallet balance
- Listener status (online/offline)
- Processed transactions count
- Failed transactions (requires manual review)

---

## Next Steps

1. ✅ Design simplified USDC treasury system
2. ⏳ Write Solidity smart contract
3. ⏳ Write treasury listener (Go)
4. ⏳ Deploy to testnet and test
5. ⏳ Security audit
6. ⏳ Deploy to mainnet
