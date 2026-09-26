# Conversa --- Specification, HATEOAS, JPA, paginação e MapStruct

## Usuário

Explique como funciona o spacification do João, com exemplo de uso

## Assistente

Se você está falando do **`Specification` do Spring Data JPA**, ele
serve para montar **filtros dinâmicos** de uma consulta sem precisar
criar dezenas de métodos no Repository.

A ideia é transformar cada regra de filtro em uma pequena especificação
e depois combiná-las com `and()` / `or()`.

### Exemplo

Imagine um `Employee`:

``` java
@Entity
public class Employee {

    private Long id;
    private String nome;
    private String matricula;
    private String departamento;
    private Boolean ativo;
}
```

E você quer pesquisar:

-   matrícula
-   nome
-   departamento
-   somente ativos

Em vez de criar:

``` java
findByMatricula(...)
findByNome(...)
findByDepartamento(...)
findByMatriculaAndNome(...)
findByMatriculaAndDepartamento(...)
findByNomeAndDepartamento(...)
...
```

você cria Specifications.

### 1. Specification

``` java
public class EmployeeSpecification {

    public static Specification<Employee> matricula(String matricula) {
        return (root, query, cb) ->
                cb.equal(root.get("matricula"), matricula);
    }

    public static Specification<Employee> nome(String nome) {
        return (root, query, cb) ->
                cb.like(
                    cb.lower(root.get("nome")),
                    "%" + nome.toLowerCase() + "%"
                );
    }

    public static Specification<Employee> departamento(String departamento) {
        return (root, query, cb) ->
                cb.equal(root.get("departamento"), departamento);
    }

    public static Specification<Employee> ativo() {
        return (root, query, cb) ->
                cb.isTrue(root.get("ativo"));
    }
}
```

O ponto importante é entender esta parte:

``` java
(root, query, cb) -> ...
```

Ela representa uma **condição que será transformada em SQL**.

Por exemplo:

``` java
cb.equal(root.get("matricula"), matricula)
```

vira conceitualmente:

``` sql
WHERE matricula = ?
```

------------------------------------------------------------------------

## 2. Repository

O Repository precisa implementar `JpaSpecificationExecutor`:

``` java
public interface EmployeeRepository
        extends JpaRepository<Employee, Long>,
                JpaSpecificationExecutor<Employee> {
}
```

Agora ele possui:

``` java
findAll(Specification<Employee> specification)
```

------------------------------------------------------------------------

## 3. Montando a consulta dinamicamente

Imagine que chegou esta requisição:

``` text
GET /employees?
    nome=Joao&
    departamento=TI&
    ativo=true
```

No Service:

``` java
public List<Employee> buscar(
        String nome,
        String departamento,
        Boolean ativo) {

    Specification<Employee> specification = null;

    if (nome != null) {
        specification =
            Specification.where(
                EmployeeSpecification.nome(nome)
            );
    }

    if (departamento != null) {
        specification =
            specification.and(
                EmployeeSpecification.departamento(departamento)
            );
    }

    if (Boolean.TRUE.equals(ativo)) {
        specification =
            specification.and(
                EmployeeSpecification.ativo()
            );
    }

    return employeeRepository.findAll(specification);
}
```

O resultado será conceitualmente:

``` sql
SELECT *
FROM employee
WHERE
    LOWER(nome) LIKE '%joao%'
    AND departamento = 'TI'
    AND ativo = true;
```

### A grande vantagem

Se amanhã vier:

``` text
GET /employees?matricula=12345
```

você usa somente:

``` java
Specification<Employee> specification =
        EmployeeSpecification.matricula("12345");

return employeeRepository.findAll(specification);
```

Se vier:

``` text
GET /employees?
    nome=Joao&
    departamento=TI
```

combina:

``` java
Specification<Employee> specification =
        EmployeeSpecification.nome("Joao")
            .and(EmployeeSpecification.departamento("TI"));
```

------------------------------------------------------------------------

# O conceito mais importante

Pense em `Specification` como uma **peça de filtro**.

``` text
Specification
      │
      ├── nome = João
      │
      ├── departamento = TI
      │
      └── ativo = true
```

Você pode montar:

``` text
nome = João
       AND
departamento = TI
       AND
ativo = true
```

ou:

``` text
departamento = TI
       OR
departamento = RH
```

usando:

``` java
EmployeeSpecification
    .departamento("TI")
    .or(EmployeeSpecification.departamento("RH"));
```

------------------------------------------------------------------------

## E por que isso combina muito bem com sua arquitetura hexagonal?

No seu caso, eu **não colocaria `Specification` no domínio** se ela
depender do JPA.

Eu deixaria algo assim:

``` text
domain
 └── Employee

application
 ├── input
 │    └── BuscarEmployeeUseCase
 └── output
      └── BuscarEmployeeOutput

application/usecase
 └── BuscarEmployeeService

infra
 └── persistence
      ├── EmployeeEntity
      ├── EmployeeRepository
      └── specification
           └── EmployeeSpecification
```

Assim:

``` text
Controller
    ↓
Input / Use Case
    ↓
Service
    ↓
Output
    ↓
Persistence
    ↓
Specification + JPA
    ↓
Banco
```

A **Specification é uma preocupação da persistência**, porque ela está
dizendo ao JPA **como montar a consulta**.

Se quiser, posso mostrar uma versão **bem mais limpa**, usando
`Specification` com seu exemplo de **Employee por matrícula**, seguindo
exatamente aquela arquitetura hexagonal simplificada que estávamos
montando.

------------------------------------------------------------------------

## Usuário

Sim

## Assistente

Perfeito. Vou manter **a arquitetura hexagonal simples** que estávamos
usando, sem criar camadas desnecessárias.

A ideia será:

``` text
Controller
    ↓
Input
    ↓
UseCase
    ↓
Service
    ↓
Output
    ↓
Repository
    ↓
Specification
    ↓
JPA
```

### 1. Input

``` java
public interface BuscarEmployeeInput {

    Employee buscarPorMatricula(String matricula);
}
```

### 2. Use Case / Service

O próprio Service implementa o Input e usa o Output:

``` java
@Service
public class BuscarEmployeeService implements BuscarEmployeeInput {

    private final BuscarEmployeeOutput output;

    public BuscarEmployeeService(BuscarEmployeeOutput output) {
        this.output = output;
    }

    @Override
    public Employee buscarPorMatricula(String matricula) {

        return output.buscarPorMatricula(matricula);
    }
}
```

Aqui continua simples: o **Use Case não conhece JPA**.

------------------------------------------------------------------------

### 3. Output

``` java
public interface BuscarEmployeeOutput {

    Employee buscarPorMatricula(String matricula);
}
```

------------------------------------------------------------------------

### 4. Adapter de persistência

Agora entra o JPA:

``` java
@Component
public class EmployeePersistenceAdapter
        implements BuscarEmployeeOutput {

    private final EmployeeRepository repository;

    public EmployeePersistenceAdapter(EmployeeRepository repository) {
        this.repository = repository;
    }

    @Override
    public Employee buscarPorMatricula(String matricula) {

        return repository
                .findOne(EmployeeSpecification.matricula(matricula))
                .orElseThrow();
    }
}
```

Perceba a separação:

``` text
Service
   ↓
BuscarEmployeeOutput
   ↓
PersistenceAdapter
   ↓
EmployeeRepository
   ↓
Specification
```

------------------------------------------------------------------------

