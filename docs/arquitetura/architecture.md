# Arquitetura

**Sistema de Gestão de Biotério**

> Este documento segue o padrão fullstack descrito em
> [`../reference-architecture.md`](../reference-architecture.md) (arquitetura de
> referência do time, extraída do SATAC). Onde o sistema diverge do padrão, a
> divergência e o motivo estão explícitos no texto — o restante deve ser lido
> como aplicação direta do padrão de referência.

## Containers

As unidades de implantação do sistema: quais processos rodam, quais tecnologias utilizam e como se comunicam entre si.

| Container | Tecnologia | Responsabilidade |
|---|---|---|
| **Aplicação Web** | React (SPA), PWA | Renderiza a interface no navegador/celular do usuário (pesquisador, aluno, veterinário, administrador, auditor CEUA). Instalável como PWA, com fila local de sincronização para tolerar conectividade instável dentro do biotério (RNF02). Toda interação passa por chamadas à API Backend. |
| **API Backend** | Spring Boot / REST | Concentra toda a lógica de negócio: cadastro de animais, gestão de caixas/gaiolas, registro imutável de procedimentos, protocolos CEUA, vínculos aluno–pesquisador, relatórios e alertas. Autentica requisições via o módulo IAM embarcado, persiste dados no banco e aciona o serviço de e-mail quando necessário. |
| **Módulo IAM** | Biblioteca Java (dependência Maven, embarcada na API Backend) | Gerencia autenticação, RBAC (perfis pesquisador, aluno, veterinário, administrador, auditor CEUA) e auditoria de segurança. Não é um serviço externo — roda dentro do processo da API Backend. Ver [`IAM.md`](./IAM.md) para o guia de integração. |
| **Banco de Dados** | PostgreSQL | Armazena o estado persistente do sistema: animais, caixas, procedimentos, protocolos, vínculos aluno–pesquisador, usuários/perfis e logs de auditoria. Backup automático (RNF03). |
| **Serviço de E-mail** | Externo (SMTP / API) | Envia e-mails transacionais: recuperação de senha e notificações de alertas (protocolo vencendo, caixa superlotada, uso de animais próximo do limite — RF08). |

## Componentes

Os componentes abaixo correspondem aos módulos internos da API Backend. Cada módulo encapsula as regras de negócio de um contexto específico do sistema, alinhado aos requisitos funcionais em [`../requisitos/requirements.md`](../requisitos/requirements.md).

| Módulo | Responsabilidade | Requisitos relacionados |
|---|---|---|
| **animais** | Cadastro de animais, histórico de caixas e status (vivo, em experimento, eutanasiado, óbito, transferido), vínculo com protocolo(s). | RF01 |
| **caixas** | Cadastro e localização de caixas/gaiolas, ocupação, alerta de superlotação, histórico de ocupantes. | RF02 |
| **procedimentos** | Registro **imutável** de procedimentos realizados em animais, com correções feitas como novos registros vinculados ao original. | RF03 |
| **protocolos** | Protocolos/projetos CEUA: aprovação, validade, quantidade de animais aprovados, alertas de vencimento e de uso próximo do limite. | RF04 |
| **vinculos** | Vínculo aluno–pesquisador (1 ativo por vez), histórico de vínculos encerrados, resolução do escopo de dados do aluno. | RF05 |
| **relatorios** | Geração de relatórios por protocolo, relatório anual CEUA/CONCEA, histórico de animal, vínculos por pesquisador. | RF07 |
| **alertas** | Agregação de alertas entre módulos (procedimentos pendentes, protocolo vencendo, superlotação, limite de animais) para exibição e notificação. | RF08 |
| **shared** | Código transversal: configuração de segurança, tratamento global de exceções e utilitários. | RNF03, RNF05 |

O controle de perfil de acesso (RF05, parte de usuários e perfis) e o log de auditoria (RF06) são resolvidos pelo módulo IAM embarcado, descrito em [`IAM.md`](./IAM.md), e não por um módulo de negócio próprio — com exceção do vínculo aluno–pesquisador em si (regra de negócio específica do domínio, não genérica de IAM), que vive no módulo **vinculos**.

## Decisões de Arquitetura

### Autenticação e Perfis de Acesso (IAM)

O sistema utiliza o **módulo IAM** como biblioteca (dependência Maven) embarcada na API Backend — não como um serviço externo. O módulo resolve autenticação (login/token opaco), RBAC (roles e permissions) e auditoria de segurança de forma automática.

