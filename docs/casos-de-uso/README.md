# Casos de Uso

Documentação de casos de uso do Sistema de Gestão de Biotério, derivada dos requisitos funcionais em [`../requisitos/requirements.md`](../requisitos/requirements.md).

Estrutura de cada caso de uso conforme [`uc-template.md`](uc-template.md).

## Índice

| Caso de Uso | Requisitos | Atores |
|---|---|---|
| [UC 01 — Cadastro de Animal](uc01-cadastro-de-animal.md) | RF01, RF02, RF04, RF05 | Pesquisador, Aluno, Veterinário |
| [UC 02 — Consulta de Animal](uc02-consulta-de-animal.md) | RF01, RF03, RF05, RF08 | Pesquisador, Aluno, Veterinário, Administrador, Auditor CEUA |
| [UC 03 — Cadastro e Gestão de Caixa/Gaiola](uc03-cadastro-e-gestao-de-caixa.md) | RF02, RF08 | Veterinário, Administrador |
| [UC 04 — Transferência de Animal entre Caixas](uc04-transferencia-de-animal-entre-caixas.md) | RF01, RF02, RF08 | Pesquisador, Aluno, Veterinário |
| [UC 05 — Registro de Procedimento](uc05-registro-de-procedimento.md) | RF03, RF04, RF05, RF06 | Pesquisador, Aluno, Veterinário |
| [UC 06 — Correção de Procedimento](uc06-correcao-de-procedimento.md) | RF03, RF06 | Pesquisador, Aluno, Veterinário |
| [UC 07 — Cadastro de Protocolo/Projeto CEUA](uc07-cadastro-de-protocolo.md) | RF04 | Pesquisador, Administrador |
| [UC 08 — Consulta de Protocolo](uc08-consulta-de-protocolo.md) | RF04, RF05, RF08 | Pesquisador, Aluno, Veterinário, Auditor CEUA, Administrador |
| [UC 09 — Gestão de Usuários e Perfis de Acesso](uc09-gestao-de-usuarios-e-perfis.md) | RF05 | Administrador |
| [UC 10 — Vínculo Aluno–Pesquisador](uc10-vinculo-aluno-pesquisador.md) | RF05, RF06 | Pesquisador, Administrador |
| [UC 11 — Consulta de Trilha de Auditoria](uc11-consulta-de-trilha-de-auditoria.md) | RF06 | Auditor CEUA, Administrador |
| [UC 12 — Geração de Relatórios](uc12-geracao-de-relatorios.md) | RF07 | Pesquisador, Administrador, Auditor CEUA |
| [UC 13 — Consulta de Alertas e Notificações](uc13-consulta-de-alertas.md) | RF08 | Pesquisador, Aluno, Veterinário, Administrador |
| [UC 14 — Configuração de Alertas](uc14-configuracao-de-alertas.md) | RF08 | Administrador |

## Notas gerais

- **Escopo de dados do Aluno:** em todos os casos de uso acima, um usuário com perfil Aluno só enxerga animais, caixas, procedimentos e protocolos do pesquisador ao qual está **atualmente** vinculado (UC10). Ao trocar de vínculo, perde a visibilidade dos dados do vínculo anterior, mesmo de registros que ele próprio criou — os registros em si nunca são apagados ou reatribuídos.
- **Imutabilidade de procedimentos:** UC05 e UC06 formam um par — nunca existe edição direta de um procedimento já salvo, apenas registro (UC05) e correção como novo registro vinculado (UC06).
- **Somente leitura:** o perfil Auditor CEUA nunca tem acesso de escrita em nenhum caso de uso — apenas consulta (UC02, UC08, UC11, UC12, UC13).
