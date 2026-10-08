# Relatório de QA — Plano de Coleta Seguro Auto 2016–2026

Status em 2026-09-22. Ver `QA/fontes.csv` para proveniência (URL, timestamp, SHA-256) de cada arquivo bruto, e `QA/cobertura_temporal.csv` para lacunas por fonte.

## 1. SUSEP — concluído

- `RAW/BaseCompleta.zip` (551 MB) baixado da URL oficial e preservado integralmente. Extraídos `SES_UF2.csv`, `Ses_cias.csv`, `Ses_ramos.csv`, `Ses_seguros.csv`, `ses_gruposramos.csv`.
- Filtro de automóvel: `gracodigo == '05'`, classificação oficial presente na própria base (tabela `ses_gruposramos.csv`, "05 - Automóvel"). Ramos incluídos: 0520, 0523, 0524, 0525, 0526, 0527, 0531, 0542, 0544, 0553, 0588, 0589 (inclui DPVAT, que a SUSEP classifica dentro do grupo Automóvel — ver `QA/susep_ramos_automovel_selecionados.csv`).
- Saída: `PROCESSED/susep_auto_empresa_uf_mes.csv` — 638.057 linhas, 117 empresas, cobertura mensal completa 201601–202607 (127/127 meses).

## 2. Dimensões

- `PROCESSED/dim_uf.csv`: 27 UFs, códigos IBGE, região.
- `PROCESSED/dim_empresa.csv`: 117 seguradoras do ramo Automóvel (derivadas do SUSEP). Campos de CNPJ/Receita Federal/Reclame Aqui **não preenchidos** — fase de CNPJ explicitamente adiada nesta sessão (decisão do usuário). `mapping_status=pendente` para todas.

## 3. IBGE — concluído

- População (SIDRA 6579), desocupação (4099), renda per capita (7532), IPCA (1737). RAW em JSON preservado com hash.
- **Lacuna real da fonte**: população não tem 2022/2023 na tabela 6579 (ano do Censo Demográfico 2022 e o seguinte — IBGE não publicou a estimativa de referência nesses anos).
- Renda per capita: variável correta é a categoria "Total" (código 49283) da classificação 1042; sem essa correção o endpoint retornava apenas valores ausentes (`..`) — corrigido.
- Saída consolidada em formato longo (cada indicador na sua granularidade nativa, sem cruzamento artificial UF×mês): `PROCESSED/socioeconomico_uf_periodo.csv`.

## 4. Consumidor.gov.br — concluído com lacunas documentadas

- **Fonte primária indisponível**: `dados.mj.gov.br` tem falha total de resolução DNS (não apenas instabilidade/502 como o plano antecipava) — confirmado via `curl`, `dig` e fetch em navegador real.
- **Fallback aplicado**: Internet Archive Wayback Machine, explicitamente identificado e documentado por arquivo em `QA/fontes.csv` (URL original + URL arquivada + timestamp do snapshot).
- Lista de 87 recursos obtida via API pública do portal unificado `dados.gov.br` (que espelha metadados mas não hospeda os arquivos — confirmado que não há proxy de download funcional).
- **62 de 87 arquivos** obtidos com sucesso (3,8 GB). **25 indisponíveis** (sem snapshot arquivado) — ver `QA/consumidor_gov_download_log.csv`. Concentrados em: 2025-05 a 2026-06 (recentes demais para terem sido arquivados) e lacunas pontuais em 2018/2020/2021/2022/2023/2024.
- **Mudança de taxonomia detectada e corrigida**: o rótulo do segmento de seguros mudou de "Corretoras e Sociedades de Seguros, Capitalização e Previdência" (até 2021-03) para "Seguros, Capitalização e Previdência" (a partir de 2021-04). Ambos tratados como equivalentes.
- **Delimitador inconsistente**: `basecompleta2022-04.csv` usa TAB em vez de `;` — detecção automática implementada.
- 2019 (ambos semestres): arquivo de origem não tem colunas "Ano/Mês Abertura" — 7.500 reclamações do setor de seguros ficam sem `ano_mes` (mantidas no dataset, não descartadas, não inventadas).
- Arquivos `finalizada2020-01` a `07`: não têm a coluna "Segmento de Mercado" na fonte — impossível identificar reclamações de seguros nesses meses por este método.
- **Casamento empresa**: candidato por normalização de nome + desempate documentado por segmento regulatório (entidades "Seguros Gerais" preferidas sobre "Vida/Previdência/Capitalização/Saúde" do mesmo grupo, quando a ambiguidade permite resolução única). **Não é confirmado por CNPJ** (fase adiada). 110 de 211 nomes fantasia distintos mapeados a uma seguradora do ramo Automóvel; 101 permanecem em `QA/empresas_nao_mapeadas.csv` para revisão futura com CNPJ (inclui casos de ambiguidade genuína, ex.: "Allianz Seguros" e "HDI Seguros" têm duas entidades jurídicas SUSEP com o mesmo nome de marca, indistinguíveis sem CNPJ).
- Saída: `PROCESSED/consumidor_empresa_uf_mes.csv` — 171.779 linhas, cobertura 201405–202504 (lacunas listadas em `QA/cobertura_temporal.csv`).
- Concatenado bruto completo (todos os setores, não só seguros) preservado em `PROCESSED/consumidor_gov_completo_raw_concat.parquet` (9.051.612 linhas) para eventual reuso.