# 5. Repository

Aqui está o detalhe importante:

``` java
public interface EmployeeRepository
        extends JpaRepository<EmployeeEntity, Long>,
                JpaSpecificationExecutor<EmployeeEntity> {
}
```

O `JpaSpecificationExecutor` fornece:

``` java
findOne(...)
findAll(...)
count(...)
exists(...)
```

aceitando `Specification`.

------------------------------------------------------------------------

# 6. Specification

Agora podemos criar:

``` java
public final class EmployeeSpecification {

    private EmployeeSpecification() {
    }

    public static Specification<EmployeeEntity> matricula(
            String matricula) {

        return (root, query, cb) ->
                cb.equal(
                    root.get("matricula"),
                    matricula
                );
    }
}
```

Isso:

``` java
EmployeeSpecification.matricula("12345")
```

representa algo equivalente a:

``` sql
WHERE matricula = '12345'
```

------------------------------------------------------------------------

# 7. E aqui aparece a vantagem

Hoje você precisa somente de matrícula:

``` java
repository.findOne(
    EmployeeSpecification.matricula(matricula)
);
```

Mas amanhã pode precisar:

``` text
matrícula
nome
departamento
cargo
ativo
```

Você pode criar:

``` java
public static Specification<EmployeeEntity> nome(String nome) {
    return (root, query, cb) ->
            cb.like(
                cb.lower(root.get("nome")),
                "%" + nome.toLowerCase() + "%"
            );
}
```

E:

``` java
public static Specification<EmployeeEntity> ativo(Boolean ativo) {
    return (root, query, cb) ->
            cb.equal(root.get("ativo"), ativo);
}
```

Então pode combinar:

``` java
Specification<EmployeeEntity> spec =
        Specification
            .where(EmployeeSpecification.nome("João"))
            .and(EmployeeSpecification.ativo(true));
```

Resultado:

``` sql
WHERE
    LOWER(nome) LIKE '%joão%'
    AND ativo = true
```

------------------------------------------------------------------------

## Mas para o seu caso eu faria ainda mais simples

Se o endpoint é **exatamente "buscar Employee por matrícula"**, eu **não
usaria Specification ainda**.

Faria:

``` java
public interface EmployeeRepository
        extends JpaRepository<EmployeeEntity, Long> {

    Optional<EmployeeEntity> findByMatricula(String matricula);
}
```

E o adapter:

``` java
@Component
public class EmployeePersistenceAdapter
        implements BuscarEmployeeOutput {

    private final EmployeeRepository repository;

    @Override
    public Employee buscarPorMatricula(String matricula) {

        return repository
                .findByMatricula(matricula)
                .map(EmployeeMapper::toDomain)
                .orElseThrow();
    }
}
```

Isso é mais simples.

### Quando `Specification` passa a valer a pena?

Quando seu endpoint começa a aceitar **filtros opcionais e
combináveis**:

``` text
GET /employees?
    matricula=123&
    nome=Joao&
    departamento=TI&
    ativo=true
```

Aí a Specification começa a resolver um problema real:

``` text
           ┌── matrícula
           ├── nome
Request ───┼── departamento ──→ Specification
           ├── cargo
           └── ativo
                         ↓
                       JPA
```

**Resumo:** `Specification` não é uma obrigação da arquitetura
hexagonal. É uma ferramenta de **persistência para consultas
dinâmicas**. Para uma busca simples por matrícula, `findByMatricula()` é
mais limpo. Para filtros combináveis, `Specification` passa a fazer
sentido.

------------------------------------------------------------------------

## Usuário

Consigo usar o byId, em um metodo e ou com spacification,

``` java
public interface EmployeeRepository
        extends JpaRepository<EmployeeEntity, Long> {

    Optional<EmployeeEntity> findByMatricula(String matricula);
}

Utilizando o mesmo repositório?
```

## Assistente

Sim. **Você pode usar os dois no mesmo `EmployeeRepository`**. Na
verdade, é bastante comum.

Como `JpaRepository` já possui `findById()`, você não precisa declarar
esse método.

``` java
public interface EmployeeRepository
        extends JpaRepository<EmployeeEntity, Long>,
                JpaSpecificationExecutor<EmployeeEntity> {

    Optional<EmployeeEntity> findByMatricula(String matricula);
}
```

Você passa a ter **três formas de consulta no mesmo Repository**:

### 1. Por ID

Herdado do `JpaRepository`:

``` java
Optional<EmployeeEntity> employee =
        repository.findById(id);
```

Internamente:

``` sql
SELECT *
FROM employee
WHERE id = ?
```

------------------------------------------------------------------------

### 2. Por matrícula

Método derivado do Spring Data:

``` java
Optional<EmployeeEntity> employee =
        repository.findByMatricula(matricula);
```

Gera conceitualmente:

``` sql
SELECT *
FROM employee
WHERE matricula = ?
```

------------------------------------------------------------------------

### 3. Por Specification

Como o Repository implementa:

``` java
JpaSpecificationExecutor<EmployeeEntity>
```

você pode fazer:

``` java
Optional<EmployeeEntity> employee =
        repository.findOne(
            EmployeeSpecification.matricula(matricula)
        );
```

Ou uma consulta mais complexa:

``` java
Specification<EmployeeEntity> specification =
        Specification
            .where(EmployeeSpecification.nome("João"))
            .and(EmployeeSpecification.ativo(true));

List<EmployeeEntity> employees =
        repository.findAll(specification);
```

------------------------------------------------------------------------

## Então o mesmo Repository fica assim

``` java
public interface EmployeeRepository
        extends JpaRepository<EmployeeEntity, Long>,
                JpaSpecificationExecutor<EmployeeEntity> {

    Optional<EmployeeEntity> findByMatricula(String matricula);
}
```

E você pode usar:

``` java
repository.findById(id);
```

ou:

``` java
repository.findByMatricula(matricula);
```

ou:

``` java
repository.findOne(specification);
```

ou:

``` java
repository.findAll(specification);
```

### Eu seguiria esta regra

  Necessidade                           Usaria
  ------------------------------------- ---------------------
  Buscar por ID                         `findById()`
  Buscar por uma propriedade simples    `findByMatricula()`
  Vários filtros opcionais              `Specification`
  Combinar `AND` / `OR` dinamicamente   `Specification`
  Consulta muito específica/complexa    Query própria

Ou seja, **não precisa transformar tudo em Specification**.

Para seu projeto, eu deixaria o Repository exatamente dessa forma e
usaria cada mecanismo conforme a necessidade. Isso mantém a arquitetura
simples sem perder flexibilidade.

------------------------------------------------------------------------

## Usuário

Explique sobre Hateos

## Assistente

**HATEOAS** significa **Hypermedia As The Engine Of Application State**.
É uma abordagem para APIs REST em que a própria resposta informa ao
cliente **quais ações/recursos podem ser acessados a partir daquele
estado**.

A ideia principal é:

> Em vez de o cliente precisar conhecer todas as URLs da API
> antecipadamente, a resposta fornece links para os próximos passos
> possíveis.

### Sem HATEOAS

Você poderia ter:

``` json
{
  "id": 10,
  "nome": "João",
  "matricula": "12345"
}
```

O frontend sabe que, para buscar contratos, deve chamar:

``` text
GET /employees/10/contratos
```

Mas essa URL foi **codificada no cliente**.

------------------------------------------------------------------------

### Com HATEOAS

