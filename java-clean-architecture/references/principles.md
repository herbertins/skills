# Clean Architecture Principles — Detalhamento

## Índice

- P1: Regra de Dependência
- P2: Pureza do Domínio
- P3: Separação de Entidade JPA
- P4: Repository como Porta
- P5: Fronteira de DTO
- P6: Use Case por Operação
- P7: Controller Fino
- P8: Domínio Rico (Anti-Anêmico)
- P9: Injeção por Construtor
- P10: @Transactional na Borda Correta

---

## P1 — Regra de Dependência (CRÍTICO)

**Definição:** As setas de dependência em imports apontam apenas para dentro. `presentation` e `infrastructure` podem importar `application`; `application` pode importar `domain`; `domain` não importa nada de fora de si mesmo.

```
presentation  ──import──►  application  ──import──►  domain
infrastructure ──import──►  application  ──import──►  domain
```

**Por que importa:** É este princípio que torna o domínio testável sem Spring, substituível sem reescrever regras de negócio, e portável entre frameworks.

**Regras para agentes de IA:**
- Antes de gerar qualquer import, verificar se vai na direção correta
- Se `application` precisar de algo de `infrastructure`, provavelmente é uma porta (interface) que deve ser definida em `application` e implementada em `infrastructure`
- Se `domain` precisar de algo de `application`, a lógica provavelmente pertence ao domínio mesmo

**Correto vs. Violação:**
```java
// ✅ application importa domain (correto)
import br.com.itau.account.domain.account.Account;

// ❌ domain importa application (violação P1)
import br.com.itau.account.application.dto.AccountDto;

// ❌ application importa infrastructure (violação P1)
import br.com.itau.account.infrastructure.persistence.AccountJpaEntity;
```

---

## P2 — Pureza do Domínio (CRÍTICO)

**Definição:** Nenhuma anotação de framework entra em `domain/`. Nem Spring (`@Component`, `@Service`, `@Autowired`), nem JPA (`@Entity`, `@Table`, `@Id`), nem Jakarta Validation (`@NotNull`, `@Size`).

**Por que importa:** Se o domínio importa Spring, você não consegue testar as regras de negócio sem subir um contexto Spring. Isso torna testes lentos, frágeis e caros. O domínio deve ser testável com `new`.

**Regras para agentes de IA:**
- Ao criar qualquer arquivo em `domain/`, verificar imports — se aparecer `org.springframework.*` ou `jakarta.persistence.*`, mover a responsabilidade para a camada correta
- Validação de invariantes de negócio no construtor do aggregate (Java puro), não com `@NotNull` de Jakarta

**Correto vs. Violação:**
```java
// ✅ domain puro
public class Account {
    private final AccountId id;
    private BigDecimal balance;

    public Account(AccountId id, BigDecimal initialBalance) {
        if (initialBalance.compareTo(BigDecimal.ZERO) < 0)
            throw new AccountException("Balance cannot be negative");
        this.id = id;
        this.balance = initialBalance;
    }
}

// ❌ domain com Spring (violação P2)
@Service  // ← fora
public class Account { ... }

// ❌ domain com JPA (violação P2 e P3)
@Entity  // ← fora
public class Account { ... }
```

---

## P3 — Separação de Entidade JPA (CRÍTICO)

**Definição:** O objeto de domínio e a entidade JPA são classes diferentes. O domain object representa o estado e comportamento de negócio; o `JpaEntity` representa como esse estado é persistido no banco.

**Por que importa:** Schema de banco e modelo de domínio têm razões diferentes para mudar. Acoplá-los força mudanças de negócio a afetar o schema e vice-versa. A separação também permite evolução independente (ex: migrar de JPA para outro mecanismo de persistência sem tocar no domínio).

**Regras para agentes de IA:**
- Sempre que gerar uma entidade de domínio, criar em paralelo sua `JpaEntity` em `infrastructure/persistence/`
- Nunca adicionar `@Column`, `@JoinColumn`, `@OneToMany` ao domain object
- O `EntityMapper` é obrigatório — não fazer conversão inline no `RepositoryImpl`

**Estrutura correta:**
```
domain/account/Account.java                           ← Java puro
infrastructure/persistence/account/
  ├── AccountJpaEntity.java                           ← @Entity aqui
  ├── AccountJpaRepository.java                       ← Spring Data
  ├── AccountEntityMapper.java                        ← conversão
  └── AccountRepositoryImpl.java                      ← implementa domain port
```

---

## P4 — Repository como Porta

**Definição:** A interface do repository é definida no domínio, com a linguagem do domínio (operações de negócio). A implementação fica em infrastructure e usa Spring Data / JPA.

**Por que importa:** O domínio declara o que precisa; a infrastructure entrega. Isso inverte o controle — o domínio não depende de como os dados são persistidos.

**Regras para agentes de IA:**
- Interface em `domain/<aggregate>/` com métodos que fazem sentido para o negócio
- Nunca expor `Page<T>`, `Pageable`, `@Query` ou qualquer construto JPA/Spring Data na interface de domínio
- Se precisar de paginação ou queries complexas, criar método na interface com tipos de domínio próprios

