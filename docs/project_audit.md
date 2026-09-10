# Auditoria do projeto

> **Como ler este documento:** as seções registram revisões sucessivas. Contagens de verificações, tempos e descrições de ferramentas nas revisões antigas representam aquele momento. A seção final registra a revisão textual atual; scripts auxiliares, apresentação e Word não fazem parte da entrega atual.


**Moeda adotada:** dólar americano (USD), por premissa do projeto baseada no contexto de Seattle. Não houve conversão cambial nem alteração dos valores. A fonte não confirmou explicitamente a moeda. Preços e erros monetários estão em USD; percentuais, R² e contribuições SHAP em log mantêm suas próprias escalas.

## Objetivo e conclusão

Revisar dados, código, avaliação e comunicação da entrega existente. Auditoria realizada em 08/09/2026, sem nova busca de modelos, alteração dos hiperparâmetros ou treinamento com o teste.

**O split, as métricas e as previsões salvas são coerentes com o código revisado. Não foi encontrado uso direto do alvo nas features nem ajuste das estatísticas do pipeline com o teste. Entretanto, não é possível certificar ausência total de data leakage ou de overfitting.** As principais ressalvas são a demografia sem data de referência confirmada, a exploração anterior de todo o histórico e a diferença entre erros de treino, validação e teste.

Foram aprovadas **131 verificações automáticas**, registradas em [auditoria_projeto.json](../reports/metrics/auditoria_projeto.json). Os notebooks 01, 02, 03 e 06 também executaram em kernels novos, sobre cópias isoladas dos arquivos. O 06 carregou os modelos já avaliados. Os treinamentos de 04 e 05 não foram reexecutados nesta auditoria; seus códigos, manifestos, previsões e métricas foram revisados. [Registro de execução](../reports/metrics/auditoria_execucao_notebooks.json).

## Achados por prioridade

| Prioridade | Achado | Evidência e consequência | Encaminhamento |
|---|---|---|---|
| Alta | Disponibilidade temporal da demografia não comprovada | O modelo usa 25 indicadores cuja fonte e período não estão confirmados. Podem descrever informação posterior às vendas. | Confirmar fonte, data de referência e data de publicação antes de defender uma simulação histórica de produção. Excluir `hous_val_amt` não resolve a pendência nas demais colunas. |
| Alta | Teste visto na exploração | EDA e EDA pós-merge observaram o histórico completo antes do split. Decisões humanas podem ter sido influenciadas por esse conhecimento. | Declarar a limitação; não chamar o teste de completamente intocado. Para nova versão, reservar dados posteriores antes da exploração orientada à modelagem. |
| Alta | Subestimação e erro elevado em parte dos imóveis | Viés médio −45.704,70; MAE 278.380,19 acima de 1 milhão. | Definir uso e tolerância com o negócio. Não apresentar o modelo como apto à precificação automática. |
| Média | Ausência de overfitting não demonstrada | MAE de treino 41.185,22; CV 62.249,43; teste 74.089,41. | Reportar a diferença e investigar em uma nova rodada de desenvolvimento. Não usar o teste já examinado para escolher correções. |
| Média | Interpretação dos atributos incompleta | `grade`, indicadores demográficos e sufixo `15` têm significados inferidos. Há 12 anos de construção e seis anos de reforma posteriores ao ano da venda. | Confirmar definições e a cronologia com o fornecedor. Não corrigir por intuição nem afirmar que esses campos estavam disponíveis no momento da previsão. |
| Média | Reexecução integral pode interromper o cache | O 06 confere hashes completos dos registros de 04 e 05, que incluem informações de execução. Mesmo parâmetros iguais podem gerar hashes diferentes após novo treinamento. | Preservar os registros avaliados e usar cópia separada para experimentos. Documentado no README; não removida a proteção de integridade. |
| Baixa — corrigida | Comunicação desatualizada | O plano ainda dizia que o teste não havia sido avaliado; duração de seleção citava cerca de 36 s, enquanto a configuração registra 30,5 s. | Corrigidos os textos e distinguido o estado ao encerrar 05 da avaliação posterior em 06. |
| Baixa — corrigida | Rótulos incompletos | Alguns gráficos não explicitavam unidade de preço/área; título da permutação sugeria frequência de uso. | Incluídas unidades e alterado para impacto no erro. Figuras e saídas correspondentes atualizadas. |

