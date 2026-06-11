# Clean Architecture Violations in `br.com.itau.payment`

### Violations Found

**Violation 1 — `@Entity`/`@Table` on a domain class (`domain/Payment.java`)**
JPA is an infrastructure concern. Domain objects must be pure POJOs with zero dependency on any framework. Placing `@Entity @Table(name="payment")` inside `domain/` makes the innermost layer depend on the outermost, directly breaking the Dependency Rule.

**Violation 2 — `PaymentController` injects `PaymentRepository` directly**
The presentation layer (controller) reaches straight into the infrastructure layer (repository), skipping the application/use-case layer entirely. A controller must only depend on an inbound port (use-case interface); it must never know that a database exists.

**Violation 3 — No use-case / application service layer**
There is nothing between the HTTP handler and persistence. All business orchestration is absent. Without this layer the code cannot be unit-tested in isolation and violates the separation-of-concerns boundary.

**Violation 4 — No domain port (outbound interface) for persistence**
`PaymentRepository` is presumably a Spring Data `JpaRepository` — an infrastructure artifact. There is no domain interface that declares the persistence contract in framework-agnostic terms. The domain must own that interface; infrastructure must implement it.

---

### Corrected File Structure

```
br/com/itau/payment/
├── domain/
│   ├── Payment.java                          # pure POJO, no JPA
│   └── port/
│       └── outbound/
│           └── PaymentRepository.java        # plain Java interface (domain port)
├── application/
│   └── usecase/
│       ├── CreatePaymentUseCase.java         # inbound port interface
│       └── CreatePaymentUseCaseImpl.java     # @Service, implements port, injects domain port
├── infrastructure/
│   └── persistence/
│       ├── PaymentJpaEntity.java             # @Entity @Table lives here
│       ├── PaymentJpaRepository.java         # extends JpaRepository
│       └── PaymentRepositoryAdapter.java     # @Component, implements domain PaymentRepository
└── presentation/
    └── PaymentController.java                # injects CreatePaymentUseCase only
```

---

### Before / After Code Snippets

**Fix 1 — Strip JPA from domain; add a JPA entity in infrastructure**

BEFORE `domain/Payment.java`:
```java
@Entity
@Table(name = "payment")
public class Payment { ... }   // imports jakarta.persistence.*
```

AFTER `domain/Payment.java`:
```java
public class Payment {           // no framework imports at all
    private Long id;
    private String description;
    private Double amount;
    public Payment(Long id, String description, Double amount) { ... }
    // getters only
}
```

NEW `infrastructure/persistence/PaymentJpaEntity.java`:
```java
@Entity
@Table(name = "payment")
public class PaymentJpaEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String description;
    private Double amount;
    // JPA-required no-arg constructor + getters/setters
}
```

---

**Fix 2 — Domain port (outbound interface)**

NEW `domain/port/outbound/PaymentRepository.java`:
```java
package br.com.itau.payment.domain.port.outbound;
import br.com.itau.payment.domain.Payment;
import java.util.Optional;

public interface PaymentRepository {
    Payment save(Payment payment);
    Optional<Payment> findById(Long id);
}
```

---

**Fix 3 — Infrastructure adapter implementing the domain port**

NEW `infrastructure/persistence/PaymentRepositoryAdapter.java`:
```java
@Component
public class PaymentRepositoryAdapter implements PaymentRepository {
    private final PaymentJpaRepository jpaRepository;
    // constructor injection
    @Override public Payment save(Payment p) {
        return toDomain(jpaRepository.save(toEntity(p)));
    }
    @Override public Optional<Payment> findById(Long id) {
        return jpaRepository.findById(id).map(this::toDomain);
    }
    // toEntity() and toDomain() mappers
}
```

---

**Fix 4 — Use-case layer (inbound port + implementation)**

NEW `application/usecase/CreatePaymentUseCase.java`:
```java
public interface CreatePaymentUseCase {
    Payment execute(Payment payment);
}
```

NEW `application/usecase/CreatePaymentUseCaseImpl.java`:
```java
@Service
public class CreatePaymentUseCaseImpl implements CreatePaymentUseCase {
    private final PaymentRepository paymentRepository; // domain port, not JPA
    @Override public Payment execute(Payment payment) {
        // business rules here
        return paymentRepository.save(payment);
    }
}
```

---

**Fix 5 — Controller depends only on the inbound port**

BEFORE:
```java
public class PaymentController {
    private final PaymentJpaRepository paymentRepository; // VIOLATION
    @PostMapping public Payment create(@RequestBody Payment p) {
        return paymentRepository.save(p); // skips use-case layer
    }
}
```

AFTER:
```java
public class PaymentController {
    private final CreatePaymentUseCase createPaymentUseCase; // inbound port only
    @PostMapping public Payment create(@RequestBody Payment p) {
        return createPaymentUseCase.execute(p);
    }
}
```

---

### Summary Table

| # | Violation | Fix |
|---|-----------|-----|
| 1 | `@Entity`/`@Table` on `domain/Payment` | Move to `infrastructure/persistence/PaymentJpaEntity` |
| 2 | No domain outbound port | Create `domain/port/outbound/PaymentRepository` interface |
| 3 | No infrastructure adapter | Create `PaymentRepositoryAdapter` implementing domain port |
| 4 | No use-case layer | Create `CreatePaymentUseCase` interface + `CreatePaymentUseCaseImpl` |
| 5 | Controller injects repository directly | Controller injects `CreatePaymentUseCase` (inbound port) only |