## 5. Google Trends — concluído

- Fonte: Google Trends **Explore** (não a API alfa, que só cobre uma janela contínua de 5 anos), via exportação manual de CSV assistida por navegador.
- Termos fixos: "seguro auto", "seguro de carro", "cotação seguro auto", "corretora de seguros", "seguros". Local: Brasil. Período: 01/01/2016–31/08/2026 (último mês completo antes da coleta em 2026-09-22). Tipo: Pesquisa na Web. Categoria: Todas as categorias (mesma em todas as consultas).
- `trends_tempo.csv`: vem de **uma única consulta com os 5 termos comparados juntos** (Trends permite até 5 termos por consulta), garantindo que os índices 0–100 fiquem na mesma escala relativa entre os termos ao longo do tempo. 640 linhas (128 meses × 5 termos).
- `trends_regiao.csv`: vem de **5 consultas individuais** (uma por termo). Necessário porque a visão "Interesse por sub-região" muda de métrica quando vários termos são comparados juntos — passa a mostrar a % de participação relativa entre os termos naquela região, não um índice 0–100 por termo. Essa versão comparada foi preservada em `RAW/google_trends/subregiao_comparado_5termos_*.csv` mas **não** foi usada no cálculo, por misturar uma métrica diferente. 135 linhas (27 UF × 5 termos).
- Todos os 7 arquivos RAW preservados exatamente como exportados pelo Google, com hash em `QA/fontes.csv`.
- Confirmado: índice é relativo 0–100, não convertido para "quantidade de buscas" em nenhum momento.

## 6. Reclame Aqui — concluído (117 de 117 empresas processadas)

