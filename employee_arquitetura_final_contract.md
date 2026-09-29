# Arquitetura Final — Employee Service

## 1. Princípios

Arquitetura hexagonal simples, organizada por:

- `domain` → modelo de negócio
- `application` → contexto de negócio e casos de uso
- `infrastructure/input` → entrada na aplicação
- `infrastructure/output` → saída da aplicação
- `contract` → construção do contrato externo

Evitar packages criados apenas para seguir padrões. Não existe `EmployeeFieldResolver`.

---

## 2. Estrutura

```text
com.company.employee
│
├── domain
│   ├── Employee.java
│   ├── ThirdParty.java
│   ├── Contato.java
│   ├── EmployeeField.java
│   └── AllowedField.java
│
├── application
│   ├── employee
│   │   ├── input
│   │   │   ├── GetEmployeeByIdInput.java
│   │   │   ├── GetEmployeesInput.java
│   │   │   └── GetEmployeePhotoByIdInput.java
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
│   │   └── usecase
│   │       ├── GetThirdPartyByIdUseCase.java
│   │       ├── GetThirdPartiesUseCase.java
│   │       └── GetThirdPartyPhotoByIdUseCase.java
│   │
│   ├── permission
│   │   ├── input
│   │   │   └── GetAllowedFieldsInput.java
│   │   └── usecase
│   │       └── GetAllowedFieldsUseCase.java
│   │
│   └── output
│       ├── EmployeeOutput.java
│       ├── ThirdPartyOutput.java
│       └── AllowedFieldsOutput.java
│
└── infrastructure
    ├── input
    │   └── rest
    │       ├── employee
    │       │   ├── EmployeeController.java
    │       │   └── contract
    │       │       ├── EmployeeResponse.java
    │       │       ├── EmployeePresenter.java
    │       │       ├── EmployeeResponseFactory.java
    │       │       ├── EmployeeResponseRules.java
    │       │       └── EmployeeResponseMapper.java
    │       │
    │       └── thirdparty
    │           ├── ThirdPartyController.java
    │           └── contract
    │               ├── ThirdPartyResponse.java
    │               ├── ThirdPartyPresenter.java
    │               ├── ThirdPartyResponseFactory.java
    │               ├── ThirdPartyResponseRules.java
    │               └── ThirdPartyResponseMapper.java
    │
    └── output
        ├── jpa
        │   ├── employee
        │   │   ├── EmployeeRepositoryAdapter.java
        │   │   ├── EmployeeJpaRepository.java
        │   │   ├── EmployeeEntity.java
        │   │   └── EmployeePersistenceMapper.java
        │   └── thirdparty
        │       ├── ThirdPartyRepositoryAdapter.java
        │       ├── ThirdPartyJpaRepository.java
        │       ├── ThirdPartyEntity.java
        │       └── ThirdPartyPersistenceMapper.java
        │
        └── redis
            └── permission
                └── AllowedFieldsRedisAdapter.java
```

---

# 3. Domain

O domínio não conhece Spring, JPA, Redis ou REST.

### Employee

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

### Contato

```java
public class Contato {

    private String telefone;
    private String celular;

    // getters
}
```

O domínio possui `telefone` e `celular`. Ele não precisa conhecer a lista genérica do contrato.

---

# 4. AllowedField

O Redis possui códigos como:

```text
nr_name
nr_cpf
nr_phone
```

A infraestrutura converte esses valores para:

```java
public record AllowedField(String name) {
}
```

A aplicação trabalha com:

```text
List<AllowedField>
```

e não espalha `String` do Redis pelo código.

---

# 5. EmployeeField

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

    public static Optional<EmployeeField> fromCode(String code) {
        return Optional.ofNullable(BY_CODE.get(code));
    }
}
```

Não existe `EmployeeFieldResolver`.

A conversão acontece diretamente no Presenter.

---

# 6. Application

A Application é organizada por contexto:

```text
employee
thirdparty
permission
```

Dentro de cada contexto:

```text
input
usecase
```

Os Outputs ficam em:

```text
application/output
```

porque são portas de saída da aplicação.

---

# 7. Inputs

Exemplo:

```java
public interface GetEmployeeByIdInput {

