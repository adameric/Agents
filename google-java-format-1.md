# GOOGLE JAVA FORMAT

## Referência Oficial do Google Java Format

Este documento define a referência operacional para utilizar o **google-java-format** como base mecânica de formatação Java do agente `JAVA FORMAT ULTIMATE`.

O projeto oficial descreve o `google-java-format` como uma ferramenta que reformata código-fonte Java para seguir o **Google Java Style**. O formatter deliberadamente não expõe configuração para seu algoritmo de formatação, com o objetivo de manter um único padrão de formatação. citeturn0view0

Repositório oficial:
https://github.com/google/google-java-format

---

## 1. IDENTIDADE

**Ferramenta:** `google-java-format`

**Objetivo:** Formatar código-fonte Java de acordo com o Google Java Style.

**Papel no JAVA FORMAT ULTIMATE:**

```text
GOOGLE JAVA FORMAT
        ↓
Base de formatação mecânica / estrutural
        ↓
JAVA PROFESSIONAL SEMANTIC FORMATTER
        ↓
Agrupamento semântico + quebra mínima + regras de preservação
```

O Google Java Format é a **referência de formatação base**, e não a política semântica completa do `JAVA FORMAT ULTIMATE`.

---

## 2. PRINCÍPIO FUNDAMENTAL

Use o Google Java Format para as convenções mecânicas de formatação que ele já define.

Não tente reproduzir ou substituir seu algoritmo interno de formatação com propriedades arbitrárias do IntelliJ ou do `.editorconfig`.

The official project explicitly states that its formatter algorithm is not configurable. citeturn0view0

---

## 3. INDENTAÇÃO

O Google Java Format padrão utiliza:

```text
2 espaços
```

O projeto oficial diferencia isso da variante AOSP, que utiliza 4 espaços. citeturn0view0

Para o `JAVA FORMAT ULTIMATE`, utilize a convenção padrão do Google Java Format:

```text
indent = 2 espaços
tabs = proibidos
```

---

## 4. GOOGLE JAVA STYLE

O objetivo do formatter é fazer o código-fonte Java seguir o Google Java Style.

Use o Google Java Format como referência mecânica para:

- indentação;
- quebras de linha;
- chaves;
- continuação de expressões;
- expressões;
- chamadas de métodos;
- parâmetros;
- streams;
- lambdas;
- comentários/Javadoc;
- imports;
- outras estruturas de formatação Java.

O agente semântico pode adicionar regras de apresentação, mas não pode alterar a semântica do código Java.

---

## 5. NÃO CRIAR UM ALGORITMO PRÓPRIO DE FORMATTER

Não invente um algoritmo substituto e o chame de Google Java Format.

A distinção é:

```text
Google Java Format
    = Google-defined formatter

JAVA FORMAT ULTIMATE
    = Google Java Format baseline
      + semantic formatting policy
      + preservation policy
      + minimal-breaking policy
```

---

## 6. REFERÊNCIA DE LINHA DE COMANDO

O formatter oficial pode ser executado com:

```bash
java -jar /path/to/google-java-format-VERSION-all-deps.jar <options> [files...]
```

O formatter pode operar sobre:

- arquivos completos;
- linhas selecionadas;
- offsets específicos;
- saída padrão;
- arquivos diretamente no local.

Entre as opções oficiais relevantes estão:

```text
--lines
--offset
--replace
--aosp
--fix-imports-only
--skip-sorting-imports
--skip-removing-unused-import
--skip-reflowing-long-strings
--skip-javadoc-formatting
--dry-run
--set-exit-if-changed
```

The formatter also supports an `@<filename>` argument file. citeturn0view0

---

## 7. VERSÃO DO FORMATTER

Não fixe uma versão do formatter nas regras semânticas, a menos que a implementação exija reprodutibilidade.

Quando for necessária reprodutibilidade:

```text
Fixe explicitamente a versão do google-java-format.
```

A versão instalada passa a fazer parte da cadeia de ferramentas, não das regras semânticas de formatação.

---

## 8. RUNTIME JAVA

O projeto oficial informa que o formatter de linha de comando utiliza o módulo `jdk.compiler` para analisar o código Java e, portanto, deve ser executado com um JDK cuja versão seja igual ou superior à versão da linguagem Java do código formatado.

A versão mínima atual do Java é documentada pelo projeto como Java 21. citeturn0view0

Isso não deve ser interpretado como uma restrição à versão do Java do código-fonte que pode ser formatado; é um requisito do runtime/ferramenta.

---

## 9. INTELLIJ IDEA

Existe um plugin oficial do `google-java-format` para IntelliJ, disponibilizado pelo repositório de plugins da JetBrains.

