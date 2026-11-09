# Fluxo de Desenvolvimento (WORKFLOW)

## 1. Objetivo

Definir o processo obrigatório para planejamento, especificação, revisão, aprovação, implementação e manutenção do projeto.

O objetivo é garantir que todos os agentes trabalhem com base em requisitos documentados, decisões arquiteturais aprovadas e especificações técnicas validadas.

Este processo também estabelece como tratar mudanças identificadas durante o desenvolvimento.

## 2. Estrutura e Fontes Oficiais

| Documento | Responsabilidade |
|---|---|
| `CLAUDE.md` | Instruções gerais para os agentes |
| `docs/WORKFLOW.md` | Processo de desenvolvimento |
| `docs/requisitos/` | Requisitos funcionais e não funcionais |
| `docs/casos-de-uso/` | Casos de uso derivados dos requisitos |
| `docs/arquitetura/` | Arquitetura aprovada (`architecture.md`, `IAM.md`) |
| `docs/CHANGELOG-DECISIONS.md` | Registro de mudanças e aprovações |
| `docs/templates/IMPLEMENTATION-PLAN.md` | Template para criação do plano de cada mudança |
| `docs/planos-desenvolvimento/CHG-XXX.md` | Plano de implementação de cada mudança (um arquivo por CHG) |
| `docs/egm/` | Artefatos produzidos pelo EGM |

### Hierarquia das informações

- Os requisitos aprovados definem o que o sistema deve fazer.
- A arquitetura aprovada define como o sistema deve ser estruturado.
- A especificação técnica detalha como uma funcionalidade será implementada.
- O plano de implementação transforma a especificação em tarefas executáveis.
- O Changelog registra as mudanças e suas aprovações.

Os documentos oficiais devem ser mantidos consistentes entre si.

## 3. Fluxo Inicial do Projeto

Este fluxo deve ser utilizado quando o projeto possui escopo definido, mas ainda não possui implementação.

### Fase 1 — Análise do Projeto

O agente deve:

- Analisar os requisitos existentes.
- Analisar os casos de uso.
- Consultar os documentos arquiteturais.
- Identificar tecnologias, restrições e integrações.
- Identificar ambiguidades e informações ausentes.

Nenhum requisito ou decisão existente deve ser alterado nesta fase.

### Fase 2 — Elaboração da Especificação (EGM Draft)

Executar `/egm-draft`.

Objetivos:

- Transformar os requisitos em uma especificação técnica.
- Considerar a arquitetura existente.
- Identificar componentes envolvidos.
- Definir responsabilidades e dependências.
- Identificar decisões técnicas pendentes.

Resultado esperado:

- Especificação técnica de implementação.
- Premissas e decisões necessárias.
- Dependências.
- Riscos e questões em aberto.

Armazenar os artefatos em `docs/egm/drafts/`.

### Fase 3 — Revisão Independente (EGM Review)

Executar `/egm-review`.

Objetivos:

- Avaliar a especificação de forma independente.
- Identificar requisitos ausentes.
- Detectar inconsistências arquiteturais.
- Avaliar riscos de segurança e manutenção.
- Identificar problemas técnicos e funcionais.

Resultado esperado:

- Relatório de revisão.
- Problemas identificados.
- Classificação de severidade.
- Recomendações.

Armazenar os relatórios em `docs/egm/reviews/`.

### Fase 4 — Resolução (EGM Resolve)

Executar `/egm-resolve`, quando houver problemas que precisem de resolução.

Objetivos:

- Analisar os apontamentos.
- Resolver ambiguidades.
- Avaliar recomendações conflitantes.
- Atualizar a especificação técnica.

Resultado esperado:

- Especificação revisada.
- Problemas resolvidos.
- Questões pendentes.
- Decisões que precisam de aprovação humana.

### Fase 5 — Identificação de Mudanças

Comparar a especificação produzida pelo EGM com os requisitos e a arquitetura existentes.

Identificar mudanças necessárias e classificá-las como:

1. Correção de requisito.
2. Novo requisito.
3. Nova decisão arquitetural.
4. Alteração de requisito existente.
5. Alteração de decisão arquitetural existente.

Registrar todas as propostas em `docs/CHANGELOG-DECISIONS.md`.

Não modificar os documentos oficiais nesta fase.

### Fase 6 — Aprovação das Mudanças

O responsável humano deve analisar cada proposta.

Cada mudança deve receber uma das seguintes decisões:

- PENDENTE: aguardando avaliação.
- APROVADA: autorizada para evolução técnica e documental.
- REJEITADA: não deve ser incorporada.
- CANCELADA: não será mais considerada.

A aprovação de uma mudança não representa automaticamente a aprovação da implementação.