A resposta poderia ser:

``` json
{
  "id": 10,
  "nome": "João",
  "matricula": "12345",
  "_links": {
    "self": {
      "href": "/employees/10"
    },
    "contratos": {
      "href": "/employees/10/contratos"
    }
  }
}
```

Agora o cliente recebe:

``` text
Employee
   │
   ├── self
   │     GET /employees/10
   │
   └── contratos
         GET /employees/10/contratos
```

O servidor está dizendo:

> "Este Employee existe e, a partir dele, você pode consultar os
> contratos."

------------------------------------------------------------------------

# No Spring Boot

O Spring possui suporte através do **Spring HATEOAS**.

Por exemplo:

``` java
public class EmployeeResponse extends RepresentationModel<EmployeeResponse> {

    private Long id;
    private String nome;
    private String matricula;
}
```

No Controller:

``` java
@GetMapping("/{id}")
public EntityModel<EmployeeResponse> buscar(@PathVariable Long id) {

    EmployeeResponse employee = service.buscar(id);

    return EntityModel.of(
        employee,
        linkTo(
            methodOn(EmployeeController.class)
                .buscar(id)
        ).withSelfRel(),

        linkTo(
            methodOn(EmployeeController.class)
                .listarContratos(id)
        ).withRel("contratos")
    );
}
```

A resposta pode ficar:

``` json
{
  "id": 10,
  "nome": "João",
  "matricula": "12345",
  "_links": {
    "self": {
      "href": "http://localhost:8080/employees/10"
    },
    "contratos": {
      "href": "http://localhost:8080/employees/10/contratos"
    }
  }
}
```

------------------------------------------------------------------------

# E existe uma ideia ainda mais interessante

Imagine que o Employee esteja **ativo**.

A API pode retornar:

``` json
{
  "id": 10,
  "nome": "João",
  "ativo": true,
  "_links": {
    "self": {
      "href": "/employees/10"
    },
    "contratos": {
      "href": "/employees/10/contratos"
    },
    "inativar": {
      "href": "/employees/10",
      "method": "PATCH"
    }
  }
}
```

Se estiver inativo:

``` json
{
  "id": 10,
  "nome": "João",
  "ativo": false,
  "_links": {
    "self": {
      "href": "/employees/10"
    },
    "ativar": {
      "href": "/employees/10",
      "method": "PATCH"
    }
  }
}
```

Ou seja, os **links disponíveis podem representar as transições de
estado possíveis**.

É justamente daí que vem:

``` text
Hypermedia
    ↓
Engine
    ↓
of Application
    ↓
State
```

------------------------------------------------------------------------

# HATEOAS não é simplesmente "colocar links"

Esse é um ponto importante.

Um JSON assim:

``` json
{
  "id": 10,
  "url": "/employees/10"
}
```

não significa necessariamente que você esteja aplicando HATEOAS de forma
completa.

A ideia é que os links tenham **relações semânticas**:

``` json
"_links": {
  "self": "...",
  "contratos": "...",
  "dependentes": "...",
  "historico": "..."
}
```

Essas relações (`rel`) dizem **o que aquele link representa**.

------------------------------------------------------------------------

## No seu projeto hexagonal

Eu separaria assim:

``` text
                ┌── HATEOAS
                │
Controller ─────┤
                ↓
             Input
                ↓
             UseCase
                ↓
             Service
                ↓
             Output
                ↓
          Persistence
```

Isso é importante porque **HATEOAS é uma preocupação da API HTTP**, não
do domínio.

Seu `Employee` de domínio não deveria precisar saber que existe:

``` java
RepresentationModel
EntityModel
Link
```

O domínio continua limpo.

### Em resumo

``` text
REST tradicional:

Cliente → "Eu sei que existe /employees/10/contratos"


HATEOAS:

Cliente → "Me dê o Employee"
             ↓
API       → "Aqui está o Employee
             e estes são os recursos
             que você pode acessar."
```

Para a arquitetura hexagonal que você está montando, **HATEOAS pode
ficar restrito ao Controller/adapter HTTP**, mantendo `domain`,
`application` e seus `use cases` independentes do Spring HATEOAS.

------------------------------------------------------------------------

## Usuário

Crie a implementação do hateos

## Assistente

Claro. Vou montar uma implementação **simples e completa de HATEOAS**
para o seu caso de `Employee`, mantendo a arquitetura hexagonal que
estamos usando.

A ideia será:

``` text
HTTP
 ↓
Controller
 ↓
Input / UseCase
 ↓
Service
 ↓
Output
 ↓
Persistence
```

E o **HATEOAS fica somente no Controller**, sem contaminar o domínio.

## 1. Dependência

No `pom.xml`:

``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-hateoas</artifactId>
</dependency>
```

------------------------------------------------------------------------

# 2. Response

Não colocaria HATEOAS diretamente no seu `Employee` de domínio.

Crie um DTO:

``` java
public record EmployeeResponse(
        Long id,
        String matricula,
        String nome
) {
}
```

------------------------------------------------------------------------

# 3. Controller

Aqui começa a implementação do HATEOAS:

``` java
@RestController
@RequestMapping("/employees")
public class EmployeeController {

    private final BuscarEmployeeInput input;

    public EmployeeController(BuscarEmployeeInput input) {
        this.input = input;
    }

    @GetMapping("/{id}")
    public EntityModel<EmployeeResponse> buscar(
            @PathVariable Long id) {

        Employee employee = input.buscarPorId(id);

        EmployeeResponse response = new EmployeeResponse(
                employee.getId(),
                employee.getMatricula(),
                employee.getNome()
        );

        return EntityModel.of(
                response,

                linkTo(
                        methodOn(EmployeeController.class)
                                .buscar(id)
                ).withSelfRel(),

                linkTo(
                        methodOn(EmployeeController.class)
                                .buscarPorMatricula(employee.getMatricula())
                ).withRel("por-matricula")
        );
    }

