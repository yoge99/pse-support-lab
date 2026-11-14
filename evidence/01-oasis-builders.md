# Evidence: Incident 1 — Oasis Builders Supply

## 1. Traffic Replay Logs
During `loadgen load`, the import for Oasis Builders took over 120 seconds:
```
[12:13:45] oasis  uploading oasis-builders-nov-2026.xlsx (169 KB)
[12:15:47] oasis  HTTP 201 in 121.7s import=a16c9ffc-db10-430b-9710-51372efab318 status=Completed invoices=8
[12:15:55] oasis  invoices: Accepted 8
```

## 2. Code Evidence: N+1 Network Lookups During Row Validation
In `services/api/internal/validate/validate.go`:
```go
// Line 69-71: loops through every row in the invoice
for _, row := range g.Rows {
    errs = append(errs, v.validateRow(ctx, row)...)
}

// Line 165-173: inside validateRow, queries the ASP directory per row
if buyer := r.Get(sheet.ColBuyerPeppolID); buyer != "" {
    registered, err := v.dir.IsRegistered(ctx, buyer)
    ...
}
```

In `services/asp-mock/main.go`:
```go
// Line 40: artificial latency mimicking real SMP lookup
lookupLatency = 200 * time.Millisecond
```
Calculation:
`600 rows × 200 ms = 120,000 ms = 120 seconds`.

## 3. Code Evidence: Web Reverse Proxy Timeout
In `web/app/api/[...path]/route.ts`:
```typescript
// Line 6:
const UPSTREAM_TIMEOUT_MS = 60_000;

// Line 25:
signal: AbortSignal.timeout(UPSTREAM_TIMEOUT_MS)

// Line 42-53:
const timedOut = err instanceof Error && (err.name === 'TimeoutError' || err.name === 'AbortError');
const status = timedOut ? 504 : 502;
return Response.json({ error: timedOut ? 'Gateway Timeout' : 'Bad Gateway' }, { status });
```
Because the backend takes 121.7s, the client receives a 504 Gateway Timeout after 60s, despite the backend continuing and succeeding.

## 4. Code Evidence: False Duplicate Error on Retry
In `services/api/internal/validate/validate.go`:
```go
// Line 75-81:
status, exists, err := v.invoices.ExistingStatus(ctx, seller.ID, g.InvoiceNumber, importID)
if err != nil {
    v.log.Error("duplicate check failed", "invoice_number", g.InvoiceNumber, "error", err)
} else if exists {
    errs = append(errs, Error{"AE-BR-09", sheet.ColInvoiceNumber,
        fmt.Sprintf("Document number '%s' already exists for this seller with status '%s'.", g.InvoiceNumber, status), first.Number})
}
```
When Rashid retried after the 504 error, the background job had already inserted the first batch into PostgreSQL. Subsequent uploads were rejected with `AE-BR-09`.
