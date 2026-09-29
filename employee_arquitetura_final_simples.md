# Arquitetura Hexagonal — Employee Service
## Java 21 — Versão Final Simplificada

## 1. Objetivo

Este documento consolida a arquitetura definida para o microserviço Java 21.

A solução utiliza arquitetura hexagonal, mas com uma estrutura de packages simples, evitando excesso de camadas e subpackages.

O serviço possui três responsabilidades principais:

- Employee
- ThirdParty
- Permission / Allowed Fields

---

# 2. Endpoints

## Employee

- `GetEmployeeById`
- `GetEmployees`
- `GetEmployeePhotoById`

## ThirdParty

- `GetThirdPartyById`
- `GetThirdParties`
- `GetThirdPartyPhotoById`

## Permission

- `GetAllowedFields`

Total:

```text
6 endpoints de Employee/ThirdParty
+
1 caso de uso de AllowedFields
=
7 Use Cases
```

---

# 3. Estrutura de packages

A estrutura final será propositalmente simples:

```text
com.company.employee
│
├── application
│   ├── input
│   ├── output
│   └── usecase
│
├── domain
│
└── infrastructure
    ├── input
    ├── output
    ├── persistence
    └── presentation
```

Não serão criados packages separados para:

```text
getbyid
getall
getphoto
factory
rules
mapper
resolver
```

quando isso não agregar organização real.

---

# 4. Visão geral da arquitetura

```text
                    HTTP
                     │
                     ▼
              infrastructure
                  input
                     │
                     ▼
               application
                 usecase
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
       Domain              application
                              output
                                │
                   ┌────────────┴────────────┐
                   ▼                         ▼
              Database                     Redis
```

Para o retorno HTTP:

```text
Domain
   +
AllowedField
   │
   ▼
EmployeePresenter
   │
   ├── EmployeeResponseFactory
   ├── EmployeeResponseRules
   └── EmployeeResponseMapper
   │
   ▼
EmployeeResponse
```

---

# 5. Domain

O Domain representa o modelo de negócio.

Não deve conhecer:

- Spring
- JPA
- Redis
- Controller
- HTTP
- DTO de API
- Entity de banco

Estrutura:

```text
domain
├── Employee.java
├── ThirdParty.java
├── Contato.java
├── EmployeeField.java
└── AllowedField.java
```

---

# 6. Employee

A tabela do banco pode ser enorme e gerar uma Entity grande.

Isso não significa que o contrato HTTP precisa possuir a mesma estrutura.

A Entity representa persistência.

O `Employee` representa o modelo utilizado pela aplicação.

Exemplo:

```java
public class Employee {

    private Long id;
    private String matricula;
    private String nome;
    private String cpf;
    private String telefone;
    private String celular;
    private String cidade;
    private String estado;
    private Long departmentId;

    // getters / setters
}
```

A classe pode conter os dados necessários para o domínio e para os diferentes casos de uso.

---

# 7. Contato

No domínio, os dados podem ser agrupados de forma semântica.

```java
public class Contato {

    private String telefone;
    private String celular;

    public String getTelefone() {
        return telefone;
    }

    public String getCelular() {
        return celular;
    }
}
```

Isso evita que o domínio precise conhecer a estrutura genérica do contrato.

Por exemplo:

```text
Domain

Contato
├── telefone
└── celular
```

Enquanto o contrato pode precisar:

```text
contacts
├── { itemName: TELEFONE, itemId: ... }
└── { itemName: CELULAR, itemId: ... }
```

A transformação fica na Presentation.

---

# 8. EmployeeField

`EmployeeField` representa os campos que podem ser apresentados no contexto Employee.

```java
public enum EmployeeField {

    NAME("nr_name"),
    CPF("nr_cpf"),
    PHONE("nr_phone"),
    MOBILE("nr_mobile"),
    CITY("nr_city"),
    STATE("nr_state"),
    DEPARTMENT("nr_department");

    private static final Map<String, EmployeeField> BY_CODE =
            Arrays.stream(values())
                    .collect(Collectors.toUnmodifiableMap(
                            field -> field.code,
                            Function.identity()
                    ));

    private final String code;

    EmployeeField(String code) {
        this.code = code;
    }

    public static Optional<EmployeeField> fromCode(
            String code) {

        return Optional.ofNullable(
                BY_CODE.get(code));
    }
}
```

