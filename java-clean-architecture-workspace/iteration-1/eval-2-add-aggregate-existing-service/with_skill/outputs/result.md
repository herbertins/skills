# Adicionando o Aggregate `CreditLimit` ao `account-service`
## (Using java-clean-architecture skill)

## 1. Árvore de Arquivos Completa

```
br/com/itau/account/
├── domain/
│   ├── account/
│   │   ├── Account.java                          # existente
│   │   ├── AccountRepository.java                # existente
│   │   └── AccountException.java                 # existente
│   │
│   └── creditlimit/                              [NOVO]
│       ├── CreditLimit.java
│       ├── CreditLimitRepository.java            # porta de saída (interface)
│       └── CreditLimitException.java
│
├── application/
│   ├── usecase/
│   │   └── creditlimit/                          [NOVO]
│   │       ├── SetCreditLimitUseCase.java
│   │       ├── SetCreditLimitUseCaseImpl.java
│   │       ├── GetCreditLimitUseCase.java
│   │       └── GetCreditLimitUseCaseImpl.java
│   │
│   └── dto/
│       └── creditlimit/                          [NOVO]
│           ├── SetCreditLimitInputDto.java
│           ├── SetCreditLimitOutputDto.java
│           ├── GetCreditLimitInputDto.java
│           └── GetCreditLimitOutputDto.java
│
├── infrastructure/
│   └── persistence/
│       └── creditlimit/                          [NOVO]
│           ├── CreditLimitJpaEntity.java         # @Entity aqui
│           ├── CreditLimitJpaRepository.java     # Spring Data interface
│           ├── CreditLimitRepositoryImpl.java    # implementa domain port
│           └── CreditLimitEntityMapper.java
│
└── presentation/
    └── rest/
        └── creditlimit/                          [NOVO]
            ├── CreditLimitController.java
            ├── SetCreditLimitRequest.java
            ├── SetCreditLimitResponse.java
            └── GetCreditLimitResponse.java
```

---

## 2. Código Java

### `domain/creditlimit/CreditLimit.java`
```java
package br.com.itau.account.domain.creditlimit;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

public class CreditLimit {
    private final UUID id;
    private final UUID accountId;
    private BigDecimal limitAmount;
    private LocalDateTime updatedAt;

    public CreditLimit(UUID id, UUID accountId, BigDecimal limitAmount, LocalDateTime updatedAt) {
        this.id = id;
        this.accountId = accountId;
        this.limitAmount = limitAmount;
        this.updatedAt = updatedAt;
    }

    public static CreditLimit define(UUID accountId, BigDecimal limitAmount) {
        validate(limitAmount);
        return new CreditLimit(UUID.randomUUID(), accountId, limitAmount, LocalDateTime.now());
    }

    public void updateLimit(BigDecimal newLimit) {
        validate(newLimit);
        this.limitAmount = newLimit;
        this.updatedAt = LocalDateTime.now();
    }

    private static void validate(BigDecimal amount) {
        if (amount == null || amount.compareTo(BigDecimal.ZERO) < 0)
            throw new CreditLimitException("Limite não pode ser nulo ou negativo.");
    }

    public UUID getId() { return id; }
    public UUID getAccountId() { return accountId; }
    public BigDecimal getLimitAmount() { return limitAmount; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
}
```

### `domain/creditlimit/CreditLimitRepository.java`
```java
package br.com.itau.account.domain.creditlimit;

import java.util.Optional;
import java.util.UUID;

public interface CreditLimitRepository {
    CreditLimit save(CreditLimit creditLimit);
    Optional<CreditLimit> findByAccountId(UUID accountId);
}
```

### `application/usecase/creditlimit/SetCreditLimitUseCase.java`
```java
package br.com.itau.account.application.usecase.creditlimit;

import br.com.itau.account.application.dto.creditlimit.SetCreditLimitInputDto;
import br.com.itau.account.application.dto.creditlimit.SetCreditLimitOutputDto;

public interface SetCreditLimitUseCase {
    SetCreditLimitOutputDto execute(SetCreditLimitInputDto input);
}
```

