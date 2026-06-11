# loan-service — Scaffold Completo com Clean Architecture
## (Using java-clean-architecture skill — br.com.itau.loan)

## Estrutura de Pacotes

```
br/com/itau/loan/
├── domain/
│   └── loan/
│       ├── Loan.java                              # aggregate root
│       ├── LoanId.java                            # value object
│       ├── LoanStatus.java                        # enum de estado
│       ├── LoanRepository.java                    # porta de saída (interface)
│       └── LoanException.java                     # exceções de domínio
│
├── application/
│   ├── usecase/
│   │   └── loan/
│   │       ├── RequestLoanUseCase.java            # porta de entrada
│   │       ├── RequestLoanUseCaseImpl.java
│   │       ├── ApproveLoanUseCase.java
│   │       ├── ApproveLoanUseCaseImpl.java
│   │       ├── CancelLoanUseCase.java
│   │       └── CancelLoanUseCaseImpl.java
│   └── dto/
│       └── loan/
│           ├── RequestLoanInputDto.java
│           ├── RequestLoanOutputDto.java
│           ├── ApproveLoanInputDto.java
│           ├── ApproveLoanOutputDto.java
│           ├── CancelLoanInputDto.java
│           └── CancelLoanOutputDto.java
│
├── infrastructure/
│   ├── persistence/
│   │   └── loan/
│   │       ├── LoanJpaEntity.java                 # @Entity aqui
│   │       ├── LoanJpaRepository.java             # Spring Data interface
│   │       ├── LoanEntityMapper.java              # JpaEntity ↔ domain
│   │       └── LoanRepositoryImpl.java            # implementa LoanRepository
│   └── config/
│       └── PersistenceConfig.java
│
└── presentation/
    └── rest/
        └── loan/
            ├── LoanController.java
            ├── RequestLoanRequest.java
            ├── ApproveLoanRequest.java
            ├── CancelLoanRequest.java
            └── LoanResponse.java
```

---

## Código Java — Gerado na Ordem da Skill

### 1. `domain/loan/LoanId.java`
```java
package br.com.itau.loan.domain.loan;

import java.util.UUID;

public record LoanId(String value) {
    public LoanId {
        if (value == null || value.isBlank())
            throw new LoanException("LoanId cannot be blank");
    }

    public static LoanId generate() {
        return new LoanId(UUID.randomUUID().toString());
    }
}
```

### 2. `domain/loan/LoanStatus.java`
```java
package br.com.itau.loan.domain.loan;

public enum LoanStatus {
    REQUESTED, APPROVED, CANCELLED
}
```

### 3. `domain/loan/Loan.java`
```java
package br.com.itau.loan.domain.loan;

import java.math.BigDecimal;
import java.time.LocalDateTime;

// Java puro — sem @Entity, sem @Service, sem imports Spring/JPA
public class Loan {

    private final LoanId id;
    private final String customerId;
    private final BigDecimal amount;
    private LoanStatus status;
    private final LocalDateTime requestedAt;
    private LocalDateTime updatedAt;

    public Loan(LoanId id, String customerId, BigDecimal amount,
                LoanStatus status, LocalDateTime requestedAt, LocalDateTime updatedAt) {
        if (amount == null || amount.compareTo(BigDecimal.ZERO) <= 0)
            throw new LoanException("Loan amount must be positive");
        if (customerId == null || customerId.isBlank())
            throw new LoanException("CustomerId is required");
        this.id = id;
        this.customerId = customerId;
        this.amount = amount;
        this.status = status;
        this.requestedAt = requestedAt;
        this.updatedAt = updatedAt;
    }

    public static Loan request(String customerId, BigDecimal amount) {
        LocalDateTime now = LocalDateTime.now();
        return new Loan(LoanId.generate(), customerId, amount, LoanStatus.REQUESTED, now, now);
    }

    public void approve() {
        if (status != LoanStatus.REQUESTED)
            throw new LoanException("Only REQUESTED loans can be approved. Current: " + status);
        this.status = LoanStatus.APPROVED;
        this.updatedAt = LocalDateTime.now();
    }

    public void cancel() {
        if (status == LoanStatus.CANCELLED)
            throw new LoanException("Loan is already cancelled");
        this.status = LoanStatus.CANCELLED;
        this.updatedAt = LocalDateTime.now();
    }

    public LoanId getId() { return id; }
    public String getCustomerId() { return customerId; }
    public BigDecimal getAmount() { return amount; }
    public LoanStatus getStatus() { return status; }
    public LocalDateTime getRequestedAt() { return requestedAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
}
```

