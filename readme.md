# Instructions — Guia de Instruções para Agentes LLM

## O que é este diretório?

Este diretório contém conjuntos completos de instruções destinados a serem consumidos por **agentes LLM** (como o GitHub Copilot, Cursor, Cline, Aider, entre outros) durante sessões de desenvolvimento. Cada subdiretório corresponde a uma linguagem/plataforma e fornece regras, convenções, boas práticas e diretivas de segurança que o agente deve seguir ao gerar, rever ou modernizar código.

O objetivo é transformar o agente num membro da equipa que conhece e respeita as convenções do projeto desde o primeiro prompt.

---

## Estrutura e Conteúdo

```
instructions/
├── readme.md                        ← este ficheiro
├── python/
│   └── github/
│       ├── copilot-instructions.md          # Instruções-raiz (arquitetura, commits, referências)
│       ├── python-coding.instructions.md    # Convenções de código Python (PEP 8, tipagem, async)
│       ├── documentation.instructions.md    # Docstrings, README, CHANGELOG, ADRs
│       ├── testing.instructions.md          # pytest, fixtures, testes de segurança
│       ├── project-structure.instructions.md# Layout src/, pyproject.toml, CI/CD
│       ├── security.instructions.md         # OWASP Top 10, validação, auth, segredos
│       ├── code-review.agent.md             # Agente de revisão automática de código
│       └── code-modernization.agent.md      # Agente de migração Python 3.7 → 3.11+
│
├── c-sharp/
│   └── github/
│       ├── copilot-instructions.md          # Instruções-raiz (.NET 8, arquitetura, commits)
│       ├── csharp-coding.instructions.md    # Convenções C# 10-12 (naming, DI, async, LINQ)
│       ├── documentation.instructions.md    # XML doc comments, Swagger, DocFX, ADRs
│       ├── testing.instructions.md          # xUnit, FluentAssertions, testes de segurança
│       ├── project-structure.instructions.md# Clean Architecture, Central Package Mgmt, Docker
│       ├── security.instructions.md         # OWASP Top 10, ASP.NET Core hardening
│       ├── code-review.agent.md             # Agente de revisão automática de código
│       └── code-modernization.agent.md      # Agente de migração .NET Framework → .NET 8
│
├── javascript/
│   └── github/
│       ├── copilot-instructions.md          # Instruções-raiz (TypeScript, Angular 17+, commits)
│       ├── javascript-coding.instructions.md# Convenções TS/JS/Angular (signals, DI, RxJS)
│       ├── documentation.instructions.md    # TSDoc, Compodoc, README, CHANGELOG, ADRs
│       ├── testing.instructions.md          # Jest, Angular TestBed, Playwright, segurança
│       ├── project-structure.instructions.md# Layout Angular, ESLint flat config, CI/CD, Docker
│       ├── security.instructions.md         # OWASP Top 10, XSS, CSP, Angular security
│       ├── code-review.agent.md             # Agente de revisão automática de código
│       └── code-modernization.agent.md      # Agente de migração AngularJS/JS → Angular 17+/TS
│
└── java/
    └── github/
        ├── copilot-instructions.md          # Instruções-raiz (Java 21, Spring Boot 3.x, commits)
        ├── java-coding.instructions.md      # Convenções Java 21 (records, sealed, pattern matching)
        ├── documentation.instructions.md    # Javadoc, OpenAPI/Swagger, README, CHANGELOG, ADRs
        ├── testing.instructions.md          # JUnit 5, Mockito, AssertJ, Testcontainers, ArchUnit
        ├── project-structure.instructions.md# Hexagonal architecture, Maven/Gradle, CI/CD, Docker
        ├── security.instructions.md         # OWASP Top 10, Spring Security, deserialization, Log4Shell
        ├── code-review.agent.md             # Agente de revisão automática de código
        └── code-modernization.agent.md      # Agente de migração Java 8/11 → 21 + Spring Boot 2 → 3
```

### Tipos de ficheiro

| Sufixo | Finalidade | Quando é carregado |
|--------|------------|--------------------|
| `.instructions.md` | Regras e convenções que o agente deve seguir | Automaticamente, como contexto de fundo |
| `.agent.md` | Define um agente especializado com inputs, outputs e comportamento | Quando o utilizador invoca o agente ou pede uma tarefa específica |

---

## Porquê usar ficheiros de instruções?

