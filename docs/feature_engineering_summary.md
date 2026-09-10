# Preparação das variáveis

**Moeda adotada:** dólar americano (USD), por premissa do projeto baseada no contexto de Seattle. Não houve conversão cambial nem alteração dos valores. A fonte não confirmou explicitamente a moeda. Preços e erros monetários estão em USD; percentuais, R² e contribuições SHAP em log mantêm suas próprias escalas.

## Objetivo

Criar candidatas reproduzíveis com as mesmas regras para histórico e previsão. O notebook [03_feature_engineering.ipynb](../notebooks/03_feature_engineering.ipynb) foi executado sem split ou treinamento.

## Transformações implementadas

As 44 entradas do merge foram preservadas e receberam sete candidatas:

| Variável | Regra |
|---|---|
| `has_recorded_basement` | Indica área de porão positiva; ausência ou valor negativo permanece ausente na derivada |
| `has_renovation_year` | Indica ano de reforma positivo; zero não é interpretado como ano real |
| `basement_area_ratio` | Área de porão dividida pela área habitável positiva |
| `living_area_per_bedroom` | Área habitável dividida pela quantidade positiva de quartos |
| `bathrooms_per_bedroom` | Banheiros divididos pela quantidade positiva de quartos |
| `log_living_area` | Logaritmo de 1 + área não negativa |
| `log_lot_area` | Logaritmo de 1 + área não negativa |

A função não usa preço, ID, data, estatísticas de outras casas ou data atual do sistema. Zero quartos produz ausência nas razões, sem excluir linhas. Não houve correção automática de outliers ou do registro com 33 quartos.

## Resultados

- 21.613 linhas históricas e 100 futuras, com 51 entradas candidatas na mesma ordem e com os mesmos tipos.
- 13 ausências históricas em cada razão por quarto; nenhuma ausência derivada nos exemplos futuros.
- Mediana de área por quarto: aproximadamente 576,67 na unidade de área da fonte por quarto. Essa razão não mede o tamanho físico de um quarto.
- Mediana de banheiros por quarto: 0,625.
- Proporção mediana de porão: zero; máximo observado de aproximadamente 0,667.
- Os valores originais, a ordem das linhas e o preço foram preservados.

As hipóteses sobre utilidade das razões e dos logaritmos foram posteriormente comparadas em grupos no notebook 05. Não há evidência de ganho preditivo nesta etapa.

## Verificações

Conferidos os valores das colunas originais, o esquema entre histórico e futuros, a transformação de uma casa sozinha ou em lote e a reordenação das entradas. Os CSVs são exportados diretamente das tabelas preparadas; não há releitura automática nessa etapa.

Casos simulados verificaram denominadores inválidos, área negativa, porão ausente, CEP desconhecido, indicadores demográficos ausentes e coluna obrigatória faltante. Simulações não foram exportadas como dados.

## Conjuntos candidatos

Foram registrados quatro conjuntos: 18 entradas físicas originais; 25 físicas com derivadas; 50 com demografia sem `hous_val_amt`; e todas as 51 candidatas. Nesta etapa 03 não há conjunto vencedor; a seleção posterior está documentada no notebook 05. Fonte e disponibilidade temporal dos indicadores precisam ser esclarecidas antes do uso operacional; preservar `hous_val_amt` no arquivo não autoriza automaticamente sua inclusão no modelo.

## Arquivos gerados

- `data/processed/features_historico.csv`
- `data/processed/features_futuros.csv`
- `data/processed/identificacao_alvo_historico.csv`: preço, ID, data e posição original, sem misturar essas informações nas features
- `data/processed/contrato_features.json`: ordem, tipos, regras, conjuntos candidatos e hashes das fontes
- Catálogo, diagnóstico, ausências, resumo e plano de pipeline em `reports/metrics/features_*.csv`
- Comparação gráfica das áreas originais e logarítmicas em `reports/figures/features_areas_log.png`

A correspondência entre features históricas e identificação/alvo usa a mesma ordem das linhas originais. Se os arquivos forem reordenados, esse alinhamento precisa ser mantido explicitamente.

## Próximo passo

No notebook 04: definir o split e ajustar preenchimento de ausências, codificação de CEP e normalização apenas no treino, antes dos baselines. Seleção de conjuntos e comparação de tratamentos continuarão no notebook 05. A função por linha deverá integrar o pipeline definitivo na implementação da solução. A EDA anterior sobre todo o histórico será registrada na avaliação.
