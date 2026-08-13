# Projeto DDI — Análise de Tickets Abertos

**Especificação de indicadores, análises estatísticas e dashboard**
Versão 1.0 · 13/08/2026

---

## Status: aguardando dados

Esta é a especificação executável da análise. Nenhum número real foi
calculado ainda, porque as duas fontes necessárias não estão acessíveis
a partir desta sessão:

| Insumo | Situação | O que destrava |
|---|---|---|
| Tickets abertos do projeto DDI | Conector Atlassian Rovo (Jira) **não instalado** | Instalar em claude.ai → Connectors e habilitar neste chat |
| Manual da marca DDI | Pasta OneDrive local **não alcançável** — esta sessão roda em contêiner na nuvem | Anexar o PDF/PPTX na conversa |

Nenhum indicador abaixo foi preenchido com valores fictícios. As fórmulas,
cortes e gráficos estão definidos e prontos para executar sobre o extrato real.

---

## 1. Modelo de dados

### Extração (JQL)

```sql
-- Backlog aberto, ordenado do mais antigo para o mais novo
project = DDI AND statusCategory != Done ORDER BY created ASC
```

Usar `statusCategory != Done` em vez de `status != Closed`: pega qualquer
status aberto sem precisar enumerar o workflow, e sobrevive a mudanças de
workflow.

### Extração complementar (baseline de fluxo)

```sql
-- Resolvidos nos últimos 180 dias, para calcular vazão e lead time de referência
project = DDI AND statusCategory = Done AND resolved >= -180d ORDER BY resolved ASC
```

Sem essa segunda extração não é possível calcular vazão, projeção de
drenagem do backlog nem comparar a idade do aberto contra o histórico.

### Campos necessários

| Campo | Uso analítico | Se faltar |
|---|---|---|
| `key` | Identificador, drill-down | Bloqueante |
| `assignee.displayName` | Corte por analista | Bloqueante — tratar nulo como "Não atribuído" |
| `created` | Base de todo o aging | Bloqueante |
| `updated` | Staleness (dias sem movimento) | Perde-se o indicador de fila parada |
| `status` / `statusCategory` | Estágio, CFD | Bloqueante |
| `priority` | Ponderação de carga, SLA | Usar peso uniforme |
| `issuetype` / `components` | Corte por categoria | Bloqueante |
| **país** | Corte geográfico | **Verificar onde vive**: custom field, component, label ou prefixo no summary |
| `resolutiondate` | Nulo por definição no aberto; usado no baseline | — |
| `duedate` / campos de SLA | % fora de SLA | Substituir por meta interna por prioridade |

> **Ponto a confirmar antes de rodar:** o campo "país" raramente é nativo
> no Jira. Preciso saber se é um custom field (`customfield_XXXXX`), um
> `component`, uma `label`, ou se está embutido no texto. Isso muda a
> extração e é a única ambiguidade estrutural real da análise.

---

## 2. Indicadores

### 2.1 Volume e distribuição

| # | Indicador | Fórmula | Por quê |
|---|---|---|---|
| V1 | Backlog aberto total | `COUNT(*)` | Número-âncora |
| V2 | Tickets por analista | `COUNT(*) GROUP BY assignee` | Pedido direto |
| V3 | Tickets não atribuídos | `COUNT(*) WHERE assignee IS NULL` | Fila órfã — costuma ser a maior fonte de idade oculta |
| V4 | Tickets por país | `COUNT(*) GROUP BY pais` | Pedido direto |
| V5 | Tickets por categoria | `COUNT(*) GROUP BY categoria` | Pedido direto |
| V6 | Tickets por prioridade | `COUNT(*) GROUP BY priority` | Contexto de risco |
| V7 | Tickets por status | `COUNT(*) GROUP BY status` | Onde a fila trava |

### 2.2 Aging — "a quantos dias"

Idade de cada ticket: `idade_dias = hoje − created`.

| # | Indicador | Fórmula | Por quê |
|---|---|---|---|
| A1 | Idade mediana | `MEDIAN(idade_dias)` | **Mediana, não média** — ver §3.1 |
| A2 | Idade P90 / P95 | `PERCENTILE(idade_dias, 0.90 / 0.95)` | A cauda é onde mora o problema |
| A3 | Ticket mais antigo | `MAX(idade_dias)` | Caso-limite para narrativa |
| A4 | Distribuição por faixa | Faixas: `0–3 · 4–7 · 8–15 · 16–30 · 31–60 · 60+` | Corte operacional acionável |
| A5 | % do backlog com +30 dias | `COUNT(idade>30) / COUNT(*)` | KPI de saúde da fila |
| A6 | Aging mediano por analista | `MEDIAN(idade) GROUP BY assignee` | Distingue "muitos tickets" de "tickets velhos" |
| A7 | Aging mediano por país | `MEDIAN(idade) GROUP BY pais` | Idem, no eixo geográfico |
| A8 | Aging mediano por categoria | `MEDIAN(idade) GROUP BY categoria` | Identifica categoria que emperra |

