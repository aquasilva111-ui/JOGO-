# WORLD NATION — MASTER PLAN
### Plano Mestre de Arquitetura, Simulação e Experiência

**Base:** AUDIT.md (Fase 0) · Spec Master §1–199 · Diretriz de UX/UI do produto (2026-09)
**Repositório:** aquasilva111-ui/JOGO-DE-SIMULA-O · **Branch:** main

---

## 0. DIRETRIZ DE PRODUTO — UX/UI (PILAR OFICIAL)

> **Diretriz do dono do produto:** a experiência é um **mix de Dummynation + Globo 3D**, com **animações no estilo First Strike** e **complexidade de conquista e desenvolvimento no nível Hearts of Iron**.

Tradução dessa diretriz em regras de engenharia (compondo com §143–158 do spec, que permanecem válidos):

### 0.1 Os três pilares de experiência

**PILAR A — ACESSIBILIDADE DUMMYNATION + GLOBO.**
- O **globo 3D é o palco principal** (eleva §150 de "later" para track contínuo). A biblioteca `three` já é dependência do frontend — hoje morta; será ativada.
- Interação direta: clicar num país/região/cidade no globo abre comando contextual — nunca navegar por menus para agir sobre o mundo (§146/§147).
- O mapa 2D TopoJSON atual **não é descartado**: vira camada de fallback de performance e modo de precisão cartográfica.

**PILAR B — LINGUAGEM DE ANIMAÇÃO FIRST STRIKE.**
- Sensação de "comando planetário": câmera fluida que voa entre continentes, zoom contínuo globo → país → região → cidade (LOD §149).
- **Arcos animados** para todo fluxo físico real: shipments marítimos/aéreos (já existem no backend), futuras rotas militares. Arco = dado real, nunca decoração (§62).
- **Pulsos radiais** para eventos do EventBus: construção concluída, tecnologia desbloqueada, carga entregue, crise. Alerta clicado → câmera voa até o local (§154).
- Microinterações: hover com brilho sutil, seleção com anel pulsante, transições suaves de cor de território quando controle muda.
- Paleta mantida (§143): navy/carvão profundo, painéis de vidro, ciano = interação, ouro = econômico/estratégico, verde = positivo, vermelho = guerra.

**PILAR C — PROFUNDIDADE HEARTS OF IRON.**
- Conquista: células táticas, frontlines, suprimento, cerco, ocupação vs. propriedade legal (§113–127) — com UX de ordens de alto nível (DEFENDER FRENTE, AVANÇAR, CERCAR) e micro opcional.
- Desenvolvimento: cadeias produtivas físicas, construção temporal com materiais e trabalho, gargalos explicáveis (§142) — a satisfação de HOI de "construir uma nação", mas com causalidade real em vez de bônus arbitrários.

### 0.2 Regra de ouro (inalterada)
**THE MAP IS THE GAME. THE PANELS EXPLAIN THE MAP. THE SIMULATION GIVES THE MAP LIFE.**
85–100% de presença de mapa. Painéis sempre contextuais e dispensáveis.

### 0.3 O que a diretriz NÃO muda
A prioridade do spec (§197): **não pular para guerra ou polish visual antes da fundação ser confiável.** O globo entra cedo (Fase 3) porque é a *interface* — mas animações e guerra só depois que os dados por trás forem reais.

---

## 1. ARQUITETURA ATUAL (verdade, não README)

```
Browser (Vercel static)
  └─ React SPA ── REST /api/state, /api/command ──► ???
  └─ WebSocket /ws ──► ???                        (sem backend deployado:
  └─ services/api.ts FALLBACK ENGINE ◄── fabrica   o site roda 100% fake)
       estado local e finge sucesso de comandos

Backend (só existe local):
  server.py ── SimulationEngine (singleton global)
    ├─ SimulationWorld: 16 países, 30 regiões, 15 instituições
    ├─ WorldMarket: order book + preços amortecidos + HHI
    ├─ trade.py: matching preço-tempo, contratos, sanções
    ├─ logistics.py: trânsito por distância euclidiana de âncoras
    ├─ research.py: SC + árvore de 12 techs
    └─ loop 500ms: step_ticks(speed) + broadcast FULL STATE
```

---

## 2. GAP ANALYSIS (síntese do AUDIT.md)

