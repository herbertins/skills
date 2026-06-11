# Clean Architecture Compliance Report — br.com.itau.payment
## (Using java-clean-architecture skill)

## Violations Found (7 total)

| # | Violation | Princípio(s) | Criticidade |
|---|-----------|--------------|-------------|
| V1 | `Payment` tem `@Entity @Table` em `domain/` | P2 (Pureza do Domínio) + P3 (Separação JPA) | 🔴 CRÍTICO |
| V2 | `PaymentController` injeta `PaymentRepository` diretamente | P1 (Regra de Dependência) + P7 (Controller Fino) | 🔴 CRÍTICO |
| V3 | Nenhum `PaymentJpaEntity` em `infrastructure/persistence/` | P3 | 🔴 CRÍTICO |
| V4 | Nenhuma camada `application/` (use cases + DTOs) | P1 + P6 (Use Case por Operação) | 🟠 ALTO |
| V5 | Nenhum `PaymentEntityMapper` | P3 | 🟠 ALTO |
| V6 | Domain object provavelmente exposto via resposta HTTP | P5 (Fronteira de DTO) | 🟠 ALTO |
| V7 | Nenhum `PaymentRepositoryImpl` em infrastructure | P4 (Repository como Porta) | 🟠 ALTO |

---

## Estrutura Corrigida

```
br/com/itau/payment/
├── domain/
│   └── payment/
│       ├── Payment.java                   # Java puro — sem @Entity, sem Spring
│       ├── PaymentRepository.java         # interface (port de saída)
│       └── PaymentException.java
│
├── application/
│   ├── usecase/
│   │   └── payment/
│   │       ├── CreatePaymentUseCase.java      # interface (port de entrada)
│   │       └── CreatePaymentUseCaseImpl.java  # @Service + @Transactional
│   └── dto/
│       └── payment/
│           ├── PaymentInputDto.java
│           └── PaymentOutputDto.java
│
├── infrastructure/
│   └── persistence/
│       └── payment/
│           ├── PaymentJpaEntity.java          # @Entity @Table aqui
│           ├── PaymentJpaRepository.java      # Spring Data interface
│           ├── PaymentEntityMapper.java       # converte JpaEntity ↔ domain
│           └── PaymentRepositoryImpl.java     # implementa domain/PaymentRepository
│
└── presentation/
    └── rest/
        └── payment/
            ├── PaymentController.java
            ├── PaymentRequest.java
            └── PaymentResponse.java
```

---

## Código Before/After por Violação

### V1 + V3 — @Entity no domain object (P2 + P3)

**ANTES** `domain/payment/Payment.java`:
```java
@Entity
@Table(name = "payment")  // violação P2 + P3
public class Payment {
    @Id
    @GeneratedValue
    private Long id;
    private String description;
    private Double amount;
    // getters/setters
}
```

**DEPOIS** `domain/payment/Payment.java`:
```java
package br.com.itau.payment.domain.payment;

// zero imports Spring ou JPA
public class Payment {
    private final PaymentId id;
    private final String description;
    private final Double amount;

    public Payment(PaymentId id, String description, Double amount) {
        if (amount == null || amount <= 0)
            throw new PaymentException("Amount must be positive");
        this.id = id;
        this.description = description;
        this.amount = amount;
    }

    public PaymentId getId() { return id; }
    public String getDescription() { return description; }
    public Double getAmount() { return amount; }
}
```

**NOVO** `infrastructure/persistence/payment/PaymentJpaEntity.java`:
```java
package br.com.itau.payment.infrastructure.persistence.payment;

import jakarta.persistence.*;

@Entity
@Table(name = "payment_payment")  // prefixo serviço_aggregate
public class PaymentJpaEntity {

    @Id
    @Column(name = "id")
    private String id;

    @Column(name = "description", nullable = false)
    private String description;

    @Column(name = "amount", nullable = false)
    private Double amount;

    protected PaymentJpaEntity() {}

    public PaymentJpaEntity(String id, String description, Double amount) {
        this.id = id;
        this.description = description;
        this.amount = amount;
    }

    public String getId() { return id; }
    public String getDescription() { return description; }
    public Double getAmount() { return amount; }
}
```

---

### V2 — Repository injetado no Controller (P1 + P7)

**ANTES** `presentation/rest/payment/PaymentController.java`:
```java
@RestController
@RequestMapping("/api/v1/payments")
public class PaymentController {

    private final PaymentRepository paymentRepository;  // violação P7

    public PaymentController(PaymentRepository paymentRepository) {
        this.paymentRepository = paymentRepository;
    }

    @PostMapping
    public Payment create(@RequestBody Payment payment) {
        return paymentRepository.save(payment);  // viola P5: expõe domain object
    }
}
```

**DEPOIS** `presentation/rest/payment/PaymentController.java`:
```java
@RestController
@RequestMapping("/api/v1/payments")
public class PaymentController {

    private final CreatePaymentUseCase createPaymentUseCase;  // injeta use case

    public PaymentController(CreatePaymentUseCase createPaymentUseCase) {
        this.createPaymentUseCase = createPaymentUseCase;
    }

    @PostMapping
    public ResponseEntity<PaymentResponse> create(@RequestBody PaymentRequest request) {
        var input = new PaymentInputDto(request.getDescription(), request.getAmount());
        var output = createPaymentUseCase.execute(input);
        return ResponseEntity.ok(new PaymentResponse(output));
    }
}
```

---

### V4 + V6 — Use case e DTOs ausentes (P1 + P5 + P6)

**NOVO** `application/usecase/payment/CreatePaymentUseCase.java`:
```java
package br.com.itau.payment.application.usecase.payment;

import br.com.itau.payment.application.dto.payment.PaymentInputDto;
import br.com.itau.payment.application.dto.payment.PaymentOutputDto;

public interface CreatePaymentUseCase {
    PaymentOutputDto execute(PaymentInputDto input);
}
```

**NOVO** `application/usecase/payment/CreatePaymentUseCaseImpl.java`:
```java
@Service
public class CreatePaymentUseCaseImpl implements CreatePaymentUseCase {

    private final PaymentRepository paymentRepository;  // domain port

    public CreatePaymentUseCaseImpl(PaymentRepository paymentRepository) {
        this.paymentRepository = paymentRepository;
    }

    @Override
    @Transactional
    public PaymentOutputDto execute(PaymentInputDto input) {
        var payment = new Payment(
            new PaymentId(UUID.randomUUID().toString()),
            input.description(),
            input.amount()
        );
        var saved = paymentRepository.save(payment);
        return new PaymentOutputDto(saved.getId().value(), saved.getDescription(), saved.getAmount());
    }
}
```

---

## Resumo de Ações Priorizadas

1. **P0 — imediato:** Mover `@Entity` de `Payment.java` para novo `PaymentJpaEntity.java` em `infrastructure/persistence/payment/`
2. **P0 — imediato:** Criar interface `PaymentRepository` em `domain/payment/` + `PaymentRepositoryImpl` em `infrastructure/`
3. **P0 — imediato:** Criar `PaymentEntityMapper` com conversão em ambos os sentidos
4. **P1 — este sprint:** Criar `CreatePaymentUseCase` (interface) + `CreatePaymentUseCaseImpl` em `application/`
5. **P1 — este sprint:** Refatorar `PaymentController` para injetar `CreatePaymentUseCase`
6. **P1 — este sprint:** Criar `PaymentRequest`/`PaymentResponse` DTOs em `presentation/` (nunca expor domain object via HTTP)
