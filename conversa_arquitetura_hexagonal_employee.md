# Conversa — Arquitetura Hexagonal / Java 21 / Employee

> Observação: este arquivo reproduz fielmente o conteúdo das mensagens que está disponível no contexto desta conversa. Algumas mensagens anteriores aparecem apenas como `Skipped message(s)` no contexto disponível; o conteúdo dessas mensagens não está acessível para reprodução literal e, portanto, não foi inventado ou reconstruído.

---

## Usuário

Estou desenvolvendo um micro serviço em Java, usando arquitetura hexagonal.

A arquitetura que planejo.
Infraestrutura
 - input
    Controller
-output
   Adaptar
   Repositório
   Entity
   Mapper
    ....
Domein
  Modelo
  Enuns
    ....
Application
Input
Output
UseCase

Faça todo o planejamento.

Contrato Employee
   Com apenas um endpoint
   GetById

Objeto de contrato, extremamente normalizado
Com diversos níveis de objetos.

A entidade principal é apenas uma tabela enorme.

O grande problema é esse uma tabela vai gerar uma entidade.

Mais o retorno do objeto é normalizados em diversas classes.

---

## Usuário

Então sempre no domínio vai ter a estrutura exata dos objetos do contrato

---

## Usuário

Se eu criar uma domein, com uma única classe, que represent mais ou menos o contrato, mais no contrato eu separo

---

## Usuário

Como ficaria a arquitetura final

---

## Usuário

O contrato, é envolvendo em uma Warper com as classes, eles ficam no target.

Nao queria criar apenas um mapper, só object-struct, para preencher o contrato no target

---

## Usuário

toTaget pode ser outro nome tipo toContract

---

## Usuário

Se tiver GetEmployeeAll e GetemployeePhito

---

## Usuário

Todos seram em um Warper diferente da classe de dominio

---

## Usuário

A arquitetura completa

---

## Usuário

Ok, como ficaria o mapper do contrato, precisa fazer os de para Soure e target,

---

## Usuário

Exemplo, existe uma Lista, chamada Contatos, Ela é genérica
ItemName , ItemId

No meu domínio, tenho Telefone , celular.

Isso é 2 item da lista

Como faria no mapper?

---

## Usuário

Ficou complexo

---

## Usuário

Se eu criar o Contatos no domínio?

---

## Usuário

Como ficaria
Contato para lista  no mapper

---

## Usuário

Vai ser um objeto Contato
Telefone,Celular...

Como ficaria com a Lista genérica no mappper

---

## Usuário

Se tiver várias transformações na interface o default, nao quebraria ?

---

## Usuário

Resolvi  ao criar o Contato no domínio, como deixa mais organizado

```java
@Mapper(componentModel = "spring")
public class ContatoMapper {

    public List<ContatoItem> toContract(Contato source) {

        return List.of(
            new ContatoItem("TELEFONE", source.getTelefone()),
            new ContatoItem("CELULAR", source.getCelular())
        );
    }
}
```

---

## Assistente

Sim. Se você decidiu que `Contato` é um conceito do **domínio**, dá para deixar esse mapper bem organizado e ainda manter a regra simples.

Eu faria assim:

```java
@Mapper(componentModel = "spring")
public class ContatoContractMapper {

    public List<ContatoItem> toContract(Contato source) {
        return List.of(
            createItem("TELEFONE", source.getTelefone()),
            createItem("CELULAR", source.getCelular())
        );
    }

    private ContatoItem createItem(String name, String value) {
        return new ContatoItem(name, value);
    }
}
```

Mas, pensando no seu caso, **eu nem usaria `@Mapper` do MapStruct nessa classe**, porque aqui não há mapeamento automático. É uma transformação manual.

Ficaria:

```java
@Component
public class ContatoContractMapper {

    public List<ContatoItem> toContract(Contato source) {
        return List.of(
            new ContatoItem("TELEFONE", source.getTelefone()),
            new ContatoItem("CELULAR", source.getCelular())
        );
    }
}
```

### E o mapper principal

```java
@Mapper(
    componentModel = "spring",
    uses = ContatoContractMapper.class
)
public interface EmployeeContractMapper {

    EmployeeWrapper toContract(Employee source);
}
```

A ideia é que **não tenha `default`, `expression` nem lógica dentro dele**.

---

## Usuário

Expression?

---

## Assistente

No MapStruct, `expression` é uma forma de você dizer:

> **"Para preencher este campo do target, execute este código Java."**

Por exemplo, você poderia fazer:

```java
@Mapper(componentModel = "spring")
public interface EmployeeContractMapper {

    @Mapping(
        target = "contatos",
        expression = "java(contatoMapper.toContract(source.getContato()))"
    )
    EmployeeWrapper toContract(
        Employee source
    );
}
```

Aqui:

```text
source.getContato()
        ↓
contatoMapper.toContract(...)
        ↓
List<ContatoItem>
        ↓
target.contatos
```

### Mas para o seu projeto, eu evitaria `expression`

Porque começa a deixar o mapper assim:

```java
@Mapping(
    target = "contatos",
    expression = "java(contatoMapper.toContract(source.getContato()))"
)
@Mapping(
    target = "enderecos",
    expression = "java(enderecoMapper.toContract(source.getEndereco()))"
)
@Mapping(
    target = "documentos",
    expression = "java(documentoMapper.toContract(source.getDocumentos()))"
)
```

Fica mais difícil de ler.

Eu prefiro usar o `uses` do MapStruct:

```java
@Mapper(
    componentModel = "spring",
    uses = {
        ContatoContractMapper.class,
        EnderecoContractMapper.class,
        DocumentoContractMapper.class
    }
)
public interface EmployeeContractMapper {

    EmployeeWrapper toContract(Employee source);
}
```

E:

```java
@Component
public class ContatoContractMapper {

    public List<ContatoItem> toContract(Contato source) {
        return List.of(
            new ContatoItem("TELEFONE", source.getTelefone()),
            new ContatoItem("CELULAR", source.getCelular())
        );
    }
}
```

O MapStruct consegue utilizar esse método para fazer a conversão.

### Resumindo

```text
@Mapping
    ↓
mapeamento simples/específico

uses
    ↓
delegar transformação para outro mapper

expression
    ↓
executar Java arbitrário
```

**No seu cenário, eu usaria `uses` e deixaria `expression` como último recurso.**

---

## Usuário

Como ficaria o mapper final com vários coversores

---

## Assistente

Sim. Se você terá **vários conversores especializados**, eu deixaria o mapper principal praticamente declarativo.

### Estrutura

```text
infrastructure
└── input
    └── contract
        └── mapper
            ├── EmployeeContractMapper
            ├── ContatoContractMapper
            ├── AddressContractMapper
            ├── DocumentContractMapper
            └── DepartmentContractMapper
```

### Mapper principal

```java
@Mapper(
    componentModel = "spring",
    uses = {
        ContatoContractMapper.class,
        AddressContractMapper.class,
        DocumentContractMapper.class,
        DepartmentContractMapper.class
    }
)
public interface EmployeeContractMapper {

    EmployeeWrapper toContract(Employee source);
}
```

A ideia é que **não tenha `default`, `expression` nem lógica dentro dele**.

---

### Conversor de Contato

```java
@Component
public class ContatoContractMapper {

    public List<ContatoItem> toContract(Contato source) {

        return List.of(
            new ContatoItem(
                "TELEFONE",
                source.getTelefone()
            ),
            new ContatoItem(
                "CELULAR",
                source.getCelular()
            )
        );
    }
}
```

### Conversor de endereço

```java
@Component
public class AddressContractMapper {

    public AddressContract toContract(Address source) {

        return new AddressContract(
            source.getStreet(),
            source.getCity(),
            source.getState()
        );
    }
}
```

### Conversor de documentos

```java
@Component
public class DocumentContractMapper {

    public List<DocumentItem> toContract(Document source) {

        return List.of(
            new DocumentItem("CPF", source.getCpf()),
            new DocumentItem("RG", source.getRg())
        );
    }
}
```

### Conversor de departamento

```java
@Component
public class DepartmentContractMapper {

    public DepartmentContract toContract(
            Department source) {

        return new DepartmentContract(
            source.getId(),
            source.getName()
        );
    }
}
```

---

## Usuário

Cada item vai ter um Soure e target ?

---

## Assistente

Sim — **conceitualmente, cada conversor trabalha com um `source` e um `target`**.

Mas você não precisa declarar literalmente os nomes `source` e `target`.

Por exemplo, para `Contato`:

```java
@Component
public class ContatoContractMapper {

    public List<ContatoItem> toContract(Contato source) {

        return List.of(
            new ContatoItem("TELEFONE", source.getTelefone()),
            new ContatoItem("CELULAR", source.getCelular())
        );
    }
}
```

Aqui:

```text
Source
Contato
 ├── telefone
 └── celular

        ↓

Target
List<ContatoItem>
 ├── TELEFONE
 └── CELULAR
```

Para endereço:

```java
@Component
public class AddressContractMapper {

    public AddressContract toContract(Address source) {

        return new AddressContract(
            source.getStreet(),
            source.getCity(),
            source.getState()
        );
    }
}
```

Aqui:

```text
Source              Target
Address       →     AddressContract
```

E no mapper principal:

```java
@Mapper(
    componentModel = "spring",
    uses = {
        ContatoContractMapper.class,
        AddressContractMapper.class
    }
)
public interface EmployeeContractMapper {

    EmployeeWrapper toContract(Employee source);
}
```

Portanto, você pode pensar sempre neste padrão:

```text
┌─────────────────────┐
│       SOURCE        │
│   objeto do Domain  │
└──────────┬──────────┘
           │
           │ ContractMapper
           ▼
┌─────────────────────┐
│       TARGET        │
│ objeto do contrato  │
└─────────────────────┘
```

