# Relatório de QA — Dados Externos (Frota, Sinistralidade, Emplacamentos)

Data: 2026-09-22. Ver `README.md` para detalhe por fonte e `fontes.csv` para proveniência completa (URL, data/hora, SHA-256).

## 1. Quais fontes foram obtidas?

Todas as 6 fontes do escopo, mais a investigação sobre novas apólices:

| # | Fonte | Status |
|---|---|---|
| 1 | SENATRAN — Frota de veículos | Completo, 2016-01 a 2026-07 |
| 2 | AUTOSEG/SUSEP — Exposição e sinistros | Completo dentro do disponível: 2016-S1 a 2020-S2 (sem banco para download 2021+) |
| 3 | SINESP — Roubo/furto de veículos | Parcial: 2015-01 a 2022-12 (fonte com falha de DNS; recurso mais recente inacessível) |
| 4 | RENAEST — Sinistros de trânsito | Completo dentro do disponível: 2018-01 a 2026-04, com lacunas estaduais reais |
| 5 | PRF — Acidentes em rodovias federais | Completo: 2016-01 a 2025-12 (2026 não publicado pela fonte) |
| 6 | Emplacamentos — Fenabrave (nacional + % regional) | Completo em nível nacional/regional-percentual: 2016-01 a 2026-08 (6 meses com corrupção de fonte excluídos) |

## 2. Qual período real de cada uma?

Ver tabela acima e `cobertura_temporal.csv` linha a linha (ano × mês × fonte).

## 3. Qual último mês/semestre disponível?

- SENATRAN: 2026-07
- AUTOSEG: 2020-S2 (não há banco mais recente para download)
- SINESP: 2022-12
- RENAEST: 2026-04
- PRF: 2025-12
- Emplacamentos: 2026-08

## 4. Existem períodos faltantes?

Sim, todos documentados e não preenchidos:
- AUTOSEG: 2021-01 em diante (sem banco de dados para download, só painel on-line, não usado por instrução do plano)
- SINESP: 2023-01 em diante (fonte com DNS fora do ar; recurso mais recente 2015-2026 inacessível, sem snapshot arquivado)
- RENAEST: lacunas estaduais desiguais dentro de 2018-2026 (ver `renaest_cobertura_uf.csv` — algumas UFs com 1 mês, outras com 100)
- PRF: 2026 (ano em curso, não publicado pela fonte)
- Emplacamentos: 6 meses pontuais com corrupção de fonte no PDF original (2020-04/05/06, 2023-09, 2024-01/04)

## 5. Existem UFs faltantes?

- SENATRAN, SINESP, RENAEST, PRF: 27/27 UFs presentes (RENAEST com cobertura temporal desigual por UF, não UFs ausentes)
- AUTOSEG: geografia por "região de circulação" (sub-estadual), não UF direta; `uf_aproximada` derivada de texto oficial cobre 100% das 41 regiões
- Emplacamentos: sem UF (limitação da fonte, documentada — só nacional + % por 5 macrorregiões)

## 6. Houve mudança de schema?

Sim, em 3 fontes, todas documentadas:
- **PRF**: formato de data (`DD/MM/AA` em 2016 → `AAAA-MM-DD` de 2017+); colunas de geolocalização/unidade só a partir de determinado ano (`prf_schema_por_arquivo.csv`)
- **SENATRAN**: nomenclatura de arquivo inconsistente entre anos; algumas planilhas incluem uma "Tabela 2" extra no mesmo arquivo
- **Fenabrave**: padrão de nome de arquivo mudou de ano para ano (links capturados via navegador, não adivinhados)

## 7. Existem duplicidades?

Ver `duplicidades.csv`. Praticamente nenhuma: só **1 registro exatamente duplicado** em 729.030 linhas de `prf_acidentes_ocorrencia.csv` (mesmo `id`, mesma data/UF/município) — duplicidade da própria fonte (arquivo de 2016), não removida, documentada.

## 8. Existem arquivos corrompidos?

Sim: **6 de 127** informativos mensais da Fenabrave têm corrupção de mapeamento de caracteres na origem (texto ilegível mesmo nos títulos, não um problema da nossa extração) — excluídos e listados em `fenabrave_arquivos_nao_extraidos.csv`. Nenhum outro arquivo corrompido detectado nas demais fontes (todos os ZIPs validados com `unzip -t`).

## 9. Quais limitações metodológicas existem?

- **AUTOSEG**: `REGIAO` é uma região de circulação sub-estadual da SUSEP, não UF — `uf_aproximada` é uma aproximação baseada em texto oficial, não uma partição exata por UF.
- **SINESP**: cobertura só até 2022; fonte primária com falha de DNS, dependente de fallback do Internet Archive.
- **RENAEST**: cobertura estadual desigual (nem toda UF tem a série completa 2018-2026).
- **Emplacamentos**: sem granularidade UF (só nacional + % por 5 macrorregiões); a camada regional é percentual, não absoluta.
- **PRF × RENAEST**: não somados entre si (metodologias e coberturas geográficas diferentes — PRF é só rodovias federais).
- **`delta_exposicao_segurada`**: mede variação líquida de exposição, não contagem de novas apólices (ver seção 7 do plano e `investigacao_novos_seguros.md`).

## 10. Qual base está pronta para análise?

`senatran_frota_uf_mes_tipo.csv`, `prf_acidentes_uf_mes.csv` / `prf_acidentes_ocorrencia.csv`, `renaest_uf_mes.csv`, `sinesp_roubo_furto_uf_mes.csv`, `autoseg_semestre_geografia.csv` — todas com granularidade UF × período, chaves validadas, sem duplicidades relevantes.

## 11. Qual precisa de revisão?

- `emplacamentos_novos_regiao_percentual.csv`: validado internamente (somas ≈100%), mas é percentual, não absoluto — precisa de tratamento especial em qualquer análise futura (não multiplicar ingenuamente por total nacional sem documentar que é uma estimativa derivada).
- `autoseg_sinistros_causa_semestre.csv`: cobre só a cobertura CASCO (única tabela do AUTOSEG com causa de sinistro detalhada e chave por região) — não representa RCDP/RCDM/APP.
- Séries com lacunas (SINESP, AUTOSEG pós-2020, RENAEST por UF) precisam de tratamento explícito de dados faltantes em qualquer modelagem futura (não interpolar).

## 12. Foi encontrada fonte pública de novos seguros?

**Não.** Ver `investigacao_novos_seguros.md` para detalhe completo. O único dataset público SUSEP no catálogo nacional é a base SES (agregada semestral, já usada). O SRO tem consulta pública, mas exclusiva para Seguro Garantia, por apólice individual, com CAPTCHA obrigatório — não é uma base agregada e não cobre Automóvel.

## 13. É possível distinguir novas apólices de renovações?

**Não**, em nenhuma fonte pública encontrada.

## 14. Em quais níveis geográficos?

Não aplicável (não há fonte).

## 15. Qual a cobertura histórica real dos emplacamentos?

2016-01 a 2026-08 (121 de 127 meses extraídos com sucesso), em nível **nacional por categoria de veículo**, mais uma camada de **participação percentual entre 5 macrorregiões** (não UF, não valor absoluto). SENATRAN não publica série de emplacamentos novos (só estoque de frota); Fenabrave foi a única fonte institucional que atendeu aos critérios do plano (período identificável, metodologia documentada, valores reproduzíveis), com a limitação de granularidade geográfica explicitada.
