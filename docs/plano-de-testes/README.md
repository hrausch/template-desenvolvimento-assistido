# Plano de Testes

Estratégia de teste do **Sistema de Gestão de XXXX**, derivada de [`../requisitos/requirements.md`](../requisitos/requirements.md) e [`../casos-de-uso/`](../casos-de-uso/).

O objetivo do teste aqui é **pegar regressão em regra de negócio** — não atingir meta de cobertura nem produzir evidência de processo. O que custa manutenção sem pegar bug fica de fora, e cada corte está justificado abaixo.

## 1. Duas camadas, não quatro

| Camada | Por quê |
|---|---|
| **Entidade** (unitário, sem Spring) | Rápido e barato. É onde mora a invariante, graças à Rich Entity. |
| **Fluxo HTTP** (integração) | Cobre controller, aplicação e persistência de uma vez, pelo caminho que o usuário usa de verdade. |

`@DataJpaTest` entra como terceiro caso **só quando a query tem lógica própria**.

### O que ficou de fora, e por quê

- **Camada intermediária com Mockito.** Para CRUD, ela testa a mesma coisa que o teste de integração, com o dobro da manutenção e acoplada à implementação — quebra quando você renomeia um método, não quando quebra a regra.
- **Testcontainers.** O banco em memória serve até algum fluxo precisar de comportamento específico do PostgreSQL. Adicione quando esse fluxo aparecer, não antes.
- **Gate de cobertura.** O relatório pode ser gerado para consulta (`./mvnw verify`), mas **não quebra o build**. Gate de porcentagem produz teste escrito para o número: cobre getters, deixa a regra de borda sem assertiva, e dá um verde que não significa nada.
- **Matriz de rastreabilidade e IDs de caso de teste** (`TC-X.Y`). Amarrar cada teste a uma tabela em outro documento custa um commit por teste escrito e não pega bug nenhum. O nome do teste já diz o que ele cobre.

## 2. Backend

| O que testar | Tipo | Ferramenta |
|---|---|---|
| Rich Entity: invariante, factory, transição de estado | Unitário | JUnit + AssertJ, sem Spring |
| Fluxo HTTP de um caso de uso: caminho feliz + o erro que importa | Integração | `BaseIntegrationTest` |
| Query customizada ou constraint não óbvia | Integração | `@DataJpaTest` — só quando a query tem lógica própria |

Regras:

- **`BaseIntegrationTest`** (`@SpringBootTest + @AutoConfigureMockMvc + @ActiveProfiles("test") + @Transactional`) oferece os helpers de alto nível: criar usuário autenticado, criar usuário com contexto, executar request com token e com contexto. **Não repetir setup de autenticação em cada teste** — é esse arquivo que decide se os próximos N testes são baratos.
- **O único teste obrigatório por feature com recurso contextual é o de isolamento:** usuário do contexto A recebe **404** ao acessar recurso do contexto B. Sem ele a regressão passa despercebida e vaza dado de outro cliente.
- **Regra de cálculo tem teste próprio.** É aqui que o bug caro aparece.
- **Em teste de cálculo, o valor esperado é um literal conferido à mão.** Calcular o esperado com a mesma expressão do código prova apenas que o código concorda consigo mesmo — e passa verde com a fórmula errada.
- **Agregação se testa pela decomposição, não pelo total.** Soma errada quase sempre acerta o total e erra a divisão por categoria ou por membro. Quando o caso de uso traz identidades de consistência, cada identidade é um teste.
- **Idempotência se testa chamando duas vezes**, não inspecionando o guard: a segunda chamada não cria nada e não devolve erro.
- **CRUD sem regra não precisa de teste por operação.** Um fluxo que cria e lê de volta já cobre o caminho.
- Nomenclatura: `should_x_when_y`.
- **Teste vermelho depois de refatoração é sinal de regressão**, não de teste desatualizado. Entender a causa antes de alterar o teste.

Comando: `./mvnw test` (com relatório de cobertura, para consulta: `./mvnw verify`).

## 3. Frontend

**Vitest + Testing Library + MSW.** Ordem de prioridade: utils puros → hooks → fluxos críticos (login, o fluxo central do sistema, troca de contexto).

- MSW intercepta HTTP nos testes — **não mockar o cliente HTTP diretamente**.
- Componente visual sem lógica não precisa de teste próprio; testar pelo fluxo que o usa.
- **Teste obrigatório, se o sistema tem contexto:** trocar de contexto não deixa dado do anterior em tela nem em cache.
- **Armadilha conhecida:** em ambiente `jsdom` o `axios` usa o adapter XHR, que o MSW não intercepta — toda chamada falha com `AxiosError: Network Error` mesmo com o handler certo registrado, e o sintoma não aponta para a causa. Force `axios.defaults.adapter = 'http'` no setup do Vitest, que usa o módulo `http` do Node por baixo e é interceptado de verdade.
- Comando: `npm test`.
