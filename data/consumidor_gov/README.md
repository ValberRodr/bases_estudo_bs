# Reextração Consumidor.gov.br — 33 recursos ausentes

## Status: downloads concluídos e validados (33/33)

```text
Esperados: 33
Baixados com sucesso: 33
Ainda ausentes: 0
Inválidos/bloqueados: 0
```

## Mudança de método em relação à extração anterior

A extração original (`scripts/05_consumidor_gov_download.sh`) dependia de:
1. `dados.mj.gov.br` (catálogo CKAN oficial do MJSP) — **hoje com falha total de DNS** (confirmado nesta sessão via `curl`/`dig`, não é instabilidade passageira).
2. Fallback via Internet Archive Wayback Machine — não tinha snapshot para os 33 recursos listados no prompt (status `indisponivel_sem_fallback` no log original), por isso ficaram ausentes.

Para esta reextração, a fonte primária usada foi o **portal ao vivo consumidor.gov.br** (`https://www.consumidor.gov.br/pages/dadosabertos/externo/`, aba "Dados Abertos"), que está no ar e publica os mesmos arquivos mensais via download disparado por JavaScript (não há URL estática por trás do botão — o download foi feito clicando no botão real da interface, via navegador, conforme instruído). Cada período foi localizado pelo título exato publicado no portal (ex.: "Dados - Agosto/2026", "Dados - 2º Semestre/2018"), não apenas pelo nome do arquivo.

`dados.gov.br` (API unificada) também foi testado como alternativa e está retornando `401 Unauthorized` (passou a exigir autenticação Bearer) — não foi usável.

## Os 33 períodos solicitados e recuperados

| Período | Título no portal | Linhas (dados) | Colunas | Observação |
|---|---|---:|---:|---|
| 2018-S2 | Dados - 2º Semestre/2018 | 306.965 | 20 | datas em DD/MM/AAAA; confirmado Jul-Dez/2018 |
| 2020-01 a 2020-07 | Dados - Jan a Jul/2020 | 74.357–122.551 | 20 | — |
| 2020-09 | Dados - Set 2020 | 109.431 | 19 | título no portal sem a barra "/" |
| 2021-10 | Dados - Out/2021 | 120.708 | 19 | — |
| 2022-03, 08, 10, 11 | Dados - Mar/Ago/Out/Nov 2022 | 90.138–127.635 | 19 | — |
| 2023-09 | Dados - Set/2023 | 132.980 | 19 | — |
| 2024-07 | Dados - Jul/2024 | 55.778 | 19 | — |
| 2025-01, 05–12 | Dados - Jan, Maio–Dez/2025 | 148.779–275.053 | 19 (21 em 09/2025*) | *2025-09 teve 2 colunas extras no cabeçalho, não investigado a fundo — preservado como veio |
| 2026-01 a 2026-08 | Dados - Jan–Ago/2026 | 287.036–431.744 | 19 | — |

\* 2026-05 veio empacotado como `.zip` contendo um `.7z` interno (`finalizadas_2026-05.7z`), diferente do padrão `.zip`→`.csv` direto dos demais meses. Extraído e validado com `py7zr`; conteúdo confirmado consistente com os meses vizinhos (352.368 linhas, 19 colunas, mesmo cabeçalho). O `.zip` bruto original foi preservado como baixado.

**Total de registros novos (soma bruta, sem deduplicação/filtro):** 6.479.282
**Total de bytes baixados (arquivos comprimidos originais):** 190,9 MB

## Validação aplicada a cada arquivo

- SHA-256 e tamanho em bytes calculados sobre o arquivo bruto exatamente como baixado.
- Arquivo aberto e descompactado para checagem (zip, e no caso de 2026-05 também 7z).
- Cabeçalho lido e contagem de colunas registrada.
- Contagem de linhas de dados (excluindo cabeçalho).
- Para uma amostra (2018-S2, 2020-09, 2023-09, 2026-08) a coluna "Data Finalização" foi verificada linha a linha para confirmar que os meses presentes no arquivo batem com o período esperado — todos confirmados.
- Nenhum HTML de página de erro foi salvo como `.csv`/`.zip`.

## Erros/bloqueios encontrados

Nenhum. Os 33 recursos foram localizados e baixados com sucesso na primeira tentativa via portal ao vivo.

## Arquivos de saída desta etapa

```text
consumidor_gov_reextracao/
    raw_novos/                                  (33 arquivos .zip brutos, renomeados consumidor_gov_<periodo>.zip,
                                                  preservando o conteúdo original byte a byte)
    manifesto_33_arquivos_faltantes.csv
    README_REEXTRACAO.md                        (este arquivo)
```

## Etapa 2 — Consolidação e recorte Auto (concluída)

Script: `scripts/07_consolidar_novos_e_recorte_auto.py`.

### Achados importantes durante a consolidação

1. **7 dos 33 períodos "ausentes" já existiam, válidos, na extração anterior.**
   `finalizada2020-01.csv` a `finalizada2020-07.csv` estavam presentes em `RAW/consumidor_gov/` com conteúdo
   íntegro — a auditoria que gerou a lista de 33 provavelmente não reconheceu esse padrão de nome (sem "s" em
   "finalizada"). Para não duplicar linhas, os novos downloads desses 7 períodos **não** foram incorporados à base
   consolidada (ficam só em `raw_novos/` como referência). Resultado: **26 períodos genuinamente novos** entraram na
   consolidação, não 33.