### 4. `domain/loan/LoanRepository.java`
```java
package br.com.itau.loan.domain.loan;

import java.util.List;
import java.util.Optional;

public interface LoanRepository {
    Loan save(Loan loan);
    Optional<Loan> findById(LoanId id);
    List<Loan> findByCustomerId(String customerId);
}
```

### 5. `domain/loan/LoanException.java`
```java
package br.com.itau.loan.domain.loan;

public class LoanException extends RuntimeException {
    public LoanException(String message) {
        super(message);
    }
}
```

### 6. `application/dto/loan/RequestLoanInputDto.java`
```java
package br.com.itau.loan.application.dto.loan;

import java.math.BigDecimal;

public record RequestLoanInputDto(String customerId, BigDecimal amount) {}
```

### 7. `application/dto/loan/RequestLoanOutputDto.java`
```java
package br.com.itau.loan.application.dto.loan;

import br.com.itau.loan.domain.loan.LoanStatus;
import java.math.BigDecimal;
import java.time.LocalDateTime;

public record RequestLoanOutputDto(
    String id,
    String customerId,
    BigDecimal amount,
    LoanStatus status,
    LocalDateTime requestedAt
) {}
```

### 8. `application/dto/loan/ApproveLoanInputDto.java`
```java
package br.com.itau.loan.application.dto.loan;

public record ApproveLoanInputDto(String loanId) {}
```

### 9. `application/dto/loan/ApproveLoanOutputDto.java`
```java
package br.com.itau.loan.application.dto.loan;

import br.com.itau.loan.domain.loan.LoanStatus;
import java.time.LocalDateTime;

public record ApproveLoanOutputDto(String id, LoanStatus status, LocalDateTime updatedAt) {}
```

### 10. `application/dto/loan/CancelLoanInputDto.java`
```java
package br.com.itau.loan.application.dto.loan;

public record CancelLoanInputDto(String loanId) {}
```

### 11. `application/dto/loan/CancelLoanOutputDto.java`
```java
package br.com.itau.loan.application.dto.loan;

import br.com.itau.loan.domain.loan.LoanStatus;
import java.time.LocalDateTime;

public record CancelLoanOutputDto(String id, LoanStatus status, LocalDateTime updatedAt) {}
```

### 12. `application/usecase/loan/RequestLoanUseCase.java`
```java
package br.com.itau.loan.application.usecase.loan;

import br.com.itau.loan.application.dto.loan.RequestLoanInputDto;
import br.com.itau.loan.application.dto.loan.RequestLoanOutputDto;

public interface RequestLoanUseCase {
    RequestLoanOutputDto execute(RequestLoanInputDto input);
}
```

### 13. `application/usecase/loan/RequestLoanUseCaseImpl.java`
```java
package br.com.itau.loan.application.usecase.loan;

import br.com.itau.loan.application.dto.loan.RequestLoanInputDto;
import br.com.itau.loan.application.dto.loan.RequestLoanOutputDto;
import br.com.itau.loan.domain.loan.Loan;
import br.com.itau.loan.domain.loan.LoanRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class RequestLoanUseCaseImpl implements RequestLoanUseCase {

    private final LoanRepository loanRepository;

    public RequestLoanUseCaseImpl(LoanRepository loanRepository) {
        this.loanRepository = loanRepository;
    }

    @Override
    @Transactional
    public RequestLoanOutputDto execute(RequestLoanInputDto input) {
        Loan loan = Loan.request(input.customerId(), input.amount());
        Loan saved = loanRepository.save(loan);
        return new RequestLoanOutputDto(
            saved.getId().value(),
            saved.getCustomerId(),
            saved.getAmount(),
            saved.getStatus(),
            saved.getRequestedAt()
        );
    }
}
```

### 14. `application/usecase/loan/ApproveLoanUseCase.java`
```java
package br.com.itau.loan.application.usecase.loan;

import br.com.itau.loan.application.dto.loan.ApproveLoanInputDto;
import br.com.itau.loan.application.dto.loan.ApproveLoanOutputDto;

public interface ApproveLoanUseCase {
    ApproveLoanOutputDto execute(ApproveLoanInputDto input);
}
```