A única diferença é que **um Source pode gerar um Target diferente**, inclusive uma lista:

```text
Contato
   ↓
List<ContatoItem>
```

ou:

```text
Address
   ↓
AddressContract
```

ou:

```text
Document
   ↓
List<DocumentItem>
```

**Essa padronização deixa os conversores muito fáceis de entender.**

---

## Usuário

Falo do mapper principal

---

## Assistente

Ah, entendi. Você está perguntando se **cada item do `EmployeeContractMapper` precisa ter seu próprio `source` e `target`**.

**Não.** No mapper principal existe **um único Source e um único Target**.

```java
@Mapper(
    componentModel = "spring",
    uses = {
        ContatoContractMapper.class,
        AddressContractMapper.class,
        DocumentContractMapper.class
    }
)
public interface EmployeeContractMapper {

    EmployeeWrapper toContract(Employee source);
}
```

Aqui:

```text
Source
Employee
   │
   │
   ▼
EmployeeContractMapper
   │
   │
   ▼
Target
EmployeeWrapper
```

O `Employee` contém os objetos:

```text
Employee
├── Contato
├── Address
├── Document
└── Department
```

E o `EmployeeWrapper` contém os objetos correspondentes:

```text
EmployeeWrapper
├── contatos
├── address
├── documents
└── department
```

O `uses` diz ao MapStruct:

> "Quando você encontrar um campo que precisa de uma transformação especial, pode utilizar esses conversores."

Então:

```text
Employee ──────────────────────► EmployeeWrapper
   │                                   │
   ├── Contato ──► ContatoMapper ──► contatos
   │
   ├── Address ──► AddressMapper ──► address
   │
   ├── Document ─► DocumentMapper ─► documents
   │
   └── Department ► DepartmentMapper ► department
```

### Portanto, o padrão fica

**Mapper principal:**

```java
EmployeeWrapper toContract(Employee source);
```

**Conversores especializados:**

```java
List<ContatoItem> toContract(Contato source);
AddressContract toContract(Address source);
List<DocumentItem> toContract(Document source);
DepartmentContract toContract(Department source);
```

Ou seja:

**1 Source/Target no mapper principal.**

**Vários Source/Target nos conversores especializados.**

---

## Usuário

Crie o mappper
Com vários souce e target,

Os nomes são iguais mais estam em objetos diferentes

Exempko 
Model o nome

Contrato
Objeto.objeto.nome

---

## Assistente

Sim. Nesse caso o `EmployeeContractMapper` pode ter **vários `@Mapping`**, cada um indicando explicitamente:

```text
source = campo no Domain
target = caminho dentro do Contract
```

### Exemplo

Domínio:

```java
public class Employee {
    private String nome;
    private String cpf;
    private String cidade;
}
```

Contrato:

```java
public class EmployeeWrapper {
    private PersonalData personalData;
    private Address address;
}
```

```java
public class PersonalData {
    private String nome;
    private Documentos documentos;
}
```

```java
public class Documentos {
    private String cpf;
}
```

```java
public class Address {
    private Location location;
}
```

```java
public class Location {
    private String cidade;
}
```

Então o mapper principal fica:

```java
@Mapper(
    componentModel = "spring",
    uses = {
        ContatoContractMapper.class
    }
)
public interface EmployeeContractMapper {

    @Mapping(
        source = "nome",
        target = "personalData.nome"
    )
    @Mapping(
        source = "cpf",
        target = "personalData.documentos.cpf"
    )
    @Mapping(
        source = "cidade",
        target = "address.location.cidade"
    )
    EmployeeWrapper toContract(Employee source);
}
```

A ideia é exatamente:

```text
SOURCE                         TARGET

Employee.nome          →       EmployeeWrapper.personalData.nome

Employee.cpf           →       EmployeeWrapper.personalData.documentos.cpf

Employee.cidade        →       EmployeeWrapper.address.location.cidade
```

### E com vários objetos

Você pode ter:

```java
@Mapper(
    componentModel = "spring",
    uses = {
        ContatoContractMapper.class,
        DepartmentContractMapper.class
    }
)
public interface EmployeeContractMapper {

    @Mapping(source = "nome",
             target = "personalData.nome")

    @Mapping(source = "cpf",
             target = "personalData.documentos.cpf")

    @Mapping(source = "email",
             target = "personalData.contact.email")

    @Mapping(source = "telefone",
             target = "personalData.contact.phone")

    @Mapping(source = "cidade",
             target = "address.location.cidade")

    @Mapping(source = "estado",
             target = "address.location.estado")

    @Mapping(source = "departmentId",
             target = "organization.department.id")

    @Mapping(source = "departmentName",
             target = "organization.department.name")

    EmployeeWrapper toContract(Employee source);
}
```

Isso é justamente o que você descreveu:

```text
Domain

Employee
├── nome
├── cpf
├── email
├── telefone
├── cidade
├── estado
├── departmentId
└── departmentName


             ↓


Contract

EmployeeWrapper
├── personalData
│   ├── nome
│   ├── documentos
│   │   └── cpf
│   └── contact
│       ├── email
│       └── phone
│
├── address
│   └── location
│       ├── cidade
│       └── estado
│
└── organization
    └── department
        ├── id
        └── name
```

E o mais interessante: **se os nomes forem iguais, você só precisa informar o caminho do `target`**:

```java
@Mapping(source = "nome", target = "personalData.nome")
```

O `source` é `Employee.nome`, enquanto o `target` é `EmployeeWrapper.personalData.nome`.

Para os casos em que existe uma transformação estrutural, como `Contato → List<ContatoItem>`, aí entra o `uses = ContatoContractMapper.class`.

---

## Usuário

Bom ok, em relação a esse ponto, apresente a arquitetura final

---

## Assistente

Perfeito. Consolidando tudo que definimos, eu usaria esta como **arquitetura final** para o seu microserviço `Employee`.

A ideia central é:

- **Domain**: modelo simples e independente do contrato.
- **Application**: casos de uso + portas.
- **Infrastructure Input**: Controller + contratos HTTP.
- **Infrastructure Output**: persistência.
- **MapStruct** para os mapeamentos normais.
- **Conversores especializados** apenas quando uma transformação não for um simples `source → target`.

### Estrutura final

```text
employee-service
│
├── domain
│   └── model
│       ├── Employee.java
│       ├── Contato.java
│       └── ...
│
├── application
│   │
│   ├── input
│   │   └── EmployeeUseCase.java
│   │
│   ├── output
│   │   └── EmployeeOutput.java
│   │
│   └── service
│       └── EmployeeService.java
│
└── infrastructure
    │
    ├── input
    │   │
    │   ├── EmployeeController.java
    │   │
    │   └── contract
    │       │
    │       ├── get-by-id
    │       │   ├── EmployeeWrapper.java
    │       │   ├── PersonalData.java
    │       │   ├── Documents.java
    │       │   ├── Address.java
    │       │   ├── Contact.java
    │       │   ├── Organization.java
    │       │   ├── Department.java
    │       │   └── ...
    │       │
    │       ├── get-all
    │       │   ├── EmployeeAllWrapper.java
    │       │   └── EmployeeItem.java
    │       │
    │       └── get-photo
    │           ├── EmployeePhotoWrapper.java
    │           └── Photo.java
    │
    └── output
        │
        └── persistence
            ├── EmployeeEntity.java
            ├── EmployeeRepository.java
            ├── EmployeePersistenceAdapter.java
            └── mapper
                └── EmployeeEntityMapper.java
```

E para os mappers do contrato:

```text
infrastructure
└── input
    └── contract
        └── mapper
            ├── EmployeeContractMapper.java
            ├── ContatoContractMapper.java
            ├── AddressContractMapper.java
            ├── DocumentContractMapper.java
            └── DepartmentContractMapper.java
```

### Domain

O domínio não conhece JSON, Controller, JPA ou contrato.

```java
public class Employee {

    private Long id;
    private String nome;
    private String cpf;
    private String email;

    private String cidade;
    private String estado;

    private Contato contato;

    private Long departmentId;
    private String departmentName;
}
```

E `Contato` é um conceito do domínio:

```java
public class Contato {

    private String telefone;
    private String celular;
}
```

Assim:

```text
Employee
│
├── nome
├── cpf
├── email
├── cidade
├── estado
├── Contato
│   ├── telefone
│   └── celular
│
├── departmentId
└── departmentName
```

### Application Input

A porta de entrada:

```java
public interface EmployeeUseCase {

    Employee getById(Long id);

    List<Employee> getAll();

    Employee getPhoto(Long id);
}
```

### Application Output

A porta de saída:

```java
public interface EmployeeOutput {

    Optional<Employee> findById(Long id);

    List<Employee> findAll();

    Optional<Employee> findPhoto(Long id);
}
```

A regra é:

```text
Application
     │
     │ conhece
     ▼
EmployeeOutput

Infrastructure
     │
     │ implementa
     ▼
EmployeePersistenceAdapter
```

### Service

```java
@Service
public class EmployeeService implements EmployeeUseCase {

    private final EmployeeOutput employeeOutput;

    public EmployeeService(EmployeeOutput employeeOutput) {
        this.employeeOutput = employeeOutput;
    }

    @Override
    public Employee getById(Long id) {
        return employeeOutput.findById(id)
                .orElseThrow(() ->
                    new EmployeeNotFoundException(id));
    }

    @Override
    public List<Employee> getAll() {
        return employeeOutput.findAll();
    }

    @Override
    public Employee getPhoto(Long id) {
        return employeeOutput.findPhoto(id)
                .orElseThrow(() ->
                    new EmployeeNotFoundException(id));
    }
}
```