    @GetMapping("/matricula/{matricula}")
    public EntityModel<EmployeeResponse> buscarPorMatricula(
            @PathVariable String matricula) {

        Employee employee =
                input.buscarPorMatricula(matricula);

        EmployeeResponse response = new EmployeeResponse(
                employee.getId(),
                employee.getMatricula(),
                employee.getNome()
        );

        return EntityModel.of(
                response,

                linkTo(
                        methodOn(EmployeeController.class)
                                .buscar(employee.getId())
                ).withSelfRel()
        );
    }
}
```

Agora a resposta:

``` json
{
  "id": 10,
  "matricula": "12345",
  "nome": "João",
  "_links": {
    "self": {
      "href": "http://localhost:8080/employees/10"
    },
    "por-matricula": {
      "href": "http://localhost:8080/employees/matricula/12345"
    }
  }
}
```

------------------------------------------------------------------------

# 4. O que está acontecendo?

Esta parte:

``` java
linkTo(
    methodOn(EmployeeController.class)
        .buscar(id)
).withSelfRel()
```

cria:

``` json
"self": {
    "href": "/employees/10"
}
```

E:

``` java
linkTo(
    methodOn(EmployeeController.class)
        .buscarPorMatricula(employee.getMatricula())
).withRel("por-matricula")
```

cria:

``` json
"por-matricula": {
    "href": "/employees/matricula/12345"
}
```

O interessante é que você **não precisa escrever a URL manualmente**:

``` java
"/employees/" + id
```

O Spring monta a URL a partir do Controller.

------------------------------------------------------------------------

# 5. Podemos criar um endpoint de contratos

Imagine:

``` java
@GetMapping("/{id}/contratos")
public List<ContratoResponse> contratos(
        @PathVariable Long id) {

    return ...
}
```

Agora podemos colocar esse link no Employee:

``` java
return EntityModel.of(
        response,

        linkTo(
                methodOn(EmployeeController.class)
                        .buscar(id)
        ).withSelfRel(),

        linkTo(
                methodOn(EmployeeController.class)
                        .contratos(id)
        ).withRel("contratos")
);
```

A resposta:

``` json
{
  "id": 10,
  "matricula": "12345",
  "nome": "João",

  "_links": {
    "self": {
      "href": "/employees/10"
    },
    "contratos": {
      "href": "/employees/10/contratos"
    }
  }
}
```

------------------------------------------------------------------------

# 6. Uma melhoria importante

Se você começar a ter muitos Controllers, colocar toda essa construção
de links dentro do método pode deixar o Controller grande.

Você pode criar um **Assembler**:

``` java
@Component
public class EmployeeModelAssembler
        implements RepresentationModelAssembler<EmployeeResponse,
                                               EntityModel<EmployeeResponse>> {

    @Override
    public EntityModel<EmployeeResponse> toModel(
            EmployeeResponse employee) {

        return EntityModel.of(
                employee,

                linkTo(
                        methodOn(EmployeeController.class)
                                .buscar(employee.id())
                ).withSelfRel(),

                linkTo(
                        methodOn(EmployeeController.class)
                                .buscarPorMatricula(
                                        employee.matricula()
                                )
                ).withRel("por-matricula")
        );
    }
}
```

Então o Controller fica bem mais limpo:

``` java
@RestController
@RequestMapping("/employees")
public class EmployeeController {

    private final BuscarEmployeeInput input;
    private final EmployeeModelAssembler assembler;

    public EmployeeController(
            BuscarEmployeeInput input,
            EmployeeModelAssembler assembler) {

        this.input = input;
        this.assembler = assembler;
    }

    @GetMapping("/{id}")
    public EntityModel<EmployeeResponse> buscar(
            @PathVariable Long id) {

        Employee employee = input.buscarPorId(id);

        EmployeeResponse response =
                new EmployeeResponse(
                        employee.getId(),
                        employee.getMatricula(),
                        employee.getNome()
                );

        return assembler.toModel(response);
    }
}
```

### Eu usaria essa segunda abordagem no seu projeto

Fica:

``` text
adapter/in
│
├── EmployeeController
│
├── EmployeeResponse
│
└── EmployeeModelAssembler
        │
        └── HATEOAS
```

Enquanto o restante permanece:

``` text
application
    ↓
domain
    ↓
application/output
    ↓
adapter/out
```

Assim o **HATEOAS fica completamente isolado na camada HTTP**, que é
exatamente o que queremos em uma arquitetura hexagonal limpa.

------------------------------------------------------------------------

## Usuário

Implemente o GetAll, com off-set e size

Vao retornar um hateos, com size

Faça em java

## Assistente

Sim. Vamos fazer um `GET /employees?offset=0&size=10` retornando
**HATEOAS + paginação**, mantendo a implementação simples.

Vou usar `PagedModel`, que é o formato apropriado do Spring HATEOAS para
coleções paginadas.

### Controller

``` java
@RestController
@RequestMapping("/employees")
public class EmployeeController {

    private final BuscarEmployeeInput input;
    private final EmployeeModelAssembler assembler;

    public EmployeeController(
            BuscarEmployeeInput input,
            EmployeeModelAssembler assembler) {

        this.input = input;
        this.assembler = assembler;
    }

    @GetMapping
    public PagedModel<EntityModel<EmployeeResponse>> getAll(
            @RequestParam(defaultValue = "0") int offset,
            @RequestParam(defaultValue = "10") int size) {

        Page<Employee> page = input.getAll(offset, size);

        return assembler.toModel(page);
    }
}
```

### Input

``` java
public interface BuscarEmployeeInput {

    Page<Employee> getAll(int offset, int size);
}
```

### Use Case / Service

``` java
@Service
public class BuscarEmployeeService
        implements BuscarEmployeeInput {

    private final BuscarEmployeeOutput output;

    public BuscarEmployeeService(BuscarEmployeeOutput output) {
        this.output = output;
    }

    @Override
    public Page<Employee> getAll(int offset, int size) {

        return output.getAll(offset, size);
    }
}
```

### Output

``` java
public interface BuscarEmployeeOutput {

    Page<Employee> getAll(int offset, int size);
}
```

### Adapter de persistência

Aqui convertemos `offset` + `size` para `Pageable`:

``` java
@Component
public class EmployeePersistenceAdapter
        implements BuscarEmployeeOutput {

    private final EmployeeRepository repository;

    public EmployeePersistenceAdapter(
            EmployeeRepository repository) {

        this.repository = repository;
    }

    @Override
    public Page<Employee> getAll(int offset, int size) {

        Pageable pageable =
                PageRequest.of(
                        offset / size,
                        size
                );

        return repository
                .findAll(pageable)
                .map(EmployeeMapper::toDomain);
    }
}
```

### Repository

``` java
public interface EmployeeRepository
        extends JpaRepository<EmployeeEntity, Long> {
}
```

------------------------------------------------------------------------

## Assembler do HATEOAS

``` java
@Component
public class EmployeeModelAssembler
        implements RepresentationModelAssembler<EmployeeResponse,
                                               EntityModel<EmployeeResponse>> {

    @Override
    public EntityModel<EmployeeResponse> toModel(
            EmployeeResponse employee) {

        return EntityModel.of(
                employee,

                linkTo(
                        methodOn(EmployeeController.class)
                                .buscar(employee.id())
                ).withSelfRel()
        );
    }

    public PagedModel<EntityModel<EmployeeResponse>> toModel(
            Page<Employee> page) {

        List<EntityModel<EmployeeResponse>> content =
                page.getContent()
                        .stream()
                        .map(this::toResponse)
                        .map(this::toModel)
                        .toList();

        return PagedModel.of(
                content,
                new PagedModel.PageMetadata(
                        page.getSize(),
                        page.getNumber(),
                        page.getTotalElements(),
                        page.getTotalPages()
                ),

                linkTo(
                        methodOn(EmployeeController.class)
                                .getAll(
                                        0,
                                        page.getSize()
                                )
                ).withRel("first"),

                linkTo(
                        methodOn(EmployeeController.class)
                                .getAll(
                                        Math.max(
                                                0,
                                                (page.getNumber() - 1)
                                                        * page.getSize()
                                        ),
                                        page.getSize()
                                )
                ).withRel("prev"),

                linkTo(
                        methodOn(EmployeeController.class)
                                .getAll(
                                        (page.getNumber() + 1)
                                                * page.getSize(),
                                        page.getSize()
                                )
                ).withRel("next")
        );
    }

    private EmployeeResponse toResponse(Employee employee) {

        return new EmployeeResponse(
                employee.getId(),
                employee.getMatricula(),
                employee.getNome()
        );
    }
}
```

### DTO

``` java
public record EmployeeResponse(
        Long id,
        String matricula,
        String nome
) {
}
```

------------------------------------------------------------------------

## Porém, há um detalhe importante

`PageRequest` trabalha com **número da página**, enquanto sua API está
usando **offset**.

Por exemplo:

``` text
offset = 0
size   = 10
```

significa:

``` text
registros 0 → 9
```

Já:

``` text
offset = 10
size   = 10
```

significa:

``` text
registros 10 → 19
```

Por isso:

``` java
PageRequest.of(offset / size, size)
```

funciona quando o `offset` é múltiplo do `size`.

------------------------------------------------------------------------

## Resposta

Para:

``` http
GET /employees?offset=0&size=2
```

poderíamos ter:

``` json
{
  "_embedded": {
    "employees": [
      {
        "id": 1,
        "matricula": "1001",
        "nome": "João",
        "_links": {
          "self": {
            "href": "/employees/1"
          }
        }
      },
      {
        "id": 2,
        "matricula": "1002",
        "nome": "Maria",
        "_links": {
          "self": {
            "href": "/employees/2"
          }
        }
      }
    ]
  },
  "_links": {
    "first": {
      "href": "/employees?offset=0&size=2"
    },
    "next": {
      "href": "/employees?offset=2&size=2"
    }
  },
  "page": {
    "size": 2,
    "totalElements": 10,
    "totalPages": 5,
    "number": 0
  }
}
```

A estrutura fica:

``` text
GET /employees?offset=0&size=2
                │
                ▼
        ┌────────────────┐
        │   Controller   │
        └───────┬────────┘
                ▼
             Input
                ▼
             Service
                ▼
             Output
                ▼
        PersistenceAdapter
                ▼
             JPA
                │
                ▼
             Page<Employee>
                │
                ▼
        EmployeeAssembler
                │
                ▼
          PagedModel
                │
                ▼
             HATEOAS
