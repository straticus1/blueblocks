# Blockchain Sync Design - Complete Guide

**Date**: 2026-01-14
**Status**: DESIGN DOCUMENT

---

## Question: How to Sync Blockchain for New Nodes?

### Options Compared

| Method | Speed | Memory | Resume | Bandwidth | Complexity |
|--------|-------|--------|--------|-----------|------------|
| **Raw JSON** | Slow | High | No | High | Low |
| **Gzip JSON** | Medium | High | No | Medium | Low |
| **HTTP/2 Stream** | Fast | Low | Yes | Medium | Medium |
| **Snapshot** | Fastest | Low | N/A | Low | High |

---

## Recommended: Multi-Method Sync

Implement **all three methods** and let nodes choose based on their needs:

### 1. **Stream Sync** (Default - Best for Most Cases)

**How it works**:
```
Client → Server: GET /sync/stream?from_height=0
Server → Client: HTTP/2 Server-Sent Events (SSE)

For each block:
  1. Server gzips block JSON
  2. Sends as SSE event
  3. Client receives, decompresses, validates
  4. Client applies block
  5. Client checkpoints every 1000 blocks

If disconnected:
  - Client resumes from last checkpoint
  - GET /sync/stream?from_height=5000
```

**Advantages**:
✅ Memory efficient (process block-by-block)
✅ Resumable (checkpoint every N blocks)
✅ Progressive (see sync progress)
✅ Gzip compression (saves bandwidth)
✅ HTTP/2 multiplexing

**Disadvantages**:
❌ Slightly more complex than JSON
❌ Requires HTTP/2 support

**Use When**:
- New node joining network
- Node recovering from being offline
- Default for most sync operations

### 2. **Snapshot Sync** (Fastest - For Quick Bootstrap)

**How it works**:
```
Client → Server: GET /sync/snapshot?type=recent
Server:
  1. Create snapshot of last 10,000 blocks
  2. Include state (accounts, contracts)
  3. Sign snapshot with node's key
  4. Bzip2 compress (better than gzip for large data)

Client:
  1. Download snapshot (~10-100 MB)
  2. Verify signature
  3. Import snapshot
  4. Sync remaining blocks via stream
```

**Advantages**:
✅ Fastest initial sync
✅ Downloads state + blocks together
✅ Bzip2 compression (better ratio)
✅ Verified by signature

**Disadvantages**:
❌ Trust required (must trust snapshot creator)
❌ Not verifiable from genesis
❌ Larger initial download

**Use When**:
- New node wants fast bootstrap
- Node trusts the network
- Healthcare app needs quick setup

### 3. **JSON Sync** (Simplest - For Small Chains)

**How it works**:
```
Client → Server: GET /sync/json?from=0&to=1000
Server:
  1. Read blocks 0-1000
  2. Marshal to JSON array
  3. Gzip compress
  4. Send with Content-Encoding: gzip

Client:
  1. Download JSON
  2. Decompress
  3. Unmarshal all blocks
  4. Validate and apply
```

**Advantages**:
✅ Simplest implementation
✅ Standard HTTP compression
✅ Easy debugging (can curl it)
✅ No special libraries needed

**Disadvantages**:
❌ High memory usage (loads all blocks)
❌ Not resumable
❌ Slower for large chains

**Use When**:
- Chain is small (< 10,000 blocks)
- Testing/development
- One-time export/import

---

## Recommended Implementation

### Server Side (API Endpoints)