### Infrastructure Output

A tabela pode ser enorme:

```java
@Entity
@Table(name = "EMPLOYEE")
public class EmployeeEntity {

    @Id
    private Long id;

    private String nome;
    private String cpf;
    private String email;

    private String cidade;
    private String estado;

    private String telefone;
    private String celular;

    private Long departmentId;
    private String departmentName;

    // muitos outros campos...
}
```

Isso **não precisa ser igual ao Domain**.

### Repository

```java
public interface EmployeeRepository
        extends JpaRepository<EmployeeEntity, Long> {
}
```

### Entity Mapper

Aqui usamos MapStruct:

```java
@Mapper(componentModel = "spring")
public interface EmployeeEntityMapper {

    Employee toDomain(EmployeeEntity source);

    List<Employee> toDomain(
        List<EmployeeEntity> source
    );
}
```

Fluxo:

```text
EmployeeEntity
      │
      │ toDomain()
      ▼
Employee
```

### Persistence Adapter

```java
@Component
public class EmployeePersistenceAdapter
        implements EmployeeOutput {

    private final EmployeeRepository repository;
    private final EmployeeEntityMapper mapper;

    public EmployeePersistenceAdapter(
            EmployeeRepository repository,
            EmployeeEntityMapper mapper) {

        this.repository = repository;
        this.mapper = mapper;
    }

    @Override
    public Optional<Employee> findById(Long id) {
        return repository.findById(id)
                .map(mapper::toDomain);
    }

    @Override
    public List<Employee> findAll() {
        return mapper.toDomain(repository.findAll());
    }

    @Override
    public Optional<Employee> findPhoto(Long id) {
        return repository.findById(id)
                .map(mapper::toDomain);
    }
}
```

### Contrato

Agora vem a parte mais importante.

O contrato pode ter uma estrutura completamente diferente.

```text
EmployeeWrapper
│
├── personalData
│   ├── nome
│   └── documents
│       └── cpf
│
├── address
│   └── location
│       ├── cidade
│       └── estado
│
├── contacts
│
└── organization
    └── department
        ├── id
        └── name
```

Cada endpoint pode ter seu próprio Wrapper:

```text
GetById
    EmployeeWrapper

GetAll
    EmployeeAllWrapper

GetPhoto
    EmployeePhotoWrapper
```

### Mapper principal do contrato

Aqui está exatamente o ponto que você definiu.

```java
@Mapper(
    componentModel = "spring",
    uses = {
        ContatoContractMapper.class,
        DocumentContractMapper.class,
        DepartmentContractMapper.class
    }
)
public interface EmployeeContractMapper {

    @Mapping(
        source = "nome",
        target = "personalData.nome"
    )
    @Mapping(
        source = "cpf",
        target = "personalData.documents.cpf"
    )
    @Mapping(
        source = "cidade",
        target = "address.location.cidade"
    )
    @Mapping(
        source = "estado",
        target = "address.location.estado"
    )
    @Mapping(
        source = "departmentId",
        target = "organization.department.id"
    )
    @Mapping(
        source = "departmentName",
        target = "organization.department.name"
    )
    EmployeeWrapper toContract(Employee source);
}
```

Observe:

```text
SOURCE                          TARGET

Employee.nome           →       personalData.nome

Employee.cpf            →       personalData.documents.cpf

Employee.cidade         →       address.location.cidade

Employee.estado         →       address.location.estado

Employee.departmentId   →       organization.department.id

Employee.departmentName →       organization.department.name
```

O `EmployeeContractMapper` tem:

**1 Source:**

```java
Employee
```

**1 Target:**

```java
EmployeeWrapper
```

E vários `@Mapping` definem os caminhos.

### Conversor especializado

Quando a transformação não é simplesmente:

```text
campo → campo
```

usamos um conversor.

Seu exemplo de `Contato`:

```text
Contato
├── telefone
└── celular

       ↓

List<ContatoItem>
├── TELEFONE
└── CELULAR
```

```java
@Component
public class ContatoContractMapper {

    public List<ContatoItem> toContract(Contato source) {

        return List.of(
            new ContatoItem(
                "TELEFONE",
                source.getTelefone()
            ),
            new ContatoItem(
                "CELULAR",
                source.getCelular()
            )
        );
    }
}
```

### Controller

```java
@RestController
@RequestMapping("/employees")
public class EmployeeController {

    private final EmployeeUseCase useCase;
    private final EmployeeContractMapper mapper;

    public EmployeeController(
            EmployeeUseCase useCase,
            EmployeeContractMapper mapper) {

        this.useCase = useCase;
        this.mapper = mapper;
    }

    @GetMapping("/{id}")
    public EmployeeWrapper getById(
            @PathVariable Long id) {

        Employee employee = useCase.getById(id);

        return mapper.toContract(employee);
    }
}
```

### Fluxo completo

```text
                         HTTP
                          │
                          ▼
                ┌──────────────────┐
                │ EmployeeController│
                └────────┬─────────┘
                         │
                         ▼
                ┌──────────────────┐
                │  EmployeeUseCase │
                └────────┬─────────┘
                         │
                         ▼
                ┌──────────────────┐
                │ EmployeeService  │
                └────────┬─────────┘
                         │
                         ▼
                ┌──────────────────┐
                │ EmployeeOutput   │
                └────────┬─────────┘
                         │
                         ▼
                ┌──────────────────┐
                │PersistenceAdapter│
                └────────┬─────────┘
                         │
                         ▼
                ┌──────────────────┐
                │ EmployeeRepository│
                └────────┬─────────┘
                         │
                         ▼
                       BANCO
                         │
                         ▼
                  EmployeeEntity
                         │
                         │ MapStruct
                         ▼
                      Employee
                         │
                         │ MapStruct
                         ▼
                 EmployeeWrapper
                         │
                         ▼
                        JSON
```

### A separação final

```text
┌─────────────────────────────────────────────┐
│ DOMAIN                                      │
│                                             │
│ Employee                                    │
│ Contato                                     │
│                                             │
│ Não conhece HTTP/JPA/JSON                   │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│ APPLICATION                                 │
│                                             │
│ Input                                       │
│ Output                                      │
│ Service                                     │
│                                             │
│ Orquestra o caso de uso                     │
└───────────────┬─────────────────┬───────────┘
                │                 │
                ▼                 ▼
       ┌───────────────┐   ┌─────────────────┐
       │ INPUT         │   │ OUTPUT          │
       │               │   │                 │
       │ Controller    │   │ Persistence     │
       │ Contract      │   │ Entity          │
       │ ContractMapper│   │ Repository      │
       └───────────────┘   └─────────────────┘
```

### Regra de ouro

```text
Banco
  ↓
Entity
  ↓
EntityMapper
  ↓
Domain
  ↓
ContractMapper
  ↓
Contract/Wrapper
  ↓
JSON
```

---

## Usuário

Tenho uma regra de negocio apenas.
Me ajude a encaixar na solução.

A regra é vou fazer mais uma consulta no redis.
Passando a matrícula, e nela vai retornar o Campos que posso preencher no contrato.

O retorno do Redis vai ser Campos da tabela.
O que acha melhor fazer ?

---

## Assistente

Sim. Eu colocaria essa regra **na Application/Service**, não no mapper e não no Controller.

A ideia é separar duas coisas:

1. **Buscar o Employee no banco**
2. **Consultar o Redis para descobrir quais campos podem ser expostos/preenchidos**

### Fluxo

```text
Controller
    │
    ▼
EmployeeUseCase
    │
    ▼
EmployeeService
    │
    ├──────────────► EmployeeOutput
    │                    │
    │                    ▼
    │                 Banco
    │                    │
    │                    ▼
    │                 Employee
    │
    └──────────────► EmployeeFieldsOutput
                         │
                         ▼
                       Redis
                         │
                         ▼
                  Campos permitidos
```

Eu criaria uma **segunda porta de saída** para o Redis:

```java
public interface EmployeeFieldsOutput {

    Set<String> findAllowedFields(String matricula);
}
```

E o adapter:

```java
@Component
public class EmployeeFieldsRedisAdapter
        implements EmployeeFieldsOutput {

    private final RedisTemplate<String, Object> redis;

    @Override
    public Set<String> findAllowedFields(String matricula) {

        // consulta Redis
        // retorna os campos permitidos

    }
}
```

## E o Service

```java
@Service
public class EmployeeService implements EmployeeUseCase {

    private final EmployeeOutput employeeOutput;
    private final EmployeeFieldsOutput fieldsOutput;

    @Override
    public Employee getById(Long id) {

        Employee employee = employeeOutput
                .findById(id)
                .orElseThrow();

        Set<String> allowedFields =
                fieldsOutput.findAllowedFields(
                        employee.getMatricula());

        employee.applyAllowedFields(allowedFields);

        return employee;
    }
}
```

Mas aqui eu faria uma pequena mudança.

### Eu não colocaria `applyAllowedFields()` dentro do Employee

Porque o Redis está dizendo **o que pode ser preenchido no contrato**, e isso é uma regra de apresentação/integração, não necessariamente uma regra intrínseca do `Employee`.

Então prefiro:

```text
Employee
      +
AllowedFields
      │
      ▼
ContractMapper
      │
      ▼
EmployeeWrapper
```

Por exemplo:

```java
public EmployeeWrapper toContract(
        Employee employee,
        Set<String> allowedFields) {

    // monta contrato considerando os campos permitidos
}
```

### Porém existe uma questão importante

Você disse:

> "O retorno do Redis vai ser Campos da tabela."

Exemplo:

```text
Redis

matricula = 12345

[
   "NOME",
   "CPF",
   "TELEFONE",
   "DEPARTMENT_NAME"
]
```

