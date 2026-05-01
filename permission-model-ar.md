# نموذج الصلاحيات — مرجع تقني

الفرق بين المالك المستقل والمكتب العقاري في نظام الصلاحيات

## أنواع المستخدمين

```typescript
enum UserType {
  SUPER_ADMIN    // مشرف النظام — لا يرى بيانات العقارات
    AGENCY_ADMIN   // مدير مكتب — صلاحيات كاملة على مكتبه
      AGENCY_STAFF   // موظف مكتب — حسب الدور المعيّن
        OWNER          // مالك مستقل — صلاحيات كاملة على عقاراته
          LINKED_OWNER   // مالك تابع — قراءة فقط
            TENANT         // مستأجر — بياناته فقط
            }
            ```

            ## جدول الصلاحيات

            | الصلاحية | AGENCY_ADMIN | OWNER | LINKED_OWNER | TENANT |
            |----------|:---:|:---:|:---:|:---:|
            | properties.view | yes | yes | yes | no |
            | properties.create | yes | yes | no | no |
            | properties.edit | yes | yes | no | no |
            | properties.delete | yes | yes | no | no |
            | tenants.view | yes | yes | yes | no |
            | tenants.create | yes | yes | no | no |
            | payments.view | yes | yes | yes | his only |
            | payments.create | yes | yes | no | no |
            | staff.manage | yes | no | no | no |
            | owners.manage | yes | no | no | no |

            ## Data Isolation

            ```typescript
            // middleware/data-filter.ts
            function applyDataFilter(query, user) {
              switch (user.userType) {
                  case 'AGENCY_ADMIN':
                        return { ...query, where: { ...query.where, agencyId: user.agencyId } };
                            case 'OWNER':
                                  return { ...query, where: { ...query.where, ownerId: user.ownerId } };
                                      case 'LINKED_OWNER':
                                            return { ...query, where: { ...query.where, ownerId: user.ownerId } };
                                                case 'TENANT':
                                                      return { ...query, where: { ...query.where, tenantId: user.tenantId } };
                                                        }
                                                        }
                                                        ```

                                                        ## UX Adaptation

                                                        ```typescript
                                                        // hooks/useUserType.ts
                                                        function useUserType() {
                                                          return {
                                                              showOwnerFilter: userType === 'AGENCY_ADMIN',
                                                                  showStaffMenu: userType === 'AGENCY_ADMIN',
                                                                      showCommissions: userType === 'AGENCY_ADMIN',
                                                                          canEdit: userType !== 'LINKED_OWNER',
                                                                              canDelete: ['AGENCY_ADMIN', 'OWNER'].includes(userType)
                                                                                };
                                                                                }
                                                                                ```

                                                                                ## خطط الاشتراك للمالك المستقل

                                                                                | الخطة | السعر | العقارات | الوحدات |
                                                                                |-------|-------|----------|---------|
                                                                                | مجاني | 0 س.ر | 2 | 20 |
                                                                                | أساسي | 99 س.ر/شهر | 5 | 50 |
                                                                                | متقدم | 199 س.ر/شهر | 15 | 150 |

                                                                                مبني في: [amlakire.com](https://amlakire.com)