O `Map` é criado uma única vez.

Como é imutável:

```java
Collectors.toUnmodifiableMap(...)
```

é apropriado.

---

# 9. AllowedField

O Redis possui códigos externos como:

```text
nr_name
nr_cpf
nr_phone
```

A aplicação não deve espalhar essas Strings.

Por isso existe:

```java
public record AllowedField(
        String name
) {
}
```

Exemplo:

```java
new AllowedField("nr_name");
```

O significado é:

> este campo foi permitido para ser retornado.

---

# 10. Diferença entre AllowedField e EmployeeField

São conceitos diferentes.

### AllowedField

Representa uma autorização:

```text
"este campo está permitido"
```

### EmployeeField

Representa um campo conhecido pelo contexto Employee:

```text
NAME
CPF
PHONE
```

Fluxo:

```text
Redis
  ↓
"nr_name"
  ↓
AllowedField
  ↓
EmployeeField.fromCode()
  ↓
EmployeeField.NAME
```

O `AllowedField` não conhece `EmployeeField`.

Não existe `EmployeeFieldResolver`.

A conversão é simples e fica diretamente no Presenter.

---

# 11. Application

Application contém os casos de uso e suas portas.

Estrutura:

```text
application
├── input
├── output
└── usecase
```

---

# 12. Input

Os Inputs são as portas de entrada da aplicação.

Exemplo:

```java
public interface GetEmployeeByIdInput {

    EmployeeResponse execute(Long id);
}
```

Outro:

```java
public interface GetEmployeesInput {

    EmployeeResponse execute(
            GetEmployeesQuery query);
}
```

E:

```java
public interface GetAllowedFieldsInput {

    List<AllowedField> execute(
            String matricula);
}
```

---

# 13. Output

Os Outputs são as portas de saída.

Exemplo:

```java
public interface EmployeeRepositoryOutput {

    Optional<Employee> findById(Long id);

    List<Employee> findAll(
            GetEmployeesQuery query);
}
```

Permission:

```java
public interface AllowedFieldsOutput {

    List<AllowedField> findByMatricula(
            String matricula);
}
```

A Application não sabe se isso é:

- JPA
- JDBC
- Redis
- API externa
- arquivo

Ela conhece somente a interface.

---

# 14. Use Cases

Os sete casos de uso ficam juntos.

```text
application
└── usecase
    ├── GetEmployeeByIdUseCase.java
    ├── GetEmployeesUseCase.java
    ├── GetEmployeePhotoByIdUseCase.java
    ├── GetThirdPartyByIdUseCase.java
    ├── GetThirdPartiesUseCase.java
    ├── GetThirdPartyPhotoByIdUseCase.java
    └── GetAllowedFieldsUseCase.java
```

Não é necessário criar um package para cada Use Case.

---

# 15. GetEmployeeByIdUseCase

```java
@Service
public class GetEmployeeByIdUseCase
        implements GetEmployeeByIdInput {

    private final EmployeeRepositoryOutput repository;
    private final GetAllowedFieldsInput allowedFields;
    private final EmployeePresenter presenter;

    public GetEmployeeByIdUseCase(
            EmployeeRepositoryOutput repository,
            GetAllowedFieldsInput allowedFields,
            EmployeePresenter presenter) {

        this.repository = repository;
        this.allowedFields = allowedFields;
        this.presenter = presenter;
    }

    @Override
    public EmployeeResponse execute(Long id) {

        Employee employee =
                repository.findById(id)
                        .orElseThrow();

        List<AllowedField> fields =
                allowedFields.execute(
                        employee.getMatricula());

        return presenter.present(
                employee,
                fields);
    }
}
```

O Use Case orquestra.

Ele não deve saber:

- como o banco funciona;
- como o Redis funciona;
- como o Response é montado.

