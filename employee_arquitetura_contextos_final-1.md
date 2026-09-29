# Microserviço Employee — Arquitetura Hexagonal por Contexto

## 1. Escopo

Java 21, arquitetura hexagonal, com:

- 3 endpoints Employee: `GetById`, `GetAll`, `GetPhotoById`
- 3 endpoints ThirdParty: `GetById`, `GetAll`, `GetPhotoById`
- 1 caso de uso transversal: `GetAllowedFields`
- permissões consultadas no Redis
- Redis retorna `List<String>` como `nr_name`
- `EmployeeField` traduz códigos externos para conceitos internos
- `Factory`, `Rules`, `Mapper` e `Presenter` para montagem dos contratos

Total: **7 casos de uso**.

---

## 2. Estrutura final

```text
employee-service
│
├── application
│   ├── employee
│   │   ├── input
│   │   │   ├── GetEmployeeByIdInput.java
│   │   │   ├── GetEmployeesInput.java
│   │   │   └── GetEmployeePhotoByIdInput.java
│   │   ├── output
│   │   │   ├── EmployeeRepositoryOutput.java
│   │   │   └── EmployeePhotoOutput.java
│   │   └── usecase
│   │       ├── GetEmployeeByIdUseCase.java
│   │       ├── GetEmployeesUseCase.java
│   │       └── GetEmployeePhotoByIdUseCase.java
│   │
│   ├── thirdparty
│   │   ├── input
│   │   │   ├── GetThirdPartyByIdInput.java
│   │   │   ├── GetThirdPartiesInput.java
│   │   │   └── GetThirdPartyPhotoByIdInput.java
│   │   ├── output
│   │   │   ├── ThirdPartyRepositoryOutput.java
│   │   │   └── ThirdPartyPhotoOutput.java
│   │   └── usecase
│   │       ├── GetThirdPartyByIdUseCase.java
│   │       ├── GetThirdPartiesUseCase.java
│   │       └── GetThirdPartyPhotoByIdUseCase.java
│   │
│   └── permission
│       ├── input
│       │   └── GetAllowedFieldsInput.java
│       ├── output
│       │   └── AllowedFieldsOutput.java
│       └── usecase
│           └── GetAllowedFieldsUseCase.java
│
├── domain
│   ├── employee
│   │   ├── model
│   │   │   ├── Employee.java
│   │   │   └── Contato.java
│   │   └── enums
│   │       └── EmployeeField.java
│   └── thirdparty
│       └── model
│           └── ThirdParty.java
│
└── infrastructure
    ├── input
    │   ├── employee
    │   │   └── EmployeeController.java
    │   └── thirdparty
    │       └── ThirdPartyController.java
    │
    ├── output
    │   ├── employee
    │   │   ├── repository
    │   │   └── photo
    │   ├── thirdparty
    │   │   ├── repository
    │   │   └── photo
    │   └── permission
    │       └── redis
    │
    ├── persistence
    │   ├── employee
    │   │   ├── entity
    │   │   └── mapper
    │   └── thirdparty
    │       ├── entity
    │       └── mapper
    │
    └── presentation
        ├── employee
        │   ├── dto
        │   ├── factory
        │   ├── mapper
        │   ├── presenter
        │   └── rules
        └── thirdparty
            ├── dto
            ├── factory
            ├── mapper
            ├── presenter
            └── rules
```

---

## 3. Princípio

A organização principal é por contexto funcional:

```text
application
├── employee
├── thirdparty
└── permission
```

Isso evita concentrar todos os `input`, `output` e `usecase` em três packages gigantes.

---

## 4. Permission

`AllowedFields` é transversal a Employee e ThirdParty, portanto fica separado.

### Input

```java
public interface GetAllowedFieldsInput {
    List<String> execute(String matricula);
}
```

### Output

```java
public interface AllowedFieldsOutput {
    List<String> findByMatricula(String matricula);
}
```

