# SUPERBASE — Seguro Auto Brasil 2016-2026

Camada analítica construída a partir das bases granulares de `PROCESSED/` e `dados_externos/*/processed/`, que **permanecem intactas** — esta pasta só lê, nunca modifica ou apaga as fontes originais. Scripts de construção: `scripts/23_superbase_uf_mes.py`, `24_superbase_features.py`, `25_superbase_nacional_mes.py`, `26_superbase_empresa_mes.py`.

Não existe um único arquivo monolítico: os dados têm granularidades e cadências incompatíveis (empresa vs UF vs nacional; mensal vs trimestral vs semestral vs "período único de coleta"), então forçar tudo numa única linha esconderia isso. Em vez disso, há **5 tabelas**, cada uma numa granularidade honesta:

## 1. `superbase_uf_mes.csv` — painel UF × mês (3.429 linhas = 27 UF × 127 meses, 2016-01 a 2026-07)

Grade completa (todas as UF, todos os meses do período) — células sem dado real ficam **vazias, nunca zero**. Junta:

- **SUSEP auto** (agregado somando as seguradoras): `susep_premio_direto`, `susep_premio_retido`, `susep_sinistro_direto`, `susep_premio_retido_liquido`, `susep_n_seguradoras_ativas`
- **SENATRAN frota**: `frota_total` e por tipo-chave (`frota_automovel`, `frota_motocicleta`, `frota_caminhao`, `frota_onibus`, `frota_utilitario`)
- **Consumidor.gov** (candidatos por marca, não confirmados por CNPJ): `consumidor_n_reclamacoes`, `consumidor_pct_respondida`, `consumidor_pct_resolvida`
- **SINESP**: `sinesp_roubo_veiculos`, `sinesp_furto_veiculos`, `sinesp_roubo_furto_veiculos` (só até 2022-12)
- **RENAEST**: `renaest_*` (cobertura estadual desigual)
- **PRF**: `prf_*` (só até 2025-12)
- **AUTOSEG** (cobertura CASCO, via `uf_aproximada`): `autoseg_*_semestre` — **valor semestral repetido nos 6 meses daquele semestre**, com `autoseg_semestre_ref` mostrando de qual semestre veio. Só 2016-S1 a 2020-S2.
- **IBGE**: `populacao_ano`/`renda_real_per_capita_ano` (anual, repetido nos 12 meses, com `ibge_ano_ref`); `taxa_desocupacao_trimestre` (trimestral, repetido nos 3 meses, com `ibge_trimestre_ref`); `ipca_numero_indice_nacional` (mensal, nacional, mesmo valor em todas as UF do mês)
- **Google Trends nacional**: `trends_nacional_*` (mensal, nacional, mesmo valor em todas as UF do mês — **não é dado regional**, é a série nacional repetida por conveniência)

### Colunas derivadas (engenharia de atributos)

| Coluna | Fórmula | Nota |
|---|---|---|
| `susep_sinistralidade` | sinistro direto / prêmio direto | mesmo mês |
| `autoseg_sinistralidade_casco` | indenizações / prêmios (CASCO) | mesmo semestre |
| `autoseg_frequencia_sinistro_casco` | sinistros / exposição (CASCO) | mesmo semestre |
| `susep_premio_direto_real_201601`, `susep_sinistro_direto_real_201601` | valor nominal × (IPCA jan/2016 ÷ IPCA do mês) | deflacionado a preços de jan/2016 |
| `renda_real_per_capita_ano_deflacionada_201601` | idem | mesma base |
| `crescimento_frota_aa` | (frota_total − frota_total 12 meses atrás) / frota_total 12 meses atrás | ano contra ano |
| `sinesp_roubos_furtos_por_100mil_veiculos`, `prf_acidentes_por_100mil_veiculos`, `renaest_sinistros_por_100mil_veiculos` | contagem × 100.000 / frota_total | mesmo mês |
| `autoseg_exposicao_por_1000_veiculos` | exposição CASCO (semestre) × 1000 / frota_automovel (mês) | denominador do mês, não média do semestre |
| `consumidor_reclamacoes_por_1000_veiculos` | reclamações × 1000 / frota_automovel | proxy, não é reclamação por apólice real |
| `autoseg_delta_exposicao_segurada_casco` | exposição(semestre) − exposição(semestre anterior imediato) | **NÃO é "novas apólices"** — variação líquida (pode incluir novas contratações, cancelamentos, renovações, substituição de veículo) |