## Dados, merge e leakage

A auditoria reproduziu os merges a partir dos CSVs originais e comparou todas as colunas aos arquivos exportados. A tabela demográfica possui CEP único; a junção é à esquerda, muitos-para-um. As 21.613 vendas e os 100 exemplos futuros permanecem separados, com ordem e valores originais preservados. A simulação de CEP desconhecido não aparece nas exportações.

As sete derivadas foram recalculadas independentemente e coincidem com os arquivos preparados. Usam somente atributos da própria linha; as divisões inválidas ficam ausentes. Não foram usadas médias de preço por CEP, target encoding ou o preço das correspondências históricas dos futuros. `price`, `id` e `date` estão fora das features do modelo; `hous_val_amt` também ficou fora da configuração selecionada.

As medianas do imputador do LightGBM salvo coincidem com as calculadas somente nas 17.191 linhas de treino, e o conjunto de categorias do codificador coincide com os CEPs desse treino. No código de CV, cada janela constrói seu próprio pipeline antes de `fit`. O uso de pipelines com estatísticas aprendidas somente no treino é consistente com a [orientação do scikit-learn sobre leakage](https://scikit-learn.org/stable/common_pitfalls.html).

Esses controles não comprovam a disponibilidade histórica das próprias fontes. Uma junção correta pode acrescentar informação coletada depois da venda. Além da demografia, os anos posteriores à venda precisam de esclarecimento. Não foi comprovado que sejam vazamento, mas tampouco foi descartado esse risco.

## Split e validação cruzada

O corte em 10/03/2015 foi reproduzido pela posição de aproximadamente 80% das datas ordenadas. Todas as vendas a partir desse dia compõem o teste, sem dividir um dia entre treino e teste. O percentual não fica exatamente 80/20 porque as datas são preservadas e há exclusão de revendas antigas.

| Partição | Registros | Período |
|---|---:|---|
| Treino e validação | 17.191 | 02/05/2014 a 09/03/2015 |
| Teste final | 4.331 | 10/03/2015 a 27/05/2015 |
| Vendas antigas excluídas por ID do teste | 91 | Antes do corte |

| Janela | Treino | Validação | Período de validação |
|---|---:|---:|---|
| 1 | 11.241 | 2.392 | 26/10/2014 a 09/12/2014 |
| 2 | 13.626 | 1.626 | 10/12/2014 a 23/01/2015 |
| 3 | 15.235 | 1.917 | 24/01/2015 a 09/03/2015 |

As janelas foram reconstruídas pelo calendário: 45 dias cada, não 45 linhas. Treino sempre precede validação e não compartilha IDs com ela. Todas as linhas de CV estão na parcela anterior ao teste. É esperado que uma venda usada para validar uma janela possa integrar o treino de uma janela posterior: nesse momento ela já pertence ao passado.

O protocolo avalia vendas posteriores de imóveis cujos IDs não foram usados no ajuste. Não avalia uma região desconhecida: todos os CEPs do teste já aparecem no treino. Também não simula atraso de publicação dos dados; esse atraso precisa ser conhecido para decidir se um intervalo entre treino e validação é necessário.

## Métricas e contexto de negócio

MAE, RMSE, R², erro absoluto mediano, percentil 90, erro percentual mediano, viés e proporções dentro de ±10%/±20% foram recalculados a partir das previsões salvas. Os valores coincidem. As métricas são calculadas após a transformação inversa do alvo, na escala original do preço (USD). Não houve corte de previsões negativas do Ridge para melhorar o resultado: suas 13 previsões não positivas permanecem contabilizadas.

O MAE da CV é a média simples dos erros das três janelas, dando peso igual aos períodos. Não é o MAE de todas as vendas de validação agrupadas, pois os tamanhos das janelas diferem. RMSE e R² de seleção também são médias por janela. Os 4.331 registros de teste recebem uma avaliação única.

| Medida | Valor | Interpretação |
|---|---:|---|
| Preço médio no teste | 554.588,88 | Referência da escala dos preços nesse período |
| Preço mediano no teste | 465.000,00 | Metade das vendas fica abaixo desse preço |
| MAE do LightGBM | 74.089,41 | Tamanho médio do erro; não é erro máximo |
| MAE / preço médio | 13,36% | Razão entre duas médias; não é MAPE |
| Erro percentual mediano | 10,03% | Metade dos erros relativos por imóvel fica abaixo desse valor |
| Dentro de ±20% | 80,56% | Frequência observada, não garantia para a próxima casa |
| Viés médio | −45.704,70 | Subestimação em média |

**Esse MAE é bom ou ruim?** Há melhora de 27,48% frente ao Ridge no mesmo teste, mas isso não define aceitação de negócio. Para apoio com revisão humana, pode ser uma candidata a avaliação operacional; para fixar preços automaticamente, os erros e a subestimação exigem critérios muito mais claros. Não foi comprovado benefício financeiro, redução de perdas ou tolerância aceitável. A pergunta pendente é: qual decisão usará o preço e quanto custa errar para cima ou para baixo?

Adotamos USD como premissa pelo contexto de Seattle, sem conversão; a fonte não confirmou a moeda. Também não interpretamos R² de 0,8768 como 87,68% de acerto. A comparação final entre LightGBM e Ridge envolve diferentes entradas e parâmetros, além do algoritmo.

## Overfitting e seleção

O modelo salvo teve MAE de 41.185,22 nas próprias vendas usadas no ajuste final. Esse é um diagnóstico dentro da amostra, não evidência de generalização. O MAE médio das três validações foi 62.249,43 e o teste foi 74.089,41, 19,02% maior que a CV.

A diferença é compatível com alguma perda de desempenho fora do treino. Porém, não identifica isoladamente sobreajuste: composição dos imóveis, mudança de período e tamanhos de treino também diferem. Não foram produzidas curvas de aprendizado ou intervalos de incerteza capazes de sustentar uma conclusão mais forte.

Há controles de complexidade, como profundidade, quantidade de folhas, mínimo de amostras e regularização, mas eles não garantem ausência de overfitting. Optuna e seleção de features reutilizam a mesma CV; o melhor resultado pode se adaptar a essas janelas. A busca foi pequena, não uma comparação exaustiva de todos os algoritmos otimizados.

Não foi demonstrado que cada variável criada melhora a previsão. As derivadas isoladas pioraram ligeiramente os modelos comparados; a demografia ajudou quando adicionada ao grupo com derivadas. O conjunto físico mais demografia **sem derivadas** não foi testado. Esse é um experimento futuro legítimo, sem escolher o vencedor pelo teste já visto. A margem de 1% da regra de simplicidade é uma escolha operacional, não um teste estatístico de equivalência.

## SHAP, importância e sentido de negócio

| Feature | Aumento médio do MAE ao embaralhar | Leitura plausível, não causal |
|---|---:|---|
| `lat` | 50.113,32 | Localização diferencia mercados locais |
| `sqft_living` | 42.333,13 | Tamanho habitável ajuda a diferenciar imóveis |
| `grade` | 39.691,55 | Possível padrão construtivo; definição ainda a confirmar |
| `per_bchlr` | 18.932,02 | Possível indicador educacional regional, podendo representar diferenças entre bairros |
| `per_prfsnl` | 17.967,15 | Outra característica agregada da região, não dos compradores individuais |

A permutação mede dependência preditiva do modelo em uma amostra aleatória de 600 vendas, com três repetições. Não mede número de vezes que uma coluna foi usada nas árvores, direção do efeito ou importância causal. As primeiras três posições fazem sentido como associações imobiliárias; as interpretações demográficas precisam do dicionário e não autorizam inferências sobre uma pessoa.

Correlações entre colunas podem dividir importância; embaralhar uma área mantendo seu log e suas razões cria combinações impossíveis. A interpretação deve respeitar essas limitações, conforme a [documentação de importância por permutação](https://scikit-learn.org/stable/modules/permutation_importance.html). Permutação por grupos consistentes e estabilidade em outras amostras são melhorias futuras, não validações realizadas.

O SHAP foi calculado pelo mecanismo nativo `pred_contrib=True` do LightGBM para três exemplos escolhidos pelos percentis 10, 50 e 90 do preço previsto. A auditoria recalculou as contribuições e confirmou que `expm1(valor_base + soma_contribuições)` reproduz cada preço. Os valores estão na escala logarítmica e não podem ser somados diretamente como valores monetários. Esses três exemplos não permitem declarar um ranking SHAP global.

## Código, erros e reprodução

Pontos verificados: funções para preparação e avaliação; ordem explícita das features; `merge(validate="many_to_one")`; bloqueio de colunas obrigatórias ausentes; tratamento de divisões por zero; rejeição de valores infinitos; imputação dentro do treino; categorias desconhecidas tratadas pelo codificador; limite de threads; falhas explícitas de memória/tempo na seleção; hashes antes de reaproveitar a avaliação. A sintaxe de todas as células de código foi validada, incluindo comandos do IPython.

Há limitações de manutenção: funções repetidas entre notebooks 03/06 e 05/06, dependências de variáveis globais e ordem de execução. As verificações operacionais dos notebooks 01 a 06 usam `if` com `ValueError`, inclusive para integridade das partições, colunas e artefatos. Foram mantidos `assert` nos testes de casos simulados; esses testes devem ser executados sem `-O`. As funções `pd.testing.assert_frame_equal` e `np.testing.assert_allclose` foram preservadas: são chamadas de funções, não instruções `assert` removidas pelo Python. Uma API ainda precisaria de esquema próprio de validação. Não foi implementada API, portanto não há testes HTTP, Docker ou deploy a declarar aprovados.

Uma categoria desconhecida tecnicamente aceita pelo encoder não implica previsão confiável. Com `drop="first"` e `handle_unknown="ignore"`, ela não recebe uma categoria nova aprendida. O contrato documental propõe bloquear CEP não suportado; esse comportamento ainda não existe como endpoint.

Reexecutar 04/05 pode modificar hashes de registros mesmo mantendo parâmetros, devido a tempos e recursos registrados. A interrupção do cache de 06 nessas condições é uma proteção conservadora, não prova de erro nos preços. Separar assinatura semântica do experimento e informações voláteis é uma melhoria de engenharia futura.

## Gráficos, narrativa e correções

Foram revisados os comandos de geração e as figuras principais de EDA, seleção, erros, segmentos e explicabilidade. Acrescentadas unidades nos rótulos de preços/MAE e áreas, mantendo a ressalva sobre `sqft`. O ranking por permutação agora descreve impacto no erro. Os gráficos não foram usados para remover extremos nem esconder previsões negativas.

Nos histogramas logarítmicos, o eixo mostra `log(1 + valor)`, não moeda ou área direta. O boxplot do notebook 02 representa medianas por CEP, não vendas individuais. Barras de volume representam vendas registradas no dataset, não demanda total; os meses de borda têm cobertura parcial. As tabelas de segmentos informam contagens para limitar conclusões sobre grupos pequenos.

O README foi reorganizado em problema, dados, abordagem, decisões, resultados, limitações, entregas e reprodução. O enunciado anterior foi preservado em arquivo próprio. Plano e resumo da seleção foram sincronizados com o registro atual; o resumo do merge foi corrigido para seis figuras. Acrescentado contexto de preço e diagnóstico de treino à documentação de avaliação e comunicação.

## O que permanece pendente

Confirmar significado, unidade e disponibilidade temporal dos dados; definir uso e tolerância de erro; obter uma avaliação posterior independente; investigar subestimação e complexidade sem escolher pelo teste atual. Esses pontos dependem de informação adicional ou de uma nova rodada experimental, não de corrigir textos para declarar o modelo aprovado.

O projeto atende ao escopo de modelagem e documentação da arquitetura. API, Docker, MLflow, LLM e monitoramento permanecem propostas; seus comportamentos em produção não foram testados. Os resultados atuais devem ser apresentados como evidência histórica com limitações explícitas.


### MAPE: erro percentual médio como complemento

**Resultado no teste:** LightGBM teve **MAPE de 12,77%**, contra **20,17% do Ridge**, nas mesmas 4.331 vendas. Para cada imóvel, dividimos o erro absoluto pelo preço real, multiplicamos por 100 e depois calculamos a média. Isso expressa o tamanho médio do erro relativo; não significa 87,23% de acerto nem uma garantia para cada previsão.

**Por que incluir:** o MAE de USD 74.089,41 informa a escala monetária; o MAPE facilita comparar erros relativos entre preços diferentes. Ele também é diferente dos **10,03% de erro percentual mediano** e dos **13,36% da razão entre MAE e preço médio**.

**Cuidados e decisão:** o mesmo erro em USD pesa mais em imóveis baratos. Preços próximos de zero podem distorcer o MAPE; no teste, o menor preço é USD 81.000. Mantemos **MAE como critério principal de seleção**. MAPE foi acrescentado às previsões existentes, sem novos treinamentos ou escolha pelo teste; não há tolerância de negócio aprovada.

Na validação do candidato final, a média do MAPE das três janelas é **11,97%**. Agrupando todas as vendas de validação, é **11,92%**: as quantidades de vendas por janela diferem. Para comparar períodos com peso igual, usamos o primeiro valor.

**Leitura por faixa:** o MAPE foi **15,08%** até USD 300 mil; **11,46%** de USD 300 a 600 mil; **12,29%** de USD 600 mil a 1 milhão; e **17,39%** acima de USD 1 milhão. O erro relativo também é maior no grupo mais caro. As faixas são definidas pelo preço real e servem à avaliação retrospectiva, não para classificar automaticamente o risco de uma previsão futura.




### Correção da assinatura no notebook 06

Após reexecução de 04/05, os hashes integrais da configuração e do protocolo diferiram do registro original. As fontes e os três artefatos avaliados mantiveram seus hashes. Conferimos que os pipelines clonados salvos correspondem aos pipelines construídos com os parâmetros atuais, incluindo pré-processamento, e reproduzimos as previsões de ambos no teste sem treinamento.

O registro recebeu uma assinatura do conteúdo, desconsiderando somente os blocos de medições `orcamento` e `resources`. A assinatura integral anterior foi preservada, com anotação da migração. Essa correção substitui a limitação de cache por medições voláteis descrita anteriormente. Não recuperamos os JSONs anteriores para comparar campo a campo; a migração foi sustentada pelos artefatos conferidos. Mudanças futuras no restante do conteúdo continuam bloqueadas. Modelos, preços previstos e métricas não foram alterados.


### Substituição das verificações críticas

Foram substituídas 102 instruções `assert` por condições equivalentes com `ValueError` nos notebooks 01 a 06 (6, 12, 13, 19, 27 e 25 substituições, respectivamente). Oito instruções foram mantidas nos testes de casos simulados dos notebooks 02 e 03. As condições originais foram preservadas e as mensagens ausentes foram acrescentadas.

Validação da alteração: todas as células passaram na compilação, incluindo sintaxe do IPython. As células de leitura, partições e reaproveitamento do notebook 06 foram executadas com otimização nível 2 e chamadas de treinamento bloqueadas. O cache foi reaproveitado. Casos deliberadamente inválidos de sobreposição entre treino e teste, coluna ausente e nomes duplicados levantaram `ValueError` mesmo com otimização. Não houve reexecução completa dos notebooks ou novos treinamentos nesta revisão; modelos, previsões e métricas foram preservados.


### Organização das validações nos notebooks

A revisão seguinte substitui a estratégia de converter cada conferência em uma exceção. Foram retiradas 21 conferências redundantes, incluindo quantidades fixas nas exportações, verificações de cópias e repetição da exclusão do teste dentro de cada treinamento. A separação do teste continua verificada ao construir ou carregar as partições.

Na junção, uma comparação das colunas originais cobre valores, quantidade e ordem das linhas, junto do `validate="many_to_one"` do pandas. As exportações de 02 e 03 usam as tabelas já preparadas sem reler cada CSV imediatamente. As mensagens das análises foram atualizadas para refletir essa mudança. Foram simplificadas negações duplas e separadas as definições de preparação do reaproveitamento dos resultados nos notebooks 05 e 06.

Permanecem as verificações de entradas, contratos, alinhamento com o alvo, separação temporal e de imóveis, assinatura da avaliação, integridade dos artefatos e validade das previsões. Testes de casos simulados continuam em suas seções específicas. Não foi acrescentado um mecanismo genérico de validação nem tratamento que esconda falhas.

Validação: compilação de todas as células; execução completa de 01, 02, 03 e 06 em cópias temporárias dos dados e artefatos; execução da preparação, partições e reaproveitamento do baseline de 04/05 sem os treinamentos. Chamadas de `fit` foram bloqueadas durante a revisão. Os casos simulados foram executados normalmente; as demais células foram compiladas com otimização nível 2. Nenhum modelo foi retreinado e os resultados da entrega não foram sobrescritos.


## Revisão textual da entrega atual

**Escopo:** README, plano, índice dos notebooks, resumos técnicos, dicionário, documentos de API/deploy e células Markdown dos notebooks 01 a 09. O enunciado original e os arquivos em `reports/archive/` foram preservados como fontes/histórico. PDFs antigos e suas prévias não foram regenerados nesta revisão; não devem ser usados como referência das últimas alterações textuais.

**Inconsistências corrigidas:**

- Plano com etapas 05/06 ainda descritas como planejadas e documentos descritos como futuros. O estado final agora distingue análises executadas de operação apenas documentada.
- Descrição de feature engineering como pipeline completo, embora a função por linha ainda seja externa ao joblib.
- Referências à releitura de CSVs removida na organização do código.
- Promessas de tratamento de outliers em 03/05: razões e logaritmos foram comparados, mas corte/remoção por limites não foi executado.
- Descrição de matrizes que sugeria Random Forest esparsa; os três modelos de árvores usam matrizes densas, e Ridge usa esparsa.
- Tempos e memória diferentes entre os textos: a referência atual é o JSON de cada experimento (seleção: 30,53 segundos e 239,21 MiB amostrados; baseline: 0,96 segundo e 228,36 MiB). Não são limites nem medidas de pico.
- Ambiguidade nas unidades: R² não tem unidade; SHAP está em log(1 + preço), e a reconstrução monetária requer expm1 após somar as contribuições e o valor base.
- Métricas de teste no resumo de seleção substituídas pela distinção entre CV e avaliação posterior. MAPE foi incluído nas tabelas principais de avaliação e comunicação.
- Descrição de `hous_val_amt` como exclusão antecipada: o grupo foi comparado no 05, e a escolha segue a regra de MAE e simplicidade. A dúvida temporal não está resolvida.
- Frases de exemplo da LLM no passado, agora condicionais e explicitamente hipotéticas; nenhuma API de LLM foi executada.
- Índice com descrição antiga de diagramas Mermaid, agora substituída pela informação das imagens incorporadas.

**Verificações realizadas nesta revisão:** métricas recalculadas das previsões salvas para Ridge e LightGBM; hashes de fontes e artefatos avaliados conferidos; separação externa de datas/IDs e ausência de teste nas janelas verificadas; soma SHAP convertida por expm1 comparada às previsões; características e ordem dos 100 futuros conferidas com o CSV original; sintaxe das células compilada; links locais e anexos dos notebooks conferidos sem destinos ausentes. Não houve treinamento nem alteração de preços previstos.

**Conclusão sustentada:** MAE de teste de USD 74.089,41, MAPE de 12,77% e redução do MAE de 27,48% frente ao Ridge continuam consistentes com os arquivos numéricos. Persistem as limitações de EDA no histórico completo, disponibilidade temporal da demografia, seleção e ajuste nas mesmas janelas e subestimação de imóveis caros. A revisão não certifica ausência total de leakage, overfitting ou falhas em produção.


### Atualização posterior do relatório executivo em PDF

A pedido do autor, `reports/relatorio_executivo.pdf` foi atualizado após a revisão textual, substituindo a versão anterior no mesmo caminho. O documento mantém oito páginas e incorpora os resultados salvos, as limitações, o papel apenas documental de API/Docker/LLM/MLflow e o esquema de deploy simplificado. As prévias em `reports/pdf_preview/` foram atualizadas. A estrutura do PDF e a paginação foram conferidas, com inspeção visual das prévias de resultados e deploy. Não houve treinamento nem alteração das métricas. Esta atualização substitui a ressalva anterior sobre esse PDF não ter sido regenerado.
