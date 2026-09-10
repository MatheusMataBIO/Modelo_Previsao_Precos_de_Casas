# Previsão de preços de casas 

**Moeda adotada:** dólar americano (USD), por premissa do projeto baseada no contexto de Seattle. Não houve conversão cambial nem alteração dos valores. A fonte não confirmou explicitamente a moeda. Preços e erros monetários estão em USD; percentuais, R² e contribuições SHAP em log mantêm suas próprias escalas.

Estimar preços de imóveis da região de Seattle a partir de características físicas e indicadores demográficos por CEP. É um problema de **regressão**, desenvolvido em Python e Jupyter. O [enunciado preservado](docs/enunciado_desafio.md) descreve os requisitos; o [plano do projeto](PLANO_PROJETO.md) organiza as etapas.

**Resultado:** LightGBM com MAE de teste de **74.089,41**, redução de **27,48%** frente ao Ridge nas mesmas vendas. Foram exportadas [100 previsões com todas as características originais](reports/predictions/future_unseen_predictions.csv). O resultado é uma avaliação histórica; não há aprovação para uso automático em produção.

## Dataset com os preços previstos

**Arquivo final:** [reports/predictions/future_unseen_predictions.csv](reports/predictions/future_unseen_predictions.csv).

Esse CSV contém os **100 imóveis** de `data/raw/future_unseen_examples.csv`, na mesma ordem, com **21 colunas**:

- As **18 características originais** de cada imóvel, com seus valores preservados.
- **`predicted_price`**: preço estimado pelo LightGBM, em USD conforme a premissa monetária do projeto.
- **`source_row`**: posição da linha no arquivo original, começando em zero e sem contar o cabeçalho.
- **`model_version`**: versão do modelo utilizado (`avaliacao_v1`).

O arquivo é gerado pelo [notebook 06 — avaliação e explicabilidade](notebooks/06_avaliacao_explicabilidade.ipynb), na seção de previsão dos exemplos futuros. No GitHub, abra o link do CSV acima e use **Download raw file** para baixá-lo.

**Arquivo de avaliação separado:** [reports/predictions/avaliacao_teste.csv](reports/predictions/avaliacao_teste.csv) contém as 4.331 vendas de teste, com preço real e previsões de LightGBM e Ridge. As métricas vêm desse teste; os 100 exemplos futuros não possuem preço real para avaliação.

## Dados e abordagem

| Arquivo em `data/raw/` | Conteúdo | Linhas |
|---|---|---:|
| `kc_house_data.csv` | Características, data, identificador e preço das vendas | 21.613 |
| `zipcode_demographics.csv` | 26 indicadores e CEP | 70 |
| `future_unseen_examples.csv` | 18 características, sem preço | 100 |

Fluxo: EDA do histórico completo → merge por CEP → EDA complementar → criação de variáveis → split temporal → comparação de modelos → avaliação final → documentação da operação e comunicação.

- **Merge:** junção à esquerda muitos-para-um, feita separadamente para histórico e futuros, preservando linhas, ordem e valores originais.
- **Features:** sete candidatas calculadas por linha. O modelo selecionado usa 18 características físicas, sete derivadas e 25 indicadores, sem `hous_val_amt`. `price`, `id` e `date` não são entradas do modelo.
- **Split:** 17.191 vendas anteriores a 10/03/2015 para treino e validação; 4.331 vendas de 10/03 a 27/05/2015 para teste. Foram excluídas 91 vendas antigas de imóveis presentes no teste, evitando compartilhamento de IDs.
- **Validação:** três janelas posteriores de 45 dias, com treino crescente e exclusão de IDs compartilhados em cada janela. Imputação, codificação do CEP e eventual padronização aprendem somente no respectivo treino.
- **Modelos:** Ridge, Random Forest, XGBoost e LightGBM. Seleção pelo MAE médio das janelas; até 1% acima do menor erro, a regra favorece menos colunas. Essa margem não prova equivalência estatística.
- **Ajuste:** Optuna apenas para o candidato escolhido, com oito tentativas, CPU e até duas threads. A busca não foi exaustiva. LightGBM usa `log1p(price)` no treinamento; `expm1` devolve as previsões à escala original antes das métricas.
- **Extremos:** mantidos. A mediana resume os dados; não remove nem corrige outliers. Não foram excluídos erros grandes da avaliação.

