# Resultados da EDA — histórico completo

**Moeda adotada:** dólar americano (USD), por premissa do projeto baseada no contexto de Seattle. Não houve conversão cambial nem alteração dos valores. A fonte não confirmou explicitamente a moeda. Preços e erros monetários estão em USD; percentuais, R² e contribuições SHAP em log mantêm suas próprias escalas.

A análise executada em `notebooks/01_eda.ipynb` utiliza todas as **21.613 vendas**, sem split, filtragem por data ou exclusão de imóveis repetidos. Não houve limpeza dos arquivos originais nem treinamento de modelo.

## Escopo e fluxo

**EDA geral → merge → EDA complementar → preparação das variáveis → split e modelagem.**

O split é definido no notebook 04, depois desta EDA. Transformações que aprendem estatísticas são ajustadas apenas no treino de cada partição. A avaliação do notebook 06 registra que a exploração inicial considerou o histórico completo.

## Achados observados

- Histórico: 21.613 linhas e 21 colunas; demografia: 70 linhas e 27 colunas; exemplos futuros: 100 linhas e 18 colunas.
- Não há valores ausentes reconhecidos pelo pandas nem linhas inteiramente duplicadas. Há 176 IDs repetidos, com 177 ocorrências além da primeira.
- Todas as linhas encontram CEP na demografia.
- Os 100 exemplos futuros coincidem nas 18 características com algum registro histórico. Não foram recuperados preços por correspondência; esse arquivo não demonstra desempenho independente.
- Preço mediano: 450.000, em USD, por premissa do projeto.
- Assimetria do preço: aproximadamente 4,024; **1.146 extremos** pela regra de 1,5 × IQR, preservados.
- **13** registros com zero quartos e **10** com zero banheiros.
- **12** registros com ano de construção posterior à venda e **6** com reforma posterior.
- Maiores correlações absolutas de Spearman com preço: `grade`, `sqft_living` e `sqft_living15`. Correlação não demonstra causa nem importância de modelo.

As contagens de problemas podem se sobrepor. Extremos e zeros precisam de investigação, sem correção automática. Fonte, definição, unidade e período dos indicadores demográficos continuam pendentes. O uso de `hous_val_amt` foi posteriormente comparado em grupos no notebook 05; não foi comprovado vazamento pelo nome da coluna.

## Próxima etapa e artefatos

Implementar a junção por CEP, validar preservação de linhas e fazer EDA pós-merge. Ao comparar regiões, distinguir estatísticas por venda de estatísticas com uma linha por CEP.

Os cinco gráficos estão em `reports/figures/`. As tabelas de qualidade e estatísticas completas são `eda_quality_full_history.csv` e `eda_summary_full_history.csv`, em `reports/metrics/`. A síntese numérica está em `eda_findings.json`.

Não há divisão ativa nesta EDA. O manifesto e o protocolo temporais anteriores foram arquivados em `reports/archive/eda_temporal_anterior/` e não devem ser reutilizados como partições do fluxo atual.
