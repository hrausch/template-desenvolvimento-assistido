# Arquitetura

**Sistema de Gestão de XXXX**

> Este documento segue o padrão fullstack descrito em
> [`../reference-architecture.md`](../reference-architecture.md) (arquitetura de
> referência do time). Onde o sistema diverge do padrão, a
> divergência e o motivo devem ficar explícitos no texto — o restante deve ser lido
> como aplicação direta do padrão de referência.

## Containers

As unidades de implantação do sistema: quais processos rodam, quais tecnologias utilizam e como se comunicam entre si.

| Container | Tecnologia | Responsabilidade |
|---|---|---|
| **Aplicação Web** | React (SPA) | Renderiza a interface no navegador/celular do usuário. Toda interação passa por chamadas à API Backend. |
| **API Backend** | Spring Boot / REST | Concentra toda a lógica de negócio dos módulos do sistema (ver Componentes abaixo). Autentica requisições via o módulo IAM embarcado, persiste dados no banco. |
| **Módulo IAM** | Biblioteca Java (dependência Maven, embarcada na API Backend) | Gerencia autenticação, RBAC (perfis de acesso) e auditoria de segurança. Não é um serviço externo — roda dentro do processo da API Backend. Ver [`IAM.md`](./IAM.md) para o guia de integração. |
| **Banco de Dados** | PostgreSQL | Armazena o estado persistente do sistema. |
| **Serviço de E-mail** | Externo (SMTP / API) | Envia e-mails transacionais (recuperação de senha, notificações), se aplicável. |

> Adicione/remova containers conforme a necessidade do projeto (ex.: fila de mensagens, storage de arquivos, cache).

## Componentes

Os componentes abaixo correspondem aos módulos internos da API Backend. Cada módulo encapsula as regras de negócio de um contexto específico do sistema, alinhado aos requisitos funcionais em [`../requisitos/requirements.md`](../requisitos/requirements.md).

| Módulo | Responsabilidade | Requisitos relacionados |
|---|---|---|
| **`<modulo1>`** | | RF01 |
| **`<modulo2>`** | | RF02 |
| **shared** | Código transversal: configuração de segurança, tratamento global de exceções e utilitários. | — |

O controle de perfil de acesso e o log de auditoria são resolvidos pelo módulo IAM embarcado, descrito em [`IAM.md`](./IAM.md), e não por um módulo de negócio próprio — com exceção de regras de escopo de dados específicas do domínio (se houver), que devem viver no módulo de negócio responsável, não no IAM.

## Decisões de Arquitetura

### Autenticação e Perfis de Acesso (IAM)

O sistema utiliza o **módulo IAM** como biblioteca (dependência Maven) embarcada na API Backend — não como um serviço externo. O módulo resolve autenticação (login/token opaco), RBAC (roles e permissions) e auditoria de segurança de forma automática.

**Perfis (roles) do sistema:**

> Defina os perfis de acesso do projeto (ver RF de usuários/perfis em [`../requisitos/requirements.md`](../requisitos/requirements.md)).

| Role | Perfil de negócio | Uso típico |
|---|---|---|
| `ROLE_ADMINISTRADOR` | Administrador do sistema | Gerencia usuários e perfis, acesso total de escrita. |
| `ROLE_...` | | |

Decida se o sistema usa o recurso de contexto multi-tenant (`X-Context-Id`) do módulo IAM ou se todas as roles são atribuídas de forma global ao usuário — registre a decisão e o motivo aqui.

**Regra:** todo endpoint que exige perfil específico deve ser protegido com `@PreAuthorize`, usando as permissions granulares descritas em [`IAM.md`](./IAM.md).

**Detalhes de segurança na configuração do Spring:**

