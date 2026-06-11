# Service Scaffolding Guide

## Parte 1 — Novo Microsserviço do Zero

### Pré-requisitos (levantar com o usuário antes de gerar código)

1. **Nome do serviço** (ex: `account-service`) → define `br.com.itau.account`
2. **Aggregates** (ex: `Account`, `Transaction`) — lista completa e qual é o aggregate root de cada um
3. **Operações de negócio** por aggregate → uma operação = um use case
4. **Integrações externas** (outros microsserviços, filas, APIs externas)
5. **Banco de dados** (PostgreSQL, MySQL, MongoDB) → afeta `PersistenceConfig`

### Ordem de Geração

Gere nesta ordem — cada camada depende da anterior:

```
1. domain/<aggregate>/<Aggregate>.java               (aggregate root)
2. domain/<aggregate>/<Aggregate>Repository.java     (port interface)
3. domain/<aggregate>/<Aggregate>Exception.java      (exceções)
4. application/dto/<aggregate>/*Dto.java             (input/output DTOs)
5. application/usecase/<aggregate>/*UseCase.java     (interfaces de porta)
6. application/usecase/<aggregate>/*UseCaseImpl.java (implementações)
7. infrastructure/persistence/<aggregate>/*JpaEntity.java   (entidade JPA)
8. infrastructure/persistence/<aggregate>/*JpaRepository.java
9. infrastructure/persistence/<aggregate>/*EntityMapper.java
10. infrastructure/persistence/<aggregate>/*RepositoryImpl.java
11. infrastructure/config/PersistenceConfig.java
12. presentation/rest/<aggregate>/*Request.java
13. presentation/rest/<aggregate>/*Response.java
14. presentation/rest/<aggregate>/*Controller.java
15. Testes unitários para domain, use cases, mappers
16. application.properties / bootstrap.yml
```

---

### Templates por Arquivo

#### 1. Aggregate Root (`domain/<aggregate>/<Aggregate>.java`)

```java
package br.com.itau.<service>.domain.<aggregate>;

// Nenhum import Spring, JPA ou Jakarta aqui
public class <Aggregate> {

    private final <Aggregate>Id id;
    private <FieldType> <field>;
    // ... outros campos de negócio

    // Construtor com validação de invariantes
    public <Aggregate>(<Aggregate>Id id, <FieldType> <field>) {
        if (<field> == null) throw new <Aggregate>Exception("...");
        this.id = id;
        this.<field> = <field>;
    }

    // Comportamentos de negócio como métodos
    public void <operation>(<params>) {
        // lógica de negócio aqui
    }

    // Getters (sem setters públicos — estado muda por comportamentos)
    public <Aggregate>Id getId() { return id; }
    public <FieldType> get<Field>() { return <field>; }
}
```

> Aggregate root controla as invariantes do seu contexto. Se você precisa de setters públicos para tudo, o domínio está anêmico — mova a lógica de negócio do use case para cá.

#### 2. Repository Interface (`domain/<aggregate>/<Aggregate>Repository.java`)

```java
package br.com.itau.<service>.domain.<aggregate>;

import java.util.Optional;

public interface <Aggregate>Repository {
    <Aggregate> save(<Aggregate> <aggregate>);
    Optional<<Aggregate>> findById(<Aggregate>Id id);
    void delete(<Aggregate>Id id);
    // apenas operações que o domínio precisa — sem métodos JPA específicos
}
```

> Esta interface vive no domínio. A implementação (`RepositoryImpl`) fica em infrastructure. O domínio nunca sabe que existe JPA.

#### 3. Use Case Interface (`application/usecase/<aggregate>/<Operation>UseCase.java`)

```java
package br.com.itau.<service>.application.usecase.<aggregate>;

import br.com.itau.<service>.application.dto.<aggregate>.<Aggregate>InputDto;
import br.com.itau.<service>.application.dto.<aggregate>.<Aggregate>OutputDto;

public interface <Operation>UseCase {
    <Aggregate>OutputDto execute(<Aggregate>InputDto input);
}
```

#### 4. Use Case Implementation (`application/usecase/<aggregate>/<Operation>UseCaseImpl.java`)

```java
package br.com.itau.<service>.application.usecase.<aggregate>;

import br.com.itau.<service>.domain.<aggregate>.<Aggregate>;
import br.com.itau.<service>.domain.<aggregate>.<Aggregate>Repository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service  // única anotação Spring permitida aqui
public class <Operation>UseCaseImpl implements <Operation>UseCase {

    private final <Aggregate>Repository repository;

    public <Operation>UseCaseImpl(<Aggregate>Repository repository) {
        this.repository = repository;
    }

    @Override
    @Transactional  // @Transactional vive aqui, nunca no domain service
    public <Aggregate>OutputDto execute(<Aggregate>InputDto input) {
        // 1. Construir/buscar aggregate via domain factory ou repository
        // 2. Invocar comportamento no aggregate
        // 3. Persistir via repository interface
        // 4. Mapear resultado para OutputDto e retornar
    }
}
```

#### 5. JPA Entity (`infrastructure/persistence/<aggregate>/<Aggregate>JpaEntity.java`)

```java
package br.com.itau.<service>.infrastructure.persistence.<aggregate>;

import jakarta.persistence.*;

@Entity
@Table(name = "<service>_<aggregate>")  // prefixo do serviço evita colisão entre schemas
public class <Aggregate>JpaEntity {

    @Id
    @Column(name = "id")
    private String id;

    @Column(name = "<field>", nullable = false)
    private <FieldType> <field>;

    // Construtor sem args exigido pelo JPA (package-private ou protected)
    protected <Aggregate>JpaEntity() {}

    public <Aggregate>JpaEntity(String id, <FieldType> <field>) {
        this.id = id;
        this.<field> = <field>;
    }

    // Getters apenas
}
```

