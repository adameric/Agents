# Microserviço Employee — Arquitetura Hexagonal Final

## 1. Objetivo

Microserviço Java 21 utilizando arquitetura hexagonal, com:

- endpoint `GET Employee by ID`;
- entidade de persistência correspondente a uma tabela grande;
- domínio independente da estrutura da tabela;
- response altamente normalizado;
- permissões/campos permitidos consultados no Redis;
- Redis retornando `List<String>` com códigos como `nr_name`;
- `EmployeeField` traduzindo o código externo para um conceito interno;
- montagem dinâmica do response sem dezenas de `if`;
- Factory para inicializar a estrutura do response;
- Rules para definir como cada campo preenche o response;
- Presenter para orquestrar a montagem;
- Mapper para transformações específicas.

---

# 2. Arquitetura final

```text
employee-service
│
├── application
│   ├── input
│   │   └── GetEmployeeInput.java
│   ├── output
│   │   ├── EmployeeRepositoryOutput.java
│   │   └── EmployeePermissionOutput.java
│   └── usecase
│       └── GetEmployeeUseCase.java
│
├── domain
│   ├── model
│   │   ├── Employee.java
│   │   └── Contato.java
│   └── enums
│       └── EmployeeField.java
│
└── infrastructure
    ├── input
    │   └── controller
    │       └── EmployeeController.java
    │
    ├── output
    │   ├── repository
    │   │   ├── EmployeeRepositoryAdapter.java
    │   │   └── EmployeeJpaRepository.java
    │   └── redis
    │       └── EmployeePermissionAdapter.java
    │
    ├── persistence
    │   ├── entity
    │   │   └── EmployeeEntity.java
    │   └── mapper
    │       └── EmployeePersistenceMapper.java
    │
    └── presentation
        ├── dto
        │   ├── EmployeeResponse.java
        │   ├── PersonalData.java
        │   ├── Address.java
        │   ├── Location.java
        │   ├── Organization.java
        │   ├── Department.java
        │   └── ContactItem.java
        ├── mapper
        │   └── EmployeeResponseMapper.java
        ├── rules
        │   └── EmployeeResponseRules.java
        ├── factory
        │   └── EmployeeResponseFactory.java
        └── presenter
            └── EmployeePresenter.java
```

---

# 3. Responsabilidades

| Componente | Responsabilidade |
|---|---|
| `Employee` | Modelo de domínio |
| `Contato` | Modelo de domínio para contatos |
| `EmployeeField` | Campos permitidos e tradução dos códigos externos |
| `EmployeeEntity` | Representação da tabela |
| `EmployeePersistenceMapper` | `Entity → Domain` |
| `EmployeeRepositoryOutput` | Porta de saída para Employee |
| `EmployeePermissionOutput` | Porta de saída para permissões |
| `EmployeeRepositoryAdapter` | Implementação da porta de persistência |
| `EmployeePermissionAdapter` | Implementação da porta Redis |
| `EmployeeResponse` | Estrutura de resposta da API |
| `EmployeeResponseFactory` | Criação/inicialização do response |
| `EmployeeResponseRules` | Como cada campo preenche o response |
| `EmployeeResponseMapper` | Transformações específicas/reutilizáveis |
| `EmployeePresenter` | Orquestra a montagem do response |
| `GetEmployeeUseCase` | Orquestra o caso de uso |
| `EmployeeController` | Entrada HTTP |

---

# 4. Fluxo

```text
HTTP
 │
 ▼
EmployeeController
 │
 ▼
GetEmployeeUseCase
 │
 ├─────────────────────────────┐
 │                             │
 ▼                             ▼
EmployeeRepositoryOutput   EmployeePermissionOutput
 │                             │
 ▼                             ▼
RepositoryAdapter           RedisAdapter
 │                             │
 ▼                             ▼
Employee                    List<String>
                              ["nr_name",
                               "nr_cpf",
                               "nr_phone"]
 │                             │
 └──────────────┬──────────────┘
                ▼
       EmployeePresenter
                │
                ▼
          EmployeeField
                │
       nr_name → NAME
       nr_cpf  → CPF
       nr_phone → PHONE
                │
                ▼
      EmployeeResponseRules
                │
                ▼
      EmployeeResponseFactory
                │
                ▼
        EmployeeResponse
                │
                ▼
           Controller
                │
                ▼
              JSON
```

