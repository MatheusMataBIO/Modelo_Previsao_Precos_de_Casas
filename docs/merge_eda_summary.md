# Merge e EDA complementar

**Moeda adotada:** dólar americano (USD), por premissa do projeto baseada no contexto de Seattle. Não houve conversão cambial nem alteração dos valores. A fonte não confirmou explicitamente a moeda. Preços e erros monetários estão em USD; percentuais, R² e contribuições SHAP em log mantêm suas próprias escalas.

## Objetivo

Conectar imóveis e indicadores por CEP, validar a integridade da junção e investigar relações entre regiões e preços no histórico completo. Implementação executada em [02_merge_eda_complementar.ipynb](../notebooks/02_merge_eda_complementar.ipynb).

## Junção validada

- CEP textual, sem espaços e com cinco dígitos nos arquivos recebidos.
- Uma linha demográfica por CEP, validada antes da junção.
- Junção à esquerda muitos-para-um: 21.613 vendas e 100 exemplos futuros preservados.
- Conferência da ordem e de todos os valores originais; nenhum indicador demográfico ausente após a junção.
- Simulação de CEP desconhecido: a linha é preservada com indicadores ausentes. O exemplo artificial não participa da EDA nem das exportações.
- Exportação de 47 colunas no histórico e 44 nos exemplos futuros, incluindo 26 indicadores adicionados. As tabelas são exportadas após a conferência de integridade da junção.

## Resultados observados e hipóteses

Há 70 CEPs, com 50 a 602 vendas por região. Por isso, descrevemos tanto as vendas individuais quanto os preços medianos por CEP com um peso igual para cada região. As linhas de vendas não são observações demográficas independentes.

| Relação com preço | Spearman por venda | Spearman por CEP |
|---|---:|---:|
| Possível renda domiciliar mediana | 0,264 | 0,391 |
| Possível renda por pessoa | 0,624 | 0,845 |
| Possível valor habitacional | 0,677 | 0,913 |

Agregar por CEP muda os pesos e reduz variação entre casas da mesma região. Correlações maiores nessa análise não provam melhor desempenho de um modelo.

As quatro faixas do indicador de renda por pessoa têm 18, 19, 15 e 18 CEPs; valores repetidos impedem grupos exatamente iguais. A mediana dos preços medianos regionais passa de 278.138,50 na primeira faixa a 704.900 na última. A área regional resumida passa de 1.657,5 para 2.217,5 e a classificação resumida de 7 para 8.

**Hipótese:** parte da diferença de preços entre faixas pode acompanhar diferenças no tipo de casa vendido, além do contexto regional. Não foi isolado um efeito causal de renda.

Os possíveis percentuais de formação `per_bchlr` e `per_prfsnl` têm associação de 0,944. Renda por pessoa e `hous_val_amt` têm associação de 0,924.

**Hipótese:** alguns indicadores podem carregar informações semelhantes sobre localização. Comparar grupos de variáveis na modelagem; não excluir colunas apenas pela correlação.

**Hipótese de utilidade:** demografia pode acrescentar contexto além de área e localização. Será necessário comparar modelos com e sem indicadores nas mesmas partições para verificar esse ganho.

Os nomes dos indicadores são interpretações provisórias. Fonte, unidades, denominadores e datas continuam pendentes. `hous_val_amt` foi preservado na junção; sua inclusão foi posteriormente comparada em grupos no notebook 05, sem integrar o modelo escolhido. Origem e período continuam pendentes; não foi comprovado nem descartado vazamento temporal.

## Artefatos e próximo passo

- `data/processed/imoveis_com_demografia.csv`
- `data/processed/futuros_com_demografia.csv`
- `reports/metrics/merge_integridade.csv` e `merge_registro.json`
- Tabelas `reports/metrics/eda_pos_merge_*.csv` e seis figuras `reports/figures/eda_pos_merge_*.png`

Os resumos de preço por CEP e as faixas de renda são apenas relatórios descritivos; não foram adicionados às entradas exportadas. Qualquer futura variável derivada de preços deverá ser calculada somente no treino de cada partição.

Próxima etapa: preparação das variáveis no notebook 03, seguida de split e modelagem no notebook 04. Esta EDA considerou todo o histórico e não produziu métricas de previsão.

## Análises adicionais de concentração e outliers

O notebook inclui barras dos 15 CEPs com mais vendas e vendas mensais. Abril de 2015 tem o maior volume observado, com 2.231 registros. Os meses inicial e final têm cobertura parcial; a série curta e a cobertura desconhecida não sustentam conclusões de sazonalidade ou demanda total.

A regra de 1,5 × IQR identifica 1.146 vendas extremas globalmente e 1.087 dentro dos respectivos CEPs:

| Classificação | Vendas |
|---|---:|
| Extremo nas duas regras | 527 |
| Somente no histórico inteiro | 619 |
| Somente dentro do CEP | 560 |
| Não sinalizado | 19.907 |

Os registros sinalizados nas duas regras têm preço mediano de 1.600.000, área mediana de 4.120 e grade mediano de 10; nos não sinalizados, os valores são 430.000, 1.830 e 7. **Hipótese:** parte dos extremos acompanha diferenças reais de tamanho, padrão e região. Isso não confirma sua validade.

O boxplot agora tem caixas azuis. Seus pontos isolados são medianas regionais, não vendas individuais: CEPs 98118 e 98133 na primeira faixa; 98004 e 98039 na última. A tabela de identificação evita confundir esses pontos com os outliers individuais.

Mediana é uma medida de resumo, não tratamento de outliers. Nenhuma venda foi removida ou limitada; os diagnósticos ficam apenas em relatórios. O notebook 03 cria razões e logaritmos, mas não corta nem remove outliers. Tratamentos por limites permanecem como melhoria futura: seus limites deverão ser aprendidos no treino e comparados na validação.

Foram acrescentados três gráficos (CEPs, meses e área versus preços extremos) e relatórios `eda_pos_merge_concentracao_ceps.csv`, `eda_pos_merge_vendas_mensais.csv` e `eda_pos_merge_outliers_*.csv`. O total atual é de seis figuras no notebook 02.