Eu **não usaria esses nomes de coluna diretamente espalhados pelo código**.

Criaria um objeto específico:

```java
public record EmployeeFieldPermission(
        Set<String> fields
) {}
```

E a porta:

```java
public interface EmployeeFieldsOutput {

    EmployeeFieldPermission findByMatricula(
            String matricula);
}
```

Assim o Redis fica isolado.

---

## E o desenho

```text
                    CONTROLLER
                        │
                        ▼
                 EmployeeUseCase
                        │
                        ▼
                 EmployeeService
                   │          │
                   │          │
                   ▼          ▼
            EmployeeOutput   EmployeeFieldsOutput
                   │          │
                   ▼          ▼
                 Banco      Redis
                   │          │
                   ▼          ▼
                Employee   AllowedFields
                   │          │
                   └────┬─────┘
                        ▼
               EmployeeContractMapper
                        │
                        ▼
                 EmployeeWrapper
```

O ponto importante é: **não deixe `"NM_EMPLOYEE"`, `"NR_CPF"` etc. vazarem para o Domain.** Esses nomes pertencem à infraestrutura; a aplicação deve trabalhar com conceitos como `NAME`, `CPF`, `PHONE` ou uma estrutura de permissão equivalente.

---

## Usuário

Se eu estou recebendo uma lista de colunas ?

Como seria o service da permissão?
Já que no objeto de negocio,  ao tem haver com a coluna no banco

---

## Assistente

Exatamente. **Se o Redis retorna nomes de colunas da tabela, eu não colocaria isso no `Employee` do domínio.** Coluna de banco é detalhe de infraestrutura.

Eu separaria em 3 conceitos:

```text
Redis
  ↓
colunas da tabela
  ↓
Adapter traduz para uma permissão técnica
  ↓
Application usa essa permissão
  ↓
Contract Mapper decide o que vai para o contrato
```

### 1. Redis retorna as colunas

Por exemplo:

```text
matricula = 123

[
    "NM_EMPLOYEE",
    "NR_CPF",
    "NR_PHONE",
    "CD_DEPARTMENT"
]
```

Isso é **Infrastructure**, porque são nomes físicos da tabela.

### 2. Crie uma porta de saída específica

```java
public interface EmployeeFieldPermissionOutput {

    Set<String> findAllowedFields(String matricula);
}
```

O adapter Redis conhece as colunas:

```java
@Component
public class EmployeeFieldPermissionRedisAdapter
        implements EmployeeFieldPermissionOutput {

    @Override
    public Set<String> findAllowedFields(String matricula) {

        // Redis
        return ...;
    }
}
```

Até aqui, o domínio não sabe que existem colunas.

### 3. E o Service?

Eu **não criaria um `PermissionService` no Domain**.

Colocaria essa coordenação no `EmployeeService`, porque faz parte do caso de uso:

```java
@Service
public class EmployeeService implements EmployeeUseCase {

    private final EmployeeOutput employeeOutput;
    private final EmployeeFieldPermissionOutput permissionOutput;

    @Override
    public Employee getById(Long id) {

        Employee employee = employeeOutput
                .findById(id)
                .orElseThrow();

        Set<String> allowedFields =
                permissionOutput.findAllowedFields(
                        employee.getMatricula()
                );

        // aqui acontece a aplicação da regra

        return employee;
    }
}
```

**Mas aqui surge o problema que você percebeu:** `Employee` não deveria receber `"NM_EMPLOYEE"` ou `"NR_CPF"`.

Então eu mudaria o desenho.

### 4. Não passe coluna para o Domain

Faça a conversão:

```text
Redis
"NM_EMPLOYEE"
"NR_CPF"
"NR_PHONE"

        ↓

Application

NOME
CPF
TELEFONE

        ↓

Contract
```

Por exemplo:

```java
public enum EmployeeField {

    NAME,
    CPF,
    PHONE,
    DEPARTMENT
}
```

E a porta poderia retornar:

```java
public interface EmployeeFieldPermissionOutput {

    Set<EmployeeField> findAllowedFields(String matricula);
}
```

### 5. Minha solução preferida para seu caso

Criaria um `EmployeeFieldPermissionService` **na Application**:

```java
@Service
public class EmployeeFieldPermissionService {

    private final EmployeeFieldPermissionOutput output;

    public EmployeeFieldPermissionService(
            EmployeeFieldPermissionOutput output) {

        this.output = output;
    }

    public EmployeeFieldPermission getPermission(
            String matricula) {

        Set<String> columns =
                output.findAllowedFields(matricula);

        return EmployeeFieldPermission.from(columns);
    }
}
```

E:

```java
public record EmployeeFieldPermission(
        boolean name,
        boolean cpf,
        boolean phone,
        boolean department
) {

    public static EmployeeFieldPermission from(
            Set<String> columns) {

        return new EmployeeFieldPermission(
            columns.contains("NM_EMPLOYEE"),
            columns.contains("NR_CPF"),
            columns.contains("NR_PHONE"),
            columns.contains("CD_DEPARTMENT")
        );
    }
}
```

Agora você tem uma separação muito boa:

```text
              REDIS
                │
                │ "NM_EMPLOYEE"
                │ "NR_CPF"
                │ "NR_PHONE"
                ▼
       RedisPermissionAdapter
                │
                ▼
   EmployeeFieldPermissionService
                │
                │ traduz
                ▼
     EmployeeFieldPermission
                │
                │
                ▼
          Application
```

### 6. Mas eu faria uma alteração importante

Como seu objetivo final é **decidir quais campos aparecem no contrato**, eu não colocaria `EmployeeFieldPermission` no Domain.

Colocaria:

```text
application
├── input
├── output
├── service
│   ├── EmployeeService
│   └── EmployeeFieldPermissionService
└── model
    └── EmployeeFieldPermission
```

Então:

```text
Domain
Employee
Contato
...

Application
EmployeeService
EmployeeFieldPermissionService
EmployeeFieldPermission

Infrastructure
RedisAdapter
EmployeeEntity
Repository
Contract
Mappers
```

### 7. Fluxo final

```text
                       Controller
                           │
                           ▼
                    EmployeeService
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
       EmployeeOutput       EmployeeFieldPermissionService
              │                         │
              ▼                         ▼
            Banco             EmployeeFieldPermissionOutput
              │                         │
              ▼                         ▼
           Employee                   Redis
              │                         │
              └──────────┬──────────────┘
                         ▼
                  Employee + Permission
                         │
                         ▼
                EmployeeContractMapper
                         │
                         ▼
                  EmployeeWrapper
```

E conceitualmente:

```text
Banco:
    Employee

Redis:
    "quais campos estão autorizados?"

Application:
    transforma coluna técnica → significado da aplicação

Contract:
    transforma Employee + permissão → JSON
```

**Essa é a separação que eu considero mais limpa para seu cenário.**

O ponto principal é: **não deixe `"NM_EMPLOYEE"`, `"NR_CPF"` etc. vazarem para o Domain.** Esses nomes pertencem à infraestrutura; a aplicação deve trabalhar com conceitos como `NAME`, `CPF`, `PHONE` ou uma estrutura de permissão equivalente.

---

## Usuário

Como serie essa classe para verificar os Campos permitido

---

## Assistente

Eu faria uma classe **bem simples**, responsável apenas por responder:

> "Este campo está permitido?"

Por exemplo:

```java
public class EmployeeFieldPermission {

    private final Set<String> allowedFields;

    public EmployeeFieldPermission(Set<String> allowedFields) {
        this.allowedFields = allowedFields;
    }

    public boolean isAllowed(String field) {
        return allowedFields.contains(field);
    }
}
```

Mas, no seu caso, eu prefiro **não usar `String` espalhado pelo código**. Use um `enum`:

```java
public enum EmployeeField {

    NAME,
    CPF,
    PHONE,
    DEPARTMENT
}
```

E a classe:

```java
public class EmployeeFieldPermission {

    private final Set<EmployeeField> allowedFields;

    public EmployeeFieldPermission(
            Set<EmployeeField> allowedFields) {

        this.allowedFields = allowedFields;
    }

    public boolean isAllowed(EmployeeField field) {
        return allowedFields.contains(field);
    }
}
```

Então o uso fica muito limpo:

```java
if (permission.isAllowed(EmployeeField.CPF)) {
    // pode colocar CPF no contrato
}
```

Ou:

```java
if (permission.isAllowed(EmployeeField.PHONE)) {
    // pode colocar telefone
}
```

### Mas existe um detalhe importante no seu cenário

O Redis retorna **nome de coluna**:

```text
NM_EMPLOYEE
NR_CPF
NR_PHONE
```

Então eu faria a tradução **uma única vez**, no adapter:

```java
@Component
public class EmployeeFieldPermissionRedisAdapter
        implements EmployeeFieldPermissionOutput {

    @Override
    public Set<EmployeeField> findAllowedFields(
            String matricula) {

        Set<String> columns = findColumnsFromRedis(matricula);

        return columns.stream()
                .map(this::toField)
                .filter(Objects::nonNull)
                .collect(Collectors.toSet());
    }

    private EmployeeField toField(String column) {

        return switch (column) {
            case "NM_EMPLOYEE" -> EmployeeField.NAME;
            case "NR_CPF" -> EmployeeField.CPF;
            case "NR_PHONE" -> EmployeeField.PHONE;
            case "CD_DEPARTMENT" -> EmployeeField.DEPARTMENT;
            default -> null;
        };
    }
}
```

Assim, depois que saiu do adapter Redis, **acabaram os nomes de coluna**.

O restante da aplicação trabalha somente com:

```text
EmployeeField.NAME
EmployeeField.CPF
EmployeeField.PHONE
EmployeeField.DEPARTMENT
```