```

**Observação:** para uma API realmente baseada em `offset/size`, eu
faria uma pequena adaptação no assembler para **não gerar o `prev`
quando estiver no primeiro bloco e não gerar `next` quando estiver no
último**. Isso evita links inválidos como `next` além da última página.

------------------------------------------------------------------------

## Usuário

Como ficaria no jpa

## Assistente

No JPA você **não precisa implementar `OFFSET` e `LIMIT` manualmente**.
O Spring Data JPA faz isso através de `Pageable`.

### Repository

``` java
public interface EmployeeRepository
        extends JpaRepository<EmployeeEntity, Long> {
}
```

O `JpaRepository` já possui:

``` java
Page<EmployeeEntity> findAll(Pageable pageable);
```

E você chama:

``` java
Page<EmployeeEntity> page =
        employeeRepository.findAll(
                PageRequest.of(0, 10)
        );
```

Isso gera algo equivalente a:

``` sql
SELECT
    *
FROM employee
LIMIT 10
OFFSET 0;
```

Para a segunda página:

``` java
Page<EmployeeEntity> page =
        employeeRepository.findAll(
                PageRequest.of(1, 10)
        );
```

equivale a:

``` sql
SELECT *
FROM employee
LIMIT 10
OFFSET 10;
```

------------------------------------------------------------------------

### Se quiser ordenar

É recomendável definir uma ordenação:

``` java
Pageable pageable = PageRequest.of(
    offset / size,
    size,
    Sort.by("id").ascending()
);
```

Conceitualmente:

``` sql
SELECT *
FROM employee
ORDER BY id ASC
LIMIT 10
OFFSET 20;
```

Isso é importante porque, sem `ORDER BY`, a ordem dos registros não é
garantida.

------------------------------------------------------------------------

### Mas existe uma questão no seu `offset`

Se você quer aceitar:

``` text
offset=25&size=10
```

o `PageRequest` **não representa diretamente esse offset**, porque ele
trabalha com página:

``` text
page = 2
size = 10
```

que significa:

``` text
OFFSET 20
```

e não `25`.

Se sua API realmente precisa aceitar **qualquer valor de offset**, por
exemplo:

``` text
GET /employees?offset=25&size=10
```

aí eu faria de outra forma no JPA, usando `EntityManager`/query com:

``` java
query.setFirstResult(offset);
query.setMaxResults(size);
```

Ficaria mais fiel ao contrato da API:

``` text
offset → setFirstResult()
size   → setMaxResults()
```

e o SQL gerado seria efetivamente:

``` sql
LIMIT 10 OFFSET 25
```

**Para uma API com `offset + size`, eu prefiro essa segunda abordagem**,
porque `offset` representa exatamente o deslocamento solicitado, sem a
conversão `offset / size`.

------------------------------------------------------------------------

## Usuário

Só mandar no repository?

## Assistente

Sim. **Se você está falando de deixar a paginação no próprio
Repository**, pode fazer de forma bem simples.

Com Spring Data JPA, o Repository já aceita `Pageable`:

``` java
public interface EmployeeRepository
        extends JpaRepository<EmployeeEntity, Long> {
}
```

E você chama:

``` java
Page<EmployeeEntity> page =
        employeeRepository.findAll(
                PageRequest.of(0, 10)
        );
```

Isso gera algo equivalente a:

``` sql
SELECT *
FROM employee
LIMIT 10
OFFSET 0;
```

Para a segunda página:

``` java
Page<EmployeeEntity> page =
        employeeRepository.findAll(
                PageRequest.of(1, 10)
        );
```

equivale a:

``` sql
SELECT *
FROM employee
LIMIT 10
OFFSET 10;
```

### Se quiser `offset` e `size` diretamente

O Spring Data JPA **não possui `findAll(offset, size)` nativo**. O
padrão é:

``` java
Pageable pageable = PageRequest.of(
    offset / size,
    size
);

repository.findAll(pageable);
```

Então, se seu contrato for:

``` http
GET /employees?offset=0&size=10
```

pode simplesmente converter antes de chamar o Repository.

Se você quer **literalmente `offset=25&size=10`**, sem precisar
transformar `offset` em número de página, aí precisa de uma
implementação customizada no Repository com `EntityManager`.

------------------------------------------------------------------------

## Usuário

Dar para usar com spacification?

## Assistente

Sim. **`Specification` + paginação é justamente uma combinação muito
comum no Spring Data JPA.**

O mesmo Repository pode ter:

``` java
public interface EmployeeRepository
        extends JpaRepository<EmployeeEntity, Long>,
                JpaSpecificationExecutor<EmployeeEntity> {
}
```

E você pode fazer:

``` java
Specification<EmployeeEntity> specification =
        EmployeeSpecification.ativo(true);

Pageable pageable =
        PageRequest.of(0, 10);

Page<EmployeeEntity> page =
        employeeRepository.findAll(
                specification,
                pageable
        );
```

O método utilizado é:

``` java
findAll(
    Specification<T> specification,
    Pageable pageable
)
```

### Exemplo completo

Specification:

``` java
public class EmployeeSpecification {

    public static Specification<EmployeeEntity> ativo(Boolean ativo) {
        return (root, query, cb) ->
                cb.equal(root.get("ativo"), ativo);
    }
}
```

Repository:

``` java
public interface EmployeeRepository
        extends JpaRepository<EmployeeEntity, Long>,
                JpaSpecificationExecutor<EmployeeEntity> {
}
```

Uso:

``` java
Specification<EmployeeEntity> spec =
        EmployeeSpecification.ativo(true);

Pageable pageable =
        PageRequest.of(0, 10);

Page<EmployeeEntity> result =
        repository.findAll(spec, pageable);