**Perfis (roles) do sistema:**

| Role | Perfil de negócio | Uso típico |
|---|---|---|
| `ROLE_PESQUISADOR` | Pesquisador responsável por projeto(s)/protocolo(s) CEUA | Cadastra animais, registra procedimentos, gerencia seus protocolos, cria/encerra vínculos com alunos, gera relatórios. |
| `ROLE_ALUNO` | Aluno vinculado a um pesquisador | Cadastra animais e registra procedimentos, com visão restrita aos dados do pesquisador ao qual está **atualmente** vinculado (ver decisão de escopo abaixo). |
| `ROLE_VETERINARIO` | Veterinário / responsável técnico do biotério | Gerencia caixas/gaiolas, registra procedimentos (ex.: eutanásia, atendimento), acesso irrestrito a todos os protocolos por responsabilidade técnica sobre o biotério. |
| `ROLE_ADMINISTRADOR` | Administrador do sistema | Gerencia usuários e perfis, vínculos aluno–pesquisador de qualquer par, configura alertas, acesso total de escrita. |
| `ROLE_AUDITOR_CEUA` | Auditor da comissão de ética (CEUA) | Acesso de **leitura** a tudo, incluindo a trilha de auditoria — nenhuma permission de escrita. |

O sistema **não utiliza o recurso de contexto multi-tenant** (`X-Context-Id`) do módulo IAM — todas as roles são atribuídas de forma global ao usuário. O suporte a contexto existe na biblioteca, mas fica sem uso neste projeto. O escopo de dados do aluno (por vínculo ativo) é resolvido em nível de aplicação, não pelo `X-Context-Id` do IAM — ver decisão dedicada abaixo.

**Regra:** todo endpoint que exige perfil específico deve ser protegido com `@PreAuthorize`, usando as permissions granulares (ex.: `PROCEDIMENTO_REGISTRAR`, `PROTOCOLO_GERENCIAR`) descritas em [`IAM.md`](./IAM.md).

**Detalhes de segurança na configuração do Spring:**

