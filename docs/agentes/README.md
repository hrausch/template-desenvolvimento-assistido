# Sistema de multiagentes — desenvolvimento assistido

Este diretório é sobre **o processo de trabalho**, não sobre o produto. O produto está nos outros diretórios de `docs/`. O que construir e em que ordem está em [`../plano-desenvolvimento.md`](../plano-desenvolvimento.md).

Escrito para **sobreviver ao fechamento da sessão**: quem abrir o projeto daqui a um mês instala os seis agentes com os comandos da seção 5, sem reconstruir o raciocínio e sem precisar do histórico de conversa.

---

## 1. O que o sistema de agentes resolve

O trabalho atravessa **duas stacks** e um **gradiente de risco** que vai de "CRUD com a regra escrita no caso de uso" até "arredondamento que passa no build verde e cobra o valor errado". Um executor único aplanaria esse gradiente, e é exatamente o gradiente que decide em que modelo cada passo roda.

Daí o roster da seção 4: a coluna "agente" de cada linha do plano carrega a decisão sobre risco, em vez de deixá-la para quem despacha.

---

## 2. Onde os arquivos ficam — e por quê

São **três artefatos com papéis diferentes**, e confundir os dois últimos é o erro fácil aqui:

| Artefato | Onde | Papel |
|---|---|---|
| Esta prosa e o `plano-desenvolvimento.md` | o repo de docs | Lidos como arquivo comum, pelo caminho do repositório irmão. Uma cópia só, sempre a atual. |
| `agentes/<nome>.md` | `docs/agentes/` | **A fonte** dos seis agentes: arquivos reais, com frontmatter válido, versionados. Não são descobertos como agentes aqui — são depósito. |
| `.claude/agents/<nome>.md` | cada repo de código | **A instalação:** cópia da fonte, feita com o comando da seção 5. É onde a descoberta olha, e cada repo recebe só os agentes que ali podem rodar. |

**A regra de descoberta**, conferida na documentação do Claude Code:

> Project subagents are discovered by **walking up** from the current working directory, so every `.claude/agents/` between there and the repository root is scanned.

A busca **sobe**, não desce — e o repo de docs é *irmão* dos repos de código, não ancestral. Logo um `.claude/agents/` dentro do repo de docs só seria descoberto com a sessão aberta nele próprio, nunca de dentro do backend. **É por isso que os seis arquivos ficam em `docs/agentes/`**, que é depósito lido por um comando de cópia, não diretório de descoberta.

Dentro de `.claude/agents/` a varredura **é** recursiva, e a identidade do agente vem só do campo `name` — subpasta não muda nome nem invocação.

A sessão é aberta **dentro de cada repo de código** (no backend nas fases de backend, no frontend nas de frontend). A instalação fica em:

| Repositório | Agentes em `.claude/agents/` |
|---|---|
| backend | `devjr`, `devpleno`, `devsenior`, `infra-specialist`, `revisor` |
| frontend | `frontend-specialist`, `infra-specialist`, `revisor` |

`infra-specialist` e `revisor` vão para os dois — **um corpo só**, com uma seção por repositório, copiado duas vezes. Se uma cópia divergir, resolve-se copiando de novo, não editando dentro de `.claude/agents/`.

---

## 3. Três fatos que desenham o sistema

**1. Fork não serve para baratear.** Existem dois tipos de subagente. O *fork* herda todo o contexto da conversa, mas roda obrigatoriamente no modelo da sessão — pedir Sonnet num fork de sessão Opus não funciona, o parâmetro é ignorado. Para cair em Sonnet tem de ser um agente **novo**, que começa sem contexto nenhum.

Isso não é problema porque o `plano-desenvolvimento.md` **é** o briefing. "Execute o passo 4.3 do plano; pare quando `./mvnw -B verify` estiver verde" é instrução completa e autossuficiente — e é por isso que o plano numera cada passo e nomeia o agente em cada linha.

**2. Esforço só existe no arquivo de definição.** Numa chamada avulsa, quem despacha escolhe apenas o modelo. Para que `effort: medium` e `effort: high` signifiquem coisas diferentes, precisa existir o arquivo em `.claude/agents/`. `/model` é comando do usuário e vale para a sessão inteira; o nível de esforço não é ajustável por comando.