### Use Case

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
    public List<String> execute(String matricula) {
        return output.findByMatricula(matricula);
    }
}
```

### Redis Adapter

```java
@Component
public class AllowedFieldsRedisAdapter
        implements AllowedFieldsOutput {

    @Override
    public List<String> findByMatricula(
            String matricula) {

        // Consulta Redis.
        return List.of(
                "nr_name",
                "nr_cpf",
                "nr_phone"
        );
    }
}
```

---

## 5. EmployeeField

O Redis retorna:

```text
nr_name
nr_cpf
nr_phone
```

O domínio trabalha com:

```text
NAME
CPF
PHONE
```

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

    public String getCode() {
        return code;
    }

    public static Optional<EmployeeField> fromCode(
            String code) {
        return Optional.ofNullable(BY_CODE.get(code));
    }
}
```

Fluxo:

```text
"nr_name"
    ↓
EmployeeField.fromCode(...)
    ↓
EmployeeField.NAME
```

---

## 6. Employee Application

```text
application/employee
├── input
│   ├── GetEmployeeByIdInput
│   ├── GetEmployeesInput
│   └── GetEmployeePhotoByIdInput
├── output
│   ├── EmployeeRepositoryOutput
│   └── EmployeePhotoOutput
└── usecase
    ├── GetEmployeeByIdUseCase
    ├── GetEmployeesUseCase
    └── GetEmployeePhotoByIdUseCase
```

Exemplo:

```java
public interface GetEmployeeByIdInput {
    EmployeeResponse execute(Long id);
}
```

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

        List<String> fields =
                allowedFields.execute(
                        employee.getMatricula());

        return presenter.present(
                employee,
                fields);
    }
}
```

---

## 7. ThirdParty Application

```text
application/thirdparty
├── input
│   ├── GetThirdPartyByIdInput
│   ├── GetThirdPartiesInput
│   └── GetThirdPartyPhotoByIdInput
├── output
│   ├── ThirdPartyRepositoryOutput
│   └── ThirdPartyPhotoOutput
└── usecase
    ├── GetThirdPartyByIdUseCase
    ├── GetThirdPartiesUseCase
    └── GetThirdPartyPhotoByIdUseCase
```

A estrutura é paralela à de Employee, mas os modelos e contratos podem ser completamente diferentes.

---

## 8. Domain

```text
domain
├── employee
│   ├── model
│   │   ├── Employee
│   │   └── Contato
│   └── enums
│       └── EmployeeField
└── thirdparty
    └── model
        └── ThirdParty
```

O domínio não conhece:

- JPA
- Redis
- Controller
- Response DTO
- infraestrutura

---

## 9. Persistence

```text
infrastructure/persistence
├── employee
│   ├── entity
│   └── mapper
└── thirdparty
    ├── entity
    └── mapper
```

### EmployeeEntity

Pode representar a tabela grande:

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

    // getters
}
```

### Persistence Mapper

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

Responsabilidade:

```text
EmployeeEntity → Employee
```

---

## 10. Presentation

Cada contexto possui sua própria apresentação:

```text
presentation
├── employee
│   ├── dto
│   ├── factory
│   ├── mapper
│   ├── presenter
│   └── rules
└── thirdparty
    ├── dto
    ├── factory
    ├── mapper
    ├── presenter
    └── rules
```

Isso permite que Employee e ThirdParty tenham contratos diferentes.

---

## 11. Employee Factory

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

Responsabilidade:

```text
CRIAR E INICIALIZAR A ÁRVORE DO RESPONSE
```

---

## 12. Employee Rules

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

Responsabilidade:

```text
EmployeeField → como preencher EmployeeResponse
```

---

## 13. Employee Presenter

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
            List<String> allowedFields) {

        EmployeeResponse target =
                factory.create();

        allowedFields.stream()
                .map(EmployeeField::fromCode)
                .flatMap(Optional::stream)
                .map(rules::get)
                .filter(Objects::nonNull)
                .forEach(rule ->
                        rule.accept(
                                source,
                                target));

        return target;
    }
}
```

Fluxo:

```text
List<String>
    ↓
EmployeeField
    ↓
Rules
    ↓
target
    ↓
EmployeeResponse
```

`rule.accept(source, target)` modifica o mesmo objeto criado pela Factory.

---

## 14. Employee Mapper

O Mapper fica reservado para transformações específicas e reutilizáveis.

Exemplo:

```java
@Component
public class EmployeeResponseMapper {

    public ContactItem toContactItem(
            String name,
            String value) {

        return new ContactItem(
                name,
                value);
    }
}
```

O Mapper não decide quais campos são permitidos.

---

## 15. Controllers

```text
infrastructure/input
├── employee
│   └── EmployeeController
└── thirdparty
    └── ThirdPartyController
