# Design System — Sistema de Gestão de XXXX

**Objetivo:** descreva aqui a identidade visual pretendida (tom, público, contexto de uso — desktop/mobile, uso profissional/casual, etc.) e como ela se conecta ao propósito do sistema.

---

## 1. Princípios

> Liste os princípios que vão guiar decisões visuais e de interação. Exemplos de princípios comuns — adapte ou substitua pelos que fizerem sentido para o projeto:

- **Clareza acima de estética.** Contraste alto, poucos elementos por tela, foco na tarefa.
- **Consistência.** Mesmos componentes e cores em toda a aplicação, para reduzir erro do usuário.
- **Acessibilidade.** Contraste mínimo AA (WCAG 2.1), alvos de toque ≥ 44px.

---

## 2. Cores

### 2.1 Paleta principal

> Defina a paleta do projeto. Estrutura sugerida (primária, secundária, neutros, fundo, superfície):

| Papel | Nome | Hex | Uso |
|---|---|---|---|
| Primária | | | Cabeçalhos, botões principais, links, navegação |
| Primária clara | | | Estados hover/focus da primária |
| Secundária | | | Ações de confirmação |
| Neutro escuro | | | Texto principal |
| Neutro médio | | | Texto secundário, labels |
| Neutro claro | | | Bordas, divisores, fundo de cards |
| Fundo | | | Fundo geral da aplicação |
| Superfície | | | Cards, modais, formulários |

### 2.2 Cores semânticas (status e alertas)

| Papel | Hex | Uso |
|---|---|---|
| Sucesso | | Ação concluída, status ativo/válido |
| Atenção | | Estado próximo de um limite/vencimento |
| Erro/Crítico | | Falha, limite excedido, status vencido |
| Informação | | Notificações neutras, dicas |
| Inativo/Encerrado | | Status encerrado/inativo |

**Regra de uso:** cor semântica nunca é o único indicador — sempre acompanhada de ícone e texto (acessibilidade para daltonismo).

### 2.3 Modo escuro (opcional)
Se aplicável, manter a mesma paleta com inversão de luminância, preservando as cores semânticas sem alteração de matiz.

---

## 3. Tipografia

### 3.1 Fontes

| Uso | Fonte | Fallback |
|---|---|---|
| Interface (textos, botões, tabelas) | **Inter** | system-ui, -apple-system, Segoe UI, Roboto, sans-serif |
| Dados numéricos/tabulares (IDs, datas, contagens) | **Inter** (variante tabular — `font-variant-numeric: tabular-nums`) | mesma família |
| Documentos/relatórios exportados (PDF) | **Source Serif 4** ou **Georgia** | serif |

> Inter é um default razoável por legibilidade em telas pequenas e boa distinção entre caracteres semelhantes (0/O, 1/l/I) — troque se o projeto tiver identidade visual própria.

### 3.2 Escala tipográfica

| Estilo | Tamanho | Peso | Uso |
|---|---|---|---|
| Display | 28px / 1.2 | 700 (Bold) | Título de página |
| H1 | 22px / 1.3 | 700 | Título de seção |
| H2 | 18px / 1.3 | 600 (Semibold) | Subtítulo, título de card |
| Corpo | 16px / 1.5 | 400 (Regular) | Texto padrão, formulários |
| Corpo pequeno | 14px / 1.4 | 400 | Legendas, campos secundários |
| Label | 13px / 1.2 | 600 (Semibold), uppercase, letter-spacing 0.02em | Rótulos de campo |
| Caption | 12px / 1.3 | 400 | Notas de rodapé, timestamps |

**Mínimo mobile:** nenhum texto de interface abaixo de 14px (exceto caption), para leitura sem esforço no celular.

---

## 4. Espaçamento e grid

- **Unidade base:** 4px (escala: 4, 8, 12, 16, 24, 32, 48, 64)
- **Padding padrão de card/formulário:** 16px (mobile), 24px (desktop)
- **Grid desktop:** 12 colunas, gutter 24px, largura máxima de conteúdo 1200px
- **Grid mobile:** coluna única, margem lateral 16px
- **Altura mínima de alvo de toque:** 44px (botões, itens de lista, checkboxes)

---

## 5. Componentes-chave

> Liste os componentes centrais do sistema e suas variações. Exemplos genéricos:

### 5.1 Botões
- **Primário:** ação principal da tela
- **Secundário** (contorno, fundo transparente): ações alternativas
- **Destrutivo** (cor de erro): ações irreversíveis com confirmação obrigatória
- Bordas arredondadas: 8px. Altura mínima: 44px.

### 5.2 Cards
- Fundo branco/superfície, borda 1px, raio 12px, sombra sutil
- Indicador de status como *badge* colorido (cores semânticas, item 2.2)

### 5.3 Badges de status
- Formato pílula, fundo em tom claro da cor semântica (10% opacidade), texto na cor sólida
- Sempre com texto, nunca só cor (ex.: "● Ativo", "● Vencido")

### 5.4 Formulários
- Campos obrigatórios marcados com `*` em Erro/Crítico
- Validação visual imediata quando aplicável

### 5.5 Tabelas/listas
- Zebra striping sutil
- Cabeçalho fixo (sticky) ao rolar
- Números alinhados à direita, texto à esquerda

---

## 6. Iconografia

- Estilo de linha (*outline*), peso 1.5–2px, biblioteca recomendada: **Lucide** ou **Phosphor Icons**
- Ícones sempre acompanhados de texto/label em contextos críticos (nunca só ícone para ações de dados)

---

## 7. Tom de voz na interface

- Direto e técnico, sem jargão de marketing
- Confirmações de ações irreversíveis sempre explícitas: "Esta ação não pode ser desfeita."
- Mensagens de erro descrevem o problema e a ação corretiva

---

## 8. Aplicação por perfil de usuário

> Se o sistema tem múltiplos perfis (ver RF de usuários/perfis em [`../requisitos/requirements.md`](../requisitos/requirements.md)), descreva a ênfase visual de cada um.

| Perfil | Ênfase visual |
|---|---|
| | |
