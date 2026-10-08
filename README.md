# Bases — Estudo Seguro Auto Brasil (2016–2026)

Bases de dados **consolidadas, validadas e públicas** sobre o mercado brasileiro de seguro auto,
construídas a partir de 10 fontes oficiais (SUSEP, SENATRAN, PRF, SINESP, RENAEST, AUTOSEG, IBGE,
Google Trends, Reclame Aqui, Receita Federal), cada uma auditada e entregue como **uma base
consolidada por tema** — sem fragmentos, sem dados brutos, sem duplicidade.

Além das bases por tema, há a **SUPERBASE**: uma camada analítica que cruza todas as fontes acima
num único conjunto de painéis (UF × mês, empresa × mês, nacional × mês), pronta para análise.

## Estrutura

```
data/
  superbase/        painéis cruzados (UF×mês, empresa×mês, nacional×mês) — a síntese de tudo
  dimensoes/         dim_uf.csv, dim_empresa.csv — tabelas de apoio (código↔nome, CNPJ, cadastro)
  susep/              prêmio/sinistro por seguradora × UF × mês
  consumidor_gov/     reclamações do segmento "Seguros de Veículos", nível de registro, com UF,
                       mês e seguradora (dado aberto, já anonimizado) — 2014-05 a 2026-08
  senatran/           frota por UF × mês × tipo de veículo
  prf/                acidentes de trânsito por UF × mês
  sinesp/             roubo/furto de veículos por UF × mês (até 2022-12)
  renaest/             sinistros estaduais por UF × mês (cobertura desigual entre UFs)
  autoseg/             exposição/prêmio/sinistro CASCO por região × semestre (até 2020-S2)
  emplacamentos/       emplacamentos de veículos novos por mês/categoria/região (FENABRAVE)
  ibge/                população, renda, desocupação e IPCA por UF (formato longo consolidado)
  google_trends/       interesse de busca por termos de seguro auto, nacional × mês
  reclame_aqui/        histórico mensal de reputação por seguradora (últimos 6 meses)
docs/
  relatorio_qa_principal.md         QA do pipeline SUSEP/Consumidor.gov/IBGE/Google Trends/Reclame Aqui/CNPJ
  relatorio_qa_dados_externos.md    QA do pipeline SENATRAN/AUTOSEG/SINESP/RENAEST/PRF/FENABRAVE
  fontes_principal.csv              proveniência (URL, timestamp, SHA-256) de cada arquivo-fonte
  fontes_dados_externos.csv         idem, para as fontes do segundo pipeline
```

## Por que cada base aparece só uma vez

Cada pasta em `data/` contém a versão **mais consolidada** daquele tema — nunca um corte parcial
que já esteja embutido em outro arquivo. Exemplos de decisões aplicadas na curadoria:

- **Consumidor.gov**: só o recorte final "Seguros de Veículos", já consolidado entre a extração
  original e a reextração de 26 períodos que estavam ausentes (ver `data/consumidor_gov/README.md`)
  — não os 88+ arquivos mensais brutos, nem a concatenação de todos os setores do portal.
- **PRF**: só o agregado por UF×mês — não as ocorrências individuais (que são o insumo bruto do
  agregado).
- **AUTOSEG**: `autoseg_semestre_geografia.csv` já é a fusão de prêmio+exposição+sinistro+
  indenização por região×semestre — os dois arquivos-fonte que o compõem não são incluídos
  separadamente.
- **IBGE**: `socioeconomico_uf_periodo.csv` já consolida população, renda, desocupação e IPCA em
  formato longo — os 4 indicadores individuais que o alimentam não são incluídos separadamente.
- **Google Trends**: a série temporal (`google_trends/trends_tempo.csv`) e o corte regional
  (`superbase/dim_uf_trends_regional.csv`) são cortes diferentes e complementares, não fragmentos
  um do outro — ambos incluídos, cada um uma vez.
