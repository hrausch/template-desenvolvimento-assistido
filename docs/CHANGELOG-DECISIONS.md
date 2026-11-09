# Registro de Mudanças e Decisões (CHANGELOG)

## 1. Objetivo

Registrar todas as mudanças propostas, analisadas, aprovadas, rejeitadas e implementadas no projeto.

Este documento garante rastreabilidade das alterações de requisitos, arquitetura, regras de negócio e decisões técnicas.

## 2. Regras

- Toda mudança relevante deve possuir um identificador único.
- Nenhuma mudança pode ser considerada aprovada sem decisão humana explícita.
- A aprovação deve ser registrada neste documento.
- Mudanças aprovadas devem seguir o fluxo definido em `docs/WORKFLOW.md`.
- Mudanças rejeitadas não devem ser implementadas.
- Uma mudança somente pode ser marcada como IMPLEMENTADA após sua validação.
- Não apagar registros antigos. Atualizar seu histórico e status.

## 3. Status Possíveis

| Status | Significado |
|---|---|
| PENDENTE | Aguardando aprovação humana |
| APROVADA | Autorizada para evolução técnica e documental |
| EM ANÁLISE | Em elaboração ou revisão técnica |
| AGUARDANDO APROVAÇÃO FINAL | Especificação e plano aguardam aprovação |
| PRONTA PARA IMPLEMENTAÇÃO | Plano aprovado |
| EM IMPLEMENTAÇÃO | Desenvolvimento em andamento |
| IMPLEMENTADA | Implementação concluída e validada |
| REJEITADA | Mudança não autorizada |
| CANCELADA | Mudança deixou de ser necessária |

## 4. Tipos de Mudança

- CORREÇÃO DE REQUISITO
- NOVO REQUISITO
- ALTERAÇÃO DE REQUISITO
- NOVA DECISÃO ARQUITETURAL
- ALTERAÇÃO ARQUITETURAL
- ALTERAÇÃO DE BANCO DE DADOS
- ALTERAÇÃO DE CONTRATO DE API
- MELHORIA TÉCNICA

## 5. Registro de Mudanças

Novas propostas devem ser adicionadas abaixo, mantendo os registros anteriores.

---

## CHG-001 — [Título da mudança]

### Identificação

- **Data de criação:** AAAA-MM-DD
- **Origem:** [Agente, funcionalidade ou revisão]
- **Tipo:** [Tipo da mudança]
- **Complexidade:** [Pequena / Média / Grande]
- **Status:** PENDENTE

### 1. Contexto e justificativa

[Descrever o problema ou a necessidade identificada.]

### 2. Situação atual

[Descrever como o sistema funciona atualmente e quais documentos definem esse comportamento.]

### 3. Alteração proposta

[Descrever detalhadamente o que precisa ser alterado.]

### 4. Motivo da alteração

[Explicar por que a mudança é necessária.]

### 5. Impactos identificados

- **Requisitos:** [Documentos e casos de uso afetados]
- **Backend:** [Serviços, módulos e regras afetadas]
- **Frontend:** [Telas e componentes afetados]
- **Database:** [Entidades, tabelas e relacionamentos afetados]
- **APIs:** [Endpoints e contratos afetados]
- **Integrações:** [Serviços externos afetados]
- **Testes:** [Cenários que precisam ser criados ou modificados]

### 6. Avaliação técnica

[Recomendação inicial do agente e alternativas consideradas.]

### 7. Decisão humana

- **Status:** PENDENTE
- **Data da decisão:** 
- **Responsável pela decisão:** 
- **Observações:** 

### 8. Especificação EGM

- **Draft:** [Referência do documento]
- **Review:** [Referência do relatório]
- **Resolve:** [Referência da especificação revisada]
- **Resultado:** [Resumo das decisões técnicas]

### 9. Documentação atualizada

- [ ] Requisitos atualizados.
- [ ] Casos de uso atualizados.
- [ ] Arquitetura atualizada.
- [ ] Banco de dados atualizado.
- [ ] Contratos de API atualizados.
- [ ] Outras documentações atualizadas.

### 10. Plano de implementação

- **Referência:** `docs/planos-desenvolvimento/CHG-001.md` (criado a partir do template `docs/templates/IMPLEMENTATION-PLAN.md`)
- **Aprovação do plano:** PENDENTE

### 11. Implementação

- **Status:** NÃO INICIADA
- **Responsável:** 
- **Data de início:** 
- **Data de conclusão:** 

### 12. Validação

- [ ] Critérios de aceitação atendidos.
- [ ] Testes executados.
- [ ] Documentação validada.
- [ ] Implementação revisada.

### 13. Resultado final

[Descrever o resultado e a decisão final.]

---

## Histórico de mudanças

| ID | Título | Tipo | Complexidade | Status |
|---|---|---|---|---|
| CHG-001 | [Título] | [Tipo] | [Nível] | PENDENTE |