- **Método**: para cada uma das 117 seguradoras, busca via API pública `api.reclameaqui.com.br/search-service` (a partir do contexto do navegador — essa API tem proteção Cloudflare que bloqueia acesso direto via curl; **nenhuma proteção anti-bot foi contornada**, apenas usado o navegador real). Para o melhor candidato (validado por sobreposição de tokens de marca, exigindo que o token principal do nome SUSEP apareça no nome retornado — heurística conservadora, não fuzzy matching solto), a página pública da empresa foi buscada e os dados extraídos dos `props` de Astro Islands renderizados no servidor (dados estruturados públicos embutidos no HTML, não uma API JSON separada — mas preferido sobre parsing visual de DOM por ser mais robusto).
- **Resultado do casamento**: 81 de 117 empresas mapeadas com confiança (score 1.0 ou 0.5 = correspondência parcial de marca, ex.: "HDI Global Seguros" → perfil "HDI Seguros"). 33 sem correspondência confiável (`QA/ra_empresas_sem_match.csv` — muitas são seguradoras pequenas/nicho sem página própria no RA, ou apenas produto DPVAT/resseguro sem atendimento direto ao consumidor). **3 casos revisados manualmente e descartados** por serem falsos-positivos de busca (`QA/ra_empresas_descartadas_revisao_manual.csv`): "Consórcio do Seguro DPVAT"→casou com produto de consórcio da Porto por coincidência textual; "Bradesco Auto/RE" → casou com a página do banco, não da seguradora (a seguradora Bradesco foi capturada corretamente por outra consulta); "AXA XL Seguros" (linhas corporativas) → casou com "AXA Seguros" (pessoa física), marcas distintas.
- **CNPJ**: capturado diretamente da página do RA para 79 das 81 empresas mapeadas, e gravado em `dim_empresa.csv`. **Não é uma validação cruzada contra fonte independente** (a fase de CNPJ/Receita Federal foi adiada) — é um insumo para a reconciliação futura, não uma confirmação definitiva. `mapping_status="candidato_reclame_aqui"` em todos os casos.
- **Datação (importante)**: cada linha de `ra_metricas_periodo.csv` carrega seu **período real** (`periodo_inicio`/`periodo_fim`, vindos da própria página) além do timestamp da coleta (`data_coleta`) — não é um resumo do estado atual. Os 5 tipos de janela observados são: `SIX_MONTHS`, `TWELVE_MONTHS`, `LAST_YEAR` (ano civil 2025), `PAST_LAST_YEAR` (ano civil 2024) e `LAST_THREE_YEARS` (**nota**: a aba rotulada "Geral" na interface do RA corresponde, nos dados subjacentes, a uma janela móvel dos últimos 3 anos — 2023-09-22 a 2026-09-21 — e não a um acumulado desde o cadastro da empresa; isso foi verificado nos dados brutos, não assumido a partir do rótulo da interface).
- Conforme regra do plano, **isto não é apresentado como série 2016–2026**: é uma camada complementar de reputação recente, com janelas datadas mas coletadas em um único instante (2026-09-22T03:22:44Z). Uma coleta futura deve *adicionar* novas linhas com novo `data_coleta`, nunca sobrescrever as existentes, para preservar o histórico de coletas.
- `ra_historico_mensal.csv`: classificação de reputação mês a mês, últimos 6 meses antes da coleta (não uma série completa 2016–2026, também datada com `ano_mes` + `data_coleta`).
- Não foram coletados textos individuais de reclamações nem dados pessoais — apenas métricas agregadas públicas.
- RAW preservado em `RAW/reclame_aqui/ra_raw_extract_20260922T032244Z.json` com hash em `QA/fontes.csv`.

## 7. CNPJ / Receita Federal — concluído (107 de 117 empresas, 91%)

- **Mudança de abordagem em relação ao plano original**: o plano previa baixar o dump nacional "Dados Abertos CNPJ" da Receita Federal (dezenas de GB, dezenas de milhões de registros) só para extrair ~117 CNPJs. Em vez disso, usamos a **BrasilAPI** (`brasilapi.com.br/api/cnpj/v1/{cnpj}`), que republica os mesmos dados abertos oficiais da RFB mas permite consulta unitária por CNPJ via HTTP simples — sem proteção anti-bot, sem necessidade de baixar a base completa. Não existe API oficial da própria Receita Federal para consulta unitária sem captcha.
- **Descoberta de CNPJ em duas fontes, por ordem de prioridade**:
  1. **SUSEP — consulta de entidades** (`www2.susep.gov.br/safe/certidoes/gateway/pessoa/empresas/ativas`, endpoint público descoberto inspecionando a página de emissão de certidões, exatamente como o plano previa). Casamento por **código exato** (`codigoFip == codigo_susep`) — não é fuzzy matching, é correspondência de identificador. Resolveu 32 empresas com confiança máxima (`mapping_status=confirmado_codigo_susep`).
  2. **Reclame Aqui**: CNPJ capturado da própria página pública durante a fase 6, usado como ponto de partida para as 75 empresas que o RA já havia identificado com um CNPJ válido (14 dígitos). Consultado na Receita Federal via BrasilAPI para confirmar existência/situação cadastral real (`mapping_status=confirmado_cnpj_receita`).
