# How-to — Implementação local do Orbisa / Helix + Langfuse

Plano de implementação local (Windows), baseado no `README.md` e nos prompts em `prompts/`.

## Pré-requisitos

| Item | Situação em 2026-10-17 | Ação |
|---|---|---|
| Docker 29.4 | instalado, Docker Desktop **parado** | abrir o Docker Desktop e deixar ~8 GB de RAM livres para ele |
| `make` | **não instalado** | `winget install GnuWin32.Make`, ou usar os comandos equivalentes abaixo |
| `uv` 0.9.30 / Python 3.12.10 | OK | — |
| `.env` | não existe | criar no Passo 1 |

---

## A ideia geral

Duas peças já estão prontas:

- **Helix**: um "motor de contexto". Ele lê dados de 7 sistemas (Slack, Confluence, Zendesk…), monta um
  **grafo de conhecimento com datas** no Neo4j e expõe esse grafo como ferramentas MCP (`search_context`,
  `find_entities`, `read_source`, `list_sources`).
- **Ask Orbisa**: um agente (Claude Agent SDK) que usa essas ferramentas para responder perguntas.

**O problema:** ninguém sabe se as respostas estão certas, por que estão certas ou erradas, nem quanto
custam. O que você vai construir é a pasta `src/observability/` (hoje vazia), que liga tudo ao **Langfuse**,
uma plataforma de observabilidade de LLM que roda localmente no Docker.

Uma comparação ajuda: o Helix e o agente são o carro. O Langfuse é o painel e a caixa-preta. Você vai
instalar os sensores.

---

## Fase 0: Subir o ambiente (infraestrutura)

### Passo 1: Configurar os segredos

```powershell
cd C:\Users\Raul\.git\ws-5-langfuse-observability
Copy-Item .env.example .env
# edite o .env e coloque sua ANTHROPIC_API_KEY
uv sync --python 3.12
uv run python scripts/init_env.py   # = "make env": gera as senhas e as chaves do Langfuse
```

*Por quê:* o Langfuse precisa de 5 bancos e serviços de apoio, e cada um precisa de uma senha. O script gera
todas para você (e pode ser rodado de novo sem problema).

### Passo 2: Subir a stack

```powershell
docker compose -f infra/docker-compose.yml --env-file .env up -d --build --wait   # = make up
```

Isso sobe cerca de 9 containers:

- **Neo4j**: o grafo.
- **helix-mcp**: o servidor de ferramentas, na porta 8765.
- **Langfuse**: `langfuse-web` (a interface, na porta 3000) e `langfuse-worker`.
- **Bases do Langfuse**: Postgres (configurações), ClickHouse (os traces, que é o volume pesado),
  Redis (fila) e MinIO (arquivos).
- **alert-sink**: recebe os alertas.

| Serviço | URL |
|---|---|
| Langfuse | http://localhost:3000 (login: `LANGFUSE_INIT_USER_EMAIL` / `LANGFUSE_INIT_USER_PASSWORD` do `.env`) |
| Helix MCP | http://localhost:8765/mcp |
| Neo4j Browser | http://localhost:7474 (usuário `neo4j`, senha `NEO4J_PASSWORD`) |
| MinIO | http://localhost:9090 |
| alert-sink | http://localhost:8787 |

*Verificação:* abra o Langfuse e faça login. O projeto "Ask Orbisa" já vem criado, mas **vazio**.

### Passo 3: Popular o grafo

```powershell
docker compose -f infra/docker-compose.yml --env-file .env run --rm helix-ingest   # = make ingest (~8 min, gasta API)
```

*Por quê:* o Claude Haiku lê os documentos e extrai entidades e fatos com validade no tempo
(`valid_at` / `invalid_at`). Se você bater em limite de requisições (rate limit), reduza `SEMAPHORE_LIMIT`
no `.env`. Pode rodar de novo sem medo: o que já foi carregado é pulado (`.helix/ingested.json`).

### Passo 4: Fazer a primeira pergunta