---

# 16. GetAllowedFieldsUseCase

```java
@Service
public class GetAllowedFieldsUseCase
        implements GetAllowedFieldsInput {

    private final AllowedFieldsOutput output;

    public GetAllowedFieldsUseCase(
            AllowedFieldsOutput output) {

        this.output = output;
    }

    @Override
    public List<AllowedField> execute(
            String matricula) {

        return output.findByMatricula(
                matricula);
    }
}
```

---

# 17. Query

Para consultas simples:

```java
execute(Long id)
```

não é necessário criar um objeto.

Para consultas com vários parâmetros, usar `Query`.

Exemplo:

```java
public record GetEmployeesQuery(
        List<Long> ids,
        String name,
        String department,
        Integer page,
        Integer size
) {
}
```

Uso:

```java
useCase.execute(query);
```

---

# 18. Command

Command é apropriado quando existe uma ação/operação com entrada mais complexa.

Exemplo futuro:

```java
public record UpdateEmployeeCommand(
        Long id,
        String name,
        String department,
        String phone
) {
}
```

Uso:

```java
useCase.execute(command);
```

Não criar Command ou Query apenas para seguir um padrão.

A regra é:

```text
Entrada simples
    ↓
execute(id)

Consulta com vários critérios
    ↓
Query

Operação complexa
    ↓
Command
```

---

# 19. Infrastructure

A Infrastructure contém as implementações das portas.

Estrutura:

```text
infrastructure
├── input
├── output
├── persistence
└── presentation
```

---

# 20. Infrastructure Input

Aqui ficam os Controllers.

```text
infrastructure
└── input
    ├── EmployeeController.java
    └── ThirdPartyController.java
```

Exemplo:

```java
@RestController
@RequestMapping("/employees")
public class EmployeeController {

    private final GetEmployeeByIdInput getEmployeeById;

    public EmployeeController(
            GetEmployeeByIdInput getEmployeeById) {

        this.getEmployeeById = getEmployeeById;
    }

    @GetMapping("/{id}")
    public EmployeeResponse getById(
            @PathVariable Long id) {

        return getEmployeeById.execute(id);
    }
}
```

O Controller não acessa Repository.

O fluxo é:

```text
Controller
    ↓
Input
    ↓
UseCase
```

---

# 21. Infrastructure Output

Implementações das portas de saída.

Exemplo:

```text
infrastructure
└── output
    ├── EmployeeRepositoryAdapter.java
    └── AllowedFieldsRedisAdapter.java
```

---

# 22. EmployeeRepositoryAdapter

```java
@Component
public class EmployeeRepositoryAdapter
        implements EmployeeRepositoryOutput {

    private final EmployeeJpaRepository repository;
    private final EmployeePersistenceMapper mapper;

    public EmployeeRepositoryAdapter(
            EmployeeJpaRepository repository,
            EmployeePersistenceMapper mapper) {

        this.repository = repository;
        this.mapper = mapper;
    }

    @Override
    public Optional<Employee> findById(Long id) {

        return repository.findById(id)
                .map(mapper::toDomain);
    }
}
```

---

# 23. Redis Adapter

O Redis continua trabalhando com String internamente.

```java
@Component
public class AllowedFieldsRedisAdapter
        implements AllowedFieldsOutput {

    private final RedisTemplate<String, String> redis;

    public AllowedFieldsRedisAdapter(
            RedisTemplate<String, String> redis) {

        this.redis = redis;
    }

    @Override
    public List<AllowedField> findByMatricula(
            String matricula) {

        List<String> fields =
                redis.opsForList()
                     .range(matricula, 0, -1);

        return fields.stream()
                .map(AllowedField::new)
                .toList();
    }
}
```

Fronteira:

```text
Redis
    ↓
List<String>
    ↓
Adapter
    ↓
List<AllowedField>
```

Depois disso, `String` não precisa circular pela Application.

---

# 24. Persistence

A tabela pode ser enorme.

A Entity representa a tabela.

