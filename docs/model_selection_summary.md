# Seleção de modelo

**Moeda adotada:** dólar americano (USD), por premissa do projeto baseada no contexto de Seattle. Não houve conversão cambial nem alteração dos valores. A fonte não confirmou explicitamente a moeda. Preços e erros monetários estão em USD; percentuais, R² e contribuições SHAP em log mantêm suas próprias escalas.

O notebook `05_selecao_modelo.ipynb` compara **Ridge, Random Forest, XGBoost e LightGBM** nas três janelas temporais do notebook 04. Como o alvo é um preço contínuo, usamos `Ridge`, `RandomForestRegressor`, `XGBRegressor` e `LGBMRegressor`. Ridge é regressão linear com regularização e permanece como baseline previamente validado.

## Resultado atual

Selecionado: **LightGBM com 50 entradas e transformação log1p do preço**, revertida antes das métricas. São 18 características físicas, sete derivadas e 25 indicadores demográficos, sem `hous_val_amt`. Sua disponibilidade histórica ainda precisa ser confirmada antes da implantação.

| Configuração | MAE médio da CV |
|---|---:|
| Ridge inicial, 18 entradas | 97.344,71 |
| Random Forest inicial, 18 entradas | 69.859,88 |
| XGBoost inicial, 18 entradas | 67.780,27 |
| LightGBM inicial, 18 entradas | 66.411,63 |
| LightGBM selecionado, 50 entradas e log | 62.249,43 |

O candidato final apresentou RMSE médio de 111.291,13 e R² médio de 0,9000. MAEs por janela: 60.093,61; 60.407,63; e 66.247,05. Nenhuma previsão de validação foi não positiva. Estas são métricas da seleção, não do teste final.

## Sequência e decisões

1. Comparar os quatro algoritmos com 18 entradas, reutilizando Ridge após conferir dados, parâmetros e partições.
2. Nos dois melhores, LightGBM e XGBoost, comparar quatro grupos de entradas: físicas, físicas com derivadas, demografia sem `hous_val_amt` e todas as candidatas.
3. Avaliar o log do preço na combinação escolhida.
4. Ajustar somente o candidato escolhido com Optuna: até oito tentativas e 90 segundos, sequencialmente.
5. Congelar parâmetros e ordem das colunas para a avaliação final.

O critério principal é o MAE médio das três janelas, na escala original. Entre configurações até 1% acima do menor MAE, preferimos menos colunas e depois menor erro. Essa tolerância é operacional, não evidência de equivalência estatística.

Na comparação de grupos, XGBoost com 51 entradas teve MAE de 64.932,86. LightGBM com 50 teve 65.372,33, diferença de apenas 0,68%; foi escolhido pela regra de simplicidade. O log reduziu seu MAE para 63.121,53. O Optuna reduziu para 62.249,43, com sete tentativas completas e uma interrompida. Resultados parciais de tentativas interrompidas não entram na seleção.

## Hipóteses e limites

As sete derivadas aumentaram o MAE médio em 0,57% no LightGBM e 0,07% no XGBoost. A demografia sem `hous_val_amt`, adicionada ao grupo com derivadas, melhorou as três janelas nos dois modelos: redução média de 2,12% e 3,07%, respectivamente. Acrescentar `hous_val_amt` piorou o LightGBM em 0,12%, mas melhorou o XGBoost em 1,24%.

Esses experimentos medem contribuição preditiva de grupos e não causalidade. Não testamos demografia sem derivadas; não demonstramos que todas as 50 entradas sejam necessárias. A origem, interpretação e disponibilidade temporal dos indicadores continuam pendentes. O registro detalhado está em `reports/metrics/selecao_hipoteses.csv`.

A mesma CV foi utilizada para seleção e ajuste, portanto seu melhor resultado pode ser otimista. A EDA anterior examinou o histórico completo. Ao encerrar a seleção, o teste ainda não tinha métricas (a avaliação posterior está em `evaluation_summary.md`); os 100 exemplos futuros não participaram da seleção nem constituem prova independente de generalização.

## Recursos, implementação e reprodução

A execução atual fez 52 ajustes novos e reaproveitou três, levando aproximadamente 30,53 segundos na execução registrada. A maior memória do processo observada entre etapas foi de aproximadamente 239,21 MiB, não o pico exato nem o consumo de toda a máquina.

Cada pipeline aprende imputação, codificação de CEP e eventual padronização apenas no treino da janela. Ridge utiliza matriz esparsa e padronização; os modelos de árvores recebem entradas densas. O LightGBM recebe DataFrames com nomes consistentes entre treino e previsão. Não há corte de outliers ou uso de early stopping com conjuntos adicionais nesta etapa.

Usamos CPU, até duas threads e experimentos sequenciais. O orçamento global é de 300 segundos de ajuste e previsão, sem contar pausas de leitura. Os limites são verificados entre ajustes; um ajuste em andamento pode ultrapassá-los. Referências: [Optuna Study.optimize](https://optuna.readthedocs.io/en/stable/reference/generated/optuna.study.Study.html), [XGBRegressor](https://xgboost.readthedocs.io/en/stable/python/python_api.html#xgboost.XGBRegressor) e [LGBMRegressor](https://lightgbm.readthedocs.io/en/stable/pythonapi/lightgbm.LGBMRegressor.html).

Execute 03, 04 e 05 em ordem com o ambiente de `requirements.txt`. Os parâmetros finais, versões e hashes estão em `artifacts/configuracao_modelo_selecionado.json`, versão `selected_candidate_v2`. As métricas e gráficos ficam em `reports/`. O JSON descreve a configuração congelada; ainda não é um estimador treinado para produção. A comparação anterior foi arquivada em `reports/archive/selecao_v1/`.


### MAPE complementar na validação

O candidato final teve MAPE médio de **11,97%** nas três janelas. Agrupando as 5.935 vendas de validação, o MAPE foi **11,92%**; a diferença vem do peso de cada janela. MAE permaneceu como critério de seleção. Esses percentuais complementam a interpretação e não representam acurácia.

As métricas do teste, calculadas posteriormente, estão em [evaluation_summary.md](evaluation_summary.md) e não orientaram a escolha do modelo.
