# Notebooks do projeto

**Moeda adotada:** dólar americano (USD), por premissa do projeto baseada no contexto de Seattle. Não houve conversão cambial nem alteração dos valores. A fonte não confirmou explicitamente a moeda. Preços e erros monetários estão em USD; percentuais, R² e contribuições SHAP em log mantêm suas próprias escalas.

O notebook 01 contém a EDA executada sobre as 21.613 vendas do histórico completo. O notebook `02_merge_eda_complementar.ipynb` contém merge validado, EDA complementar, hipóteses e exportação dos dados combinados. O notebook 03 está executado: cria sete variáveis candidatas e salva as entradas históricas e futuras com um contrato de colunas. O notebook 04 está executado com split, três janelas de validação temporal e baselines; o notebook 05 compara Ridge, Random Forest, XGBoost e LightGBM, investiga grupos de features e ajusta o candidato com Optuna. O notebook 06 avalia o modelo e gera as previsões futuras. As etapas 07 e 08 estão documentadas, sem código executável; a comunicação no 09 está concluída em Markdown, com gráficos e resumo executivo.

Fluxo: **EDA geral → merge → EDA pós-merge → preparação das variáveis → split e modelagem**.

O notebook 02 executa de forma independente, lendo os três arquivos originais. Salva `imoveis_com_demografia.csv` e `futuros_com_demografia.csv` em `data/processed/`; tabelas e gráficos ficam em `reports/`. Veja `docs/merge_eda_summary.md` para resultados. O notebook 04 define treino, validação e teste. Transformações que aprendem estatísticas serão ajustadas apenas no treino; a avaliação registrará que a exploração inicial considerou todo o histórico.

Selecione `.venv/Scripts/python.exe` no Jupyter e execute as células do notebook 01 em ordem. A execução lê `data/raw/` e regenera gráficos e tabelas em `reports/`; não gera partições. Veja `docs/eda_summary.md` para conclusões.

Os arquivos da divisão anterior estão em `reports/archive/eda_temporal_anterior/`, apenas como histórico. O notebook ativo desta etapa é `01_eda.ipynb`.

O notebook 03 lê os CSVs do merge e gera `features_historico.csv`, `features_futuros.csv`, `identificacao_alvo_historico.csv` e `contrato_features.json` em `data/processed/`. Execute suas células em ordem. Ele não ajusta imputadores, normalização ou codificação de CEP; essas etapas serão ajustadas no treino após o split.

O notebook 04 executa nove ajustes leves, um por vez, e grava os manifestos de split e CV. Consulte `docs/baseline_summary.md` e reutilize as mesmas partições na etapa 05. No encerramento do notebook 04, o teste permanecia sem métricas; ele foi avaliado posteriormente no 06.

O notebook 05 reutiliza o split e a CV do notebook 04. Execute suas células em ordem; os limites de treinamento não contam pausas de leitura. Ele salva os experimentos, o registro de hipóteses e a configuração escolhida para a avaliação final. Consulte `docs/model_selection_summary.md`.

O notebook 06 está executado, com avaliação final, explicabilidade e previsões futuras. Reexecute suas células em ordem para reproduzir os relatórios usando os modelos salvos. Resultados em `docs/evaluation_summary.md`; a etapa 07 documenta como integrar a preparação e o modelo em uma API.

Os notebooks `07_empacotamento_api.ipynb` e `08_deploy_monitoramento.ipynb` são somente documentação, sem células de código. Basta ler as células Markdown. Descrevem empacotamento, contrato HTTP, Docker, arquitetura, monitoramento e atualização do modelo. Nenhum endpoint, contêiner ou serviço foi implementado. Os diagramas são imagens incorporadas aos notebooks, acompanhadas de explicação textual. O código Mermaid fica disponível nos documentos em `docs/`.

As seções 9 dos notebooks 07 e 08 documentam também a integração com API externa de LLM para explicações: contrato, dados enviados, verificações de texto, contingência, custo, observabilidade e versionamento. Não há integração executada nem alteração no cálculo do preço.

O notebook `09_comunicacao_stakeholders.ipynb` contém 13 células Markdown para apresentação. Os dois gráficos executivos foram produzidos a partir dos relatórios salvos; não houve novo ajuste de modelo. Consulte também `docs/executive_summary.md`.


### Organização das verificações

As validações ficam próximas da leitura e preparação dos dados, da definição das partições e do reaproveitamento dos modelos. Elas interrompem a execução quando continuar poderia comprometer a avaliação. As análises utilizam os dados já conferidos, evitando repetir controles em cada bloco. Os casos simulados ficam em seções próprias dos notebooks 02 e 03. Nos notebooks 05 e 06, as definições de preparação e o reaproveitamento dos resultados estão em células separadas.
