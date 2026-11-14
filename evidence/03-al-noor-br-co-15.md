# Evidence: Incident 3 — Al Noor Fuel & Lubricants (BR-CO-15)

## 1. Customer Report & Symptoms
Customer reported that several invoices from `samples/al-noor-fuel-nov-2026.xlsx` failed with:
`BR-CO-15: Invoice total amount with VAT (IBT-112) must equal Invoice total amount without VAT (IBT-109) + Invoice total VAT amount (IBT-110).`

Replay output:
```
[12:13:56] al-noor  invoices: Accepted 3, Rejected 3
```
3 invoices were accepted; 3 were rejected by the ASP mock.

## 2. Peppol Rule Definition
In `services/asp-mock/main.go`:
```go
// Lines 225-230:
expectedInclusive := cents(t.TaxExclusiveAmount) + cents(t.TaxAmount)
if cents(t.TaxInclusiveAmount) != expectedInclusive {
    errs = append(errs, ruleError{"BR-CO-15", fmt.Sprintf(
        "Invoice total amount with VAT (IBT-112) must equal Invoice total amount without VAT (IBT-109) + Invoice total VAT amount (IBT-110). Got %s, expected %s.",
        money(cents(t.TaxInclusiveAmount)), money(expectedInclusive))})
}
```

## 3. Code Evidence: Discrepant Calculation in API Mapper
In `services/api/internal/pint/pint.go`:
```go
// Line 171:
gross += l.Quantity * l.UnitPrice * (1 + l.VATRate/100)

// Lines 181-186:
return subtotals, Totals{
    LineExtensionAmount: round2(net),
    TaxExclusiveAmount:  round2(net),
    TaxAmount:           round2(vat),
    TaxInclusiveAmount:  round2(gross), // <--- BUG!
    PayableAmount:       round2(gross),
}
```
Instead of setting `TaxInclusiveAmount = round2(net + vat)` as required by Peppol specification, the code re-calculates `gross` by multiplying `Quantity * UnitPrice * (1 + VATRate/100)`.
For fuel products where unit prices contain 3 decimal places (e.g. 2.945 AED/litre), rounding `gross` separately results in a 1-cent discrepancy ($0.01$ AED) compared to `TaxExclusiveAmount + TaxAmount`, triggering rule `BR-CO-15`.