### Fase 7 — Consolidação Documental

Após a aprovação das mudanças:

- Atualizar os requisitos afetados.
- Atualizar os casos de uso afetados.
- Atualizar os documentos de arquitetura.
- Atualizar banco de dados e integrações, quando necessário.
- Preservar os requisitos não afetados.
- Atualizar o Changelog.

A documentação atualizada passa a representar a nova referência oficial do projeto.

### Fase 8 — Sincronização e Revisão Final

Atualizar a especificação técnica para refletir os documentos aprovados.

Executar nova revisão independente quando houver alterações relevantes.

Verificar:

- Consistência entre requisitos e arquitetura.
- Consistência entre documentação e especificação.
- Ausência de problemas críticos.
- Clareza das decisões técnicas.
- Dependências necessárias à implementação.

### Fase 9 — Plano de Implementação

Criar um plano individual para a mudança, usando `docs/templates/IMPLEMENTATION-PLAN.md` como template.

O plano deve ser armazenado em `docs/planos-desenvolvimento/CHG-XXX.md`, associado ao identificador da mudança no Changelog (ex.: `docs/planos-desenvolvimento/CHG-015.md`).

O plano deve conter:

- Identificação da mudança ou funcionalidade.
- Escopo aprovado.
- Requisitos relacionados.
- Documentos arquiteturais relacionados.
- Componentes afetados.
- Alterações de backend.
- Alterações de frontend.
- Alterações de banco de dados.
- Integrações necessárias.
- Testes e critérios de aceitação.
- Tarefas ordenadas por dependência.

Regras:

- Cada CHG deve possuir seu próprio plano.
- Nunca sobrescrever planos de outras mudanças nem excluir planos anteriores.
- Atualizar o plano correspondente durante a implementação.

O plano deve ser apresentado para aprovação humana.

Não iniciar a implementação antes dessa aprovação.

### Fase 10 — Implementação

Após a aprovação do plano:

- Executar as tarefas na ordem definida.
- Seguir a especificação técnica.
- Respeitar a arquitetura aprovada.
- Atender aos critérios de aceitação.
- Executar os testes previstos.
- Registrar o andamento das tarefas.

Qualquer necessidade de mudança deve seguir a seção 4 deste documento.

### Fase 11 — Validação e Conclusão

Após a implementação:

- Executar os testes.
- Validar os critérios de aceitação.
- Verificar a documentação.
- Registrar desvios e decisões.
- Atualizar o status das mudanças no Changelog.
- Marcar as tarefas concluídas no plano.

Uma mudança só pode ser marcada como IMPLEMENTADA após a validação de sua conclusão.

---

## 4. Gestão de Mudanças Durante a Implementação

Esta seção é obrigatória para todos os agentes, inclusive aqueles responsáveis pela implementação.

### 4.1. Identificação da necessidade

Durante o desenvolvimento, um agente pode identificar:

- Um requisito ausente.
- Uma regra de negócio não documentada.
- Uma inconsistência entre requisitos.
- Uma limitação da arquitetura.
- Uma necessidade de alteração no banco de dados.
- Uma mudança necessária em contratos de API.
- Uma oportunidade de melhoria técnica.

O agente deve identificar a necessidade e registrar a proposta.

### 4.2. Registro no Changelog

Toda alteração que modifique requisitos, arquitetura ou contratos aprovados deve ser registrada em:

`docs/CHANGELOG-DECISIONS.md`

O registro deve conter:

- Identificador único.
- Origem da necessidade.
- Justificativa.
- Descrição da alteração.
- Situação atual.
- Alteração proposta.
- Impactos técnicos e funcionais.
- Complexidade estimada.
- Recomendação do agente.

O agente deve interromper somente a parte do trabalho afetada pela mudança. Outras tarefas independentes podem continuar.

### 4.3. Classificação da complexidade

As mudanças devem ser classificadas em três níveis.

#### Nível 1 — Pequena

Exemplos:

- Correção de texto ou nomenclatura.
- Ajuste de validação já previsto.
- Correção de documentação sem alteração de comportamento.
- Ajuste técnico localizado que não altera contratos ou arquitetura.

Fluxo:

1. Registrar a mudança.
2. Obter aprovação, quando alterar comportamento ou documentação aprovada.
3. Atualizar a documentação aplicável.
4. Implementar.
5. Executar testes.

Não exige necessariamente o fluxo completo do EGM.

#### Nível 2 — Média

Exemplos:

- Inclusão de campo em uma entidade.
- Alteração de regra de negócio.
- Criação ou alteração de endpoint.
- Inclusão de funcionalidade em módulo existente.
- Alteração de estrutura de dados sem mudança arquitetural ampla.

Fluxo:

1. Registrar no Changelog.
2. Obter aprovação da mudança de escopo.
3. Executar EGM Draft.
4. Executar EGM Review.
5. Executar EGM Resolve, se necessário.
6. Atualizar requisitos e arquitetura.
7. Sincronizar a especificação.
8. Criar o plano de implementação.
9. Obter aprovação final.
10. Implementar.

#### Nível 3 — Grande

Exemplos:

- Alteração da estratégia de autenticação.
- Mudança de banco de dados.
- Introdução de nova arquitetura.
- Substituição de tecnologias estruturais.
- Mudança significativa no modelo de domínio.
- Alterações que impactem diversos módulos.

Fluxo:

1. Registrar no Changelog.
2. Obter aprovação para explorar a mudança.
3. Executar EGM Brainstorm.
4. Executar EGM Draft.
5. Executar EGM Review.
6. Executar EGM Resolve, se necessário.
7. Apresentar as decisões arquiteturais propostas.
8. Obter aprovação humana.
9. Atualizar requisitos e arquitetura.
10. Sincronizar a especificação.
11. Criar o plano de implementação.
12. Obter aprovação final.
13. Implementar.

A classificação deve considerar o impacto e o risco, não apenas a quantidade de código envolvida.

### 4.4. Aprovação da mudança versus aprovação da implementação

Existem duas aprovações distintas:

**Aprovação da mudança:**

Autoriza a evolução do escopo e a análise da solução técnica.

**Aprovação do plano de implementação:**

Autoriza a execução da solução técnica detalhada e especificada.

Uma aprovação não substitui a outra.

### 4.5. Atualização da documentação

Somente após a aprovação das decisões técnicas:

- Atualizar requisitos.
- Atualizar casos de uso.
- Atualizar arquitetura.
- Atualizar contratos de API.
- Atualizar documentação de banco de dados.
- Atualizar demais documentos afetados.

O agente deve informar quais documentos foram alterados.

### 4.6. Retomada da implementação

Após a aprovação do plano:

- Atualizar o status da mudança.
- Executar as tarefas aprovadas.
- Não ampliar o escopo por iniciativa própria.
- Registrar novas necessidades no Changelog.
- Atualizar o plano conforme as tarefas forem concluídas.

---

## 5. Novas Funcionalidades e Ideias

Quando surgir uma nova funcionalidade:

1. Registrar a ideia ou necessidade.
2. Utilizar EGM Brainstorm quando houver dúvidas sobre o problema ou possíveis soluções.
3. Definir comportamento esperado.
4. Definir critérios de aceitação.
5. Registrar as mudanças no Changelog.
6. Obter aprovação humana.
7. Seguir o fluxo correspondente à complexidade.
8. Atualizar a documentação.
9. Criar e aprovar o plano de implementação.
10. Implementar.

O Brainstorm é opcional quando os requisitos já estão claros.

## 6. Regras Obrigatórias de Governança

1. Nenhum agente pode modificar requisitos aprovados silenciosamente.
2. Nenhum agente pode alterar decisões arquiteturais aprovadas sem autorização.
3. Toda mudança relevante deve possuir registro no Changelog.
4. Toda aprovação deve ser explícita.
5. Toda implementação deve possuir especificação e escopo aprovados.
6. O EGM deve revisar as especificações conforme a complexidade da mudança.
7. Artefatos do EGM não substituem a documentação oficial.
8. O plano de implementação deve ser aprovado antes da execução.
9. Mudanças descobertas durante a implementação devem retornar ao processo de gestão de mudanças.
10. Toda implementação deve ser validada antes de ser considerada concluída.

## 7. Critérios para Iniciar a Implementação (Definition of Ready)

Uma funcionalidade estará pronta para implementação quando:

- [ ] Requisitos documentados e aprovados.
- [ ] Arquitetura documentada e aprovada.
- [ ] Especificação técnica concluída.
- [ ] Revisão técnica realizada conforme a complexidade.
- [ ] Problemas críticos resolvidos.
- [ ] Mudanças necessárias aprovadas e incorporadas.
- [ ] Plano de implementação criado.
- [ ] Plano de implementação aprovado.
- [ ] Critérios de aceitação definidos.

## 8. Critérios de Conclusão (Definition of Done)

Uma funcionalidade estará concluída quando:

- [ ] Implementação seguir a especificação aprovada.
- [ ] Critérios de aceitação atendidos.
- [ ] Testes executados com sucesso.
- [ ] Não existirem desvios arquiteturais não documentados.
- [ ] Documentação técnica atualizada.
- [ ] Alterações registradas no Changelog.
- [ ] Tarefas do plano atualizadas.
- [ ] Status das mudanças atualizado para IMPLEMENTADO, quando aplicável.