**Regra seguida em todo indicador**: denominador ausente ou zero → resultado `NULL`, nunca 0 ou infinito.

## 2. `superbase_nacional_mes.csv` — série nacional × mês (128 linhas, 2016-01 a 2026-08)

Fontes que só existem em nível nacional (sem UF): `emplacamentos_novos_*` (Fenabrave, por categoria de veículo — **nunca chamado de "novos_seguros"**), `ipca_numero_indice_nacional`, `trends_nacional_*`.

## 3. `superbase_empresa_mes.csv` — painel empresa × mês (11.306 linhas, 117 seguradoras)

Agregado **nacionalmente** (soma entre todas as UF) por seguradora: `susep_premio_direto`, `susep_sinistro_direto`, `susep_sinistralidade`, `susep_n_uf_atuacao`, `consumidor_n_reclamacoes`, `consumidor_reclamacoes_por_milhao_premio`. Atributos cadastrais de `dim_empresa.csv` (CNPJ, situação cadastral, `mapping_status`, `grupo_cnpj_id`) anexados como colunas repetidas — são atributos da empresa, não uma série temporal. **Não é uma grade completa**: só existem linhas para (empresa, mês) onde havia dado real em pelo menos uma das duas fontes.

## 4. `dim_empresa_reclame_aqui_snapshot.csv` — cópia de referência

Métricas do Reclame Aqui (janelas móveis: 6 meses, 12 meses, 2025, 2024, últimos 3 anos), coletadas num único instante (2026-09-22T03:22:44Z). **Não é uma série mensal** — deliberadamente fora do painel `superbase_empresa_mes.csv` para não fabricar variação temporal que a fonte não tem. Cada linha já carrega `periodo_inicio`/`periodo_fim`/`data_coleta` (ver `PROCESSED/ra_metricas_periodo.csv` para a versão original).

## 5. `dim_uf_trends_regional.csv` — cópia de referência

Índice percentual do Google Trends por UF, para um único período de consulta agregado (2016-01-01 a 2026-08-31) — **não varia por mês**. Fora do painel mensal pelo mesmo motivo do item 4.

---

## Limitações herdadas das bases granulares (ver `QA/relatorio_qa.md` e `dados_externos/99_qa/relatorio_qa.md` para detalhe completo)

- `uf_aproximada` do AUTOSEG é derivada de texto oficial da SUSEP, não uma partição exata de UF.
- Consumidor.gov → empresa é candidato por nome de marca, não confirmado por CNPJ (119 de 235 nomes fantasia continuam sem mapeamento — ver `QA/empresas_nao_mapeadas.csv`).
- SINESP para em 2022-12; AUTOSEG para em 2020-S2; Emplacamentos-Fenabrave não tem UF.
- Consumidor.gov tem uma lacuna real (não reprocessável) em 2020-01 a 2020-05: os arquivos-fonte desses meses não têm a coluna "Segmento de Mercado", então é impossível isolar reclamações de seguros neles por este método.
- `sinistralidade` pode ter outliers extremos em células UF×mês de baixo volume (denominador pequeno) — real, não erro de processamento; recomenda-se ponderar por volume em qualquer agregação nacional.

## Reprocessamento de 2026-10-08 (Consumidor.gov)

`consumidor_empresa_uf_mes.csv` foi reprocessado para incorporar 26 períodos recuperados numa
reextração (ver `consumidor_gov_reextracao/README_REEXTRACAO.md`) que antes ficavam ausentes por
falha de DNS na fonte original e falta de snapshot no Wayback Machine. Esses 26 arquivos não têm
"Data Abertura" (só "Data Finalização") — `scripts/06_consumidor_gov_process.py` foi ajustado para
calcular `ano_mes` a partir de "Data Finalização" quando "Data Abertura" não existe (coluna
`fonte_ano_mes` registra qual data foi usada, linha a linha, sem substituição silenciosa). Efeito:
cobertura nacional sem reclamações caiu de 38 para 5 meses (2020-01 a 2020-05, lacuna real da
fonte, não deste reprocessamento); total de reclamações mapeadas subiu de 182.488 para 310.059
linhas. Nenhuma outra coluna da SUPERBASE foi recalculada ou alterada neste reprocessamento
(confirmado bit-a-bit contra a versão anterior).
