# Seleção de modelo

O notebook `05_selecao_modelo.ipynb` compara Ridge, Random Forest, Extra Trees e HistGradientBoosting nas três janelas temporais definidas no notebook 04. As configurações são comparadas pelo MAE médio na escala original do preço; RMSE, R² e variação entre janelas complementam a leitura.

## Resultado da execução

Selecionado: **HistGradientBoosting**, com 50 entradas (18 físicas, sete derivadas e 25 indicadores demográficos, sem `hous_val_amt`) e transformação `log1p` do preço, revertida antes das métricas. A disponibilidade temporal dos indicadores continua pendente de confirmação antes da implantação.

| Configuração | MAE médio da CV |
|---|---:|
| Ridge inicial, 18 entradas | 97.344,71 |
| Random Forest inicial, 18 entradas | 69.859,88 |
| Extra Trees inicial, 18 entradas | 67.364,74 |
| HistGradientBoosting inicial, 18 entradas | 65.668,25 |
| HistGradientBoosting selecionado, 50 entradas, log e ajuste | 62.456,58 |

A redução relativa ao Ridge foi de 35,84%. O candidato final apresentou RMSE médio de 112.160,85 e R² médio de 0,8988. Os MAEs por janela foram 59.631,47; 62.576,32; e 65.161,95. Nenhuma previsão de validação do candidato foi não positiva. Estas são métricas usadas na seleção, não resultados do teste final.

O Optuna reduziu o MAE de 63.375,43 para 62.456,58, ganho de 1,45% sobre a configuração anterior à busca. Houve oito tentativas: cinco completas e três interrompidas. Ao todo, 48 ajustes novos e três reaproveitados; aproximadamente 120 segundos de experimentos e 284 MiB de memória do processo observada entre etapas, não pico exato. Esses números descrevem esta execução e variam conforme a máquina.

As sete derivadas reduziram o erro em 0,53% no boosting e aumentaram em 0,40% no Extra Trees. Ao acrescentar demografia sem `hous_val_amt` ao conjunto com derivadas, houve melhora nas três janelas dos dois candidatos: 2,56% no boosting e 2,22% no Extra Trees. `hous_val_amt` piorou o boosting em 0,26% e melhorou o Extra Trees em 0,61%. Portanto, os resultados sustentam decisões específicas para o modelo, não uma afirmação universal sobre cada variável. Não comparamos demografia sem as derivadas, nem garantimos que as 50 entradas sejam todas necessárias.

## Sequência executada

1. Comparar os quatro algoritmos com as 18 colunas físicas originais, reaproveitando Ridge após conferir suas métricas e partições.
2. Comparar quatro grupos de colunas nos dois melhores algoritmos: físicas; físicas com sete derivadas; enriquecimento demográfico sem `hous_val_amt`; todas as candidatas.
3. Comparar preço original e transformação logarítmica na combinação selecionada.
4. Usar Optuna apenas no candidato escolhido, com até oito tentativas e 90 segundos de busca.
5. Congelar a configuração para o notebook 06, sem medir o erro do teste nesta etapa.

Na seleção, configurações com MAE até 1% acima do menor erro são elegíveis; priorizamos menos colunas e depois menor MAE. A tolerância é operacional, não um teste de equivalência estatística. A busca não garante o melhor resultado possível para cada algoritmo.

Cada ajuste aprende imputação, codificação do CEP e eventual padronização apenas no treino da respectiva janela. Não cortamos outliers nem aprendemos estatísticas na validação. O logaritmo do alvo, quando utilizado, é desfeito antes de medir o erro.

## Hipóteses e limites da interpretação

`reports/metrics/selecao_hipoteses.csv` registra a diferença de MAE e o número de janelas com melhora para cada mudança de grupo. Esse experimento mede contribuição preditiva conjunta, não confirma causalidade nem a importância individual de cada coluna. A interpretação e a disponibilidade histórica dos indicadores demográficos ainda dependem de documentação da fonte.

As mesmas janelas foram consultadas em vários experimentos. Portanto, a métrica da configuração selecionada serve à escolha e pode ser otimista; a avaliação final ficará no notebook 06. A EDA anterior examinou o histórico completo, limitação que também deve acompanhar a avaliação. Os 100 exemplos futuros não participam da seleção nem constituem evidência independente de generalização.

## Recursos e reprodução

Execute os notebooks 03 e 04 antes deste, usando o ambiente registrado em `requirements.txt`. Execute o notebook 05 em ordem. São usadas até duas threads, um experimento por vez, reaproveitamento de configurações idênticas e interrupção de tentativas pouco promissoras pelo Optuna. O orçamento global é de 300 segundos de ajuste e previsão, sem contar pausas de leitura. Os limites de tempo são verificados entre ajustes e não cancelam um treinamento já iniciado. O limite do Optuna segue esse comportamento: [documentação oficial de Study.optimize](https://optuna.readthedocs.io/en/stable/reference/generated/optuna.study.Study.html).

O HistGradientBoosting usa `early_stopping=False` para preservar a definição externa das janelas, sem adicionar sua validação interna automática: [documentação oficial do estimador](https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.HistGradientBoostingRegressor.html).

As tabelas ficam em `reports/metrics/selecao_*.csv`, os gráficos em `reports/figures/selecao_*.png` e a configuração congelada em `artifacts/configuracao_modelo_selecionado.json`. Esse JSON descreve o candidato; ainda não é um estimador treinado para produção.