```

Conceitualmente o JPA fará:

``` sql
SELECT *
FROM employee
WHERE ativo = true
LIMIT 10
OFFSET 0;
```

E o `Page` ainda traz:

``` java
result.getContent();       // registros
result.getTotalElements(); // total
result.getTotalPages();    // páginas
result.getSize();          // tamanho
result.getNumber();        // página atual
result.hasNext();          // possui próxima?
result.hasPrevious();      // possui anterior?
```

Então para seu HATEOS fica bem interessante:

``` text
Specification
      +
Pageable
      ↓
JpaSpecificationExecutor
      ↓
Page<EmployeeEntity>
      ↓
Mapper
      ↓
PagedModel
      ↓
HATEOAS
```

**E tudo continua no mesmo `EmployeeRepository`.**

------------------------------------------------------------------------

## Usuário

O object struct, realiza o de para model para dto

## Assistente

Sim. Se você está falando do **ObjectStruct/MapStruct**, ele é
justamente usado para fazer o **de-para entre objetos**, por exemplo:

``` text
Model/Domain  →  DTO
DTO            →  Model/Domain
Entity         →  Domain
Domain         →  Entity
```

Só uma correção de nome: provavelmente você quis dizer **MapStruct**,
que é a biblioteca Java mais comum para isso.

### Exemplo

Seu domínio:

``` java
public class Employee {

    private Long id;
    private String matricula;
    private String nome;
}
```

Seu DTO:

``` java
public record EmployeeResponse(
        Long id,
        String matricula,
        String nome
) {}
```

O mapper:

``` java
@Mapper(componentModel = "spring")
public interface EmployeeMapper {

    EmployeeResponse toResponse(Employee employee);

    Employee toDomain(EmployeeResponse response);
}
```

Como os nomes são iguais:

``` text
Employee                 EmployeeResponse
---------                ----------------
id        ─────────────→ id
matricula ─────────────→ matricula
nome      ─────────────→ nome
```

O MapStruct gera a implementação automaticamente em **compile time**.

### Se os nomes forem diferentes

``` java
public record EmployeeResponse(
        Long employeeId,
        String registration,
        String name
) {}
```

Você informa explicitamente:

``` java
@Mapper(componentModel = "spring")
public interface EmployeeMapper {

    @Mapping(source = "id", target = "employeeId")
    @Mapping(source = "matricula", target = "registration")
    @Mapping(source = "nome", target = "name")
    EmployeeResponse toResponse(Employee employee);
}
```

E também pode fazer o inverso:

``` java
@Mapper(componentModel = "spring")
public interface EmployeeMapper {

    @Mapping(source = "id", target = "employeeId")
    @Mapping(source = "matricula", target = "registration")
    @Mapping(source = "nome", target = "name")
    EmployeeResponse toResponse(Employee employee);

    @InheritInverseConfiguration
    Employee toDomain(EmployeeResponse response);
}
```

Então, na arquitetura que estamos discutindo:

``` text
JPA Entity
    ↓
Entity → Domain
    ↓
Domain Model
    ↓
Domain → DTO
    ↓
HTTP Response + HATEOAS
```

O **MapStruct faz os de-para**, enquanto o **HATEOAS fica responsável
pelos links da resposta HTTP**.

------------------------------------------------------------------------

## Usuário

Mais se os tipos for diferentes tipo string para int, ele faz a
conversão?

## Assistente

Sim, **o MapStruct consegue fazer algumas conversões automaticamente**,
inclusive entre tipos simples.

Por exemplo:

``` java
public class Employee {
    private Integer idade;
}
```

e:

``` java
public record EmployeeResponse(
    String idade
) {}
```

O MapStruct consegue gerar a conversão equivalente a:

``` java
String.valueOf(employee.getIdade())
```

### O contrário também

``` java
public class Employee {
    private String idade;
}
```

``` java
public record EmployeeResponse(
    Integer idade
) {}
```

O MapStruct pode gerar uma conversão equivalente a:

``` java
Integer.parseInt(employee.getIdade())
```

Então:

``` text
Integer → String  ✅
String  → Integer ✅
```

### Mas atenção

Para conversões mais complexas, você pode precisar informar como fazer.

Por exemplo:

``` text
String → LocalDate
```

ou:

``` text
String → Enum
```

ou uma regra própria:

``` text
"1" → ATIVO
"0" → INATIVO
```

Nesse caso, você pode criar um método auxiliar:

``` java
default Status toStatus(String value) {

    return "1".equals(value)
            ? Status.ATIVO
            : Status.INATIVO;
}
```

E o MapStruct utiliza esse método.

**Resumo:** para tipos básicos compatíveis, o MapStruct faz o `de-para`
automaticamente; para conversões que envolvem regra de negócio ou
transformação específica, você fornece o método de conversão.

------------------------------------------------------------------------

## Usuário

Se for nomes diferentes ?

## Assistente

Sim. Se os **nomes dos campos forem diferentes**, você usa
`@Mapping(source, target)`.

Por exemplo:

``` java
public class Employee {
    private String matricula;
    private String nome;
}
```

DTO:

``` java
public record EmployeeResponse(
    String registration,
    String name
) {}
```

O MapStruct não consegue saber sozinho que `matricula` corresponde a
`registration`. Então:

``` java
@Mapper(componentModel = "spring")
public interface EmployeeMapper {

    @Mapping(source = "matricula", target = "registration")
    @Mapping(source = "nome", target = "name")
    EmployeeResponse toResponse(Employee employee);
}
```

O resultado será equivalente a:

``` java
response.registration = employee.getMatricula();
response.name = employee.getNome();
```

### E pode combinar nome diferente + tipo diferente

``` java
public class Employee {
    private Integer idade;
}
```

``` java
public record EmployeeResponse(
    String age
) {}
```

Mapper:

``` java
@Mapper(componentModel = "spring")
public interface EmployeeMapper {

    @Mapping(source = "idade", target = "age")
    EmployeeResponse toResponse(Employee employee);
}
```

O MapStruct faz:

``` text
Employee.idade (Integer)
        ↓
EmployeeResponse.age (String)
```

e realiza a conversão de tipo automaticamente quando há uma conversão
suportada.

### Se forem objetos aninhados

Também funciona:

``` java
class Employee {
    private Dados dados;
}

class Dados {
    private String nome;
}
```

DTO:

``` java
record EmployeeResponse(
    String nome
) {}
```

Você pode fazer:

``` java
@Mapper(componentModel = "spring")
public interface EmployeeMapper {

    @Mapping(source = "dados.nome", target = "nome")
    EmployeeResponse toResponse(Employee employee);
}
```

Ou seja:

``` text
source = "dados.nome"
          ↓
Employee
 └── dados
      └── nome

target = "nome"
          ↓
EmployeeResponse
 └── nome
```

Esse é justamente um dos pontos fortes do **MapStruct**: você consegue
declarar vários `source` → `target` no mapper principal e deixar a
implementação do de-para ser gerada automaticamente.

------------------------------------------------------------------------

## Usuário

Alguma forma de mapear na classe, para o mapper entender

## Assistente

Sim. Se você quer **deixar o mapeamento definido na própria classe**,
para não precisar repetir `@Mapping` em cada método do mapper, há
algumas opções no MapStruct.

### 1. `@Mapping` no próprio método --- padrão

O MapStruct não oferece uma anotação que você coloque diretamente no
atributo da classe dizendo:

``` java
// "este campo corresponde àquele campo"
```

para que todos os mappers descubram isso automaticamente.

Mas você pode criar uma **configuração reutilizável**.

### 2. `@MapperConfig` --- melhor para padronizar

Por exemplo:

``` java
@MapperConfig
public interface EmployeeMappingConfig {