```

Employee:

```java
@RestController
@RequestMapping("/employees")
public class EmployeeController {

    private final GetEmployeeByIdInput getById;

    public EmployeeController(
            GetEmployeeByIdInput getById) {
        this.getById = getById;
    }

    @GetMapping("/{id}")
    public EmployeeResponse getById(
            @PathVariable Long id) {

        return getById.execute(id);
    }
}
```

Os demais endpoints recebem seus respectivos `Input`.

---

## 16. Output

```text
infrastructure/output
├── employee
│   ├── repository
│   └── photo
├── thirdparty
│   ├── repository
│   └── photo
└── permission
    └── redis
```

Cada adapter implementa uma porta definida na Application.

Exemplo:

```text
application
    ↓
EmployeeRepositoryOutput
    ↑
EmployeeRepositoryAdapter
```

E:

```text
application
    ↓
AllowedFieldsOutput
    ↑
AllowedFieldsRedisAdapter
```

---

## 17. Fluxo Employee

```text
GET /employees/{id}
          │
          ▼
EmployeeController
          │
          ▼
GetEmployeeByIdUseCase
          │
          ├── EmployeeRepositoryOutput
          │        ↓
          │   EmployeeRepositoryAdapter
          │        ↓
          │   EmployeeEntity
          │        ↓
          │   PersistenceMapper
          │        ↓
          │      Employee
          │
          └── GetAllowedFieldsInput
                   ↓
             GetAllowedFieldsUseCase
                   ↓
             AllowedFieldsOutput
                   ↓
                  Redis
                   ↓
              List<String>
                   ↓
           EmployeePresenter
                   ↓
       ┌───────────┼───────────┐
       ↓           ↓           ↓
    Factory      Rules       Mapper
       │           │           │
       └───────────┼───────────┘
                   ↓
          EmployeeResponse
```

---

## 18. Fluxo ThirdParty

```text
GET /third-parties/{id}
          │
          ▼
ThirdPartyController
          │
          ▼
GetThirdPartyByIdUseCase
          │
          ├── ThirdPartyRepositoryOutput
          │
          └── GetAllowedFieldsInput
                   ↓
             GetAllowedFieldsUseCase
                   ↓
                  Redis
                   ↓
              List<String>
                   ↓
          ThirdPartyPresenter
                   ↓
       ┌───────────┼───────────┐
       ↓           ↓           ↓
    Factory      Rules       Mapper
       │           │           │
       └───────────┼───────────┘
                   ↓
         ThirdPartyResponse
```

---

## 19. Regra de organização

A regra prática final é:

```text
Contexto
   │
   ├── Application
   │   ├── Input
   │   ├── Output
   │   └── UseCase
   │
   ├── Domain
   │
   └── Infrastructure
       ├── Input
       ├── Output
       ├── Persistence
       └── Presentation
```

Contextos:

```text
Employee
ThirdParty
Permission
```

---

## 20. Benefícios

### Navegação

Para alterar Employee:

```text
application/employee
domain/employee
infrastructure/.../employee
```

Para alterar ThirdParty:

```text
application/thirdparty
domain/thirdparty
infrastructure/.../thirdparty
```

Para alterar permissões:

```text
application/permission
infrastructure/output/permission
```

### Evita packages gigantes

Evita:

```text
application
├── input
├── output
└── usecase
```

com dezenas de classes misturadas.

### Evita abstração prematura

Employee e ThirdParty podem compartilhar código somente quando houver uma necessidade real.

Não é necessário criar uma superclasse ou genericamente abstrair os dois apenas porque os endpoints possuem nomes semelhantes.

---

## 21. Arquitetura final resumida

```text
                    APPLICATION
                         │
          ┌──────────────┼──────────────┐
          │              │              │
       Employee       ThirdParty     Permission
          │              │              │
        3 UC            3 UC            1 UC
          │              │              │
          └──────────────┼──────────────┘
                         │
                       DOMAIN
                    ┌────┴────┐
                    │         │
                 Employee  ThirdParty
                    │         │
                    └────┬────┘
                         │
                  INFRASTRUCTURE
                         │
          ┌──────────────┼──────────────┐
          │              │              │
        Input          Output       Presentation
          │              │              │
     Controllers      Adapters      DTO/Factory
                                    Rules/Mapper
                                    Presenter
