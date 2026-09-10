# Split, validação cruzada e baselines

**Moeda adotada:** dólar americano (USD), por premissa do projeto baseada no contexto de Seattle. Não houve conversão cambial nem alteração dos valores. A fonte não confirmou explicitamente a moeda. Preços e erros monetários estão em USD; percentuais, R² e contribuições SHAP em log mantêm suas próprias escalas.

## Objetivo e escopo

O [notebook 04](../notebooks/04_split_baseline.ipynb) estabelece uma referência de desempenho com 18 características físicas originais. Não compara ainda as features derivadas, indicadores demográficos ou quatro algoritmos finais. Foram feitos nove ajustes: mediana, regressão linear e Ridge, cada um em três janelas temporais.

## Escolha da validação

O cenário é prever vendas posteriores de imóveis não vistos no treino. A data de corte do teste, definida pela posição aproximada de 80% das vendas ordenadas, é **10/03/2015**:

- Treino e validação: 17.191 registros.
- Teste final: 4.331 registros.
- Vendas anteriores separadas porque seus IDs também aparecem no teste: 91.

Nesta etapa 04, o teste não foi utilizado em métricas ou treinamento. A avaliação posterior está no notebook 06. A EDA anterior examinou o histórico completo, portanto essa limitação é explicitada; não é um teste nunca observado durante a exploração.

A validação cruzada usa treino crescente e três janelas posteriores de 45 dias. Para evitar que dias com muitas vendas distorçam a duração das janelas, aplicamos `TimeSeriesSplit` sobre um calendário diário completo e mapeamos os dias para as linhas. A documentação explica a necessidade de espaçamento comparável: [TimeSeriesSplit](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.TimeSeriesSplit.html).

| Janela | Fim do treino | Validação | Vendas de treino | Vendas validadas | Vendas antigas separadas por ID |
|---|---|---|---:|---:|---:|
| 1 | 25/10/2014 | 26/10/2014–09/12/2014 | 11.241 | 2.392 | 15 |
| 2 | 09/12/2014 | 10/12/2014–23/01/2015 | 13.626 | 1.626 | 22 |
| 3 | 23/01/2015 | 24/01/2015–09/03/2015 | 15.235 | 1.917 | 39 |

Dentro de cada janela, nenhum ID é compartilhado entre treino e validação. Vendas validadas numa janela podem integrar um treino posterior porque, nesse momento, já pertencem ao passado. Nenhum ID de teste aparece na CV. As janelas cobrem 5.935 vendas distintas; o início do histórico é usado somente como treino inicial.

Não há intervalo vazio adicional entre treino e validação nesta referência, pois o atraso de disponibilidade dos preços não foi informado. Se houver atraso na aplicação real, será necessário rever esse parâmetro. Não avaliamos generalização para CEPs inteiramente novos: não houve CEP desconhecido nas validações executadas.

## Pipeline

Regressão linear e Ridge usam o mesmo pipeline, clonado e ajustado novamente em cada treino:

1. Números: imputação pela mediana do treino e padronização.
2. CEP: preenchimento categórico e one-hot com categoria de referência removida; categorias desconhecidas são ignoradas.
3. Modelo: regressão linear ou Ridge com `alpha=1.0`, fixo e sem busca.

A referência constante prevê a mediana dos preços de treino e não aprende relações com as features. Não houve corte de outliers, transformação do preço, ajuste de hiperparâmetros nem previsão do arquivo futuro.

## Resultados de validação

As métricas abaixo são médias com peso igual para cada janela. MAE, seu desvio e RMSE estão em USD; R² não tem unidade. MAE não é erro máximo e R² não é percentual de acerto.

| Referência | MAE médio | Desvio do MAE entre janelas | RMSE médio | R² médio |
|---|---:|---:|---:|---:|
| Mediana | 218.136,66 | 1.080,15 | 359.954,33 | -0,041 |
| Regressão linear | 97.285,31 | 1.100,65 | 153.553,25 | 0,810 |
| Ridge | 97.344,71 | 922,20 | 153.417,31 | 0,811 |

A regressão linear reduziu o MAE médio em aproximadamente 55,4% frente à mediana. A diferença de cerca de 59 unidades de preço entre linear e Ridge é pequena em comparação com a variação entre janelas; não sustenta uma declaração de superioridade ampla.

A regressão linear produziu 21 previsões não positivas nas 5.935 vendas validadas; Ridge produziu 20. Elas foram mantidas nas métricas para não esconder limitações. **Hipótese:** transformação do alvo ou modelos não lineares podem melhorar esse comportamento. Investigar na validação do notebook 05, sem escolher tratamentos pelo teste.

## Recursos e verificações

- Nove ajustes sequenciais, no máximo duas threads nas bibliotecas numéricas.
- Cerca de 0,96 segundo para o laço de treinamento/avaliação na execução registrada; não é o tempo total de abertura ou execução do notebook.
- Maior memória Python observada entre etapas: aproximadamente 228,36 MiB, não uma medição exata de pico nem a memória de toda a máquina.
- Conferidos alinhamento das features e preços, hash da fonte, cronologia, separação de IDs, ausência de teste nas previsões e valores finitos.

Os tempos podem variar na reexecução. Não foram exportados modelos finais ou publicados serviços.

## Artefatos e próximos passos

- `data/processed/split_manifest.csv`: divisão externa, baseada na posição original da venda.
- `data/processed/cv_manifest.csv`: papéis de treino, validação e exclusão por janela.
- `reports/metrics/baseline_cv_*.csv`: períodos, métricas, resumo e previsões de validação.
- `reports/metrics/baseline_protocolo.json`: configurações, fontes, limitações e custo observado.
- Figuras `baseline_cv_temporal.png` e `baseline_mae_por_janela.png`.

O notebook 05 deverá reutilizar os manifestos e conferir os hashes antes de comparar quatro candidatos, conjuntos de features e, posteriormente, Optuna com orçamento limitado. O modelo escolhido será avaliado no teste apenas no notebook 06.


## MAPE complementar na validação

Média das três janelas: mediana constante 44,38%; regressão linear 20,89%; Ridge 20,89%. Os cálculos usam as previsões salvas, sem novos ajustes. MAPE expressa erro relativo e não substitui MAE na escolha. As médias com peso por venda são registradas separadamente em `reports/metrics/baseline_mape_resumo.csv`.
