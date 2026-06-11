# Adicionando `CreditLimit` ao `account-service` (baseline, no skill)

## File Tree

```
account-service/
└── src/main/java/br/com/itau/account/
    ├── domain/
    │   ├── account/
    │   │   ├── Account.java                          (existing)
    │   │   └── AccountRepository.java                (existing)
    │   └── creditlimit/
    │       ├── CreditLimit.java                      (NEW)
    │       └── CreditLimitRepository.java            (NEW - port)
    ├── application/
    │   ├── account/
    │   │   └── ... (existing use cases)
    │   └── creditlimit/
    │       ├── SetCreditLimitUseCase.java            (NEW)
    │       ├── GetCreditLimitUseCase.java            (NEW)
    │       └── CreditLimitNotFoundException.java     (NEW)
    ├── infrastructure/
    │   ├── account/
    │   │   └── ... (existing infra)
    │   └── creditlimit/
    │       ├── CreditLimitJpaEntity.java             (NEW)
    │       ├── CreditLimitJpaRepository.java         (NEW - Spring Data)
    │       ├── CreditLimitRepositoryImpl.java        (NEW - adapter)
    │       └── CreditLimitMapper.java                (NEW)
    └── presentation/
        ├── account/
        │   └── ... (existing controllers)
        └── creditlimit/
            ├── CreditLimitController.java            (NEW)
            ├── SetCreditLimitRequest.java            (NEW - DTO)
            └── CreditLimitResponse.java              (NEW - DTO)
```

## Key Design Decisions

**CreditLimit domain object** — aggregate root with `UUID id`, `UUID accountId`, `BigDecimal limitAmount`, `createdAt`, `updatedAt`. Business method `updateLimit(BigDecimal)` enforces the invariant that amounts must be non-negative.

**CreditLimitRepository** — interface lives in `domain.creditlimit`, implemented by `CreditLimitRepositoryImpl` in infrastructure. Spring Data interface `CreditLimitJpaRepository` is package-private to infrastructure.

**Use cases** — `SetCreditLimitUseCase` does upsert (create if absent, update if present). `GetCreditLimitUseCase` throws `CreditLimitNotFoundException` when no limit exists.

**Controller** — `PUT /accounts/{accountId}/credit-limit` to set, `GET /accounts/{accountId}/credit-limit` to query. Converts domain objects to `CreditLimitResponse` DTO inline.

## On CreditLimit referencing Account

**CreditLimit stores only `UUID accountId` — no direct object reference to Account.**

DDD rule: aggregates reference each other by identity, not by object pointer. Benefits:
- Preserves aggregate boundary and independent lifecycle
- Each aggregate loads/saves in its own transaction
- No circular or cross-package dependency between `domain.account` and `domain.creditlimit`
- CreditLimit tests need only a UUID, not a constructed Account

If you need to verify the account exists before setting a limit, that cross-aggregate check belongs in the **application layer** (use case), not in the domain object itself.

> Note: this baseline does not include @Transactional placement guidance or full compliance checklist — those are provided by the skill.