    @Mapping(source = "matricula", target = "registration")
    @Mapping(source = "nome", target = "name")
    EmployeeResponse map(Employee employee);
}
```

Porém, `@MapperConfig` é mais adequado para **configurações
compartilhadas de mappers**, e não como um "metadata" colocado no
modelo.

------------------------------------------------------------------------

### 3. O que você provavelmente está procurando

Se sua intenção é algo como:

``` java
public class Employee {

    @MapTo("registration")
    private String matricula;

    @MapTo("name")
    private String nome;
}
```

e então:

``` java
@Mapper
public interface EmployeeMapper {

    EmployeeResponse toResponse(Employee employee);
}
```

**O MapStruct não trabalha dessa forma nativamente.**

Ele foi projetado para que o mapeamento fique no mapper:

``` java
@Mapper(componentModel = "spring")
public interface EmployeeMapper {

    @Mapping(source = "matricula", target = "registration")
    @Mapping(source = "nome", target = "name")
    EmployeeResponse toResponse(Employee employee);
}
```

Isso é proposital: o **modelo não fica acoplado ao mecanismo de
mapeamento**.

------------------------------------------------------------------------

### Uma alternativa muito boa para muitos campos

Se você tem vários mapeamentos repetidos:

``` java
@Mapping(source = "contrato.dados.nome", target = "nome")
@Mapping(source = "contrato.dados.codigo", target = "codigo")
@Mapping(source = "contrato.endereco.cidade", target = "cidade")
...
```

você pode criar **métodos de conversão auxiliares** e usar `uses`:

``` java
@Mapper(
    componentModel = "spring",
    uses = {
        ContratoMapper.class,
        EnderecoMapper.class
    }
)
public interface EmployeeMapper {

    EmployeeResponse toResponse(Employee employee);
}
```

Assim o MapStruct procura os outros mappers para resolver os objetos
complexos.

Para o cenário que você descreveu anteriormente --- **nomes diferentes,
objetos aninhados e vários `source → target`** --- eu usaria `@Mapping`
no mapper principal + `uses` para delegar os objetos complexos. Isso
evita colocar anotações de infraestrutura dentro das classes de domínio.

------------------------------------------------------------------------

## Usuário

Contrato ficaria como?

## Assistente

Se você está falando de **`Contrato` como objeto aninhado**, eu faria o
mapper separado e deixaria o `EmployeeMapper` usar o `ContratoMapper`.

Por exemplo, suponha:

``` java
class Employee {
    private String nome;
    private Contrato contrato;
}
```

E:

``` java
class Contrato {
    private String numero;
    private LocalDate dataInicio;
}
```

Enquanto o DTO é:

``` java
public record EmployeeResponse(
        String name,
        String contractNumber,
        LocalDate startDate
) {}
```

O `ContratoMapper` poderia ser:

``` java
@Mapper(componentModel = "spring")
public interface ContratoMapper {

    @Mapping(source = "numero", target = "contractNumber")
    @Mapping(source = "dataInicio", target = "startDate")
    ContratoResponse toResponse(Contrato contrato);
}
```

Mas, nesse caso, como o DTO final não possui um objeto `contrato`, e sim
campos "achatados", eu faria diretamente no mapper principal:

``` java
@Mapper(componentModel = "spring")
public interface EmployeeMapper {

    @Mapping(source = "nome", target = "name")
    @Mapping(source = "contrato.numero", target = "contractNumber")
    @Mapping(source = "contrato.dataInicio", target = "startDate")
    EmployeeResponse toResponse(Employee employee);
}
```

### Se o DTO também tiver `Contrato`

Aí fica mais interessante separar:

``` java
public record EmployeeResponse(
        String name,
        ContratoResponse contrato
) {}
```

``` java
public record ContratoResponse(
        String contractNumber,
        LocalDate startDate
) {}
```

`ContratoMapper`:

``` java
@Mapper(componentModel = "spring")
public interface ContratoMapper {

    @Mapping(source = "numero", target = "contractNumber")
    @Mapping(source = "dataInicio", target = "startDate")
    ContratoResponse toResponse(Contrato contrato);
}
```

E o principal:

``` java
@Mapper(
    componentModel = "spring",
    uses = ContratoMapper.class
)
public interface EmployeeMapper {

    @Mapping(source = "nome", target = "name")
    EmployeeResponse toResponse(Employee employee);
}
```

O MapStruct entende:

``` text
Employee
 ├── nome
 │      ↓
 │   EmployeeResponse.name
 │
 └── contrato
       ↓
   ContratoMapper
       ↓
   ContratoResponse
```

### Essa segunda abordagem é a que eu usaria

Porque cada objeto conhece apenas seu próprio de-para:

``` text
EmployeeMapper
     │
     └── ContratoMapper
              │
              └── ContratoResponse
```

E se `Contrato` tiver vários objetos internos:

``` text
Contrato
 ├── dados
 ├── endereco
 ├── empresa
 └── periodo
```

você pode continuar decompondo:

``` java
@Mapper(
    componentModel = "spring",
    uses = {
        DadosContratoMapper.class,
        EnderecoMapper.class,
        EmpresaMapper.class
    }
)
public interface ContratoMapper {
}
```

Assim o **mapper principal não vira um arquivo gigante com dezenas de
`source` e `target`**.

------------------------------------------------------------------------

## Usuário

No target objeto.objeto.atributo

O mapper instância os objetos ?

## Assistente

Sim, **mas depende da situação**. No MapStruct, quando o `target` aponta
para uma propriedade aninhada, ele pode criar os objetos intermediários
necessários --- **desde que consiga instanciá-los**.

Exemplo:

``` java
class EmployeeResponse {
    private DadosResponse dados;
}

class DadosResponse {
    private String nome;
}
```

E você faz:

``` java
@Mapper(componentModel = "spring")
public interface EmployeeMapper {

    @Mapping(
        source = "dados.nome",
        target = "dados.nome"
    )
    EmployeeResponse toResponse(Employee employee);
}
```

Se `EmployeeResponse.dados` estiver `null`, o código gerado pelo
MapStruct pode fazer conceitualmente:

``` java
EmployeeResponse response = new EmployeeResponse();

if (employee.getDados() != null) {

    if (response.getDados() == null) {
        response.setDados(new DadosResponse());
    }

    response.getDados()
            .setNome(employee.getDados().getNome());
}
```

Ou seja:

``` text
EmployeeResponse
       │
       └── dados
             │
             └── nome
```

O MapStruct pode criar o `dados`.

### Porém, existe uma condição importante

Ele precisa saber **como criar `DadosResponse`**.

Por exemplo, isso funciona normalmente:

``` java
public class DadosResponse {

    public DadosResponse() {
    }

    // getters/setters
}
```

Se houver apenas:

``` java
public DadosResponse(String nome) {
    this.nome = nome;
}
```

e nenhum construtor sem argumentos, o MapStruct pode não conseguir gerar
a instanciação automaticamente.

------------------------------------------------------------------------

### E no caso de vários níveis

Por exemplo:

``` java
@Mapping(
    source = "contrato.dados.nome",
    target = "contrato.dados.nome"
)
```

O alvo:

``` text
EmployeeResponse
 └── contrato
      └── dados
           └── nome
