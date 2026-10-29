# 01 — Problem statement

Decidido em 2026-10-17 (prompts 03 e 04).

## Frase do problema (candidato E)

> Quando uma resposta está errada, ou certa por acaso, o time não consegue dizer se o problema está nos
> dados, na extração, na busca ou no raciocínio do agente, porque não há visibilidade do que o Helix
> devolveu em cada passo. Consequência: defeitos no grafo passam despercebidos, e as correções são
> feitas no lugar errado.

**Evidência que motivou a escolha:** em 2026-10-15, `search_context("who approves refunds")` devolve a regra
"refund approval moved from Finance to RevOps" (`slack/revops/2026-03-03.json`) como *no longer true*
(válida de 2026-03-03 a 2026-03-03). Nenhum fato válido diz que RevOps aprova. Mesmo assim o agente acertou
a Q01, lendo os documentos com `read_source`: certo apesar do grafo.

## Usuário principal

**O time de Plataforma e IA**, que mantém o Helix e o Ask Orbisa e precisa saber *onde* corrigir.
Ken Watanabe (Suporte) e Tomás Herrera (RevOps) entram como revisores que confirmam a verdade (prompt 11).

## Escopo

**Diagnóstico profundo:** as 6 perguntas **temporal + âncora** de `data/orbisa/truth/questions.jsonl`.

| Categoria | IDs |
|---|---|
| âncora | Q01 |
| temporal | Q02, Q03, Q07, Q09, Q21 |

As 22 perguntas rodam só como referência para a nota geral.

## Critérios de sucesso

| # | Critério | Meta | Como medir |
|---|---|---|---|
| C1 | Toda resposta é rastreável até os fatos | 100% das execuções das 6 perguntas | Script verifica **todos** os traces de uma rodada pela API do Langfuse: um trace com chamadas de ferramenta → fatos (status, valid_from/valid_until, fontes) → resposta |
| C2 | O grafo devolve o fato certo na data certa | ≥ 0,8 | Nota de código por pergunta: fração dos `expected_facts` que aparecem como `true on as_of` no que o Helix devolveu. Deve falhar hoje (defeito da RevOps) |
| C3 | Toda falha tem local identificado | 100% das perguntas com nota < 1 | Nota categórica `failure_location` (dados / extração / busca / raciocínio), dada por um juiz sobre o trace e revisada por um humano, com o link do trace como evidência |
| C4 | Detectar "certo por acaso" | Todo caso sinalizado | Regra: acerto = 1 e C2 < 1 → sinal |
| C5 | Acerto nas 6 perguntas | ≥ 0,8 de média em 3 rodadas | Experimento no Langfuse com juiz `claude-sonnet-5-5` (correct 1 / partial 0,5 / incorrect 0) |

## Restrições

| # | Restrição |
|---|---|
| R1 | Só self-hosted: Langfuse v4 no Docker Compose, sem Langfuse Cloud |
| R2 | O agente e o Helix funcionam sem chaves do Langfuse e com o Langfuse fora do ar, provado com testes |
| R3 | Nenhum dado de cliente sai pela observabilidade. A única saída externa são chamadas de modelo à Anthropic |
| R4 | Não reconstruir o Helix (grafo, ontologia, busca), salvo defeito medido e decisão explícita |
| R5 | Não alterar as perguntas de referência |
| R6 | Sem recursos Enterprise: o mascaramento é feito no SDK, no nosso código |
| R7 | Código novo em `src/observability/`. Mudanças pequenas em `src/context/` e `src/clients/` |
| R8 | O agente roda pela assinatura (`HELIX_AGENT_PROVIDER=subscription`). A instrumentação não pode quebrar esse modo |
| R9 | Orçamento de API (ingestão + juiz + avaliadores) ≤ US$ 5. Estimativa: ~US$ 2,80, dos quais ~US$ 1 da ingestão já foi gasto |

## Fora do escopo

| # | Item |
|---|---|
| F1 | Diagnóstico profundo das outras 16 perguntas (conflito, apelido, multi-fonte, premissa falsa, consulta simples) |
| F2 | Corrigir o grafo ou a extração. Hoje: medir e localizar. Corrigir exige decisão explícita (R4) |
| F3 | Observabilidade da ingestão (custo e traces do `make ingest`) |
| F4 | Novas fontes ou sistemas |
| F5 | Trocar o modelo do agente |
| F6 | Recursos Enterprise e Langfuse Cloud |

## Riscos em aberto

| # | Risco | Mitigação |
|---|---|---|
| K1 | **Juiz com dados de clientes:** assumimos que o juiz pode receber respostas e fatos, como uma chamada de modelo do Helix v0 (R3). O CTO não confirmou (pergunta em aberto #5) | Registrado. Se for negado, usar só notas de código (C1, C2, C4) |
| K2 | **Mascaramento indefinido:** o brief não diz quais campos mascarar (pergunta em aberto #4) | Decidir no design (prompt 05) e implementar no prompt 08 |
| K3 | **Amostra pequena:** 6 perguntas, com respostas que variam entre execuções | 3 rodadas no C5; relatar sempre quantas execuções embasam cada número |
| K4 | **C2 depende de casar textos:** `expected_facts` são frases escritas à mão e os fatos do grafo são extraídos pelo Haiku, com redação diferente | Definir a regra de correspondência no design e validá-la contra a Q01 antes de confiar |
| K5 | **Juiz não calibrado** | Calibração antes de usar: as respostas esperadas devem tirar "correct" e as armadilhas não |
| K6 | **Cota da assinatura e RAM:** cerca de 18 a 40 execuções do agente consomem a cota de uso; o host fica com ~0,5 GB livre com a stack no ar | Rodar em lotes pequenos; usar `make down` quando não estiver em uso |
