# Design System — Sistema de Gestão de Biotério

**Objetivo:** identidade visual clara, sóbria e funcional, adequada a um sistema de uso científico/clínico (registro de dados sensíveis para auditoria CEUA/CONCEA), legível em desktop e celular, inclusive em ambiente de biotério (uso com luvas, telas pequenas, pressa).

---

## 1. Princípios

- **Clareza acima de estética.** Cada tela é preenchida rapidamente, muitas vezes de pé, no biotério. Contraste alto, toques grandes, poucos elementos por tela.
- **Confiança e seriedade.** É um sistema de auditoria científica — a linguagem visual deve transmitir precisão e institucionalidade, não "app comercial".
- **Consistência.** Mesmos componentes e cores em toda a aplicação, para reduzir erro de registro.
- **Acessibilidade.** Contraste mínimo AA (WCAG 2.1), textos legíveis em luz de laboratório, alvos de toque ≥ 44px.

---

## 2. Cores

### 2.1 Paleta principal

| Papel | Nome | Hex | Uso |
|---|---|---|---|
| Primária | Azul Institucional | `#1E3A5F` | Cabeçalhos, botões principais, links, navegação |
| Primária clara | Azul Suave | `#4A6B8A` | Estados hover/focus da primária |
| Secundária | Verde Biológico | `#2E7D5B` | Ações de confirmação (salvar, aprovar procedimento) |
| Neutro escuro | Grafite | `#1F2429` | Texto principal |
| Neutro médio | Cinza-azulado | `#5B6770` | Texto secundário, labels |
| Neutro claro | Cinza-névoa | `#E7EAED` | Bordas, divisores, fundo de cards |
| Fundo | Branco-gelo | `#F7F8FA` | Fundo geral da aplicação |
| Superfície | Branco | `#FFFFFF` | Cards, modais, formulários |

### 2.2 Cores semânticas (status e alertas)

| Papel | Hex | Uso |
|---|---|---|
| Sucesso | `#2E7D5B` | Procedimento concluído, vínculo ativo, protocolo válido |
| Atenção | `#B8860B` | Protocolo perto do vencimento, caixa perto da capacidade máxima |
| Erro/Crítico | `#B3261E` | Superlotação, protocolo vencido, limite de animais excedido |
| Informação | `#2B6CB0` | Notificações neutras, dicas |
| Inativo/Encerrado | `#8A8F94` | Vínculo encerrado, animal com status "óbito"/"transferido" |

**Regra de uso:** cor semântica nunca é o único indicador — sempre acompanhada de ícone e texto (acessibilidade para daltonismo, comum o suficiente para não assumir).

### 2.3 Modo escuro (opcional, fase 2)
Manter a mesma paleta com inversão de luminância (fundo `#12161A`, superfície `#1B2126`, texto `#E7EAED`), preservando as cores semânticas sem alteração de matiz.

---

## 3. Tipografia

### 3.1 Fontes

| Uso | Fonte | Fallback |
|---|---|---|
| Interface (textos, botões, tabelas) | **Inter** | system-ui, -apple-system, Segoe UI, Roboto, sans-serif |
| Dados numéricos/tabulares (IDs, datas, contagens) | **Inter** (variante tabular/monoespaçada numérica — `font-variant-numeric: tabular-nums`) | mesma família |
| Documentos/relatórios exportados (PDF) | **Source Serif 4** ou **Georgia** | serif |

Justificativa: Inter tem excelente legibilidade em telas pequenas e boa distinção entre caracteres semelhantes (0/O, 1/l/I) — importante para IDs de animais e números de protocolo.

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
| Dado crítico (ID, contagem) | 16–20px | 600 | Números de destaque em cards |

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

### 5.1 Botões
- **Primário** (Azul Institucional, texto branco): ação principal da tela (ex.: "Salvar procedimento")
- **Secundário** (contorno Azul Institucional, fundo transparente): ações alternativas (ex.: "Cancelar")
- **Destrutivo** (Erro/Crítico): ações irreversíveis com confirmação obrigatória (ex.: "Encerrar vínculo")
- Bordas arredondadas: 8px. Altura mínima: 44px.

### 5.2 Cards de animal/caixa
- Fundo branco, borda 1px `#E7EAED`, raio 12px, sombra sutil (`0 1px 3px rgba(0,0,0,0.08)`)
- Indicador de status como *badge* colorido (cores semânticas, item 2.2) no canto superior direito

### 5.3 Badges de status
- Formato pílula, fundo em tom claro da cor semântica (10% opacidade), texto na cor sólida
- Sempre com texto, nunca só cor (ex.: "● Ativo", "● Vencido")

### 5.4 Formulário de procedimento
- Campos obrigatórios marcados com `*` em Erro/Crítico
- Campo de protocolo CEUA com autocomplete e validação visual imediata (verde = protocolo válido e dentro do limite; amarelo = perto do limite; vermelho = vencido/excedido)
- Após salvar: confirmação visual clara + aviso de que o registro é imutável, antes da confirmação final

### 5.5 Tabelas/listas (auditoria, relatórios)
- Zebra striping sutil (`#F7F8FA` nas linhas pares)
- Cabeçalho fixo (sticky) ao rolar
- Números alinhados à direita, texto à esquerda

### 5.6 QR/leitura de caixa (mobile)
- Botão de câmera com alto contraste, fixo no canto inferior direito, sempre acessível com uma mão

---

## 6. Iconografia

- Estilo de linha (*outline*), peso 1.5–2px, biblioteca recomendada: **Lucide** ou **Phosphor Icons**
- Ícones sempre acompanhados de texto/label em contextos críticos (nunca só ícone para ações de dados)

---

## 7. Tom de voz na interface

- Direto e técnico, sem jargão de marketing ("Salvar procedimento", não "Ótimo trabalho! Vamos salvar isso 🎉")
- Confirmações de ações irreversíveis sempre explícitas: "Esta ação não pode ser desfeita."
- Mensagens de erro descrevem o problema e a ação corretiva (ex.: "Caixa 12 já está no limite de 5 animais. Selecione outra caixa ou remova um animal.")

---

## 8. Aplicação por perfil de usuário

| Perfil | Ênfase visual |
|---|---|
| Pesquisador/Aluno (registro em campo) | Fluxo mobile-first, poucos campos por tela, leitura de QR |
| Auditor CEUA | Foco em tabelas, filtros e exportação; densidade de informação maior, modo desktop priorizado |
| Administrador | Painéis de configuração, gestão de vínculos e alertas |