```go
// lib/sync/server.go

package sync

import (
	"compress/gzip"
	"encoding/json"
	"fmt"
	"net/http"
	"strconv"
	"time"
)

type SyncServer struct {
	blockchain *Blockchain
}

// Method 1: Stream Sync (SSE with gzip)
func (s *SyncServer) HandleStreamSync(w http.ResponseWriter, r *http.Request) {
	// Parse query parameters
	fromHeight, _ := strconv.ParseUint(r.URL.Query().Get("from_height"), 10, 64)

	// Setup SSE
	w.Header().Set("Content-Type", "text/event-stream")
	w.Header().Set("Cache-Control", "no-cache")
	w.Header().Set("Connection", "keep-alive")
	w.Header().Set("X-Accel-Buffering", "no") // Disable nginx buffering

	flusher, ok := w.(http.Flusher)
	if !ok {
		http.Error(w, "Streaming not supported", http.StatusInternalServerError)
		return
	}

	// Stream blocks
	chainHeight := s.blockchain.GetHeight()

	for height := fromHeight; height <= chainHeight; height++ {
		block, err := s.blockchain.GetBlockByHeight(height)
		if err != nil {
			fmt.Fprintf(w, "event: error\ndata: {\"error\": \"%v\"}\n\n", err)
			flusher.Flush()
			return
		}

		// Gzip compress block
		blockJSON, _ := json.Marshal(block)
		compressed := gzipCompress(blockJSON)

		// Send as SSE event
		fmt.Fprintf(w, "event: block\n")
		fmt.Fprintf(w, "data: %s\n", base64Encode(compressed))
		fmt.Fprintf(w, "id: %d\n\n", height)

		flusher.Flush()

		// Rate limit (don't overwhelm client)
		time.Sleep(10 * time.Millisecond)
	}

	fmt.Fprintf(w, "event: complete\ndata: {\"height\": %d}\n\n", chainHeight)
	flusher.Flush()
}

// Method 2: JSON Sync (with gzip)
func (s *SyncServer) HandleJSONSync(w http.ResponseWriter, r *http.Request) {
	fromHeight, _ := strconv.ParseUint(r.URL.Query().Get("from"), 10, 64)
	toHeight, _ := strconv.ParseUint(r.URL.Query().Get("to"), 10, 64)

	// Limit range
	if toHeight-fromHeight > 10000 {
		http.Error(w, "Range too large (max 10,000 blocks)", http.StatusBadRequest)
		return
	}

	// Get blocks
	blocks := make([]Block, 0, toHeight-fromHeight)
	for height := fromHeight; height <= toHeight; height++ {
		block, err := s.blockchain.GetBlockByHeight(height)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		blocks = append(blocks, *block)
	}

	// Marshal to JSON
	blocksJSON, err := json.Marshal(blocks)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	// Gzip and send
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("Content-Encoding", "gzip")

	gzipWriter := gzip.NewWriter(w)
	defer gzipWriter.Close()

	gzipWriter.Write(blocksJSON)
}

// Method 3: Snapshot Sync
func (s *SyncServer) HandleSnapshotSync(w http.ResponseWriter, r *http.Request) {
	snapshotType := r.URL.Query().Get("type") // "recent" or "full"

	var snapshot Snapshot
	var err error

	switch snapshotType {
	case "recent":
		// Last 10,000 blocks + state
		snapshot, err = s.createRecentSnapshot(10000)
	case "full":
		// Full blockchain from genesis
		snapshot, err = s.createFullSnapshot()
	default:
		http.Error(w, "Invalid snapshot type", http.StatusBadRequest)
		return
	}

	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	// Sign snapshot
	signature := s.signSnapshot(snapshot)
	snapshot.Signature = signature

	// Bzip2 compress (better compression for large data)
	snapshotJSON, _ := json.Marshal(snapshot)
	compressed := bzip2Compress(snapshotJSON)

	// Send
	w.Header().Set("Content-Type", "application/octet-stream")
	w.Header().Set("Content-Disposition", "attachment; filename=snapshot.bz2")
	w.Header().Set("X-Snapshot-Height", strconv.FormatUint(snapshot.Height, 10))
	w.Header().Set("X-Snapshot-Hash", snapshot.Hash)

	w.Write(compressed)
}

func (s *SyncServer) createRecentSnapshot(blockCount uint64) (Snapshot, error) {
	currentHeight := s.blockchain.GetHeight()
	fromHeight := uint64(0)
	if currentHeight > blockCount {
		fromHeight = currentHeight - blockCount
	}

	// Get blocks
	blocks := make([]Block, 0, blockCount)
	for height := fromHeight; height <= currentHeight; height++ {
		block, _ := s.blockchain.GetBlockByHeight(height)
		blocks = append(blocks, *block)
	}

	// Get state (accounts, contracts)
	accounts, _ := s.blockchain.GetAllAccounts()
	contracts, _ := s.blockchain.GetAllContracts()

	snapshot := Snapshot{
		Height:    currentHeight,
		Blocks:    blocks,
		Accounts:  accounts,
		Contracts: contracts,
		Timestamp: time.Now().Unix(),
	}

	// Calculate hash
	snapshot.Hash = calculateSnapshotHash(snapshot)

	return snapshot, nil
}

type Snapshot struct {
	Height    uint64                 `json:"height"`
	Hash      string                 `json:"hash"`
	Blocks    []Block                `json:"blocks"`
	Accounts  []Account              `json:"accounts"`
	Contracts []Contract             `json:"contracts"`
	Timestamp int64                  `json:"timestamp"`
	Signature string                 `json:"signature"`
}
```