## Resultados e significado do erro

| Métrica no mesmo teste temporal | Ridge | LightGBM |
|---|---:|---:|
| MAE (USD) | 102.170,13 | 74.089,41 |
| MAPE | 20,17% | **12,77%** |
| RMSE (USD) | 172.212,44 | 129.025,96 |
| R² | 0,7805 | 0,8768 |
| Erro absoluto mediano (USD) | 72.542,24 | 44.655,82 |
| Previsões dentro de ±20% do preço real | 61,37% | 80,56% |
| Viés médio (previsto − real, USD) | −36.637,55 | −45.704,70 |

O preço médio do teste é **554.588,88** e a mediana é **465.000,00**. O MAE equivale a **13,36% do preço médio**, uma referência de escala — não MAPE. O erro percentual mediano por imóvel é **10,03%**. Adotamos dólar americano (USD) como premissa pelo contexto de Seattle, sem conversão dos valores.

O modelo melhorou a referência, mas o erro é relevante: entre os 315 imóveis acima de 1 milhão, o MAE é 278.380,19. Não há tolerância de negócio definida. As faixas de ±10% e ±20% são descritivas, não critérios de aprovação. A redução frente ao Ridge combina mudanças de algoritmo, features e parâmetros.

MAE do LightGBM: **41.185,22 no treino final**, **62.249,43 na média da validação** e **74.089,41 no teste**. O teste ficou 19,02% acima da validação. Há diferença de desempenho dentro e fora do treino; não afirmamos ausência de overfitting nem atribuímos toda a diferença a ele, pois os períodos e as amostras também mudam.

## Explicabilidade e limitações

A importância por permutação em 600 vendas, com três repetições, destaca **latitude, área habitável e `grade`**. Localização, tamanho e possível padrão construtivo têm interpretação imobiliária plausível, mas as definições das colunas ainda precisam de confirmação. O notebook 06 calcula **SHAP global nas mesmas 600 vendas**, com barras e beeswarm, e **SHAP local para três exemplos**, na escala log(1 + preço). Somar o valor de referência e as contribuições e aplicar `expm1` reproduz as previsões. O ranking global destaca latitude, `grade` e área habitável. São descrições da amostra e dos casos explicados, sem garantia para todos os imóveis. Veja a [análise de SHAP global](docs/shap_global.md).

Limitações principais:

- A EDA observou todo o histórico antes do split. O teste ficou reservado para a comparação preditiva, mas não foi totalmente desconhecido durante a exploração.
- Fonte e período dos indicadores demográficos não foram confirmados. Excluir `hous_val_amt` não elimina essa incerteza nas outras colunas; não é possível certificar ausência total de vazamento temporal.
- Seleção e Optuna usaram as mesmas janelas, podendo tornar a melhor CV otimista. Novas melhorias precisam de avaliação posterior independente; o teste já examinado não deve orientar ajustes.
- Há subestimação, sobretudo nos imóveis caros; não foram produzidos intervalos de previsão nem comprovado desempenho em outros períodos ou regiões. Não há CEP novo no teste.
- Os 100 futuros coincidem nas entradas com registros históricos e não possuem preço observado. Demonstram execução da inferência, não generalização independente.
- Importâncias descrevem dependência do modelo, não causas do preço. Features correlacionadas e indicadores regionais exigem cautela; não demonstramos que todas as 50 entradas sejam necessárias.

Detalhes: [avaliação](docs/evaluation_summary.md), [ficha do modelo](docs/model_card.md), [resumo executivo](docs/executive_summary.md) e [auditoria do projeto](docs/project_audit.md).

## Organização e escopo entregue