- Token opaco em `Authorization: Bearer`, sessão stateless (`SessionCreationPolicy.STATELESS`) — o `IamTokenAuthFilter` roda antes do `UsernamePasswordAuthenticationFilter` (ver [`IAM.md §4.6`](./IAM.md#46-configurar-spring-security)).
- `authenticationEntryPoint` configurado para devolver 401 (não autenticado) e `accessDeniedHandler` para devolver 403 (autenticado, sem permissão). Sem o `accessDeniedHandler`, uma negação de permissão sai como 401 e o frontend confunde com sessão expirada, disparando um redirect de login indevido.
- **Sem cadastro público.** O sistema não expõe auto-registro: contas são criadas por um Administrador (RF05). `permitAll` explícito e enumerado (`/v1/auth/login`, `/v1/confirmation/**` para os fluxos de recuperação de senha) — `/v1/users` exige `ROLE_ADMINISTRADOR`. Todo o resto cai em `anyRequest().authenticated()` por padrão.
- Recursos de desenvolvimento (console do H2, `frameOptions` desabilitado) habilitados **apenas no perfil `dev`** — nunca em `prod`.

### Escopo de dados por vínculo aluno–pesquisador

Regra central de RF05: um aluno só enxerga os animais, procedimentos e protocolos do pesquisador ao qual está **atualmente** vinculado (vínculo ativo); ao trocar de pesquisador, perde a visibilidade dos dados do vínculo anterior — mesmo dos registros que ele próprio criou. Ao mesmo tempo, os registros de procedimento feitos pelo aluno **nunca são apagados ou reatribuídos**: permanecem visíveis para o pesquisador responsável e para o auditor CEUA.

Essa é uma regra de negócio do domínio, não um mecanismo genérico de multi-tenancy — por isso **não** é implementada com o `X-Context-Id` do módulo IAM (que representaria um contexto arbitrário, não a semântica específica de "vínculo ativo com um pesquisador"). Em vez disso:

1. O módulo `vinculos` expõe um componente de domínio (`VinculoAtivoResolver`) que resolve o `pesquisadorId` do vínculo ativo de um `ROLE_ALUNO` a partir do `IamContext.getPrincipal()`.
2. Todo use case de leitura/escrita em `animais`, `procedimentos` e `protocolos` que atende um `ROLE_ALUNO` aplica esse `pesquisadorId` como filtro obrigatório — via dependência de `domain/` do módulo `vinculos`, nunca duplicando a regra em cada módulo.
3. Encerrar um vínculo (`VINCULO_ENCERRAR`) não apaga nem reatribui os registros já criados pelo aluno; apenas marca o vínculo como encerrado, o que remove a permissão de leitura futura para esse aluno sobre os dados daquele pesquisador. O histórico do próprio vínculo (quem esteve vinculado a quem, e quando) é preservado para auditoria (RF06).

### Imutabilidade de procedimentos

RF03 exige que um procedimento salvo nunca seja editado ou apagado — requisito central para credibilidade em auditoria CEUA/CONCEA (RNF05). Consequência arquitetural: o recurso `procedimentos` **não tem endpoint de `PUT`/`PATCH`/`DELETE`**. Uma correção é modelada como um novo registro (`POST /v1/procedimentos`) com um campo `corrigeProcedimentoId` apontando para o registro original, que permanece intacto. A entidade `Procedimento` (Rich Entity) expõe apenas `Procedimento.create(...)` e `Procedimento.corrigir(original, ...)` — nunca um setter de campo já persistido.

### Erros — formato único

Toda a API responde erros no mesmo formato — **RFC 9457 Problem Details** (`ProblemDetail`, nativo do Spring 6+): `{ "status", "title", "detail" }`. Um `ApiExceptionHandler` central (`@RestControllerAdvice` em `shared/exception/`) concentra a tabela exceção → HTTP:

| Exceção | HTTP |
|---|---|
| `ResourceNotFoundException` (ex.: animal, caixa, procedimento ou protocolo inexistente) | 404 |
| Bean Validation (`MethodArgumentNotValidException`) | 400 |
| Regra de negócio violada (ex.: tentativa de alterar procedimento já salvo, protocolo vencido usado em novo procedimento) | 422 |
| Conflito / vínculo impede a operação (ex.: caixa já no limite de capacidade, aluno já com vínculo ativo) | 409 |
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
- Backup automático do banco de produção (RNF03) — dados de pesquisa não podem ser perdidos; rotina de backup e teste de restauração documentados no README de deploy do repositório de backend.
- `spring-boot-starter-actuator` com `/actuator/health` exposto para o healthcheck do container (ver [`../cicd/CICD.md`](../cicd/CICD.md)).
- **Exceção documentada:** o módulo IAM embarcado cria e gerencia suas próprias tabelas (`iam_*`) via `ddl-auto` da biblioteca, fora do controle do Flyway do sistema — ver [`IAM.md §2.3`](./IAM.md#23-configurar-banco-de-dados). O Flyway do sistema versiona apenas o schema dos módulos de negócio (animais, caixas, procedimentos, protocolos, vinculos).

### Testes

- **Integração como espinha dorsal**: `BaseIntegrationTest` com `@SpringBootTest + @AutoConfigureMockMvc + @ActiveProfiles("test") + @Transactional`, com helpers de alto nível (ex.: `createPesquisadorAndGetToken`, `performWithToken`) para não repetir o setup de autenticação em cada teste.
- Teste unitário para `domain/` e lógica de entidade (Rich Entity) — cobertura obrigatória para a regra de imutabilidade de `procedimentos` e para o `VinculoAtivoResolver`; teste de integração para o fluxo controller → banco por caso de uso.
- Nomenclatura: `should_x_when_y` nos testes unitários; `@DisplayName("TC-X.Y: …")` amarrando ao caso de teste documentado em [`../plano-de-testes/`](../plano-de-testes/).
- JaCoCo no build com gate de cobertura sobre `usecase/` e `domain/` — DTOs e classes de configuração ficam fora do gate.

### Backend

O backend adota a abordagem **Feature-based + Layered**: o código é organizado por funcionalidade de negócio (animais, caixas, procedimentos, protocolos, vinculos, relatorios, alertas), e dentro de cada funcionalidade as responsabilidades são separadas em camadas bem definidas. Isso facilita a localização do código, limita o acoplamento entre módulos e torna o sistema mais fácil de manter e evoluir.

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
| **usecase/** | Uma classe por caso de uso (ex.: `RegistrarProcedimentoUseCase`, nunca um `ProcedimentoService` genérico com vários métodos). Nome sempre `*UseCase`, nunca `*Service`. Orquestra chamadas ao repositório e à camada `domain/`. |
| **domain/** | Regras de domínio reutilizadas por mais de um use case do módulo, ou por outro módulo (ex.: `VinculoAtivoResolver` de `vinculos`, consultado por `animais` e `procedimentos`; cálculo de ocupação de uma caixa). Fica vazia em módulos simples — só é criada quando a duplicação aparece. Um módulo pode depender do `domain/` de outro módulo, nunca do `usecase/` ou `controller/` alheio. |
| **dto/** | Objetos de transferência de dados usados na comunicação entre Controller e UseCase, implementados como `record`. DTO de resposta expõe um factory estático (`XxxResponseDTO.from(entity)`); DTO de criação usa Bean Validation obrigatória; DTO de atualização (PATCH) tem todos os campos anuláveis — `null` significa "não alterar". `procedimentos` não tem DTO de atualização, por não ter endpoint de edição (ver decisão de imutabilidade). |
| **persistence/** | Repositórios JPA. Não contém lógica de negócio — apenas operações de leitura e escrita. |
| **persistence/model/** | Entidades JPA que mapeiam as tabelas do banco de dados, seguindo o padrão *Rich Entity*: transições de estado vivem na própria entidade (ex.: `animal.transferirPara(caixa)`, `vinculo.encerrar()`) em vez de espalhadas pelos use cases. Entidade expõe factory estático `Entity.create(...)` e `@PrePersist` para os campos derivados. |

**Fluxo de dados**

```
Request (DTO) → Controller → UseCase → [Domain] → Repository → Banco de Dados
```

**Estrutura de pastas**

Cada módulo de negócio é autossuficiente — contém seu controller, casos de uso, entidades, repositório e DTOs organizados em subpastas. Código genuinamente transversal (exceções globais, configurações de segurança, utilitários) fica no pacote `shared`.

```
src/main/java/br/bioterio/
│
├── animais/            ← cadastro e histórico de animais (RF01)
│   ├── controller/
│   │   └── AnimalController.java
│   ├── usecase/
│   │   └── AnimalUseCase.java
│   ├── dto/
│   │   ├── AnimalRequestDTO.java
│   │   └── AnimalResponseDTO.java
│   └── persistence/
│       ├── AnimalRepository.java
│       └── model/
│           └── Animal.java
│
├── caixas/             ← caixas/gaiolas, ocupação, superlotação (RF02)
│   ├── controller/
│   │   └── CaixaController.java
│   ├── usecase/
│   │   └── CaixaUseCase.java
│   ├── domain/                        ← cálculo de ocupação/alerta de superlotação
│   │   └── OcupacaoCaixaCalculator.java
│   ├── dto/
│   │   ├── CaixaRequestDTO.java
│   │   └── CaixaResponseDTO.java
│   └── persistence/
│       ├── CaixaRepository.java
│       └── model/
│           └── Caixa.java
│
├── procedimentos/      ← registro imutável de procedimentos (RF03)
│   ├── controller/
│   │   └── ProcedimentoController.java
│   ├── usecase/
│   │   ├── RegistrarProcedimentoUseCase.java
│   │   └── CorrigirProcedimentoUseCase.java
│   ├── dto/
│   │   ├── ProcedimentoRequestDTO.java
│   │   └── ProcedimentoResponseDTO.java
│   └── persistence/
│       ├── ProcedimentoRepository.java
│       └── model/
│           └── Procedimento.java
│
├── protocolos/         ← protocolos/projetos CEUA (RF04)
│   ├── controller/
│   │   └── ProtocoloController.java
│   ├── usecase/
│   │   └── ProtocoloUseCase.java
│   ├── domain/                        ← regra de limite de animais aprovados
│   │   └── LimiteAnimaisChecker.java
│   ├── dto/
│   │   ├── ProtocoloRequestDTO.java
│   │   └── ProtocoloResponseDTO.java
│   └── persistence/
│       ├── ProtocoloRepository.java
│       └── model/
│           └── Protocolo.java
│
├── vinculos/           ← vínculo aluno–pesquisador (RF05)
│   ├── controller/
│   │   └── VinculoController.java
│   ├── usecase/
│   │   ├── CriarVinculoUseCase.java
│   │   └── EncerrarVinculoUseCase.java
│   ├── domain/                        ← consultado por animais, procedimentos e protocolos
│   │   └── VinculoAtivoResolver.java
│   ├── dto/
│   │   └── VinculoResponseDTO.java
│   └── persistence/
│       ├── VinculoRepository.java
│       └── model/
│           └── Vinculo.java
│
├── relatorios/         ← relatórios por protocolo, anual CEUA, histórico de animal (RF07)
│   ├── controller/
│   │   └── RelatorioController.java
│   ├── usecase/
│   │   └── RelatorioUseCase.java
│   └── dto/
│       └── RelatorioResponseDTO.java
│
├── alertas/            ← agregação de alertas entre módulos (RF08)
│   ├── controller/
│   │   └── AlertaController.java
│   ├── usecase/
│   │   └── AlertaUseCase.java
│   └── dto/
│       └── AlertaResponseDTO.java
│
└── shared/             ← código compartilhado entre módulos
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
│   │
│   ├── animais/
│   │   ├── components/
│   │   │   ├── AnimalForm.tsx
│   │   │   └── AnimalCard.tsx
│   │   ├── hooks/
│   │   │   └── useAnimal.ts
│   │   ├── services/
│   │   │   └── animalService.ts     ← chamadas HTTP (axios/fetch)
│   │   ├── types/
│   │   │   └── animal.types.ts
│   │   └── index.ts
│   │
│   ├── caixas/
│   │   ├── components/
│   │   │   ├── CaixaForm.tsx
│   │   │   ├── CaixaCard.tsx
│   │   │   └── LeitorQrCaixa.tsx    ← leitura de QR/código de barras (RNF01)
│   │   ├── hooks/
│   │   │   └── useCaixa.ts
│   │   ├── services/
│   │   │   └── caixaService.ts
│   │   └── types/
│   │       └── caixa.types.ts
│   │
│   ├── procedimentos/
│   │   ├── components/
│   │   │   ├── ProcedimentoForm.tsx
│   │   │   └── HistoricoProcedimentos.tsx
│   │   ├── hooks/
│   │   │   └── useProcedimento.ts
│   │   ├── services/
│   │   │   └── procedimentoService.ts
│   │   └── types/
│   │       └── procedimento.types.ts
│   │
│   ├── protocolos/
│   │   ├── components/
│   │   │   ├── ProtocoloForm.tsx
│   │   │   └── ProtocoloBadge.tsx   ← válido/perto do limite/vencido
│   │   ├── hooks/
│   │   │   └── useProtocolo.ts
│   │   ├── services/
│   │   │   └── protocoloService.ts
│   │   └── types/
│   │       └── protocolo.types.ts
│   │
│   ├── vinculos/
│   │   ├── components/
│   │   │   └── VinculoAlunoPesquisador.tsx
│   │   ├── hooks/
│   │   │   └── useVinculo.ts
│   │   ├── services/
│   │   │   └── vinculoService.ts
│   │   └── types/
│   │       └── vinculo.types.ts
│   │
│   ├── relatorios/
│   │   ├── components/
│   │   │   └── RelatorioExport.tsx
│   │   ├── hooks/
│   │   │   └── useRelatorio.ts
│   │   ├── services/
│   │   │   └── relatorioService.ts
│   │   └── types/
│   │       └── relatorio.types.ts
│   │
│   └── alertas/
│       ├── components/
│       │   └── PainelAlertas.tsx
│       ├── hooks/
│       │   └── useAlertas.ts
│       ├── services/
│       │   └── alertaService.ts
│       └── types/
│           └── alerta.types.ts
│
├── shared/                          ← compartilhado entre features
│   ├── components/
│   │   ├── Button.tsx
│   │   └── Modal.tsx
│   ├── hooks/
│   │   └── useDebounce.ts
│   ├── offline/                     ← fila de sincronização offline (RNF02)
│   │   └── syncQueue.ts
│   └── types/
│       └── api.types.ts
│
├── pages/                           ← só monta as features na tela
│   ├── AnimaisPage.tsx
│   ├── CaixasPage.tsx
│   ├── ProcedimentosPage.tsx
│   ├── ProtocolosPage.tsx
│   ├── VinculosPage.tsx
│   ├── RelatoriosPage.tsx
│   └── AlertasPage.tsx
│
└── app/
    ├── App.tsx
    └── routes.tsx
```

**Limite de tamanho:** componente com mais de ~250 linhas ou mais de um papel (ex.: lista + modal + formulário no mesmo arquivo) é dividido. Sem esse limite, uma feature tende a virar um único componente `Manager` monolítico conforme cresce.

**PWA e funcionamento offline (RNF02)**

- Service worker com estratégia *cache-first* para o shell da aplicação e *network-first* para dados; formulários de registro rápido (procedimento, cadastro de animal) gravam localmente (IndexedDB) quando offline e entram na `shared/offline/syncQueue.ts`, sincronizada assim que a conexão volta.
- Conflito de sincronização (ex.: caixa que ficou lotada enquanto o registro estava na fila) é resolvido no backend pelas mesmas regras de negócio do fluxo online (422/409 da tabela de erros) — o cliente reapresenta o erro ao usuário para correção manual, nunca sobrescreve silenciosamente.

**Cliente HTTP e sessão**

Um único `apiClient` (axios) em `shared/services/` (ou `shared/api/`) com:

- interceptor de **request**: injeta `Authorization: Bearer <token>` e remove o `Content-Type` quando o body é `FormData` (o browser define o boundary);
- interceptor de **response**: `401` → limpa as credenciais e redireciona para o login — a expiração de sessão é tratada uma vez no interceptor, não repetida tela a tela.

O sistema não usa o header de contexto (`X-Context-Id`) do módulo IAM — ver [decisão de multi-tenancy](#autenticação-e-perfis-de-acesso-iam) acima.

**Estado**

- **Estado de servidor → TanStack Query.** Cache, loading, erro, retry e invalidação de cada chamada HTTP (animais, caixas, procedimentos, protocolos) não são reimplementados à mão em `useState`/`useEffect` — esse padrão é sujeito a race conditions e não tem cache. Query keys por feature (ex.: `['procedimentos', animalId]`).
- **Estado de UI → local** (`useState`) ou contexto pequeno quando realmente compartilhado entre componentes. Sem store global por padrão.
- Composições caras de N chamadas no cliente (ex.: montar o painel de alertas buscando indicadores um a um) indicam endpoint agregado faltando no backend — resolver no contrato (`/v1/alertas`), não empilhando `Promise.all` no frontend.

**Rotas**

- Objetos de rota (`createBrowserRouter`) com rotas-layout: um `ProtectedLayout` pai para tudo que exige autenticação (guard de sessão + casca visual comum), com `<Outlet/>` para as páginas de cada perfil (pesquisador, aluno, veterinário, administrador, auditor CEUA). Guard escrito uma vez, não repetido rota a rota.
- Caminhos em kebab-case; rotas públicas (`/login`, `/recuperar-senha`) ficam fora do layout protegido.

**Qualidade**

- ESLint + `typescript-eslint` estritos desde o início; build = `tsc -b` + Vite.
- **Vitest + Testing Library + MSW**, na ordem utils puros → hooks → fluxos críticos (login, registro de procedimento, transferência entre caixas, encerramento de vínculo).
- Design tokens do design system (ver [`../design-system/design-system.md`](../design-system/design-system.md)) declarados no tema do Tailwind — nunca hex inline em `className`.
- `ErrorBoundary` no root da aplicação + mecanismo único de notificação (toast) em `shared/`.

---

## Contrato de API (OpenAPI)

- `openapi.yaml` é escrito **antes** do código das features novas — request/response nascem no contrato, não no controller. Ainda não existe neste repositório; deve ser criado seguindo o checklist do [`reference-architecture.md §7`](../reference-architecture.md#7-checklist-para-iniciar-um-projeto-novo) quando o backend for iniciado.
- O backend expõe `springdoc-openapi`; a divergência entre o spec efetivo e o `openapi.yaml` versionado é verificada no CI (ou em checklist de release).
- Types do frontend gerados a partir do contrato com `openapi-typescript`, eliminando a triplicação spec × DTO Java × `*.types.ts` mantidos manualmente.
- Convenções fixas do contrato do sistema:
  - autenticação: `Authorization: Bearer <token opaco>`;
  - erros: Problem Details (ver seção de erros acima);
  - coleções com potencial de crescimento (ex.: listagem de procedimentos, animais): paginadas (`page`, `size`, envelope `{ content, page, totalElements }`).
  - o sistema **não** usa `X-Context-Id` — omitido do contrato.
