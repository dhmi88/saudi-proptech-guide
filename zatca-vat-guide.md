# ZATCA VAT & E-Invoicing Guide for Saudi Real Estate

## VAT Rules for Real Estate

| Property Type | VAT Rate | Notes |
|---------------|----------|-------|
| Residential Rent | 0% (Exempt) | Apartments, villas, residential units |
| Commercial Rent | 15% | Shops, offices, warehouses |
| Property Sales | 0% VAT | Subject to 5% RETT instead |
| Mixed-use Building | Split | Residential units exempt, commercial units 15% |

## Important Distinctions

- Residential rent is **exempt**, not **zero-rated**
- Exempt = no input tax recovery
- Zero-rated = input tax recovery allowed

## VAT Registration Thresholds

| Threshold | Requirement |
|-----------|-------------|
| > SAR 375,000 annual taxable revenue | Mandatory registration |
| SAR 187,500 - 375,000 | Optional registration |
| < SAR 187,500 | No registration required |

## ZATCA E-Invoicing (FATOORA)

### Phase 1 - Generation
- Issue electronic invoices
- Store locally
- QR code required

### Phase 2 - Integration
- Connect with ZATCA via API
- Real-time validation
- Clearance before sending to buyer

## Implementation in Amlaki

```typescript
interface Unit {
  id: string;
    type: 'RESIDENTIAL' | 'COMMERCIAL';
      monthlyRent: number;
      }

      function calculateVAT(unit: Unit): { rate: number; amount: number } {
        if (unit.type === 'RESIDENTIAL') {
            return { rate: 0, amount: 0 };
              }
                const rate = 0.15;
                  return { rate, amount: unit.monthlyRent * rate };
                  }
                  ```

                  ## Penalties

                  | Violation | Penalty |
                  |-----------|---------|
                  | Late VAT filing | 5-25% of tax due |
                  | No e-invoicing | Financial penalties |
                  | Incorrect tax calculation | Fines + back taxes |

                  Built at: [amlakire.com](https://amlakire.com)
