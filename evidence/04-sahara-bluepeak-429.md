# Evidence: Incident 4 — Sahara Medical / BluePeak ERP

## 1. Symptoms & Traffic Replay
Batch of 40 invoices pushed by ERP connector.
`loadgen load` output:
```
[12:13:58] sahara HTTP 201 in 12.6s import=85bd9356-a13d-465f-8a26-08d16d91519f status=Completed invoices=40
[12:14:06] sahara invoices: Submitted 32, Accepted 8
```
8 invoices were Accepted; 32 remained stuck in `Submitted` status with no ASP reference.

## 2. Code Evidence: ASP Fair-Use Rate Limiting
In `services/asp-mock/main.go`:
```go
// Lines 36-37:
senderBurst      = 8
senderRatePerSec = 2.0

// Lines 274-284:
if ok, wait := s.limiter.allow(sender, time.Now()); !ok {
    retryAfter := int(math.Ceil(wait.Seconds()))
    w.Header().Set("Retry-After", strconv.Itoa(retryAfter))
    s.log.Warn("rate limit exceeded", "sender", sender, "retry_after_s", retryAfter)
    documentsTotal.WithLabelValues("throttled").Inc()
    writeJSON(w, http.StatusTooManyRequests, map[string]string{
        "error":   "rate_limited",
        "message": fmt.Sprintf("Too many documents from sender %s. Retry after %ds.", sender, retryAfter),
    })
    return
}
```

## 3. Code Evidence: Missing Status Handler in Submitter
In `services/api/internal/submit/submit.go`:
```go
// Line 114: status set to Submitted prior to HTTP post
_ = s.store.SetStatus(ctx, id, store.StatusSubmitted, "Sent to ASP")

// Lines 133-155: Switch statement handles 200, 422, and 5xx, but OMITTED HTTP 429!
switch resp.StatusCode {
case http.StatusOK:
    ...
case http.StatusUnprocessableEntity:
    ...
case http.StatusInternalServerError, http.StatusBadGateway, http.StatusServiceUnavailable, http.StatusGatewayTimeout:
    ...
}
```
When HTTP 429 is returned by the ASP, the response is unhandled. The invoice remains stuck in `Submitted` status with no retry and no ASP reference.
