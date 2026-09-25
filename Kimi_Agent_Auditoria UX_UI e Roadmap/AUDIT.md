# AUDIT.md — FASE 0: AUDITORIA TÉCNICA DO REPOSITÓRIO

**Projeto:** WORLD NATION SIMULATOR
**Repositório:** aquasilva111-ui/JOGO-DE-SIMULA-O
**Branch auditada:** `main` @ commit `7f0d989`
**Data:** 2026-09-25
**Método:** leitura integral via GitHub API + rastreamento estático de fluxos (comando → backend → simulação → estado → frontend).

> **Limitação declarada (honestidade técnica, spec §183):** o ambiente de auditoria não teve acesso de rede direto ao GitHub para clonar e executar `pytest`/`npm run build` localmente. As classificações abaixo derivam de análise estática completa de todos os arquivos de código. A primeira ação da Fase 1 inclui CI para transformar essa verificação em automática e contínua.

---

## 1. RESUMO EXECUTIVO

O projeto **existe e tem uma espinha dorsal real**: backend FastAPI + WebSocket, motor de ticks hora/dia/mês, livro de ofertas com matching por prioridade preço-tempo, shipments que não teleportam, sistema de créditos científicos e instituições com fórmula de capacidade efetiva. Isso é uma base a preservar.

Porém, o projeto hoje **viola frontalmente 5 das 10 leis não-negociáveis (§196)** e o README anuncia funcionalidades que não existem (§183). O problema mais grave é o **motor de fallback falso no frontend** (`services/api.ts`), que reporta sucesso para comandos que nunca chegaram ao backend — exatamente o comportamento proibido por §8. O site em produção na Vercel opera 100% nesse modo, pois não existe backend deployado.

**Estado geral: PROTÓTIPO FUNCIONAL COM NÚCLEO DE MERCADO REAL, cercado de camadas visuais que simulam profundidade inexistente.**

---

## 2. CLASSIFICAÇÃO DE SISTEMAS

### BACKEND (Python / FastAPI)

