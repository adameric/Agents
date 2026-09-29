# Arquitetura Hexagonal — Employee Service

## 1. Objetivo

Microserviço Java 21 utilizando arquitetura hexagonal.

Contextos:

- Employee
- ThirdParty
- Permission

Endpoints:

### Employee
- GetEmployeeById
- GetEmployees
- GetEmployeePhotoById

### ThirdParty
- GetThirdPartyById
- GetThirdParties
- GetThirdPartyPhotoById

### Permission
- GetAllowedFields

---

# 2. Estrutura

```text
application
├── employee
│   ├── input
│   ├── query
│   ├── output
│   └── usecase
│
├── thirdparty
│   ├── input
│   ├── query
│   ├── output
│   └── usecase
│
└── permission
    ├── input
    ├── output
    └── usecase

domain
├── employee
├── thirdparty
└── permission

infrastructure
├── input
├── output
├── persistence
└── presentation
```

---

# 3. Permission

O Redis armazena códigos:

```text
nr_name
nr_cpf
nr_phone
```

A infraestrutura converte esses valores para:

```java
List<AllowedField>
```

A aplicação não trabalha diretamente com `String`.

---

# 4. AllowedField

```java
package com.example.employee.domain.permission.model;

public record AllowedField(
        String name
) {
}
```

Exemplos:

```java
new AllowedField("nr_name");
new AllowedField("nr_cpf");
new AllowedField("nr_phone");
```

---

# 5. AllowedFieldsOutput

```java
package com.example.employee.application.permission.output;

import com.example.employee.domain.permission.model.AllowedField;

import java.util.List;

public interface AllowedFieldsOutput {

    List<AllowedField> findByMatricula(
            String matricula);
}
```

A Application não conhece Redis.

---

# 6. GetAllowedFieldsInput

```java
package com.example.employee.application.permission.input;

import com.example.employee.domain.permission.model.AllowedField;

import java.util.List;

public interface GetAllowedFieldsInput {

    List<AllowedField> execute(
            String matricula);
}
```

---

# 7. GetAllowedFieldsUseCase

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

# 8. Redis Adapter

O Redis retorna `String`, mas o adapter converte imediatamente para o objeto utilizado pela aplicação.

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
```

---

# 9. EmployeeField

`EmployeeField` continua específico do contexto Employee.

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

---

# 10. EmployeeFieldResolver

`AllowedField` não conhece `EmployeeField`.

A conversão é responsabilidade do contexto Employee.

```java
public final class EmployeeFieldResolver {

    private EmployeeFieldResolver() {
    }

    public static Optional<EmployeeField> resolve(
            AllowedField field) {

        return EmployeeField.fromCode(
                field.name());
    }
}
```

Assim:

```text
AllowedField
     ↓
EmployeeFieldResolver
     ↓
EmployeeField
```

---

# 11. EmployeeResponseFactory

A Factory cria a árvore completa do contrato.

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

A Factory não conhece permissões.

---

# 12. EmployeeResponseRules

As Rules sabem como preencher cada campo.

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

---

# 13. EmployeePresenter

O Presenter recebe `List<AllowedField>`.

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
                .map(EmployeeFieldResolver::resolve)
                .flatMap(Optional::stream)
                .map(rules::get)
                .filter(Objects::nonNull)
                .forEach(rule ->
                        rule.accept(source, target));

        return target;
    }
}
```

O ponto principal é:

```java
rule.accept(source, target);
```

A Rule modifica o `target` criado pela Factory.

---

# 14. GetEmployeeByIdUseCase

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

---

# 15. Employee GetEmployees

Como `GetEmployees` pode receber vários IDs, filtros, paginação etc., utiliza uma Query.

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

Input:

```java
public interface GetEmployeesInput {

    EmployeeResponse execute(
            GetEmployeesQuery query);
}
```

A regra é:

```text
Poucos parâmetros
    ↓
execute(parâmetro)

Consulta complexa
    ↓
Query

Operação de alteração complexa
    ↓
Command
```

Não é necessário criar Query ou Command para todos os casos de uso.

---

# 16. Command x Query

## Query

Representa uma consulta.