### 15. `application/usecase/loan/ApproveLoanUseCaseImpl.java`
```java
@Service
public class ApproveLoanUseCaseImpl implements ApproveLoanUseCase {

    private final LoanRepository loanRepository;

    public ApproveLoanUseCaseImpl(LoanRepository loanRepository) {
        this.loanRepository = loanRepository;
    }

    @Override
    @Transactional
    public ApproveLoanOutputDto execute(ApproveLoanInputDto input) {
        Loan loan = loanRepository.findById(new LoanId(input.loanId()))
            .orElseThrow(() -> new LoanException("Loan not found: " + input.loanId()));
        loan.approve();  // lógica no domínio
        Loan saved = loanRepository.save(loan);
        return new ApproveLoanOutputDto(saved.getId().value(), saved.getStatus(), saved.getUpdatedAt());
    }
}
```

### 16. `application/usecase/loan/CancelLoanUseCase.java`
```java
public interface CancelLoanUseCase {
    CancelLoanOutputDto execute(CancelLoanInputDto input);
}
```

### 17. `application/usecase/loan/CancelLoanUseCaseImpl.java`
```java
@Service
public class CancelLoanUseCaseImpl implements CancelLoanUseCase {

    private final LoanRepository loanRepository;

    public CancelLoanUseCaseImpl(LoanRepository loanRepository) {
        this.loanRepository = loanRepository;
    }

    @Override
    @Transactional
    public CancelLoanOutputDto execute(CancelLoanInputDto input) {
        Loan loan = loanRepository.findById(new LoanId(input.loanId()))
            .orElseThrow(() -> new LoanException("Loan not found: " + input.loanId()));
        loan.cancel();  // lógica no domínio
        Loan saved = loanRepository.save(loan);
        return new CancelLoanOutputDto(saved.getId().value(), saved.getStatus(), saved.getUpdatedAt());
    }
}
```

### 18. `infrastructure/persistence/loan/LoanJpaEntity.java`
```java
package br.com.itau.loan.infrastructure.persistence.loan;

import br.com.itau.loan.domain.loan.LoanStatus;
import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "loan_loan")   // prefixo serviço_aggregate — evita colisão de schema
public class LoanJpaEntity {

    @Id
    @Column(name = "id")
    private String id;

    @Column(name = "customer_id", nullable = false)
    private String customerId;

    @Column(name = "amount", nullable = false, precision = 19, scale = 2)
    private BigDecimal amount;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private LoanStatus status;

    @Column(name = "requested_at", nullable = false)
    private LocalDateTime requestedAt;

    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;

    protected LoanJpaEntity() {}

    public LoanJpaEntity(String id, String customerId, BigDecimal amount,
                          LoanStatus status, LocalDateTime requestedAt, LocalDateTime updatedAt) {
        this.id = id; this.customerId = customerId; this.amount = amount;
        this.status = status; this.requestedAt = requestedAt; this.updatedAt = updatedAt;
    }

    public String getId() { return id; }
    public String getCustomerId() { return customerId; }
    public BigDecimal getAmount() { return amount; }
    public LoanStatus getStatus() { return status; }
    public LocalDateTime getRequestedAt() { return requestedAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
}
```

### 19. `infrastructure/persistence/loan/LoanJpaRepository.java`
```java
package br.com.itau.loan.infrastructure.persistence.loan;

import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

interface LoanJpaRepository extends JpaRepository<LoanJpaEntity, String> {
    List<LoanJpaEntity> findByCustomerId(String customerId);
}
```

### 20. `infrastructure/persistence/loan/LoanEntityMapper.java`
```java
package br.com.itau.loan.infrastructure.persistence.loan;

import br.com.itau.loan.domain.loan.Loan;
import br.com.itau.loan.domain.loan.LoanId;
import org.springframework.stereotype.Component;

@Component
public class LoanEntityMapper {

    public LoanJpaEntity toJpaEntity(Loan domain) {
        return new LoanJpaEntity(
            domain.getId().value(),
            domain.getCustomerId(),
            domain.getAmount(),
            domain.getStatus(),
            domain.getRequestedAt(),
            domain.getUpdatedAt()
        );
    }

    public Loan toDomain(LoanJpaEntity entity) {
        return new Loan(
            new LoanId(entity.getId()),
            entity.getCustomerId(),
            entity.getAmount(),
            entity.getStatus(),
            entity.getRequestedAt(),
            entity.getUpdatedAt()
        );
    }
}
```

