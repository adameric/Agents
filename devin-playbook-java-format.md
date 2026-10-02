# Playbook — JAVA Professional Semantic Formatter

## Identificação

**Nome:** JAVA Professional Semantic Formatter  
**Tipo:** Playbook de formatação de código  
**Objetivo:** Formatar código Java sem alterar lógica, estrutura, semântica ou comportamento.

---

# 1. Objetivo

Formatar código Java existente para um padrão profissional, com foco em **legibilidade semântica**.

A formatação deve permitir que o desenvolvedor consiga compreender visualmente o fluxo já existente no método:

```text
ENTRADA
  ↓
VALIDAÇÃO
  ↓
PREPARAÇÃO
  ↓
OBTENÇÃO / CONSULTA
  ↓
TRANSFORMAÇÃO
  ↓
PROCESSAMENTO
  ↓
EFEITO COLATERAL / PERSISTÊNCIA
  ↓
RETORNO
```

Essas etapas não podem ser criadas, reorganizadas ou inferidas como novas estruturas de código.

Elas somente devem ser expressas visualmente quando já estiverem presentes no código.

---

# 2. Regra Absoluta

> **NÃO ALTERE O CÓDIGO. ALTERE SOMENTE A FORMATAÇÃO.**

A tarefa é exclusivamente de apresentação.

---

# 3. Quando aplicar

Aplicar este Playbook quando o objetivo for:

- formatar código Java
- padronizar estilo Java
- melhorar a legibilidade visual
- organizar indentação
- organizar quebras de linha
- organizar linhas em branco
- melhorar a densidade visual
- tornar o fluxo de métodos mais evidente

Não aplicar para:

- refatoração
- correção de bugs
- otimização
- modernização
- migração de Java
- alteração arquitetural
- revisão de código
- alteração de comportamento

---

# 4. Resultado Esperado

O resultado deve:

- ser código Java válido
- preservar integralmente o código original
- usar 2 espaços de indentação
- apresentar hierarquia visual clara
- agrupar conceitos semanticamente relacionados
- utilizar linhas em branco apenas quando houver mudança de contexto
- manter streams visualmente contínuos
- manter lambdas proporcionais à sua complexidade
- evitar verticalização excessiva
- evitar linhas excessivamente longas quando houver uma quebra natural
- evitar quebras desnecessárias
- parecer código Java profissional de produção

---

# 5. Hierarquia de Decisão

Quando duas regras entrarem em conflito, utilizar esta ordem:

1. **Preservação do código**
2. **Legibilidade semântica**
3. **Hierarquia visual**
4. **Agrupamento semântico**
5. **Unidade visual mínima**
6. **Densidade natural**
7. **Consistência local**
8. **Comprimento da linha**
9. **Estética**

Uma regra de maior prioridade sempre vence uma regra de menor prioridade.

Exemplo:

> Legibilidade vence limite de coluna.

---

# 6. Preservação Absoluta

Nunca alterar:

- lógica
- comportamento
- estrutura
- ordem de execução
- condições
- operadores
- chamadas de métodos
- argumentos
- variáveis
- nomes
- tipos
- generics
- annotations
- literals
- strings
- regex
- imports
- package
- records
- enums
- lambdas
- streams
- exceções
- retornos
- comentários
- Javadoc

Nunca:

- adicionar código
- remover código
- criar variáveis
- remover variáveis
- criar métodos
- remover métodos
- renomear elementos
- reordenar elementos
- refatorar
- simplificar
- otimizar
- modernizar

---

# 7. Indentação

Usar:

> **2 espaços por nível.**

Nunca utilizar TAB para indentação.

Exemplo:

```java
public void process(Order order) {
  if (order.isValid()) {
    processOrder(order);
  }
}
```

A indentação deve representar a hierarquia sintática real.

---

# 8. Comprimento de Linha

Utilizar aproximadamente **90 caracteres** como referência.

90 caracteres é uma meta, não uma regra absoluta.

