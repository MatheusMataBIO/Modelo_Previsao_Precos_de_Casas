# Dicionário de dados

**Moeda adotada:** dólar americano (USD), por premissa do projeto baseada no contexto de Seattle. Não houve conversão cambial nem alteração dos valores. A fonte não confirmou explicitamente a moeda. Preços e erros monetários estão em USD; percentuais, R² e contribuições SHAP em log mantêm suas próprias escalas.

Versão inicial baseada no enunciado original, nos nomes das colunas e no diagnóstico dos CSVs. Significados não definidos pelo enunciado estão explicitamente marcados como hipóteses a confirmar.

## Fontes e unidades

Os arquivos originais estão em `data/raw/`: imóveis com preço (21.613 linhas), indicadores por CEP (70 linhas) e exemplos sem preço (100 linhas). O enunciado original confirma características físicas, preço e indicadores demográficos por CEP, mas não fornece moeda, metodologia de coleta, ano de referência demográfica ou escalas completas.

## Variáveis dos imóveis

| Coluna | Interpretação e uso inicial |
|---|---|
| `id` | Identificador de imóvel; há repetições. Usar em auditoria e controle das partições, não como preditor. |
| `date` | Data associada à venda, lida no formato `YYYYMMDDTHHMMSS`; usar no corte temporal. |
| `price` | Alvo contínuo: preço. USD adotado como premissa; moeda não explicitada no enunciado original. |
| `bedrooms`, `bathrooms` | Quantidade de quartos e banheiros, conforme os nomes. Banheiros fracionários exigem confirmação da convenção. |
| `sqft_living`, `sqft_lot` | Área habitável e de terreno, interpretação pelos nomes; `sqft` sugere pés quadrados. |
| `floors` | Número de pavimentos, interpretação pelo nome; valores fracionários requerem confirmação. |
| `waterfront` | Indicador de relação com frente d'água, hipótese pelo nome; confirmar codificação. |
| `view`, `condition`, `grade` | Códigos possivelmente associados a vista, conservação e padrão construtivo. Confirmar escalas e critérios. |
| `sqft_above`, `sqft_basement` | Áreas acima do solo e de porão, interpretação pelos nomes. A soma coincide com `sqft_living` no histórico completo, dentro da tolerância numérica usada. |
| `yr_built` | Ano de construção, interpretação pelo nome. |
| `yr_renovated` | Ano de reforma; zero pode significar ausência de reforma registrada, mas a convenção precisa ser confirmada. |
| `zipcode` | CEP textual, chave de enriquecimento demográfico. |
| `lat`, `long` | Latitude e longitude, interpretação pelos nomes; referência geodésica não fornecida. |
| `sqft_living15`, `sqft_lot15` | Medidas de área; significado do sufixo `15` não informado. Não assumir ano ou definição de vizinhança sem fonte. |

Os exemplos futuros contêm as mesmas entradas físicas, mas não têm `id`, `date` nem `price`.

## Variáveis demográficas

As descrições abaixo são hipóteses derivadas das abreviações, não definições confirmadas pela fonte.

| Colunas | Interpretação provisória |
|---|---|
| `zipcode` | CEP usado como chave. |
| `ppltn_qty` | Quantidade de população. |
| `urbn_ppltn_qty`, `sbrbn_ppltn_qty` | Contagens de população urbana e suburbana. |
| `farm_ppltn_qty`, `non_farm_qty` | Contagens de categorias agrícolas e não agrícolas; universo não confirmado. |
| `medn_hshld_incm_amt` | Renda domiciliar mediana. |
| `medn_incm_per_prsn_amt` | Indicador de renda por pessoa; definição estatística exata a confirmar. |
| `hous_val_amt` | Indicador de valor habitacional; agregação e período desconhecidos. |
| `edctn_less_than_9_qty`, `edctn_9_12_qty` | Contagens por faixas de escolaridade; limites e conclusão dos ciclos a confirmar. |
| `edctn_high_schl_qty`, `edctn_some_clg_qty` | Contagens relacionadas a ensino médio e frequência ao ensino superior. |
| `edctn_assoc_dgre_qty`, `edctn_bchlr_dgre_qty`, `edctn_prfsnl_qty` | Contagens por categorias de formação; equivalências e critérios a confirmar. |
| `per_urbn`, `per_sbrbn`, `per_farm`, `per_non_farm` | Possíveis percentuais das categorias populacionais. |
| `per_less_than_9`, `per_9_to_12`, `per_hsd`, `per_some_clg`, `per_assoc`, `per_bchlr`, `per_prfsnl` | Possíveis percentuais por escolaridade. |

Confirmar denominadores e critérios antes de recalcular percentuais ou assumir que grupos somam 100%.

## Disponibilidade temporal e dúvidas

Confirmar moeda, unidades, escalas de avaliação, significado de zeros, origem dos dados e ano de referência dos indicadores demográficos. `hous_val_amt` foi comparado em grupos de features no notebook 05 e não integra o conjunto selecionado, conforme a regra de MAE e simplicidade. Essa escolha não comprova nem elimina vazamento temporal. A disponibilidade histórica de todos os indicadores demográficos continua pendente.

O diagnóstico e as decisões observadas estão em [eda_summary.md](eda_summary.md).