```java
@Entity
@Table(name = "employee")
public class EmployeeEntity {

    @Id
    private Long id;

    private String matricula;
    private String nome;
    private String cpf;
    private String telefone;
    private String celular;
    private String cidade;
    private String estado;
    private Long departmentId;

    // getters / setters
}
```

É normal que:

```text
1 tabela grande
      ↓
1 Entity grande
```

Isso não obriga:

```text
Entity
      ↓
Response
```

a terem a mesma estrutura.

---

# 25. Persistence Mapper

Responsável somente por:

```text
Entity → Domain
```

Exemplo:

```java
@Component
public class EmployeePersistenceMapper {

    public Employee toDomain(
            EmployeeEntity source) {

        return new Employee(
                source.getId(),
                source.getMatricula(),
                source.getNome(),
                source.getCpf(),
                source.getTelefone(),
                source.getCelular(),
                source.getCidade(),
                source.getEstado(),
                source.getDepartmentId()
        );
    }
}
```

---

# 26. Presentation

Aqui está a transformação:

```text
Domain
    ↓
Response
```

Estrutura simples:

```text
infrastructure
└── presentation
    ├── EmployeeResponse.java
    ├── EmployeePresenter.java
    ├── EmployeeResponseFactory.java
    ├── EmployeeResponseRules.java
    └── EmployeeResponseMapper.java
```

Não é necessário criar packages separados para:

```text
factory
rules
mapper
presenter
```

---

# 27. EmployeeResponse

O Response representa exatamente o contrato da API.

Pode possuir diversos níveis.

Exemplo:

```java
public class EmployeeResponse {

    private PersonalData personalData;
    private Address address;
    private Organization organization;
    private List<ContactItem> contacts;

    // getters / setters
}
```

---

# 28. Response normalizado

Exemplo:

```text
EmployeeResponse
│
├── personalData
│   ├── nome
│   └── cpf
│
├── contacts
│   ├── ContactItem
│   │   ├── itemName
│   │   └── itemId
│   └── ContactItem
│
├── address
│   └── location
│       ├── cidade
│       └── estado
│
└── organization
    └── department
        └── id
```

O Domain pode ser diferente.

---

# 29. EmployeeResponseFactory

A Factory cria a estrutura vazia do Response.

```java
@Component
public class EmployeeResponseFactory {

    public EmployeeResponse create() {

        EmployeeResponse response =
                new EmployeeResponse();

        response.setPersonalData(
                new PersonalData());

        response.setAddress(
                new Address());

        response.getAddress()
                .setLocation(
                        new Location());

        response.setOrganization(
                new Organization());

        response.getOrganization()
                .setDepartment(
                        new Department());

        response.setContacts(
                new ArrayList<>());

        return response;
    }
}
```

A Factory não decide quais campos serão preenchidos.

Ela somente prepara o `target`.

---

# 30. EmployeeResponseRules

As Rules definem os preenchimentos.

```java
public final class EmployeeResponseRules {

    private EmployeeResponseRules() {
    }

    public static Map<EmployeeField,
            BiConsumer<Employee, EmployeeResponse>> create() {

        return Map.of(

            EmployeeField.NAME,
            (source, target) ->
                target.getPersonalData()
                      .setNome(source.getNome()),

            EmployeeField.CPF,
            (source, target) ->
                target.getPersonalData()
                      .setCpf(source.getCpf()),

            EmployeeField.PHONE,
            (source, target) ->
                target.getContacts()
                      .add(new ContactItem(
                              "TELEFONE",
                              source.getTelefone()
                      )),

            EmployeeField.MOBILE,
            (source, target) ->
                target.getContacts()
                      .add(new ContactItem(
                              "CELULAR",
                              source.getCelular()
                      )),

            EmployeeField.CITY,
            (source, target) ->
                target.getAddress()
                      .getLocation()
                      .setCidade(
                              source.getCidade()),

            EmployeeField.STATE,
            (source, target) ->
                target.getAddress()
                      .getLocation()
                      .setEstado(
                              source.getEstado()),

            EmployeeField.DEPARTMENT,
            (source, target) ->
                target.getOrganization()
                      .getDepartment()
                      .setId(
                              source.getDepartmentId())
        );
    }
}
```