```

## 22. Decisão arquitetural

A decisão principal é:

> **Organizar os packages primeiro pelo contexto funcional e, dentro de cada contexto, separar Input, Output e UseCase na Application, mantendo Domain e Infrastructure separados.**

Isso preserva a arquitetura hexagonal e torna o projeto mais fácil de navegar e evoluir com os 7 casos de uso.


---

# 23. Command x Query

Com a evolução dos endpoints, nem todo caso de uso precisa receber um `Command` ou `Query`.

A decisão deve considerar a complexidade dos parâmetros.

## 23.1 Operações simples

Para uma operação com um único parâmetro, manter o método simples:

```java
public interface GetEmployeeByIdInput {

    EmployeeResponse execute(Long id);
}
```

Uso:

```java
useCase.execute(id);
```

Não há necessidade de criar:

```java
GetEmployeeByIdCommand
```

apenas para encapsular um único `Long`.

---

# 24. Quando usar Query

`GetEmployees` é uma operação de consulta.

Se ela receber vários critérios, é apropriado encapsular os parâmetros em uma `Query`.

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

O Input:

```java
public interface GetEmployeesInput {

    EmployeeResponse execute(
            GetEmployeesQuery query);
}
```

O Use Case:

```java
@Service
public class GetEmployeesUseCase
        implements GetEmployeesInput {

    private final EmployeeRepositoryOutput repository;
    private final EmployeePresenter presenter;

    public GetEmployeesUseCase(
            EmployeeRepositoryOutput repository,
            EmployeePresenter presenter) {

        this.repository = repository;
        this.presenter = presenter;
    }

    @Override
    public EmployeeResponse execute(
            GetEmployeesQuery query) {

        List<Employee> employees =
                repository.findAll(
                        query.ids(),
                        query.name(),
                        query.department(),
                        query.page(),
                        query.size());

        return presenter.present(employees);
    }
}
```

O Controller cria a Query:

```java
@GetMapping
public EmployeeResponse getAll(
        @RequestParam List<Long> ids,
        @RequestParam(required = false) String name,
        @RequestParam(required = false) String department,
        @RequestParam(defaultValue = "0") Integer page,
        @RequestParam(defaultValue = "20") Integer size) {

    var query = new GetEmployeesQuery(
            ids,
            name,
            department,
            page,
            size
    );

    return useCase.execute(query);
}
```

---

# 25. Command x Query — conceito

A distinção pode ser simples:

```text
Command
    → representa uma intenção/operação

Query
    → representa uma consulta
```

No contexto deste microserviço:

```text
GET /employees/{id}
    → consulta simples
    → execute(Long id)

GET /employees
    → consulta com vários critérios
    → execute(GetEmployeesQuery)

GET /employees/{id}/photo
    → consulta simples
    → execute(Long id)
```

Portanto, não é necessário transformar todos os métodos em objetos.

---

# 26. Quando usar Command

Um `Command` passa a fazer sentido quando uma operação possui vários dados de entrada ou representa uma ação que merece ser encapsulada.

Exemplo:

```java
public record UpdateEmployeeCommand(
        Long id,
        String name,
        String department,
        String phone
) {
}
```

Nesse caso:

```java
useCase.execute(command);
```

é mais interessante do que:

```java
useCase.execute(
        id,
        name,
        department,
        phone
);
```

O Command evita métodos com muitos parâmetros e cria um objeto explícito para a intenção da operação.

---

# 27. Quando usar Query

Use uma `Query` quando a operação é uma consulta e possui parâmetros suficientes para justificar o encapsulamento.

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

Ela representa:

```text
"Quero consultar Employees usando estes critérios."
```

Isso também facilita a evolução.

Hoje:

```java
GetEmployeesQuery(
    ids,
    name,
    department
)
```

Amanhã pode receber:

```java
GetEmployeesQuery(
    ids,
    name,
    department,
    status,
    sort,
    page,
    size
)
```

Sem transformar o método `execute()` em uma lista enorme de parâmetros.

---

# 28. Regra prática adotada

A arquitetura passa a seguir esta regra:

```text
Poucos parâmetros
    ↓