| Gap | Impacto | Fase que resolve |
|---|---|---|
| Fallback fake no frontend | Destrói confiança do jogador; viola §8 | **1** |
| Sem backend em produção | Site é uma maquete | **1** |
| Produção inexistente (receitas nunca executadas) | Economia é estática | **5** |
| 16 países / 30 regiões / geometria zero | Não é um simulador mundial | **2–3** |
| Construção instantânea | Viola §88 | **9** |
| Sem persistência | Mundo zera a cada restart | **1 (mínimo) / 17 (completo)** |
| Broadcast full-state 500ms | Não escala (§156/157) | **1** |
| PIB estático | Cadeia causal §3 quebrada | **12** |
| Sem globo/First Strike UX | Diretriz de produto não atendida | **3, 8, 15, 18** |
| Sem guerra real | Pilar HOI ausente | **15** |

---

## 3. ARQUITETURA ALVO

```
Browser
  └─ React + TypeScript + Vite + Tailwind
     ├─ GlobeRenderer (three/R3F)  ◄── palco principal
     ├─ MapRenderer2D (d3/topojson) ◄── fallback/precisão
     ├─ WorldEventsLayer (arcos, pulsos, câmera) ◄── linguagem First Strike
     ├─ ContextualPanels (inspetores) ◄── explicam o mapa
     └─ GameClient: NUNCA simula; exibe estado + envia Commands
            │  HTTPS REST (snapshot/comandos) + WSS (deltas/eventos)
            ▼
FastAPI (servidor persistente)
  └─ SimulationCore (autoritativo)
     ├─ Clock/Scheduler (ticks hora→dia→mês→ano) + fases §12
     ├─ CommandBus → validação → mutação transacional (§13/§14)
     ├─ EventBus → deltas → WS (§157)
     ├─ Sistemas: geografia · recursos · energia · produção ·
     │   logística · mercado · atores · governo · pesquisa ·
     │   diplomacia · guerra · IA (todos com reads/writes/deps)
     ├─ Invariants + AuditLedger (§139/§140) + Explainability (§141)
     └─ Persistence: snapshots + saves versionados (§135/§136)
            ▼
PostgreSQL (saves, audit) · Reality DB (condição inicial §134, read-only)
```

**Separação permanente (§134):** `REALITY DATABASE` (geografia, demografia, economia, recursos iniciais — com fonte/ano/confiança §133) ≠ `SIMULATION SAVE` (tudo após o início do jogo).

---

## 4. GRAFO DE DEPENDÊNCIAS (ordem real de construção)

```
F1 Fundação (backend autoritativo, sem fake, deltas, CI, deploy)
   └─► F2 Geografia mundial (195 países + importador Admin-1 + geometria)
        └─► F3 Mapa genérico + GLOBO 3D + map modes  ── UX Pilar A
             └─► F4 Mundo material (deposits, potencial energético, terra/água)
                  └─► F5 Produção (facilities executam receitas; gargalos)
                       └─► F6 Logística (grafo multimodal, portos, rotas reais)
                            └─► F7 Mercado real (supply/demand físicos → preço)
                                 └─► F8 Mercado no globo (arcos First Strike, dependências) ── UX Pilar B
                                      └─► F9 Construção temporal (projeto→materiais→obra)
                                           └─► F10 Instituições no mapa (LOD)
                                                └─► F11 População & trabalho
                                                     └─► F12 Dinheiro & contabilidade (PIB emerge)
                                                          └─► F13 Pesquisa→capacidade→deployment
                                                               └─► F14 Diplomacia (tratados, sanções)
                                                                    └─► F15 Guerra (células, frontes, suprimento, paz) ── UX Pilar C
                                                                         └─► F16 IA (mesmos Commands do jogador)
                                                                              └─► F17 Save/perf/determinismo
                                                                                   └─► F18 Polish UX (onboarding, mobile, áudio, acessibilidade)
```

Regra: nenhuma fase começa sem a anterior estar **testada e reportada** (§182).

---

## 5. FASES DE IMPLEMENTAÇÃO (detalhe)