### `application/usecase/creditlimit/SetCreditLimitUseCaseImpl.java`
```java
package br.com.itau.account.application.usecase.creditlimit;

import br.com.itau.account.application.dto.creditlimit.*;
import br.com.itau.account.domain.creditlimit.CreditLimit;
import br.com.itau.account.domain.creditlimit.CreditLimitRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.UUID;

@Service
public class SetCreditLimitUseCaseImpl implements SetCreditLimitUseCase {

    private final CreditLimitRepository creditLimitRepository;

    public SetCreditLimitUseCaseImpl(CreditLimitRepository creditLimitRepository) {
        this.creditLimitRepository = creditLimitRepository;
    }

    @Override
    @Transactional
    public SetCreditLimitOutputDto execute(SetCreditLimitInputDto input) {
        CreditLimit creditLimit = creditLimitRepository
            .findByAccountId(input.getAccountId())
            .map(existing -> { existing.updateLimit(input.getLimitAmount()); return existing; })
            .orElseGet(() -> CreditLimit.define(input.getAccountId(), input.getLimitAmount()));

        CreditLimit saved = creditLimitRepository.save(creditLimit);
        return new SetCreditLimitOutputDto(saved.getId(), saved.getAccountId(),
            saved.getLimitAmount(), saved.getUpdatedAt());
    }
}
```

### `application/usecase/creditlimit/GetCreditLimitUseCase.java`
```java
package br.com.itau.account.application.usecase.creditlimit;

import br.com.itau.account.application.dto.creditlimit.GetCreditLimitInputDto;
import br.com.itau.account.application.dto.creditlimit.GetCreditLimitOutputDto;

public interface GetCreditLimitUseCase {
    GetCreditLimitOutputDto execute(GetCreditLimitInputDto input);
}
```

### `application/usecase/creditlimit/GetCreditLimitUseCaseImpl.java`
```java
@Service
public class GetCreditLimitUseCaseImpl implements GetCreditLimitUseCase {

    private final CreditLimitRepository creditLimitRepository;

    public GetCreditLimitUseCaseImpl(CreditLimitRepository creditLimitRepository) {
        this.creditLimitRepository = creditLimitRepository;
    }

    @Override
    @Transactional(readOnly = true)
    public GetCreditLimitOutputDto execute(GetCreditLimitInputDto input) {
        CreditLimit creditLimit = creditLimitRepository
            .findByAccountId(input.getAccountId())
            .orElseThrow(() -> new CreditLimitException(
                "Limite não encontrado para conta: " + input.getAccountId()));
        return new GetCreditLimitOutputDto(creditLimit.getId(), creditLimit.getAccountId(),
            creditLimit.getLimitAmount(), creditLimit.getUpdatedAt());
    }
}
```

### `infrastructure/persistence/creditlimit/CreditLimitJpaEntity.java`
```java
package br.com.itau.account.infrastructure.persistence.creditlimit;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

@Entity
@Table(name = "account_credit_limit")
public class CreditLimitJpaEntity {
    @Id
    @Column(name = "id")
    private UUID id;

    @Column(name = "account_id", nullable = false)
    private UUID accountId;

    @Column(name = "limit_amount", nullable = false, precision = 19, scale = 2)
    private BigDecimal limitAmount;

    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;

    protected CreditLimitJpaEntity() {}

    public CreditLimitJpaEntity(UUID id, UUID accountId, BigDecimal limitAmount, LocalDateTime updatedAt) {
        this.id = id; this.accountId = accountId;
        this.limitAmount = limitAmount; this.updatedAt = updatedAt;
    }

    public UUID getId() { return id; }
    public UUID getAccountId() { return accountId; }
    public BigDecimal getLimitAmount() { return limitAmount; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
}
```

### `infrastructure/persistence/creditlimit/CreditLimitEntityMapper.java`
```java
@Component
public class CreditLimitEntityMapper {
    public CreditLimit toDomain(CreditLimitJpaEntity e) {
        return new CreditLimit(e.getId(), e.getAccountId(), e.getLimitAmount(), e.getUpdatedAt());
    }
    public CreditLimitJpaEntity toJpaEntity(CreditLimit d) {
        return new CreditLimitJpaEntity(d.getId(), d.getAccountId(), d.getLimitAmount(), d.getUpdatedAt());
    }
}
```