Exemplo:

```java
GetEmployeesQuery
```

Uso:

```java
useCase.execute(query);
```

## Command

Representa uma intenção/operação, normalmente utilizada quando uma operação possui vários dados de entrada.

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

Para:

```java
execute(Long id)
```

não há necessidade de criar um Command apenas para encapsular um único parâmetro.

---

# 17. Persistência

A tabela Employee pode ser enorme.

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
}
```

O Persistence Mapper converte:

```text
EmployeeEntity
      ↓
Employee
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

# 18. Separação entre os Mappers

Existem responsabilidades diferentes.

## Persistence Mapper

```text
Entity → Domain
```

## Presentation Mapper

```text
Domain → pequenos objetos do Response
```

## Presenter

```text
AllowedField
    ↓
EmployeeField
    ↓
Rule
    ↓
Response
```

Não concentrar todas essas responsabilidades em um único Mapper.

---

# 19. Fluxo completo

```text
GET /employees/{id}
          │
          ▼
EmployeeController
          │
          ▼
GetEmployeeByIdUseCase
          │
          ├───────────────► EmployeeRepositoryOutput
          │                         │
          │                         ▼
          │                      Employee
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
                                    ▼
                       EmployeeFieldResolver
                                    │
                                    ▼
                         EmployeeResponseRules
                                    │
                                    ▼
                            EmployeeResponse
```

---

# 20. Responsabilidades

| Classe | Responsabilidade |
|---|---|
| `AllowedField` | Representar um campo permitido |
| `EmployeeField` | Representar campo do contexto Employee |
| `AllowedFieldsRedisAdapter` | Converter dado do Redis para domínio |
| `EmployeeFieldResolver` | Resolver `AllowedField` para `EmployeeField` |
| `EmployeeResponseFactory` | Criar árvore do Response |
| `EmployeeResponseRules` | Definir como preencher cada campo |
| `EmployeePresenter` | Orquestrar as Rules |
| `EmployeePersistenceMapper` | Converter Entity para Domain |
| `GetAllowedFieldsUseCase` | Consultar campos permitidos |
| `GetEmployeeByIdUseCase` | Orquestrar busca do Employee |

---

# 21. Decisão arquitetural final

A fronteira externa utiliza a representação do sistema externo:

```text
Redis
    ↓
List<String>
```

O Adapter converte para um objeto semântico:

```text
List<AllowedField>
```

A aplicação passa a trabalhar exclusivamente com:

```text
AllowedField
```

E o contexto Employee faz sua própria resolução:

```text
AllowedField
    ↓
EmployeeField
    ↓
EmployeeResponseRules
    ↓
EmployeeResponse
```

Isso evita espalhar códigos `String` de Redis pela aplicação e mantém Permission desacoplado de Employee.

---

# 22. Arquitetura consolidada

```text
                         HTTP
                          │
                          ▼
                    Controller
                          │
                          ▼
                     Input Port
                          │
                          ▼
                       UseCase
                          │
              ┌───────────┴───────────┐
              │                       │
              ▼                       ▼
       Repository Output       AllowedFields Input
              │                       │
              ▼                       ▼
       Repository Adapter      AllowedFields UseCase
              │                       │
              ▼                       ▼
          Database             AllowedFields Output
                                      │
                                      ▼
                                     Redis
                                      │
                                      ▼
                                List<String>
                                      │
                                      ▼
                                  Adapter
                                      │
                                      ▼
                              List<AllowedField>
                                      │
                                      ▼
                                  Presenter
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
                 Factory           Resolver           Rules
                    │                 │                 │
                    └─────────────────┼─────────────────┘
                                      ▼
                               Response Contract
```

## Regra final

Não criar abstrações apenas por padrão.

Usar:

```text
execute(Long id)
```

quando a entrada for simples.

Usar:

```text
execute(GetEmployeesQuery query)
```

quando a consulta possuir vários critérios.

Usar `Command` quando uma operação de ação/alteração possuir entrada complexa.

E manter:

```text
Redis String
    ↓
Adapter
    ↓
AllowedField
```

como fronteira entre infraestrutura e aplicação.
