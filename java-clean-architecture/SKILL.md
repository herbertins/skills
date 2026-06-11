---
name: java-clean-architecture
description: >
  Expert em Clean Architecture com Spring Boot para microsserviços Itaú (br.com.itau).
  SEMPRE leia esta skill ANTES de propor qualquer correção ou plano que envolva: estrutura
  de camadas, organização de pacotes, posicionamento de componentes Spring, padrões de
  repository, design de use cases, fronteiras de DTO, separação de entidades JPA, ou
  qualquer mudança estrutural em um microsserviço. Os padrões aqui enforcement camadas
  explícitas (domain/application/infrastructure/presentation) com regra de dependência
  estrita — raciocinar sem ler esta skill produz violações como @Entity em objetos de
  domínio ou repositories injetados em controllers.
  Triggers: criar microsserviços, "scaffold a service", "cria um serviço", avaliar
  compliance de Clean Architecture, identificar violações de camada, entender regra de
  dependência, corrigir imports entre camadas, revisar arquitetura, "ports and adapters",
  "use case", "domain entity", "repository pattern", "JPA entity", "DTO boundary",
  "application service", "dependency rule", "clean arch", "camadas", "arquitetura limpa",
  "microsserviço", "microservice", "compliance", "aggregate", "port", "adapter",
  "inbound", "outbound", "domain purity", "br.com.itau", "itau", "domain layer",
  "infrastructure layer", "application layer", "presentation layer".
---

# Java Clean Architecture Expert (br.com.itau)

Você é um especialista em Clean Architecture para microsserviços Spring Boot no contexto Itaú. Esta skill cobre design, scaffold, avaliação e compliance de microsserviços que seguem o padrão de camadas explícitas com separação total entre domínio e infraestrutura.

## Core Philosophy

- **Domain = Linguagem do negócio**: Nenhum framework entra na camada de domínio — apenas Java puro
- **Application = Orquestrador**: Use cases coordenam domínio e portas, sem lógica de negócio própria
- **Infrastructure = Detalhe técnico**: JPA, HTTP clients, filas — adaptadores para o mundo externo
- **Presentation = Fronteira de entrada**: Controllers traduzem HTTP para use cases e vice-versa

## Theoretical Foundations

| Padrão | Fonte | O que adotamos |
|--------|-------|----------------|
| Clean Architecture | R. Martin (2017) | Regra de dependência: camadas internas ignoram camadas externas |
| Hexagonal / Ports & Adapters | A. Cockburn (2005) | Domínio expõe portas; infrastructure implementa adaptadores |
| DDD — Aggregate | E. Evans (2003) | Aggregate root controla consistência; repository por aggregate |
| Screaming Architecture | R. Martin (2017, ch. 21) | `ls domain/` revela o negócio, não frameworks |
| Package by Feature | R. Martin; P. Webb (Spring) | Organize por aggregate/feature, não por camada técnica dentro de domain |

## Estrutura de Pacotes

```
br.com.itau.<service>/
├── domain/
│   ├── <aggregate>/
│   │   ├── <Aggregate>.java              # aggregate root (Java puro, sem anotações Spring/JPA)
│   │   ├── <Aggregate>Repository.java    # interface de repositório (porta de saída)
│   │   ├── <Aggregate>Service.java       # domain service (quando lógica não cabe no aggregate)
│   │   └── <Aggregate>Exception.java     # exceções de domínio
│   └── shared/
│       └── <ValueObject>.java            # value objects reutilizados entre aggregates
│
├── application/
│   ├── usecase/
│   │   └── <aggregate>/
│   │       ├── <Operation>UseCase.java       # porta de entrada (interface)
│   │       └── <Operation>UseCaseImpl.java   # implementação do use case
│   └── dto/
│       └── <aggregate>/
│           ├── <Aggregate>InputDto.java
│           └── <Aggregate>OutputDto.java
│
├── infrastructure/
│   ├── persistence/
│   │   └── <aggregate>/
│   │       ├── <Aggregate>JpaEntity.java         # @Entity JPA (separado do domínio)
│   │       ├── <Aggregate>JpaRepository.java     # interface Spring Data
│   │       ├── <Aggregate>RepositoryImpl.java    # implementa domain/<Aggregate>Repository
│   │       └── <Aggregate>EntityMapper.java      # converte JpaEntity ↔ domain object
│   ├── external/
│   │   └── <partner>/
│   │       ├── <Partner>Client.java              # Feign/RestTemplate interface
│   │       └── <Partner>ClientImpl.java          # adaptador para serviço externo
│   └── config/
│       ├── PersistenceConfig.java
│       └── BeanConfig.java
│
└── presentation/
    └── rest/
        └── <aggregate>/
            ├── <Aggregate>Controller.java
            ├── <Aggregate>Request.java           # DTO de entrada HTTP
            └── <Aggregate>Response.java          # DTO de saída HTTP
```

> **Nota**: `domain/shared/` é para value objects genuinamente compartilhados (ex: `Money`, `CPF`). Se um objeto só é usado por um aggregate, ele vai dentro da pasta daquele aggregate.

## Os 10 Princípios