- **Bug corrigido durante o processo**: 4 páginas do Reclame Aqui retornavam valores de CNPJ inválidos/placeholder ("00000000000", "raichu" — nome do sistema interno de assets do RA, "--") que estavam sendo aceitos sem validação. Corrigido no script 08 (agora exige exatamente 14 dígitos numéricos, rejeita sequência de zeros) e as 4 empresas foram re-resolvidas com sucesso pela via SUSEP (código exato).
- **Validação cruzada**: para as empresas com CNPJ obtido via SUSEP (fonte independente do Reclame Aqui), comparei contra o CNPJ que a página do RA exibe — sem conflitos encontrados nos casos onde ambas as fontes tinham um valor. Importante ressalvar: para a maioria das 75 empresas cujo CNPJ veio originalmente do próprio Reclame Aqui, a consulta à Receita Federal confirma que o CNPJ **existe e está ativo**, mas não é uma segunda fonte independente confirmando que a *empresa* encontrada pela busca no RA é exatamente a mesma da SUSEP — isso permanece garantido apenas pela heurística de casamento de marca (script 08), não por CNPJ duplo-checado, exceto nos 32 casos resolvidos via código SUSEP.
- **10 empresas permanecem sem CNPJ** (`QA/empresas_sem_cnpj.csv`) — não aparecem na lista de entidades *ativas* da SUSEP (provavelmente extintas/incorporadas) e não foram encontradas com confiança no Reclame Aqui.
- Dados cadastrais gravados em `dim_empresa.csv`: `cnpj`, `nome_fantasia_receita`, `situacao_cadastral`, `data_situacao_cadastral`, `data_abertura` (data de início de atividade), `cnae_principal`, `municipio`, `uf_receita`.
- RAW preservado: `RAW/receita/{cnpj}.json` (uma resposta por CNPJ) e `RAW/ses_extracted/susep_entidades_ativas.json` (lista completa de 2.585 entidades ativas da SUSEP). Proveniência completa em `QA/fontes.csv`.

## 8. Pendente (não executado nesta sessão)

- As 10 empresas sem CNPJ (`QA/empresas_sem_cnpj.csv`) — precisariam de busca manual ou consulta ao dump completo da Receita Federal (não disponível via BrasilAPI, que só cobre CNPJs ativos/conhecidos).
- Reconciliação fina dos mapeamentos candidatos do Consumidor.gov.br (101 nomes fantasia ainda sem candidato) usando os CNPJs agora disponíveis não foi refeita nesta sessão — o Consumidor.gov.br não expõe CNPJ nos dados brutos, então essa reconciliação exigiria um passo adicional de matching indireto.

## 9. Reconciliação de CNPJ entre bases (script 11)

A pedido do usuário, revisei onde o CNPJ agora disponível (fase 7) poderia ser usado para resolver pendências deixadas nas fases anteriores.

