# TITULO DO PROJETO

Repositório (monorepo) do **Sistema de Gestão de XXXX**: ....

| Pasta | Conteúdo |
|---|---|
| [`docs/`](docs/) | Requisitos, casos de uso, design system, arquitetura, plano de testes e CI/CD. |
| `backend/` | API (Spring Boot / Java) — ver [`docs/arquitetura/architecture.md`](docs/arquitetura/architecture.md). Criar quando a implementação de código for iniciada. |
| `frontend/` | Aplicação web (React / TypeScript) — idem. Criar quando a implementação de código for iniciada. |

Backend e frontend vivem no mesmo repositório (monorepo), com CI/CD segmentada por pasta — ver [`docs/cicd/CICD.md`](docs/cicd/CICD.md). Até que `backend/` e `frontend/` existam, o único conteúdo deste repositório é a documentação em `docs/`.