### FASE 1 — FUNDAÇÃO DE PRODUÇÃO *(próxima, começa agora)*
1. **Eliminar falso sucesso (§8):** remover o fallback engine de `api.ts`. Backend indisponível → comandos retornam `BACKEND_UNAVAILABLE`; a UI entra em **MODO DEMONSTRAÇÃO explícito** (badge permanente, comandos de escrita desabilitados, dados marcados como estáticos) ou falha visível — nunca sucesso falso. Badge `ONLINE · SIMULAÇÃO AUTORITATIVA` quando conectado.
2. **Deltas de estado (§157):** WS passa a enviar `INITIAL_SNAPSHOT` + eventos delta (`ENTITY_UPDATED`, `PRICE_CHANGED`, `SHIPMENT_PROGRESS`...), não full-state a cada 500 ms.
3. **Correções de integridade do backend:** bug do evento `TRADE` em contratos (trade.py); reconciliar refs de instituições em `regions_data.py` (criar as ~48 ou remover as refs); tarifa creditada a um governo, não "ao comprador".
4. **Persistência mínima:** snapshot save/load em disco (JSON versionado, `saveSchemaVersion`) para sobreviver restart.
5. **Higiene do repositório:** remover `AGENTS.project.md` (projeto errado); reescrever README com verdade técnica; consolidar configs raiz vs `frontend/`; renomear `countries195.ts` → `countries.ts` até ter 195 de verdade.
6. **CI:** GitHub Actions — `pytest` (backend) + `npm run build` + `oxlint` (frontend) em todo push.
7. **Deploy:** backend em servidor persistente (Render/Railway/Fly/VPS — decisão do dono) com `VITE_API_BASE_URL`/`VITE_WS_BASE_URL` na Vercel. DEPLOYMENT.md reescrito sem celebrar modo fake.

**Teste de aceite da fase:** derrubar o backend → UI mostra MODO DEMONSTRAÇÃO e nenhum comando finge sucesso; subir o backend → badge ONLINE, comando `MARKET_BUY` gera shipment real observável via WS delta.

### FASE 2 — GEOGRAFIA MUNDIAL
- Roster real de ~195 países (ISO3 + código numérico + capital + continente + `regionIds[]`) com provenance (§133).
- Importador Admin-1 (fonte com licença redistribuível, ex.: geoBoundaries/Natural Earth — documentar dataset/licença/versão/ano §18).
- Microestados: `COUNTRY_AS_SINGLE_REGION`. Geometria estática separada de estado político (§19).
- Validação automática de cobertura (teste que falha se países < 190).

### FASE 3 — MAPA GENÉRICO + GLOBO *(Pilar A)*
- Remover dependências Brasil-específicas; seleção genérica país→região→cidade→instituição.
- **Globo 3D (three/R3F):** esfera com fronteiras Admin-0/Admin-1, picking por raycast, câmera fluida, transição 2D↔3D, LOD por zoom (§149).
- `MapModeDefinition` genérico (§148): POLITICAL, ADMIN, POPULATION, GDP... como shaders/cores sobre a mesma geometria.

### FASE 4 — MUNDO MATERIAL
`WorldMaterialBank`: `ResourceDeposit` reais (§25) por região com fonte/ano/confiança; potenciais renováveis (§28); terra agrícola e água como fundações. Lei §29: petróleo existe como depósitos físicos, não bônus.

### FASE 5 — PRODUÇÃO
Facilities executam `PRODUCTION_RECIPES` por tick; gargalo = min(condição, trabalho, energia, água, insumos, logística) (§39); inventários físicos em storage (§41/§42); `market.note_flows` finalmente alimentado → preços passam a refletir produção real.

### FASE 6 — LOGÍSTICA
Grafo multimodal (nós: cidades/portos/fábricas/minas; arestas: rodovia/ferrovia/rota marítima/pipeline) (§44/§45); `Shipment` percorre rota nó a nó (§46); portos com capacidade/congestão (§49). Aposenta `math.hypot` de âncoras.

### FASE 7 — MERCADO REAL
Preço = oferta física (produção+estoques+importações) vs demanda real (§51/§54); `DeliveredPrice` com breakdown exibido (§53); contratos spot/longos completos (§58); dependency/vulnerability (§60).

### FASE 8 — MERCADO NO GLOBO *(Pilar B)*
Selecionar PETRÓLEO muda o globo: produtores/exportadores/importadores, instalações, portos, **arcos animados de shipments reais** com espessura=volume, tooltip completo (§61/§62). "WHY THIS PRICE?" (§153). Alertas clicáveis que voam a câmera (§154).

### FASE 9 — CONSTRUÇÃO
`ConstructionProject` com estados (§90), materiais entregues fisicamente, trabalhadores, ETA, validação territorial (§91); marcador no globo com progresso (§92). Aposenta construção instantânea.

### FASE 10 — INSTITUIÇÕES NO MAPA
Universidades/hospitais/empresas como entidades espaciais com LOD (§96/§97); reconciliação total das refs de `regions_data.py`.

### FASE 11 — POPULAÇÃO & TRABALHO
Coortes (§73), demografia (§74), ocupações/skills (§75), mercado de trabalho (§77), escassez de skill como gargalo real (§78), educação/saúde como pipelines (§79/§80).

