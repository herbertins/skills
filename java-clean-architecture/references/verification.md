# Verification & Compliance Assessment

## Seção 1 — Comandos de Detecção

Execute a partir da raiz do projeto Java (`src/main/java/br/com/itau/<service>/`).

### Violações de P1 (Regra de Dependência)

```bash
# domain importando application (violação crítica)
grep -r "import br.com.itau.*.application\." src/main/java/br/com/itau/*/domain/

# domain importando infrastructure (violação crítica)
grep -r "import br.com.itau.*.infrastructure\." src/main/java/br/com/itau/*/domain/

# application importando infrastructure (violação)
grep -r "import br.com.itau.*.infrastructure\." src/main/java/br/com/itau/*/application/

# application importando presentation (violação)
grep -r "import br.com.itau.*.presentation\." src/main/java/br/com/itau/*/application/
```

### Violações de P2 (Pureza do Domínio)

```bash
# anotações Spring no domínio
grep -rn "@Service\|@Component\|@Autowired\|@Repository" src/main/java/br/com/itau/*/domain/

# anotações JPA no domínio
grep -rn "@Entity\|@Table\|@Column\|@Id\|@GeneratedValue\|@OneToMany\|@ManyToOne" \
  src/main/java/br/com/itau/*/domain/

# imports Spring/JPA no domínio
grep -rn "import org.springframework\|import jakarta.persistence" \
  src/main/java/br/com/itau/*/domain/
```

### Violações de P3 (Separação JPA)

```bash
# @Entity fora de infrastructure/persistence
grep -rn "@Entity" src/main/java/br/com/itau/ | grep -v "infrastructure/persistence"
```

### Violações de P5 (Fronteira de DTO)

```bash
# domain objects retornados diretamente por controllers (verificar manualmente tipos de retorno)
grep -rn "ResponseEntity<" src/main/java/br/com/itau/*/presentation/ | \
  grep -v "Request\|Response\|Dto"

# domain objects em assinatura de use case (exposição indevida)
grep -rn "import br.com.itau.*.domain\." src/main/java/br/com/itau/*/presentation/
```

### Violações de P6 (Use Case por Operação)

```bash
# serviços de aplicação genéricos com muitos métodos (candidatos a refatoração)
grep -rn "public.*UseCase\b" src/main/java/br/com/itau/*/application/usecase/ | wc -l
# se a contagem for baixa mas os serviços têm muitos métodos, investigar manualmente
```

### Violações de P7 (Controller Fino)

```bash
# controllers injetando repositories (violação grave)
grep -rn "Repository" src/main/java/br/com/itau/*/presentation/

# controllers com lógica de negócio (heurística — métodos muito longos)
awk '/class.*Controller/,0' src/main/java/br/com/itau/*/presentation/**/*Controller.java | \
  awk 'NF > 0' | wc -l
```

### Violações de P10 (@Transactional)

```bash
# @Transactional no domínio
grep -rn "@Transactional" src/main/java/br/com/itau/*/domain/
```

### Sumário rápido (rodar tudo de uma vez)

```bash
BASE="src/main/java/br/com/itau"
SERVICE="<service>"  # substitua pelo nome do serviço

echo "=== P1: domain → application/infrastructure ==="
grep -r "import br.com.itau.$SERVICE.application\.\|import br.com.itau.$SERVICE.infrastructure\." \
  $BASE/$SERVICE/domain/ 2>/dev/null | wc -l

echo "=== P2: Spring/JPA no domain ==="
grep -rn "@Service\|@Component\|@Entity\|@Autowired\|import org.springframework\|import jakarta.persistence" \
  $BASE/$SERVICE/domain/ 2>/dev/null | wc -l

echo "=== P3: @Entity fora de infrastructure ==="
grep -rn "@Entity" $BASE/$SERVICE/ 2>/dev/null | grep -v "infrastructure/persistence" | wc -l

echo "=== P7: Repository injetado em controller ==="
grep -rn "Repository" $BASE/$SERVICE/presentation/ 2>/dev/null | wc -l

echo "=== P10: @Transactional no domain ==="
grep -rn "@Transactional" $BASE/$SERVICE/domain/ 2>/dev/null | wc -l
```

---

## Seção 2 — Rubrica de Avaliação por Princípio

Para cada princípio, atribuir: ✅ Compliant | ⚠️ Partial | ❌ Violation