### 1. Consistência
Sem instruções, cada resposta do agente é uma interpretação individual. Com instruções, todo o código gerado segue as mesmas convenções — naming, formatação, tratamento de erros, segurança — como se houvesse um guia de estilo vivo a supervisionar cada linha.

### 2. Segurança por defeito
Os ficheiros `security.instructions.md` garantem que o agente nunca gera código com vulnerabilidades conhecidas (SQL injection, XSS, segredos hardcoded, `eval()`, `BinaryFormatter`, etc.), independentemente de quem faz o prompt.

### 3. Onboarding instantâneo
Novos membros da equipa (humanos ou agentes) absorvem imediatamente as convenções do projeto sem necessidade de formação ou documentação externa.

### 4. Qualidade reprodutível
Revisões de código e modernizações seguem sempre os mesmos critérios, eliminando variabilidade entre sessões.

### 5. Conhecimento acumulado
As instruções funcionam como repositório vivo de decisões técnicas (ADRs, escolhas de bibliotecas, padrões de arquitetura) que evolui com o projeto.

---

## Como integrar num projeto

### Passo 1 — Copiar o subdiretório adequado

Copie a pasta `github/` do subdiretório da linguagem pretendida para a raiz do seu repositório, dentro de `.github/`:

```
# Para um projeto Python
cp -r instructions/python/github/ <meu-projeto>/.github/

# Para um projeto C#
cp -r instructions/c-sharp/github/ <meu-projeto>/.github/

# Para um projeto JavaScript / TypeScript / Angular
cp -r instructions/javascript/github/ <meu-projeto>/.github/

# Para um projeto Java / Spring Boot
cp -r instructions/java/github/ <meu-projeto>/.github/
```

A estrutura resultante no projeto será:

```
meu-projeto/
├── .github/
│   ├── copilot-instructions.md
│   ├── python-coding.instructions.md   (ou csharp-coding / javascript-coding)
│   ├── documentation.instructions.md
│   ├── testing.instructions.md
│   ├── project-structure.instructions.md
│   ├── security.instructions.md
│   ├── code-review.agent.md
│   └── code-modernization.agent.md
├── src/
├── tests/
└── ...
```

### Passo 2 — Personalizar

Edite os ficheiros para refletir as especificidades do seu projeto:

- **`copilot-instructions.md`** — Atualize o nome do projeto, stack concreta, referências a módulos internos.
- **`*-coding.instructions.md`** — Ajuste regras de naming, bibliotecas preferidas, versões mínimas.
- **`security.instructions.md`** — Adicione políticas específicas da organização (ex.: provider de secrets, WAF, compliance).
- **`project-structure.instructions.md`** — Atualize a árvore de diretórios e pipelines de CI/CD.

### Passo 3 — Commit

Faça commit dos ficheiros de instruções no repositório. Eles são código — devem ser versionados, revistos em PR, e evoluir com o projeto.

```bash
git add .github/*.md
git commit -m "docs: add LLM agent instructions for [Python|C#|JavaScript|Java]"
```

---

## Como são passados como contexto ao agente LLM

### GitHub Copilot (VS Code / Visual Studio)

O GitHub Copilot carrega automaticamente ficheiros de instruções colocados em `.github/`:

| Ficheiro | Comportamento |
|----------|--------------|
| `.github/copilot-instructions.md` | Carregado **sempre** como contexto de fundo em qualquer interação |
| `.github/*.instructions.md` | Carregados automaticamente quando relevantes para a tarefa |
| `.github/*.agent.md` | Disponíveis como agentes invocáveis (ex.: `@code-review`) |

> **Nota**: O frontmatter YAML no topo de cada ficheiro (campo `description`) ajuda o Copilot a decidir quando incluir o ficheiro no contexto.

### Cursor

No Cursor, coloque os ficheiros na pasta `.cursor/rules/` ou referencie-os em `.cursorrules`:

```
.cursor/
└── rules/
    ├── copilot-instructions.md
    ├── python-coding.instructions.md
    └── ...
```

### Outros agentes (Cline, Aider, Continue, etc.)

A maioria dos agentes aceita instruções via:

1. **Ficheiros de sistema** — Configuráveis no settings do agente (system prompt files).
2. **Referência direta** — Incluir o ficheiro no prompt: `@file:.github/security.instructions.md`.
3. **Contexto de projeto** — Alguns agentes fazem scan automático de `.md` na raiz ou em `.github/`.

### Inclusão manual no prompt