| Sistema | Arquivo | Status | Observação |
|---|---|---|---|
| Servidor API + WS | `backend/server.py` | **WORKING** | REST + WebSocket + command bridge funcionais. Broadcast de estado completo a cada 500 ms (viola §157 — sem deltas). |
| Relógio de simulação | `core/time_util.py` | **WORKING** | Ticks horários, 24/dia, 360 dias/ano. Correto. |
| Event Bus | `core/buses.py` | **WORKING** | Pub/sub com histórico. Base correta para §13 e §139. |
| Motor (orquestrador) | `sim/engine.py` | **PARTIAL** | Ticks hora/dia/mês + command handler existem. Sem fases declaradas com `reads[]/writes[]/dependencies[]` (§12). Sem transações atômicas (§14). Sem snapshots/persistência (§10). |
| Mercado global | `sim/market.py` | **PARTIAL** | Livro de ofertas, preços amortecidos, dependência HHI: reais. Mas `note_flows()` **nunca é chamado** — produção/consumo não alimentam preço; `top_producers()` retorna sempre vazio. |
| Matching de trade | `sim/trade.py` | **PARTIAL** | Matching preço-tempo, sanções, tarifas, contratos 1–12 meses: reais. **Bug:** evento `TRADE` só é publicado para vendas spot (indentação); contratos não geram evento. Tarifa é paga pelo comprador ao próprio comprador (`buyer.treasury += duty_b`) — país=governo (viola §2/§77 lei 8). |
| Logística | `sim/logistics.py` | **HARDCODED** | "Distância" = `math.hypot` entre **âncoras de países** (graus lat/lon tratados como plano). Sem portos, rotas, capacidade, modais. É literalmente o anti-exemplo de §183 ("distância entre centroides"). Positivo: existe trânsito temporal e interceptação por guerra. |
| Catálogo de produtos | `sim/products.py` | **WORKING** | 45 produtos + 11 receitas, data-driven (§36/§37). Bom alicerce. |
| Produção industrial | — | **MISSING (crítico)** | `PRODUCTION_RECIPES` **nunca é executado por nenhum sistema**. Nenhuma fábrica produz. Estoques iniciais só diminuem (queima de diesel militar) ou mudam via trade. Economia = estoques estáticos + livro de ofertas. |
| Países | `sim/country_data.py` | **PARTIAL** | **16 países** — README diz 195 (viola §16/§183). Sem `regionIds[]`, sem capital estruturada, sem continente, sem geometria, sem provenance (§133). |
| Regiões (Admin-1) | `sim/regions_data.py` | **PARTIAL** | **30 regiões para 13 países**; Brasil tem 8 dos 27 estados (README diz 27 — §183). Modelo de região é bom (anchor, terreno, controller/legalOwner já separados — §19/§116). Sem geometria real (§18). |
| Instituições | `sim/institutions.py` | **PARTIAL** | Fórmula de capacidade efetiva (§39) e degradação/manutenção (§86) implementadas. Mas só **15 instituições hardcoded**; **~48 IDs referenciados em `regions_data.py` não existem** (refs quebradas — integridade de dados BROKEN). |
| Pesquisa | `sim/research.py` | **WORKING** | SC por instituição, pré-requisitos, custo duplo (SC + duração mínima), árvore de 12 techs/8 domínios. Sólido. |
| Governo/Fiscal | `engine._monthly_tick` | **HARDCODED** | PIB **estático**; imposto = PIB × alíquota (viola §70 — PIB não emerge de atividade). Corrupção desvia para elite com conservação de dinheiro (bom). Sem dívida, sem balanço de pagamentos. |
| Militar | `engine._daily_tick` | **PLACEHOLDER** | `soldiers = número` (viola §114). Único efeito: queima diesel e derruba moral. Sem unidades, células, frontes, suprimento. |
| Diplomacia | `engine.world.diplomacy` | **PLACEHOLDER** | Dict par→estado usado só para interceptar cargas. Sem comandos, tratados, sanções por comando. |
| IA | `engine._ai_market_decisions` | **PLACEHOLDER** | IA só trade aleatório (60% chance/mês). Não avalia segurança alimentar/energética/etc. (§129). Não usa command bus (§128). |
| População/Demografia | — | **MISSING** | População é um float estático por país/região. |
| Recursos naturais (deposits) | — | **MISSING** | Regiões têm lista de strings `resources[]`; não há `ResourceDeposit` (§25). |
| Energia física/grid | — | **MISSING** | País tem MW agregados; sem `EnergyAsset`, sem rede (§31–34). |
| Construção | `BUILD_INSTITUTION` | **BROKEN (viola §88)** | Construção **instantânea**: debita tesouro e a instituição já existe operando. Sem `ConstructionProject`, materiais, trabalho, prazo. |
| Save/Load | — | **MISSING** | Estado 100% em memória; restart do server = reset do mundo (§135). |
| Testes | `tests/test_simulation.py` | **WORKING** | 6 testes, incluindo teste anti-teleporte de carga. Boa semente; cobertura pequena. |

### FRONTEND (React / TS / Vite)

| Sistema | Arquivo | Status | Observação |
|---|---|---|---|
| Ponte API/WS | `services/api.ts` | **BROKEN (crítico, viola §8)** | **`fallbackState` fabricado + comandos retornam sucesso falso** (`BUILD_INSTITUTION`, `MARKET_BUY/SELL`, `START_RESEARCH`, `SET_FISCAL`, `AUDIT_CORRUPTION` respondem "sucesso" sem backend). Sem distinção visual ONLINE vs DEMO. |
| Lista de países | `services/countries195.ts` | **HARDCODED + viola §16** | O arquivo chamado `countries195` contém **30 países** — o exemplo exato proibido por §16/§183. Diverge do backend (16). |
| Mapa | `components/Map/TacticalMap.tsx` | **PARTIAL** | TopoJSON + d3-geo 2D estilo HoI. Sem globo 3D: `three` está no `package.json` **mas não é importado em nenhum arquivo** (dependência morta). |
| HUD / Navegação / Views / Modais | `components/*` | **PARTIAL** | 4 views + 6 modais estruturados. UX atual é dashboard com mapa, não "planetary command interface" (§143). Sem sistema de alertas clicáveis (§154), sem explainability UI (§142/§153). |
| Dados estáticos | `services/gameData.ts` | **HARDCODED** | `BRAZIL_STATES`, catálogos duplicados do backend (viola "fonte única de verdade"). |