### Client Side (Sync Implementation)

```go
// lib/sync/client.go

package sync

import (
	"bufio"
	"compress/gzip"
	"encoding/base64"
	"encoding/json"
	"fmt"
	"net/http"
	"strings"
	"time"
)

type SyncClient struct {
	nodeURL    string
	blockchain *Blockchain
}

// Method 1: Stream Sync
func (c *SyncClient) SyncStream() error {
	currentHeight := c.blockchain.GetHeight()

	fmt.Printf("🔄 Starting stream sync from height %d\n", currentHeight)

	req, _ := http.NewRequest("GET",
		fmt.Sprintf("%s/sync/stream?from_height=%d", c.nodeURL, currentHeight),
		nil)
	req.Header.Set("Accept", "text/event-stream")

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	if resp.StatusCode != 200 {
		return fmt.Errorf("sync failed: %s", resp.Status)
	}

	// Read SSE stream
	scanner := bufio.NewScanner(resp.Body)
	var eventType string
	var eventData string
	blocksReceived := 0
	lastCheckpoint := time.Now()

	for scanner.Scan() {
		line := scanner.Text()

		if strings.HasPrefix(line, "event: ") {
			eventType = strings.TrimPrefix(line, "event: ")
		} else if strings.HasPrefix(line, "data: ") {
			eventData = strings.TrimPrefix(line, "data: ")
		} else if line == "" {
			// End of event
			if eventType == "block" {
				// Decode and decompress
				compressed, _ := base64.StdEncoding.DecodeString(eventData)
				blockJSON := gzipDecompress(compressed)

				// Unmarshal block
				var block Block
				json.Unmarshal(blockJSON, &block)

				// Validate and apply block
				if err := c.blockchain.AddBlock(block); err != nil {
					return fmt.Errorf("failed to apply block %d: %v", block.Height, err)
				}

				blocksReceived++

				// Progress update
				if blocksReceived%100 == 0 {
					fmt.Printf("   Synced %d blocks (current: %d)\n", blocksReceived, block.Height)
				}

				// Checkpoint every 1000 blocks
				if blocksReceived%1000 == 0 && time.Since(lastCheckpoint) > 5*time.Second {
					c.blockchain.Checkpoint()
					lastCheckpoint = time.Now()
					fmt.Printf("   ✓ Checkpoint at height %d\n", block.Height)
				}
			} else if eventType == "complete" {
				fmt.Printf("✅ Sync complete! Received %d blocks\n", blocksReceived)
				return nil
			} else if eventType == "error" {
				return fmt.Errorf("sync error: %s", eventData)
			}

			// Reset for next event
			eventType = ""
			eventData = ""
		}
	}

	return scanner.Err()
}

// Method 2: JSON Sync (batched)
func (c *SyncClient) SyncJSON() error {
	const batchSize = 1000
	currentHeight := c.blockchain.GetHeight()

	fmt.Printf("🔄 Starting JSON sync from height %d\n", currentHeight)

	for {
		toHeight := currentHeight + batchSize

		url := fmt.Sprintf("%s/sync/json?from=%d&to=%d",
			c.nodeURL, currentHeight, toHeight)

		resp, err := http.Get(url)
		if err != nil {
			return err
		}
		defer resp.Body.Close()

		// Decompress gzip
		gzipReader, _ := gzip.NewReader(resp.Body)
		defer gzipReader.Close()

		// Unmarshal blocks
		var blocks []Block
		if err := json.NewDecoder(gzipReader).Decode(&blocks); err != nil {
			return err
		}

		if len(blocks) == 0 {
			fmt.Println("✅ Sync complete!")
			return nil
		}

		// Apply blocks
		for _, block := range blocks {
			if err := c.blockchain.AddBlock(block); err != nil {
				return fmt.Errorf("failed to apply block %d: %v", block.Height, err)
			}
		}

		fmt.Printf("   Synced blocks %d - %d\n", currentHeight, currentHeight+uint64(len(blocks)))
		currentHeight += uint64(len(blocks))
	}
}

// Method 3: Snapshot Sync
func (c *SyncClient) SyncSnapshot() error {
	fmt.Println("📸 Downloading snapshot...")

	resp, err := http.Get(c.nodeURL + "/sync/snapshot?type=recent")
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	// Get snapshot metadata
	snapshotHeight := resp.Header.Get("X-Snapshot-Height")
	snapshotHash := resp.Header.Get("X-Snapshot-Hash")

	fmt.Printf("   Snapshot height: %s\n", snapshotHeight)
	fmt.Printf("   Snapshot hash: %s\n", snapshotHash)

	// Decompress bzip2
	decompressed := bzip2Decompress(resp.Body)

	// Unmarshal snapshot
	var snapshot Snapshot
	if err := json.Unmarshal(decompressed, &snapshot); err != nil {
		return err
	}

	// Verify signature
	if !c.verifySnapshot(snapshot) {
		return fmt.Errorf("invalid snapshot signature")
	}

	fmt.Println("   ✓ Signature verified")

	// Import snapshot
	fmt.Println("   Importing accounts...")
	for _, account := range snapshot.Accounts {
		c.blockchain.ImportAccount(account)
	}

	fmt.Println("   Importing contracts...")
	for _, contract := range snapshot.Contracts {
		c.blockchain.ImportContract(contract)
	}

	fmt.Println("   Importing blocks...")
	for i, block := range snapshot.Blocks {
		if err := c.blockchain.AddBlock(block); err != nil {
			return err
		}
		if i%1000 == 0 {
			fmt.Printf("   Imported %d / %d blocks\n", i, len(snapshot.Blocks))
		}
	}

	fmt.Printf("✅ Snapshot imported! Now at height %d\n", snapshot.Height)
	fmt.Println("   Syncing remaining blocks...")

	// Sync remaining blocks via stream
	return c.SyncStream()
}
```