E a classe de permissão fica extremamente simples:

```java
public class EmployeeFieldPermission {

    private final Set<EmployeeField> allowedFields;

    public EmployeeFieldPermission(
            Set<EmployeeField> allowedFields) {
        this.allowedFields = allowedFields;
    }

    public boolean isAllowed(EmployeeField field) {
        return allowedFields.contains(field);
    }
}
```

**Eu usaria exatamente essa abordagem** se a lista do Redis for relativamente estável.

---

## Usuário

Vou usar mapper manual, isso vai fica complexo, pois todo os set, vai ter um if

---

## Assistente

Sim. Se você fizer um mapper manual assim:

```java
target.setCpf(
    permission.isAllowed(CPF)
        ? source.getCpf()
        : null
);
```

para **dezenas de campos**, vai ficar muito verboso.

Para o seu cenário, **a permissão não deve ficar espalhada pelo mapper**. Crie uma classe que faça a filtragem do `Employee` antes do mapeamento.

### Ficaria assim

```text
Banco
  ↓
Employee
  ↓
PermissionService
  ↓
Employee filtrado
  ↓
ContractMapper
  ↓
Wrapper
```

Por exemplo:

```java
public class EmployeeFieldFilter {

    public Employee filter(
            Employee employee,
            Set<EmployeeField> allowedFields) {

        if (!allowedFields.contains(EmployeeField.CPF)) {
            employee.setCpf(null);
        }

        if (!allowedFields.contains(EmployeeField.PHONE)) {
            employee.setTelefone(null);
        }

        if (!allowedFields.contains(EmployeeField.NAME)) {
            employee.setNome(null);
        }

        return employee;
    }
}
```

Mas ainda temos os `if`.

Então, **melhor ainda**, se a regra for simplesmente "campo permitido ou não", eu evitaria modificar o `Employee`.

### Minha preferência no seu caso

Faça o mapper manual e centralize a decisão em um método:

```java
public class EmployeeContractMapper {

    public EmployeeWrapper toContract(
            Employee source,
            EmployeeFieldPermission permission) {

        EmployeeWrapper target = new EmployeeWrapper();

        setBasicData(target, source, permission);
        setAddress(target, source, permission);
        setContact(target, source, permission);
        setDepartment(target, source, permission);

        return target;
    }
}
```

E cada bloco trata um grupo:

```java
private void setBasicData(
        EmployeeWrapper target,
        Employee source,
        EmployeeFieldPermission permission) {

    if (permission.isAllowed(EmployeeField.NAME)) {
        target.getPersonalData().setNome(source.getNome());
    }

    if (permission.isAllowed(EmployeeField.CPF)) {
        target.getPersonalData().setCpf(source.getCpf());
    }
}
```

Ainda existem `if`, mas ficam **organizados por responsabilidade**, em vez de 50 `if` em um método gigante.

---

### Existe uma alternativa ainda melhor

Se a permissão vem como **lista de colunas**, você pode criar um mapa de funções:

```java
private final Map<EmployeeField, BiConsumer<Employee, EmployeeWrapper>>
    mappings = Map.of(
        EmployeeField.NAME,
        (e, t) -> t.getPersonalData().setNome(e.getNome()),

        EmployeeField.CPF,
        (e, t) -> t.getPersonalData().setCpf(e.getCpf()),

        EmployeeField.PHONE,
        (e, t) -> t.getContacts().setTelefone(e.getTelefone())
    );
```

Então:

```java
public EmployeeWrapper toContract(
        Employee source,
        Set<EmployeeField> allowedFields) {

    EmployeeWrapper target = new EmployeeWrapper();

    allowedFields.forEach(field -> {
        BiConsumer<Employee, EmployeeWrapper> mapping =
                mappings.get(field);

        if (mapping != null) {
            mapping.accept(source, target);
        }
    });

    return target;
}
```

Isso elimina o padrão:

```text
if (...)
    set(...);

if (...)
    set(...);

if (...)
    set(...);
```

### Porém, para o seu projeto, eu escolheria uma solução mais simples

Como você já tem **um contrato extremamente normalizado**, eu não tentaria criar um framework genérico de permissões.

Eu faria:

```text
EmployeeService
       │
       ├── busca Employee
       │
       └── busca campos permitidos
                 │
                 ▼
        EmployeeFieldPermission
                 │
                 ▼
       EmployeeContractMapper
                 │
                 ▼
          EmployeeWrapper
```

E o mapper manual teria métodos por **bloco do contrato**:

```text
toContract()
 ├── mapPersonalData()
 ├── mapAddress()
 ├── mapContacts()
 ├── mapDocuments()
 └── mapOrganization()
```

Isso mantém a complexidade sob controle e, principalmente, **não cria uma abstração genérica demais para resolver um problema que é específico do seu contrato**.

---

## Usuário

```java
public EmployeeWrapper toContract(
        Employee source,
        Set<EmployeeField> allowedFields) {

    EmployeeWrapper target = new EmployeeWrapper();

    allowedFields.forEach(field -> {
        BiConsumer<Employee, EmployeeWrapper> mapping =
                mappings.get(field);

        if (mapping != null) {
            mapping.accept(source, target);
        }
    });

    return target;
}
```

Com essa solução, revisando, como ficaria as outras classes

---

## Assistente

Sim. Com essa solução, eu faria uma pequena reorganização para deixar o desenho consistente.

A ideia passa a ser:

```text
Redis → colunas permitidas
          ↓
   PermissionAdapter
          ↓
 Set<EmployeeField>
          ↓
     EmployeeService
          ↓
 Employee + campos permitidos
          ↓
 EmployeeContractMapper
          ↓
    EmployeeWrapper
```

### 1. Domain

O domínio continua sem saber de permissões ou colunas.

```text
domain
└── model
    ├── Employee.java
    └── Contato.java
```

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
    private String departmentName;

    private Contato contato;
}
```

---

### 2. Application — Use Case

A porta de entrada:

```java
public interface EmployeeUseCase {

    Employee getById(Long id);
}
```

Aqui tem uma mudança importante:

**o UseCase pode retornar o contrato?**

Eu prefiro **não**.

Para manter a Application independente da infraestrutura HTTP, faria:

```java
public interface EmployeeUseCase {

    Employee getById(Long id);
}
```

E o Controller faz o contrato.

---

### 3. Application — Outputs

Você passa a ter duas portas de saída:

```text
application
├── input
│   └── EmployeeUseCase
│
└── output
    ├── EmployeeOutput
    └── EmployeeFieldPermissionOutput
```

### EmployeeOutput

```java
public interface EmployeeOutput {

    Optional<Employee> findById(Long id);
}
```

### Permission Output

```java
public interface EmployeeFieldPermissionOutput {

    Set<EmployeeField> findAllowedFields(String matricula);
}
```

---

### 4. EmployeeField

Eu colocaria isso na **Application**, porque representa a necessidade da aplicação, não a tabela.

```java
public enum EmployeeField {

    NAME,
    CPF,
    PHONE,
    MOBILE,
    CITY,
    STATE,
    DEPARTMENT
}
```

---

### 5. Permission Service

Aqui você pode ter:

```java
public class EmployeeFieldPermission {

    private final Set<EmployeeField> allowedFields;

    public EmployeeFieldPermission(
            Set<EmployeeField> allowedFields) {

        this.allowedFields = allowedFields;
    }

    public boolean isAllowed(EmployeeField field) {
        return allowedFields.contains(field);
    }
}
```

Mas, com o mapper usando diretamente `Set<EmployeeField>`, **essa classe nem é obrigatória**.

Eu simplificaria e eliminaria.

O próprio:

```text
Set<EmployeeField>
```

já representa os campos permitidos.

---

### 6. EmployeeService

Agora o Service coordena as duas consultas:

```java
@Service
public class EmployeeService implements EmployeeUseCase {

    private final EmployeeOutput employeeOutput;
    private final EmployeeFieldPermissionOutput permissionOutput;

    public EmployeeService(
            EmployeeOutput employeeOutput,
            EmployeeFieldPermissionOutput permissionOutput) {

        this.employeeOutput = employeeOutput;
        this.permissionOutput = permissionOutput;
    }

    @Override
    public Employee getById(Long id) {

        Employee employee = employeeOutput
                .findById(id)
                .orElseThrow(() ->
                        new EmployeeNotFoundException(id));

        return employee;
    }
}
```

Aqui está uma decisão importante:

**O Service não precisa chamar o mapper.**

Ele continua entregando o domínio.

Mas precisamos levar a permissão até o Controller.

Então temos duas opções.

---

### 7. Eu criaria um objeto de resultado da Application

Essa é uma solução muito limpa:

```java
public record EmployeeResult(
        Employee employee,
        Set<EmployeeField> allowedFields
) {
}
```

Agora:

```java
public interface EmployeeUseCase {

    EmployeeResult getById(Long id);
}
```

E o Service:

```java
@Service
public class EmployeeService implements EmployeeUseCase {

    private final EmployeeOutput employeeOutput;
    private final EmployeeFieldPermissionOutput permissionOutput;

    public EmployeeService(
            EmployeeOutput employeeOutput,
            EmployeeFieldPermissionOutput permissionOutput) {

        this.employeeOutput = employeeOutput;
        this.permissionOutput = permissionOutput;
    }

    @Override
    public EmployeeResult getById(Long id) {

        Employee employee = employeeOutput
                .findById(id)
                .orElseThrow(() ->
                        new EmployeeNotFoundException(id));

        Set<EmployeeField> allowedFields =
                permissionOutput.findAllowedFields(
                        employee.getMatricula()
                );

        return new EmployeeResult(
                employee,
                allowedFields
        );
    }
}
```

Isso fica muito bom.

---

### 8. Controller

Agora o Controller faz a transformação HTTP:

```java
@RestController
@RequestMapping("/employees")
public class EmployeeController {