- **Descoberta principal**: 9 CNPJs são compartilhados por 20 `codigo_susep` diferentes — ou seja, `codigo_susep` **não é um identificador estável de empresa ao longo do tempo**. A mesma pessoa jurídica (mesmo CNPJ, confirmado pela Receita Federal) aparece registrada na SUSEP sob mais de um código, tipicamente por reformulação/renomeação societária. Exemplos: "Allianz Brasil Seguradora S.A" (código 01015) e "Allianz Seguros S.A." (código 05177) são o mesmo CNPJ 61.573.796/0001-66; HDI aparece sob 3 códigos diferentes com o mesmo CNPJ. Lista completa em `QA/grupos_cnpj_susep.csv`, e a coluna `grupo_cnpj_id` foi adicionada a `dim_empresa.csv`.
- **Consequência 1 — ambiguidades resolvidas**: no casamento de nomes do Consumidor.gov.br (script 06), casos antes marcados como "ambíguo" (múltiplos candidatos empatados, sem CNPJ para desempatar) agora são resolvidos quando todos os candidatos empatados compartilham o mesmo `grupo_cnpj_id` — não é mais uma escolha arbitrária, é a mesma empresa real. Resolveu 5 casos: Allianz Seguros, HDI Seguros, Chubb Seguros, Itaú Unibanco Capitalização, Itaú Vida e Previdência. Essa lógica está integrada permanentemente em `find_candidate()` no script 06 (não é um patch avulso).
- **Bug encontrado e corrigido durante essa revisão**: a função de casamento de marca (script 06) comparava o token de marca contra a razão social por **substring**, não por token inteiro — isso causava falsos positivos silenciosos: "MG Seguros" batia com "**B**MG SEGURADORA" (substring "MG" dentro de "BMG"), e "Mon Seguros" batia com "**MON**GERAL AEGON" (substring "MON" dentro de "MONGERAL"). Ambos os casos faziam parte dos 110 matches originalmente aceitos como corretos. Corrigido para exigir correspondência de token inteiro; as duas empresas voltaram a "sem candidato" (não há empresa correta correspondente na base de 117 seguradoras auto). Resultado líquido depois de todas as correções: **113 nomes fantasia mapeados** (de 110 antes, com 2 falsos positivos removidos e 5 novos resolvidos por CNPJ), totalizando **182.488 linhas** em `consumidor_empresa_uf_mes.csv`.
- **Consequência 2 — completude do Reclame Aqui**: `empresa_id=64` ("Bradesco Auto/RE Companhia de Seguros") havia sido descartado na fase 6 por bater erroneamente com a página do banco. Agora sabemos que compartilha CNPJ com `empresa_id=69` ("Bradesco Seguros S.A"), que tem o perfil correto do RA. As métricas e o histórico mensal de `empresa_id=69` foram replicados para `empresa_id=64` em `ra_metricas_periodo.csv`/`ra_historico_mensal.csv` (linhas marcadas com `match_score="replicado_mesmo_cnpj_de_empresa_id_69_grupo_G09"`, não como um match de busca independente), e `dim_empresa.csv` foi atualizado com o mesmo `nome_reclame_aqui`/`url_reclame_aqui`. Isso evita que um pesquisador que junte os dados do SUSEP (que tem linhas próprias para `empresa_id=64`) com o Reclame Aqui encontre uma lacuna artificial.
- **Nota para uso futuro**: séries históricas da SUSEP (prêmio/sinistro por `coenti`) que atravessam uma reformulação societária ficam "cortadas" entre dois `codigo_susep` — uma análise de série temporal por empresa deveria agrupar por `grupo_cnpj_id` quando a coluna estiver preenchida, para não interpretar a mudança de código como queda real de operação.

## 10. Campos codificados decodificados (de/para)

Revisão feita a pedido do usuário, para que nenhuma tabela final exponha apenas códigos internos sem tradução:

- `susep_auto_empresa_uf_mes.csv`: `coenti`→`noenti` e `ramos`→`noramo` já decodificados inline desde a extração original (script 01); `gracodigo` documentado em `QA/susep_ramos_automovel_selecionados.csv`.
- `ra_metricas_periodo.csv`: `tipo_periodo` (códigos internos em inglês do RA: `SIX_MONTHS`, `TWELVE_MONTHS`, `LAST_YEAR`, `PAST_LAST_YEAR`, `LAST_THREE_YEARS`) agora tem `tipo_periodo_label` em português, calculado dinamicamente a partir do ano real de `periodo_inicio` (não hardcoded, para não ficar desatualizado em coletas futuras). `reputacao_status`/`classificacao_ra` decodificados via dicionário oficial `statusLabels` **extraído da própria página do RA** (não adivinhado) — corrigidos dois códigos que antes ficavam sem tradução (`NOT_RECOMMENDED`→"Não recomendada", `NO_INDEX`→"Sem índice").
- `consumidor_empresa_uf_mes.csv`: campo `respondida` (código S/N da fonte) agora tem `respondida_label` (Sim/Não).
- `ibge_*` e `socioeconomico_uf_periodo.csv`: códigos de UF do IBGE já vêm com `uf_nome` ao lado desde a extração original.
- `dim_uf.csv` e `dim_empresa.csv` funcionam como as tabelas de/para centrais para código de UF↔nome↔região e código SUSEP↔razão social↔CNPJ, respectivamente — qualquer tabela que só tenha o código pode ser cruzada com essas duas.

## 11. Regras do plano verificadas nesta sessão

- Nenhum mês/ano foi inventado; ausências permanecem como ausências (não preenchidas com zero nem interpoladas).
- Nenhuma associação de empresa foi tratada como definitiva por fuzzy matching isolado — todas ficam marcadas como candidatas, com confiança e fundamentação registradas.
- Todo fallback de fonte primária indisponível foi explicitamente identificado e documentado (não houve substituição silenciosa).
- Toda transformação está em código reproduzível (`scripts/01` a `scripts/06`).