---

## Compression Comparison

### For Blocks (JSON data)

| Method | Size | Ratio | Speed |
|--------|------|-------|-------|
| **None** | 1.0 MB | 1.0x | Fastest |
| **Gzip -6** | 0.15 MB | 6.7x | Fast |
| **Bzip2 -9** | 0.12 MB | 8.3x | Slow |
| **Zstd** | 0.14 MB | 7.1x | Very Fast |

**Recommendation**:
- **Stream/JSON**: Gzip (good balance)
- **Snapshot**: Bzip2 (best compression)

---

## Bandwidth Savings Example

**Scenario**: 10,000 blocks, 100 KB each

| Method | Size | Time (10 Mbps) |
|--------|------|----------------|
| Raw JSON | 1.0 GB | 13 minutes |
| Gzip Stream | 150 MB | 2 minutes |
| Bzip2 Snapshot | 120 MB | 1.6 minutes |

**Savings**: 85-88% bandwidth reduction

---

## Production Recommendations

### For Healthcare Blockchain

1. **Default: Stream Sync**
   - Use for all normal sync operations
   - Gzip compression
   - Resume on disconnect
   - Progress tracking

2. **Fast Bootstrap: Snapshot Sync**
   - For new healthcare providers joining
   - Download snapshot + sync remaining
   - Quick setup (< 5 minutes)
   - Trust network operators

3. **Testing: JSON Sync**
   - For development/testing
   - Easy debugging
   - Export/import chains

### Implementation Priority

1. ✅ **Stream Sync** (do this first)
2. ⚠️ **JSON Sync** (simple, do second)
3. 🔄 **Snapshot Sync** (complex, do later)

---

## Summary

**Recommendation**: **Implement Stream Sync with gzip compression**

**Why**:
✅ Memory efficient (100 MB vs 1 GB)
✅ Resumable (survives disconnects)
✅ Fast enough (2-3 minutes for 10K blocks)
✅ Standard HTTP/2 (no special infrastructure)
✅ Progressive (see sync progress)

**Implementation**:
```go
// Server: Send gzip-compressed SSE
// Client: Receive, decompress, validate, apply
// Checkpoint: Every 1000 blocks
// Resume: From last checkpoint
```

This gives you the best balance of performance, reliability, and implementation complexity! 🚀