---

# 5. Domínio

## 5.1 Employee

```java
package com.example.employee.domain.model;

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

    public Employee(
            Long id,
            String matricula,
            String nome,
            String cpf,
            String telefone,
            String celular,
            String cidade,
            String estado,
            Long departmentId) {

        this.id = id;
        this.matricula = matricula;
        this.nome = nome;
        this.cpf = cpf;
        this.telefone = telefone;
        this.celular = celular;
        this.cidade = cidade;
        this.estado = estado;
        this.departmentId = departmentId;
    }

    public Long getId() { return id; }
    public String getMatricula() { return matricula; }
    public String getNome() { return nome; }
    public String getCpf() { return cpf; }
    public String getTelefone() { return telefone; }
    public String getCelular() { return celular; }
    public String getCidade() { return cidade; }
    public String getEstado() { return estado; }
    public Long getDepartmentId() { return departmentId; }
}
```

## 5.2 Contato

```java
package com.example.employee.domain.model;

public class Contato {

    private String telefone;
    private String celular;

    public Contato(
            String telefone,
            String celular) {

        this.telefone = telefone;
        this.celular = celular;
    }

    public String getTelefone() { return telefone; }
    public String getCelular() { return celular; }
}
```

---

# 6. EmployeeField

O Redis retorna strings como `nr_name`. O domínio traduz esses códigos para conceitos internos.

```java
package com.example.employee.domain.enums;

import java.util.Arrays;
import java.util.Map;
import java.util.Optional;
import java.util.function.Function;
import java.util.stream.Collectors;

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

    public static Optional<EmployeeField> fromCode(String code) {
        return Optional.ofNullable(BY_CODE.get(code));
    }
}
```

O fluxo é:

```text
"nr_name"
    ↓
EmployeeField.fromCode(...)
    ↓
EmployeeField.NAME
```

---

# 7. Application — Input

```java
package com.example.employee.application.input;

import com.example.employee.infrastructure.presentation.dto.EmployeeResponse;

public interface GetEmployeeInput {

    EmployeeResponse execute(Long id);
}
```

---

# 8. Application — Outputs

## EmployeeRepositoryOutput

```java
package com.example.employee.application.output;

import com.example.employee.domain.model.Employee;

import java.util.Optional;

public interface EmployeeRepositoryOutput {

    Optional<Employee> findById(Long id);
}
```

## EmployeePermissionOutput

```java
package com.example.employee.application.output;

import java.util.List;

public interface EmployeePermissionOutput {

    List<String> findAllowedFields(String matricula);
}
```

A aplicação conhece apenas a porta. Ela não conhece Redis.

---

# 9. Use Case

```java
package com.example.employee.application.usecase;

import com.example.employee.application.input.GetEmployeeInput;
import com.example.employee.application.output.EmployeePermissionOutput;
import com.example.employee.application.output.EmployeeRepositoryOutput;
import com.example.employee.domain.model.Employee;
import com.example.employee.infrastructure.presentation.dto.EmployeeResponse;
import com.example.employee.infrastructure.presentation.presenter.EmployeePresenter;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class GetEmployeeUseCase
        implements GetEmployeeInput {

    private final EmployeeRepositoryOutput employeeRepository;
    private final EmployeePermissionOutput permissionOutput;
    private final EmployeePresenter presenter;

    public GetEmployeeUseCase(
            EmployeeRepositoryOutput employeeRepository,
            EmployeePermissionOutput permissionOutput,
            EmployeePresenter presenter) {

        this.employeeRepository = employeeRepository;
        this.permissionOutput = permissionOutput;
        this.presenter = presenter;
    }

    @Override
    public EmployeeResponse execute(Long id) {

        Employee employee =
                employeeRepository.findById(id)
                        .orElseThrow();

        List<String> allowedFields =
                permissionOutput.findAllowedFields(
                        employee.getMatricula());

        return presenter.present(
                employee,
                allowedFields);
    }
}
```