    private final EmployeeUseCase useCase;
    private final EmployeeContractMapper contractMapper;

    public EmployeeController(
            EmployeeUseCase useCase,
            EmployeeContractMapper contractMapper) {

        this.useCase = useCase;
        this.contractMapper = contractMapper;
    }

    @GetMapping("/{id}")
    public EmployeeWrapper getById(
            @PathVariable Long id) {

        EmployeeResult result =
                useCase.getById(id);

        return contractMapper.toContract(
                result.employee(),
                result.allowedFields()
        );
    }
}
```

Agora a responsabilidade fica perfeita:

```text
Controller
    │
    │ chama
    ▼
UseCase
    │
    ▼
EmployeeService
    │
    ├── Banco → Employee
    │
    └── Redis → Set<EmployeeField>
    │
    ▼
EmployeeResult
    │
    ▼
ContractMapper
    │
    ▼
EmployeeWrapper
```

---

### 9. Contract Mapper

Agora entra sua solução:

```java
@Component
public class EmployeeContractMapper {

    private final Map<EmployeeField,
            BiConsumer<Employee, EmployeeWrapper>> mappings;

    public EmployeeContractMapper() {

        mappings = Map.of(

            EmployeeField.NAME,
                (e, t) ->
                    t.getPersonalData()
                     .setNome(e.getNome()),

            EmployeeField.CPF,
                (e, t) ->
                    t.getPersonalData()
                     .setCpf(e.getCpf()),

            EmployeeField.PHONE,
                (e, t) ->
                    t.getContacts()
                     .add(new ContatoItem(
                         "TELEFONE",
                         e.getTelefone()
                     )),

            EmployeeField.MOBILE,
                (e, t) ->
                    t.getContacts()
                     .add(new ContatoItem(
                         "CELULAR",
                         e.getCelular()
                     )),

            EmployeeField.CITY,
                (e, t) ->
                    t.getAddress()
                     .getLocation()
                     .setCidade(e.getCidade()),

            EmployeeField.STATE,
                (e, t) ->
                    t.getAddress()
                     .getLocation()
                     .setEstado(e.getEstado()),

            EmployeeField.DEPARTMENT,
                (e, t) ->
                    t.getOrganization()
                     .getDepartment()
                     .setId(e.getDepartmentId())
        );
    }

    public EmployeeWrapper toContract(
            Employee source,
            Set<EmployeeField> allowedFields) {

        EmployeeWrapper target =
                new EmployeeWrapper();

        allowedFields.forEach(field -> {

            BiConsumer<Employee, EmployeeWrapper> mapping =
                    mappings.get(field);

            if (mapping != null) {
                mapping.accept(source, target);
            }
        });

        return target;
    }
}
```

Aqui você conseguiu uma coisa interessante:

**não existe um `if` para cada campo.**

Você tem somente:

```text
allowedFields.forEach(...)
```

E a associação fica centralizada no `Map`.

---

### 10. Redis Adapter

O Redis continua escondendo os nomes físicos:

```java
@Component
public class EmployeeFieldPermissionRedisAdapter
        implements EmployeeFieldPermissionOutput {

    @Override
    public Set<EmployeeField> findAllowedFields(
            String matricula) {

        Set<String> columns =
                findColumnsFromRedis(matricula);

        return columns.stream()
                .map(this::toField)
                .filter(Objects::nonNull)
                .collect(Collectors.toSet());
    }

    private EmployeeField toField(String column) {

        return switch (column) {

            case "NM_EMPLOYEE" ->
                    EmployeeField.NAME;

            case "NR_CPF" ->
                    EmployeeField.CPF;

            case "NR_PHONE" ->
                    EmployeeField.PHONE;

            case "NR_MOBILE" ->
                    EmployeeField.MOBILE;

            case "NM_CITY" ->
                    EmployeeField.CITY;

            case "NM_STATE" ->
                    EmployeeField.STATE;

            case "CD_DEPARTMENT" ->
                    EmployeeField.DEPARTMENT;

            default -> null;
        };
    }
}
```

Assim:

```text
Redis
"NR_CPF"
     ↓
EmployeeField.CPF
```

Depois disso, **nenhuma outra camada conhece `NR_CPF`**.

---

### 11. Persistência

Continua igual:

```text
infrastructure
└── output
    └── persistence
        ├── EmployeeEntity.java
        ├── EmployeeRepository.java
        ├── EmployeePersistenceAdapter.java
        └── EmployeeEntityMapper.java
```

```java
@Mapper(componentModel = "spring")
public interface EmployeeEntityMapper {

    Employee toDomain(EmployeeEntity source);
}
```

---

### 12. Arquitetura final

```text
employee-service
│
├── domain
│   └── model
│       ├── Employee
│       └── Contato
│
├── application
│   │
│   ├── input
│   │   └── EmployeeUseCase
│   │
│   ├── output
│   │   ├── EmployeeOutput
│   │   └── EmployeeFieldPermissionOutput
│   │
│   ├── model
│   │   ├── EmployeeResult
│   │   └── EmployeeField
│   │
│   └── service
│       └── EmployeeService
│
└── infrastructure
    │
    ├── input
    │   │
    │   ├── EmployeeController
    │   │
    │   └── contract
    │       ├── EmployeeWrapper
    │       ├── PersonalData
    │       ├── Address
    │       ├── Contact
    │       └── ...
    │
    │       └── mapper
    │           ├── EmployeeContractMapper
    │           └── ...
    │
    └── output
        │
        ├── persistence
        │   ├── EmployeeEntity
        │   ├── EmployeeRepository
        │   ├── EmployeePersistenceAdapter
        │   └── EmployeeEntityMapper
        │
        └── redis
            └── EmployeeFieldPermissionRedisAdapter
```

### O fluxo final fica muito simples

```text
                    GET /employee/{id}
                            │
                            ▼
                      Controller
                            │
                            ▼
                       UseCase
                            │
                            ▼
                    EmployeeService
                       │          │
                       │          │
                    Banco       Redis
                       │          │
                       ▼          ▼
                    Employee   Fields
                       │          │
                       └────┬─────┘
                            ▼
                     EmployeeResult
                            │
                            ▼
                  EmployeeContractMapper
                            │
                            ▼
                    EmployeeWrapper
                            │
                            ▼
                           JSON
```

### Um detalhe que eu manteria

Eu **não usaria MapStruct no `EmployeeContractMapper`** nesse cenário.

Como agora existe uma regra dinâmica:

```text
"quais campos devem ser preenchidos?"
```

o mapper manual com `Map<EmployeeField, BiConsumer<...>>` faz mais sentido.

O **MapStruct continua sendo excelente para o `Entity → Domain`**, onde o mapeamento é estático.

Assim você fica com:

```text
Entity → Domain
        ↓
     MapStruct

Domain + Permissions → Contract
        ↓
   Mapper manual
```

Isso é, na minha visão, a solução mais coerente com a regra que você adicionou.

---

## Usuário

```java
@Component
public class EmployeeContractMapper {

    private final EmployeeContractFactory factory;

    private final Map<EmployeeField,
            BiConsumer<Employee, EmployeeWrapper>> mappings;

    public EmployeeContractMapper(
            EmployeeContractFactory factory) {

        this.factory = factory;

        mappings = Map.of(
            EmployeeField.NAME,
                (e, t) ->
                    t.getPersonalData()
                     .setNome(e.getNome()),

            EmployeeField.CPF,
                (e, t) ->
                    t.getPersonalData()
                     .setCpf(e.getCpf()),

            EmployeeField.PHONE,
                (e, t) ->
                    t.getContacts().add(
                        new ContatoItem(
                            "TELEFONE",
                            e.getTelefone()
                        )
                    ),

            EmployeeField.MOBILE,
                (e, t) ->
                    t.getContacts().add(
                        new ContatoItem(
                            "CELULAR",
                            e.getCelular()
                        )
                    ),

            EmployeeField.CITY,
                (e, t) ->
                    t.getAddress()
                     .getLocation()
                     .setCidade(e.getCidade()),

            EmployeeField.STATE,
                (e, t) ->
                    t.getAddress()
                     .getLocation()
                     .setEstado(e.getEstado()),

            EmployeeField.DEPARTMENT,
                (e, t) ->
                    t.getOrganization()
                     .getDepartment()
                     .setId(e.getDepartmentId())
        );
    }

    public EmployeeWrapper toContract(
            Employee source,
            Set<EmployeeField> allowedFields) {

        EmployeeWrapper target = factory.create();

        allowedFields.forEach(field -> {

            BiConsumer<Employee, EmployeeWrapper> mapping =
                    mappings.get(field);

            if (mapping != null) {
                mapping.accept(source, target);
            }
        });

        return target;
    }
}
```

Mais o construtor nao quebraria ?

Quando usa o mapping ainda nao foi instsciado, os objetos do target

---

## Assistente

Não. **O construtor não quebra por causa disso.** Há duas coisas diferentes acontecendo.

No construtor você está criando apenas as **funções (`BiConsumer`)**:

```java
mappings = Map.of(
    EmployeeField.NAME,
    (e, t) ->
        t.getPersonalData().setNome(e.getNome())
);
```

Essa lambda **não é executada no construtor**.

Ela só é executada depois, aqui:

```java
mapping.accept(source, target);
```

E nesse momento:

```java
EmployeeWrapper target = factory.create();
```

já foi executado.

O fluxo é:

```text
Criação do Mapper
        │
        ▼
Construtor
        │
        ├── cria Map
        │
        └── cria as lambdas
              │
              │ NÃO executa
              ▼
        Mapper pronto