| Etapa | Notebook | Entrega |
|---|---|---|
| 01 | [EDA](notebooks/01_eda.ipynb) | Qualidade, distribuições e associações |
| 02 | [Merge e EDA complementar](notebooks/02_merge_eda_complementar.ipynb) | Integridade, análises regionais e outliers |
| 03 | [Feature engineering](notebooks/03_feature_engineering.ipynb) | Candidatas, casos especiais e contrato de colunas |
| 04 | [Split e baseline](notebooks/04_split_baseline.ipynb) | Manifestos temporais e referências |
| 05 | [Seleção](notebooks/05_selecao_modelo.ipynb) | Comparação, hipóteses preditivas e Optuna |
| 06 | [Avaliação e explicabilidade](notebooks/06_avaliacao_explicabilidade.ipynb) | Teste, SHAP global/local, permutação e previsões futuras |
| 07 | [Empacotamento e API](notebooks/07_empacotamento_api.ipynb) | Documentação de inferência, contrato HTTP, Docker e LLM |
| 08 | [Deploy e monitoramento](notebooks/08_deploy_monitoramento.ipynb) | Documentação de arquitetura, MLflow, monitoramento e aprendizado contínuo |
| 09 | [Comunicação](notebooks/09_comunicacao_stakeholders.ipynb) | Apresentação em Markdown com resultados e gráficos |

**API, Docker, infraestrutura, MLflow e API externa de LLM são propostas documentais, sem implementação ou publicação.** A LLM proposta redigiria explicações a partir de fatos verificados; o preço continuaria sendo calculado pelo LightGBM. Os diagramas e o reentreinamento proposto estão no notebook 08 e em [deployment.md](docs/deployment.md).

`data/raw/` preserva os arquivos recebidos; `data/processed/` guarda merges, features e partições; `artifacts/` contém os pipelines e contratos; `reports/` contém métricas, gráficos e previsões. `reports/archive/` guarda resultados antigos, fora da avaliação atual.

## Estrutura dos arquivos

A árvore abaixo lista os arquivos versionados da entrega, incluindo dados processados, modelos, previsões, relatórios e documentação. `.venv/`, `.git/` e checkpoints locais não fazem parte dessa listagem. `reports/archive/` contém resultados antigos, separados dos relatórios ativos.

<details>
<summary>Expandir a estrutura completa do projeto</summary>