| Princípio | Como verificar | Peso |
|-----------|---------------|------|
| P1 Regra de Dependência | grep de imports cruzados (ver acima) | P0 (crítico) |
| P2 Pureza do Domínio | grep de anotações Spring/JPA em domain/ | P0 (crítico) |
| P3 Separação JPA | @Entity apenas em infrastructure/persistence | P0 (crítico) |
| P4 Repository como Porta | interface em domain/, impl em infrastructure/ | P1 (alto) |
| P5 Fronteira de DTO | controllers retornam Response/Dto, não domain objects | P1 (alto) |
| P6 Use Case por Operação | um UseCase interface por operação de negócio | P1 (alto) |
| P7 Controller Fino | controller não injeta repository; sem lógica de negócio | P1 (alto) |
| P8 Domínio Rico | aggregates têm comportamentos além de get/set | P2 (médio) |
| P9 Injeção por Construtor | sem @Autowired em campo | P2 (médio) |
| P10 @Transactional correto | anotação em UseCaseImpl ou RepositoryImpl | P2 (médio) |

---

## Seção 3 — Scoring de Maturidade

Calcular score baseado nas violações encontradas:

| Violações P0 | Violações P1 | Violações P2 | Score | Nível |
|-------------|-------------|-------------|-------|-------|
| 0 | 0 | 0 | 100 | 🟢 Compliant |
| 0 | 0-2 | 0-3 | 80-99 | 🟡 Minor Issues |
| 0 | 3+ | — | 60-79 | 🟠 Needs Work |
| 1+ | — | — | < 60 | 🔴 Critical Violations |

### Template de Relatório

```markdown
# Clean Architecture Compliance Report — <service-name>
Data: <data>

## Sumário Executivo
Score: XX/100 — Nível: [Compliant / Minor Issues / Needs Work / Critical Violations]

## Violações P0 (Críticas — resolver imediatamente)
- [ ] <arquivo>: <descrição da violação>

## Violações P1 (Altas — resolver neste sprint)
- [ ] <arquivo>: <descrição da violação>

## Violações P2 (Médias — backlog técnico)
- [ ] <arquivo>: <descrição da violação>

## Recomendações Priorizadas
1. <ação concreta>
2. <ação concreta>
```

---

## Seção 4 — Padrões de Estrutura de Arquivos (verificação manual)

### Estrutura esperada por aggregate

```
domain/<aggregate>/
  <Aggregate>.java                ✅ existe
  <Aggregate>Repository.java      ✅ é interface (não implements)
  <Aggregate>Exception.java       opcional mas recomendado

application/usecase/<aggregate>/
  <Op>UseCase.java                ✅ existe por operação de negócio
  <Op>UseCaseImpl.java            ✅ existe, implementa a interface

application/dto/<aggregate>/
  <Aggregate>InputDto.java        ✅ ou record equivalente
  <Aggregate>OutputDto.java       ✅ ou record equivalente

infrastructure/persistence/<aggregate>/
  <Aggregate>JpaEntity.java       ✅ tem @Entity
  <Aggregate>JpaRepository.java   ✅ extends JpaRepository
  <Aggregate>RepositoryImpl.java  ✅ implements domain/<Aggregate>Repository
  <Aggregate>EntityMapper.java    ✅ conversão em ambos os sentidos

presentation/rest/<aggregate>/
  <Aggregate>Controller.java      ✅ injeta UseCase (não Repository)
  <Aggregate>Request.java
  <Aggregate>Response.java
```

### Verificar ausência de arquivos errados

```bash
# Entity mapper ausente (possível conversão inline)
for agg in $(ls src/main/java/br/com/itau/<service>/infrastructure/persistence/); do
  if ! ls src/main/java/br/com/itau/<service>/infrastructure/persistence/$agg/*EntityMapper* 2>/dev/null; then
    echo "⚠️ EntityMapper ausente em: $agg"
  fi
done

# RepositoryImpl ausente (possível uso direto do Spring Data na camada errada)
for agg in $(ls src/main/java/br/com/itau/<service>/infrastructure/persistence/); do
  if ! ls src/main/java/br/com/itau/<service>/infrastructure/persistence/$agg/*RepositoryImpl* 2>/dev/null; then
    echo "⚠️ RepositoryImpl ausente em: $agg"
  fi
done
```