### FASE 12 — DINHEIRO & CONTABILIDADE
Ledger universal (§66), PIB por valor adicionado emergente (§70/§71), balanço de pagamentos (§72), bancos depois (§67). Teste de reconciliação mensal (§159).

### FASE 13 — PESQUISA → DEPLOYMENT
Manter `research.py`; adicionar TRL (§103), difusão de conhecimento (§104), e separar descoberta de capacidade deployada (§105): tecnologia desbloqueia *receitas/projetos*, não bônus mágicos.

### FASE 14 — DIPLOMACIA
Relações, tratados, sanções/embargos reais (afetam rotas e matching), acordos comerciais, ajuda (§111); blocos depois (§112).

### FASE 15 — GUERRA *(Pilar C — complexidade HOI)*
Células táticas ativadas em teatros de guerra (§115/§117), unidades físicas (§114), frontlines emergentes (§118), ordens de alto nível + micro (§119), combate multifator (§120), suprimento pela rede logística (§121), cerco (§122), ocupação com custo/resistência (§124), capitulação ≠ anexação (§125), paz que muda `legalOwner` (§126/§127). **UX First Strike:** arcos de operações, pulsos de batalha, território que muda de cor suavemente no globo.

### FASE 16 — IA
IA nacional/empresarial/militar avaliando necessidades reais (§129–132) e agindo **pelos mesmos Commands do jogador** (§128).

### FASE 17 — SAVE / PERFORMANCE
Save completo versionado (§135/§136), determinismo (random streams com seed), reconciliation tests (§159–161), LOD/agregação adaptativa (§158).

### FASE 18 — POLISH UX
Onboarding interativo no globo, tooltips explicativos (§141/§142 em todo lugar), mobile (bottom sheet §155), áudio, acessibilidade, hierarquia visual final.

---

## 6. PLANO DE MIGRAÇÃO DE DADOS

1. **Reality DB (novo, read-only):** `data/reality/` — países, regiões Admin-1, depósitos, potenciais, demografia inicial. Cada registro: `source`, `referenceYear`, `retrievalDate`, `unit`, `confidence` (§26/§133).
2. **Migração dos dados atuais:** os 16 países e 30 regiões hardcoded viram *seed* da Reality DB marcados `confidence: ESTIMATED` até substituição por fontes reais; nada é deletado antes do substituto existir.
3. **Simulation Save:** `saveSchemaVersion: 1` desde a Fase 1; toda mudança de schema incrementa e traz função de migração.
4. **Renomeações honestas:** `countries195.ts` → `countries.ts`; README reescrito por fase.

## 7. ESTRATÉGIA DE TESTES

- **CI obrigatório (Fase 1):** pytest + build + lint em todo push; PR quebrado não mergeia.
- **Regressão:** manter os 6 testes atuais; todo bug encontrado vira teste (começando pelos 3 da Fase 1).
- **Cadeia de integração (§186):** Brasil importa petróleo saudita → navio viaja → refino → diesel → preço reage. Será o "teste sagrado" das Fases 5–7.
- **Testes de gargalo (§187–189):** lítio, grid, engenheiros.
- **Testes de guerra (§190–192):** captura de região produtora, corte de ferrovia, controller vs legalOwner.
- **Teste de UX (§193):** fluxo completo de jogador novo sem documentação — checklist manual por release.
- **Conservação (§140):** invariantes de inventário/dinheiro/população rodando como validação de fase (§12, passo 19).

## 8. ESTRATÉGIA DE PERFORMANCE

- **Rede:** snapshot + deltas (§157); nunca full-state por tick.
- **Simulação:** camadas de resolução (§158) — agregado em paz, tático em guerra; empresas pequenas agregadas, estratégicas individuais.
- **Render:** LOD de globo (§149), viewport culling, geometria simplificada por zoom, clustering de instituições; animações em GPU (shaders/CSS transforms), nunca re-render React por tick — estado em store com refs mutáveis para a camada WebGL.
- **Dados:** lazy loading de geometria Admin-1 por região visível; cache de projeções.

---

## 9. PRÓXIMO PASSO IMEDIATO

**Fase 1 — Fundação de Produção** (escopo em §5 acima). Pendente de uma decisão do dono do produto: **plataforma de hospedagem do backend** (Render / Railway / Fly.io / VPS Hostinger) e autorização de commits em branch de trabalho. Sem isso, o site continuará sendo uma maquete bonita com um cérebro falso — exatamente o que este spec existe para eliminar.
