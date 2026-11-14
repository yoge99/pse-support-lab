# Evidence: Incident 2 — API Container Restarts & In-Flight Submissions

## 1. Grafana Alert Firing
From `inbox/02-alert-api-restarted-marina-bay.md`:
```
alertname = InvoicelyApiRestarted
service = api
severity = critical
description = The api container has restarted more than twice within 15 minutes.
Value: restarts=3
```

## 2. Code Evidence: Memory Leak in SourceCache
In `services/api/internal/importer/sources.go`:
```go
// Line 22-26:
type SourceCache struct {
    mu      sync.RWMutex
    entries map[string]*Source
}

// Line 31-36: Unbounded in-memory map without TTL, eviction, or maximum size
func (c *SourceCache) Put(importID string, s *Source) {
    s.StoredAt = time.Now()
    c.mu.Lock()
    c.entries[importID] = s
    c.mu.Unlock()
}
```
Every uploaded file (`Raw []byte`, `Workbook`, and parsed `Invoices`) is permanently retained in RAM. During peak morning traffic (240 uploads from 4 customers), memory grows linearly until hitting container limits.

## 3. Code Evidence: Hard Container Memory Limit
In `docker-compose.yml`:
```yaml
// Lines 60-61:
mem_limit: 128m
memswap_limit: 128m
```
When memory exceeds 128MB, the Linux kernel OOM killer sends `SIGKILL` (exit code 137). Docker restarts the container due to `restart: unless-stopped`.

## 4. Code Evidence: In-Memory Submission Queue Loss
In `services/api/internal/submit/submit.go`:
```go
// Line 65:
queue: make(chan string, 100000)

// Line 69-74:
func (s *Submitter) Enqueue(ids ...string) {
    for _, id := range ids {
        metrics.QueueDepth.Inc()
        s.queue <- id
    }
}
```
Invoices are enqueued into an in-memory Go channel. When the API container is terminated, any invoice IDs waiting in `s.queue` are destroyed.
Because `main.go` has no startup scan to re-enqueue unsubmitted invoices with `status = 'Validated'`, those invoices remain orphaned in `Validated` status indefinitely.