Instalação:

```text
IntelliJ IDEA
    ↓
Settings
    ↓
Plugins
    ↓
Marketplace
    ↓
google-java-format
    ↓
Install
```

O plugin fica desabilitado por padrão.

Habilite-o em:

```text
Project Settings
    ↓
google-java-format Settings
    ↓
Enable google-java-format
```

Quando habilitado, o plugin substitui as ações normais:

```text
Reformat Code
Optimize Imports
```

ações. citeturn0view0

---

## 10. CONFIGURAÇÃO DO JRE DO INTELLIJ

A documentação oficial do plugin informa que podem ser necessárias exportações adicionais da JVM para o runtime da IDE.

A configuração documentada é:

```text
--add-exports=jdk.compiler/com.sun.tools.javac.api=ALL-UNNAMED
--add-exports=jdk.compiler/com.sun.tools.javac.code=ALL-UNNAMED
--add-exports=jdk.compiler/com.sun.tools.javac.file=ALL-UNNAMED
--add-exports=jdk.compiler/com.sun.tools.javac.parser=ALL-UNNAMED
--add-exports=jdk.compiler/com.sun.tools.javac.tree=ALL-UNNAMED
--add-exports=jdk.compiler/com.sun.tools.javac.util=ALL-UNNAMED
```

A documentação oficial orienta colocar essas opções nas opções personalizadas da VM do IntelliJ e reiniciar a IDE depois. citeturn0view0

---

## 11. ECLIPSE

O projeto oficial também fornece um plugin para Eclipse.

Ele disponibiliza:

```text
google-java-format
    = 2 espaços

aosp-java-format
    = 4 spaces
```

The formatter Google padrão uses 2 espaços. citeturn0view0

---

## 12. USO COMO BIBLIOTECA

O `google-java-format` também pode ser incorporado a softwares que geram código-fonte Java.

The official Maven dependency uses:

```xml
<dependency>
  <groupId>com.google.googlejavaformat</groupId>
  <artifactId>google-java-format</artifactId>
  <version>${google-java-format.version}</version>
</dependency>
```

Gradle:

```gradle
dependencies {
  implementation 'com.google.googlejavaformat:google-java-format:$googleJavaFormatVersion'
}
```

A API principal do formatter é:

```java
com.google.googlejavaformat.java.Formatter
```

Por exemplo:

```java
String formattedSource = new Formatter().formatSource(sourceString);
```

Essas APIs são documentadas pelo projeto oficial. citeturn0view0

---

## 13. PAPEL DA API DO FORMATTER

Quando o `JAVA FORMAT ULTIMATE` for implementado programaticamente, o Google Java Format pode ser utilizado como motor de formatação mecânica.

Conceitualmente:

```text
Java Source
    ↓
Google Java Format
    ↓
Google-formatted Java
    ↓
Semantic post-processing / policy validation
    ↓
JAVA FORMAT ULTIMATE output
```

Entretanto, qualquer pós-processamento deve obedecer ao contrato de preservação.

---

## 14. REQUISITO DE PRESERVAÇÃO

O Google Java Format é um formatter, não um motor de refatoração.

Para o `JAVA FORMAT ULTIMATE`, o contrato de preservação continua absoluto:

```text
NENHUMA ALTERAÇÃO DE LÓGICA
NENHUMA ALTERAÇÃO DE COMPORTAMENTO
NENHUMA REFACTORING ESTRUTURAL
NENHUMA RENOMEAÇÃO
NENHUM CÓDIGO NOVO
NENHUM CÓDIGO REMOVIDO
```

Formatação é apenas apresentação.

---

## 15. IMPORTS

O Google Java Format possui comportamento relacionado a imports e opções oficiais de linha de comando como:

```text
--fix-imports-only
--skip-sorting-imports
--skip-removing-unused-import
```

Para o `JAVA FORMAT ULTIMATE`, a política própria de preservação do agente tem precedência quando o requisito explícito for preservar os imports exatamente.

Não introduza silenciosamente alterações nos imports quando o contrato do formatter semântico determinar que eles devem permanecer intactos. citeturn0view0

---

## 16. COMENTÁRIOS E JAVADOC

O Google Java Format possui comportamento de formatação para comentários e Javadoc e disponibiliza:

```text
--skip-javadoc-formatting
```

Para o `JAVA FORMAT ULTIMATE`, comentários e Javadoc devem permanecer semanticamente e textualmente preservados, salvo quando a implementação selecionada definir explicitamente alterações de espaços apenas como permitidas. citeturn0view0

---

## 17. FORMATAÇÃO POR LINHAS