```

pode exigir a criação de:

``` text
new EmployeeResponse()
    ↓
new ContratoResponse()
    ↓
new DadosResponse()
    ↓
setNome(...)
```

**Mas há uma diferença importante:** se esses objetos tiverem lógica
própria de conversão, eu prefiro usar `uses` + mappers específicos:

``` java
@Mapper(
    componentModel = "spring",
    uses = {
        ContratoMapper.class
    }
)
public interface EmployeeMapper {
}
```

Assim o `EmployeeMapper` não precisa conhecer todos os detalhes de
`Contrato`.

Se quiser, posso mostrar **exatamente o código que o MapStruct gera por
baixo para `target = "contrato.dados.nome"`**, incluindo os
`null checks` e as instanciações.

------------------------------------------------------------------------

## Usuário

Sim

## Assistente

Sim. Vamos ver exatamente o que acontece quando você usa um `target`
aninhado como:

``` java
@Mapping(
    source = "contrato.dados.nome",
    target = "contrato.dados.nome"
)
```

### 1. Objetos

Imagine o Model:

``` java
public class Employee {

    private Contrato contrato;

    // getters/setters
}
```

``` java
public class Contrato {

    private Dados dados;

    // getters/setters
}
```

``` java
public class Dados {

    private String nome;

    // getters/setters
}
```

E o DTO:

``` java
public class EmployeeResponse {

    private ContratoResponse contrato;

    // getters/setters
}
```

``` java
public class ContratoResponse {

    private DadosResponse dados;

    // getters/setters
}
```

``` java
public class DadosResponse {

    private String nome;

    // getters/setters
}
```

------------------------------------------------------------------------

## 2. Mapper

Você escreve apenas:

``` java
@Mapper(componentModel = "spring")
public interface EmployeeMapper {

    @Mapping(
        source = "contrato.dados.nome",
        target = "contrato.dados.nome"
    )
    EmployeeResponse toResponse(Employee employee);
}
```

O MapStruct gera a implementação.

Conceitualmente, ela fica parecida com:

``` java
@Override
public EmployeeResponse toResponse(Employee employee) {

    if (employee == null) {
        return null;
    }

    EmployeeResponse response =
            new EmployeeResponse();

    if (employee.getContrato() != null) {

        if (response.getContrato() == null) {
            response.setContrato(
                new ContratoResponse()
            );
        }

        if (employee.getContrato().getDados() != null) {

            if (response.getContrato().getDados() == null) {
                response.getContrato().setDados(
                    new DadosResponse()
                );
            }

            response.getContrato()
                    .getDados()
                    .setNome(
                        employee.getContrato()
                                .getDados()
                                .getNome()
                    );
        }
    }

    return response;
}
```

Ou seja, ele consegue cuidar da árvore:

``` text
Employee
   │
   └── contrato
         │
         └── dados
              │
              └── nome
```

para:

``` text
EmployeeResponse
   │
   └── contrato
         │
         └── dados
              │
              └── nome
```

------------------------------------------------------------------------

# 3. E se os nomes forem diferentes?

Aí fica ainda mais interessante.

Model:

``` text
Employee
 └── contrato
      └── dados
           └── nome
```

DTO:

``` text
EmployeeResponse
 └── contract
      └── information
           └── fullName
```

Você pode fazer:

``` java
@Mapper(componentModel = "spring")
public interface EmployeeMapper {

    @Mapping(
        source = "contrato.dados.nome",
        target = "contract.information.fullName"
    )
    EmployeeResponse toResponse(Employee employee);
}
```

O MapStruct entende:

``` text
contrato.dados.nome
        ↓
contract.information.fullName
```

E pode criar os objetos intermediários necessários.

------------------------------------------------------------------------

# 4. E aqui existe uma diferença importante

Se você tiver:

``` java
@Mapping(
    source = "contrato.dados.nome",
    target = "contract.information.fullName"
)
```

o MapStruct está fazendo **mapeamento de propriedades**.

Mas se você tiver:

``` text
Contrato → ContratoResponse
```

com regras próprias, é melhor criar:

``` java
@Mapper(componentModel = "spring")
public interface ContratoMapper {

    @Mapping(
        source = "dados.nome",
        target = "information.fullName"
    )
    ContratoResponse toResponse(Contrato contrato);
}
```

E no principal:

``` java
@Mapper(
    componentModel = "spring",
    uses = ContratoMapper.class
)
public interface EmployeeMapper {

    EmployeeResponse toResponse(Employee employee);
}
```

Aí o MapStruct delega:

``` text
EmployeeMapper
      │
      └── ContratoMapper
             │
             └── DadosMapper
```

Isso é particularmente útil quando o `Contrato` aparece em vários DTOs.

------------------------------------------------------------------------

## Um detalhe sobre `null`

Imagine:

``` java
employee.getContrato() == null
```

O MapStruct não vai fazer:

``` java
response.getContrato().getDados()
```

e causar `NullPointerException`.

Ele gera verificações de `null` para navegar pela estrutura.

Portanto:

``` text
Employee
  contrato = null
```

resulta normalmente em:

``` text
EmployeeResponse
  contrato = null
```

E se:

``` text
contrato ≠ null
dados = null
```

fica:

``` text
EmployeeResponse
  contrato
    dados = null
```

sem precisar você escrever manualmente toda essa lógica.

**Então, sim: para `target` aninhado, o MapStruct consegue instanciar os
objetos intermediários e fazer o caminho `objeto.objeto.atributo`, desde
que as classes sejam instanciáveis e o mapeamento seja suportado.**

------------------------------------------------------------------------

## Usuário

Os nomes são iguais Aí só preciso definir o target

## Assistente

Exatamente. **Se os nomes dos atributos são iguais, você pode informar
apenas o `target`**, e o MapStruct entende o `source` pelo caminho
correspondente.

Por exemplo:

``` java
@Mapper(componentModel = "spring")
public interface EmployeeMapper {

    @Mapping(target = "contrato.dados.nome")
    EmployeeResponse toResponse(Employee employee);
}
```

Aqui:

``` text
source: contrato.dados.nome
target: contrato.dados.nome
```

Como os nomes são iguais, você não precisa escrever:

``` java
@Mapping(
    source = "contrato.dados.nome",
    target = "contrato.dados.nome"
)
```

**Porém, há uma ressalva:** na API do MapStruct, `@Mapping` normalmente
exige `source` quando você está fazendo esse tipo de mapeamento;
`target` sozinho é usado em alguns cenários específicos, mas não é a
forma geral de dizer "mapear automaticamente o mesmo caminho". Para
nomes iguais, o mais simples é **nem colocar `@Mapping`**:

``` java
@Mapper(componentModel = "spring")
public interface EmployeeMapper {

    EmployeeResponse toResponse(Employee employee);
}
```

Se `Employee` e `EmployeeResponse` tiverem:

``` text
contrato.dados.nome
```

com a mesma estrutura, o MapStruct tenta fazer o mapeamento
automaticamente.

Se os nomes forem diferentes:

``` text
Employee:
contrato.dados.nome

DTO:
contrato.dados.nomeCompleto
```

Aí:

``` java
@Mapping(
    source = "contrato.dados.nome",
    target = "contrato.dados.nomeCompleto"
)
```

**Regra prática:** nomes e estrutura iguais → deixe o MapStruct fazer
sozinho; nomes diferentes → `source` + `target`.