Como o Map é imutável:

```java
Map.of(...)
```

é suficiente.

Não precisa ser `@Component`.

---

# 31. EmployeePresenter

O Presenter aplica as regras.

```java
@Component
public class EmployeePresenter {

    private final EmployeeResponseFactory factory;

    private final Map<EmployeeField,
            BiConsumer<Employee, EmployeeResponse>> rules;

    public EmployeePresenter(
            EmployeeResponseFactory factory) {

        this.factory = factory;
        this.rules =
                EmployeeResponseRules.create();
    }

    public EmployeeResponse present(
            Employee source,
            List<AllowedField> allowedFields) {

        EmployeeResponse target =
                factory.create();

        allowedFields.stream()
                .map(AllowedField::name)
                .map(EmployeeField::fromCode)
                .flatMap(Optional::stream)
                .map(rules::get)
                .filter(Objects::nonNull)
                .forEach(rule ->
                        rule.accept(source, target));

        return target;
    }
}
```

Não existe `EmployeeFieldResolver`.

A conversão é simples e fica no próprio Presenter:

```java
.map(AllowedField::name)
.map(EmployeeField::fromCode)
```

---

# 32. O `rule.accept()` modifica o Target?

Sim.

Neste trecho:

```java
rule.accept(source, target);
```

a Rule recebe:

```text
source = Employee
target = EmployeeResponse
```

Por exemplo:

```java
EmployeeField.NAME,
(source, target) ->
    target.getPersonalData()
          .setNome(source.getNome())
```

O `target` foi criado pela Factory:

```java
EmployeeResponse target =
        factory.create();
```

Depois a Rule modifica esse mesmo objeto.

No final:

```java
return target;
```

retorna o Response já preenchido.

---

# 33. Mapper de Presentation

Quando houver transformações maiores ou repetitivas, o Mapper pode ficar separado.

Responsabilidade:

```text
Domain → estruturas do Response
```

Exemplo:

```java
@Component
public class EmployeeResponseMapper {

    public PersonalData toPersonalData(
            Employee source) {

        PersonalData target =
                new PersonalData();

        target.setNome(source.getNome());
        target.setCpf(source.getCpf());

        return target;
    }
}
```

Não é necessário transformar todo preenchimento em Mapper.

As Rules continuam adequadas para o controle de campos permitidos.

---

# 34. Exemplo de lista genérica de contatos

O domínio possui:

```java
public class Contato {

    private String telefone;
    private String celular;

    // getters
}
```

O contrato possui:

```java
public class ContactItem {

    private String itemName;
    private String itemId;

    public ContactItem(
            String itemName,
            String itemId) {

        this.itemName = itemName;
        this.itemId = itemId;
    }
}
```

As Rules fazem:

```java
EmployeeField.PHONE,
(source, target) ->
    target.getContacts()
          .add(new ContactItem(
                  "TELEFONE",
                  source.getTelefone()
          ))
```

E:

```java
EmployeeField.MOBILE,
(source, target) ->
    target.getContacts()
          .add(new ContactItem(
                  "CELULAR",
                  source.getCelular()
          ))
```

Assim o domínio não precisa conhecer a lista genérica do contrato.

---

# 35. Null e Factory

O principal motivo para a Factory existir é preparar a árvore do Response.

Sem Factory:

```java
target.getAddress()
      .getLocation()
      .setCidade(...)
```

poderia gerar `NullPointerException`.

Com Factory:

```text
EmployeeResponse
    ↓
Address
    ↓
Location
```

já existem.

Então as Rules podem simplesmente fazer:

```java
target.getAddress()
      .getLocation()
      .setCidade(...)
```

---

# 36. Constructor e Rules

O Map de Rules não precisa ser criado usando objetos do Response.

As Rules são apenas funções:

```java
BiConsumer<Employee, EmployeeResponse>
```

Por isso:

```java
this.rules =
        EmployeeResponseRules.create();
```

não precisa instanciar `EmployeeResponse`.

