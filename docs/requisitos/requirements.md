# Requisitos — Sistema de Gestão de Biotério

**Contexto:** sistema web, acessível também via celular, para controle de animais de laboratório (camundongos), procedimentos realizados e rastreabilidade para auditoria do conselho de ética (CEUA/CONCEA).

**Data:** 24/09/2026

---

## 1. Requisitos Funcionais

### RF01 — Cadastro de animais
- ID único do animal (ex: código de caixa + número individual, ou microchip/tatuagem, se aplicável)
- Espécie e linhagem (ex: C57BL/6, BALB/c)
- Data de nascimento (ou data de recebimento, se adquirido de fornecedor)
- Sexo
- Genótipo, quando aplicável (linhagens transgênicas/knockout)
- Origem (biotério próprio, fornecedor, outro laboratório)
- Caixa/gaiola atual, com histórico de transferências entre caixas
- Status do animal: vivo, em experimento, eutanasiado, óbito, transferido
- Vínculo com o(s) projeto(s)/protocolo(s) CEUA sob os quais o animal está sendo utilizado

### RF02 — Gestão de caixas/gaiolas
- ID da caixa e localização física (sala, estante, prateleira)
- Capacidade máxima e ocupação atual
- Alerta automático de superlotação
- Histórico de ocupantes por caixa
- Condições ambientais monitoradas (opcional): temperatura, ciclo claro/escuro

### RF03 — Registro de procedimentos
- Tipo de procedimento (coleta de sangue, administração de substância, cirurgia, eutanásia, etc.)
- Data e hora do procedimento
- Responsável pela execução (usuário do sistema)
- Protocolo CEUA vinculado (número de aprovação)
- Anestesia/analgesia utilizada, quando aplicável
- Observações e desfecho
- **Imutabilidade:** um procedimento salvo não pode ser editado ou apagado. Qualquer correção gera um novo registro vinculado ao original, preservando o histórico original intacto — requisito central para credibilidade em auditoria.

### RF04 — Protocolos e projetos de pesquisa
- Número do protocolo CEUA, data de aprovação e data de validade
- Pesquisador responsável pelo protocolo
- Espécie e quantidade de animais aprovados para o protocolo
- Alerta quando o uso real de animais se aproxima do limite aprovado
- Alerta de protocolo próximo do vencimento

### RF05 — Usuários e perfis de acesso
Perfis do sistema: **pesquisador**, **aluno**, **veterinário/responsável técnico**, **administrador**, **auditor CEUA** (acesso de leitura).

**Regras de vínculo aluno–pesquisador:**
- Um pesquisador pode ter vários alunos e vários projetos.
- Um aluno é vinculado a **1 pesquisador por vez** — o sistema não permite dois vínculos ativos simultâneos. Para trocar de pesquisador/projeto, o vínculo atual precisa ser encerrado antes de abrir um novo.
- O aluno enxerga apenas os projetos e dados do pesquisador ao qual está **atualmente** vinculado (vínculo ativo).
- Quando um vínculo é encerrado, o histórico do vínculo é preservado no sistema (quem esteve vinculado a quem, e quando) para fins de auditoria — mas o **acesso do aluno** àqueles dados é revogado, incluindo aos próprios registros que ele fez enquanto o vínculo estava ativo.
- Os registros de procedimentos feitos pelo aluno **nunca são apagados ou reatribuídos** quando o vínculo termina — permanecem associados a ele e ao projeto, visíveis para o pesquisador responsável e para o auditor CEUA.
- Se o aluno posteriormente entrar em projeto de outro pesquisador, ele passa a enxergar apenas os dados desse novo vínculo ativo — não acumula visibilidade de vínculos anteriores.

### RF06 — Trilha de auditoria (audit log)
- Registro de todas as alterações relevantes no sistema: quem, quando, o quê (estado anterior/novo)
- O log de auditoria é **somente leitura** — não editável nem por administrador
- Cobre no mínimo: cadastro/edição de animais, procedimentos, vínculos de usuários, protocolos

### RF07 — Relatórios
- Relatório por protocolo: animais utilizados x quantidade aprovada, lista de procedimentos realizados
- Relatório anual para CEUA/CONCEA, exportável em PDF e/ou Excel
- Histórico completo de um animal específico (linha do tempo de caixas, procedimentos e status)
- Relatório de vínculos ativos/encerrados por pesquisador (para prestação de contas de equipe)

### RF08 — Alertas e notificações
- Procedimentos agendados/pendentes
- Protocolo próximo do vencimento
- Superlotação de caixa
- Uso de animais se aproximando do limite aprovado no protocolo

---

## 2. Requisitos Não Funcionais

- **RNF01 — Acesso mobile:** interface responsiva, com fluxo otimizado para registro rápido em campo (dentro do biotério). Considerar leitura de QR code/código de barras na caixa para abrir o cadastro do animal sem digitação manual.
- **RNF02 — Funcionamento offline parcial:** app tipo PWA, com sincronização posterior, para tolerar conectividade instável dentro do biotério.
- **RNF03 — Segurança e backup:** backup automático dos dados; controle de acesso por perfil (RF05); dados de pesquisa não podem ser perdidos.
- **RNF04 — Responsividade:** mesma base funcional para desktop (cadastro mais completo) e mobile (registro rápido).
- **RNF05 — Auditabilidade:** todo dado relevante para a CEUA deve ser rastreável até o usuário e o momento exato do registro (ver RF03 e RF06).

---

## 3. Entidades Principais (conceitual)

- **Animal** — pertence a uma Caixa; está vinculado a um ou mais Procedimentos e a um Protocolo
- **Caixa** — contém vários Animais
- **Procedimento** — realizado em um Animal, por um Usuário, sob um Protocolo
- **Protocolo** — pertence a um Pesquisador; define espécie/quantidade aprovada
- **Usuário** — Pesquisador ou Aluno; Aluno possui um Vínculo ativo com um Pesquisador
- **Vínculo (Aluno–Pesquisador)** — histórico de associações, com status ativo/encerrado

---

## 4. Perguntas em aberto

- Quais tipos de procedimento precisam de campos específicos além dos genéricos (ex: cirurgia exige mais detalhes que coleta de sangue)?
- O sistema precisa gerar automaticamente o relatório anual no formato exigido pela CEUA/CONCEA da instituição, ou a exportação genérica (PDF/Excel) é suficiente para depois preencher o formulário oficial?
- Haverá integração com sistemas já usados pelo IFPR (ex: autenticação institucional)?
