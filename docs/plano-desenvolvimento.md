# Plano de desenvolvimento

> **Esqueleto de template.** Substitua os placeholders e apague esta citação. O formato não é decorativo: ele é o **briefing dos agentes** de [`agentes/`](agentes/), que começam sem contexto nenhum e leem daqui o passo pedido e a seção "Decisões e pendências".

**O que construir e em que ordem**, cobrindo todos os requisitos de [`requisitos/requirements.md`](requisitos/requirements.md) e os casos de uso de [`casos-de-uso/`](casos-de-uso/). Cada passo tem um número estável, o agente que o executa e o critério objetivo de pronto.

Quem executa o quê, e por que o roster é esse, está em [`agentes/`](agentes/).

## O que este plano não contém — de propósito

**Este arquivo fica obsoleto quando a última fase terminar.** Por isso ele não guarda nenhuma regra permanente: quem procurar aqui "todo finder carrega o id do contexto" ou "dinheiro é `BigDecimal` escala 2" não encontra, e isso é intencional — essas regras valem para a feature que vier *depois* do plano, e morreriam com ele.

| O que você procura | Onde está |
|---|---|
| Isolamento, dinheiro, estrutura, erros, schema, autenticação, frontend | [`arquitetura/architecture.md`](arquitetura/architecture.md) e [`reference-architecture.md`](reference-architecture.md) |
| O que o sistema deve fazer | [`requisitos/requirements.md`](requisitos/requirements.md) |
| Fluxo e regra de um caso de uso | [`casos-de-uso/`](casos-de-uso/) |
| Como testar, e qual teste prova algo | [`plano-de-testes/README.md`](plano-de-testes/README.md) |
| Cor, tipografia, espaçamento | [`design-system/design-system.md`](design-system/design-system.md) |
| Branch, pipeline, deploy | [`cicd/CICD.md`](cicd/CICD.md) |
| Como o agente trabalha, e quando parar | [`agentes/README.md`](agentes/README.md), seção 4 |

**Aqui ficam só as quatro coisas que são deste plano:** a ordem dos passos, o agente de cada um, o critério de pronto, e as decisões que ainda não foram tomadas.

## Estado de partida

| Repositório | Situação |
|---|---|
| `<nome>` | Esta documentação. Repositório **irmão** dos de código, não submódulo. |
| `<nome>-backend` | <vazio / o que já existe> |
| `<nome>-frontend` | <vazio / o que já existe> |

---

## Decisões e pendências

**Leia esta seção antes de executar qualquer passo.** Se a sua dúvida está nas pendências abertas, ela não é sua para resolver: pare e devolva a pergunta.

### Resolvidas — já aplicadas aos documentos normativos

Registradas para que ninguém reabra a discussão; o texto que vale é o do documento citado.

| # | Decisão | Onde vive agora |
|---|---|---|
| D1 | | |

### Abertas

| # | Pendência | Resolver em |
|---|---|---|
| P1 | | passo **x.y** |

> **Uma pendência aberta bloqueia o passo.** Fase com decisão em aberto não está pronta para ser delegada — está pronta para ser conversada. Prefira registrar a pendência a deixar um agente adivinhar: o palpite dele vai parecer uma decisão depois.

---

## Fase 0 — Fundação

O que todas as features vão herdar. **Vale caprichar:** um erro aqui se multiplica pelo número de features.

| # | O que | Agente | Pronto quando |
|---|---|---|---|
| **0.1** | Projeto Maven + Spring Boot, pacote raiz, dependências. **Instalar os agentes e o `.claude/settings.local.json`** (ver [`agentes/README.md`](agentes/README.md), seção 5) | **sessão principal** — não há agente instalado antes dele | `./mvnw -B verify` verde; um agente consegue ler este plano sem prompt |
| **0.2** | `shared/` — exceções de negócio e o handler central, cobrindo **cada linha** da tabela de erros do `architecture.md` | `devpleno` | Teste que provoca cada linha e confere status e formato |
| **0.3** | Segurança: integração do IAM, filtro, `authenticationEntryPoint` → 401 e `accessDeniedHandler` → 403, `permitAll` enumerado | `devsenior` | Teste: 401 sem token, 401 com token inválido, 403 sem permissão, 200 em rota pública |
| **0.4** | Flyway, perfis `dev`/`test`/`prod`, timezone UTC, actuator | `infra-specialist` | App sobe; healthcheck responde; nenhum segredo com valor real |
| **0.5** | `BaseIntegrationTest` com os helpers de autenticação e de contexto | `devpleno` | Um teste de fumaça por helper. **Decide se os próximos N testes são baratos** |
| **0.6** | `Dockerfile` multi-stage e compose | `infra-specialist` | `docker compose up -d` sobe; healthcheck responde |
| **0.7** | Workflows de CI (ver [`cicd/workflows/`](cicd/workflows/)) | `infra-specialist` | Workflow verde no push |

**Checkpoint** (recomendado) — a fundação é herdada por tudo.

---

## Fase 1 — <nome da fase>

<RFxx, RFyy · UC xx, UC yy>

| # | O que | Agente | Pronto quando |
|---|---|---|---|
| **1.1** | | | |

**Checkpoint** — <obrigatório/recomendado>, e o que se procura primeiro.

---

## Fases seguintes

> Uma fase por agrupamento de requisitos que faça sentido construir junto. Duas heurísticas que se pagaram:
>
> - **O passo que estabelece o isolamento entre contextos vem cedo e tem checkpoint obrigatório.** Tudo depois dele depende de ele estar certo, e nenhuma fase posterior o reexamina.
> - **A primeira feature escrita com a estrutura completa é o piloto, e tem checkpoint imediatamente depois** — antes de o padrão ser repetido mais N vezes. Um erro pego ali é um erro; pego no fim, são N.
>
> Quando o frontend já existe, ele não vai todo no fim: entra em ondas, cada uma depois da fase de backend de que depende. Deixar o app aberto e quebrado durante todo o build do backend não é opção quando já existe um app que funciona.

---

## Cobertura

Checagem de completude **deste plano** — não é matriz de rastreabilidade de teste, que [`plano-de-testes/README.md`](plano-de-testes/README.md) descarta de propósito.

| RF | Backend | Frontend |
|---|---|---|
| RF01 | | |

| UC | Passos | Observação |
|---|---|---|
| UC 01 | | |

---

## Checkpoints

| CP | Depois de | Obrigatório | O que se procura primeiro |
|---|---|---|---|
| CP-0 | Fase 0 | recomendado | `BaseIntegrationTest` e o filtro de autenticação: todas as features vão herdá-los |

---

## Regra geral

**Delega-se execução, não julgamento.** Fase com decisão em aberto não está pronta para ser delegada. É por isso que alguns passos rodam na sessão principal, e por isso um passo bloqueado por pendência fica bloqueado em vez de levar um palpite.