---

# 10. Persistence

## EmployeeEntity

A Entity pode representar diretamente a tabela grande.

```java
package com.example.employee.infrastructure.persistence.entity;

import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;

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

    public Long getId() { return id; }
    public String getMatricula() { return matricula; }
    public String getNome() { return nome; }
    public String getCpf() { return cpf; }
    public String getTelefone() { return telefone; }
    public String getCelular() { return celular; }
    public String getCidade() { return cidade; }
    public String getEstado() { return estado; }
    public Long getDepartmentId() { return departmentId; }
}
```

## EmployeeJpaRepository

```java
package com.example.employee.infrastructure.output.repository;

import com.example.employee.infrastructure.persistence.entity.EmployeeEntity;
import org.springframework.data.jpa.repository.JpaRepository;

public interface EmployeeJpaRepository
        extends JpaRepository<EmployeeEntity, Long> {
}
```

## EmployeePersistenceMapper

Responsabilidade exclusiva:

```text
EmployeeEntity → Employee
```

```java
package com.example.employee.infrastructure.persistence.mapper;

import com.example.employee.domain.model.Employee;
import com.example.employee.infrastructure.persistence.entity.EmployeeEntity;
import org.springframework.stereotype.Component;

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

## EmployeeRepositoryAdapter

```java
package com.example.employee.infrastructure.output.repository;

import com.example.employee.application.output.EmployeeRepositoryOutput;
import com.example.employee.domain.model.Employee;
import com.example.employee.infrastructure.persistence.mapper.EmployeePersistenceMapper;
import org.springframework.stereotype.Component;

import java.util.Optional;

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

# 11. Redis

## EmployeePermissionAdapter

O Redis retorna uma lista de strings.

```java
package com.example.employee.infrastructure.output.redis;

import com.example.employee.application.output.EmployeePermissionOutput;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
public class EmployeePermissionAdapter
        implements EmployeePermissionOutput {

    @Override
    public List<String> findAllowedFields(
            String matricula) {

        // Implementação real da consulta ao Redis.

        return List.of(
                "nr_name",
                "nr_cpf",
                "nr_phone"
        );
    }
}
```

---

# 12. Presentation DTOs

## EmployeeResponse

```java
package com.example.employee.infrastructure.presentation.dto;

import java.util.List;

public class EmployeeResponse {

    private PersonalData personalData;
    private Address address;
    private Organization organization;
    private List<ContactItem> contacts;

    public PersonalData getPersonalData() {
        return personalData;
    }

    public void setPersonalData(PersonalData personalData) {
        this.personalData = personalData;
    }

    public Address getAddress() {
        return address;
    }

    public void setAddress(Address address) {
        this.address = address;
    }

    public Organization getOrganization() {
        return organization;
    }

    public void setOrganization(Organization organization) {
        this.organization = organization;
    }

    public List<ContactItem> getContacts() {
        return contacts;
    }

    public void setContacts(List<ContactItem> contacts) {
        this.contacts = contacts;
    }
}
```

## PersonalData

```java
package com.example.employee.infrastructure.presentation.dto;

public class PersonalData {

    private String nome;
    private String cpf;

    public String getNome() { return nome; }
    public void setNome(String nome) { this.nome = nome; }

    public String getCpf() { return cpf; }
    public void setCpf(String cpf) { this.cpf = cpf; }
}
```

## Address

```java
package com.example.employee.infrastructure.presentation.dto;

public class Address {

    private Location location;

    public Location getLocation() {
        return location;
    }

    public void setLocation(Location location) {
        this.location = location;
    }
}
```

## Location

```java
package com.example.employee.infrastructure.presentation.dto;

public class Location {

    private String cidade;
    private String estado;

    public String getCidade() { return cidade; }
    public void setCidade(String cidade) { this.cidade = cidade; }

    public String getEstado() { return estado; }
    public void setEstado(String estado) { this.estado = estado; }
}
```