Os objetos do Target só são criados quando:

```java
EmployeeResponse target =
        factory.create();
```

Portanto não existe problema de ordem de instanciação.

---

# 37. Por que não usar MapStruct para essa parte?

O cenário possui uma particularidade:

```text
Employee
     +
List<AllowedField>
     ↓
Response parcialmente preenchido
```

O problema não é simplesmente:

```text
Source → Target
```

Existe uma decisão dinâmica:

```text
campo permitido?
    │
    ├── sim → aplica regra
    │
    └── não → não preenche
```

O Map de Rules representa essa decisão de forma simples.

Não é necessário transformar isso em dezenas de métodos ou configurações complexas.

---

# 38. Fluxo completo do Employee

```text
GET /employees/{id}
        │
        ▼
EmployeeController
        │
        ▼
GetEmployeeByIdInput
        │
        ▼
GetEmployeeByIdUseCase
        │
        ├───────────────► EmployeeRepositoryOutput
        │                       │
        │                       ▼
        │                    Database
        │                       │
        │                       ▼
        │                    Employee
        │
        └───────────────► GetAllowedFieldsInput
                                │
                                ▼
                         GetAllowedFieldsUseCase
                                │
                                ▼
                         AllowedFieldsOutput
                                │
                                ▼
                              Redis
                                │
                                ▼
                          List<String>
                                │
                                ▼
                        Redis Adapter
                                │
                                ▼
                       List<AllowedField>
                                │
                                ▼
                         EmployeePresenter
                                │
                     ┌──────────┼──────────┐
                     ▼          ▼          ▼
                  Factory      Rules     EmployeeField
                     │          │          │
                     └──────────┴──────────┘
                                │
                                ▼
                         EmployeeResponse
                                │
                                ▼
                              HTTP
```

---

# 39. Fluxo de dados

## Banco

```text
EmployeeEntity
      ↓
EmployeePersistenceMapper
      ↓
Employee
```

## Redis

```text
String
      ↓
AllowedFieldsRedisAdapter
      ↓
AllowedField
```

## Contrato

```text
Employee
    +
AllowedField
    ↓
EmployeePresenter
    ↓
EmployeeField
    ↓
EmployeeResponseRules
    ↓
EmployeeResponse
```

---

# 40. Responsabilidade de cada classe

| Classe | Responsabilidade |
|---|---|
| `Employee` | Modelo de domínio |
| `ThirdParty` | Modelo de domínio |
| `Contato` | Modelo de domínio |
| `EmployeeField` | Campos do contexto Employee |
| `AllowedField` | Campo permitido retornado pela regra de acesso |
| `EmployeeController` | Entrada HTTP |
| `GetEmployeeByIdUseCase` | Orquestra busca do Employee |
| `GetEmployeesUseCase` | Orquestra consulta de Employees |
| `GetEmployeePhotoByIdUseCase` | Busca foto |
| `GetThirdPartyByIdUseCase` | Orquestra busca do ThirdParty |
| `GetThirdPartiesUseCase` | Orquestra consulta de ThirdParty |
| `GetThirdPartyPhotoByIdUseCase` | Busca foto |
| `GetAllowedFieldsUseCase` | Consulta campos permitidos |
| `EmployeeRepositoryOutput` | Porta de saída do Employee |
| `AllowedFieldsOutput` | Porta de saída da permissão |
| `EmployeeRepositoryAdapter` | Implementação do acesso ao banco |
| `AllowedFieldsRedisAdapter` | Implementação do acesso ao Redis |
| `EmployeeEntity` | Representação da tabela |
| `EmployeePersistenceMapper` | Entity → Domain |
| `EmployeeResponse` | Contrato de resposta |
| `EmployeeResponseFactory` | Cria o Response |
| `EmployeeResponseRules` | Regras de preenchimento |
| `EmployeePresenter` | Orquestra o preenchimento do Response |
| `EmployeeResponseMapper` | Transformações auxiliares Domain → Response |

---

# 41. Estrutura final completa

