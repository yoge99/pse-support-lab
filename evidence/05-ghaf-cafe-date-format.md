# Evidence: Incident 5 — Ghaf Tree Café

## 1. Customer Report
Hessa Al Ketbi reported that all 4 invoices were rejected with:
`[Invoice issue date*] must be a date in YYYY-MM-DD format (got '13/11/2026').`

Replay output:
```
[12:13:55] ghaf invoices: Rejected 4
```

## 2. Code Evidence: Date Parser Implementation
In `services/api/internal/sheet/sheet.go`:
```go
// Line 167:
var isoDate = regexp.MustCompile(`^\d{4}-\d{2}-\d{2}$`)

// Lines 171-186:
func ParseDate(v string) (string, bool) {
    if isoDate.MatchString(v) {
        if _, err := time.Parse("2006-01-02", v); err == nil {
            return v, true
        }
        return "", false
    }
    if serial, err := strconv.ParseFloat(v, 64); err == nil && serial > 0 && serial < 2958466 {
        t, err := excelize.ExcelDateToTime(serial, false)
        if err != nil {
            return "", false
        }
        return t.Format("2006-01-02"), true
    }
    return "", false
}
```
`ParseDate` only accepts ISO dates (`YYYY-MM-DD`) or Excel internal date serial numbers. Text entered in `DD/MM/YYYY` format is rejected by Rule `AE-BR-04`.