## Organization

```java
package com.example.employee.infrastructure.presentation.dto;

public class Organization {

    private Department department;

    public Department getDepartment() {
        return department;
    }

    public void setDepartment(Department department) {
        this.department = department;
    }
}
```

## Department

```java
package com.example.employee.infrastructure.presentation.dto;

public class Department {

    private Long id;

    public Long getId() { return id; }

    public void setId(Long id) {
        this.id = id;
    }
}
```

## ContactItem

```java
package com.example.employee.infrastructure.presentation.dto;

public class ContactItem {

    private String itemName;
    private String itemId;

    public ContactItem() {
    }

    public ContactItem(
            String itemName,
            String itemId) {

        this.itemName = itemName;
        this.itemId = itemId;
    }

    public String getItemName() { return itemName; }
    public void setItemName(String itemName) {
        this.itemName = itemName;
    }

    public String getItemId() { return itemId; }
    public void setItemId(String itemId) {
        this.itemId = itemId;
    }
}
```

---

# 13. Response Factory

A Factory inicializa toda a árvore do response.

```java
package com.example.employee.infrastructure.presentation.factory;

import com.example.employee.infrastructure.presentation.dto.*;
import org.springframework.stereotype.Component;

import java.util.ArrayList;

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

A vantagem é que as Rules podem fazer diretamente:

```java
target.getAddress()
      .getLocation()
      .setCidade(...);
```

sem verificar se os objetos intermediários são `null`.

---

# 14. Response Mapper

O Mapper faz transformações específicas e reutilizáveis.

Ele não decide quais campos são permitidos.

```java
package com.example.employee.infrastructure.presentation.mapper;

import com.example.employee.infrastructure.presentation.dto.ContactItem;
import org.springframework.stereotype.Component;

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

---

# 15. Response Rules

As Rules associam:

```text
EmployeeField
       ↓
como preencher EmployeeResponse
```

```java
package com.example.employee.infrastructure.presentation.rules;

import com.example.employee.domain.enums.EmployeeField;
import com.example.employee.domain.model.Employee;
import com.example.employee.infrastructure.presentation.dto.ContactItem;
import com.example.employee.infrastructure.presentation.dto.EmployeeResponse;

import java.util.Map;
import java.util.function.BiConsumer;

public final class EmployeeResponseRules {

    private EmployeeResponseRules() {
    }

    public static Map<EmployeeField,
            BiConsumer<Employee, EmployeeResponse>> create() {

        return Map.of(

                EmployeeField.NAME,
                (source, target) ->
                        target.getPersonalData()
                                .setNome(
                                        source.getNome()),

                EmployeeField.CPF,
                (source, target) ->
                        target.getPersonalData()
                                .setCpf(
                                        source.getCpf()),

                EmployeeField.PHONE,
                (source, target) ->
                        target.getContacts()
                                .add(
                                        new ContactItem(
                                                "TELEFONE",
                                                source.getTelefone()
                                        )
                                ),

                EmployeeField.MOBILE,
                (source, target) ->
                        target.getContacts()
                                .add(
                                        new ContactItem(
                                                "CELULAR",
                                                source.getCelular()
                                        )
                                ),

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

---

# 16. Presenter

O Presenter recebe:

```java
Employee
List<String>
```

e retorna:

```java
EmployeeResponse
```

```java
package com.example.employee.infrastructure.presentation.presenter;

import com.example.employee.domain.enums.EmployeeField;
import com.example.employee.domain.model.Employee;
import com.example.employee.infrastructure.presentation.dto.EmployeeResponse;
import com.example.employee.infrastructure.presentation.factory.EmployeeResponseFactory;
import com.example.employee.infrastructure.presentation.rules.EmployeeResponseRules;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Optional;
import java.util.function.BiConsumer;

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

## O que acontece em `rule.accept(source, target)`?