### 2.3 Estagnação (o indicador que quase todo backlog esquece)

Um ticket com 40 dias que foi atualizado ontem está *em andamento*. Um
ticket com 40 dias sem toque há 25 está *abandonado*. São problemas
diferentes e exigem ações diferentes.

| # | Indicador | Fórmula |
|---|---|---|
| E1 | Dias desde último update | `hoje − updated` |
| E2 | Tickets parados (>14d sem update) | `COUNT(dias_sem_update > 14)` |
| E3 | Taxa de estagnação por analista | `COUNT(parados) / COUNT(*) GROUP BY assignee` |
| E4 | **Risco crítico** | `prioridade ALTA` **E** `dias_sem_update > 7` | 

E4 é a lista que a liderança precisa ver primeiro. É a única tabela do
dashboard que existe para gerar ação imediata, não entendimento.

### 2.4 Carga e equilíbrio da fila

| # | Indicador | Fórmula | Por quê |
|---|---|---|---|
| C1 | Carga ponderada por analista | `SUM(peso_prioridade)` — ex.: Crítica 5, Alta 3, Média 2, Baixa 1 | 12 tickets baixos ≠ 12 críticos |
| C2 | Índice de Gini da distribuição | Gini sobre tickets por analista | Mede desequilíbrio em **um** número |
| C3 | Share do analista mais carregado | `MAX(count) / SUM(count)` | Detecta ponto único de falha |
| C4 | Razão carga/vazão individual | `backlog_analista / resolvidos_por_semana_analista` | Semanas de fila por pessoa — o número mais acionável de todos |

C4 é o indicador que responde de fato "quem está afogado": um analista com
30 tickets e vazão de 15/semana está melhor que um com 12 e vazão de 2.

### 2.5 Fluxo (exige a extração de resolvidos)

| # | Indicador | Fórmula | Por quê |
|---|---|---|---|
| F1 | Taxa de chegada semanal | `COUNT(created) por semana` | Demanda |
| F2 | Vazão semanal | `COUNT(resolved) por semana` | Capacidade |
| F3 | Fluxo líquido | `F1 − F2` | **Positivo = backlog crescendo.** Se F3>0 sustentado, nenhum mutirão resolve — é problema de capacidade |
| F4 | Projeção de drenagem | `backlog / max(F2 − F1, 0)` | Semanas para zerar no ritmo atual |
| F5 | Lead time P50/P85 dos resolvidos | Percentis de `resolved − created` | Benchmark honesto para julgar o aberto |

---

## 3. Rigor estatístico

### 3.1 Mediana, sempre — nunca média

Distribuições de aging de backlog são log-normais com cauda longa à
direita. Um ticket travado há 400 dias desloca a média e produz um número
que não descreve nenhum ticket real. Reportar **mediana + P90**; usar média
apenas se a assimetria for verificada como baixa (`skewness < 0.5`).

### 3.2 Viés de censura à direita — a armadilha central desta análise

Analisar **apenas tickets abertos** enviesa sistematicamente qualquer
conclusão sobre tempo de resolução. Os tickets rápidos já saíram da
amostra; o que resta é, por construção, o subconjunto lento. A idade média
do backlog aberto **não é** uma estimativa do tempo de resolução.

Tratamento correto: **análise de sobrevivência (Kaplan-Meier)**, tratando os
abertos como observações censuradas à direita. Isso responde à pergunta
certa — "qual a probabilidade de um ticket ainda estar aberto após N dias?"
— e permite comparar países e categorias sem o viés.

### 3.3 Testes de diferença entre grupos

Para afirmar que "o país X demora mais" é preciso testar, não olhar barras:

- **Kruskal-Wallis** (não-paramétrico, não assume normalidade) para
  diferença de aging entre países / categorias / analistas
- **Dunn com correção de Bonferroni** como post-hoc, para saber *quais*
  pares diferem
- Reportar **tamanho de efeito** (ε² ou diferença de medianas com IC95%),
  não só p-valor — com n grande, diferenças irrelevantes ficam significantes

### 3.4 Sanidade por Lei de Little

`WIP = vazão × lead time`. Se o backlog medido divergir muito do previsto
pela lei, há erro de extração (status fantasma, tickets duplicados, projeto
com sub-tarefas contadas duas vezes). É uma checagem de 30 segundos que
pega a maioria dos erros de dados.

### 3.5 Cortes com massa mínima

Não reportar mediana de aging para grupos com n < 5. Agrupar caudas em
"Outros" e declarar o n em todo gráfico segmentado.

---

## 4. Gráficos por análise

Seleção pela pergunta que cada visual responde:

| Análise | Gráfico | Por que este, e não o óbvio |
|---|---|---|
| Tickets por analista | **Barra horizontal ordenada** | Nomes são longos (ilegíveis em coluna vertical); ordenar por valor faz o ranking ser lido sem esforço. **Nunca pizza** — comparar ângulos entre 8+ analistas é impreciso |
| Composição de aging por analista | **Barra empilhada 100%** por faixa de idade | Mostra *qualidade* da fila, não só tamanho: revela quem tem fila pequena mas podre |
| Distribuição de idade | **Histograma** + **box plot por analista** | Histograma mostra a forma (confirma a cauda longa); box plot expõe outliers e mediana simultaneamente |
| Tickets por país | **Choropleth** se >8 países; **barra ordenada** se ≤8 | Mapa só ganha da barra quando a geografia em si informa; abaixo disso a barra é mais precisa |
| Tickets por categoria | **Pareto** (barra ordenada + linha acumulada) | Responde diretamente "quais poucas categorias são 80% do backlog" — a pergunta de priorização |
| Analista × categoria | **Heatmap** | Expõe especialização e concentração de conhecimento numa matriz densa |
| Analista × faixa de aging | **Heatmap** com escala sequencial | Localiza a célula-problema exata |
| Aging × prioridade vs. SLA | **Gráfico de bullet** por prioridade | Feito para "real vs. meta"; comunica violação sem legenda |
| Fluxo semanal | **Linha dupla** (chegadas vs. resolvidos) | A pergunta é tendência e cruzamento — território natural da linha |
| Evolução do backlog | **Cumulative Flow Diagram** (área empilhada por status) | Padrão da indústria para backlog: banda que engrossa = gargalo naquele status |
| Probabilidade de resolução | **Curva Kaplan-Meier** por país/categoria | Único visual estatisticamente correto sob censura (§3.2) |
| Risco crítico | **Tabela** ordenada por dias sem update | Ação exige identificar o ticket, não a tendência. Gráfico aqui seria decorativo |

### Regras de codificação visual

- Faixas de aging usam escala **sequencial** de um matiz só (claro→escuro =
  novo→velho). Cores categóricas para dado ordinal destroem a leitura.
- Status RAG apenas para semáforo de meta, nunca como paleta de séries.
- Verde/vermelho jamais como único portador de informação — ~8% dos homens
  têm deficiência de visão de cores. Sempre duplicar em posição, rótulo ou
  forma.
- Todo eixo de contagem começa em zero. Eixo de aging pode usar escala log
  se a cauda comprimir demais o corpo da distribuição.

---

## 5. Layout do dashboard

Hierarquia: métrica mais importante em cima à esquerda; resumo → tendência →
detalhe de cima para baixo. Máx. 5–8 visuais por página.

### Página 1 — Executivo

```
┌────────────────────────────────────────────────────────────────┐
│  Backlog: N   │ Idade mediana: N d │ P90: N d │ +30d: N%       │
│  Não atrib.: N│ Parados >14d: N    │ Fluxo líq.: ±N/sem        │
├──────────────────────────────┬─────────────────────────────────┤
│  Cumulative Flow Diagram     │  Composição por faixa de aging  │
├──────────────────────────────┼─────────────────────────────────┤
│  Pareto de categorias        │  Chegadas vs. Resolvidos        │
└──────────────────────────────┴─────────────────────────────────┘
```

### Página 2 — Por analista

Ranking (barra horizontal) · Heatmap analista × faixa de aging ·
Box plot de idade por analista · Tabela com carga ponderada, Gini e
razão carga/vazão (C4).

### Página 3 — Geográfico e categoria

Mapa/barra por país · Heatmap país × categoria · Kaplan-Meier por país ·
Aging mediano por categoria com IC95%.

### Página 4 — Ação

Somente tabelas: risco crítico (E4) · não atribuídos ordenados por idade ·
top 20 mais antigos · violações de SLA.

---

## 6. Definição de KPI (formato validável)

```yaml
kpi:
  name: "Idade Mediana do Backlog Aberto"
  owner: "Coordenação DDI"
  purpose: "Medir a saúde temporal da fila de tickets abertos"
  formula: "MEDIAN(hoje - created) WHERE statusCategory != Done"
  data_source: "jira.project = DDI"
  granularity: "diária"
  target: 7                  # a definir com a operação
  warning_threshold: 14
  critical_threshold: 30
  dimensions: ["assignee", "pais", "categoria", "prioridade"]
  caveats:
    - "Mediana, não média — distribuição log-normal (§3.1)"
    - "Não estima tempo de resolução: amostra censurada à direita (§3.2)"
    - "Grupos com n<5 não são reportados (§3.5)"
```

Metas e limiares acima são **placeholders** — precisam vir do acordo de
nível de serviço real do DDI. Um KPI com meta inventada é pior que um KPI
sem meta.

---

## 7. Como isso vira número

1. Instalar o conector **Atlassian Rovo** e habilitá-lo neste chat
2. Confirmar onde vive o campo **país** (§1)
3. Rodar as duas extrações JQL (§1)
4. Executar §2 e §3, com checagem de Little (§3.4) antes de publicar
5. Aplicar tokens do **manual da marca DDI** (anexar na conversa)
6. Publicar o dashboard de §5

Passos 1, 2 e 5 dependem de você. Os demais, não.