```text
.
├── artifacts/
│   ├── .gitkeep
│   ├── configuracao_modelo_selecionado.json
│   ├── model_card.json
│   ├── modelo_avaliado.joblib
│   └── ridge_avaliado.joblib
├── data/
│   ├── processed/
│   │   ├── .gitkeep
│   │   ├── contrato_features.json
│   │   ├── cv_manifest.csv
│   │   ├── features_futuros.csv
│   │   ├── features_historico.csv
│   │   ├── futuros_com_demografia.csv
│   │   ├── identificacao_alvo_historico.csv
│   │   ├── imoveis_com_demografia.csv
│   │   └── split_manifest.csv
│   └── raw/
│       ├── future_unseen_examples.csv
│       ├── kc_house_data.csv
│       └── zipcode_demographics.csv
├── docs/
│   ├── baseline_summary.md
│   ├── data_dictionary.md
│   ├── deployment.md
│   ├── eda_summary.md
│   ├── enunciado_desafio.md
│   ├── evaluation_summary.md
│   ├── executive_summary.md
│   ├── feature_engineering_summary.md
│   ├── merge_eda_summary.md
│   ├── model_card.md
│   ├── model_selection_summary.md
│   ├── packaging_api.md
│   ├── project_audit.md
│   └── shap_global.md
├── notebooks/
│   ├── 01_eda.ipynb
│   ├── 02_merge_eda_complementar.ipynb
│   ├── 03_feature_engineering.ipynb
│   ├── 04_split_baseline.ipynb
│   ├── 05_selecao_modelo.ipynb
│   ├── 06_avaliacao_explicabilidade.ipynb
│   ├── 07_empacotamento_api.ipynb
│   ├── 08_deploy_monitoramento.ipynb
│   ├── 09_comunicacao_stakeholders.ipynb
│   └── README.md
├── reports/
│   ├── archive/
│   │   ├── eda_temporal_anterior/
│   │   │   ├── eda_quality_development.csv
│   │   │   ├── eda_summary_development.csv
│   │   │   ├── README.md
│   │   │   ├── split_manifest.csv
│   │   │   └── split_protocol.json
│   │   └── selecao_v1/
│   │       ├── 05_selecao_modelo.ipynb
│   │       ├── configuracao_modelo_selecionado.json
│   │       ├── model_selection_summary.md
│   │       ├── selecao_comparacao_inicial.csv
│   │       ├── selecao_configuracoes.json
│   │       ├── selecao_experimentos.csv
│   │       ├── selecao_features.csv
│   │       ├── selecao_hipoteses.csv
│   │       ├── selecao_janelas_final.csv
│   │       ├── selecao_mae_por_janela.png
│   │       ├── selecao_modelos_iniciais.png
│   │       ├── selecao_optuna_trials.csv
│   │       ├── selecao_previsoes_cv.csv
│   │       └── selecao_transformacao_alvo.csv
│   ├── figures/
│   │   ├── .gitkeep
│   │   ├── avaliacao_erros.png
│   │   ├── avaliacao_importancias.png
│   │   ├── avaliacao_segmentos.png
│   │   ├── avaliacao_shap_global_barras.png
│   │   ├── avaliacao_shap_global_beeswarm.png
│   │   ├── avaliacao_shap_local.png
│   │   ├── baseline_cv_temporal.png
│   │   ├── baseline_mae_por_janela.png
│   │   ├── eda_area_grade.png
│   │   ├── eda_correlations.png
│   │   ├── eda_location.png
│   │   ├── eda_pos_merge_faixas_renda.png
│   │   ├── eda_pos_merge_indicadores_preco.png
│   │   ├── eda_pos_merge_outliers_area.png
│   │   ├── eda_pos_merge_redundancia.png
│   │   ├── eda_pos_merge_vendas_mensais.png
│   │   ├── eda_pos_merge_vendas_por_cep.png
│   │   ├── eda_price_distribution.png
│   │   ├── esquema_deploy.png
│   │   ├── esquema_inferencia.png
│   │   ├── esquema_llm.png
│   │   ├── executivo_mae.png
│   │   ├── executivo_segmentos.png
│   │   ├── features_areas_log.png
│   │   ├── selecao_mae_por_janela.png
│   │   └── selecao_modelos_iniciais.png
│   ├── metrics/
│   │   ├── .gitkeep
│   │   ├── auditoria_execucao_notebooks.json
│   │   ├── auditoria_projeto.json
│   │   ├── avaliacao_amostra_importancia.csv
│   │   ├── avaliacao_importancias.csv
│   │   ├── avaliacao_maiores_erros.csv
│   │   ├── avaliacao_metricas.csv
│   │   ├── avaliacao_por_faixa_preco.csv
│   │   ├── avaliacao_por_grade.csv
│   │   ├── avaliacao_por_waterfront.csv
│   │   ├── avaliacao_por_zipcode.csv
│   │   ├── avaliacao_registro.json
│   │   ├── avaliacao_shap_global.csv
│   │   ├── avaliacao_shap_global_importancias.csv
│   │   ├── avaliacao_shap_local.csv
│   │   ├── baseline_cv_janelas.csv
│   │   ├── baseline_cv_previsoes.csv
│   │   ├── baseline_cv_resultados.csv
│   │   ├── baseline_cv_resumo.csv
│   │   ├── baseline_mape_janelas.csv
│   │   ├── baseline_mape_resumo.csv
│   │   ├── baseline_protocolo.json
│   │   ├── eda_findings.json
│   │   ├── eda_overview.csv
│   │   ├── eda_pos_merge_associacoes.csv
│   │   ├── eda_pos_merge_concentracao_ceps.csv
│   │   ├── eda_pos_merge_faixas_renda.csv
│   │   ├── eda_pos_merge_outliers_boxplot.csv
│   │   ├── eda_pos_merge_outliers_diagnostico.csv
│   │   ├── eda_pos_merge_outliers_perfil.csv
│   │   ├── eda_pos_merge_outliers_resumo.csv
│   │   ├── eda_pos_merge_por_cep.csv
│   │   ├── eda_pos_merge_redundancia.csv
│   │   ├── eda_pos_merge_vendas_mensais.csv
│   │   ├── eda_price_correlations.csv
│   │   ├── eda_quality_full_history.csv
│   │   ├── eda_schema.csv
│   │   ├── eda_summary_full_history.csv
│   │   ├── eda_zipcode_coverage.csv
│   │   ├── executivo_exemplos.csv
│   │   ├── features_ausencias.csv
│   │   ├── features_catalogo.csv
│   │   ├── features_diagnostico.csv
│   │   ├── features_plano_pipeline.csv
│   │   ├── features_resumo_derivadas.csv
│   │   ├── merge_integridade.csv
│   │   ├── merge_registro.json
│   │   ├── selecao_comparacao_inicial.csv
│   │   ├── selecao_configuracoes.json
│   │   ├── selecao_experimentos.csv
│   │   ├── selecao_features.csv
│   │   ├── selecao_hipoteses.csv
│   │   ├── selecao_janelas_final.csv
│   │   ├── selecao_mape_janelas.csv
│   │   ├── selecao_mape_resumo.csv
│   │   ├── selecao_optuna_trials.csv
│   │   ├── selecao_previsoes_cv.csv
│   │   └── selecao_transformacao_alvo.csv
│   ├── pdf_preview/
│   │   ├── pagina_01.png
│   │   ├── pagina_02.png
│   │   ├── pagina_03.png
│   │   ├── pagina_04.png
│   │   ├── pagina_05.png
│   │   ├── pagina_06.png
│   │   ├── pagina_07.png
│   │   ├── pagina_08.png
│   │   ├── pagina_09.png
│   │   └── pagina_10.png
│   ├── predictions/
│   │   ├── .gitkeep
│   │   ├── avaliacao_teste.csv
│   │   ├── future_unseen_predictions.csv
│   │   └── README.md
│   └── relatorio_executivo.pdf
├── .gitattributes
├── .gitignore
├── PLANO_PROJETO.md
├── README.md
└── requirements.txt
```