### 21. `infrastructure/persistence/loan/LoanRepositoryImpl.java`
```java
package br.com.itau.loan.infrastructure.persistence.loan;

import br.com.itau.loan.domain.loan.Loan;
import br.com.itau.loan.domain.loan.LoanId;
import br.com.itau.loan.domain.loan.LoanRepository;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public class LoanRepositoryImpl implements LoanRepository {

    private final LoanJpaRepository jpaRepository;
    private final LoanEntityMapper mapper;

    public LoanRepositoryImpl(LoanJpaRepository jpaRepository, LoanEntityMapper mapper) {
        this.jpaRepository = jpaRepository;
        this.mapper = mapper;
    }

    @Override
    public Loan save(Loan loan) {
        return mapper.toDomain(jpaRepository.save(mapper.toJpaEntity(loan)));
    }

    @Override
    public Optional<Loan> findById(LoanId id) {
        return jpaRepository.findById(id.value()).map(mapper::toDomain);
    }

    @Override
    public List<Loan> findByCustomerId(String customerId) {
        return jpaRepository.findByCustomerId(customerId).stream()
            .map(mapper::toDomain)
            .toList();
    }
}
```

### 22. `presentation/rest/loan/RequestLoanRequest.java`
```java
package br.com.itau.loan.presentation.rest.loan;

import java.math.BigDecimal;

public class RequestLoanRequest {
    private String customerId;
    private BigDecimal amount;

    public String getCustomerId() { return customerId; }
    public void setCustomerId(String v) { customerId = v; }
    public BigDecimal getAmount() { return amount; }
    public void setAmount(BigDecimal v) { amount = v; }
}
```

### 23. `presentation/rest/loan/LoanResponse.java`
```java
package br.com.itau.loan.presentation.rest.loan;

import br.com.itau.loan.domain.loan.LoanStatus;
import java.math.BigDecimal;
import java.time.LocalDateTime;

public record LoanResponse(
    String id,
    String customerId,
    BigDecimal amount,
    LoanStatus status,
    LocalDateTime requestedAt,
    LocalDateTime updatedAt
) {}
```

### 24. `presentation/rest/loan/LoanController.java`
```java
package br.com.itau.loan.presentation.rest.loan;

import br.com.itau.loan.application.dto.loan.*;
import br.com.itau.loan.application.usecase.loan.*;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/loans")
public class LoanController {

    private final RequestLoanUseCase requestLoanUseCase;
    private final ApproveLoanUseCase approveLoanUseCase;
    private final CancelLoanUseCase cancelLoanUseCase;

    public LoanController(RequestLoanUseCase requestLoanUseCase,
                          ApproveLoanUseCase approveLoanUseCase,
                          CancelLoanUseCase cancelLoanUseCase) {
        this.requestLoanUseCase = requestLoanUseCase;
        this.approveLoanUseCase = approveLoanUseCase;
        this.cancelLoanUseCase = cancelLoanUseCase;
    }

    @PostMapping
    public ResponseEntity<LoanResponse> request(@RequestBody RequestLoanRequest req) {
        var output = requestLoanUseCase.execute(
            new RequestLoanInputDto(req.getCustomerId(), req.getAmount()));
        return ResponseEntity.ok(new LoanResponse(
            output.id(), output.customerId(), output.amount(),
            output.status(), output.requestedAt(), null));
    }

    @PatchMapping("/{id}/approve")
    public ResponseEntity<LoanResponse> approve(@PathVariable String id) {
        var output = approveLoanUseCase.execute(new ApproveLoanInputDto(id));
        return ResponseEntity.ok(new LoanResponse(
            output.id(), null, null, output.status(), null, output.updatedAt()));
    }

    @PatchMapping("/{id}/cancel")
    public ResponseEntity<LoanResponse> cancel(@PathVariable String id) {
        var output = cancelLoanUseCase.execute(new CancelLoanInputDto(id));
        return ResponseEntity.ok(new LoanResponse(
            output.id(), null, null, output.status(), null, output.updatedAt()));
    }
}
```

---

## Anti-Pattern Check (da skill)

| Checklist | Status |
|-----------|--------|
| `Loan.java` sem @Entity, @Service, @Component | ✅ |
| `LoanRepository` é interface em `domain/` | ✅ |
| @Entity apenas em `LoanJpaEntity` em `infrastructure/persistence/` | ✅ |
| 3 use cases distintos (Request, Approve, Cancel) | ✅ |
| `LoanEntityMapper` com conversão em ambos os sentidos | ✅ |
| `LoanRepositoryImpl` implementa interface de `domain/` | ✅ |
| Controller injeta UseCase via construtor, não Repository | ✅ |
| @Transactional em `UseCaseImpl`, não em domain | ✅ |
| Injeção por construtor em todos os componentes | ✅ |
| Tabela prefixada `loan_loan` para evitar colisão | ✅ |