Regras:

- preferir aproximadamente 90 caracteres
- evitar linhas acima de aproximadamente 120 quando houver quebra natural
- não quebrar uma expressão simples apenas para atingir 90
- não criar quebras artificiais
- legibilidade sempre vence comprimento

Regra:

> **Não quebre por número. Quebre por estrutura.**

---

# 9. Unidade Visual Mínima

Constructos simples devem permanecer juntos.

Exemplo preferido:

```java
return new ApprovalDecision(false, BigDecimal.ZERO, RiskLevel.REJECTED, refusalReasons);
```

Quando necessário:

```java
return new ApprovalDecision(
    false,
    BigDecimal.ZERO,
    RiskLevel.REJECTED,
    refusalReasons);
```

Evitar:

```java
return new ApprovalDecision(
    false,
    BigDecimal.ZERO,
    RiskLevel.REJECTED,
    refusalReasons
);
```

Não colocar delimitadores de fechamento isolados sem necessidade estrutural.

---

# 10. Linhas em Branco

Regra fundamental:

> **Linha em branco separa conceitos, não instruções.**

Adicionar uma linha em branco quando existir uma mudança real de contexto semântico.

Exemplo:

```java
validate(request);

User user = findUser(request);
Context context = createContext(user);

List<Item> items = loadItems(context);
List<Item> validItems = filterValidItems(items);

Result result = processItems(validItems, context);

save(result);

return result;
```

Não adicionar linha em branco automaticamente após:

- `if`
- `for`
- `while`
- `switch`
- declaração
- chamada de método
- lambda
- operação de stream

---

# 11. Agrupamento de Ifs

Ifs relacionados devem permanecer agrupados.

Exemplo:

```java
if (conditionA) {
  processA();
}
if (conditionB) {
  processB();
}
if (conditionC) {
  processC();
}
```

Não inserir linhas em branco entre eles se fizerem parte da mesma etapa lógica.

---

# 12. Ifs Aninhados

Preservar exatamente o aninhamento existente.

Exemplo:

```java
if (applicant != null) {
  if (applicant.creditScore() > 300) {
    if (!applicant.isPoliticallyExposed()) {
      if (applicant.monthlyIncome() != null
          && applicant.monthlyIncome().compareTo(new BigDecimal("1500")) >= 0) {
        process(applicant);
      }
    }
  }
}
```

Nunca transformar em:

```java
if (applicant == null) {
  return;
}
```

Não criar guard clauses.

Não inverter condições.

Não reduzir níveis de aninhamento.

---

# 13. Streams

Streams são **uma unidade visual única**.

Não adicionar linhas em branco entre operações.

Exemplo:

```java
List<Customer> activeCustomers =
    customers.stream()
        .filter(Customer::isActive)
        .filter(customer -> customer.balance().compareTo(BigDecimal.ZERO) > 0)
        .map(this::enrich)
        .sorted(Comparator.comparing(Customer::name))
        .toList();
```

Preservar:

- ordem
- operações
- lambdas
- method references
- terminal operation

---

# 14. Lambdas

## Lambda simples

Manter compacta:

```java
.map(Customer::name)
.filter(customer -> customer.isActive())
```

## Lambda complexa

Quebrar proporcionalmente:

```java
.map(
    customer ->
        customer.isActive()
            ? calculateActiveValue(customer)
            : calculateInactiveValue(customer))
```

## Lambda com fluxo de controle

Tratar visualmente como um pequeno bloco:

```java
.filter(
    applicant -> {
      if (applicant.creditScore() < 500) {
        if (applicant.isPoliticallyExposed()) {
          return true;
        } else {
          return applicant.cards().stream()
              .anyMatch(c -> c.balance().compareTo(debtThreshold) > 0);
        }
      } else if (applicant.loans().stream().anyMatch(LoanHistory::defaulted)) {
        return true;
      } else {
        return false;
      }
    })
```

Nunca refatorar a lambda.

---

# 15. Chamadas de Métodos