Se o agente não suportar carregamento automático, pode sempre colar ou referenciar o conteúdo:

```
Por favor, segue as convenções definidas neste ficheiro:

<instruções>
[conteúdo do ficheiro .instructions.md]
</instruções>

Agora, implementa a seguinte funcionalidade: ...
```

---

## Dicas de boa utilização

### Mantenha os ficheiros atualizados
Instruções desatualizadas são piores do que nenhumas — geram código que parece correto mas não segue as práticas atuais. Reveja os ficheiros em cada sprint ou quando há mudanças significativas no projeto.

### Não duplique, referencie
Cada ficheiro `.instructions.md` cobre um domínio específico. Se um ficheiro precisa de regras de outro, referencie-o (`ver security.instructions.md`) em vez de copiar o conteúdo.

### Comece com o `copilot-instructions.md`
Este é o ponto de entrada. Garanta que contém uma boa visão geral do projeto e referências claras para os ficheiros especializados.

### Use os agentes para tarefas concretas
Os ficheiros `.agent.md` definem fluxos estruturados. Use-os para tarefas repetitivas:
- **`@code-review`** antes de abrir um PR.
- **`@code-modernization`** quando for hora de atualizar dependências ou migrar padrões.

### Versione e reveja como código
Os ficheiros de instruções devem passar pelo mesmo processo de review que o código. Uma regra mal definida propaga-se a todo o código gerado pelo agente.

### Adapte ao contexto do projeto
Os ficheiros fornecidos são templates abrangentes. Remova o que não se aplica e adicione o que é específico do seu domínio. Instruções mais curtas e focadas são mais eficazes do que documentos longos e genéricos.

---

## Dicas para prompts mais eficientes

### 1. Seja específico sobre o que quer

```
❌ "Cria um serviço de utilizadores"
✅ "Cria um UserService que implementa IUserService com métodos CRUD assíncronos,
    usando o repository pattern, com CancellationToken em todos os métodos,
    seguindo as convenções de csharp-coding.instructions.md"
```

### 2. Referencie os ficheiros de instruções relevantes

```
✅ "Seguindo as regras de security.instructions.md, revê este controller
    e identifica vulnerabilidades"
```

O agente prioriza regras que são explicitamente referenciadas no prompt.

### 3. Divida tarefas complexas

Em vez de pedir uma feature completa num único prompt, divida-a:

```
Prompt 1: "Cria o modelo de domínio para Order com as propriedades X, Y, Z"
Prompt 2: "Cria o repositório IOrderRepository e a implementação com EF Core"
Prompt 3: "Cria o OrderService com validação usando FluentValidation"
Prompt 4: "Cria os testes unitários para OrderService"
```

### 4. Peça revisão antes de aceitar

```
✅ "Revê o código que acabaste de gerar usando as regras de code-review.agent.md
    e corrige os problemas encontrados"
```

### 5. Use o agente de modernização proativamente

```
✅ "Analisa este ficheiro e sugere modernizações seguindo
    code-modernization.agent.md"
```

### 6. Forneça contexto de negócio

```
✅ "Este endpoint é público e exposto à internet. Implementa-o seguindo
    security.instructions.md com especial atenção a rate limiting e validação de input"
```

O agente ajusta o nível de rigor com base no contexto de risco.

### 7. Peça explicações quando necessário

```
✅ "Explica porque escolheste esta abordagem em vez de X,
    referenciando as instruções relevantes"
```

Isto ajuda a validar que o agente está efetivamente a seguir as instruções.

### 8. Itere com feedback

```
Prompt 1: "Implementa X"
Prompt 2: "O método Y deveria usar async/await com CancellationToken.
           Corrige seguindo as convenções de csharp-coding.instructions.md"
```

Correções incrementais com referência a instruções específicas são mais eficazes do que reformular o pedido inteiro.

---

## Contribuir

Para adicionar suporte a uma nova linguagem ou framework:

1. Crie um novo subdiretório em `instructions/` (ex.: `go/`, `rust/`, `kotlin/`).
2. Dentro dele, crie a pasta `github/` com os ficheiros seguindo a mesma estrutura.
3. Adapte todo o conteúdo à linguagem-alvo, mantendo a cobertura de segurança.
4. Atualize este `readme.md` com a nova entrada na árvore de diretórios.

---

## Licença

Estes ficheiros de instruções são internos ao projeto e destinam-se a uso pela equipa de desenvolvimento e pelos agentes LLM configurados no repositório.