**Correto vs. Violação:**
```java
// ✅ interface de domínio (linguagem do negócio)
public interface AccountRepository {
    Account save(Account account);
    Optional<Account> findById(AccountId id);
    List<Account> findByCustomerId(CustomerId customerId);
}

// ❌ interface com Spring Data na assinatura (vazamento de infra)
public interface AccountRepository {
    Account save(Account account);
    Page<Account> findAll(Pageable pageable);  // ← Pageable é Spring
}
```

---

## P5 — Fronteira de DTO

**Definição:** Objetos de domínio não cruzam fronteiras de camada. `presentation/` usa seus próprios `Request`/`Response`; `application/` usa `InputDto`/`OutputDto`; `domain/` usa domain objects.

**Por que importa:** Expor domain objects via HTTP cria acoplamento entre a API pública e o modelo interno. Uma refatoração de domínio quebra contratos externos. DTOs permitem evoluir os dois independentemente e filtrar campos sensíveis.

**Mapeamento de responsabilidade:**
```
HTTP Request  ──► Request DTO (presentation)
                    ──► InputDto (application)
                          ──► Domain Object (domain)
                          ◄── Domain Object
                    ◄── OutputDto (application)
◄── Response DTO (presentation)
```

---

## P6 — Use Case por Operação

**Definição:** Cada operação de negócio tem sua própria interface `UseCase` e sua implementação `UseCaseImpl`. Não criar um `AccountService` genérico com 10 métodos.

**Por que importa:** Use cases únicos respeitam o Princípio da Responsabilidade Única, são mais fáceis de testar, e tornam explícito o inventário de operações de negócio do sistema.

**Correto vs. Violação:**
```java
// ✅ use cases específicos
CreateAccountUseCase, DepositUseCase, WithdrawUseCase, GetAccountUseCase

// ❌ serviço genérico com tudo misturado
AccountApplicationService {
    createAccount(...) { ... }
    deposit(...) { ... }
    withdraw(...) { ... }
    getAccount(...) { ... }
}
```

---

## P7 — Controller Fino

**Definição:** Controller traduz HTTP para use case DTO e chama o use case. Nada mais.

**Por que importa:** Lógica de negócio no controller não é testável sem subir o contexto HTTP. Controllers gordos são difíceis de manter e frequentemente duplicam lógica.

**Regras para agentes de IA:**
- Se um controller tem mais de ~30 linhas de lógica real, algo está errado
- Validação de formato de entrada (`@Valid`) é OK no controller; validação de regras de negócio vai no domínio
- Controller nunca injeta `Repository` ou `Service` do domínio diretamente

---

## P8 — Domínio Rico (Anti-Anêmico)

**Definição:** Lógica de negócio vive no aggregate root ou domain service, não no use case. O use case orquestra, não decide.

**Por que importa:** Domínio anêmico (apenas getters/setters, lógica nos serviços de aplicação) derrota o propósito do DDD e torna o código procedural disfarçado de OO.

**Regras para agentes de IA:**
- Pergunta-teste: "O aggregate root tem comportamentos além de getters/setters?" Se não, o domínio está anêmico.
- Cálculos, validações de invariante, transições de estado → aggregate root
- Operações que envolvem múltiplos aggregates → domain service

---

## P9 — Injeção por Construtor

**Definição:** Todas as dependências são declaradas no construtor. Nunca usar `@Autowired` em campo.

**Por que importa:** Dependências no construtor são visíveis, obrigatórias e testáveis sem Spring (você instancia a classe com mocks manualmente). `@Autowired` em campo esconde dependências e exige reflexão para testar.

```java
// ✅ injeção por construtor
public class CreateAccountUseCaseImpl {
    private final AccountRepository repository;

    public CreateAccountUseCaseImpl(AccountRepository repository) {
        this.repository = repository;
    }
}

// ❌ @Autowired em campo (evitar)
@Autowired
private AccountRepository repository;
```

---

## P10 — @Transactional na Borda Correta

**Definição:** `@Transactional` fica em `UseCaseImpl` (para operações que envolvem múltiplos passos) ou em `RepositoryImpl` (para operações simples). Nunca em domain services.

**Por que importa:** Domain service não conhece persistence — colocar `@Transactional` lá viola P2 (pureza do domínio) pois a anotação implica conhecimento de transação de banco de dados.

```java
// ✅ @Transactional no use case
@Service
public class TransferFundsUseCaseImpl implements TransferFundsUseCase {

    @Override
    @Transactional
    public TransferOutputDto execute(TransferInputDto input) {
        var source = accountRepository.findById(input.sourceId()).orElseThrow();
        var target = accountRepository.findById(input.targetId()).orElseThrow();
        source.debit(input.amount());   // lógica no domínio
        target.credit(input.amount()); // lógica no domínio
        accountRepository.save(source);
        accountRepository.save(target);
        return new TransferOutputDto(source, target);
    }
}

// ❌ @Transactional no domain service (violação P10 + P2)
@Service  // ← não deveria ter Spring no domínio
public class TransferDomainService {
    @Transactional  // ← domínio não gerencia transações
    public void transfer(...) { ... }
}
```