    EmployeeResponse execute(Long id);
}
```

Permission:

```java
public interface GetAllowedFieldsInput {

    List<AllowedField> execute(String matricula);
}
```

---

# 8. Outputs

```java
public interface EmployeeOutput {

    Optional<Employee> findById(Long id);

    List<Employee> findAll(GetEmployeesQuery query);
}
```

```java
public interface AllowedFieldsOutput {

    List<AllowedField> findByMatricula(String matricula);
}
```

A Application não sabe se a implementação usa JPA, Redis, JDBC ou outra tecnologia.

---

# 9. Use Case — Employee

```java
@Service
public class GetEmployeeByIdUseCase
        implements GetEmployeeByIdInput {

    private final EmployeeOutput employeeOutput;
    private final GetAllowedFieldsInput allowedFields;
    private final EmployeePresenter presenter;

    public GetEmployeeByIdUseCase(
            EmployeeOutput employeeOutput,
            GetAllowedFieldsInput allowedFields,
            EmployeePresenter presenter) {

        this.employeeOutput = employeeOutput;
        this.allowedFields = allowedFields;
        this.presenter = presenter;
    }

    @Override
    public EmployeeResponse execute(Long id) {

        Employee employee =
                employeeOutput.findById(id)
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

O Use Case apenas orquestra.

---

# 10. Use Case — Permission

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

        return output.findByMatricula(matricula);
    }
}
```

---

# 11. Query e Command

Consulta simples:

```java
execute(Long id)
```

Consulta com vários critérios:

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

Command deve ser usado quando houver uma operação com entrada complexa.

Não criar Command ou Query apenas por obrigação.

---

# 12. Infrastructure Input — REST

Tudo que entra pela API fica em:

```text
infrastructure/input/rest
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

Fluxo:

```text
REST
 ↓
Input
 ↓
UseCase
```

---

# 13. Contract

`contract` substitui `presentation`.

Ele representa especificamente a construção do contrato externo da API.

Cada contexto possui seu próprio contrato:

```text
employee/contract
thirdparty/contract
```

Isso permite que Employee e ThirdParty tenham respostas completamente diferentes.

---

# 14. EmployeeResponseFactory

A Factory cria a estrutura do contrato.

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

A Factory cria o `target`.

Ela não aplica permissões.

---

# 15. EmployeeResponseRules

As Rules dizem como cada `EmployeeField` preenche o contrato.

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
                      .setCidade(source.getCidade()),

            EmployeeField.STATE,
            (source, target) ->
                target.getAddress()
                      .getLocation()
                      .setEstado(source.getEstado()),

            EmployeeField.DEPARTMENT,
            (source, target) ->
                target.getOrganization()
                      .getDepartment()
                      .setId(source.getDepartmentId())
        );
    }
}
```

O Map é imutável e não precisa ser Spring Bean.

---

# 16. EmployeePresenter

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

O fluxo:

```text
AllowedField
    ↓
EmployeeField.fromCode()
    ↓
Rule
    ↓
target
```

O `rule.accept(source, target)` modifica o mesmo `target` criado pela Factory.

---

# 17. EmployeeResponseMapper

Usado para transformações auxiliares mais complexas.

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

Não é necessário criar um Mapper para cada campo.

---

# 18. Lista genérica de contatos

Domínio:

```text
Contato
├── telefone
└── celular
```

Contrato:

```text
ContactItem
├── itemName
└── itemId
```

Rule:

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

Assim o domínio não conhece a estrutura genérica do contrato.

---

# 19. Infrastructure Output — JPA

JPA representa uma saída da aplicação.

```text
infrastructure
└── output
    └── jpa
        ├── employee
        │   ├── EmployeeRepositoryAdapter.java
        │   ├── EmployeeJpaRepository.java
        │   ├── EmployeeEntity.java
        │   └── EmployeePersistenceMapper.java
        │
        └── thirdparty
            ├── ThirdPartyRepositoryAdapter.java
            ├── ThirdPartyJpaRepository.java
            ├── ThirdPartyEntity.java
            └── ThirdPartyPersistenceMapper.java