```powershell
docker compose -f infra/docker-compose.yml --env-file .env run --rm agent --id Q01   # = make ask Q=Q01
```

O agente responde e mostra a resposta esperada para comparação. Agora abra o Langfuse: **não aparece nada**.
É exatamente esse o problema que você vai resolver.

---

## Fase 1: Entender e decidir (prompts 00 a 05, sem código)

O repositório foi feito para ser seguido com o Claude Code, colando os prompts de `prompts/` em ordem. Os 6
primeiros são de **pensar**, não de programar:

| # | O que acontece | Resultado |
|---|---|---|
| 00 | Combina as regras de trabalho com o Claude: ele propõe, você decide | — |
| 01 | Ler o PDF `docs/orbisa-case-study.pdf` e entender os 4 tipos de falha: desatualizado, conflito, apelido (alias) e ausente | Resumo e perguntas em aberto |
| 02 | Ligar cada parte da arquitetura a uma pasta, ver o grafo no Neo4j e listar **o que não dá para ver** | Lista de lacunas |
| 03–04 | Escolher **um** problema e definir critérios mensuráveis (ex.: "acerto ≥ 0,8 nas perguntas temporais") | `decisions/01-problem-statement.md` |
| 05 | Aprender os conceitos (trace, span, OTLP) e escolher a arquitetura | `decisions/02-solution-design.md` |

**Os conceitos-chave do passo 05:**

- **Trace**: o caminho completo de *uma* pergunta, do início ao fim.
- **Span/observation**: cada etapa dentro desse caminho. Por exemplo: o agente pensa → chama o modelo →
  chama o MCP → busca no grafo.
- **OpenTelemetry (OTLP)**: um padrão aberto para enviar esses spans. O Langfuse é só mais um destino que
  entende esse padrão.
- **traceparent**: um cabeçalho HTTP que carrega o ID do trace de um processo para outro. É ele que permite
  costurar o agente e o Helix em um único trace.

---

## Fase 2: Construir (prompts 06 a 12, o código de verdade)

Cada prompt é um **incremento** que você confere visualmente no Langfuse.

### 06: Instrumentar o Helix → `src/observability/tracing/`

Cada ferramenta MCP passa a gerar uma observation. Por exemplo, o `search_context` registra a data `as_of`,
quantos fatos encontrou, quantos escondeu por serem "do futuro" e de quais fontes vieram.
*Regra de ouro:* se o Langfuse cair, o Helix continua funcionando. Isso é provado com testes.

### 07: Instrumentar o agente via OpenTelemetry

Você não escreve spans à mão. Basta configurar variáveis de ambiente (`CLAUDE_CODE_ENABLE_TELEMETRY=1`,
`OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=…/api/public/otel/v1/traces` etc.) e o próprio CLI do Claude passa a
enviar os spans com modelo, tokens e custo.
*Pegadinha:* dentro do Docker o endereço é `http://langfuse-web:3000`. No seu Windows é
`http://localhost:3000`.

### 08: Um trace por pergunta

Hoje o agente e o Helix geram traces separados. Aqui o servidor MCP lê o `traceparent` que chega no
cabeçalho e "pendura" as buscas dentro do trace do agente. Este incremento também adiciona sessions,
usuários e mascaramento de dados sensíveis.

### 09: Gestão de prompts

O system prompt passa a ser versionado no Langfuse, com rótulos `production` e `staging`. Assim você testa
uma regra nova sem mexer no código e sabe qual versão gerou cada resposta. Se o Langfuse estiver fora, o
agente usa o arquivo local.

### 10: Dataset e experimentos (a prova de que funciona)

1. As 22 perguntas de `data/orbisa/truth/questions.jsonl` viram um dataset no Langfuse.
2. Avaliadores:
   - **De código, sem LLM:** encontrou as fontes certas? Tem a linha "Sources:"?
   - **LLM-as-a-judge:** o Claude Sonnet dá nota correto (1), parcialmente correto (0,5) ou incorreto (0).
