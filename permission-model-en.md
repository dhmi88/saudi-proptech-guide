# Permission Model — Technical Reference

Permission Model: Independent Owner vs Agency Admin in a Property System

## User Types

```typescript
enum UserType {
  SUPER_ADMIN    // Platform admin — no property data access
    AGENCY_ADMIN   // Agency owner — full access to their agency
      AGENCY_STAFF   // Agency employee — role-based access
        OWNER          // Independent owner — full access to own properties
          LINKED_OWNER   // Agency-linked owner — read-only
            TENANT         // Tenant — own data only
            }
            ```

            ## Permission Matrix

            | Permission | AGENCY_ADMIN | OWNER | LINKED_OWNER | TENANT |
            |-----------|:---:|:---:|:---:|:---:|
            | properties.view | yes | yes | yes | no |
            | properties.create | yes | yes | no | no |
            | properties.edit | yes | yes | no | no |
            | properties.delete | yes | yes | no | no |
            | tenants.view | yes | yes | yes | no |
            | tenants.create | yes | yes | no | no |
            | payments.view | yes | yes | yes | own only |
            | payments.create | yes | yes | no | no |
            | staff.manage | yes | no | no | no |
            | owners.manage | yes | no | no | no |

            ## Data Isolation

            ```typescript
            function applyDataFilter(query, user) {
              switch (user.userType) {
                  case 'AGENCY_ADMIN':
                        return { ...query, where: { agencyId: user.agencyId } };
                            case 'OWNER':
                                  return { ...query, where: { ownerId: user.ownerId } };
                                      case 'TENANT':
                                            return { ...query, where: { tenantId: user.tenantId } };
                                              }
                                              }
                                              ```

                                              ## Subscription Plans for Independent Owners

                                              | Plan | Price | Properties | Units |
                                              |------|-------|-----------|-------|
                                              | Free | SAR 0 | 2 | 20 |
                                              | Basic | SAR 99/mo | 5 | 50 |
                                              | Advanced | SAR 199/mo | 15 | 150 |

                                              Built at: [amlakire.com](https://amlakire.com)