O Google Java Format permite limitar a formatação a linhas específicas por meio de:

```text
--lines
```

e a offsets específicos de caracteres por meio de:

```text
--offset
```

Isso pode ser útil para integrações com editores e fluxos de formatação incremental. citeturn0view0

---

## 18. FORMATAÇÃO DE DIFF / PATCH

O projeto oficial fornece:

```text
google-java-format-diff.py
```

para reformatar as linhas alteradas em um patch específico.

Isso é útil em fluxos nos quais apenas as regiões modificadas devem ser consideradas. citeturn0view0

---

## 19. VALIDAÇÃO

The official formatter provides:

```text
--dry-run
--set-exit-if-changed
```

Essas opções podem ser usadas em CI para detectar arquivos que diferem da saída esperada do Google Java Format. citeturn0view0

Para o `JAVA FORMAT ULTIMATE`, valide adicionalmente:

```text
equivalência do AST
+
preservação semântica
+
idempotência
```

---

## 20. MODELO DE AÇÃO DA IDE

Com o plugin oficial do IntelliJ habilitado:

```text
Ctrl + Alt + L
        ↓
Reformat Code
        ↓
google-java-format
```

Quando habilitado, o plugin substitui a ação normal de formatação do IntelliJ pelo Google Java Format. citeturn0view0

---

## 21. VARIANTE AOSP

O Google Java Format também possui um modo AOSP:

```text
--aosp
```

The official documentation identifies the formatter AOSP as using 4-space indentation.

Para o `JAVA FORMAT ULTIMATE`, **não use o modo AOSP** a menos que seja explicitamente solicitado.

Default:

```text
Google Java Format
2 espaços
```

---

## 22. GOOGLE JAVA FORMAT VS FORMATTER DO INTELLIJ

Não trate os dois como o mesmo formatter.

```text
IntelliJ default formatter
    ≠
Google Java Format
```

Quando o plugin do Google Java Format para IntelliJ está habilitado, o formatter Google substitui o comportamento normal de `Reformat Code`. citeturn0view0

---

## 23. GOOGLE JAVA FORMAT VS EDITORCONFIG

Não crie um `.editorconfig` fictício do Google.

O algoritmo de formatação do Google Java Format não é controlado por um `.editorconfig` oficial do Google.

Use `.editorconfig` apenas para configurações genéricas do editor/arquivo quando necessário.

O comportamento real de formatação Java vem do `google-java-format`.

---

## 24. GOOGLE JAVA FORMAT VS JAVA FORMAT ULTIMATE

### Google Java Format

```text
Padrão de formatação mecânica
```

### JAVA FORMAT ULTIMATE

```text
Google Java Format
        +
Regras de formatação semântica
        +
Quebra mínima
        +
Linhas em branco semânticas
        +
Agrupamento semântico
        +
Validação de preservação
```

---

# BASE OFICIAL PARA O JAVA FORMAT ULTIMATE

Use estas características do Google Java Format como base mecânica:

```text
Formatter:
    google-java-format

Estilo:
    Google Java Style

Indentação padrão:
    2 espaços

AOSP:
    4 espaços — somente quando explicitamente solicitado

Tabs:
    Não utilizar como indentação

Algoritmo:
    Não configurável pelo usuário

IDE:
    Plugin disponível para IntelliJ / Android Studio

Ação principal:
    Reformat Code

CLI:
    google-java-format

Validação:
    --dry-run
    --set-exit-if-changed

Incremental:
    --lines
    --offset
    google-java-format-diff.py
```

---

# CONTRATO DE INTEGRAÇÃO

O agente `JAVA FORMAT ULTIMATE` deve tratar este documento como sua **base Google Java Format**.

Prioridade:

```text
JAVA FORMAT ULTIMATE preservation rules
            ↓
JAVA FORMAT ULTIMATE semantic rules
            ↓
Google Java Format mechanical conventions
            ↓
Generic editor settings
```

O algoritmo do Google Java Format não deve ser apresentado falsamente como configurável.

---

# FONTE OFICIAL

Fonte principal:

https://github.com/google/google-java-format

O repositório oficial informa que o `google-java-format` reformata código Java para seguir o Google Java Style, documenta sua CLI, plugin para IntelliJ, plugin para Eclipse e API de biblioteca, além de declarar explicitamente que o algoritmo de formatação não é configurável por projeto. citeturn0view0

# REGRA FINAL

```text
GOOGLE JAVA FORMAT = BASE MECÂNICA

JAVA FORMAT ULTIMATE = GOOGLE BASELINE
                     + SEMANTIC FORMAT
                     + MINIMAL BREAKING
                     + PRESERVATION
```
