# Arquitetura de Referência — projetos fullstack




## 1. Organização dos repositórios

```
projeto/
├── <nome>            ← docs: requisitos, use cases, wireframes, OpenAPI
├── <nome>-backend    ← Java + Spring Boot
└── <nome>-frontend   ← React + TypeScript
```

- Os três repos são **irmãos**, lado a lado. O repo de docs **não é submódulo**
  dos repos de código: de dentro do backend, a documentação é
  `../<nome>/docs/`. Editar sempre no repo de docs.
- **Por que não submódulo.** O ponteiro atrasa sozinho — é preciso lembrar de
  bumpá-lo em dois repos a cada alteração —, e uma cópia defasada da
  arquitetura é pior que nenhuma: quem a lê acerta a sintaxe e erra a regra,
  sem nada ficar vermelho. O benefício ("quem clona o código recebe a
  documentação") é de equipe grande; o custo é um job de CI só para avisar
  quando o esquecimento aconteceu. Uma cópia só, sempre a atual.
- **O preço de não ter submódulo é uma linha de configuração:** as ferramentas
  de arquivo só leem sem prompt dentro do diretório de trabalho, então cada
  repo de código precisa de `permissions.additionalDirectories` apontando para
  o repo de docs — senão um subagente trava esperando aprovação. Ver
  [`agentes/README.md §5`](agentes/README.md).
- Cada repo tem seu `CLAUDE.md`/`README` com as convenções da stack; o repo de
  docs tem o `CLAUDE.md` fullstack que explica a relação entre os três.
- **`openapi/openapi.yaml` é o contrato** — request/response nascem nele.
  Fluxo de feature: contrato → backend → frontend → atualizar docs → bump do
  submódulo.

### Documentos mínimos do repo de docs

| Arquivo | Conteúdo |
|---|---|
| `documents/requirements.md` | requisitos funcionais e de negócio |
| `documents/use-cases.md` | fluxos, pré-condições, regras |
| `documents/architecture.md` | decisões de arquitetura e seus porquês |
| `documents/models/Database_ER.mmd` | modelo ER (Mermaid) |
| `plano-de-testes/README.md` | estratégia de teste (duas camadas) |
| `plano-desenvolvimento.md` | o que construir, em que ordem, e por qual agente |
| `agentes/` | os agentes de desenvolvimento e o porquê do roster |
| `openapi/openapi.yaml` | contrato da API |

---

## 2. Backend — Spring Boot

### 2.1 Estrutura: módulos de negócio, não camadas técnicas

Pacote raiz por **módulo de domínio**; camadas técnicas *dentro* do módulo:

```
src/main/.../<app>/
├── <modulo>/                  # ex.: events, tracks, submissions
│   ├── controller/            # HTTP + DTO apenas — zero lógica de negócio
│   ├── usecase/               # 1 classe por caso de uso: XxxUseCase.execute(...)
│   ├── domain/                # componentes de domínio reutilizáveis entre use cases
│   ├── dto/                   # records de entrada/saída
│   └── persistence/
│       ├── XxxRepository.java
│       └── model/             # entidades JPA (Rich Entity)
├── config/                    # SecurityConfig, seeds, beans globais
└── shared/
    ├── exception/             # ApiExceptionHandler + exceções de negócio
    └── <infra transversal>    # email, texto, etc.
```

Regras que sustentam essa estrutura:

- **Um use case por classe** (`CreateTrackUseCase`, não `TrackService` com 15
  métodos). Nome sempre `*UseCase`; nunca `*Service`.
- **Controller magro**: recebe DTO validado, delega ao use case, devolve DTO.
  Um controller por recurso REST — se acumular 3 recursos, dividir.
- **Rich Entity**: transição de estado vive na entidade
  (`event.approve()`, `event.requireApproved()`), não espalhada nos use cases.
  Entidade tem factory estático `Entity.create(...)` + `@PrePersist`.
- **DTOs em `record`**: response com `ResponseDTO.from(entity)`; create com
  Bean Validation obrigatório; update (PATCH) com todos os campos nullable
  (`null` = não alterar).
- Módulo pode depender de `domain/` de outro módulo (ex.:
  `ApprovedEventLoader`), nunca do `usecase/` ou `controller/` alheio.

### 2.2 Multi-tenancy por contexto

- Recurso pertencente a um "contexto" (evento, organização…): o id do contexto
  vai **sempre no header** (`X-Context-Id`), nunca na URL.
- Todo use case contextual segue o mesmo prólogo:
  1. carregar e validar o contexto (`approvedEventLoader.load(eventId)`);
  2. verificar que o recurso pertence ao contexto
     (`entity.getEventId().equals(eventId)` → senão, 404);
  3. antes de deletar, verificar dependentes (`existsBy...`) → 409.

### 2.3 Segurança e IAM

- Token opaco em `Authorization: Bearer`, sessão stateless, filtro de
  autenticação antes do `UsernamePasswordAuthenticationFilter`.
- Autorização declarativa com `@PreAuthorize` no controller
  (`hasRole('CHAIR') or hasAuthority('TRACK_MANAGE')`) — roles de contexto
  resolvidas pelo `X-Context-Id`.
- `authenticationEntryPoint` → 401; `accessDeniedHandler` → 403 (sem o handler,
  negação sai como 401 e o frontend confunde com sessão expirada).
- `permitAll` explícito e enumerado (login, registro, confirmação, convites);
  `anyRequest().authenticated()` como default.
- Recursos de desenvolvimento (H2 console, frameOptions off) **apenas no
  perfil `dev`**.

### 2.4 Erros: um formato, uma tabela

- Corpo de erro único em toda a API — **RFC 9457 Problem Details**
  (`ProblemDetail` do Spring): `{ "status", "title", "detail" }`.
- `ApiExceptionHandler` central (`@RestControllerAdvice`) com a tabela
  exceção → HTTP documentada:

| Exceção | HTTP |
|---|---|
| `ResourceNotFoundException` | 404 |
| Bean Validation (`MethodArgumentNotValid…`) | 400 |
| Regra de negócio violada (estado inválido) | 422 |
| Conflito / vínculo impede operação | 409 |
| Token de fluxo expirado/consumido | 410 |
| **catch-all `Exception`** | 500 genérico + stack trace no log |

- Exceção de negócio nova = classe em `shared/exception/` + linha no handler +
  linha nesta tabela no `architecture.md` do projeto.
- "Não encontrado" **nunca** via `IllegalArgumentException` → 400.

### 2.5 Banco, perfis e migrações

- **Flyway desde o primeiro dia**: `V1__baseline.sql` e uma migração por
  mudança de schema. `ddl-auto: validate` (prod), `create-drop` (test).
- Perfis:
  - `application.yml` — comum (nome, mail, regras de negócio);
  - `application-dev.yml` — H2, console, log DEBUG;
  - `application-prod.yml` — PostgreSQL via env (`DB_URL`, `DB_USER`,
    `DB_PASSWORD`), log INFO, actuator health.
- Timezone JDBC fixo em UTC. Segredos só via variável de ambiente, com default
  vazio no yml.
- `spring-boot-starter-actuator` com `/actuator/health` exposto para o
  healthcheck do container.

### 2.6 Testes

**Duas camadas, não quatro** — detalhe e justificativa de cada corte em
[`plano-de-testes/README.md`](plano-de-testes/README.md).

- **Integração como espinha dorsal**: `BaseIntegrationTest` com
  `@SpringBootTest + @AutoConfigureMockMvc + @ActiveProfiles("test") +
  @Transactional`, e helpers de alto nível (criar usuário autenticado, criar
  usuário com contexto, request com token e com contexto). É esse arquivo que
  decide se os próximos N testes são baratos.
- Unitário para a lógica de entidade (Rich Entity); integração para o fluxo
  controller→banco. Nomenclatura: `should_x_when_y`.
- **Obrigatório por feature com recurso contextual: o teste de isolamento** —
  usuário do contexto A recebe 404 no recurso do contexto B.
- **Em teste de cálculo, o esperado é literal conferido à mão**, nunca
  calculado com a mesma expressão do código. Agregação se testa pela
  decomposição, não pelo total.
- **Sem gate de cobertura.** O relatório pode ser gerado para consulta, mas não
  quebra o build: gate de porcentagem produz teste escrito para o número.
- **Sem camada intermediária com Mockito, sem Testcontainers e sem matriz de
  rastreabilidade** (`TC-X.Y`). Os três custam manutenção sem pegar bug — a
  primeira testa o mesmo que a integração, a segunda só se paga quando um fluxo
  precisar do PostgreSQL de verdade, e a terceira cobra um commit por teste
  escrito.

---

## 3. Frontend — React + TypeScript

### 3.1 Estrutura feature-based

```
src/
├── app/         # bootstrap: App, router, providers (QueryClient, ErrorBoundary)
├── pages/       # 1 arquivo por rota — só monta features, sem lógica
├── features/
│   └── <feature>/
│       ├── components/   # visuais da feature (1 componente por arquivo)
│       ├── hooks/        # estado e orquestração
│       ├── services/     # chamadas HTTP — sem JSX, sem React
│       ├── types/        # tipos do contrato
│       └── index.ts      # API pública da feature
└── shared/
    ├── components/  # Button, Input, layouts
    ├── services/    # apiClient (axios) único
    ├── types/ utils/
```

- Feature importa de outra feature **apenas via `index.ts`**.
- `pages/` não contém regra de negócio; `services/` não contém JSX.
- **Limite de tamanho: componente com mais de ~250 linhas ou mais de um papel
  (lista + modal + formulário) é dividido.** Sem esse limite, cada feature
  tende a virar um `Manager` de 900 linhas.

### 3.2 Cliente HTTP e sessão

Um único `apiClient` (axios) em `shared/services/` com:

- interceptor de **request**: injeta `Authorization: Bearer` e remove o
  `Content-Type` quando o body é `FormData` (o browser define o boundary);
- interceptor de **response**: `401` → limpar credenciais e redirecionar para
  o login — a expiração de sessão é tratada uma vez, não por tela;
- header de contexto (`X-Context-Id`) passado explicitamente pelos services
  das rotas contextuais.

### 3.3 Estado

- **Estado de servidor → TanStack Query.** Cache, loading, erro, retry e
  invalidação não se reimplementam à mão em `useState`/`useEffect`
  (padrão sujeito a race conditions e sem cache). Query keys por feature:
  `['submissions', contextId]`.
- **Estado de UI → local** (`useState`) ou contexto pequeno quando realmente
  compartilhado. Nada de store global por padrão.
- Composições caras de N chamadas no cliente (N+1) indicam endpoint agregado
  faltando no backend — resolver no contrato, não com mais `Promise.all`.

### 3.4 Rotas

- Objetos de rota (`createBrowserRouter`) com **rotas-layout**: um
  `ProtectedLayout` pai para tudo autenticado (guard + casca), layouts
  aninhados por seção (`/admin`, `/events/:id/*`) com `<Outlet/>`.
  Guard escrito uma vez, não repetido rota a rota.
- Caminhos em kebab-case; público (`/login`, `/register`) fora do layout
  protegido.

### 3.5 Qualidade

- ESLint + `typescript-eslint` estritos desde o início; build = `tsc -b` +
  bundler.
- **Vitest + Testing Library + MSW** no template do projeto (não adiar):
  utils puros → hooks → fluxos críticos, nessa ordem.
- Design tokens do cliente declarados no tema do Tailwind
  (`@theme` / `tailwind.config`) — nunca hex inline em `className`.
- `ErrorBoundary` no root + mecanismo único de notificação (toast) em
  `shared/`.

---

## 4. Contrato e integração

- `openapi.yaml` é escrito **antes** do código nas features novas — inclusive
  antes de o backend existir: quando o frontend é construído primeiro contra
  um mock (telas para validação com stakeholders antes de comprometer
  backend), esse arquivo hand-written é o único contrato disponível e serve
  de andaime tanto para o mock quanto para a assinatura que o backend vai
  implementar depois.
- **Esse arquivo tem vida útil limitada.** Assim que o backend implementa o
  contrato, `springdoc-openapi` passa a gerar a especificação a partir do
  código real (`/v3/api-docs` + Swagger UI) — e essa versão gerada assume o
  papel de fonte única de verdade dali em diante. Não se mantêm as duas
  versões indefinidamente por projeto: manter um `openapi.yaml` hand-written
  paralelo ao springdoc depois que o backend existe só cria risco de
  divergência sem necessidade — quando o contrato precisar mudar, a mudança
  é feita no código do backend (DTO/controller), não no yaml. O
  `openapi.yaml` original pode ser arquivado/removido nesse ponto.
- Types do frontend gerados com `openapi-typescript`, apontando sempre para o
  contrato vigente: o `openapi.yaml` hand-written enquanto o backend ainda
  não existe; o `/v3/api-docs` do springdoc a partir do momento em que o
  backend estiver no ar — trocar a fonte é o gatilho do corte acima.
- Convenções fixas do contrato:
  - autenticação: `Authorization: Bearer <token opaco>`;
  - contexto: `X-Context-Id: <uuid>` em header, nunca na URL;
  - erros: Problem Details (seção 2.4);
  - coleções com potencial de crescimento: paginadas (`page`, `size`,
    envelope `{ content, page, totalElements }`).

---

## 5. CI/CD e operação

- Cada repo de código: workflow de **build + test** (PR) e **build + publish
  de imagem Docker** (main/tag). Backend roda `./mvnw test`; frontend roda
  `npm run lint && npm run build && npm test`.
- Deploy: containers com healthcheck no `/actuator/health`; frontend servido
  por nginx com proxy `/v1` para a API (mesma origem ⇒ sem CORS).
- Variáveis de ambiente de cada serviço documentadas no README de deploy.

---

## 6. Convenções de linguagem (compartilhadas)

- **Código em inglês** (classes, variáveis, arquivos, rotas);
  **comentários e strings de usuário em português**.
- Comentário existe para registrar restrição ou decisão não óbvia — não para
  narrar o código.
- Endpoints REST no plural. Booleans com `is/has/can`. Handlers com `handle*`.

---

## 7. Checklist para iniciar um projeto novo

1. Criar os três repos, irmãos. **Não submodular docs** (ver §1).
2. Docs: `requirements.md`, `use-cases.md`, `architecture.md`, ER,
   `openapi.yaml` esqueleto.
3. Backend: skeleton com `config/`, `shared/exception/` (handler + Problem
   Details + catch-all), Flyway `V1`, perfis dev/test/prod, actuator,
   `BaseIntegrationTest`.
4. Frontend: skeleton com `apiClient` (2 interceptors), router com
   `ProtectedLayout`, QueryClient, ErrorBoundary, tokens de design no tema,
   Vitest configurado com 1 teste de exemplo.
5. CI dos três fluxos (build+test, docker, sync de branches) desde o commit
   inicial.
6. `CLAUDE.md` por repo, com as convenções específicas da stack daquele repo.
7. **Sistema de agentes:** instalar os agentes de [`agentes/`](agentes/) nos
   repos de código e registrar `permissions.additionalDirectories` apontando
   para o repo de docs, senão os agentes não leem a documentação.
8. **`plano-desenvolvimento.md`:** as fases, com o agente e o critério de pronto
   em cada passo. É ele que serve de briefing aos agentes — um agente novo
   começa sem contexto nenhum, e o plano é o que o substitui.
