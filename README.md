# SemanticShelf

Busca semÃ¢ntica para catÃ¡logo de e-commerce: **vetorial + full-text com fusÃ£o RRF**, filtros aplicados *antes* da busca vetorial, entendimento de consulta em PT-BR (spellcheck, sinÃ´nimos, "atÃ© 3 mil") e rerank opcional por LLM â€” com **avaliaÃ§Ã£o medida** contra uma busca tradicional honesta.

![Busca tradicional vs. hÃ­brida](report/chart.svg)

## Resultados

32 consultas rotuladas (relevÃ¢ncia graduada 0â€“3) sobre um catÃ¡logo sintÃ©tico de 5 mil produtos. Tabela completa, por consulta: [`report/eval.md`](report/eval.md).

| Sistema | NDCG@10 | MRR | Recall@20 |
|---|---|---|---|
| Full-text (baseline: BM25, stemming, sem acentos, stopwords PT-BR) | 0,658 | 0,709 | 0,114 |
| Vetorial | 0,719 | 0,755 | 0,160 |
| **HÃ­brida (RRF)** | **0,726** | **0,754** | **0,163** |

NDCG@10 por tipo de consulta:

| | lexical | semÃ¢ntica | digitaÃ§Ã£o | filtro |
|---|---|---|---|---|
| Full-text | 1,000 | 0,424 | 0,695 | 0,801 |
| HÃ­brida | 1,000 | 0,451 | **1,000** | **1,000** |

**Como ler esses nÃºmeros:**

- Eles foram medidos com o **embedding de fixture** (`HashingEmbeddingProvider`: hashing de palavras e trigramas, local, determinÃ­stico e sem custo). Ele capta sobreposiÃ§Ã£o lexical e morfolÃ³gica, **nÃ£o semÃ¢ntica**. Por isso o ganho aparece em *digitaÃ§Ã£o* e *filtro*, e as consultas semÃ¢nticas sem nenhuma palavra em comum com o catÃ¡logo ("sapato pra academia", "mÃ¡quina de fazer cafÃ©") continuam em 0 em todos os modos.
- Os embeddings reais (`VOYAGE_API_KEY`) e o rerank com Claude (`ANTHROPIC_API_KEY`) foram feitos justamente para esse caso, mas **ainda nÃ£o foram medidos**. Rode `pnpm eval` com as chaves para preencher a linha "hÃ­brida + rerank".
- O Recall@20 Ã© baixo por construÃ§Ã£o: consultas amplas ("tÃªnis confortÃ¡vel para caminhada") tÃªm centenas de itens relevantes no catÃ¡logo e sÃ³ 20 cabem no top-20.
- Os sinÃ´nimos nÃ£o foram ajustados olhando o conjunto de avaliaÃ§Ã£o.

### Antes / depois

| Consulta | Full-text | HÃ­brida |
|---|---|---|
| `liquidificadr mondial` | 0,085: o erro de digitaÃ§Ã£o quebra o match | **1,000**: corrigido para "liquidificador", marca Mondial vira filtro |
| `air fryer atÃ© 300 reais` | 0,206: "300" e "reais" viram termos de busca | **1,000**: extrai `maxPrice: 300` como prÃ©-filtro |
| `laptop para programar` | 0,457 | **0,776**: sinÃ´nimo laptop â†’ notebook |
| `fritadeira sem Ã³leo` | 0,000 | **0,445**: sinÃ´nimo â†’ "air fryer" |

## Arquitetura

```
IngestÃ£o:  catÃ¡logo â†’ enrich (buildSearchText, versionado por hash) â†’ embed em lote â†’ Postgres (pgvector + tsvector)

Consulta:  q â”€â–º spellcheck â”€â–º filtros implÃ­citos ("atÃ© 3 mil", marca) â”€â–º sinÃ´nimos
              â”œâ”€â”€ prÃ©-filtro estruturado (WHERE)
              â”œâ”€â”€ vetorial (HNSW)  â”€â”€â”
              â”œâ”€â”€ full-text (GIN)  â”€â”€â”¼â”€â”€ RRF â”€â”€ top-k
              â””â”€â”€ rerank opcional (Claude, top-20) â”€â”€â–º resultados + scores explicÃ¡veis
```

