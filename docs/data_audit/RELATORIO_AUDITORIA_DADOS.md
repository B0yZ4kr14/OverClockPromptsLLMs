# Relatório de Auditoria de Qualidade de Dados — Edição Interativa

**Ferramenta:** skill `dataset-quality-audit` (12 dimensões de qualidade, nota 0–100, grau A+–F)
**Data:** 27/07/2026 · **Alvo:** datasets de `app/js/data.js` (`window.RPT`) extraídos para JSON
**Escopo:** 11 datasets tabulares · relatórios JSON completos nesta mesma pasta (`*_audit.json`)

## Resultado consolidado

| Dataset | Linhas × Colunas | Nota | Grau |
|---|---|---|---|
| browsecomp_steps | 3×3 | 100,0 | **A+** |
| browsecomp_cost_efficiency | 3×2 | 100,0 | **A+** |
| canonical_graph_edges | 3×3 | 100,0 | **A+** |
| dead_zone_landmarks | 4×2 | 100,0 | **A+** |
| key_numbers | 5×4 | 100,0 | **A+** |
| mitigations | 9×6 | 100,0 | **A+** |
| u_curve_points | 3×3 | 100,0 | **A+** |
| canonical_graph_nodes | 4×4 | 99,06 | **A+** |
| models_2026 | 7×6 | 98,93 | **A+** |
| **benchmarks** | 45×9 | 85,96 | **B** |
| **context_windows** | 10×6 | 85,0 | **B** |

**Média ponderada: 97,2/100 — grau A+.** Nove dos onze datasets com qualidade excelente.

## Análise dos dois graus B (veredito: padrões intencionais, não defeitos)

### `benchmarks` (85,96) — matriz de 45 benchmarks × 6 modelos
1. **Valores ausentes (22–49% em algumas colunas de modelo)** — os `null` são **fiéis ao documento-fonte**: o blog oficial não reportou aquele modelo naquele benchmark (o site renderiza "—"). Preencher com moda/mediana, como a sugestão automática indica, **fabricaria dados** — explicitamente proibido pela política editorial da obra. Mantido.
2. **Consistência de tipo (valores como `3*`, `17.3*`)** — o asterisco é a **marca metodológica do fabricante** ("reavaliação interna"), preservada por fidelidade à fonte. Melhoria estrutural opcional (não aplicada): separar `valor` numérico de `flag` booleano — implicaria reescrever a matriz e os drill-downs sem ganho factual.
3. **Outliers IQR na coluna Kimi (9,0–96,1)** — amplitude legítima entre benchmarks de naturezas distintas (ZeroBench ≈ 9 vs. AIME 96,1).
4. **Coluna constante `hl`** — flag de destaque da UI (linha do Kimi K2.5); zero informação estatística por design, não por erro.

### `context_windows` (85,0) — evolução das janelas 2022–2026
1. **Outlier (Llama 4 Scout, 10M tokens)** — valor real e documentado (Meta, abr/2025), pico histórico da série.
2. **Assimetria extrema (skewness 3,04)** — a série cobre **3 ordens de magnitude** (4K→10M); é exatamente por isso que o site a plota em **escala logarítmica** — a "transformação log" sugerida pelo auditor já é a prática adotada na visualização.

## Conclusão
Os dados da Edição Interativa estão **aptos para uso** (A+ médio). Os apontamentos residuais derivam de **decisões deliberadas de fidelidade às fontes primárias** (nulos = "não avaliado"; asteriscos = marca do fabricante; dispersão log) e **não devem ser "corrigidos"** sob pena de violar a integridade factual da obra.
