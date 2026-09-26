# Conversa — Arquitetura Hexagonal

> Transcrição estruturada da conversa disponível neste contexto, preservando o conteúdo das perguntas e respostas.

---

## Usuário

Arquitetura hexagonal, conceito e teoria e camadas

---

## Usuário

Organizar assim esta correto 
Infraestrutura
- input (Controller)
- output ( adapters (rest,oracle)
-config ( configurações, beans)

Aplication 
- input
- output
-usecase

Domion
-model
-service

Controller > input > service > usecase > output > adapter output > oracle

---

## Usuário

No caso o melhor é caso de uso  chamar o service.
Mais no service que chama o output ?

---

## Usuário

Para uma busca simples.
GetById
Regra de negocio, a busca é de um pedido, caso o pedido tenha mais que 200 reais, retorna pedido excedeu o limite.

---

## Usuário

Então nem faz sentido ter um domein

---

## Usuário

De um exemplo de caso de uso complexo que orquestra vários serviços no domein

---

## Usuário

Então o use case chama o banco, e passa o objeto para o service

---

## Usuário

Toda comunicação entre camadas precisa de  um mapper, com seus próprios dtos?

---

## Assistente

Não. **Mapper + DTO em toda comunicação entre camadas seria excesso de abstração.** Na arquitetura hexagonal, o objetivo é proteger as fronteiras importantes, não criar uma conversão para cada chamada.

Uma regra prática é:

> **Crie DTO/Mapper quando houver uma mudança de responsabilidade, modelo ou contrato. Não simplesmente porque mudou de pacote/camada.**

### No seu exemplo

Eu faria algo assim:

```text
HTTP
 ↓
Controller
 ↓
Input DTO
 ↓
Mapper
 ↓
Use Case
 ↓
Domain
 ↓
Output Port
 ↓
Adapter
 ↓
Persistence
```

Por exemplo:

```text
PedidoRequest
      ↓
PedidoCommand
      ↓
GetPedidoUseCase
      ↓
Pedido
      ↓
PedidoResponse
```

### 1. Controller → Application

Aqui **faz bastante sentido ter DTO próprio**.

```java
public record GetPedidoRequest(
    Long id
) {}
```

ou, para criação:

```java
public record CriarPedidoRequest(
    Long clienteId,
    List<ItemRequest> itens
) {}
```

O Controller transforma isso em um comando da aplicação:

```java
CriarPedidoCommand command =
    mapper.toCommand(request);
```

Porque o contrato HTTP não deveria definir o contrato do Use Case.

---

### 2. Application → Domain

Aqui **não precisa necessariamente de DTO**.

O Use Case pode trabalhar diretamente com o objeto de domínio:

```java
Pedido pedido = repository.getById(id);

pedido.excedeuLimite();
```

Não precisa fazer:

```text
Pedido
 ↓ mapper
PedidoDomainDTO
 ↓ mapper
PedidoServiceDTO
```

Isso seria burocracia sem benefício.

---

### 3. Application → Output Port

Também não necessariamente precisa de DTO.

Por exemplo:

```java
public interface PedidoRepository {

    Optional<Pedido> getById(Long id);
}
```

O Use Case recebe diretamente:

```java
Pedido pedido = repository.getById(id);
```

Aqui o `Pedido` é o modelo de domínio.

---

### 4. Output Adapter → Banco

Aqui normalmente **vale a pena separar** o modelo de persistência do domínio.

Por exemplo:

```text
Domain
  Pedido
    ↓
Persistence
  PedidoEntity
```

E:

```java
PedidoEntity entity =
    mapper.toEntity(pedido);
```

Porque `PedidoEntity` pode ter coisas específicas do JPA:

```java
@Entity
@Table(name = "PEDIDO")
public class PedidoEntity {

    @Id
    private Long id;

    @Column(name = "VL_PEDIDO")
    private BigDecimal valor;
}
```

O domínio não precisa conhecer:

```text
@Entity
@Table
@Column
@OneToMany
Hibernate
JPA
```

---

# Então eu usaria aproximadamente isto

```text
                INFRASTRUCTURE
                     │
                     ▼
              PedidoController
                     │
              PedidoRequest
                     │
                  Mapper
                     │
                     ▼
                APPLICATION
                     │
              CriarPedidoCommand
                     │
                     ▼
                Use Case
                     │
                     ▼
                  DOMAIN
                     │
                   Pedido
                     │
                     ▼
              Output Port
                     │
                     ▼
                INFRASTRUCTURE
                     │
               PedidoEntity
                     │
                  Mapper
                     │
                     ▼
                   Oracle
```

### A regra que eu recomendo

| Fronteira | DTO/Mapper? |
|---|---|
| HTTP → Application | **Sim, normalmente** |
| Application → Domain | **Não necessariamente** |
| Application → Output Port | **Não necessariamente** |
| Domain → Output Port | **Depende** |
| Domain → JPA Entity | **Sim** |
| Domain → REST externo | **Sim** |
| REST externo → Domain | **Sim** |

### O ponto mais importante

Não confunda **camada** com **fronteira de modelo**.

Você pode ter:

```text
Application
    ↓
Domain
```

e continuar usando o mesmo objeto de domínio.

Mas quando você cruza uma fronteira tecnológica ou de contrato:

```text
HTTP
Oracle
Kafka
REST externo
```

a separação com DTO/Entity tende a ser muito mais valiosa.

---

## Usuário

Então application pode usar o domínio?

---

## Assistente

Sim. **Na Arquitetura Hexagonal, a camada `Application` pode — e normalmente vai — usar o `Domain`.**

A direção esperada é:

```text
Infrastructure
       ↓
Application
       ↓
Domain
```

Mas o inverso não:

```text
Domain
   ❌
Application
```

### Exemplo

Você tem:

```text
domain
└── model
    └── Pedido
```

```java
public class Pedido {

    private BigDecimal valor;

    public boolean excedeuLimite() {
        return valor.compareTo(
            new BigDecimal("200.00")
        ) > 0;
    }
}
```

Na Application:

```java
public class GetPedidoUseCase {

    private final PedidoRepository repository;

    public PedidoResponse executar(Long id) {

        Pedido pedido = repository.getById(id);

        if (pedido.excedeuLimite()) {
            return PedidoResponse.excedeuLimite(pedido);
        }

        return PedidoResponse.normal(pedido);
    }
}
```

Aqui:

```text
Application
     │
     │ usa
     ▼
Domain
  Pedido
```

Isso é **correto**.

---

## O que não pode acontecer

O `Domain` começar a conhecer a `Application`:

```java
// ❌ Não fazer
public class Pedido {

    private GetPedidoUseCase useCase;
}
```

Ou:

```text
Domain
   ↓
Application
   ↓
Infrastructure
```

Isso inverteria a dependência.

---

## E o Infrastructure?

Também pode depender da Application:

```text
Infrastructure
       ↓
Application
       ↓
Domain
```

Por exemplo:

```java
@RestController
public class PedidoController {

    private final GetPedidoUseCase useCase;
}
```

O Controller conhece o **Input Port** da Application.

---

# A regra geral

Pense em **dependência**, não simplesmente em "quem chama quem":

```text
┌───────────────────────────┐
│      Infrastructure       │
│                           │
│ Controller / Oracle / REST│
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│       Application         │
│                           │
│ Use Cases / Ports         │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│          Domain           │
│                           │
│ Entities / Rules / VO     │
└───────────────────────────┘
```

**As dependências apontam para dentro.**

E isso explica por que a Application pode usar o Domain: o **Domain é mais interno e mais estável**.

Um detalhe: os **Output Ports** normalmente ficam na `Application`, e os adapters ficam na `Infrastructure`. Assim, a Application também define **o contrato do que precisa externamente**, sem conhecer Oracle, REST, Kafka etc.

---

## Usuário

Infra > application > domínio < infra

---

## Assistente

Quase. A sua representação:

```text
Infrastructure → Application → Domain ← Infrastructure
```

é **uma boa forma de visualizar**, mas precisamos separar **fluxo de execução** de **direção das dependências**.

### Direção das dependências

O desenho fica:

```text
Infrastructure
       ↓
Application
       ↓
Domain
```

E a Infrastructure também **implementa os ports definidos mais internamente**:

```text
                  ┌──────────────┐
                  │   DOMAIN     │
                  └──────▲───────┘
                         │
                  ┌──────┴───────┐
                  │ APPLICATION  │
                  │ Input / Output│
                  └──────▲───────┘
                         │
                  ┌──────┴───────┐
                  │INFRASTRUCTURE│
                  │Controller    │
                  │Oracle Adapter│
                  │REST Adapter  │
                  └──────────────┘
```

Ou, de forma mais intuitiva:

```text
              ┌──────────────┐
              │   DOMAIN     │
              │              │
              │ Pedido       │
              │ Regras       │
              └──────▲───────┘
                     │
                     │ usa
                     │
              ┌──────┴───────┐
              │ APPLICATION  │
              │              │
              │ Use Cases    │
              │ Input Ports  │
              │ Output Ports │
              └──────▲───────┘
                     │
                     │ implementa
                     │
              ┌──────┴───────┐
              │INFRASTRUCTURE│
              │              │
              │ Controller   │
              │ Oracle       │
              │ REST         │
              └──────────────┘
```

### Então seu raciocínio está certo em um ponto importante

Você pode pensar:

```text
Infrastructure
      ↓
Application
      ↓
Domain
      ↑
Infrastructure
```

O **primeiro `Infrastructure → Application`** acontece porque o Controller chama o Use Case.

O **segundo `Infrastructure → Domain/Application`** representa os adapters implementando os contratos definidos internamente.

Por exemplo:

```text
Controller
    ↓
GetPedidoUseCase
    ↓
PedidoRepository ←──────── PedidoOracleAdapter
    ↓                         ↑
Pedido                       │
    │                         │
    └── regra                 │
                              │
                            Oracle
```

O ponto fundamental da Hexagonal é:

> **A infraestrutura pode estar dos dois lados do hexágono, mas o domínio não conhece a infraestrutura.**

É justamente por isso que o desenho de "hexágono" faz sentido: **entrada e saída podem estar em vários lados, enquanto o domínio permanece no centro.**

---

## Usuário

Padrão command

---

## Assistente

Sim. Dentro do que estamos discutindo de **Arquitetura Hexagonal + Application + Domain**, o **Command Pattern** encaixa muito bem na camada de Application.

## 1. O que é Command?

O padrão **Command** transforma uma operação em um **objeto que representa uma intenção**.

Em vez de:

```java
useCase.executar(
    pedidoId,
    clienteId,
    valor,
    endereco
);
```

você cria:

```java
useCase.executar(
    new CriarPedidoCommand(
        pedidoId,
        clienteId,
        valor,
        endereco
    )
);
```

O objeto representa:

> "Quero criar este pedido."

---

# 2. Onde ele fica na arquitetura?

Eu organizaria assim:

```text
Infrastructure
│
└── Controller
       │
       │ Request DTO
       ▼
Application
│
├── input
│   ├── CriarPedidoUseCase
│   └── CriarPedidoCommand
│
├── usecase
│   └── CriarPedidoUseCaseImpl
│
└── output
    └── PedidoRepository
            │
            ▼
Domain
│
├── model
│   └── Pedido
│
└── service
```

Fluxo:

```text
HTTP
 ↓
Controller
 ↓
Request DTO
 ↓
Command
 ↓
Use Case
 ↓
Domain
 ↓
Output Port
 ↓
Adapter
 ↓
Oracle
```

---

# 3. Exemplo

### Command

```java
public record CriarPedidoCommand(
    Long clienteId,
    List<ItemCommand> itens
) {}
```

Ele **não deveria conter lógica de negócio**.

É apenas os dados necessários para executar a intenção.

---

### Input Port

```java
public interface CriarPedidoUseCase {

    PedidoResponse executar(
        CriarPedidoCommand command
    );
}
```

---

### Use Case

```java
public class CriarPedidoUseCaseImpl
        implements CriarPedidoUseCase {

    private final PedidoRepository repository;
    private final PedidoService pedidoService;

    @Override
    public PedidoResponse executar(
        CriarPedidoCommand command
    ) {

        Pedido pedido =
            pedidoService.criar(command);

        Pedido salvo =
            repository.salvar(pedido);

        return PedidoResponse.from(salvo);
    }
}
```

---

# 4. Command não é DTO?

Eles são parecidos, mas **conceitualmente têm papéis diferentes**.

Um DTO representa **dados transferidos entre uma interface**.

Um Command representa **uma intenção da aplicação**.

Por exemplo:

```text
HTTP
 ↓
CriarPedidoRequest
```

é DTO da API.

Depois:

```text
CriarPedidoRequest
       ↓
CriarPedidoCommand
```

O Command pertence à Application.

```java
public record CriarPedidoRequest(
    Long clienteId,
    List<ItemRequest> itens
) {}
```

↓

```java
public record CriarPedidoCommand(
    Long clienteId,
    List<ItemCommand> itens
) {}
```

Você ganha isolamento:

```text
REST
 ↓
Request DTO
 ↓
Command
 ↓
Application
```

Se amanhã trocar REST por Kafka:

```text
Kafka
 ↓
Kafka Message
 ↓
Command
 ↓
Use Case
```

O Use Case não precisa mudar.

---

# 5. Command para cada Use Case

Uma abordagem muito limpa é:

```text
application
│
├── input
│   ├── CriarPedidoUseCase.java
│   ├── CriarPedidoCommand.java
│   │
│   ├── AprovarPedidoUseCase.java
│   ├── AprovarPedidoCommand.java
│   │
│   └── CancelarPedidoUseCase.java
│       └── CancelarPedidoCommand.java
```

Então:

```text
CriarPedidoCommand
       ↓
CriarPedidoUseCase

AprovarPedidoCommand
       ↓
AprovarPedidoUseCase

CancelarPedidoCommand
       ↓
CancelarPedidoUseCase
```

Isso deixa a Application muito explícita.

---

# 6. E o Query?

Aqui fica interessante.

Em arquiteturas que usam **CQRS**, você separa:

```text
COMMAND
   ↓
altera estado

QUERY
   ↓
consulta estado
```

Por exemplo:

```text
Commands
├── CriarPedidoCommand
├── AprovarPedidoCommand
└── CancelarPedidoCommand

Queries
├── BuscarPedidoPorIdQuery
└── BuscarPedidosClienteQuery
```

Assim:

```text
POST /pedidos
     ↓
CriarPedidoCommand
     ↓
CriarPedidoUseCase
```

Enquanto:

```text
GET /pedidos/10
     ↓
BuscarPedidoPorIdQuery
     ↓
BuscarPedidoUseCase
```

### No seu `GetById`

Você poderia ter:

```java
public record BuscarPedidoPorIdQuery(
    Long pedidoId
) {}
```

E:

```java
public interface BuscarPedidoUseCase {

    PedidoResponse executar(
        BuscarPedidoPorIdQuery query
    );
}
```

---

## 7. Uma distinção importante

**Command Pattern** e **CQRS** não são a mesma coisa.

Você pode usar Command sem CQRS:

```text
Command
  ↓
Use Case
```

sem ter:

```text
Command side
      +
Query side
```

CQRS é uma decisão arquitetural maior.

---

### Para o seu projeto

Eu pensaria assim:

```text
                    INFRASTRUCTURE
                         │
                    Controller
                         │
                    Request DTO
                         │
                         ▼
                    APPLICATION
                         │
                ┌────────┴────────┐
                │                 │
             Command            Query
                │                 │
                ▼                 ▼
             Use Case          Use Case
                │                 │
                └────────┬────────┘
                         ▼
                       DOMAIN
                         │
                   Entity / Rules
                         │
                         ▼
                    Output Port
                         │
                         ▼
                    Infrastructure
                         │
                    Oracle / REST
```

**Command = "faça algo".**  
**Query = "me informe algo".**

E o **Use Case é quem executa a intenção representada pelo Command**.

---

## Usuário

No input e output de app,  geralmente são interfaces?

---

## Assistente

Sim. **Em uma implementação típica de Arquitetura Hexagonal, `application/input` e `application/output` são interfaces (Ports)**.

A ideia é:

```text
Application
├── input       ← interfaces que a aplicação oferece
├── output      ← interfaces que a aplicação precisa
└── usecase     ← implementações dos casos de uso
```

### Input Port

É a interface que define **o que a aplicação pode fazer**.

```java
public interface CriarPedidoUseCase {

    PedidoResponse executar(
        CriarPedidoCommand command
    );
}
```

O Controller depende dessa interface:

```text
Controller
    ↓
CriarPedidoUseCase  ← interface
    ↑
CriarPedidoUseCaseImpl
```

---

### Output Port

É a interface que define **o que a aplicação precisa de algo externo**.

```java
public interface PedidoRepository {

    Optional<Pedido> getById(Long id);

    Pedido salvar(Pedido pedido);
}
```

O Use Case depende dela:

```text
Use Case
   ↓
PedidoRepository  ← interface
   ↑
PedidoOracleAdapter
```

O adapter implementa:

```java
public class PedidoOracleAdapter
        implements PedidoRepository {

    // acesso ao Oracle
}
```

---

## A arquitetura fica assim

```text
                 APPLICATION
          ┌──────────────────────┐
          │                      │
Controller → Input Port          │
          │       ↓              │
          │    Use Case           │
          │       ↓              │
          │   Output Port         │
          │                      │
          └──────────▲───────────┘
                     │
                     │ implements
                     │
              INFRASTRUCTURE
                     │
              Oracle Adapter
```

Ou pensando em **dependência**:

```text
Controller
    ↓
Input Port
    ↑
UseCase

UseCase
    ↓
Output Port
    ↑
Oracle Adapter
```

O ponto importante é que **a seta de implementação aponta para a interface**, não para a tecnologia.

### E onde ficam os Commands?

Normalmente:

```text
application
├── input
│   ├── CriarPedidoUseCase.java
│   └── CriarPedidoCommand.java
│
├── output
│   └── PedidoRepository.java
│
└── usecase
    └── CriarPedidoUseCaseImpl.java
```

Então você pode pensar:

- **Input = contrato de entrada da Application**
- **Output = contrato de saída da Application**
- **UseCase = implementação do comportamento**
- **Command = dados/intenção de entrada**
- **Adapter = implementação tecnológica do Output**
- **Domain = regras de negócio**

Isso é uma estrutura bastante clássica para Hexagonal.

---

## Usuário

Oque é preseters no hexagonal

---

## Assistente

Se você quis dizer **“presenters”** (às vezes aparece como `presenter` em exemplos de Hexagonal/Clean Architecture), eles são responsáveis por **transformar o resultado da aplicação em um formato adequado para quem chamou**.

Mas há uma diferença importante: **Presenter não é um elemento obrigatório da Arquitetura Hexagonal**. Ele aparece mais explicitamente na **Clean Architecture**.

### Exemplo

Imagine:

```text
Controller
   ↓
Input Port
   ↓
Use Case
   ↓
Domain
   ↓
Output Port
   ↓
Oracle
```

Depois que o Use Case termina, você precisa transformar o resultado:

```text
Pedido
   ↓
Presenter
   ↓
HTTP Response
```

Por exemplo, o domínio pode produzir:

```java
Pedido
```

e o Presenter transformar em:

```json
{
  "id": 10,
  "valor": 250.00,
  "status": "APROVADO"
}
```

### Fluxo

```text
HTTP Request
     ↓
Controller
     ↓
Input Port
     ↓
Use Case
     ↓
Domain
     ↓
Output Port
     ↓
Oracle Adapter
     ↓
Pedido
     ↓
Presenter
     ↓
HTTP Response
```

### Presenter ≠ Mapper exatamente

Um **Mapper** normalmente converte modelos:

```text
Pedido → PedidoResponse
```

Um **Presenter** pode fazer isso também, mas sua responsabilidade arquitetural é mais ampla:

> **adaptar a saída do caso de uso para o formato que o consumidor precisa.**

Por exemplo, o mesmo Use Case poderia ser consumido por:

```text
             Use Case
             /      \
            ↓        ↓
       REST Presenter
                     Kafka Presenter
```

REST:

```json
{
  "id": 10,
  "valor": 250
}
```

Kafka poderia produzir:

```json
{
  "pedidoId": 10,
  "valorTotal": 250,
  "evento": "PEDIDO_APROVADO"
}
```

O **domínio não precisa saber nada disso**.

### Onde colocar?

Se você quiser usar Presenter explicitamente:

```text
infrastructure
└── input
    └── web
        ├── PedidoController
        ├── PedidoRequest
        └── PedidoPresenter
```

Ou, dependendo do estilo:

```text
application
└── output
    └── PedidoOutput
```

Mas para um projeto Spring Boot + Hexagonal, **eu não criaria Presenter só por criar**. Para APIs simples, um `Response DTO + Mapper` no adapter costuma ser suficiente.

A distinção mais importante é:

```text
Command       → entrada/intenção
Use Case      → executa/orquestra
Domain        → regras de negócio
Output Port   → contrato de saída
Adapter       → tecnologia externa
Presenter     → adapta o resultado para o consumidor
```

**Presenter é mais associado à Clean Architecture; na Hexagonal, você pode ter a mesma responsabilidade usando um adapter de saída/DTO mapper.**

---

## Usuário

Mais no final, depois de fazer todas as regras, preciso entrega o resultado totalmente transformado mudando classes e atributos

---

## Assistente

Exatamente. **Nesse cenário, faz bastante sentido ter um Presenter/Output Mapper**, porque o objeto que representa o domínio não deveria necessariamente ser o mesmo objeto que você entrega para o consumidor.

Imagine:

```text
Banco
  ↓
PedidoEntity
  ↓
Pedido
  ↓
Regras de negócio
  ↓
Resultado do Use Case
  ↓
Presenter / Mapper
  ↓
PedidoResponse
  ↓
Controller
  ↓
JSON
```

### Exemplo

Seu domínio:

```java
public class Pedido {

    private Long id;
    private BigDecimal valor;
    private StatusPedido status;

    public boolean excedeuLimite() {
        return valor.compareTo(
            new BigDecimal("200")
        ) > 0;
    }
}
```

Depois de executar as regras, você **não precisa devolver `Pedido` diretamente**.

Pode criar:

```java
public record PedidoResponse(
    Long codigo,
    BigDecimal valorTotal,
    String situacao,
    boolean acimaDoLimite
) {}
```

E transformar:

```java
public class PedidoPresenter {

    public PedidoResponse present(Pedido pedido) {

        return new PedidoResponse(
            pedido.getId(),
            pedido.getValor(),
            pedido.getStatus().name(),
            pedido.excedeuLimite()
        );
    }
}
```

Então:

```text
                 DOMAIN
                    │
                  Pedido
                    │
                    ▼
              Use Case finaliza
                    │
                    ▼
              PedidoPresenter
                    │
                    ▼
              PedidoResponse
                    │
                    ▼
                Controller
                    │
                    ▼
                  JSON
```

### E pode ser uma transformação grande

Por exemplo, o domínio tem:

```text
Pedido
 ├── Cliente
 ├── itens
 ├── valor
 ├── status
 └── descontos
```

Mas a API quer:

```json
{
  "pedido": {
    "codigo": 123,
    "cliente": "João",
    "total": 850.00
  },
  "resumo": {
    "quantidadeItens": 5,
    "desconto": 100.00,
    "frete": 30.00
  },
  "analise": {
    "excedeuLimite": true,
    "mensagem": "Pedido excedeu o limite"
  }
}
```

Isso é justamente uma situação em que **não é interessante expor o Domain Model**.

### O ponto arquitetural importante

Eu separaria:

```text
Pedido
```

de:

```text
PedidoResponse
```

Porque eles têm responsabilidades diferentes.

**Domain Model:**

> representa o negócio.

**Response/Presenter:**

> representa como quero apresentar o resultado para o consumidor.

E isso permite que você mude:

```text
Pedido
```

sem necessariamente quebrar:

```text
API
```

e vice-versa.

---

### Uma estrutura que eu acho muito boa para seu caso

```text
application
│
├── input
│   ├── GetPedidoUseCase.java
│   └── GetPedidoQuery.java
│
├── output
│   └── PedidoRepository.java
│
└── usecase
    └── GetPedidoUseCaseImpl.java


domain
│
└── model
    └── Pedido.java


infrastructure
│
├── input
│   └── web
│       ├── PedidoController.java
│       ├── PedidoRequest.java
│       └── PedidoPresenter.java
│
└── output
    └── oracle
        ├── PedidoEntity.java
        ├── PedidoOracleAdapter.java
        └── PedidoEntityMapper.java
```

O fluxo completo:

```text
HTTP Request
     ↓
Controller
     ↓
Request / Query
     ↓
Use Case
     ↓
Domain
     ↓
Output Port
     ↓
Output Adapter
     ↓
Oracle
     ↓
Pedido
     ↓
Use Case
     ↓
Presenter
     ↓
Response DTO
     ↓
HTTP Response
```

**Só faria uma ressalva:** se o Presenter estiver transformando o resultado **para HTTP**, eu o colocaria junto do adapter de entrada/web, porque HTTP é uma preocupação de infraestrutura. Se a transformação for um **resultado/DTO próprio da Application**, pode fazer sentido manter esse contrato na Application.

A ideia principal é: **o domínio termina suas regras sem precisar saber como o resultado será apresentado.** O Presenter pega o resultado e faz a transformação final.

---

## Usuário

Presenters geralmente fica na infra input ?

---

## Assistente

Sim. **Se o Presenter é responsável por transformar o resultado do Use Case em uma resposta HTTP, o lugar mais natural é `infrastructure/input`**, junto do Controller.

Porque o Controller é o **adaptador de entrada**, e o Presenter está adaptando a saída desse fluxo para o consumidor HTTP.

Uma estrutura boa seria:

```text
infrastructure
└── input
    └── web
        ├── controller
        │   └── PedidoController.java
        │
        ├── request
        │   └── CriarPedidoRequest.java
        │
        └── presenter
            └── PedidoPresenter.java
```

Fluxo:

```text
HTTP Request
     ↓
Controller
     ↓
Input Port
     ↓
Use Case
     ↓
Domain
     ↓
Output Port
     ↓
Output Adapter
     ↓
Oracle
     ↓
resultado
     ↓
Presenter
     ↓
Response DTO
     ↓
HTTP Response
```

### Por que `input` se o Presenter é uma saída?

Porque estamos falando do **lado da aplicação que recebe a requisição**.

O `Controller` é o **Inbound Adapter**:

```text
       INBOUND
          │
          ▼
┌─────────────────┐
│    Controller   │
│    Presenter    │
└─────────────────┘
          │
          ▼
      Application
```

O Presenter não está conversando com Oracle, Kafka etc. Ele está preparando a resposta para **quem entrou pela porta HTTP**.

---

### Exemplo

O Use Case retorna algo da Application:

```java
public record PedidoResult(
    Pedido pedido
) {}
```

O Presenter transforma:

```java
@Component
public class PedidoPresenter {

    public PedidoResponse present(PedidoResult result) {

        Pedido pedido = result.pedido();

        return new PedidoResponse(
            pedido.getId(),
            pedido.getValor(),
            pedido.excedeuLimite()
                ? "EXCEDEU_LIMITE"
                : "NORMAL"
        );
    }
}
```

Controller:

```java
@GetMapping("/{id}")
public ResponseEntity<PedidoResponse> getById(
    @PathVariable Long id
) {

    PedidoResult result =
        useCase.execute(
            new GetPedidoQuery(id)
        );

    return ResponseEntity.ok(
        presenter.present(result)
    );
}
```

### Uma regra simples para guardar

```text
infrastructure/input
    ↓
coisas relacionadas à entrada/saída do consumidor
    ├── Controller
    ├── Request DTO
    ├── Response DTO
    └── Presenter
```

Enquanto:

```text
infrastructure/output
    ↓
coisas relacionadas às dependências externas
    ├── Oracle
    ├── REST Client
    ├── Kafka Producer
    └── S3
```

**Portanto, para o cenário que você descreveu — transformar completamente o resultado final para uma resposta HTTP — `infrastructure/input/presenter` é uma ótima organização.**

---

## Usuário

Se eu pegar os valores do Path do get e colocar em uma classe request
Está certo ?
Ou é melhor colocar em um pacote vo

---

## Assistente

Sim, **está certo colocar os valores do `@PathVariable` em uma classe `Request`**. Mas eu separaria duas coisas: **DTO de entrada HTTP** e **Value Object (VO)**.

### Exemplo

Você tem:

```http
GET /pedidos/123
```

Controller:

```java
@GetMapping("/pedidos/{id}")
public ResponseEntity<?> buscar(
    @PathVariable Long id
) {
    ...
}
```

Você pode transformar isso em:

```java
public record GetPedidoRequest(
    Long id
) {}
```

e:

```java
@GetMapping("/pedidos/{id}")
public ResponseEntity<?> buscar(
    @PathVariable Long id
) {

    var request = new GetPedidoRequest(id);

    return ...
}
```

Isso é **válido**.

---

## Mas eu faria uma distinção

### Request

Representa **a entrada da interface HTTP**.

```text
infrastructure/input
    └── web
        └── request
            └── GetPedidoRequest
```

Exemplo:

```java
public record GetPedidoRequest(
    Long id
) {}
```

Ele pode existir porque a API recebe esse dado dessa maneira.

---

### VO — Value Object

Representa **um conceito do domínio**.

Por exemplo, se `id` fosse simplesmente:

```java
Long id
```

não há necessariamente motivo para criar um VO.

Mas imagine:

```text
CPF
CNPJ
Money
Email
PedidoId
CodigoProduto
```

Aí um VO pode fazer muito sentido.

Exemplo:

```java
public record PedidoId(Long value) {

    public PedidoId {
        if (value == null || value <= 0) {
            throw new IllegalArgumentException(
                "PedidoId inválido"
            );
        }
    }
}
```

Agora:

```text
GET /pedidos/123
       ↓
GetPedidoRequest
       ↓
PedidoId
       ↓
UseCase
       ↓
Domain
```

---

# O que eu evitaria

Não criaria VO simplesmente para cada parâmetro:

```text
Long
 ↓
PedidoId

String
 ↓
Nome

String
 ↓
Descricao

BigDecimal
 ↓
Valor
```

**VO deve representar um conceito que tenha significado ou comportamento próprio no domínio.**

---

## No seu caso

Para:

```http
GET /pedidos/{id}
```

se `id` é apenas um identificador simples, eu faria:

```text
Controller
   ↓
GetPedidoRequest
   ↓
GetPedidoQuery
   ↓
UseCase
   ↓
Repository
```

Se `PedidoId` tiver regras próprias ou for importante para o domínio:

```text
Controller
   ↓
GetPedidoRequest
   ↓
PedidoId (VO)
   ↓
GetPedidoQuery
   ↓
UseCase
```

### E tem uma alternativa ainda mais simples

Para um `GET` com apenas um ID, **você nem é obrigado a criar `Request`**:

```java
@GetMapping("/{id}")
public ResponseEntity<?> buscar(
    @PathVariable Long id
) {
    return ...
}
```

Você pode criar um `Query` na Application:

```java
public record GetPedidoQuery(Long id) {}
```

e fazer:

```text
@PathVariable
     ↓
GetPedidoQuery
     ↓
UseCase
```

Para uma aplicação Hexagonal, eu gosto dessa separação:

```text
HTTP Request DTO
      ↓
Command / Query
      ↓
Use Case
```

e **VO somente quando existe um conceito de domínio que merece ser encapsulado**.

---

## Usuário

Implemente exemplo, java , padrão de paginação hateos

---

## Assistente

Sim. Vamos montar um exemplo completo em **Java + Spring Boot + Arquitetura Hexagonal + paginação + HATEOAS**, mantendo a separação que discutimos.

A ideia será:

```text
GET /pedidos?page=0&size=10

Controller
   ↓
Query
   ↓
Use Case
   ↓
Output Port
   ↓
Oracle Adapter
   ↓
Page<Pedido>
   ↓
Use Case Result
   ↓
Presenter
   ↓
PagedModel<PedidoResponse>
   ↓
JSON + _links
```

## 1. Estrutura

```text
com.example.pedido
│
├── domain
│   └── model
│       └── Pedido.java
│
├── application
│   ├── input
│   │   ├── BuscarPedidosUseCase.java
│   │   └── BuscarPedidosQuery.java
│   │
│   ├── output
│   │   └── PedidoRepository.java
│   │
│   └── usecase
│       └── BuscarPedidosUseCaseImpl.java
│
└── infrastructure
    │
    ├── input
    │   └── web
    │       ├── PedidoController.java
    │       ├── request
    │       │   └── BuscarPedidosRequest.java
    │       ├── response
    │       │   └── PedidoResponse.java
    │       └── presenter
    │           └── PedidoPresenter.java
    │
    └── output
        └── oracle
            ├── PedidoEntity.java
            ├── PedidoJpaRepository.java
            └── PedidoOracleAdapter.java
```

# 2. Domain

O domínio não conhece:

- Spring MVC
- HATEOAS
- `Page`
- JPA
- Oracle
- `ResponseEntity`

```java
public class Pedido {

    private final Long id;
    private final BigDecimal valor;
    private final StatusPedido status;

    public Pedido(
            Long id,
            BigDecimal valor,
            StatusPedido status) {

        this.id = id;
        this.valor = valor;
        this.status = status;
    }

    public Long getId() {
        return id;
    }

    public BigDecimal getValor() {
        return valor;
    }

    public StatusPedido getStatus() {
        return status;
    }

    public boolean excedeuLimite() {
        return valor.compareTo(
                new BigDecimal("200.00")
        ) > 0;
    }
}
```

```java
public enum StatusPedido {
    PENDENTE,
    APROVADO,
    CANCELADO
}
```

# 3. Query

Como estamos fazendo uma consulta, podemos usar **Query** em vez de Command.

```java
public record BuscarPedidosQuery(
        int page,
        int size
) {
}
```

Você pode evoluir posteriormente para:

```java
public record BuscarPedidosQuery(
        int page,
        int size,
        String status,
        Long clienteId
) {
}
```

# 4. Input Port

```java
public interface BuscarPedidosUseCase {

    BuscarPedidosResult executar(
            BuscarPedidosQuery query
    );
}
```

# 5. Output Port

Aqui temos uma decisão importante.

O `Page` é uma abstração do Spring Data. Se você quiser manter a Application totalmente independente do Spring, eu **não colocaria `Page<Pedido>` diretamente no port**.

Criaria uma estrutura própria:

```java
public record PageResult<T>(
        List<T> content,
        int page,
        int size,
        long totalElements,
        int totalPages
) {
}
```

Então:

```java
public interface PedidoRepository {

    PageResult<Pedido> buscar(
            int page,
            int size
    );
}
```

Isso mantém:

```text
Application
      ↓
não conhece Spring Data
```

# 6. Resultado do Use Case

```java
public record BuscarPedidosResult(
        PageResult<Pedido> pagina
) {
}
```

# 7. Implementação do Use Case

O Use Case apenas orquestra:

```java
public class BuscarPedidosUseCaseImpl
        implements BuscarPedidosUseCase {

    private final PedidoRepository repository;

    public BuscarPedidosUseCaseImpl(
            PedidoRepository repository) {

        this.repository = repository;
    }

    @Override
    public BuscarPedidosResult executar(
            BuscarPedidosQuery query) {

        PageResult<Pedido> pagina =
                repository.buscar(
                        query.page(),
                        query.size()
                );

        return new BuscarPedidosResult(pagina);
    }
}
```

Observe que ele **não sabe nada sobre HATEOAS**.

Isso é importante.

# 8. Entity do Oracle

Agora entramos na Infrastructure.

```java
@Entity
@Table(name = "PEDIDO")
public class PedidoEntity {

    @Id
    private Long id;

    @Column(name = "VALOR")
    private BigDecimal valor;

    @Enumerated(EnumType.STRING)
    @Column(name = "STATUS")
    private StatusPedido status;

    // getters/setters
}
```

# 9. Spring Data

```java
public interface PedidoJpaRepository
        extends JpaRepository<PedidoEntity, Long> {
}
```

Aqui podemos usar a paginação nativa do Spring:

```java
Page<PedidoEntity>
```

# 10. Adapter Oracle

O Adapter converte:

```text
PedidoEntity
      ↓
   Pedido
```

e:

```text
Page Spring
      ↓
PageResult
```

```java
@Component
public class PedidoOracleAdapter
        implements PedidoRepository {

    private final PedidoJpaRepository repository;

    public PedidoOracleAdapter(
            PedidoJpaRepository repository) {

        this.repository = repository;
    }

    @Override
    public PageResult<Pedido> buscar(
            int page,
            int size) {

        Pageable pageable =
                PageRequest.of(page, size);

        Page<PedidoEntity> result =
                repository.findAll(pageable);

        List<Pedido> pedidos =
                result.getContent()
                        .stream()
                        .map(this::toDomain)
                        .toList();

        return new PageResult<>(
                pedidos,
                result.getNumber(),
                result.getSize(),
                result.getTotalElements(),
                result.getTotalPages()
        );
    }

    private Pedido toDomain(
            PedidoEntity entity) {

        return new Pedido(
                entity.getId(),
                entity.getValor(),
                entity.getStatus()
        );
    }
}
```

Agora temos:

```text
Oracle
   ↓
JPA
   ↓
PedidoEntity
   ↓
Adapter
   ↓
Pedido
```

# 11. Request HTTP

Para:

```http
GET /pedidos?page=0&size=10
```

podemos ter:

```java
public record BuscarPedidosRequest(
        Integer page,
        Integer size
) {

    public int getPage() {
        return page == null || page < 0
                ? 0
                : page;
    }

    public int getSize() {
        return size == null || size <= 0
                ? 10
                : Math.min(size, 100);
    }
}
```

Mas para `@RequestParam`, eu particularmente simplificaria e deixaria a validação no Controller.

# 12. Response

Agora criamos **outro modelo** para API.

```java
public record PedidoResponse(
        Long id,
        BigDecimal valor,
        String status,
        boolean excedeuLimite
) {
}
```

Observe:

```text
Pedido
   ≠
PedidoResponse
```

Isso é exatamente a separação que estávamos discutindo.

# 13. Presenter

Agora vem o HATEOAS.

O Presenter fica na Infrastructure/Input porque ele está preparando a resposta HTTP.

```java
@Component
public class PedidoPresenter {

    public EntityModel<PedidoResponse> present(
            Pedido pedido) {

        PedidoResponse response =
                new PedidoResponse(
                        pedido.getId(),
                        pedido.getValor(),
                        pedido.getStatus().name(),
                        pedido.excedeuLimite()
                );

        return EntityModel.of(
                response,
                linkTo(
                    methodOn(PedidoController.class)
                        .buscarPorId(pedido.getId())
                ).withSelfRel()
        );
    }
}
```

O resultado individual terá:

```json
{
  "id": 10,
  "valor": 250.00,
  "status": "APROVADO",
  "excedeuLimite": true,
  "_links": {
    "self": {
      "href": "/pedidos/10"
    }
  }
}
```

# 14. Paginação HATEOAS

Agora podemos criar:

```java
PagedModel<EntityModel<PedidoResponse>>
```

O Presenter:

```java
@Component
public class PedidoPresenter {

    public PagedModel<EntityModel<PedidoResponse>> present(
            BuscarPedidosResult result) {

        PageResult<Pedido> pagina =
                result.pagina();

        List<EntityModel<PedidoResponse>> content =
                pagina.content()
                        .stream()
                        .map(this::present)
                        .toList();

        PagedModel.PageMetadata metadata =
                new PagedModel.PageMetadata(
                        pagina.size(),
                        pagina.page(),
                        pagina.totalElements(),
                        pagina.totalPages()
                );

        return PagedModel.of(
                content,
                metadata
        );
    }

    private EntityModel<PedidoResponse> present(
            Pedido pedido) {

        PedidoResponse response =
                new PedidoResponse(
                        pedido.getId(),
                        pedido.getValor(),
                        pedido.getStatus().name(),
                        pedido.excedeuLimite()
                );

        return EntityModel.of(
                response,
                linkTo(
                    methodOn(PedidoController.class)
                        .buscarPorId(pedido.getId())
                ).withSelfRel()
        );
    }
}
```

Mas ainda precisamos adicionar os links de paginação.

# 15. Controller

```java
@RestController
@RequestMapping("/pedidos")
public class PedidoController {

    private final BuscarPedidosUseCase useCase;
    private final PedidoPresenter presenter;

    public PedidoController(
            BuscarPedidosUseCase useCase,
            PedidoPresenter presenter) {

        this.useCase = useCase;
        this.presenter = presenter;
    }

    @GetMapping
    public PagedModel<EntityModel<PedidoResponse>> buscar(
            @RequestParam(defaultValue = "0")
            int page,

            @RequestParam(defaultValue = "10")
            int size) {

        BuscarPedidosQuery query =
                new BuscarPedidosQuery(page, size);

        BuscarPedidosResult result =
                useCase.executar(query);

        return presenter.present(result);
    }

    @GetMapping("/{id}")
    public EntityModel<PedidoResponse> buscarPorId(
            @PathVariable Long id) {

        // implementação
        return null;
    }
}
```

# 16. HATEOAS da paginação

Para a resposta ficar realmente interessante, podemos gerar:

```text
self
first
last
prev
next
```

Por exemplo:

```json
{
  "_embedded": {
    "pedidos": [
      {
        "id": 1,
        "valor": 150.00,
        "status": "APROVADO",
        "excedeuLimite": false,
        "_links": {
          "self": {
            "href": "/pedidos/1"
          }
        }
      },
      {
        "id": 2,
        "valor": 350.00,
        "status": "PENDENTE",
        "excedeuLimite": true,
        "_links": {
          "self": {
            "href": "/pedidos/2"
          }
        }
      }
    ]
  },
  "page": {
    "size": 10,
    "totalElements": 57,
    "totalPages": 6,
    "number": 0
  },
  "_links": {
    "self": {
      "href": "/pedidos?page=0&size=10"
    },
    "first": {
      "href": "/pedidos?page=0&size=10"
    },
    "next": {
      "href": "/pedidos?page=1&size=10"
    },
    "last": {
      "href": "/pedidos?page=5&size=10"
    }
  }
}
```

E na página 2:

```json
"_links": {
  "self": {
    "href": "/pedidos?page=2&size=10"
  },
  "first": {
    "href": "/pedidos?page=0&size=10"
  },
  "prev": {
    "href": "/pedidos?page=1&size=10"
  },
  "next": {
    "href": "/pedidos?page=3&size=10"
  },
  "last": {
    "href": "/pedidos?page=5&size=10"
  }
}
```

# 17. A arquitetura completa

O que eu acho particularmente interessante nesse desenho é que **HATEOAS fica completamente fora do Application e do Domain**:

```text
┌──────────────────────────────────────────────┐
│              INFRASTRUCTURE                  │
│                                              │
│  Controller                                  │
│      ↓                                       │
│  Request                                     │
│      ↓                                       │
│  Presenter ← HATEOAS                         │
│                                              │
└───────────────────┬──────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────┐
│                 APPLICATION                  │
│                                              │
│  Query                                       │
│      ↓                                       │
│  Input Port                                  │
│      ↓                                       │
│  Use Case                                    │
│      ↓                                       │
│  Output Port                                 │
│                                              │
└───────────────────┬──────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────┐
│                   DOMAIN                     │
│                                              │
│  Pedido                                      │
│  Regras de negócio                           │
│                                              │
└──────────────────────────────────────────────┘
                    ▲
                    │
┌───────────────────┴──────────────────────────┐
│              INFRASTRUCTURE                  │
│                                              │
│  Oracle Adapter                              │
│      ↓                                       │
│  JPA                                         │
│      ↓                                       │
│  Oracle                                      │
│                                              │
└──────────────────────────────────────────────┘
```

### O ponto-chave

Você consegue trocar:

```text
HATEOAS → JSON normal
REST → GraphQL
Oracle → PostgreSQL
Spring Data → outro mecanismo
```

sem colocar essas preocupações dentro do `Domain`.

E para esse cenário, eu usaria a seguinte regra:

**Request/Response/Presenter = Infrastructure**  
**Command/Query/UseCase/Ports = Application**  
**Entity/Value Object/Domain Service = Domain**  
**JPA/Oracle/REST client = Infrastructure**

Essa separação fica bastante consistente com o modelo hexagonal que você está construindo.

---

## Usuário

Query fica em application?

---

## Assistente

Sim. **Se você está usando `Query` como objeto de entrada de um caso de uso, ela fica na `Application`.**

Por exemplo:

```text
application
├── input
│   ├── BuscarPedidosUseCase.java
│   └── BuscarPedidosQuery.java
│
├── output
│   └── PedidoRepository.java
│
└── usecase
    └── BuscarPedidosUseCaseImpl.java
```

### Fluxo

```text
HTTP
 ↓
Controller
 ↓
Request DTO                 ← Infrastructure
 ↓
Query                       ← Application
 ↓
Input Port                  ← Application
 ↓
Use Case                    ← Application
 ↓
Output Port                 ← Application
 ↓
Adapter                     ← Infrastructure
 ↓
Oracle
```

Por exemplo:

```java
public record BuscarPedidosQuery(
    int page,
    int size,
    String status
) {}
```

E:

```java
public interface BuscarPedidosUseCase {

    BuscarPedidosResult executar(
        BuscarPedidosQuery query
    );
}
```

### Query não é a mesma coisa que o SQL

Isto:

```java
public record BuscarPedidosQuery(
    int page,
    int size
) {}
```

significa:

> "Quero buscar pedidos, nesta página e neste tamanho."

Já isto:

```sql
SELECT *
FROM PEDIDO
OFFSET ...
FETCH NEXT ...
```

é responsabilidade da **Infrastructure/Output Adapter**.

Portanto:

```text
Query
  ↓
Application
  ↓
Output Port
  ↓
Oracle Adapter
  ↓
SQL
```

Isso mantém a Application independente do Oracle e do mecanismo de persistência.

---

## Usuário

Uma request vira query?

---

## Assistente

Sim — **em uma arquitetura bem separada, é comum o `Request` virar uma `Query`**, mas eles **não são a mesma coisa**.

A ideia é:

```text
HTTP Request
      ↓
Request DTO
      ↓
Mapper
      ↓
Query
      ↓
Use Case
```

### Exemplo

A API recebe:

```http
GET /pedidos?page=2&size=20&status=APROVADO
```

O `Request` pertence à infraestrutura:

```java
public record BuscarPedidosRequest(
    int page,
    int size,
    String status
) {}
```

O Controller transforma em uma `Query` da Application:

```java
@GetMapping
public ResponseEntity<?> buscar(
        BuscarPedidosRequest request) {

    BuscarPedidosQuery query =
        new BuscarPedidosQuery(
            request.page(),
            request.size(),
            request.status()
        );

    var result = useCase.executar(query);

    return ResponseEntity.ok(
        presenter.present(result)
    );
}
```

A Query:

```java
public record BuscarPedidosQuery(
    int page,
    int size,
    String status
) {}
```

E o Use Case:

```java
public interface BuscarPedidosUseCase {

    BuscarPedidosResult executar(
        BuscarPedidosQuery query
    );
}
```

### Por que separar?

Porque:

**Request** representa o contrato da API:

```text
HTTP
JSON
query parameters
headers
path variables
```

Enquanto:

**Query** representa a intenção da Application:

```text
"buscar pedidos com esses critérios"
```

Assim, amanhã você pode chamar o mesmo Use Case através de Kafka:

```text
Kafka
   ↓
KafkaMessage
   ↓
BuscarPedidosQuery
   ↓
UseCase
```

sem precisar criar uma Query baseada em HTTP.

---

### Mas para um GET extremamente simples...

Você **não precisa obrigatoriamente** ter os dois.

Por exemplo:

```http
GET /pedidos/10
```

Pode ser perfeitamente aceitável:

```java
@GetMapping("/{id}")
public ResponseEntity<?> buscar(
    @PathVariable Long id
) {
    return ...
}
```

e transformar diretamente:

```text
PathVariable
   ↓
GetPedidoQuery
   ↓
UseCase
```

Não precisa criar:

```text
PathVariable
   ↓
Request
   ↓
Mapper
   ↓
Query
```

só para adicionar camadas.

### Regra prática

```text
Request
   = contrato da entrada externa

Command/Query
   = intenção da Application

Domain
   = significado e regras do negócio
```

Então **sim: normalmente Request → Query**, mas não é uma transformação obrigatória em todos os casos.

---

## Usuário

Precisa de mapper em todas as camadas ?

---

## Assistente

Não. **Mapper em todas as camadas seria exagero.** O ideal é criar mapper nas **fronteiras onde os modelos realmente precisam ser diferentes**.

Para a arquitetura que estamos montando:

```text
Infrastructure
     ↓
Application
     ↓
Domain
     ↑
Infrastructure
```

eu pensaria assim:

| Fronteira | Mapper? | Motivo |
|---|---|---|
| Request → Query/Command | **Geralmente sim** | HTTP ≠ Application |
| Query → Use Case | **Não** | Query já é o contrato do Use Case |
| Application → Domain | **Depende** | Pode usar o Domain diretamente |
| Domain → Output Port | **Depende** | Pode passar o próprio objeto |
| Entity → Domain | **Sim, geralmente** | JPA ≠ Domain |
| Domain → Response | **Sim, geralmente** | Domain ≠ API |
| Response → JSON | **Não** | Spring/Jackson faz isso |

### No seu exemplo

Eu faria:

```text
HTTP
 ↓
PedidoRequest
 ↓ mapper
BuscarPedidosQuery
 ↓
UseCase
 ↓
PedidoRepository
 ↓
OracleAdapter
 ↓ mapper
PedidoEntity → Pedido
 ↓
UseCase
 ↓ mapper
Pedido → PedidoResponse
 ↓
Presenter
 ↓
JSON + HATEOAS
```

Mas perceba que **não existe mapper entre absolutamente tudo**.

---

### Exemplo de três fronteiras

**1. Entrada**

```java
public record BuscarPedidosRequest(
    int page,
    int size
) {}
```

Mapper:

```java
public BuscarPedidosQuery toQuery(
    BuscarPedidosRequest request) {

    return new BuscarPedidosQuery(
        request.page(),
        request.size()
    );
}
```

---

**2. Persistência**

```text
PedidoEntity
      ↓
   Mapper
      ↓
Pedido
```

Porque:

```java
@Entity
class PedidoEntity {
    // detalhes JPA
}
```

é diferente de:

```java
class Pedido {
    // regras de domínio
}
```

---

**3. Resposta**

```text
Pedido
   ↓
Presenter / Mapper
   ↓
PedidoResponse
```

Porque não queremos devolver o Domain diretamente para a API.

---

## Uma regra muito boa

Não pense:

> "Cada camada precisa ter seu próprio DTO."

Pense:

> **"Cada fronteira precisa proteger um contrato que pode evoluir independentemente?"**

Se sim → **separe os modelos e faça o mapper.**

Se não → **não crie a conversão artificialmente.**

Por exemplo, isso seria exagerado:

```text
Request
 ↓ Mapper
Command
 ↓ Mapper
ApplicationDTO
 ↓ Mapper
DomainDTO
 ↓ Mapper
Domain
```

Você provavelmente criou **mais complexidade que proteção**.

Para o seu projeto, eu buscaria algo mais próximo de:

```text
Request
   ↓ mapper
Command / Query
   ↓
UseCase
   ↓
Domain
   ↓
Output Port
   ↓
Adapter
   ↓ mapper
Entity

Domain
   ↓ mapper/presenter
Response
```

Essa estrutura mantém as fronteiras importantes sem cair em **"DTO hell"**.

---

## Usuário

Tá ficando muito complexo, quero simplificar


Contrato:
Endpoint para busca Employee por  matrícula

Em infra, vai ter um controller no input
No application input interface para entrada implementada pelo usecase
Em usecase vou chamar o service nesse service implemento output do aplication
Faço algumas regras de negocio antes de chamar o adapter para o repositório.

Voltando para a infra, no output faço a busca no repository
, vai ter um mapper, de data para model.
Respondo o domínio com o model, no service faço outras regras e devolvo para o caso de uso, que tem um maper para model para dto e mando para input do infra, lá vai transformar dto em preseter do contrato.

Simplifique

---

## Assistente

Sim. Dá para simplificar bastante sem perder a essência da arquitetura hexagonal.

Para o seu caso — **buscar Employee por matrícula** — eu faria **apenas 4 responsabilidades principais**:

```text
INFRA
  Controller
      ↓
APPLICATION
  Use Case
      ↓
DOMAIN
  Employee / regras
      ↓
APPLICATION
  Output Port
      ↓
INFRA
  Repository Adapter
      ↓
  Banco
```

## Estrutura

```text
employee/
│
├── domain/
│   └── Employee.java
│
├── application/
│   ├── EmployeeUseCase.java        ← input port
│   ├── EmployeeRepository.java     ← output port
│   └── EmployeeService.java        ← use case
│
└── infrastructure/
    ├── input/
    │   └── EmployeeController.java
    │
    └── output/
        └── EmployeeRepositoryAdapter.java
```

### 1. Controller

Só recebe HTTP e chama o Use Case.

```java
@RestController
@RequestMapping("/employees")
public class EmployeeController {

    private final EmployeeUseCase useCase;

    @GetMapping("/{matricula}")
    public EmployeeResponse buscar(
            @PathVariable String matricula) {

        Employee employee =
                useCase.buscar(matricula);

        return EmployeeResponse.from(employee);
    }
}
```

### 2. Input Port

```java
public interface EmployeeUseCase {

    Employee buscar(String matricula);
}
```

### 3. Use Case / Application Service

Aqui você coordena:

```java
@Service
public class EmployeeService
        implements EmployeeUseCase {

    private final EmployeeRepository repository;

    public EmployeeService(
            EmployeeRepository repository) {

        this.repository = repository;
    }

    @Override
    public Employee buscar(String matricula) {

        Employee employee =
                repository.buscar(matricula);

        validar(employee);

        return employee;
    }

    private void validar(Employee employee) {

        if (!employee.isAtivo()) {
            throw new EmployeeInativoException();
        }
    }
}
```

Aqui está a simplificação importante:

**Você não precisa obrigatoriamente ter:**

```text
UseCase
   ↓
Domain Service
```

Se a regra for pequena, o próprio **Application Service/Use Case** pode executá-la.

---

# 4. Output Port

```java
public interface EmployeeRepository {

    Employee buscar(String matricula);
}
```

### 5. Adapter

Na Infrastructure:

```java
@Repository
public class EmployeeRepositoryAdapter
        implements EmployeeRepository {

    private final EmployeeJpaRepository repository;

    @Override
    public Employee buscar(String matricula) {

        EmployeeEntity entity =
                repository.findByMatricula(matricula)
                        .orElseThrow(
                            EmployeeNotFoundException::new
                        );

        return toDomain(entity);
    }

    private Employee toDomain(
            EmployeeEntity entity) {

        return new Employee(
                entity.getMatricula(),
                entity.getNome(),
                entity.isAtivo()
        );
    }
}
```

Aqui você mantém **um único mapper**, porque realmente existe uma fronteira importante:

```text
EmployeeEntity
      ↓
Employee
```

---

# 6. Domain

```java
public class Employee {

    private final String matricula;
    private final String nome;
    private final boolean ativo;

    public Employee(
            String matricula,
            String nome,
            boolean ativo) {

        this.matricula = matricula;
        this.nome = nome;
        this.ativo = ativo;
    }

    public boolean isAtivo() {
        return ativo;
    }

    public String getMatricula() {
        return matricula;
    }

    public String getNome() {
        return nome;
    }
}
```

---

# 7. Response

Você pode ter um DTO de resposta:

```java
public record EmployeeResponse(
        String matricula,
        String nome,
        boolean ativo) {

    public static EmployeeResponse from(
            Employee employee) {

        return new EmployeeResponse(
                employee.getMatricula(),
                employee.getNome(),
                employee.isAtivo()
        );
    }
}
```

E pronto.

Não precisa:

```text
Employee
 ↓
EmployeeDomainDTO
 ↓
EmployeeResult
 ↓
EmployeeResponse
 ↓
Presenter
```

---

# Fluxo final simplificado

```text
                  INFRA
                    │
              EmployeeController
                    │
                    │ matricula
                    ▼
              APPLICATION
                    │
              EmployeeUseCase
                    │
                    ▼
             EmployeeService
                    │
             ┌──────┴──────┐
             │             │
             │       regra de negócio
             │
             ▼
          Output Port
       EmployeeRepository
             │
             ▼
          INFRASTRUCTURE
             │
       Repository Adapter
             │
             ▼
           Database
             │
             ▼
       EmployeeEntity
             │
           mapper
             │
             ▼
          Employee
             │
             ▼
       EmployeeService
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

## E eu faria uma pequena alteração no seu desenho

Você falou:

> "No usecase vou chamar o service nesse service implemento output do application"

Eu evitaria isso.

Se `EmployeeService` é **o caso de uso**, faça:

```text
Controller
    ↓
EmployeeUseCase       ← interface
    ↑
EmployeeService       ← implementação
```

E dentro dele:

```text
EmployeeService
    ↓
EmployeeRepository    ← output port
    ↓
Adapter
```

Ou seja, **não precisa ter `UseCase → Service → Output` se o Service já é a implementação do Use Case.**

### Minha versão para o seu projeto

```text
employee
│
├── domain
│   └── Employee
│
├── application
│   ├── input
│   │   └── EmployeeUseCase
│   │
│   ├── output
│   │   └── EmployeeRepository
│   │
│   └── EmployeeService
│
└── infrastructure
    ├── input
    │   └── EmployeeController
    │
    └── output
        └── EmployeeRepositoryAdapter
```

**Só 1 mapper obrigatório de verdade:** `Entity → Domain`.

E, se o `EmployeeResponse` tiver exatamente os mesmos dados do `Employee`, você pode até simplificar mais e deixar o Controller retornar um DTO criado no próprio Service.

A regra que eu usaria daqui para frente é:

> **Não crie uma classe porque a arquitetura permite. Crie quando ela tiver uma responsabilidade real.**

Isso deixa a Hexagonal muito mais prática, especialmente para APIs CRUD/consulta.

---

## Usuário

Quero um input interface e use case que implementa ela.

---

## Assistente

Perfeito. Nesse caso, mantenha **Input Port (interface) + Use Case (implementação)** separados. E não precisa adicionar um `Service` entre eles se o próprio Use Case contém a orquestração.

Para seu exemplo de **buscar Employee por matrícula**, eu faria:

```text
employee/
│
├── domain/
│   └── Employee.java
│
├── application/
│   ├── input/
│   │   └── EmployeeInput.java
│   │
│   ├── output/
│   │   └── EmployeeOutput.java
│   │
│   └── usecase/
│       └── BuscarEmployeeUseCase.java
│
└── infrastructure/
    ├── input/
    │   └── EmployeeController.java
    │
    └── output/
        └── EmployeeRepositoryAdapter.java
```

### Input Port

```java
public interface EmployeeInput {

    Employee buscarPorMatricula(String matricula);
}
```

### Use Case implementa o Input

```java
@Service
public class BuscarEmployeeUseCase
        implements EmployeeInput {

    private final EmployeeOutput output;

    public BuscarEmployeeUseCase(
            EmployeeOutput output) {
        this.output = output;
    }

    @Override
    public Employee buscarPorMatricula(
            String matricula) {

        Employee employee =
                output.buscarPorMatricula(matricula);

        // Regras de negócio
        if (!employee.isAtivo()) {
            throw new EmployeeInativoException();
        }

        return employee;
    }
}
```

### Output Port

```java
public interface EmployeeOutput {

    Employee buscarPorMatricula(String matricula);
}
```

### Adapter

```java
@Repository
public class EmployeeRepositoryAdapter
        implements EmployeeOutput {

    private final EmployeeJpaRepository repository;

    @Override
    public Employee buscarPorMatricula(
            String matricula) {

        EmployeeEntity entity =
                repository.findByMatricula(matricula)
                        .orElseThrow(
                            EmployeeNotFoundException::new
                        );

        return new Employee(
                entity.getMatricula(),
                entity.getNome(),
                entity.isAtivo()
        );
    }
}
```

### Controller

O Controller depende **somente do Input Port**:

```java
@RestController
@RequestMapping("/employees")
public class EmployeeController {

    private final EmployeeInput input;

    public EmployeeController(
            EmployeeInput input) {
        this.input = input;
    }

    @GetMapping("/{matricula}")
    public Employee buscar(
            @PathVariable String matricula) {

        return input.buscarPorMatricula(matricula);
    }
}
```

O desenho fica muito limpo:

```text
                 INFRASTRUCTURE
                      │
                      ▼
               EmployeeController
                      │
                      │ depende
                      ▼
              ┌───────────────┐
              │ EmployeeInput │  ← INPUT PORT
              └───────▲───────┘
                      │ implements
                      │
              ┌───────┴────────────┐
              │ BuscarEmployee     │
              │ UseCase            │
              └───────┬────────────┘
                      │
                      │ depende
                      ▼
              ┌───────────────┐
              │ EmployeeOutput│  ← OUTPUT PORT
              └───────▲───────┘
                      │ implements
                      │
              ┌───────┴────────────┐
              │ RepositoryAdapter  │
              └───────┬────────────┘
                      │
                      ▼
                    Oracle
```

**Essa é uma boa simplificação:** `Input = contrato`, `UseCase = implementação`, `Output = contrato`, `Adapter = implementação`. O domínio fica no centro e não conhece nenhuma dessas tecnologias.

---

## Usuário

Quero ter um service que sempre implementa o output

---

## Assistente

Sim. Se você quer adotar **um padrão fixo em que o `Service` sempre implementa o Output Port**, dá para organizar de forma simples.

Só muda a responsabilidade: o **Use Case orquestra**, e o **Service implementa o contrato de saída** para acessar o domínio/repositório.

### Estrutura

```text
employee/
│
├── domain/
│   └── Employee.java
│
├── application/
│   ├── input/
│   │   └── EmployeeInput.java
│   │
│   ├── output/
│   │   └── EmployeeOutput.java
│   │
│   ├── usecase/
│   │   └── BuscarEmployeeUseCase.java
│   │
│   └── service/
│       └── EmployeeService.java
│
└── infrastructure/
    ├── input/
    │   └── EmployeeController.java
    │
    └── output/
        └── EmployeeRepositoryAdapter.java
```

### 1. Input Port

```java
public interface EmployeeInput {

    Employee buscarPorMatricula(String matricula);
}
```

### 2. Use Case

Implementa o Input:

```java
@Service
public class BuscarEmployeeUseCase
        implements EmployeeInput {

    private final EmployeeOutput output;

    public BuscarEmployeeUseCase(
            EmployeeOutput output) {
        this.output = output;
    }

    @Override
    public Employee buscarPorMatricula(
            String matricula) {

        Employee employee =
                output.buscarPorMatricula(matricula);

        // Regras/orquestração
        if (!employee.isAtivo()) {
            throw new EmployeeInativoException();
        }

        return employee;
    }
}
```

### 3. Output Port

```java
public interface EmployeeOutput {

    Employee buscarPorMatricula(String matricula);
}
```

### 4. Service implementa Output

Aqui está o padrão que você quer:

```java
@Service
public class EmployeeService
        implements EmployeeOutput {

    private final EmployeeRepositoryAdapter repository;

    public EmployeeService(
            EmployeeRepositoryAdapter repository) {
        this.repository = repository;
    }

    @Override
    public Employee buscarPorMatricula(
            String matricula) {

        Employee employee =
                repository.buscarPorMatricula(matricula);

        // Regras específicas do serviço, se houver

        return employee;
    }
}
```

Mas aqui existe um **problema arquitetural**: se `EmployeeService` está na Application e depende diretamente de `EmployeeRepositoryAdapter`, a Application passa a conhecer Infrastructure.

Para manter a Hexagonal correta, faça o Service depender de **outro port**:

```text
UseCase
   ↓
EmployeeOutput
   ↓
Service
   ↓
RepositoryPort
   ↓
Adapter
```

Por exemplo:

```java
public interface EmployeeOutput {
    Employee buscarPorMatricula(String matricula);
}
```

```java
@Service
public class EmployeeService
        implements EmployeeOutput {

    private final EmployeeRepository repository;

    public EmployeeService(
            EmployeeRepository repository) {
        this.repository = repository;
    }

    @Override
    public Employee buscarPorMatricula(
            String matricula) {

        return repository.buscarPorMatricula(matricula);
    }
}
```

E:

```java
public interface EmployeeRepository {

    Employee buscarPorMatricula(String matricula);
}
```

O Adapter implementa esse segundo contrato:

```java
@Repository
public class EmployeeRepositoryAdapter
        implements EmployeeRepository {

    @Override
    public Employee buscarPorMatricula(
            String matricula) {

        // Oracle
        // Entity -> Domain

        return employee;
    }
}
```

### Ficaria assim

```text
                 INFRASTRUCTURE
                      │
                 Controller
                      │
                      ▼
              ┌──────────────┐
              │ EmployeeInput│
              └──────▲───────┘
                     │
                  implements
                     │
              ┌──────┴───────┐
              │   Use Case   │
              └──────┬───────┘
                     │
                     ▼
              ┌──────────────┐
              │EmployeeOutput│
              └──────▲───────┘
                     │
                  implements
                     │
              ┌──────┴───────┐
              │   Service    │
              └──────┬───────┘
                     │
                     ▼
             Repository Port
                     ▲
                     │ implements
                     │
              Repository Adapter
                     │
                     ▼
                   Oracle
```

**Mas eu faria uma observação:** se `Service` sempre implementa `Output`, você está criando uma convenção arquitetural própria. Ela é possível, mas o nome `Output` normalmente representa o **port da aplicação para fora**, enquanto um `Service` que contém regras de negócio pode ter responsabilidade diferente.

Se sua intenção é ter um padrão consistente no projeto, eu simplificaria ainda mais para:

```text
Controller
   ↓
Input Port
   ↓
Use Case
   ↓
Output Port
   ↓
Service
   ↓
Repository Port
   ↓
Adapter
   ↓
Oracle
```

Assim você consegue manter **Input/Output como interfaces**, **Use Case como orquestrador** e **Service como implementação padronizada do Output**, sem misturar Infrastructure dentro da Application.

---

## Usuário

Ficou complexo

---

## Assistente

Sim. Ficou complexo porque estamos tentando colocar **Service + Use Case + Output + Repository Port** para uma busca simples.

Se você quer **simplificar mantendo sua ideia de Input/Output**, eu faria assim:

```text
INFRA
Controller
    ↓
APPLICATION
Input (interface)
    ↓
UseCase (implementa Input)
    ↓
Service (implementa Output)
    ↓
INFRA
Repository Adapter
    ↓
Oracle
```

### Estrutura

```text
employee/
│
├── domain/
│   └── Employee.java
│
├── application/
│   ├── input/
│   │   └── EmployeeInput.java
│   │
│   ├── output/
│   │   └── EmployeeOutput.java
│   │
│   ├── usecase/
│   │   └── BuscarEmployeeUseCase.java
│   │
│   └── service/
│       └── EmployeeService.java
│
└── infrastructure/
    ├── input/
    │   └── EmployeeController.java
    │
    └── output/
        └── EmployeeRepositoryAdapter.java
```

### Input

```java
public interface EmployeeInput {
    Employee buscar(String matricula);
}
```

### Use Case

```java
public class BuscarEmployeeUseCase
        implements EmployeeInput {

    private final EmployeeOutput output;

    public BuscarEmployeeUseCase(EmployeeOutput output) {
        this.output = output;
    }

    @Override
    public Employee buscar(String matricula) {

        Employee employee =
                output.buscar(matricula);

        // regras do caso de uso

        return employee;
    }
}
```

### Output

```java
public interface EmployeeOutput {
    Employee buscar(String matricula);
}
```

### Service

```java
@Service
public class EmployeeService
        implements EmployeeOutput {

    private final EmployeeRepositoryAdapter repository;

    public EmployeeService(
            EmployeeRepositoryAdapter repository) {

        this.repository = repository;
    }

    @Override
    public Employee buscar(String matricula) {

        Employee employee =
                repository.buscar(matricula);

        // regras de domínio específicas do serviço

        return employee;
    }
}
```

### Adapter

```java
@Repository
public class EmployeeRepositoryAdapter {

    public Employee buscar(String matricula) {

        EmployeeEntity entity =
                repository.findByMatricula(matricula);

        return new Employee(
                entity.getMatricula(),
                entity.getNome()
        );
    }
}
```

## Mas tem um detalhe

Essa estrutura deixa:

```text
Application
   ↓
Infrastructure
```

porque o `EmployeeService` conhece diretamente `EmployeeRepositoryAdapter`.

**Isso deixa de ser Hexagonal "pura".**

Se seu objetivo principal é **simplicidade**, eu prefiro aceitar isso conscientemente em um CRUD simples do que criar mais três interfaces apenas para satisfazer a arquitetura.

### Minha recomendação para esse caso

Use:

```text
Controller
    ↓
Input Port
    ↓
Use Case
    ↓
Service
    ↓
Repository Adapter
    ↓
Oracle
```

E somente:

```text
Input → interface
UseCase → implementa Input
Service → regras
Repository Adapter → banco
```

O **Output Port pode até ser eliminado** nesse caso.

Você ganha uma arquitetura muito mais fácil de entender, sem transformar uma busca de Employee em 10 classes.

---

## Usuário

Isso faca o mais simples

---

## Assistente

Sim. Vamos **zerar a complexidade** e manter apenas o que realmente agrega valor.

Para o seu caso:

> `GET /employees/{matricula}` → busca Employee → aplica regra → retorna resposta.

Eu faria **somente isso**:

```text
INFRASTRUCTURE
└── input
    └── EmployeeController
            ↓
APPLICATION
└── input
    └── EmployeeUseCase (interface)
            ↑
    └── usecase
        └── EmployeeUseCaseImpl
            ↓
DOMAIN
└── Employee
            ↑
INFRASTRUCTURE
└── output
    └── EmployeeRepository
```

### 1. Controller

```java
@RestController
@RequestMapping("/employees")
public class EmployeeController {

    private final EmployeeUseCase useCase;

    public EmployeeController(EmployeeUseCase useCase) {
        this.useCase = useCase;
    }

    @GetMapping("/{matricula}")
    public EmployeeResponse buscar(
            @PathVariable String matricula) {

        Employee employee =
                useCase.buscar(matricula);

        return EmployeeResponse.from(employee);
    }
}
```

### 2. Input Port

```java
public interface EmployeeUseCase {

    Employee buscar(String matricula);
}
```

### 3. Use Case

Aqui fica a **orquestração e as regras do caso de uso**:

```java
@Service
public class EmployeeUseCaseImpl
        implements EmployeeUseCase {

    private final EmployeeRepository repository;

    public EmployeeUseCaseImpl(
            EmployeeRepository repository) {

        this.repository = repository;
    }

    @Override
    public Employee buscar(String matricula) {

        Employee employee =
                repository.buscar(matricula);

        if (!employee.isAtivo()) {
            throw new EmployeeInativoException();
        }

        return employee;
    }
}
```

### 4. Output Port

```java
public interface EmployeeRepository {

    Employee buscar(String matricula);
}
```

### 5. Adapter

```java
@Repository
public class EmployeeRepositoryAdapter
        implements EmployeeRepository {

    private final EmployeeJpaRepository repository;

    @Override
    public Employee buscar(String matricula) {

        EmployeeEntity entity =
                repository.findByMatricula(matricula)
                        .orElseThrow(
                            EmployeeNotFoundException::new
                        );

        return new Employee(
                entity.getMatricula(),
                entity.getNome(),
                entity.isAtivo()
        );
    }
}
```

### 6. Domain

```java
public class Employee {

    private final String matricula;
    private final String nome;
    private final boolean ativo;

    public Employee(
            String matricula,
            String nome,
            boolean ativo) {

        this.matricula = matricula;
        this.nome = nome;
        this.ativo = ativo;
    }

    public boolean isAtivo() {
        return ativo;
    }

    public String getMatricula() {
        return matricula;
    }

    public String getNome() {
        return nome;
    }
}
```

### 7. Response

```java
public record EmployeeResponse(
        String matricula,
        String nome,
        boolean ativo) {

    public static EmployeeResponse from(
            Employee employee) {

        return new EmployeeResponse(
                employee.getMatricula(),
                employee.getNome(),
                employee.isAtivo()
        );
    }
}
```

## Resultado

```text
                 INFRA
                   │
              Controller
                   │
                   ▼
            ┌──────────────┐
            │ Employee     │
            │ UseCase      │ ← interface
            └──────▲───────┘
                   │
                implements
                   │
            ┌──────┴───────┐
            │ UseCaseImpl  │
            │              │
            │ regra        │
            └──────┬───────┘
                   │
                   ▼
            ┌──────────────┐
            │ Repository   │ ← interface
            └──────▲───────┘
                   │
                implements
                   │
            ┌──────┴────────┐
            │ Repository    │
            │ Adapter       │
            └──────┬────────┘
                   │
                   ▼
                 Oracle
```

**Só duas interfaces:**

```text
EmployeeUseCase
EmployeeRepository
```

**Só dois adapters/implementações principais:**

```text
EmployeeUseCaseImpl
EmployeeRepositoryAdapter
```

E **nenhum Service separado**.

Para uma busca simples, eu considero essa uma ótima base: **Hexagonal sem transformar arquitetura em burocracia.**

---

## Usuário

Mais quebra o input e output

---

## Assistente

Sim. Se você quer **manter explicitamente `input` e `output`**, mas ainda deixar o mínimo possível, eu faria assim:

```text
employee/
│
├── domain/
│   └── Employee.java
│
├── application/
│   ├── input/
│   │   └── EmployeeUseCase.java
│   │
│   └── output/
│       └── EmployeeRepository.java
│
└── infrastructure/
    ├── input/
    │   └── EmployeeController.java
    │
    └── output/
        └── EmployeeRepositoryAdapter.java
```

E o **Use Case é a implementação do Input**, sem criar outro pacote/classe desnecessário:

```java
@Service
public class EmployeeUseCase implements EmployeeUseCaseInput {

    private final EmployeeRepository repository;

    @Override
    public Employee buscar(String matricula) {

        Employee employee =
            repository.buscar(matricula);

        // regra de negócio

        return employee;
    }
}
```

Eu só ajustaria o nome para evitar conflito:

```text
application
├── input
│   └── EmployeeInput.java
│
└── output
    └── EmployeeOutput.java

EmployeeUseCase implements EmployeeInput
EmployeeRepositoryAdapter implements EmployeeOutput
```

Fica:

```text
                 INFRASTRUCTURE
                       │
                 Controller
                       │
                       ▼
              APPLICATION INPUT
              EmployeeInput
                       ▲
                       │ implements
                       │
                 EmployeeUseCase
                       │
                       │
                       ▼
              APPLICATION OUTPUT
              EmployeeOutput
                       ▲
                       │ implements
                       │
             RepositoryAdapter
                       │
                       ▼
                    Oracle
```

### Código mínimo

**Input:**

```java
public interface EmployeeInput {
    Employee buscar(String matricula);
}
```

**Use Case:**

```java
@Service
public class EmployeeUseCase
        implements EmployeeInput {

    private final EmployeeOutput output;

    public EmployeeUseCase(EmployeeOutput output) {
        this.output = output;
    }

    @Override
    public Employee buscar(String matricula) {

        Employee employee =
                output.buscar(matricula);

        // regras de negócio

        return employee;
    }
}
```

**Output:**

```java
public interface EmployeeOutput {
    Employee buscar(String matricula);
}
```

**Adapter:**

```java
@Repository
public class EmployeeRepositoryAdapter
        implements EmployeeOutput {

    @Override
    public Employee buscar(String matricula) {

        EmployeeEntity entity =
                repository.findByMatricula(matricula)
                          .orElseThrow();

        return new Employee(
                entity.getMatricula(),
                entity.getNome(),
                entity.isAtivo()
        );
    }
}
```

Então sua arquitetura fica **bem enxuta**:

```text
INFRA INPUT
Controller
    ↓
APP INPUT
EmployeeInput
    ↑
EmployeeUseCase
    ↓
APP OUTPUT
EmployeeOutput
    ↑
INFRA OUTPUT
RepositoryAdapter
    ↓
Oracle
```

E o mais importante: **não precisa de Service separado, Query, Request, Mapper entre todas as camadas ou Presenter**, a menos que surja uma necessidade real.