| # | Princípio | Criticidade | Regra Central |
|---|-----------|-------------|---------------|
| 1 | **Regra de Dependência** | 🔴 CRÍTICO | `presentation` e `infrastructure` dependem de `application`; `application` depende de `domain`; `domain` não depende de ninguém |
| 2 | **Pureza do Domínio** | 🔴 CRÍTICO | Zero anotações Spring/JPA/Jakarta em `domain/` — apenas Java puro e interfaces |
| 3 | **Separação de Entidade JPA** | 🔴 CRÍTICO | `@Entity` vive em `infrastructure/persistence/`; domain object é Java puro com estado de negócio |
| 4 | **Repository como Porta** | Alto | Interface do repository fica em `domain/`; implementação em `infrastructure/persistence/` |
| 5 | **Fronteira de DTO** | Alto | Objetos de domínio nunca saem de `application/`; `presentation/` usa Request/Response próprios |
| 6 | **Use Case por Operação** | Alto | Um use case = uma operação de negócio; `UseCaseImpl` orquestra, não decide |
| 7 | **Controller Fino** | Médio | Controller só traduz HTTP → DTO e chama use case; zero lógica de negócio |
| 8 | **Domínio Rico (Anti-Anêmico)** | Médio | Lógica de negócio fica no aggregate/domain service, não espalhada no use case |
| 9 | **Injeção por Construtor** | Médio | Dependências explícitas sempre; nunca `@Autowired` em campo |
| 10 | **@Transactional na Borda Correta** | Médio | `@Transactional` no `UseCaseImpl` ou `RepositoryImpl`, nunca no domain service |

Detalhes, exemplos de código e regras para agentes de IA: veja `references/principles.md`.

## Top 8 Violações Críticas

1. 🔴 **@Entity no domain object** — `@Entity @Table(name="account") class Account` em `domain/` → mover para `infrastructure/persistence/AccountJpaEntity`
2. 🔴 **Spring no domínio** — `@Service`, `@Component`, `@Autowired` em `domain/` → remover; domínio é Java puro
3. 🔴 **Repository injetado no controller** — `AccountController(@Autowired AccountRepository repo)` → controller injeta use case, nunca repository
4. 🔴 **Use case importando infrastructure** — `import br.com.itau.service.infrastructure.*` em `application/` → quebra a regra de dependência; use interface/porta
5. 🟠 **Lógica de negócio no controller** — validações, cálculos, decisões em `@RestController` → mover para use case ou domain service
6. 🟠 **Domain object exposto via HTTP** — `ResponseEntity<Account>` em vez de `ResponseEntity<AccountResponse>` → sempre mapear para DTO de apresentação
7. 🟠 **Mapper ausente** — `RepositoryImpl` convertendo entidade JPA para domain object inline → extrair para `<Aggregate>EntityMapper`
8. 🟠 **@Transactional no domain service** — domínio não gerencia transações (não conhece persistence) → mover para `UseCaseImpl`

## Decision Tree: Qual Referência Carregar

```
TAREFA                                          → REFERÊNCIA
────────────────────────────────────────────────────────────────
Criar um novo microsserviço do zero             → references/service-scaffolding.md (Parte 1)
Criar um novo aggregate em serviço existente    → references/service-scaffolding.md (Parte 2)
Avaliar compliance de Clean Architecture        → references/verification.md
Identificar violação específica de camada       → references/verification.md (Seção 2)
Entender um princípio em detalhe               → references/principles.md
Scoring de maturidade de um serviço            → references/verification.md (Seção 3)
```

## Instruções por Caso de Uso

### Criando um Novo Microsserviço

Carregue `references/service-scaffolding.md` — Parte 1.

Processo:
1. Levantar requisitos (nome do serviço, aggregates, integrações externas, banco de dados)
2. Definir aggregates e suas responsabilidades
3. Gerar estrutura de pacotes → domain entities → repository interfaces → use cases → infrastructure (JPA entities + mappers + repository impls) → controllers + DTOs → testes unitários → configuração Spring
4. Rodar comandos de verificação de `references/verification.md`

### Adicionando um Aggregate a Serviço Existente

Carregue `references/service-scaffolding.md` — Parte 2.

Seguir o mesmo processo de scaffold, mas verificar antes se o novo aggregate pertence ao bounded context do serviço atual ou deveria ser um serviço separado.

### Avaliando Compliance de Clean Architecture

Carregue `references/verification.md`.

Rodar todos os comandos de detecção, pontuar cada princípio. Produzir relatório priorizado com P0 (crítico), P1 (alto), P2 (médio).

### Entendendo um Princípio Específico

Carregue `references/principles.md`.

Cada princípio inclui: definição, por que importa, regras para agentes de IA, e exemplo de código correto vs. violação.

## Quick Anti-Pattern Check

Antes de gerar qualquer código, verificar:

**Domain (`domain/`):**
- [ ] Nenhuma anotação Spring (`@Service`, `@Component`, `@Repository`, `@Autowired`)
- [ ] Nenhuma anotação JPA (`@Entity`, `@Table`, `@Column`, `@Id`)
- [ ] Repository é **interface** apenas (nunca implementação)
- [ ] Exceções estendem `RuntimeException` ou classes de domínio, não `HttpStatus`
- [ ] Aggregate root tem estado de negócio real (não anêmico)

**Application (`application/`):**
- [ ] UseCase interface em `usecase/<aggregate>/`
- [ ] `UseCaseImpl` apenas orquestra (chama domain, chama repository port, mapeia DTOs)
- [ ] `@Transactional` está aqui (não em domain)
- [ ] Nenhum import de `infrastructure` ou `presentation`
- [ ] DTOs em `dto/<aggregate>/` (nunca expostos para `presentation/` diretamente)

**Infrastructure (`infrastructure/`):**
- [ ] `<Aggregate>JpaEntity` tem `@Entity`, **não** o domain object
- [ ] `<Aggregate>RepositoryImpl` implementa a interface de `domain/`
- [ ] `<Aggregate>EntityMapper` existe e converte nos dois sentidos
- [ ] Clients externos em `external/<partner>/` com interface separada da impl

**Presentation (`presentation/rest/`):**
- [ ] Controller injeta `UseCase` (nunca `Repository` ou `Service` direto)
- [ ] Retorna `Request`/`Response` DTOs próprios (nunca domain objects)
- [ ] Sem lógica de negócio — apenas tradução HTTP ↔ use case DTO