### REPOSITÓRIO / PROCESSO

| Item | Status | Observação |
|---|---|---|
| `AGENTS.project.md` | **BROKEN** | Documento de **outro projeto** ("AQUA", fork Bluesky). Enganoso; remover ou substituir. |
| `README.md` | **BROKEN** | Anuncia "195 nações", "27 estados brasileiros", "logística física com rotas marítimas" — nenhum verdadeiro (§183). |
| `DEPLOYMENT.md` | **BROKEN (viola §8)** | Documenta o modo fake como *feature* ("Standalone Client Simulation"). |
| Config duplicada | **DUPLICATED** | `package.json`, `vite.config.ts`, `tsconfig.json`, `vercel.json` na raiz **e** em `frontend/`. Fonte de drift. |
| CI/CD | **MISSING** | Sem GitHub Actions; testes nunca rodam automaticamente. |
| Backend em produção | **MISSING** | Não há backend deployado; a Vercel serve só o frontend fake. |
| Build ao vivo (Vercel) | não verificado | Sandbox sem acesso externo à Vercel; verificação manual pendente. |

---

## 3. AS 10 LEIS NÃO-NEGOCIÁVEIS (§196) — STATUS

| # | Lei | Status hoje |
|---|---|---|
| 1 | Nada físico teleporta | ⚠️ Parcial — há trânsito temporal, mas sem rotas reais |
| 2 | Recursos ≠ produção | ⚠️ Recursos são strings; produção não existe |
| 3 | Produção ≠ entrega | ✅ Conceito presente (shipments), logística fake |
| 4 | Dinheiro ≠ capacidade material | ❌ Construção instantânea com dinheiro apenas |
| 5 | Capacidade instalada ≠ output real | ✅ Fórmula de gargalo existe nas instituições |
| 6 | Pesquisa ≠ tecnologia deployada | ⚠️ Bônus aplicados direto, sem deployment físico |
| 7 | Ocupação ≠ propriedade legal | ✅ `controller`/`legalOwner` já separados nas regiões |
| 8 | País ≠ economia | ❌ País tem estoque/tesouro/PIB próprios; sem empresas reais |
| 9 | Frontend não é a simulação | ❌ **Fallback no frontend simula e finge sucesso** |
| 10 | Mundo real = condição inicial | ❌ Dados sem fonte/ano/confiança (§133) |

**Placar: 2 ✅ / 4 ⚠️ / 4 ❌**

---

## 4. ACHADOS CRÍTICOS (ordem de severidade)

1. **Falso sucesso no frontend (§8).** `api.ts` executa comandos num estado local fabricado e retorna `success: true`. Prioridade máxima da Fase 1.
2. **Produção inexistente.** O coração econômico do spec (§35–42) não tem implementação alguma: receitas definidas jamais executadas. Tudo que depende de produção (preços, PIB, trade) está capado.
3. **README/DEPLOYMENT/AGENTS.documentam um jogo que não existe** (195 países, 27 estados, rotas marítimas) — §183.
4. **Cobertura mundial ausente:** 16 países backend / 30 frontend / 30 regiões / 15 instituições / ~48 referências de instituições quebradas.
5. **Sem persistência:** qualquer restart zera o mundo.
6. **Broadcast de estado completo a cada 500 ms** — inviável para 195 países/escala (§156/§157).
7. **PIB estático** impede a cadeia causal central (§3).

## 5. O QUE PRESERVAR (spec §199: "preserve useful existing work")

- Arquitetura FastAPI + WS + command handler (estender, não reescrever).
- `products.py` (catálogo + receitas) como semente do `ProductRegistry`.
- `market.py`/`trade.py` (matching, contratos, HHI) como semente do mercado real.
- `institutions.py` (fórmula de capacidade efetiva + degradação).
- `research.py` (SC + árvore + duração mínima).
- `time_util.py` + `buses.py`.
- Modelo de região com `controller`/`legalOwner` separados.
- Testes existentes como base de regressão.
- Stack visual: Tailwind + d3-geo/topojson + three (a ativar).