</details>

## Ambiente e reprodução

Com Python 3.13 instalado, execute na raiz pelo PowerShell:

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

Selecione `.venv/Scripts/python.exe` como kernel no Jupyter do IDE. Os notebooks 01 a 06 contêm código; execute as células em ordem. Os notebooks 07 a 09 são documentos Markdown.

A auditoria já realizada está registrada em [project_audit.md](docs/project_audit.md). Os scripts auxiliares usados durante a preparação foram removidos do entregável; a execução do projeto é feita pelos notebooks.

O notebook 06 reaproveita os modelos congelados quando os registros são compatíveis. A assinatura do notebook 06 ignora somente medições de tempo e recursos; alterações nos dados ou decisões do experimento continuam bloqueando o reaproveitamento. Preserve a entrega avaliada e use uma cópia separada para novos experimentos, sem apagar o registro apenas para contornar a conferência.

O pipeline salvo recebe **50 features já preparadas**, não diretamente as 18 colunas originais; o notebook 06 demonstra esse preparo completo. A integração em um único pacote está documentada no 07. A reprodução local é descrita aqui; publicação e envio não são etapas executadas pelos notebooks.

### MAPE: erro percentual médio como complemento

**Resultado no teste:** LightGBM teve **MAPE de 12,77%**, contra **20,17% do Ridge**, nas mesmas 4.331 vendas. Para cada imóvel, dividimos o erro absoluto pelo preço real, multiplicamos por 100 e depois calculamos a média. Isso expressa o tamanho médio do erro relativo; não significa 87,23% de acerto nem uma garantia para cada previsão.

**Por que incluir:** o MAE de USD 74.089,41 informa a escala monetária; o MAPE facilita comparar erros relativos entre preços diferentes. Ele também é diferente dos **10,03% de erro percentual mediano** e dos **13,36% da razão entre MAE e preço médio**.

**Cuidados e decisão:** o mesmo erro em USD pesa mais em imóveis baratos. Preços próximos de zero podem distorcer o MAPE; no teste, o menor preço é USD 81.000. Mantemos **MAE como critério principal de seleção**. MAPE foi acrescentado às previsões existentes, sem novos treinamentos ou escolha pelo teste; não há tolerância de negócio aprovada.

Na validação do candidato final, a média do MAPE das três janelas é **11,97%**. Agrupando todas as vendas de validação, é **11,92%**: as quantidades de vendas por janela diferem. Para comparar períodos com peso igual, usamos o primeiro valor.