execute(parâmetro)
```

Exemplo:

```java
execute(Long id)
```

Quando a entrada fica complexa:

```text
Muitos parâmetros
    ↓
Query ou Command
```

Para consultas:

```text
GetEmployeesQuery
```

Para operações de alteração:

```text
UpdateEmployeeCommand
CreateEmployeeCommand
DeleteEmployeeCommand
```

---

# 29. Estrutura de packages com Query

Para Employee:

```text
application
└── employee
    ├── input
    │   ├── GetEmployeeByIdInput.java
    │   ├── GetEmployeesInput.java
    │   └── GetEmployeePhotoByIdInput.java
    │
    ├── query
    │   └── GetEmployeesQuery.java
    │
    ├── output
    │   ├── EmployeeRepositoryOutput.java
    │   └── EmployeePhotoOutput.java
    │
    └── usecase
        ├── GetEmployeeByIdUseCase.java
        ├── GetEmployeesUseCase.java
        └── GetEmployeePhotoByIdUseCase.java
```

Não é necessário criar:

```text
query
├── GetEmployeeByIdQuery
├── GetEmployeesQuery
└── GetEmployeePhotoByIdQuery
```

se os outros dois casos possuem apenas um `id`.

---

# 30. Estrutura futura com Commands

Se futuramente o microserviço passar a ter operações de escrita:

```text
application
└── employee
    ├── command
    │   ├── CreateEmployeeCommand.java
    │   ├── UpdateEmployeeCommand.java
    │   └── DeleteEmployeeCommand.java
    │
    ├── query
    │   └── GetEmployeesQuery.java
    │
    ├── input
    ├── output
    └── usecase
```

Assim:

```text
Command
    → alteração/ação

Query
    → consulta
```

---

# 31. Decisão para o projeto atual

Para os seis endpoints atuais:

```text
Employee

GetEmployeeById
    → execute(Long id)

GetEmployees
    → execute(GetEmployeesQuery)

GetEmployeePhotoById
    → execute(Long id)


ThirdParty

GetThirdPartyById
    → execute(Long id)

GetThirdParties
    → execute(GetThirdPartiesQuery)

GetThirdPartyPhotoById
    → execute(Long id)
```

E:

```text
AllowedFields

GetAllowedFields
    → execute(String matricula)
```

Caso a consulta de `ThirdParty` também possua múltiplos critérios, pode utilizar:

```java
GetThirdPartiesQuery
```

independentemente de Employee.

---

# 32. Decisão arquitetural final atualizada

A regra geral da arquitetura é:

```text
                    APPLICATION
                         │
          ┌──────────────┼──────────────┐
          │              │              │
       Employee       ThirdParty     Permission
          │              │              │
        3 UC            3 UC            1 UC
          │              │              │
          │              │              │
     Query quando    Query quando       │
     necessário      necessário         │
          │              │              │
          └──────────────┼──────────────┘
                         │
                       DOMAIN
                         │
                  INFRASTRUCTURE
```

A utilização de `Command` ou `Query` **não é obrigatória para todos os casos de uso**.

Ela deve existir quando melhorar a representação da entrada do caso de uso.

A prioridade continua sendo:

```text
Simplicidade
    ↓
Clareza
    ↓
Baixo acoplamento
    ↓
Extensibilidade quando necessária
```

Não criar `Command`/`Query` apenas por seguir um padrão.

---

# 33. Resumo da decisão

| Caso | Entrada recomendada |
|---|---|
| GetEmployeeById | `Long id` |
| GetEmployeePhotoById | `Long id` |
| GetEmployees com vários filtros | `GetEmployeesQuery` |
| GetThirdPartyById | `Long id` |
| GetThirdPartyPhotoById | `Long id` |
| GetThirdParties com vários filtros | `GetThirdPartiesQuery` |
| GetAllowedFields | `String matricula` |
| CreateEmployee futuro | `CreateEmployeeCommand` |
| UpdateEmployee futuro | `UpdateEmployeeCommand` |
| DeleteEmployee futuro | `DeleteEmployeeCommand` |

A ideia central é **usar o objeto de entrada quando ele realmente agrega valor**, e não transformar todos os métodos em `Command` ou `Query` por obrigação arquitetural.
