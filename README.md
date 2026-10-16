# SemanticShelf

Busca semântica para catálogo de e-commerce: **vetorial + full-text com fusão RRF**, filtros aplicados *antes* da busca vetorial, entendimento de consulta em PT-BR (spellcheck, sinônimos, "até 3 mil") e rerank opcional por LLM — com **avaliação medida** contra uma busca tradicional honesta.

![Busca tradicional vs. híbrida](report/chart.svg)

## Resultados

32 consultas rotuladas (relevância graduada 0–3) sobre um catálogo sintético de 5 mil produtos. Tabela completa, por consulta: [`report/eval.md`](report/eval.md).

| Sistema | NDCG@10 | MRR | Recall@20 |
|---|---|---|---|
| Full-text (baseline: BM25, stemming, sem acentos, stopwords PT-BR) | 0,658 | 0,709 | 0,114 |
| Vetorial | 0,719 | 0,755 | 0,160 |
| **Híbrida (RRF)** | **0,726** | **0,754** | **0,163** |

NDCG@10 por tipo de consulta:

| | lexical | semântica | digitação | filtro |
|---|---|---|---|---|
| Full-text | 1,000 | 0,424 | 0,695 | 0,801 |
| Híbrida | 1,000 | 0,451 | **1,000** | **1,000** |

**Como ler esses números:**

- Eles foram medidos com o **embedding de fixture** (`HashingEmbeddingProvider`: hashing de palavras e trigramas, local, determinístico e sem custo). Ele capta sobreposição lexical e morfológica, **não semântica**. Por isso o ganho aparece em *digitação* e *filtro*, e as consultas semânticas sem nenhuma palavra em comum com o catálogo ("sapato pra academia", "máquina de fazer café") continuam em 0 em todos os modos.
- Os embeddings reais (`VOYAGE_API_KEY`) e o rerank com Claude (`ANTHROPIC_API_KEY`) foram feitos justamente para esse caso, mas **ainda não foram medidos**. Rode `pnpm eval` com as chaves para preencher a linha "híbrida + rerank".
- O Recall@20 é baixo por construção: consultas amplas ("tênis confortável para caminhada") têm centenas de itens relevantes no catálogo e só 20 cabem no top-20.
- Os sinônimos não foram ajustados olhando o conjunto de avaliação.

### Antes / depois

| Consulta | Full-text | Híbrida |
|---|---|---|
| `liquidificadr mondial` | 0,085: o erro de digitação quebra o match | **1,000**: corrigido para "liquidificador", marca Mondial vira filtro |
| `air fryer até 300 reais` | 0,206: "300" e "reais" viram termos de busca | **1,000**: extrai `maxPrice: 300` como pré-filtro |
| `laptop para programar` | 0,457 | **0,776**: sinônimo laptop → notebook |
| `fritadeira sem óleo` | 0,000 | **0,445**: sinônimo → "air fryer" |

## Arquitetura

```
Ingestão:  catálogo → enrich (buildSearchText, versionado por hash) → embed em lote → Postgres (pgvector + tsvector)

Consulta:  q ─► spellcheck ─► filtros implícitos ("até 3 mil", marca) ─► sinônimos
              ├── pré-filtro estruturado (WHERE)
              ├── vetorial (HNSW)  ──┐
              ├── full-text (GIN)  ──┼── RRF ── top-k
              └── rerank opcional (Claude, top-20) ──► resultados + scores explicáveis
```

- **Ports & adapters.** `EmbeddingProvider` (Voyage / hashing), `SearchIndex` (Postgres / memória), `Reranker` (Claude), `ProductStore`.
- **Pré-filtro, não pós-filtro.** Os filtros entram no mesmo `WHERE` da busca ANN, com `hnsw.iterative_scan`. Um teste dedicado garante que filtrar por uma marca rara, com uma consulta de outra categoria, nunca vaza outra marca e não zera o resultado.
- **Reindexação incremental.** O hash (versão da receita + texto) decide o que reembedar: alterar 100 de 1000 produtos reembeda 100. Os embeddings são versionados por modelo e o custo fica registrado em `ingest_runs`.
- **Scores transparentes.** Cada resultado traz `vector`, `text`, `rrf`, `rerank` e `matchedBy`.
- **O rerank nunca derruba a busca.** Se o LLM falhar ou recusar, os resultados voltam na ordem RRF, com `rerankError`.
- **Busca vazia inteligente.** Sugere a consulta sem os filtros que a zeraram e as subcategorias mais próximas.

## Como rodar

```bash
pnpm install
pnpm catalog:generate      # data/catalogo/produtos.json (5 mil itens, seed 42)
pnpm test                  # unitários + regressão de qualidade (+ integração pgvector se houver Docker)
pnpm eval                  # report/eval.md, report/eval.json, report/chart.svg
pnpm api                   # http://localhost:3333 (índice em memória se não houver DATABASE_URL)
pnpm web                   # http://localhost:5173 (demo com comparação lado a lado)
```

Com Postgres:

```bash
pnpm db:up                                  # pgvector/pgvector:pg16 + db/schema.sql
cp .env.example .env                        # DATABASE_URL, VOYAGE_API_KEY, ANTHROPIC_API_KEY
pnpm ingest && pnpm api
pnpm bench 50000                            # p50/p95/p99 da busca híbrida
```

### API

| Rota | Descrição |
|---|---|
| `GET /buscar?q=&mode=text\|vector\|hybrid&rerank=true&understand=false&minPrice=&maxPrice=&brands=a,b&categories=&inStock=&page=&pageSize=` | Busca, com `interpreted` (correções, sinônimos, filtros inferidos) e `suggestions` quando não há resultados |
| `GET /produtos/:id/similares?k=&sameCategory=` | "Veja também" pelo mesmo índice vetorial |
| `GET /health`, `GET /facets` | Estado (backend, modelo de embedding, reranker) e facetas para a UI |

## Estrutura

```
packages/core/src/  domain/ ports/ ingest/ search/ similar/ eval/ adapters/
apps/api/           Fastify + Zod
apps/web/           React + Vite + Tailwind
scripts/            generate-catalog, ingest, eval, chart, bench
tests/unit/         73 testes · tests/integration/ Testcontainers (pgvector)
db/schema.sql       pgvector HNSW + tsvector GIN + unaccent
```

## Pendências conhecidas

- **p95 < 150 ms em 50 mil produtos no Postgres: não medido.** O `scripts/bench.ts` está pronto, mas o ambiente de desenvolvimento não tem Docker. As latências em `report/eval.md` são do índice em memória e não servem como benchmark.
- A avaliação com embeddings reais e rerank depende das chaves de API.
- O teste de integração com Testcontainers é pulado sem Docker. Na CI (GitHub Actions) ele roda.