```

Depois:

```text
toContract()
     │
     ▼
factory.create()
     │
     ▼
EmployeeWrapper
├── PersonalData ✓
├── Address ✓
│   └── Location ✓
├── Organization ✓
│   └── Department ✓
└── Contacts ✓
     │
     ▼
allowedFields.forEach()
     │
     ▼
mapping.accept(source, target)
     │
     ▼
Lambda executada
```

Por exemplo:

```java
EmployeeField.NAME,
    (e, t) ->
        t.getPersonalData()
         .setNome(e.getNome())
```

Só nesse momento `t.getPersonalData()` é acessado.

### Portanto

Isto é seguro:

```java
public EmployeeContractMapper(EmployeeContractFactory factory) {
    this.factory = factory;

    mappings = Map.of(
        EmployeeField.NAME,
        (e, t) -> t.getPersonalData().setNome(e.getNome())
    );
}
```

Porque `t` só existe quando a lambda for executada.

### O que realmente poderia quebrar?

Se a Factory fizer:

```java
public EmployeeWrapper create() {
    return new EmployeeWrapper();
}
```

e `PersonalData` não for inicializado, então **quebra na execução do mapper**, não no construtor:

```java
t.getPersonalData().setNome(...)
```

porque:

```text
t.getPersonalData() == null
```

Por isso a Factory precisa garantir a estrutura:

```java
public EmployeeWrapper create() {

    EmployeeWrapper target = new EmployeeWrapper();

    target.setPersonalData(new PersonalData());

    Address address = new Address();
    address.setLocation(new Location());
    target.setAddress(address);

    Organization organization = new Organization();

    Department department = new Department();
    organization.setDepartment(department);

    target.setOrganization(organization);

    target.setContacts(new ArrayList<>());

    return target;
}
```

Então a regra fica:

> **O construtor cria os mapeamentos. A Factory cria o objeto Target. O `accept()` executa o mapeamento sobre um Target já inicializado.**

Essa separação é segura.

---

## Usuário

Estou usando a versão java 21, algo moderno que posso usar na solução?

---

## Assistente

Sim. Com **Java 21**, dá para deixar essa solução mais moderna, mas eu evitaria usar recursos novos apenas por usar. Para o seu caso, há algumas melhorias que fazem bastante sentido.

### 1. `record` para o resultado do Use Case

Em vez de:

```java
public class EmployeeResult {
    private Employee employee;
    private Set<EmployeeField> allowedFields;
}
```

Use:

```java
public record EmployeeResult(
    Employee employee,
    Set<EmployeeField> allowedFields
) {}
```

Fica excelente para esse objeto, porque ele é apenas um agrupador de dados.

### 2. `Set.of()` / `Map.of()` já combinam muito bem

Sua solução já está usando uma característica moderna do Java:

```java
mappings = Map.of(
    EmployeeField.NAME,
    (e, t) -> t.getPersonalData().setNome(e.getNome()),

    EmployeeField.CPF,
    (e, t) -> t.getPersonalData().setCpf(e.getCpf())
);
```

Como esse mapa é fixo, eu até faria ele `static final`.

```java
private static final Map<
        EmployeeField,
        BiConsumer<Employee, EmployeeWrapper>
    > MAPPINGS = Map.of(
        EmployeeField.NAME,
        (e, t) -> t.getPersonalData().setNome(e.getNome()),

        EmployeeField.CPF,
        (e, t) -> t.getPersonalData().setCpf(e.getCpf()),

        EmployeeField.PHONE,
        (e, t) -> t.getContacts().add(
            new ContatoItem("TELEFONE", e.getTelefone())
        ),

        EmployeeField.MOBILE,
        (e, t) -> t.getContacts().add(
            new ContatoItem("CELULAR", e.getCelular())
        )
    );
```

Aí o mapper não precisa receber o mapa pelo construtor.

### 3. `switch` expression para o Redis

Aqui Java 21 fica bem interessante:

```java
private EmployeeField toField(String column) {

    return switch (column) {
        case "NM_EMPLOYEE" -> EmployeeField.NAME;
        case "NR_CPF" -> EmployeeField.CPF;
        case "NR_PHONE" -> EmployeeField.PHONE;
        case "NR_MOBILE" -> EmployeeField.MOBILE;
        case "NM_CITY" -> EmployeeField.CITY;
        case "NM_STATE" -> EmployeeField.STATE;
        case "CD_DEPARTMENT" -> EmployeeField.DEPARTMENT;
        default -> null;
    };
}
```

### 4. `enum` com a coluna do Redis

Aqui eu faria uma mudança ainda melhor.

Em vez de manter:

```java
case "NR_CPF" -> EmployeeField.CPF;
```

no Adapter, coloque a correspondência no próprio `enum`:

```java
public enum EmployeeField {

    NAME("NM_EMPLOYEE"),
    CPF("NR_CPF"),
    PHONE("NR_PHONE"),
    MOBILE("NR_MOBILE"),
    CITY("NM_CITY"),
    STATE("NM_STATE"),
    DEPARTMENT("CD_DEPARTMENT");

    private final String column;

    EmployeeField(String column) {
        this.column = column;
    }

    public String getColumn() {
        return column;
    }

    public static Optional<EmployeeField> fromColumn(
            String column) {

        return Arrays.stream(values())
                .filter(field -> field.column.equals(column))
                .findFirst();
    }
}
```

Então o Redis adapter fica:

```java
@Override
public Set<EmployeeField> findAllowedFields(
        String matricula) {

    Set<String> columns =
        findColumnsFromRedis(matricula);

    return columns.stream()
        .map(EmployeeField::fromColumn)
        .flatMap(Optional::stream)
        .collect(Collectors.toSet());
}
```

Isso deixa a relação:

```text
NM_EMPLOYEE → NAME
NR_CPF      → CPF
NR_PHONE    → PHONE
```

centralizada.

### 5. `EnumMap`

Como a chave é um `enum`, `EnumMap` também é uma opção apropriada.

Porém, como seu mapa é imutável, `Map.of()` continua sendo perfeitamente aceitável.

## Minha versão final

Eu deixaria o mapper assim:

```java
@Component
public class EmployeeContractMapper {

    private static final Map<
            EmployeeField,
            BiConsumer<Employee, EmployeeWrapper>
            > MAPPINGS = Map.of(

        EmployeeField.NAME,
            (e, t) ->
                t.getPersonalData()
                 .setNome(e.getNome()),

        EmployeeField.CPF,
            (e, t) ->
                t.getPersonalData()
                 .setCpf(e.getCpf()),

        EmployeeField.PHONE,
            (e, t) ->
                t.getContacts().add(
                    new ContatoItem(
                        "TELEFONE",
                        e.getTelefone()
                    )
                ),

        EmployeeField.MOBILE,
            (e, t) ->
                t.getContacts().add(
                    new ContatoItem(
                        "CELULAR",
                        e.getCelular()
                    )
                ),

        EmployeeField.CITY,
            (e, t) ->
                t.getAddress()
                 .getLocation()
                 .setCidade(e.getCidade()),

        EmployeeField.STATE,
            (e, t) ->
                t.getAddress()
                 .getLocation()
                 .setEstado(e.getEstado()),

        EmployeeField.DEPARTMENT,
            (e, t) ->
                t.getOrganization()
                 .getDepartment()
                 .setId(e.getDepartmentId())
    );

    private final EmployeeContractFactory factory;

    public EmployeeContractMapper(
            EmployeeContractFactory factory) {

        this.factory = factory;
    }

    public EmployeeWrapper toContract(
            Employee source,
            Set<EmployeeField> allowedFields) {

        EmployeeWrapper target = factory.create();

        allowedFields.forEach(field -> {

            var mapping = MAPPINGS.get(field);

            if (mapping != null) {
                mapping.accept(source, target);
            }
        });

        return target;
    }
}
```

O `var` aqui também é perfeitamente adequado:

```java
var mapping = MAPPINGS.get(field);
```

porque o tipo fica evidente pelo lado direito.

### E eu manteria esta divisão

```text
Java 21
│
├── record
│   └── EmployeeResult
│
├── enum
│   └── EmployeeField
│
├── switch expression
│   └── quando realmente necessário
│
├── Map.of
│   └── mapeamentos imutáveis
│
├── var
│   └── quando o tipo é óbvio
│
└── Lambda + BiConsumer
    └── seu mapper dinâmico
```

**Eu não usaria virtual threads, pattern matching ou outras features do Java 21 só para modernizar o código.** Elas não trazem benefício direto para esse problema específico.

A solução que você montou com `Factory + Map<EmployeeField, BiConsumer>` já fica bastante adequada ao Java 21.

---

## Usuário

Esse map poderia ser uma fábrica?

---

## Assistente

Sim — e **acho que fica melhor separar**.

Hoje o `EmployeeContractMapper` está fazendo duas coisas:

1. Criando o `Map<EmployeeField, BiConsumer<...>>`.
2. Executando o mapeamento.

Você pode transformar o primeiro em uma **Factory de mapeamentos**.

### 1. Mapping Factory

```java
@Component
public class EmployeeContractMappingFactory {

