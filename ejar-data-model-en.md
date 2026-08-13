# Ejar Data Model — Technical Reference

An open reference for anyone building a property management system that touches Saudi Arabia's Ejar platform: the fields it requires, their formats, a suggested data model, and ready-to-use validation functions.

Everything here is either grounded in an official source, or explicitly marked as an engineering suggestion.

## Regulatory basis

| Item | Reference |
|---|---|
| Mandatory registration of residential and commercial leases | Cabinet Resolution No. 405, dated 22 Ramadan 1437 AH |
| Supervising authority | Real Estate General Authority (REGA) |
| Platform | Ejar — ejar.sa |
| Login | Exclusively via Nafath (National Single Sign-On) |
| Brokerage commission | 2.5% of first-year rent, unless the parties agree otherwise in writing |
| Hotline | 199011 |

**"Digital Integration between the Ejar Network and Real Estate Platforms"** is an officially announced service; Ejar has concluded 12 digital integration agreements with real estate platforms.

**Registration journey via integration:** register property and unit data in the platform, register the contract, then automatic submission to the Ejar network for the parties to authenticate.

## Government verification — why it matters technically

Ejar is electronically integrated with government bodies for verification:

**Integration complete:** National Information Center, Ministry of Commerce, Ministry of Justice, Ministry of Interior.

**Integration in progress:** Saudi Post, National Water Company, Saudi Electricity Company, Ministry of Labor and Social Development.

**Engineering implication:** party identities are verified against their official source. A non-matching number is rejected at registration, not at display time. So any local validation layer exists to **reduce rejection cycles**, not to replace government verification.

That distinction should shape your design: separate what you can prove locally (length, format, date logic) from what you can't (that this number belongs to this person).

## Required fields

The minimum set that must be complete before registration begins:

### Landlord

| Field | Note |
|---|---|
| Full name | Matching the ID |
| National ID / Iqama | 10 digits |
| Phone | A valid, verifiable format |
| Birth date | Required |
| IBAN | SA + 22 digits, for rent transfers |

### Tenant

| Field | Note |
|---|---|
| Full name | Matching the ID |
| National ID / Iqama | 10 digits |
| Phone | Valid format |
| Birth date | Required |
| Nationality | Required |

### Property and unit

| Field | Note |
|---|---|
| Deed number or building permit | Required |
| Address / national address | Required |
| Usage type | Residential or commercial |

### Contract

| Field | Note |
|---|---|
| Start date | Required |
| End date | Must fall after the start date |
| Rent amount | Required |
| Payment cycle | Monthly / bi-monthly / quarterly / every 4 months / semi-annual / annual |

## Suggested data model

A registration request is **not an attribute of the contract** — it is a transaction with a lifecycle, money, and an owner. So it deserves its own model:

```prisma
model EjarContractRequest {
  id String @id @default(uuid())

    // One-to-one with the contract
      contractId String   @unique
        contract   Contract @relation(fields: [contractId], references: [id])

          agencyId      String
            requestedById String

              propertyUsageType     PropertyUsageType
                contractDurationYears Int

                  // Fees — Decimal, not Float
                    ejarFee    Decimal @db.Decimal(20, 8)
                      brokerFee  Decimal @db.Decimal(20, 8)
                        serviceFee Decimal @default(0) @db.Decimal(20, 8)
                          totalFee   Decimal @db.Decimal(20, 8)

                            // What comes back from Ejar after registration
                              ejarContractNumber String?
                                ejarContractPdf    String?

                                  status          EjarRequestStatus @default(PENDING)
                                    adminNotes      String?
                                      rejectionReason String?

                                        // Linked financial trail
                                          walletTransactionId String?
                                            refundTransactionId String?

                                              completedAt DateTime?
                                                rejectedAt  DateTime?
                                                  createdAt   DateTime  @default(now())
                                                    updatedAt   DateTime  @updatedAt

                                                      @@index([agencyId])
                                                        @@index([status])
                                                          @@map("ejar_contract_requests")
                                                          }

                                                          enum EjarRequestStatus {
                                                            PENDING
                                                              IN_PROGRESS
                                                                COMPLETED
                                                                  REJECTED
                                                                  }

                                                                  enum PropertyUsageType {
                                                                    RESIDENTIAL
                                                                      COMMERCIAL
                                                                      }
                                                                      ```

                                                                      **Design decisions in this model:**

                                                                      `contractId @unique` prevents two requests per contract at the database level, not in application code.

                                                                      `Decimal(20, 8)` — money never goes in `Float`. Rounding errors accumulate and surface at the first reconciliation.

                                                                      `walletTransactionId` and `refundTransactionId` tie the request to both money movements, making "where did this amount go?" a single query.

                                                                      `rejectionReason` is separate from `adminNotes` — the first is shown to the user, the second is internal.

                                                                      ## Validation functions

                                                                      ### Running in production

                                                                      These are the actual functions we run at [Amlaki](https://amlakire.com), shared verbatim between web and mobile:

                                                                      ```typescript
                                                                      /**
                                                                       * Unified phone regex.
                                                                        * Accepts:
                                                                         *   - 05XXXXXXXX        (Saudi local with leading 0)
                                                                          *   - 5XXXXXXXX         (Saudi local without 0)
                                                                           *   - +[1-9]\d{6,14}    (international, E.164)
                                                                            */
                                                                            export const PHONE_REGEX = /^(\+[1-9]\d{6,14}|05\d{8}|5\d{8})$/;

                                                                            export function isValidPhone(phone: string | null | undefined): boolean {
                                                                              if (!phone) return false;
                                                                                return PHONE_REGEX.test(phone.replace(/\s/g, ''));
                                                                                }

                                                                                /** Unified Saudi national ID regex (10 digits). */
                                                                                export const NATIONAL_ID_REGEX = /^\d{10}$/;

                                                                                export function isValidNationalId(id: string | null | undefined): boolean {
                                                                                  if (!id) return false;
                                                                                    return NATIONAL_ID_REGEX.test(id);
                                                                                    }
                                                                                    ```

                                                                                    **Why accept international E.164?** Because a real share of tenants and landlords have non-Saudi numbers. Restricting to `+9665` is theoretically cleaner and practically rejects valid data.

                                                                                    **Why 10 digits with no prefix check?** The "starts with 1 for citizens, 2 for residents" rule is correct, but making it a rejection condition means locally rejecting a number the authority might accept. A readiness layer catches obvious errors; it does not claim to replace government verification.

                                                                                    ### Reference functions (suggested)

                                                                                    The following are **not part of our current system** — they are offered as a reference for anyone wanting a deeper validation layer.

                                                                                    **IBAN validation via MOD-97** (ISO 13616). Many systems check only the length; an IBAN carries a checksum that can be verified arithmetically with no network call:

                                                                                    ```typescript
                                                                                    export function validateIBAN(value: string): { isValid: boolean; error?: string } {
                                                                                      const cleaned = value.replace(/\s/g, '').toUpperCase();

                                                                                        if (!/^SA\d{22}$/.test(cleaned)) {
                                                                                            return { isValid: false, error: 'A Saudi IBAN must be SA + 22 digits' };
                                                                                              }

                                                                                                const rearranged = cleaned.slice(4) + cleaned.slice(0, 4);
                                                                                                  const numeric = rearranged.replace(/[A-Z]/g, (c) => String(c.charCodeAt(0) - 55));

                                                                                                    const remainder = numeric
                                                                                                        .split('')
                                                                                                            .reduce((acc, digit) => (acc * 10 + Number(digit)) % 97, 0);
                                                                                                            
                                                                                                              return remainder === 1
                                                                                                                  ? { isValid: true }
                                                                                                                      : { isValid: false, error: 'Invalid IBAN (MOD-97 check failed)' };
                                                                                                                      }
                                                                                                                      ```
                                                                                                                      
                                                                                                                      **Duration in years** — official pricing treats a partial year as a full year:
                                                                                                                      
                                                                                                                      ```typescript
                                                                                                                      export function calculateDurationYears(startDate: Date, endDate: Date): number {
                                                                                                                        const diffDays = (endDate.getTime() - startDate.getTime()) / (1000 * 60 * 60 * 24);
                                                                                                                          return Math.ceil(diffDays / 365);
                                                                                                                          }
                                                                                                                          ```
                                                                                                                          
                                                                                                                          ## The readiness-check pattern
                                                                                                                          
                                                                                                                          Recommendation: one function returning **named gaps**, not a boolean.
                                                                                                                          
                                                                                                                          ```typescript
                                                                                                                          function validateContractData(contract: any): string[] {
                                                                                                                            const missing: string[] = [];
                                                                                                                              const owner = contract.unit?.property?.owner;
                                                                                                                                const tenant = contract.tenant;
                                                                                                                                  const property = contract.unit?.property;
                                                                                                                                  
                                                                                                                                    if (!owner?.fullName) missing.push('Landlord name');
                                                                                                                                      if (!owner?.nationalId) missing.push('Landlord national ID');
                                                                                                                                        if (!owner?.phone) missing.push('Landlord phone');
                                                                                                                                          if (!owner?.birthDate) missing.push('Landlord birth date');
                                                                                                                                          
                                                                                                                                            if (!tenant?.fullName) missing.push('Tenant name');
                                                                                                                                              if (!tenant?.nationalId) missing.push('Tenant national ID');
                                                                                                                                                if (!tenant?.phone) missing.push('Tenant phone');
                                                                                                                                                  if (!tenant?.birthDate) missing.push('Tenant birth date');
                                                                                                                                                    if (!tenant?.nationality) missing.push('Tenant nationality');
                                                                                                                                                    
                                                                                                                                                      if (!property?.deedNumber) missing.push('Deed number / building permit');
                                                                                                                                                        if (!property?.address) missing.push('Property address');
                                                                                                                                                        
                                                                                                                                                          if (!contract.startDate) missing.push('Contract start date');
                                                                                                                                                            if (!contract.endDate) missing.push('Contract end date');
                                                                                                                                                            
                                                                                                                                                              return missing;
                                                                                                                                                              }
                                                                                                                                                              ```
                                                                                                                                                              
                                                                                                                                                              **Why a string array?** Because the UI needs to show *what* is missing. "Data incomplete" alone makes the user guess; a list makes them act.
                                                                                                                                                              
                                                                                                                                                              **Why optional chaining at every level?** A contract without a unit, a unit without a property, a property without a landlord — all exist in real data. Optional chaining returns "missing" instead of throwing a TypeError.
                                                                                                                                                              
                                                                                                                                                              ## Engineering rules
                                                                                                                                                              
                                                                                                                                                              1. Two validation layers: one client-side for responsiveness, one server-side as the authority. The first alone can be bypassed with a direct HTTP call.
                                                                                                                                                              2. The validation gate precedes any operation that costs money. Do not charge and then discover the gap.
                                                                                                                                                              3. Validation formats live in one shared file across all clients. Two clients with different regexes means data accepted here and rejected there.
                                                                                                                                                              4. Fix gaps through their own entities' routes, not through an Ejar-specific endpoint. The data is not Ejar data — Ejar only exposes that it is missing.
                                                                                                                                                              5. Separate registration state from contract state structurally. A contract that is active internally may have no registration request at all.
                                                                                                                                                              6. The refund is part of the state transition, not a manual step after it. And the rejection reason is mandatory.
                                                                                                                                                              7. Do not claim an integration you have not joined. "Ejar data readiness" is not "integrated with Ejar," and your users will notice the difference.
                                                                                                                                                              
                                                                                                                                                              ## Official sources
                                                                                                                                                              
                                                                                                                                                              - Ejar platform — ejar.sa
                                                                                                                                                              - Real Estate General Authority — rega.gov.sa
                                                                                                                                                              - National Portal — my.gov.sa
                                                                                                                                                              - Login — eservices.ejar.sa/ar/nafath-login
                                                                                                                                                              - Hotline — 199011
                                                                                                                                                              
                                                                                                                                                              ---
                                                                                                                                                              
                                                                                                                                                              *This reference is based on experience building [Amlaki](https://amlakire.com), a Saudi property management system. Amlaki has not yet joined the Ejar digital integration program; what is documented here is the readiness layer that precedes it.*
                                                                                                                                                              
                                                                                                                                                              *Contributions welcome via Issues and Pull Requests.*
                                                                                                                                                              