> Prefixar o nome da tabela com o serviço (`account_transaction`, não `transaction`) evita colisão caso vários serviços compartilhem schema ou venham a ser migrados para monolito.

#### 6. Entity Mapper (`infrastructure/persistence/<aggregate>/<Aggregate>EntityMapper.java`)

```java
package br.com.itau.<service>.infrastructure.persistence.<aggregate>;

import br.com.itau.<service>.domain.<aggregate>.<Aggregate>;
import org.springframework.stereotype.Component;

@Component
public class <Aggregate>EntityMapper {

    public <Aggregate>JpaEntity toJpaEntity(<Aggregate> domain) {
        return new <Aggregate>JpaEntity(
            domain.getId().value(),
            domain.get<Field>()
        );
    }

    public <Aggregate> toDomain(<Aggregate>JpaEntity entity) {
        return new <Aggregate>(
            new <Aggregate>Id(entity.getId()),
            entity.get<Field>()
        );
    }
}
```

> Nunca faça conversão inline dentro do `RepositoryImpl`. O mapper é testável isoladamente e serve como documentação viva do contrato entre as camadas.

#### 7. Repository Implementation (`infrastructure/persistence/<aggregate>/<Aggregate>RepositoryImpl.java`)

```java
package br.com.itau.<service>.infrastructure.persistence.<aggregate>;

import br.com.itau.<service>.domain.<aggregate>.<Aggregate>;
import br.com.itau.<service>.domain.<aggregate>.<Aggregate>Repository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public class <Aggregate>RepositoryImpl implements <Aggregate>Repository {

    private final <Aggregate>JpaRepository jpaRepository;
    private final <Aggregate>EntityMapper mapper;

    public <Aggregate>RepositoryImpl(<Aggregate>JpaRepository jpaRepository,
                                      <Aggregate>EntityMapper mapper) {
        this.jpaRepository = jpaRepository;
        this.mapper = mapper;
    }

    @Override
    public <Aggregate> save(<Aggregate> aggregate) {
        var entity = mapper.toJpaEntity(aggregate);
        var saved = jpaRepository.save(entity);
        return mapper.toDomain(saved);
    }

    @Override
    public Optional<<Aggregate>> findById(<Aggregate>Id id) {
        return jpaRepository.findById(id.value())
            .map(mapper::toDomain);
    }

    @Override
    public void delete(<Aggregate>Id id) {
        jpaRepository.deleteById(id.value());
    }
}
```

#### 8. Controller (`presentation/rest/<aggregate>/<Aggregate>Controller.java`)

```java
package br.com.itau.<service>.presentation.rest.<aggregate>;

import br.com.itau.<service>.application.usecase.<aggregate>.<Operation>UseCase;
import br.com.itau.<service>.application.dto.<aggregate>.<Aggregate>InputDto;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/<aggregates>")
public class <Aggregate>Controller {

    private final <Operation>UseCase <operation>UseCase;

    public <Aggregate>Controller(<Operation>UseCase <operation>UseCase) {
        this.<operation>UseCase = <operation>UseCase;
    }

    @PostMapping
    public ResponseEntity<<Aggregate>Response> create(@RequestBody <Aggregate>Request request) {
        var input = new <Aggregate>InputDto(request.get<Field>());
        var output = <operation>UseCase.execute(input);
        return ResponseEntity.ok(new <Aggregate>Response(output));
    }
}
```

---

## Parte 2 — Novo Aggregate em Serviço Existente

Antes de criar, responder:
1. O novo aggregate pertence ao mesmo bounded context do serviço? (Se não, criar novo serviço)
2. Ele tem lifecycle independente do aggregate principal? (Sim → pasta própria em `domain/`)
3. Quais use cases novos ele requer?

Seguir a mesma ordem de geração da Parte 1, mas apenas para o novo aggregate. Verificar que não há importação cruzada entre pastas de aggregates dentro de `domain/` — se houver necessidade de comunicação, usar eventos ou passar IDs como referência (nunca o objeto inteiro do outro aggregate).

---

## Checklist Final de Scaffold

```
[ ] domain/<aggregate>/<Aggregate>.java               — Java puro, sem anotações framework
[ ] domain/<aggregate>/<Aggregate>Repository.java     — interface apenas
[ ] application/usecase/<aggregate>/*UseCase.java     — interface por operação
[ ] application/usecase/<aggregate>/*UseCaseImpl.java — @Service + @Transactional aqui
[ ] application/dto/<aggregate>/*Dto.java             — sem anotações JPA/Jackson no domínio
[ ] infrastructure/persistence/<aggregate>/*JpaEntity — @Entity aqui, não no domínio
[ ] infrastructure/persistence/<aggregate>/*Mapper    — conversão explícita nos dois sentidos
[ ] infrastructure/persistence/<aggregate>/*Impl      — implementa interface de domain/
[ ] presentation/rest/<aggregate>/*Controller         — injeta UseCase, não Repository
[ ] presentation/rest/<aggregate>/*Request/*Response  — DTOs HTTP separados dos DTOs de app
[ ] Testes unitários para domain objects (sem Spring context)
[ ] Testes unitários para use case (mockando repository interface)
```