    public Map<EmployeeField,
            BiConsumer<Employee, EmployeeWrapper>> create() {

        return Map.of(
            EmployeeField.NAME,
                (e, t) ->
                    t.getPersonalData()
                     .setNome(e.getNome()),

            EmployeeField.CPF,
                (e, t) ->
                    t.getPersonalData()
                     .setCpf(e.getCpf()),

            EmployeeField.PHONE,
                (e, t) ->
                    t.getContacts().add(
                        new ContatoItem(
                            "TELEFONE",
                            e.getTelefone()
                        )
                    ),

            EmployeeField.MOBILE,
                (e, t) ->
                    t.getContacts().add(
                        new ContatoItem(
                            "CELULAR",
                            e.getCelular()
                        )
                    ),

            EmployeeField.CITY,
                (e, t) ->
                    t.getAddress()
                     .getLocation()
                     .setCidade(e.getCidade()),

            EmployeeField.STATE,
                (e, t) ->
                    t.getAddress()
                     .getLocation()
                     .setEstado(e.getEstado()),

            EmployeeField.DEPARTMENT,
                (e, t) ->
                    t.getOrganization()
                     .getDepartment()
                     .setId(e.getDepartmentId())
        );
    }
}
```

### 2. Mapper

Agora o mapper fica bem mais limpo:

```java
@Component
public class EmployeeContractMapper {

    private final EmployeeContractFactory contractFactory;
    private final EmployeeContractMappingFactory mappingFactory;

    public EmployeeContractMapper(
            EmployeeContractFactory contractFactory,
            EmployeeContractMappingFactory mappingFactory) {

        this.contractFactory = contractFactory;
        this.mappingFactory = mappingFactory;
    }

    public EmployeeWrapper toContract(
            Employee source,
            Set<EmployeeField> allowedFields) {

        EmployeeWrapper target = contractFactory.create();

        Map<EmployeeField,
                BiConsumer<Employee, EmployeeWrapper>> mappings =
                mappingFactory.create();

        allowedFields.forEach(field -> {

            var mapping = mappings.get(field);

            if (mapping != null) {
                mapping.accept(source, target);
            }
        });

        return target;
    }
}
```

### Mas tem uma melhoria importante

Nesse caso, a Factory não precisa criar o `Map` **toda vez** que `toContract()` for chamado.

Faça o mapa uma única vez:

```java
@Component
public class EmployeeContractMappingFactory {

    private final Map<EmployeeField,
            BiConsumer<Employee, EmployeeWrapper>> mappings;

    public EmployeeContractMappingFactory() {

        this.mappings = Map.of(
            EmployeeField.NAME,
                (e, t) ->
                    t.getPersonalData()
                     .setNome(e.getNome()),

            EmployeeField.CPF,
                (e, t) ->
                    t.getPersonalData()
                     .setCpf(e.getCpf()),

            EmployeeField.PHONE,
                (e, t) ->
                    t.getContacts().add(
                        new ContatoItem(
                            "TELEFONE",
                            e.getTelefone()
                        )
                    ),

            EmployeeField.MOBILE,
                (e, t) ->
                    t.getContacts().add(
                        new ContatoItem(
                            "CELULAR",
                            e.getCelular()
                        )
                    )
        );
    }

    public BiConsumer<Employee, EmployeeWrapper> get(
            EmployeeField field) {

        return mappings.get(field);
    }
}
```

E o mapper:

```java
@Component
public class EmployeeContractMapper {

    private final EmployeeContractFactory contractFactory;
    private final EmployeeContractMappingFactory mappingFactory;

    public EmployeeContractMapper(
            EmployeeContractFactory contractFactory,
            EmployeeContractMappingFactory mappingFactory) {

        this.contractFactory = contractFactory;
        this.mappingFactory = mappingFactory;
    }

    public EmployeeWrapper toContract(
            Employee source,
            Set<EmployeeField> allowedFields) {

        EmployeeWrapper target = contractFactory.create();

        allowedFields.forEach(field -> {

            var mapping = mappingFactory.get(field);

            if (mapping != null) {
                mapping.accept(source, target);
            }
        });

        return target;
    }
}
```

### Ficaria com duas factories

```text
EmployeeContractFactory
        │
        └── cria o TARGET
            EmployeeWrapper
            ├── PersonalData
            ├── Address
            ├── Organization
            └── Contacts


EmployeeContractMappingFactory
        │
        └── cria/fornece as regras
            EmployeeField.NAME → ação
            EmployeeField.CPF  → ação
            EmployeeField.PHONE → ação
            ...
```

E o mapper apenas **orquestra**:

```text
Employee
   +
AllowedFields
   │
   ▼
EmployeeContractMapper
   │
   ├── ContractFactory → cria Target
   │
   └── MappingFactory → encontra ação
                         │
                         ▼
                    preenche Target
```

**Essa separação é interessante no seu caso**, porque agora o mapper deixa de conhecer a construção das regras de mapeamento e também não precisa conhecer como o `Wrapper` é inicializado. Ele apenas coordena as duas coisas.

---

## Usuário

Isso é muito melhor que mapper struct

---

## Assistente

Para **o seu cenário específico**, sim, essa abordagem pode ser mais adequada que usar MapStruct no mapeamento do contrato — mas não significa que seja melhor em geral.

O motivo é a sua regra dinâmica:

```text
Redis
  ↓
campos permitidos
  ↓
somente esses campos entram no contrato
```

O MapStruct é excelente quando o mapeamento é **estático**:

```text
Employee.nome → Wrapper.nome
Employee.cpf  → Wrapper.cpf
```

No seu caso, o mapeamento depende de uma condição externa:

```text
Employee + Set<EmployeeField>
             ↓
       Contract
```

Então seu desenho:

```text
EmployeeContractMapper
        │
        ├── EmployeeContractFactory
        │       └── cria Target
        │
        └── EmployeeContractMappingFactory
                └── fornece a transformação
```

fica bastante natural.

### Comparando

| Característica | MapStruct | Sua abordagem |
|---|---|---|
| Campos fixos | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Campos dinâmicos | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| Redis controla campos | Mais trabalhoso | Natural |
| Contrato muito aninhado | Bom | Bom |
| Transformações especiais | Bom | Muito bom |
| Código gerado | Sim | Não |
| Debug | Indireto | Direto |
| Regras condicionais | Pode ficar complexo | Simples |
| Java puro | Não totalmente | Sim |

E existe uma vantagem arquitetural importante:

```text
MapStruct
Employee → Contract
```

enquanto o seu modelo representa:

```text
Employee
    +
Permissões
    ↓
Contract
```

Isso é uma transformação **contextual**, não apenas um mapeamento de objetos.

### Eu faria inclusive uma distinção

Para persistência:

```text
EmployeeEntity
       ↓
   MapStruct
       ↓
Employee
```

Para contrato:

```text
Employee + AllowedFields
       ↓
EmployeeContractMapper
       ↓
EmployeeWrapper
```

Ou seja, você **não precisa escolher MapStruct ou mapper manual para o projeto inteiro**.

Pode usar cada um onde faz mais sentido.

E, no seu caso, eu manteria o `MappingFactory` como uma estrutura **imutável e criada uma única vez**, não recriada a cada chamada. Isso deixa a solução eficiente e bem previsível.

---

## Usuário

Se é imutável poderia ser static e vez do componente

---

## Assistente

Sim. **Se o mapa é realmente fixo e não depende de injeção de dependências, `static final` é melhor que `@Component`**. Você elimina um bean do Spring sem ganhar nada com ele.

Eu faria assim:

```java
public final class EmployeeContractMappingFactory {

    private EmployeeContractMappingFactory() {
    }

    private static final Map<
            EmployeeField,
            BiConsumer<Employee, EmployeeWrapper>
            > MAPPINGS = Map.of(

        EmployeeField.NAME,
            (e, t) ->
                t.getPersonalData()
                 .setNome(e.getNome()),

        EmployeeField.CPF,
            (e, t) ->
                t.getPersonalData()
                 .setCpf(e.getCpf()),

        EmployeeField.PHONE,
            (e, t) ->
                t.getContacts().add(
                    new ContatoItem(
                        "TELEFONE",
                        e.getTelefone()
                    )
                ),

        EmployeeField.MOBILE,
            (e, t) ->
                t.getContacts().add(
                    new ContatoItem(
                        "CELULAR",
                        e.getCelular()
                    )
                )
    );

    public static BiConsumer<Employee, EmployeeWrapper> get(
            EmployeeField field) {

        return MAPPINGS.get(field);
    }
}
```

E o mapper fica:

```java
@Component
public class EmployeeContractMapper {

    private final EmployeeContractFactory contractFactory;

    public EmployeeContractMapper(
            EmployeeContractFactory contractFactory) {

        this.contractFactory = contractFactory;
    }

    public EmployeeWrapper toContract(
            Employee source,
            Set<EmployeeField> allowedFields) {

        EmployeeWrapper target = contractFactory.create();

        allowedFields.forEach(field -> {

            var mapping =
                    EmployeeContractMappingFactory.get(field);

            if (mapping != null) {
                mapping.accept(source, target);
            }
        });

        return target;
    }
}
```

### Porém, eu mudaria o nome

`Factory` talvez não seja o melhor nome, porque ela **não está criando um objeto novo a cada chamada**. Ela está fornecendo regras de mapeamento.

Eu usaria:

```text
EmployeeContractMappings
```

ou:

```text
EmployeeContractMappingRegistry
```

Por exemplo:

```java
public final class EmployeeContractMappings {

    private EmployeeContractMappings() {
    }

    private static final Map<
            EmployeeField,
            BiConsumer<Employee, EmployeeWrapper>
            > MAPPINGS = Map.of(
        // ...
    );

    public static BiConsumer<Employee, EmployeeWrapper> get(
            EmployeeField field) {

        return MAPPINGS.get(field);
    }
}
```

A arquitetura fica mais clara:

```text
EmployeeContractMapper
       │
       ├── EmployeeContractFactory
       │       └── cria o Target
       │
       └── EmployeeContractMappings
               └── fornece a regra
```

**Factory:** cria objetos.

**Mappings:** contém regras estáticas.

Essa distinção eu acho melhor para o seu desenho.
