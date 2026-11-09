# Plano de Implementação (IMPLEMENTATION-PLAN)

## 1. Objetivo

Consolidar as especificações técnicas aprovadas em um plano de implementação claro, organizado e executável.

Este documento define o que será implementado, quais componentes serão afetados, a ordem das atividades e os critérios para validar a conclusão.

Nenhuma tarefa poderá ser iniciada sem aprovação humana do plano.

## 2. Informações Gerais

- **Projeto:** [Nome do projeto]
- **Funcionalidade ou mudança:** [Nome]
- **Identificador Changelog:** [CHG-XXX]
- **Especificação EGM:** [Caminho do documento]
- **Relatório de revisão:** [Caminho do documento]
- **Status geral:** RASCUNHO
- **Data de criação:** AAAA-MM-DD
- **Última atualização:** AAAA-MM-DD

## 3. Objetivo da Implementação

[Descrever o resultado esperado e o problema que será resolvido.]

## 4. Escopo Aprovado

### Incluído

- [Funcionalidade ou alteração 1]
- [Funcionalidade ou alteração 2]

### Não incluído

- [Funcionalidade fora do escopo 1]
- [Melhoria que será tratada futuramente]

## 5. Requisitos Relacionados

| ID | Requisito | Documento de referência |
|---|---|---|
| REQ-001 | [Descrição] | [Caminho] |

## 6. Referências Arquiteturais

| Documento | Decisão ou orientação |
|---|---|
| `docs/arquitetura/architecture.md` | [Orientação — backend, frontend e banco/migrações estão todos neste único arquivo] |
| `docs/arquitetura/IAM.md` | [Orientação — autenticação, RBAC e auditoria] |

Adicionar ou remover referências conforme os componentes envolvidos.

## 7. Especificação Técnica

### 7.1. Backend

- [Serviços e módulos envolvidos]
- [Regras de negócio]
- [Endpoints]
- [Validações]
- [Tratamento de erros]

### 7.2. Frontend

- [Telas e componentes]
- [Fluxos de navegação]
- [Gerenciamento de estado]
- [Integrações com APIs]
- [Validações de interface]

### 7.3. Banco de Dados

- [Entidades e tabelas]
- [Novos campos]
- [Relacionamentos]
- [Índices]
- [Migrações necessárias]

### 7.4. Integrações

- [Serviços externos]
- [Contratos]
- [Autenticação]
- [Tratamento de falhas]

### 7.5. Segurança

- [Autorização e autenticação]
- [Validação de dados]
- [Proteção de informações]
- [Riscos identificados]

## 8. Plano de Tarefas

As tarefas devem ser pequenas, objetivas e executáveis.

| ID | Tarefa | Componente | Dependência | Status |
|---|---|---|---|---|
| TASK-001 | [Descrição] | Backend | Nenhuma | PENDENTE |
| TASK-002 | [Descrição] | Database | TASK-001 | PENDENTE |
| TASK-003 | [Descrição] | Frontend | TASK-002 | PENDENTE |
| TASK-004 | [Descrição] | Testes | TASK-003 | PENDENTE |

### Status das tarefas

- PENDENTE
- EM ANDAMENTO
- BLOQUEADA
- CONCLUÍDA
- CANCELADA

## 9. Ordem de Execução

Descrever a sequência recomendada de implementação, considerando dependências técnicas.

1. [Primeira atividade]
2. [Segunda atividade]
3. [Terceira atividade]
4. [Validação final]

## 10. Testes e Critérios de Aceitação

IDs de teste seguem a convenção `TC-<UC>.<N>` definida em [`../plano-de-testes/README.md`](../plano-de-testes/README.md) — nunca um ID solto; cada linha referencia o UC/FA/RN de origem.

| ID | UC/Origem | Cenário | Resultado esperado | Status |
|---|---|---|---|---|
| TC-XX.1 | [UCxx, Fluxo Principal] | [Cenário] | [Resultado] | PENDENTE |

### Critérios de aceitação

- [ ] Requisitos atendidos.
- [ ] Regras de negócio implementadas.
- [ ] APIs funcionando conforme os contratos.
- [ ] Interface funcionando conforme especificado.
- [ ] Banco de dados atualizado corretamente.
- [ ] Testes automatizados executados.
- [ ] Tratamento de erros validado.
- [ ] Segurança validada.

## 11. Riscos e Dependências

### Riscos

- [Risco identificado]
- [Impacto potencial]
- [Estratégia de mitigação]

### Dependências

- [Serviços]
- [Bibliotecas]
- [Infraestrutura]
- [Decisões pendentes]

## 12. Aprovação do Plano

- **Status:** PENDENTE
- **Data da aprovação:**
- **Responsável:**
- **Observações:**

A implementação somente poderá começar após aprovação explícita.

## 13. Histórico de Execução

| Data | Tarefa | Responsável | Resultado |
|---|---|---|---|
| AAAA-MM-DD | TASK-001 | [Agente] | [Resultado] |

## 14. Conclusão

- **Status geral:** NÃO INICIADO
- **Tarefas concluídas:** 0
- **Tarefas pendentes:** 0
- **Testes aprovados:** 0
- **Data de conclusão:**

### Validação final

- [ ] Todas as tarefas foram concluídas.
- [ ] Critérios de aceitação atendidos.
- [ ] Testes aprovados.
- [ ] Documentação atualizada.
- [ ] Alterações registradas no Changelog.
- [ ] Revisão final realizada.