2. **Diferença estrutural de `Situação` entre as duas fontes.** Os 62 arquivos antigos (via dados.mj.gov.br/CKAN)
   incluem linhas com `Situação` em `{Finalizada avaliada, Finalizada não avaliada, Cancelada, Encerrada}`. Os 33
   arquivos baixados agora do **portal ao vivo** trazem **somente** `Finalizada avaliada`/`Finalizada não avaliada`
   — confirmado em múltiplos arquivos novos (0 linhas `Cancelada`/`Encerrada` em nenhum dos 33). Isso não foi
   "corrigido" nem filtrado — a coluna `situacao` foi preservada no dataset consolidado exatamente como veio de cada
   fonte, para que qualquer comparação de série temporal entre períodos antigos e novos normalize esse corte
   explicitamente antes de comparar volumes.

3. **Nenhum dos 26 arquivos novos tem a coluna "Data Abertura"** (nem Gestor, Canal de Origem, Data Resposta/Análise/
   Recusa, Prazo*, Análise da Recusa) — o export atual do portal ao vivo é mais enxuto que o catálogo CKAN antigo.
   Consequência: **100% das linhas de `origem_grupo = reextracao_33` usam `fonte_data_referencia =
   data_finalizacao_fallback`** (não há escolha — não existe `data_abertura` para usar). Nos dados antigos, ~91% das
   linhas tinham `data_abertura` preenchida.

4. **Bug de duplicação corrigido:** `base_completa_2025-03.csv` e `basecompleta2025-03.zip` em `RAW/consumidor_gov/`
   são **byte-a-byte idênticos** (mesmo sha256) — o mesmo período salvo duas vezes sob nomes diferentes na extração
   anterior. O `.zip` foi excluído do processamento para não contar março/2025 em dobro (impacto: -153.208 linhas na
   base completa, -1.303 linhas no recorte Auto).

5. **Bug de parsing de data corrigido:** a primeira versão do script usava `pd.to_datetime(..., format="mixed",
   dayfirst=True)`, que inverteu silenciosamente dia/mês em datas já no formato ISO (`AAAA-MM-DD`) sempre que o dia
   era ≤12 — ex. `"2026-01-11"` (11/jan/2026) virava `ano_mes_analise = "2026-11"` em vez de `"2026-01"`. Isso
   corrompeu ~173 mil linhas (datas "no futuro", até dez/2026). Corrigido detectando o formato pela forma da própria
   string (regex `^\d{4}-\d{2}-\d{2}$` vs `^\d{2}/\d{2}/\d{4}$`) e parseando cada uma com formato explícito, sem
   inferência ambígua — a base mistura os dois formatos, às vezes nas duas colunas da **mesma linha** (`Data
   Abertura` em `DD/MM/AAAA`, `Data Finalização` em `AAAA-MM-DD`, no mesmo registro). Após a correção: 0 linhas com
   `ano_mes_analise` no futuro.

### Resultado final

```text
Total linhas base completa consolidada: 14.898.935
  - origem extração anterior (62 arquivos, ~8,98 GB):  9.051.612 linhas
  - origem reextração (26 arquivos novos, 191 MB):     5.847.323 linhas
Períodos (mes_fonte_arquivo) distintos: 88 (2014 a 2026-08, mais 2018-S1/S2 e 2019-S1/S2)
Linhas com ano_mes_analise nulo (data ilegível nas duas colunas): 9 (não imputadas)

Recorte Assunto == "Seguros de Veículos": 89.398 linhas, 229 nomes fantasia distintos
Cobertura temporal do recorte Auto: 2014-05 a 2026-08
```

Top 10 empresas por volume de reclamações no recorte Auto (todas as empresas mantidas, sem filtro):

```text
Bradesco Auto/RE                      7.195
Mapfre Seguros                        6.191
Tokio Marine                          5.650
Allianz Seguros                       5.376
Suhai Seguradora                      4.574
Azul Seguros Cia de Seguros Gerais    4.528
Youse Seguradora                      4.074
Zurich Santander Seguros              3.794
HDI Seguros                           3.734
Santander Auto                        3.472
```

### Arquivos de saída

```text
consumidor_gov_reextracao/
    raw_novos/                                  33 .zip brutos (os 26 novos + os 7 duplicados, só referência)
    manifesto_33_arquivos_faltantes.csv
    qa_schemas_por_arquivo_consolidado.csv       schema por arquivo (colunas presentes/ausentes, linhas)
    qa_cobertura_consumidor_gov.csv              linhas por período x origem x fonte_data_referencia
    consumidor_gov_completo_consolidado.parquet  base completa, todos os setores, 14.898.935 linhas
    consumidor_gov_auto_completo.parquet         recorte Seguros de Veículos, 89.398 linhas
    consumidor_gov_auto_completo.csv             mesmo recorte em CSV (utf-8-sig)
    consumidor_auto_empresa_mes.csv              reclamações por empresa x ano_mes_analise (5.383 linhas)
    README_REEXTRACAO.md                         este arquivo
```

### Regras metodológicas respeitadas

Nenhuma imputação, nenhum NULL convertido em zero, nenhuma deduplicação automática de registros de reclamação
individuais (só os 2 bugs de duplicação/parsing acima, que são duplicação de **arquivos-fonte**, não de reclamações,
foram corrigidos), nenhuma mistura de mês de abertura/finalização/arquivo-fonte (as 3 colunas `ano_mes_abertura`,
`ano_mes_finalizacao`, `ano_mes_analise` + `mes_fonte_arquivo` foram mantidas separadas), recorte Auto mantém todas
as empresas (não limitado ao Bradesco). Análise estatística/market-share não foi iniciada, conforme solicitado.