Chamadas simples permanecem compactas:

```java
calculateRisk(customer);
```

Chamadas complexas podem ser quebradas:

```java
calculateRisk(
    customer,
    configuration,
    historicalData,
    requestedAmount);
```

Não quebrar mais do que o necessário.

---

# 16. Encadeamento

Manter chamadas encadeadas visualmente conectadas:

```java
repository.findByCustomerId(customerId)
    .filter(Customer::isActive)
    .map(this::toResponse)
    .orElseThrow();
```

Evitar fragmentação excessiva.

---

# 17. Ternários

Nunca converter:

```text
if → ternário
```

ou:

```text
ternário → if
```

Ternário simples:

```java
String status = active ? "ACTIVE" : "INACTIVE";
```

Ternário complexo:

```java
String status =
    active
        ? "ACTIVE"
        : "INACTIVE";
```

---

# 18. Ternários Aninhados

Preservar a estrutura.

Exemplo:

```java
.collect(
    Collectors.groupingBy(
        a ->
            a.creditScore() >= 750
                ? RiskLevel.LOW
                : (a.creditScore() >= 600
                    ? RiskLevel.MEDIUM
                    : RiskLevel.HIGH)))
```

Não transformar em:

- `if`
- `switch`
- variável auxiliar
- método auxiliar
- outra expressão

---

# 19. Expressões Booleanas

Quebrar somente quando isso melhorar a leitura:

```java
if (customer != null
    && customer.isActive()
    && customer.balance().compareTo(BigDecimal.ZERO) > 0) {
  process(customer);
}
```

Preservar integralmente:

- condições
- ordem
- operadores
- parênteses
- negações
- precedência

---

# 20. Switch

Preservar a forma original.

Não converter:

- switch clássico ↔ switch expression
- cases em métodos
- cases em mapas
- cases em polimorfismo

Somente formatar.

---

# 21. Generics

Manter estruturas simples compactas:

```java
List<String> names;
```

Quebrar estruturas complexas apenas quando necessário.

Nunca alterar tipos genéricos.

---

# 22. Records e Enums

Preservar:

- ordem
- nomes
- componentes
- constantes
- campos
- métodos
- construtores
- annotations

Apenas formatar.

---

# 23. Annotations

Preservar exatamente.

Exemplo:

```java
@Override
@Transactional
public Result process(Request request) {
  ...
}
```

Não adicionar, remover ou reordenar.

---

# 24. Comentários e Javadoc

Preservar conteúdo.

Não:

- reescrever
- corrigir
- resumir
- remover
- adicionar
- reinterpretar

Somente ajustar apresentação quando necessário.

---

# 25. Imports

Não:

- adicionar
- remover
- ordenar
- agrupar
- otimizar

Preservar os imports originais.

---

# 26. Densidade Profissional

Evitar dois extremos.

### Compressão excessiva

```java
if(a!=null){if(a.isValid()){process(a);}}
```

### Fragmentação excessiva

```java
if (
    a != null
) {
  if (
      a.isValid()
  ) {
    process(
        a
    );
  }
}
```

Preferir o meio-termo profissional:

```java
if (a != null) {
  if (a.isValid()) {
    process(a);
  }
}
```

---

# 27. Princípio da Quebra Mínima

> **Quebre somente o necessário para revelar a estrutura.**

Não quebrar:

- por simetria
- por estética
- apenas porque é possível
- apenas para atingir uma quantidade de colunas

Não compactar:

- apenas por simetria
- apenas porque cabe
- quando a estrutura ficar difícil de entender

---

# 28. Métodos Complexos

Identificar visualmente as etapas que já existem.

Exemplo:

```java
public Result process(Request request) {
  validate(request);

  Input input = loadInput(request);
  Context context = createContext(input);

  List<Item> items = loadItems(context);
  List<Item> validItems = filterValidItems(items);

  Result result = processItems(validItems, context);

  persist(result);

  return result;
}
```

Importante:

> Não criar variáveis ou etapas que não existam no código original.

---

# 29. Critério de Legibilidade

Antes de finalizar, verificar:

- As etapas principais são visíveis?
- A entrada e saída são fáceis de localizar?
- A validação é visualmente identificável?
- O fluxo de dados é fácil de acompanhar?
- As transformações estão agrupadas?
- O processamento está separado dos efeitos colaterais?
- Streams permanecem contínuos?
- Lambdas complexas estão claras?
- Ifs aninhados estão claros?
- Linhas em branco possuem significado?
- O código não está excessivamente comprimido?
- O código não está excessivamente fragmentado?

---

# 30. Critério de Preservação

Antes de finalizar, verificar:

```text
Mesmas instruções
Mesmas expressões
Mesmas condições
Mesmos operadores
Mesmas chamadas
Mesmas variáveis
Mesmos tipos
Mesmos generics
Mesmas annotations
Mesmos literais
Mesmas strings
Mesmos comentários
Mesmos imports
Mesma ordem
Mesmo aninhamento
Mesmo comportamento
```

Se qualquer item tiver sido alterado:

> desfazer a alteração e retornar somente à formatação.

---

# 31. Fluxo de Execução do Playbook

Executar nesta ordem:

## Etapa 1 — Ler

Ler todo o código antes de formatar.

## Etapa 2 — Preservar

Identificar a estrutura original.

## Etapa 3 — Classificar

Identificar:

- métodos
- declarações
- condicionais
- loops
- streams
- lambdas
- ternários
- chamadas
- expressões booleanas
- classes
- records
- enums

## Etapa 4 — Agrupar

Identificar unidades semanticamente relacionadas.

## Etapa 5 — Formatar

Aplicar:

- 2 espaços
- K&R
- quebras mínimas
- agrupamento semântico
- densidade profissional
- linhas em branco semânticas

## Etapa 6 — Revisar

Executar o teste de legibilidade.

## Etapa 7 — Validar

Executar o teste de preservação.

## Etapa 8 — Entregar

Retornar somente o código Java formatado.

---

# 32. Regra de Ouro

> **A formatação deve tornar o código existente mais fácil de entender sem alterar o que o código é.**

---

# 33. Princípios Fundamentais

1. **Preserve o código.**
2. **Formate a semântica, nunca altere a semântica.**
3. **Linhas em branco separam conceitos, não instruções.**
4. **Streams são unidades visuais.**
5. **Lambdas complexas são pequenos blocos visuais.**
6. **Constructos simples permanecem compactos.**
7. **Constructos complexos recebem apenas as quebras necessárias.**
8. **2 espaços são o padrão de indentação.**
9. **90 colunas é uma referência, não uma lei.**
10. **Legibilidade vence estética.**
11. **Agrupamento semântico vence simetria.**
12. **Quebras mínimas são preferidas.**
13. **Nunca refatore durante a formatação.**
14. **O código deve contar visualmente a história que já existe nele.**

---

# 34. Comportamento de Saída

A resposta final deve conter:

```text
<CÓDIGO JAVA FORMATADO>
```

Somente isso.

Não escrever:

```text
Aqui está o código:
```

Não escrever:

```text
Código formatado:
```

Não escrever:

```text
Pronto!
```

Não explicar alterações.

Não apresentar diff.

Não apresentar análise.

Não adicionar comentários.

---

# 35. Enforcement Final

Antes de entregar:

```text
FORMATE SOMENTE.
2 ESPAÇOS.
FORMATAÇÃO SEMÂNTICA.
QUEBRAS MÍNIMAS.
PRESERVE O CÓDIGO.
NÃO REFARE.
NÃO ALTERE A LÓGICA.
NÃO ALTERE A ESTRUTURA.
NÃO ALTERE O COMPORTAMENTO.
ALTERE SOMENTE A APRESENTAÇÃO.
RETORNE SOMENTE O CÓDIGO JAVA FORMATADO.
```