```

---

# 20. JPA Entity

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

A Entity representa a tabela, não o contrato.

---

# 21. Persistence Mapper

Responsabilidade:

```text
Entity → Domain
```

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

# 22. Repository Adapter

```java
@Component
public class EmployeeRepositoryAdapter
        implements EmployeeOutput {

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

Fluxo:

```text
EmployeeOutput
      ▲
      │
EmployeeRepositoryAdapter
      │
      ▼
EmployeeJpaRepository
      │
      ▼
EmployeeEntity
```

---

# 23. Infrastructure Output — Redis

```text
infrastructure
└── output
    └── redis
        └── permission
            └── AllowedFieldsRedisAdapter.java
```

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

Fluxo:

```text
Redis
 ↓
List<String>
 ↓
AllowedFieldsRedisAdapter
 ↓
List<AllowedField>
 ↓
Application
```

---

# 24. Fluxo completo

```text
                         HTTP
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
              ┌───────────┴───────────┐
              │                       │
              ▼                       ▼
       EmployeeOutput        GetAllowedFieldsInput
              │                       │
              ▼                       ▼
             JPA             GetAllowedFieldsUseCase
                                      │
                                      ▼
                             AllowedFieldsOutput
                                      │
                                      ▼
                                    Redis
                                      │
                                      ▼
                              List<AllowedField>
                                      │
                                      ▼
                              EmployeePresenter
                                      │
                         ┌────────────┼────────────┐
                         ▼            ▼            ▼
                      Factory       Rules      EmployeeField
                         │            │            │
                         └────────────┴────────────┘
                                      │
                                      ▼
                              EmployeeResponse
                                      │
                                      ▼
                                     HTTP
```

---

# 25. Responsabilidades

| Classe | Responsabilidade |
|---|---|
| `Employee` | Modelo de domínio |
| `ThirdParty` | Modelo de domínio |
| `Contato` | Modelo de domínio |
| `EmployeeField` | Campos do Employee |
| `AllowedField` | Campo permitido |
| `Controller` | Entrada REST |
| `Input` | Porta de entrada |
| `UseCase` | Orquestra o caso de uso |
| `Output` | Porta de saída |
| `Adapter` | Implementa uma porta de saída |
| `Entity` | Modelo de persistência |
| `PersistenceMapper` | Entity → Domain |
| `Response` | Contrato HTTP |
| `Factory` | Cria a estrutura do contrato |
| `Rules` | Define o preenchimento dos campos |
| `Presenter` | Aplica as Rules |
| `ResponseMapper` | Transformações auxiliares do contrato |

---

# 26. Query x Command

Regra simples:

```text
execute(Long id)
```

quando a entrada é simples.

```text
execute(GetEmployeesQuery query)
```

quando uma consulta possui vários critérios.

```text
execute(UpdateEmployeeCommand command)
```

quando uma operação possui uma entrada complexa.

Não criar abstrações somente para seguir um padrão.

---

# 27. Decisão arquitetural final

A estrutura usa três níveis principais:

```text
domain
application
infrastructure
```

Application:

```text
application
├── employee
│   ├── input
│   └── usecase
├── thirdparty
│   ├── input
│   └── usecase
├── permission
│   ├── input
│   └── usecase
└── output
```

Infrastructure:

```text
infrastructure
├── input
│   └── rest
│       ├── employee
│       │   └── contract
│       └── thirdparty
│           └── contract
│
└── output
    ├── jpa
    │   ├── employee
    │   └── thirdparty
    └── redis
        └── permission
```

O conceito central é:

```text
Application
    │
    ├── Input  ← entrada
    │
    └── Output → saída
          │
          ▼
Infrastructure
```

E:

```text
REST
  ↓
Contract
  ↓
Application
  ↓
JPA / Redis
```

A arquitetura permanece hexagonal, mas sem excesso de abstrações ou packages.