- **Reclame Aqui**: as métricas por janela (6m/12m/2024/2025/3 anos) só existem em
  `superbase/dim_empresa_reclame_aqui_snapshot.csv`; o histórico mensal, que é informação
  adicional não presente ali, está em `reclame_aqui/ra_historico_mensal.csv`.

## Fontes originais

SUSEP (prêmios/sinistros), SENATRAN (frota), PRF (acidentes), SINESP (roubo/furto, até 2022-12),
RENAEST (sinistros estaduais), AUTOSEG (exposição/sinistros CASCO, até 2020-S2), FENABRAVE
(emplacamentos), IBGE (população, renda, desocupação, IPCA), Google Trends, Consumidor.gov.br
(reclamações — dado aberto já anonimizado, sem nome/CPF, apenas sexo/faixa etária/cidade),
Reclame Aqui (métricas agregadas públicas) e Receita Federal/SUSEP (CNPJ e situação cadastral das
seguradoras). **Nenhum dado interno de companhia** (faturamento, NPS, sinistralidade interna) está
presente — essas informações não existem em nenhuma fonte pública.

## Validação aplicada

Detalhe completo em `docs/relatorio_qa_principal.md` e `docs/relatorio_qa_dados_externos.md`;
proveniência (URL, timestamp, hash SHA-256) de cada arquivo-fonte em `docs/fontes_principal.csv` e
`docs/fontes_dados_externos.csv`. Resumo:

- Toda extração foi auditada com checagem de sanidade (min/max/mediana) — não é coleta bruta.
- Bugs reais encontrados **nos dados oficiais** e corrigidos antes de consolidar — ex.: INMET/
  SENATRAN usando valores-sentinela como dado válido, SUSEP com ~7,5 milhões de registros
  vazios/lixo, formato de data mudando 3 vezes na PRF. Lista completa nos relatórios de QA.
- Nenhum mês/ano foi inventado; ausências permanecem ausentes (não preenchidas com zero nem
  interpoladas).
- Nenhuma associação de empresa (ex.: nome fantasia → seguradora) foi tratada como definitiva por
  fuzzy matching isolado — todas ficam marcadas em `mapping_status`, com confiança registrada.
- Todo fallback de fonte primária indisponível foi documentado explicitamente (sem substituição
  silenciosa).

## Limitações conhecidas

- **Consumidor.gov está desatualizado dentro da SUPERBASE**: as colunas `consumidor_*` de
  `superbase_uf_mes.csv`/`superbase_empresa_mes.csv` foram calculadas a partir da extração
  original (cobertura até 2025-04). O arquivo `data/consumidor_gov/consumidor_gov_auto_completo.csv`
  é mais completo (2014-05 a 2026-08, 26 períodos recuperados numa reextração posterior) e usa
  nome de marca em vez do `empresa_id`/CNPJ da SUSEP — os dois **não foram reconciliados entre si**.
  Para números de reclamação por seguradora e período, prefira recalcular a partir do arquivo
  completo em vez de usar as colunas `consumidor_*` da SUPERBASE.
- Consumidor.gov → seguradora é candidato por nome de marca (não confirmado por CNPJ para ~90 das
  211 marcas) — tratar `consumidor_n_reclamacoes` como proxy, não contagem exata.
- SINESP para em 2022-12; AUTOSEG para em 2020-S2; Emplacamentos (FENABRAVE) não tem recorte por UF.
- `uf_aproximada` do AUTOSEG é derivada de texto oficial da SUSEP, não é uma partição exata de UF.
- `susep_sinistralidade` pode ter outliers extremos em células UF×mês de baixo volume (denominador
  pequeno) — é real, não erro de processamento; pondere por volume em agregações.
- 2026 está incompleto (dado só até meados do ano) — não tratar como ano fechado.

## Licença

Dados de origem são públicos, publicados por órgãos oficiais brasileiros (SUSEP, SENATRAN, PRF,
IBGE, etc.) e por plataformas que disponibilizam dados agregados abertamente (Consumidor.gov.br,
Google Trends, Reclame Aqui). A curadoria, consolidação e documentação deste repositório são
disponibilizadas sob licença MIT (ver `LICENSE`).