- Token opaco em `Authorization: Bearer`, sessão stateless (`SessionCreationPolicy.STATELESS`) — o `IamTokenAuthFilter` roda antes do `UsernamePasswordAuthenticationFilter` (ver [`IAM.md §4.6`](./IAM.md#46-configurar-spring-security)).
- `authenticationEntryPoint` configurado para devolver 401 (não autenticado) e `accessDeniedHandler` para devolver 403 (autenticado, sem permissão). Sem o `accessDeniedHandler`, uma negação de permissão sai como 401 e o frontend confunde com sessão expirada, disparando um redirect de login indevido.
- Decida se o sistema tem cadastro público ou não (contas criadas por um Administrador). `permitAll` explícito e enumerado; todo o resto cai em `anyRequest().authenticated()` por padrão.
- Recursos de desenvolvimento (console do H2, `frameOptions` desabilitado) habilitados **apenas no perfil `dev`** — nunca em `prod`.

### [Regras de negócio específicas do domínio]

> Documente aqui decisões arquiteturais motivadas por regras de negócio específicas do projeto (ex.: imutabilidade de um tipo de registro, escopo de dados por vínculo entre entidades, fluxos de aprovação). Uma subseção por decisão, explicando a regra, a consequência arquitetural e onde ela é implementada.

### Erros — formato único

Toda a API responde erros no mesmo formato — **RFC 9457 Problem Details** (`ProblemDetail`, nativo do Spring 6+): `{ "status", "title", "detail" }`. Um `ApiExceptionHandler` central (`@RestControllerAdvice` em `shared/exception/`) concentra a tabela exceção → HTTP:

| Exceção | HTTP |
|---|---|
| `ResourceNotFoundException` | 404 |
| Bean Validation (`MethodArgumentNotValidException`) | 400 |
| Regra de negócio violada | 422 |
| Conflito / vínculo impede a operação | 409 |
| Token de fluxo expirado ou já consumido (delegado do IAM, ver [`IAM.md §6`](./IAM.md#6-exceptions-do-módulo)) | 410 |
| **catch-all `Exception`** | 500 genérico, com stack trace no log |

"Não encontrado" nunca é modelado como `IllegalArgumentException` → 400. Toda exceção de negócio nova do sistema é: (1) uma classe em `shared/exception/`, (2) um handler na `ApiExceptionHandler`, (3) uma linha nesta tabela.

### Banco de dados, perfis e migrações

- **Flyway desde a primeira migração**: `V1__baseline.sql` e uma migração nova por mudança de schema nos módulos de negócio. `ddl-auto: validate` em produção (o schema é só o que o Flyway aplicou — nunca gerado automaticamente pelo Hibernate); `create-drop` no perfil de teste.
- Perfis Spring:
  - `application.yml` — comum a todos os ambientes (nome da aplicação, e-mail, regras de negócio);
  - `application-dev.yml` — H2, console habilitado, log em `DEBUG`;
  - `application-prod.yml` — PostgreSQL via variáveis de ambiente (`DB_URL`, `DB_USER`, `DB_PASSWORD`), log em `INFO`, actuator de health habilitado.
- Timezone da conexão JDBC fixo em UTC. Segredos só via variável de ambiente, com default vazio no `.yml` — nunca valor real commitado.
- `spring-boot-starter-actuator` com `/actuator/health` exposto para o healthcheck do container (ver [`../cicd/CICD.md`](../cicd/CICD.md)).
- **Exceção documentada:** o módulo IAM embarcado cria e gerencia suas próprias tabelas (`iam_*`) via `ddl-auto` da biblioteca, fora do controle do Flyway do sistema — ver [`IAM.md §2.3`](./IAM.md#23-configurar-banco-de-dados). O Flyway do sistema versiona apenas o schema dos módulos de negócio.

### Testes

- **Integração como espinha dorsal**: `BaseIntegrationTest` com `@SpringBootTest + @AutoConfigureMockMvc + @ActiveProfiles("test") + @Transactional`, com helpers de alto nível (ex.: `createUserAndGetToken`, `performWithToken`) para não repetir o setup de autenticação em cada teste.
- Teste unitário para `domain/` e lógica de entidade (Rich Entity); teste de integração para o fluxo controller → banco por caso de uso.
- Nomenclatura: `should_x_when_y` nos testes unitários; `@DisplayName("TC-X.Y: …")` amarrando ao caso de teste documentado em [`../plano-de-testes/`](../plano-de-testes/).
- JaCoCo no build com gate de cobertura sobre `usecase/` e `domain/` — DTOs e classes de configuração ficam fora do gate.

### Backend

O backend adota a abordagem **Feature-based + Layered**: o código é organizado por funcionalidade de negócio, e dentro de cada funcionalidade as responsabilidades são separadas em camadas bem definidas. Isso facilita a localização do código, limita o acoplamento entre módulos e torna o sistema mais fácil de manter e evoluir.

**Camadas**

O fluxo de uma requisição percorre quatro camadas em sequência, cada uma com responsabilidade exclusiva:

```
┌─────────────────────────────┐
│         controller/         │  ← recebe e responde HTTP
├─────────────────────────────┤
│          usecase/           │  ← regras de negócio (1 classe por caso de uso)
├─────────────────────────────┤
│           domain/           │  ← componentes de domínio reutilizáveis entre use cases
├─────────────────────────────┤
│        persistence/         │  ← entidades e acesso ao banco
├─────────────────────────────┤
│       Banco de Dados        │
└─────────────────────────────┘
```

| Pasta | Responsabilidade |
|---|---|
| **controller/** | Recebe a requisição HTTP, valida o DTO de entrada e devolve a resposta JSON. Não contém lógica de negócio. Um controller por recurso REST — se acumular responsabilidades de mais de um recurso, divide-se. |
| **usecase/** | Uma classe por caso de uso. Nome sempre `*UseCase`, nunca `*Service`. Orquestra chamadas ao repositório e à camada `domain/`. |
| **domain/** | Regras de domínio reutilizadas por mais de um use case do módulo, ou por outro módulo. Fica vazia em módulos simples — só é criada quando a duplicação aparece. Um módulo pode depender do `domain/` de outro módulo, nunca do `usecase/` ou `controller/` alheio. |
| **dto/** | Objetos de transferência de dados usados na comunicação entre Controller e UseCase, implementados como `record`. DTO de resposta expõe um factory estático (`XxxResponseDTO.from(entity)`); DTO de criação usa Bean Validation obrigatória; DTO de atualização (PATCH) tem todos os campos anuláveis — `null` significa "não alterar". |
| **persistence/** | Repositórios JPA. Não contém lógica de negócio — apenas operações de leitura e escrita. |
| **persistence/model/** | Entidades JPA que mapeiam as tabelas do banco de dados, seguindo o padrão *Rich Entity*: transições de estado vivem na própria entidade em vez de espalhadas pelos use cases. Entidade expõe factory estático `Entity.create(...)` e `@PrePersist` para os campos derivados. |

**Fluxo de dados**

```
Request (DTO) → Controller → UseCase → [Domain] → Repository → Banco de Dados
```

**Estrutura de pastas**

Cada módulo de negócio é autossuficiente — contém seu controller, casos de uso, entidades, repositório e DTOs organizados em subpastas. Código genuinamente transversal (exceções globais, configurações de segurança, utilitários) fica no pacote `shared`.

```
src/main/java/<pacote-base>/
│
├── <modulo1>/
│   ├── controller/
│   │   └── <Modulo1>Controller.java
│   ├── usecase/
│   │   └── <Modulo1>UseCase.java
│   ├── dto/
│   │   ├── <Modulo1>RequestDTO.java
│   │   └── <Modulo1>ResponseDTO.java
│   └── persistence/
│       ├── <Modulo1>Repository.java
│       └── model/
│           └── <Modulo1>.java
│
├── <modulo2>/
│   └── ...
│
└── shared/
    ├── exception/
    │   ├── NotFoundException.java
    │   └── GlobalExceptionHandler.java
    ├── config/
    │   └── SecurityConfig.java
    └── util/
        └── DateUtils.java
```

### Frontend

Stack: React + TypeScript, organizado por feature — cada módulo de negócio tem seus próprios componentes, hooks, serviços e tipos. Código transversal (componentes genéricos, utilitários) fica em `shared`.

**Camadas**

| Pasta | Responsabilidade |
|---|---|
| `components/` | JSX, só visual — sem lógica de negócio |
| `hooks/` | Lógica reutilizável do React (`useState`, `useEffect`, etc.) |
| `services/` | Chamadas HTTP à API — sem JSX |
| `types/` | Interfaces e types TypeScript do módulo |
| `index.ts` | Porta de entrada do módulo — exporta só o que o resto precisa ver |

Feature importa de outra feature **apenas via `index.ts`**; `pages/` não contém regra de negócio e `services/` não contém JSX.

**Estrutura de pastas**

```
src/
│
├── features/
│   ├── <modulo1>/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── services/
│   │   ├── types/
│   │   └── index.ts
│   │
│   └── <modulo2>/
│       └── ...
│
├── shared/                          ← compartilhado entre features
│   ├── components/
│   │   ├── Button.tsx
│   │   └── Modal.tsx
│   ├── hooks/
│   │   └── useDebounce.ts
│   └── types/
│       └── api.types.ts
│
├── pages/                           ← só monta as features na tela
│   └── <Modulo1>Page.tsx
│
└── app/
    ├── App.tsx
    └── routes.tsx
```

**Limite de tamanho:** componente com mais de ~250 linhas ou mais de um papel (ex.: lista + modal + formulário no mesmo arquivo) é dividido. Sem esse limite, uma feature tende a virar um único componente `Manager` monolítico conforme cresce.

**Cliente HTTP e sessão**

Um único `apiClient` (axios) em `shared/services/` (ou `shared/api/`) com:

- interceptor de **request**: injeta `Authorization: Bearer <token>` e remove o `Content-Type` quando o body é `FormData` (o browser define o boundary);
- interceptor de **response**: `401` → limpa as credenciais e redireciona para o login — a expiração de sessão é tratada uma vez no interceptor, não repetida tela a tela.

**Estado**

- **Estado de servidor → TanStack Query.** Cache, loading, erro, retry e invalidação de cada chamada HTTP não são reimplementados à mão em `useState`/`useEffect` — esse padrão é sujeito a race conditions e não tem cache. Query keys por feature.
- **Estado de UI → local** (`useState`) ou contexto pequeno quando realmente compartilhado entre componentes. Sem store global por padrão.
- Composições caras de N chamadas no cliente indicam endpoint agregado faltando no backend — resolver no contrato, não empilhando `Promise.all` no frontend.

**Rotas**

- Objetos de rota (`createBrowserRouter`) com rotas-layout: um `ProtectedLayout` pai para tudo que exige autenticação (guard de sessão + casca visual comum), com `<Outlet/>` para as páginas de cada perfil. Guard escrito uma vez, não repetido rota a rota.
- Caminhos em kebab-case; rotas públicas (`/login`, `/recuperar-senha`) ficam fora do layout protegido.

**Qualidade**

- ESLint + `typescript-eslint` estritos desde o início; build = `tsc -b` + Vite.
- **Vitest + Testing Library + MSW**, na ordem utils puros → hooks → fluxos críticos.
- Design tokens do design system (ver [`../design-system/design-system.md`](../design-system/design-system.md)) declarados no tema do Tailwind — nunca hex inline em `className`.
- `ErrorBoundary` no root da aplicação + mecanismo único de notificação (toast) em `shared/`.

---

## Contrato de API (OpenAPI)

- `openapi.yaml` é escrito **antes** do código das features novas — request/response nascem no contrato, não no controller. Deve ser criado seguindo o checklist do [`reference-architecture.md §7`](../reference-architecture.md#7-checklist-para-iniciar-um-projeto-novo) quando o backend for iniciado.
- O backend expõe `springdoc-openapi`; a divergência entre o spec efetivo e o `openapi.yaml` versionado é verificada no CI (ou em checklist de release).
- Types do frontend gerados a partir do contrato com `openapi-typescript`, eliminando a triplicação spec × DTO Java × `*.types.ts` mantidos manualmente.
- Convenções fixas do contrato do sistema:
  - autenticação: `Authorization: Bearer <token opaco>`;
  - erros: Problem Details (ver seção de erros acima);
  - coleções com potencial de crescimento: paginadas (`page`, `size`, envelope `{ content, page, totalElements }`).