3. *Calibrar o juiz antes de confiar nele:* ele precisa aprovar as respostas esperadas e reprovar respostas
   propositalmente erradas.
4. Rodar as 22 perguntas com o prompt `production` e com o `staging` e comparar acerto por categoria, custo
   por resposta correta e latência p95.

### 11: Avaliação online e revisão humana

Avaliadores rodando no próprio Langfuse a cada pergunta nova (fidelidade aos fatos) e uma fila de anotação,
onde um humano revisa as respostas com nota mais baixa.

### 12: Dashboards e alertas

Painel com perguntas por dia, p95, custo por pergunta e notas. Alerta quando a fidelidade média da última
hora cair abaixo de 0,8. O alerta chega no `alert-sink` local, porque os dados não podem sair da máquina.
Você testa com uma "simulação de incêndio": um prompt ruim de propósito.

---

## Fase 3: Fechamento (prompt 13)

`decisions/03-readout.md`: um relatório de uma página para o CTO. Para cada critério de sucesso: a meta, o
valor medido, se passou ou falhou, e o link no Langfuse que prova. Os números vêm do Langfuse, não de
estimativa.

---

## Dicas práticas para Windows

| `make` | Equivalente sem make |
|---|---|
| `make env` | `uv run python scripts/init_env.py` |
| `make up` | `docker compose -f infra/docker-compose.yml --env-file .env up -d --build --wait` |
| `make ingest` | `docker compose -f infra/docker-compose.yml --env-file .env run --rm helix-ingest` |
| `make ask Q=Q07` | `docker compose -f infra/docker-compose.yml --env-file .env run --rm agent --id Q07` |
| `make ask-local Q=Q07` | `uv run python -m clients.agent --id Q07` |
| `make rebuild` | `docker compose -f infra/docker-compose.yml --env-file .env up -d --build --wait helix-mcp` |
| `make ps` | `docker compose -f infra/docker-compose.yml --env-file .env ps` |
| `make logs` | `docker compose -f infra/docker-compose.yml --env-file .env logs -f helix-mcp langfuse-web langfuse-worker` |
| `make down` | `docker compose -f infra/docker-compose.yml --env-file .env down` |
| `make test` | `uv run pytest -q` |
| `make alerts` | `uv run python scripts/alert_sink.py --tail` |

1. **Depois de mudar algo em `src/`**, rode `make rebuild` para o container do Helix pegar a mudança.
2. **Para testar "sem chaves do Langfuse"**, não edite o `.env`. Aponte `HELIX_ENV_FILE` para um arquivo
   vazio, porque os valores do `.env` sempre vencem os do shell.
3. **Este Langfuse v4 roda em modo `events_only`**: leia os dados pelos endpoints
   `/api/public/v2/observations`, `/api/public/v3/scores` e `/api/public/v2/metrics`. O endpoint v1 retorna
   erro.
4. **SDK Python do Langfuse fica na v4** (`langfuse>=4.18,<5`). Não atualize dependências.
5. **Custo estimado:** a ingestão, mais 22 perguntas × várias execuções com o Opus, mais o juiz Sonnet. Fique
   de olho no consumo da sua chave Anthropic.

## Checklist

- [ ] Docker Desktop rodando, `make` instalado
- [ ] `.env` criado com `ANTHROPIC_API_KEY`, segredos gerados
- [ ] Stack no ar, login no Langfuse OK
- [ ] Grafo populado (`make ingest`)
- [ ] `make ask Q=Q01` respondendo
- [ ] Prompts 00–05 → `decisions/01` e `decisions/02`
- [ ] 06 Helix instrumentado
- [ ] 07 Agente via OpenTelemetry
- [ ] 08 Um trace por pergunta
- [ ] 09 Gestão de prompts
- [ ] 10 Dataset + experimentos
- [ ] 11 Avaliação online + fila de revisão
- [ ] 12 Dashboard + alerta
- [ ] 13 `decisions/03-readout.md`