**3. Delegar custa visibilidade.** A saída do subagente não aparece para o usuário — a sessão principal resume. E resumo descreve o que o agente *pretendeu* fazer. A regra, então: **a sessão principal confere o diff antes de dizer que está pronto.** Nunca "o agente disse que passou".

---

## 4. O roster

| Agente | `model` | `effort` | Escreve? | Pega o quê |
|---|---|---|---|---|
| `devjr` | sonnet | medium | sim | CRUD cuja regra está inteira no caso de uso, e migração Flyway derivada da entidade |
| `devpleno` | sonnet | high | sim | Feature com regra de negócio escrita e caminho único: unicidade, 409 em uso, alerta de limite, transição de status |
| `devsenior` | opus | high | sim | Dinheiro, isolamento entre contextos, autenticação, agregação, idempotência, escrita em lote |
| `frontend-specialist` | sonnet | high | sim | Tudo no repo de frontend |
| `infra-specialist` | sonnet | medium | sim | `pom.xml`, Docker, compose, properties, Flyway (configuração), CI/CD |
| `revisor` | opus | high | **não** | Os checkpoints. Sem ferramenta de escrita no cabeçalho |

### Como se escolhe

A pergunta não é "quão difícil é", é **"se isto estiver errado, o build fica vermelho?"**.

- **Fica vermelho** → `devjr` ou `devpleno`. Compilador e teste de integração são o oráculo; o agente barato basta porque o erro se denuncia.
- **Fica verde e o número sai errado** → `devsenior`. São três lugares: aritmética de dinheiro, isolamento entre contextos e autenticação. Em nenhum deles o teste que o próprio agente escreve é prova — por isso todos têm checkpoint obrigatório.

**Se `devjr` pegar poucos passos no seu projeto, isso é informação, não defeito**: significa que o sistema tem pouco CRUD sem regra. Nesse caso, dobre `devjr` em `devpleno` e perca só a economia. O inverso — usar `devjr` onde o plano diz `devsenior` — troca custo de modelo por defeito silencioso em dinheiro, e é o único atalho deste documento que não se paga.

**O knob de custo, se precisar:** `devjr` em `haiku`. Não faça isso em quem escreve migração Flyway — `ddl-auto: validate` falha no startup, mas DDL sutilmente errada (escala de `NUMERIC`, nulidade, constraint única ausente) passa por ele.

### Regras de trabalho, comuns aos seis

Estas quatro não estão nos documentos do produto porque não falam do produto — falam de como o agente trabalha. Todo arquivo de agente as repete, e é aqui que elas vivem:

1. **Decisão não escrita → parar e devolver a pergunta.** Não escolher. O plano concentra as decisões em aberto numa seção própria; se a dúvida cai ali, para mesmo que pareça óbvio.
2. **Pronto é o comando verde, rodado de fato.** `./mvnw -B verify` no backend; `npm run lint && tsc -b && npm test && npm run build` no frontend. Nunca "deve passar".
3. **Teste vermelho depois de refatoração é regressão até prova em contrário.** Entender a causa antes de tocar no teste; nunca afrouxar uma assertiva para o build passar.
4. **O relatório final lista os arquivos alterados e cola a saída do comando.** A sessão principal confere o diff.

**As regras sobre o sistema — isolamento, dinheiro, estrutura, schema, segurança — não estão aqui nem no plano.** Estão em [`../arquitetura/architecture.md`](../arquitetura/architecture.md), [`../reference-architecture.md`](../reference-architecture.md) e [`../plano-de-testes/README.md`](../plano-de-testes/README.md), que continuam valendo depois de o plano terminar. Os seis arquivos de agente mandam lê-las em toda tarefa.

---

## 5. Instalar

Os arquivos de agente usam o token `<DOCS>` onde entra o caminho do repo de docs. O comando abaixo o substitui na cópia, então **rode-o em vez de copiar à mão**.

No repositório de **backend**:

```bash
DOCS=../<nome>/docs            # ajuste <nome> para o seu repo de docs
mkdir -p .claude/agents
for a in devjr devpleno devsenior infra-specialist revisor; do
  sed "s|<DOCS>|$DOCS|g" "$DOCS/agentes/$a.md" > ".claude/agents/$a.md"
done
```

No repositório de **frontend**:

```bash
DOCS=../<nome>/docs
mkdir -p .claude/agents
for a in frontend-specialist infra-specialist revisor; do
  sed "s|<DOCS>|$DOCS|g" "$DOCS/agentes/$a.md" > ".claude/agents/$a.md"
done
```

Depois de instalar, `/agents` mostra os que foram descobertos. Se algum não aparecer, `/doctor` aponta nome duplicado ou frontmatter inválido.

### Dar acesso de leitura à documentação

Os seis agentes leem o repo de docs em toda tarefa, e **as ferramentas de arquivo só leem sem prompt dentro do diretório de trabalho**. Sem isso, um subagente trava esperando uma aprovação que ninguém vê — porque a saída dele não aparece para o usuário. Em cada repo de código, `.claude/settings.local.json`:

```json
{
  "permissions": {
    "additionalDirectories": ["/caminho/absoluto/para/<nome>"]
  }
}
```

O caminho é absoluto e específico da máquina — é por isso que mora no arquivo `local`, não versionado, em vez do `settings.json` compartilhado. Alternativa pontual: `claude --add-dir ../<nome>` ao abrir, ou `/add-dir ../<nome>` no meio da sessão.

**Este é o passo que mais se esquece.** Se um agente devolver "não consegui ler o plano", é aqui. Os seis abrem com a instrução de parar e avisar nesse caso, em vez de travar calados.

### Por que os arquivos ficam versionados aqui

1. **Sobrevivem à máquina.** `.claude/agents/` de cada repo de código é instalação; este diretório é a fonte versionada. Trocou de notebook, apagou a pasta sem querer: reinstala.
2. **Nada aqui é descoberto por acidente.** A descoberta sobe da cwd (seção 2), e este diretório não está no caminho de subida de nenhum repo de código.

---

## 6. Checkpoints de revisão

O plano define os checkpoints; este diretório define como revisá-los. Em cada um:

- Se a sessão está em Opus, ela revisa direto — e deve, porque já tem o contexto.
- O subagente `revisor` serve quando a sessão está em Sonnet, ou quando vale uma leitura que **não** viu o código ser escrito. Esta segunda é a mais valiosa nos passos de `devsenior`: quem escreveu a aritmética tende a reler o que quis dizer.
- `/code-review ultra` roda uma revisão multiagente da branch. **É o usuário quem dispara, e é cobrado à parte** — nenhum agente aciona sozinho.

**Obrigatórios, no mínimo:** o passo que estabelece o isolamento entre contextos, os passos de autenticação, **o primeiro passo escrito com a estrutura completa** (antes de o padrão ser repetido), e todo passo de dinheiro, agregação ou idempotência.

**O de melhor retorno é o do primeiro passo**: revisar a primeira feature escrita com a estrutura completa, antes de o padrão ser repetido mais N vezes. Um erro pego ali é um erro; pego no fim, são N.

---

## 7. Armadilhas

- **A pasta que contém os repos não é repositório git.** `git diff` nela falha. De dentro de um repo funciona; da pasta-mãe, é `git -C <repo> diff`.
- **A documentação não é submódulo** dos repos de código. Era, no padrão anterior: o ponteiro atrasava sozinho, e uma cópia defasada da arquitetura é pior que nenhuma — quem a lê acerta a sintaxe e erra a regra. Agora existe uma cópia só, e o preço é a linha de `additionalDirectories` da seção 5.
- **Editar documentação é sempre no repo de docs.** Nenhum agente tem motivo para escrever ali: os cinco que escrevem trabalham nos repos de código.
- Existe uma configuração de **modelo padrão de subagente**: com ela em Sonnet, toda delegação já nasce barata sem precisar pedir. Ela **não** rebaixa `devsenior` nem `revisor`, porque `model` no arquivo de definição tem precedência.
