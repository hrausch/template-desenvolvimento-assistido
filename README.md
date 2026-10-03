# TITULO DO PROJETO

Repositório de documentação e protótipo do **Sistema de Gestão de XXXX**: ....

| Pasta | Conteúdo |
|---|---|
| [`docs/`](docs/) | Requisitos, casos de uso, design system, arquitetura, plano de testes e CI/CD. |
| [`prototipo/`](prototipo/) | Protótipo funcional das telas (React + TypeScript, API mockada com `json-server`) — ver [`prototipo/README.md`](prototipo/README.md) para rodar. |

Este é o repositório de documentação do padrão descrito em [`docs/reference-architecture.md`](docs/reference-architecture.md). Os repositórios de backend (`xxxx-backend`) e frontend (`xxxx-frontend`) são seus **irmãos**, lado a lado — **não submódulos deste**, e este não é submódulo deles: de dentro do backend, a documentação é `../xxxx/docs/`. O porquê está em [`docs/reference-architecture.md §1`](docs/reference-architecture.md#1-organização-dos-repositórios).

Até os repos de código existirem, o único código executável aqui é o protótipo de frontend, que roda isolado (`prototipo/`) com API mockada, sem depender de nenhum backend real.

## Desenvolvimento assistido

| Comece por | Para |
|---|---|
| [`docs/agentes/`](docs/agentes/) | Os seis agentes de desenvolvimento, prontos para instalar nos repos de código, e o porquê do roster |
| [`docs/plano-desenvolvimento.md`](docs/plano-desenvolvimento.md) | O que construir, em que ordem, e por qual agente — é o briefing dos agentes |

**Dois passos ao criar os repos de código**, e esquecer o segundo é o erro mais comum: instalar os agentes, e registrar `permissions.additionalDirectories` apontando para este repositório — sem isso os agentes não conseguem ler a documentação e travam em prompt de permissão. Ver [`docs/agentes/README.md §5`](docs/agentes/README.md).