### `infrastructure/persistence/creditlimit/CreditLimitRepositoryImpl.java`
```java
@Component
public class CreditLimitRepositoryImpl implements CreditLimitRepository {
    private final CreditLimitJpaRepository jpaRepository;
    private final CreditLimitEntityMapper mapper;

    public CreditLimitRepositoryImpl(CreditLimitJpaRepository jpaRepository, CreditLimitEntityMapper mapper) {
        this.jpaRepository = jpaRepository; this.mapper = mapper;
    }

    @Override
    public CreditLimit save(CreditLimit creditLimit) {
        return mapper.toDomain(jpaRepository.save(mapper.toJpaEntity(creditLimit)));
    }

    @Override
    public Optional<CreditLimit> findByAccountId(UUID accountId) {
        return jpaRepository.findByAccountId(accountId).map(mapper::toDomain);
    }
}
```

### `presentation/rest/creditlimit/CreditLimitController.java`
```java
@RestController
@RequestMapping("/v1/accounts/{accountId}/credit-limit")
public class CreditLimitController {

    private final SetCreditLimitUseCase setCreditLimitUseCase;
    private final GetCreditLimitUseCase getCreditLimitUseCase;

    public CreditLimitController(SetCreditLimitUseCase setCreditLimitUseCase,
                                  GetCreditLimitUseCase getCreditLimitUseCase) {
        this.setCreditLimitUseCase = setCreditLimitUseCase;
        this.getCreditLimitUseCase = getCreditLimitUseCase;
    }

    @PutMapping
    public ResponseEntity<SetCreditLimitResponse> setCreditLimit(
            @PathVariable UUID accountId,
            @RequestBody SetCreditLimitRequest request) {
        var output = setCreditLimitUseCase.execute(
            new SetCreditLimitInputDto(accountId, request.getLimitAmount()));
        return ResponseEntity.ok(new SetCreditLimitResponse(
            output.getId(), output.getAccountId(), output.getLimitAmount(), output.getUpdatedAt()));
    }

    @GetMapping
    public ResponseEntity<GetCreditLimitResponse> getCreditLimit(@PathVariable UUID accountId) {
        var output = getCreditLimitUseCase.execute(new GetCreditLimitInputDto(accountId));
        return ResponseEntity.ok(new GetCreditLimitResponse(
            output.getId(), output.getAccountId(), output.getLimitAmount(), output.getUpdatedAt()));
    }
}
```

---

## 3. CreditLimit deve referenciar Account?

**Decisão: Não — CreditLimit armazena apenas `UUID accountId`.**

DDD define que aggregates se referenciam por identidade, nunca por objeto. Isso garante:
- Ciclo de vida independente (cada aggregate salva/carrega na sua transação)
- Sem import cruzado entre `domain.account` e `domain.creditlimit`
- Testabilidade: CreditLimit pode ser testado com apenas um UUID, sem construir um Account

Se for necessário validar a existência da conta antes de definir o limite, essa verificação pertence ao **use case** (que pode injetar `AccountRepository` como port de domínio), não ao aggregate `CreditLimit`.

---

## 4. Compliance Check

| Princípio | Status |
|-----------|--------|
| P1 Regra de Dependência | ✅ presentation → application → domain |
| P2 Pureza do Domínio | ✅ CreditLimit: Java puro, zero Spring/JPA |
| P3 Separação JPA | ✅ @Entity apenas em CreditLimitJpaEntity |
| P4 Repository como Porta | ✅ interface em domain/, impl em infrastructure/ |
| P5 Fronteira de DTO | ✅ presentation usa Response próprios |
| P6 Use Case por Operação | ✅ Set e Get separados |
| P7 Controller Fino | ✅ injeta UseCase, zero lógica |
| P8 Domínio Rico | ✅ define() e updateLimit() encapsulam invariantes |
| P9 Injeção por Construtor | ✅ todos os componentes |
| P10 @Transactional correto | ✅ nos UseCaseImpl |