- **Ports & adapters.** `EmbeddingProvider` (Voyage / hashing), `SearchIndex` (Postgres / memÃ³ria), `Reranker` (Claude), `ProductStore`.
- **PrÃ©-filtro, nÃ£o pÃ³s-filtro.** Os filtros entram no mesmo `WHERE` da busca ANN, com `hnsw.iterative_scan`. Um teste dedicado garante que filtrar por uma marca rara, com uma consulta de outra categoria, nunca vaza outra marca e nÃ£o zera o resultado.
- **ReindexaÃ§Ã£o incremental.** O hash (versÃ£o da receita + texto) decide o que reembedar: alterar 100 de 1000 produtos reembeda 100. Os embeddings sÃ£o versionados por modelo e o custo fica registrado em `ingest_runs`.
- **Scores transparentes.** Cada resultado traz `vector`, `text`, `rrf`, `rerank` e `matchedBy`.
- **O rerank nunca derruba a busca.** Se o LLM falhar ou recusar, os resultados voltam na ordem RRF, com `rerankError`.
- **Busca vazia inteligente.** Sugere a consulta sem os filtros que a zeraram e as subcategorias mais prÃ³ximas.

## Como rodar

```bash
pnpm install
pnpm catalog:generate      # data/catalogo/produtos.json (5 mil itens, seed 42)
pnpm test                  # unitÃ¡rios + regressÃ£o de qualidade (+ integraÃ§Ã£o pgvector se houver Docker)
pnpm eval                  # report/eval.md, report/eval.json, report/chart.svg
pnpm api                   # http://localhost:3333 (Ã­ndice em memÃ³ria se nÃ£o houver DATABASE_URL)
pnpm web                   # http://localhost:5173 (demo com comparaÃ§Ã£o lado a lado)
```

Com Postgres:

```bash
pnpm db:up                                  # pgvector/pgvector:pg16 + db/schema.sql
cp .env.example .env                        # DATABASE_URL, VOYAGE_API_KEY, ANTHROPIC_API_KEY
pnpm ingest && pnpm api
pnpm bench 50000                            # p50/p95/p99 da busca hÃ­brida
```

### API

| Rota | DescriÃ§Ã£o |
|---|---|
| `GET /buscar?q=&mode=text\|vector\|hybrid&rerank=true&understand=false&minPrice=&maxPrice=&brands=a,b&categories=&inStock=&page=&pageSize=` | Busca, com `interpreted` (correÃ§Ãµes, sinÃ´nimos, filtros inferidos) e `suggestions` quando nÃ£o hÃ¡ resultados |
| `GET /produtos/:id/similares?k=&sameCategory=` | "Veja tambÃ©m" pelo mesmo Ã­ndice vetorial |
| `GET /health`, `GET /facets` | Estado (backend, modelo de embedding, reranker) e facetas para a UI |

## Estrutura

```
packages/core/src/  domain/ ports/ ingest/ search/ similar/ eval/ adapters/
apps/api/           Fastify + Zod
apps/web/           React + Vite + Tailwind
scripts/            generate-catalog, ingest, eval, chart, bench
tests/unit/         73 testes Â· tests/integration/ Testcontainers (pgvector)
db/schema.sql       pgvector HNSW + tsvector GIN + unaccent
```

## PendÃªncias conhecidas

- **p95 < 150 ms em 50 mil produtos no Postgres: nÃ£o medido.** O `scripts/bench.ts` estÃ¡ pronto, mas o ambiente de desenvolvimento nÃ£o tem Docker. As latÃªncias em `report/eval.md` sÃ£o do Ã­ndice em memÃ³ria e nÃ£o servem como benchmark.
- A avaliaÃ§Ã£o com embeddings reais e rerank depende das chaves de API.
- O teste de integraÃ§Ã£o com Testcontainers Ã© pulado sem Docker. Na CI (GitHub Actions) ele roda.


## Licença

### Uso Pessoal
Este projeto é licenciado sob a **Licença MIT** para uso pessoal, educacional e não comercial.

### Uso Comercial
Se você deseja usar este projeto comercialmente, deve obter uma licença comercial do autor.

**Uso comercial inclui:**
- Incorporar em um produto comercial
- Oferecer como serviço
- Usar em atividades com fins lucrativos
- Implantação empresarial

**Para obter uma licença comercial:**
Email: ma.polli@hotmail.com
