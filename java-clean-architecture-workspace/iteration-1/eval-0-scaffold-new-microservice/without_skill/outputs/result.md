# loan-service — Clean Architecture Structure (baseline, no skill)

## Package Structure Tree

```
loan-service/
└── src/main/java/br/com/itau/loan/
    ├── LoanServiceApplication.java
    ├── domain/
    │   ├── Loan.java
    │   └── LoanStatus.java
    ├── application/
    │   ├── port/
    │   │   ├── in/
    │   │   │   ├── RequestLoanUseCase.java
    │   │   │   ├── ApproveLoanUseCase.java
    │   │   │   └── CancelLoanUseCase.java
    │   │   └── out/
    │   │       └── LoanRepository.java
    │   ├── usecase/
    │   │   ├── RequestLoanUseCaseImpl.java
    │   │   ├── ApproveLoanUseCaseImpl.java
    │   │   └── CancelLoanUseCaseImpl.java
    │   └── dto/
    │       ├── RequestLoanCommand.java
    │       ├── ApproveLoanCommand.java
    │       └── CancelLoanCommand.java
    └── infrastructure/
        ├── persistence/
        │   ├── entity/LoanJpaEntity.java
        │   ├── mapper/LoanMapper.java
        │   └── repository/
        │       ├── LoanJpaRepository.java
        │       └── LoanRepositoryImpl.java
        └── web/
            ├── controller/LoanController.java
            └── dto/
                ├── RequestLoanRequest.java
                ├── ApproveLoanRequest.java
                ├── CancelLoanRequest.java
                └── LoanResponse.java
```

## Key Design Decisions

- `Loan` domain object is pure Java (no Spring/JPA). Business rules (`approve()`, `cancel()`) live here and throw `IllegalStateException` on invalid transitions.
- `LoanRepository` in `application.port.out` is the outbound port — domain and application layers never import JPA.
- `LoanJpaEntity` in `infrastructure.persistence.entity` carries all `@Entity`/`@Table` annotations.
- `LoanMapper` converts between domain object and JPA entity.
- `LoanController` depends only on inbound-port interfaces.
- Request/response DTOs stay exclusively in `infrastructure.web.dto`.

> Note: presentation layer placed inside `infrastructure/web/` (baseline generic approach — skill defines dedicated `presentation/` layer separate from infrastructure).
