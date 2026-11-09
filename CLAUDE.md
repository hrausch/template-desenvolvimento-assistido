# Instruções do Projeto

## 1. Objetivo

Este documento define as instruções gerais que todos os agentes de IA devem seguir durante o desenvolvimento e a manutenção deste projeto.

Todos os agentes devem conhecer e respeitar os requisitos, a arquitetura e os processos definidos na documentação oficial.

## 2. Documentação Obrigatória

Antes de executar qualquer atividade relevante, consulte os documentos aplicáveis:

- `docs/WORKFLOW.md`: processo de desenvolvimento e gestão de mudanças.
- `docs/requisitos/`: requisitos funcionais e não funcionais.
- `docs/casos-de-uso/`: casos de uso derivados dos requisitos.
- `docs/arquitetura/`: arquitetura e decisões técnicas aprovadas (`architecture.md`, `IAM.md`).
- `docs/CHANGELOG-DECISIONS.md`: registro de mudanças e aprovações.
- `docs/templates/IMPLEMENTATION-PLAN.md`: **template** usado para criar o plano de cada mudança (não é um plano em si).
- `docs/planos-desenvolvimento/CHG-XXX.md`: plano de implementação aprovado de cada mudança (um arquivo por CHG).
- `docs/egm/`: especificações e relatórios produzidos pelo EGM.

Não presuma que o histórico de conversas representa a documentação oficial do projeto.

## 3. Regras Obrigatórias

### 3.1. Respeito à documentação

- Os requisitos aprovados são a referência funcional do sistema.
- A arquitetura aprovada é a referência técnica do sistema.
- Não introduza funcionalidades não documentadas sem aprovação.
- Não altere decisões arquiteturais silenciosamente.
- Não contradiga requisitos existentes sem registrar a necessidade de mudança.

### 3.2. Gestão de mudanças

Sempre que identificar a necessidade de alterar requisitos, arquitetura, banco de dados, contratos de API ou escopo:

1. Registre a proposta em `docs/CHANGELOG-DECISIONS.md`.
2. Informe a justificativa e os impactos.
3. Classifique a complexidade da mudança.
4. Aguarde aprovação humana.
5. Siga o processo definido em `docs/WORKFLOW.md`.

Nunca considere uma sugestão da IA como uma aprovação humana.

### 3.3. Implementação

- Consulte sempre o plano da mudança em `docs/planos-desenvolvimento/CHG-XXX.md` antes de implementar funcionalidades previstas nele.
- Implemente somente tarefas aprovadas.
- Respeite a ordem e as dependências das tarefas.
- Não realize alterações fora do escopo aprovado.
- Informe imediatamente qualquer impedimento ou necessidade de mudança.
- Atualize os registros de execução após concluir cada tarefa.

### 3.4. Documentação

- Mantenha os documentos atualizados.
- Registre alterações aprovadas.
- Não substitua documentos oficiais por relatórios gerados pelo EGM.
- Preserve o histórico das decisões.
- Não marque alterações como implementadas antes de validar sua conclusão.

## 4. Uso do EGM

O EGM deve ser utilizado conforme a complexidade e o impacto das mudanças.

- Draft: elaboração da especificação técnica.
- Review: revisão independente da especificação.
- Resolve: análise e resolução dos problemas identificados.
- Brainstorm: exploração de novas ideias e decisões ainda não definidas.

Todos os artefatos do EGM devem ser armazenados em `docs/egm/`.

EGM artifact directory: docs/egm/

Os agentes devem consultar `docs/WORKFLOW.md` para identificar quais etapas são obrigatórias em cada situação.

## 5. Aprovação Humana

As seguintes decisões exigem aprovação humana explícita:

- Alteração de requisitos aprovados.
- Inclusão de novos requisitos.
- Alteração de arquitetura.
- Mudanças em contratos de API.
- Alterações estruturais no banco de dados.
- Aprovação da especificação técnica quando exigida pelo workflow.
- Aprovação do plano de implementação.

A ausência de resposta não significa aprovação.

## 6. Regra Principal

**Nenhum agente pode alterar o escopo aprovado ou tomar decisões arquiteturais relevantes sem registrar a mudança, obter aprovação humana e seguir o processo definido em `docs/WORKFLOW.md`.**

## 7. Planos de Implementação

- Cada mudança ou funcionalidade deve possuir seu próprio plano.
- Nunca sobrescreva planos de implementação anteriores.
- Os planos devem ser armazenados em `docs/planos-desenvolvimento/`.
- Cada plano deve estar associado a um identificador do Changelog (CHG).
- Consulte o plano correspondente antes de implementar.