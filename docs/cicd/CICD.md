# Git Workflow & CI/CD Strategy

**Sistema de Gestão de XXXX**

## Context

O sistema é dividido em dois repositórios independentes (multirepo):

- `xxxx-frontend` — React + TypeScript (ver [`../arquitetura/architecture.md`](../arquitetura/architecture.md))
- `xxxx-backend` — Spring Boot / Java, com o módulo IAM embarcado (ver [`../arquitetura/IAM.md`](../arquitetura/IAM.md))

Cada repositório tem suas próprias branches, pipeline de CI/CD, versionamento e ambiente de deploy. O modelo de branches abaixo é aplicado **identicamente aos dois repositórios**.

Este repositório é o **repositório de docs** do padrão descrito em [`../reference-architecture.md §1`](../reference-architecture.md#1-organização-dos-repositórios): quando `xxxx-backend` e `xxxx-frontend` forem criados, ele deve ser submodulado como `docs/` dentro dos dois. Editar sempre aqui; depois bumpar o ponteiro do submódulo nos repos de código. Cada um dos dois workflows de build+test deve incluir um job que avisa quando esse ponteiro está defasado em relação a este repositório (ver [`reference-architecture.md §5`](../reference-architecture.md#5-cicd-e-operação)).

> Se você está consultando este documento para ajudar com o fluxo de git/CI do sistema, esta é a fonte de verdade sobre como branches, PRs e deploys devem funcionar. Siga as regras aqui a menos que o usuário informe explicitamente que o processo mudou.

---

## Branch Structure

| Branch | Purpose | Protected | Environment |
|---|---|---|---|
| `main` | Código pronto para produção | Sim | Produção |
| `develop` | Branch de integração | Sim | Staging |
| `feature/*` | Novo trabalho | Não | Nenhum (apenas build/test) |
| `hotfix/*` | Correções urgentes de produção | Não | Nenhum (apenas build/test) |

**Conceito-chave:** "Staging" não é uma branch separada — é o ambiente para o qual `develop` faz deploy automaticamente a cada merge. "Produção" é o ambiente para o qual `main` faz deploy.

---

## Feature Branches

**Criada a partir de:** `develop`
**PR direcionado para:** `develop`

```bash
git checkout develop
git pull origin develop
git checkout -b feature/<escopo>-<descricao-curta>
# work, commit
git push origin feature/<escopo>-<descricao-curta>
```

**Convenção de nomenclatura:** `feature/<escopo>-<descricao-curta>`, com escopo alinhado aos módulos do sistema (ver [`../arquitetura/architecture.md`](../arquitetura/architecture.md)):

| Prefixo de escopo | Módulo |
|---|---|
| `feature/<modulo1>-*` | Módulo 1 (RF01) |
| `feature/<modulo2>-*` | Módulo 2 (RF02) |
| `feature/auth-*` / `feature/iam-*` | Autenticação e perfis de acesso |

- Abre PR → `develop`
- CI executa build/lint/test
- Revisão obrigatória antes do merge
- Merge dispara deploy automático em staging

Feature branches nunca são direcionadas diretamente para `main`.

---

## Hotfix Branches

**Criada a partir de:** `main`
**PR direcionado para:** `main`

Usada apenas quando a produção está quebrada e não pode esperar o ciclo normal `develop → main`.

```bash
git checkout main
git pull origin main
git checkout -b hotfix/<escopo>-<descricao-curta>
# fix, commit
git push origin hotfix/<escopo>-<descricao-curta>
```

**Convenção de nomenclatura:** `hotfix/<escopo>-<descricao-curta>`, mesmos escopos da tabela de feature branches acima.

- Abre PR → `main`
- CI roda, revisão, merge → dispara deploy de produção
- Marcar a correção com tag, se usando tags de versão
- **Follow-up obrigatório:** sincronizar a correção de volta para `develop` (ver abaixo), ou o bug pode reaparecer no próximo release

---

## Mantendo `develop` Sincronizada Após um Hotfix

Dois métodos aceitos:

**Opção A — Merge de `main` em `develop` (escolha padrão)**
```bash
git checkout develop
git pull origin develop
git merge main
git push origin develop
```

**Opção B — Cherry-pick apenas da correção**
```bash
git checkout develop
git pull origin develop
git cherry-pick <hotfix-commit-sha>
git push origin develop
```
Use quando `develop` já divergiu significativamente e um merge completo traria mudanças não relacionadas.

> **Automação:** o workflow `sync-main-to-develop.yml` abre automaticamente um PR `main → develop` sempre que algo é mesclado em `main`. Isso evita que as duas branches se desalinhem silenciosamente.

---

## Release: `develop` → `main`

Esta é uma **decisão manual e deliberada** — nunca automática a cada merge em `develop`.

### Quando disparar
- Cadência de release programada (ex.: quinzenal, ou conforme cronograma acordado com a equipe)
- Marco de funcionalidade completa atingido e validado em staging
- Sign-off manual de QA/responsável no ambiente de staging

**Decisão: o sistema não usa branches `release/*`.** Simplicidade: releases vão diretamente de `develop` para `main`.

```
develop → (quando pronto) → PR para main → deploy de produção
```

```bash
git checkout main
git pull origin main
# abrir PR: develop → main
```
- Revisão, CI passa, merge → deploy de produção
- Marcar o release com tag: `git tag v1.4.0` (opcional, mas recomendado para rastreabilidade)
- Não é necessário sync-back aqui, já que nada foi ramificado — `develop` e `main` já ficam alinhadas após o merge

---

## Branch Protection Rules (`main` + `develop`, ambos os repositórios)

- Exigir PR + no mínimo 1 revisão
- Exigir que os checks de CI passem (build, lint, test)
- Sem push direto
- Sem force-push
- Exigir que a branch esteja atualizada antes do merge

---

## Branch Cleanup (Após o Merge)

Branches **não são excluídas automaticamente** por padrão — isso deve ser habilitado por plataforma:

- **GitHub:** repo Settings → General → habilitar "Automatically delete head branches"

### O que excluir vs manter

| Branch | Excluir após merge? |
|---|---|
| `feature/*` | **Sim** — sempre, após o merge em `develop` |
| `hotfix/*` | **Sim** — após o merge em `main` **e** sincronizado de volta para `develop` |
| `main` | **Nunca** — permanente |
| `develop` | **Nunca** — permanente |

> **Regra geral:** não exclua uma branch `hotfix/*` até que o sync-back para `develop` esteja confirmado. Excluir uma branch só remove o ponteiro — os commits permanecem no histórico via o merge commit — mas mantê-la por um tempo facilita o cherry-pick caso o sync dê errado.

---

## Auto-Sync Workflow (`main` → `develop`)

**Plataforma:** GitHub Actions
**Arquivo:** [`workflows/CICD_sync-main-to-develop.yml`](./workflows/CICD_sync-main-to-develop.yml) (colocar em `.github/workflows/sync-main-to-develop.yml` em ambos os repositórios: `xxxx-frontend` e `xxxx-backend`)

**O que faz:**
1. Dispara a cada push em `main` (cobre merges de hotfix e de release)
2. Verifica se `develop` está atrasada em relação a `main`
3. Se sim, cria uma branch temporária `sync/main-to-develop-*`, mescla `main` nela
4. Se o merge for limpo → faz push e abre um PR para `develop`
5. Se houver conflito → aborta e falha o job, para um humano resolver o sync manualmente
6. Se `develop` já estiver atualizada → não faz nada

**Por que um PR em vez de push direto em `develop`:** mantém a regra de proteção de branch ("exigir PR + revisão") consistente, e dá a um humano a chance de revisar antes de mesclar.

**Setup necessário:**
- Adicionar o arquivo de workflow nos dois repositórios
- Nenhum secret extra necessário — usa o `GITHUB_TOKEN` embutido
- Garantir que a configuração do repositório **"Allow GitHub Actions to create and approve pull requests"** esteja habilitada (Settings → Actions → General), caso contrário `gh pr create` falhará com erro de permissão

---

## CI/CD Deploy Workflows (Docker + VPS via SSH)

**Arquivos:**
- [`workflows/CICD_backend.yml`](./workflows/CICD_backend.yml) → `.github/workflows/ci-cd.yml` em `xxxx-backend`
- [`workflows/CICD_frontend.yml`](./workflows/CICD_frontend.yml) → `.github/workflows/ci-cd.yml` em `xxxx-frontend`

**Fluxo (ambos os repositórios):**
1. Todo push/PR → job `build-and-test` (lint, test, build) — sem deploy
2. Push em `develop` → job `deploy-staging` → constrói imagem Docker, publica no GHCR, conecta via SSH na VPS, sobe o container com a config do GitHub Environment **staging**
3. Push em `main` → job `deploy-production` → mesmo fluxo, com a config do GitHub Environment **production**

**Stack de build por repositório:**
- **`xxxx-backend`** (Spring Boot / Java 17 / Maven): `mvn -B verify` para lint/test/build ([`workflows/CICD_backend.yml`](./workflows/CICD_backend.yml)), imagem Docker via `Dockerfile` multi-stage (build Maven → runtime JRE).
- **`xxxx-frontend`** (React / TypeScript / Vite): `npm ci`, `npm run lint`, `npm test`, `npm run build` ([`workflows/CICD_frontend.yml`](./workflows/CICD_frontend.yml)).

**Distinção importante — variáveis de ambiente frontend vs backend:**
- **Backend:** env vars (`DATABASE_URL`, `JWT_SECRET`, etc. — usadas também pelo módulo IAM, ver [`../arquitetura/IAM.md`](../arquitetura/IAM.md)) são passadas em **runtime** do container (`docker run -e ...`) — a mesma imagem pode rodar em qualquer ambiente.
- **Frontend:** `VITE_API_URL` é embutida no bundle JS estático em **build time**, então deve ser passada como **build-arg** do Docker, não como variável de runtime. Isso significa que o frontend precisa de uma build de imagem separada por ambiente (staging vs prod), não uma imagem compartilhada.

### Setup necessário — GitHub Environments

Em **cada repositório** (Settings → Environments), criar dois ambientes: `staging` e `production`. Popular cada um com:

| Nome | Tipo | Usado por | Exemplo (staging) | Exemplo (produção) |
|---|---|---|---|---|
| `API_URL` | Variable | build-arg do frontend | `https://api-staging.xxxx.exemplo.org` | `https://api.xxxx.exemplo.org` |
| `FRONTEND_URL` | Variable | backend (config de CORS) | `https://staging.xxxx.exemplo.org` | `https://xxxx.exemplo.org` |
| `DEPLOY_HOST` | Variable | ambos, destino SSH | IP/hostname da VPS de staging | IP/hostname da VPS de produção |
| `DEPLOY_USER` | Variable | ambos, usuário SSH | ex.: `deploy` | ex.: `deploy` |
| `DEPLOY_SSH_KEY` | **Secret** | ambos, chave SSH privada | chave de staging | chave de produção |
| `DATABASE_URL` | **Secret** | backend apenas | string de conexão do banco de staging | string de conexão do banco de produção |
| `JWT_SECRET`/segredos do IAM | **Secret** | backend apenas | secret de staging | secret de produção |

Regra geral: qualquer coisa sensível (chaves, tokens, credenciais de banco) → **Secret**. Qualquer coisa apenas informativa (uma URL, um hostname) → **Variable**. Ambos são escopados por ambiente, então `staging` e `production` nunca veem os valores um do outro.

> Os domínios `xxxx.exemplo.org` acima são placeholder — substitua pelo domínio real do projeto.

**Registro de containers:** estes workflows publicam imagens no GitHub Container Registry (`ghcr.io`) usando o `GITHUB_TOKEN` embutido — nenhuma credencial extra de registro necessária.

**Portas usadas nos workflows de exemplo** (ajustar para a configuração real da VPS):
- Backend: `8080` (prod), `8081` (staging)
- Frontend: `80` (prod), `8082` (staging)

---

## CI/CD Trigger Summary

| Branch | Trigger | Action |
|---|---|---|
| `main` | on merge | build → test → deploy para **produção** |
| `develop` | on merge | build → test → deploy para **staging** |
| `feature/*` | on push | build → test apenas |
| `hotfix/*` | on push | build → test apenas, PR fast-track para `main` |

---

## Cross-Repo Coordination (frontend ⇄ backend)

Como os dois repositórios fazem deploy independentemente mas dependem um do outro:

- **Versionamento de API:** marcar releases do backend com tag (ex.: `v1.2.0`) para que o frontend possa fixar uma versão conhecida como compatível.
- **URLs de ambiente:** a build de staging do frontend aponta para o deploy de staging do backend; a build de produção do frontend aponta para o deploy de produção do backend. Gerenciado via env vars/secrets por pipeline (ver tabela de GitHub Environments acima).
- **Ordem de merge:** quando uma funcionalidade abrange os dois repositórios (ex.: novo endpoint + tela correspondente), mesclar o backend primeiro, depois o frontend, para evitar quebrar o contrato durante o deploy.

---

## Quick Reference

```
feature/*  ← branched from develop  → PR into develop
hotfix/*   ← branched from main     → PR into main → sync back into develop
develop    → PR into main directly (no release branches)
```

## Checklist de setup do projeto
- [ ] Decidir GitHub Actions vs GitLab CI para os dois repositórios
- [ ] Adaptar [`workflows/CICD_sync-main-to-develop.yml`](./workflows/CICD_sync-main-to-develop.yml), adicionando em `.github/workflows/` nos dois repositórios
- [ ] Decidir se branches de release (`release/*`) serão usadas ou não
- [ ] Definir gerenciamento de env vars/secrets para as URLs de API entre repositórios via GitHub Environments (`staging`/`production`)
- [ ] Adaptar [`workflows/CICD_backend.yml`](./workflows/CICD_backend.yml) e [`workflows/CICD_frontend.yml`](./workflows/CICD_frontend.yml) ao stack real do projeto, substituindo os placeholders `xxxx-backend`/`xxxx-frontend`
- [ ] Definir domínio real de staging/produção (placeholder usado: `xxxx.exemplo.org`)
- [ ] Criar `Dockerfile` multi-stage do backend (build Maven → runtime JRE)
- [ ] Criar `Dockerfile` multi-stage do frontend (build Vite → Nginx)
- [ ] Submodular este repositório de docs como `docs/` nos repositórios de backend e frontend, e adicionar o job de checagem de ponteiro defasado no `build-and-test` dos dois (ver seção "Context" acima)