O `target` criado pela Factory é o mesmo objeto que todas as Rules modificam.

Exemplo:

```text
Factory
   ↓
target vazio
   ↓
NAME Rule
   ↓
target.personalData.nome preenchido
   ↓
CPF Rule
   ↓
target.personalData.cpf preenchido
   ↓
PHONE Rule
   ↓
target.contacts preenchido
   ↓
return target
```

---

# 17. Controller

```java
package com.example.employee.infrastructure.input.controller;

import com.example.employee.application.input.GetEmployeeInput;
import com.example.employee.infrastructure.presentation.dto.EmployeeResponse;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/employees")
public class EmployeeController {

    private final GetEmployeeInput getEmployeeInput;

    public EmployeeController(
            GetEmployeeInput getEmployeeInput) {

        this.getEmployeeInput =
                getEmployeeInput;
    }

    @GetMapping("/{id}")
    public EmployeeResponse getById(
            @PathVariable Long id) {

        return getEmployeeInput.execute(id);
    }
}
```

---

# 18. Exemplo completo

Employee no domínio:

```text
nome      = João
cpf       = 12345678900
telefone  = 11999999999
cidade    = São Paulo
estado    = SP
```

Redis:

```json
[
  "nr_name",
  "nr_cpf",
  "nr_phone"
]
```

Conversão:

```text
nr_name
   ↓
EmployeeField.NAME

nr_cpf
   ↓
EmployeeField.CPF

nr_phone
   ↓
EmployeeField.PHONE
```

Rules executadas:

```text
NAME
  → personalData.nome

CPF
  → personalData.cpf

PHONE
  → contacts[]
```

Resultado:

```json
{
  "personalData": {
    "nome": "João",
    "cpf": "12345678900"
  },
  "contacts": [
    {
      "itemName": "TELEFONE",
      "itemId": "11999999999"
    }
  ]
}
```

---

# 19. Separação final

A solução ficou baseada em quatro responsabilidades na apresentação:

```text
Factory
   ↓
CRIA A ESTRUTURA

Rules
   ↓
DEFINE COMO PREENCHER

Mapper
   ↓
FAZ TRANSFORMAÇÕES ESPECÍFICAS

Presenter
   ↓
ORQUESTRA
```

Na persistência:

```text
EmployeeEntity
      ↓
EmployeePersistenceMapper
      ↓
Employee
```

No Redis:

```text
Redis
      ↓
List<String>
      ↓
EmployeeField
```

---

# 20. Princípio principal

A tabela, o domínio e o contrato podem possuir estruturas completamente diferentes:

```text
BANCO
EmployeeEntity
    ↓
tabela grande
```

```text
DOMÍNIO
Employee
    ↓
modelo de negócio
```

```text
RESPONSE
EmployeeResponse
    ↓
PersonalData
Address
Location
Contacts[]
Organization
Department
...
```

Não existe necessidade de forçar uma estrutura única para os três.

A arquitetura permite que cada modelo represente sua própria responsabilidade.

---

# 21. Resumo final

```text
                 ┌──────────────────┐
                 │    Controller    │
                 └────────┬─────────┘
                          │
                          ▼
                 ┌──────────────────┐
                 │     UseCase      │
                 └────────┬─────────┘
                          │
             ┌────────────┴────────────┐
             ▼                         ▼
       Repository                 Permission
             │                         │
             ▼                         ▼
          Entity                  Redis String
             │                         │
             ▼                         ▼
       Persistence                 EmployeeField
          Mapper                       │
             │                         │
             └────────────┬────────────┘
                          ▼
                 ┌──────────────────┐
                 │    Presenter     │
                 └────────┬─────────┘
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
          Factory        Rules        Mapper
             │            │            │
             └────────────┼────────────┘
                          ▼
                 ┌──────────────────┐
                 │ EmployeeResponse │
                 └──────────────────┘
```

A ideia central é manter o **domínio independente do formato da tabela e do formato do contrato**, enquanto a camada `presentation` concentra a montagem da resposta e o controle dos campos permitidos.