```text
com.company.employee
│
├── application
│   ├── input
│   │   ├── GetEmployeeByIdInput.java
│   │   ├── GetEmployeesInput.java
│   │   ├── GetEmployeePhotoByIdInput.java
│   │   ├── GetThirdPartyByIdInput.java
│   │   ├── GetThirdPartiesInput.java
│   │   ├── GetThirdPartyPhotoByIdInput.java
│   │   └── GetAllowedFieldsInput.java
│   │
│   ├── output
│   │   ├── EmployeeRepositoryOutput.java
│   │   ├── ThirdPartyRepositoryOutput.java
│   │   ├── EmployeePhotoOutput.java
│   │   ├── ThirdPartyPhotoOutput.java
│   │   └── AllowedFieldsOutput.java
│   │
│   └── usecase
│       ├── GetEmployeeByIdUseCase.java
│       ├── GetEmployeesUseCase.java
│       ├── GetEmployeePhotoByIdUseCase.java
│       ├── GetThirdPartyByIdUseCase.java
│       ├── GetThirdPartiesUseCase.java
│       ├── GetThirdPartyPhotoByIdUseCase.java
│       └── GetAllowedFieldsUseCase.java
│
├── domain
│   ├── Employee.java
│   ├── ThirdParty.java
│   ├── Contato.java
│   ├── EmployeeField.java
│   └── AllowedField.java
│
└── infrastructure
    ├── input
    │   ├── EmployeeController.java
    │   └── ThirdPartyController.java
    │
    ├── output
    │   ├── EmployeeRepositoryAdapter.java
    │   ├── ThirdPartyRepositoryAdapter.java
    │   └── AllowedFieldsRedisAdapter.java
    │
    ├── persistence
    │   ├── EmployeeEntity.java
    │   ├── ThirdPartyEntity.java
    │   └── EmployeePersistenceMapper.java
    │
    └── presentation
        ├── EmployeeResponse.java
        ├── EmployeeResponseFactory.java
        ├── EmployeeResponseRules.java
        ├── EmployeeResponseMapper.java
        └── EmployeePresenter.java
```

---

# 42. Princípios finais

## Não criar package por classe

Evitar:

```text
factory/
rules/
mapper/
resolver/
```

quando cada um contém apenas uma classe.

## Não criar package por Use Case

Evitar:

```text
getbyid/
getall/
getphoto/
```

Os Use Cases ficam juntos.

## Não criar abstração sem necessidade

Não utilizar:

```text
EmployeeFieldResolver
```

O Presenter pode fazer diretamente:

```java
.map(AllowedField::name)
.map(EmployeeField::fromCode)
```

## Não usar Command/Query por obrigação

Usar somente quando melhorarem a modelagem da entrada.

## Manter as fronteiras importantes

```text
Application
    ↕
Domain
    ↕
Infrastructure
```

A arquitetura deve proteger essas fronteiras sem criar complexidade artificial.

---

# 43. Arquitetura final resumida

```text
┌─────────────────────────────────────────────┐
│                INFRASTRUCTURE               │
│                                             │
│  Controller                                 │
│      │                                      │
│      ▼                                      │
│  Presentation                               │
│  Factory + Rules + Presenter + Response     │
│      │                                      │
│      ▼                                      │
│  Repository Adapter / Redis Adapter         │
└───────────────┬─────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────┐
│                 APPLICATION                 │
│                                             │
│  Input → UseCase → Output                   │
└───────────────┬─────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────┐
│                   DOMAIN                    │
│                                             │
│  Employee                                   │
│  ThirdParty                                 │
│  Contato                                    │
│  EmployeeField                              │
│  AllowedField                               │
└─────────────────────────────────────────────┘
```

## Decisão

A arquitetura final privilegia **simplicidade**.

Apenas três grandes áreas:

```text
application
domain
infrastructure
```

E dentro da infraestrutura:

```text
input
output
persistence
presentation
```

Sem criar níveis adicionais apenas para seguir padrões.

A complexidade fica no código onde ela realmente existe — principalmente na apresentação condicional dos campos — e não na estrutura de packages.
