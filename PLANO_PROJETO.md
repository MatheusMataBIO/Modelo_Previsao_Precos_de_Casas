# Plano de execução — Previsão de preços de casas

**Moeda adotada:** dólar americano (USD), por premissa do projeto baseada no contexto de Seattle. Não houve conversão cambial nem alteração dos valores. A fonte não confirmou explicitamente a moeda. Preços e erros monetários estão em USD; percentuais, R² e contribuições SHAP em log mantêm suas próprias escalas.

## Objetivo e escopo

Construir uma solução em Python para prever `price` a partir das características dos imóveis e dos dados demográficos por CEP. Demonstrar estruturação do problema, decisões técnicas, qualidade do código, integração de componentes, conteinerização e observabilidade, além de comunicar os resultados para negócio.

O problema é de **regressão**: o alvo é um valor contínuo. Por isso, o baseline será regressão linear, complementada por Ridge. Regressão logística seria adequada se o alvo fosse uma classe, como “imóvel acima de um limite de preço”, o que mudaria o objetivo do desafio.

Este documento reúne o plano de trabalho e o estado final da entrega. Os itens de planejamento abaixo indicam a intenção de cada etapa; os parágrafos de execução e os resumos registram o que foi efetivamente realizado. As etapas 1 e 2 estão implementadas e executadas em `notebooks/01_eda.ipynb` e `notebooks/02_merge_eda_complementar.ipynb`. Resultados e pendências estão em `docs/eda_summary.md` e `docs/merge_eda_summary.md`. A etapa 3 também foi executada em `notebooks/03_feature_engineering.ipynb`, com resumo em `docs/feature_engineering_summary.md`: criação de candidatas por linha e contrato das entradas. Os ajustes aprendidos do pipeline ficam para o treino após o split. A etapa 4 também foi executada em `notebooks/04_split_baseline.ipynb`, com três janelas temporais, nove ajustes de baseline e relatório em `docs/baseline_summary.md`. As etapas 5 e 6 também estão executadas. As etapas 7 e 8 estão documentadas, sem implementação de API, Docker ou infraestrutura. A etapa 9 está concluída com apresentação em Markdown e resumo executivo.

| Etapa | Implementar e executar localmente | Documentar |
|---|---|---|
| 1. EDA | Diagnóstico e análise exploratória | Dicionário, hipóteses e limitações |
| 2. Merge | Junção e validações de integridade | Regra de enriquecimento e ausências |
| 3. Feature engineering | Função de criação de candidatas por linha | Justificativa e disponibilidade das variáveis |
| 4. Split e baseline | Partições, validação e baselines | Protocolo de avaliação |
| 5. Seleção do modelo | Comparação e ajuste dos candidatos | Critérios e decisão final |
| 6. Avaliação e explicabilidade | Teste final, explicações e previsões futuras | Resultados e limites de uso |
| 7. Empacotamento e API | Não implementar nesta entrega | Organização proposta, contrato, Docker e verificações futuras |
| 8. Deploy, monitoramento e versionamento | Sem provisionamento ou automação | Arquitetura, publicação, monitoramento, versionamento, reentreinamento e reversão |
| 9. Comunicação | Relatório, gráficos e README de execução | Síntese executiva e recomendações |

Conforme o README e o escopo acordado, empacotamento, API, Docker, deploy e aprendizado contínuo serão somente documentados. Não há serviço ou contêiner executável nesta entrega. Os modelos e as previsões locais já foram produzidos nos notebooks 01 a 06. A publicação do repositório permanece uma etapa posterior.

## 1. EDA — Análise exploratória dos dados

**Objetivo:** entender qualidade, significado e comportamento dos dados antes de definir tratamentos e modelos.

- Inspecionar os três arquivos: dimensões, colunas, tipos, valores ausentes, duplicatas e valores inválidos.
- Construir um dicionário das principais variáveis, explicitando unidades conhecidas e significados ainda não confirmados.
- Verificar o intervalo de `date`, repetições de `id` e possíveis revendas do mesmo imóvel.
- Comparar os esquemas de treino e inferência: o arquivo futuro não possui `price`, `id` nem `date`.
- Explorar no histórico completo a distribuição de preços, assimetria, correlações, localização, área, padrão construtivo e conservação.
- Investigar extremos de preço, área, quartos e banheiros; distinguir erros de registros plausíveis. Não excluir imóveis caros apenas por serem extremos.
- Examinar qualidade e cobertura dos dados demográficos, sem presumir a origem ou a data de referência desses indicadores.

**Entregáveis:** notebook de EDA, dicionário de dados, gráficos e registro das hipóteses e problemas encontrados.

**Critério de conclusão:** cada tratamento proposto possui uma justificativa, e as incertezas sobre os dados estão registradas.

## 2. Merge — Integração dos dados

**Objetivo:** enriquecer cada imóvel com informações de seu CEP sem perder registros ou multiplicar linhas.

- Padronizar `zipcode` como identificador textual nos três arquivos, preservando zeros à esquerda quando aplicável.
- Validar a unicidade de `zipcode` na tabela demográfica. Investigar duplicatas antes de qualquer agregação ou descarte.
- Realizar uma junção à esquerda dos imóveis com os dados demográficos, validando a relação muitos-para-um.
- Conferir quantidade e ordem dos registros antes e depois da junção, além da proporção de CEPs sem correspondência.
- Definir uma política para CEP ausente ou desconhecido: manter o imóvel, sinalizar a ausência do enriquecimento e tratar os campos faltantes no pipeline.
- Aplicar a mesma função de enriquecimento no treinamento, na previsão em lote e na API.
- Investigar `hous_val_amt`: significado, origem e disponibilidade no momento da previsão. Se isso não puder ser estabelecido, excluí-la do modelo principal e registrar a justificativa.

**Entregáveis:** função de enriquecimento reutilizável, relatório de integridade da junção e EDA pós-merge. Explorar relações entre indicadores regionais e preços, distinguindo análise por venda de análise com um peso igual por CEP.

**Critério de conclusão:** a junção preserva os imóveis e funciona para dados futuros e CEPs desconhecidos.

## 3. Feature engineering — Preparação das variáveis

**Objetivo:** produzir entradas úteis e reproduzíveis, disponíveis também no momento da inferência.

- Excluir `id` como preditor; mantê-lo como apoio à auditoria e ao controle de imóveis repetidos.
- Usar `date` para estudar a estratégia de validação. Inicialmente, não utilizá-la como entrada do modelo, pois ela não está presente no arquivo futuro.
- Tratar variáveis numéricas e categóricas separadamente; avaliar imputação, codificação de CEP e padronização para os modelos que precisarem dela.
- Avaliar indicadores como presença de porão, ocorrência de reforma e proporção entre áreas, com proteção contra divisões por zero.
- Não calcular idade do imóvel com a data atual do sistema. Uma feature de idade exigiria uma data de referência explícita e disponível na inferência; sem isso, utilizar o ano de construção.
- Avaliar transformação logarítmica de variáveis assimétricas e de `price`, comparando as previsões após retornar à escala monetária original.
- Verificar redundância entre contagens e percentuais demográficos e entre áreas relacionadas.
- Não criar features que dependam do preço observado do próprio imóvel, como preço por área. Essa medida pode ser usada em análise, mas não como entrada de inferência.
- Encapsular as transformações em pipeline. Imputação, escalonamento, seleção de variáveis e qualquer estatística aprendida serão ajustados apenas no treino de cada partição.

**Entregáveis:** função de criação de candidatas por linha, arquivos preparados e contrato das features com justificativas. A integração de todo o preparo em um pacote é documental na etapa 07.

**Critério de conclusão:** treinamento e inferência usam as mesmas regras, inclusive para valores ausentes e categorias novas.

## 4. Split e regressão linear como baseline

**Objetivo:** estabelecer uma avaliação confiável e uma referência simples de desempenho.

O fluxo será EDA geral → merge → EDA pós-merge → preparação das variáveis → split e modelagem. A separação de treino, validação e teste ocorre nesta etapa 4. A EDA inicial considera todo o histórico; essa escolha será documentada na avaliação. Transformações que aprendem estatísticas são ajustadas apenas no treino de cada partição.

- Avaliar uma separação temporal, treinando em vendas anteriores e testando em vendas posteriores, conforme a cobertura de datas permitir.
- Verificar sobreposição de `id` entre partições e documentar a política para revendas. Quando necessário, expurgar do treino os imóveis presentes no teste para avaliar imóveis ainda não vistos.
- Caso a estrutura dos dados não sustente validação temporal, justificar uma separação por grupos de imóvel. Não escolher automaticamente um split aleatório por linha.
- Reservar um teste final para a avaliação, reconhecendo que a EDA descritiva já examinou o histórico completo. Não usar o teste para ajustar modelos. No conjunto de desenvolvimento, adotar validação coerente com o split escolhido: janelas temporais ou partições por grupo.
- Definir sementes quando houver aleatoriedade e registrar as regras e os identificadores das partições.
- Treinar uma previsão constante pela mediana como referência mínima e regressão linear como baseline principal; avaliar Ridge como variante regularizada.
- Usar **MAE** como métrica principal, por expressar o erro absoluto médio na unidade de preço; complementar com **RMSE**, mais sensível a erros grandes, e **R²**, como medida adicional de ajuste.
- Não estabelecer uma meta numérica arbitrária antes de medir o baseline e discutir o contexto de uso.

**Entregáveis:** partições reproduzíveis, protocolo de avaliação e tabela dos baselines na validação.

**Critério de conclusão:** todos os candidatos poderão ser comparados nas mesmas partições, sem uso do teste para tomar decisões.

## 5. Seleção e escolha do melhor modelo

**Objetivo:** verificar se modelos mais flexíveis entregam ganho que justifique sua complexidade.

- Comparar Ridge, Random Forest, XGBoost e LightGBM, usando as interfaces de regressão, reaproveitando a referência do notebook 04.
- Manter o mesmo conjunto de partições e avaliar métricas na escala original de preço.
- Ajustar hiperparâmetros apenas nas janelas de treino e validação: Optuna no candidato selecionado, com até oito tentativas e 90 segundos de busca, um experimento por vez e até duas threads. Os limites de tempo são verificados entre ajustes, sem interromper um treinamento em andamento.
- Comparar o desempenho com e sem dados demográficos para medir o valor do enriquecimento.
- Registrar métricas por partição, dispersão, hiperparâmetros, features, tempo de treinamento e custo de inferência local.
- Priorizar MAE de validação, considerando também estabilidade, erros por segmento, simplicidade e facilidade de operação.
- Selecionar e congelar a configuração antes de abrir o teste final.

**Entregáveis:** tabela comparativa de experimentos e justificativa da escolha.

**Critério de conclusão:** a escolha decorre de evidências; o baseline poderá ser mantido caso os modelos mais complexos não tragam ganho consistente.

**Execução da etapa 05:** concluída em `notebooks/05_selecao_modelo.ipynb`, com Ridge, Random Forest, XGBoost e LightGBM. Selecionado LightGBM com 50 entradas e log do preço; MAE médio de CV de 62.249,43. Foram 52 ajustes novos e três reaproveitados em cerca de 30,53 segundos no registro salvo. Tempos e memória variam entre execuções. Configuração v2 salva em `artifacts/configuracao_modelo_selecionado.json`; resultados e limitações em `docs/model_selection_summary.md`. Ao encerrar a etapa 05, o teste ainda não havia sido avaliado; sua avaliação posterior está registrada na etapa 06.

## 6. Avaliação final e explicabilidade

**Objetivo:** medir a generalização da solução escolhida e explicar seus acertos e limitações.

- Avaliar o pipeline selecionado uma única vez no teste reservado, reportando MAE, MAPE, RMSE e R².
- Comparar com o baseline congelado no mesmo teste, sem usar essa comparação para iniciar novos ajustes no teste.
- Produzir gráficos de preço real versus previsto, resíduos e distribuição dos erros.
- Analisar erros por faixas de preço, região e características relevantes, informando o número de exemplos por grupo.
- Apresentar importância global por permutação e, se adequado ao modelo e ao custo, explicações SHAP de exemplos individuais.
- Explicar que importância preditiva não demonstra causalidade e que variáveis correlacionadas podem dividir importância.
- Gerar previsões para `future_unseen_examples.csv`, preservando a ordem e adicionando `predicted_price` e uma referência de linha.
- Identificar previsões inválidas ou não finitas e registrar comportamentos incompatíveis com o domínio antes de concluir a entrega.
- Preservar o modelo efetivamente avaliado para as previsões futuras. Não houve reajuste com o teste; um novo treinamento seria uma versão distinta, com avaliação própria.
- Não atribuir métricas de acurácia ao arquivo futuro, pois ele não contém o preço real.

**Entregáveis:** relatório final de avaliação, gráficos de explicabilidade, CSV de previsões e ficha do modelo com dados, métricas, versão e limitações.

**Critério de conclusão:** é possível identificar qual artefato foi avaliado e qual gerou as previsões entregues.

**Execução da etapa 06:** concluída em `notebooks/06_avaliacao_explicabilidade.ipynb`. LightGBM: MAE de teste 74.089,41, contra 102.170,13 do Ridge. Foram avaliadas 4.331 vendas, sem ajuste de parâmetros no teste. Salvos modelos, explicabilidade, erros por segmento e 100 previsões futuras. Ver `docs/evaluation_summary.md` e `docs/model_card.md`.

## 7. Empacotamento e API de inferência — somente documentação

**Objetivo:** explicar como o fluxo validado poderia virar um pacote e uma API, sem implementá-los.

- Descrever os artefatos existentes e a organização futura do pacote.
- Definir entrada de 18 características, enriquecimento por CEP, sete derivadas e seleção das 50 entradas do modelo.
- Propor contrato, endpoints, resposta, validações e tratamento de erros, diferenciando capacidade do pipeline de política operacional.
- Explicar imagem, contêiner, Dockerfile, dependências e verificações de compatibilidade.
- Documentar como verificar equivalência com o notebook 06 e medir carga antes de operar.
- Descrever integração com API externa de LLM para explicação textual: contexto permitido, contrato, validação, limites de consumo e contingência; o regressor permanece responsável pelo preço.

**Entregáveis concluídos:** `notebooks/07_empacotamento_api.ipynb` e `docs/packaging_api.md`, com células somente Markdown e diagrama. Pacote, API e imagem não foram criados.

**Critério de conclusão:** a documentação permite entender o que seria implementado e como seria verificado, sem atribuir execução às propostas.

## 8. Estratégia de deploy, monitoramento e aprendizado contínuo — somente documentação

**Objetivo:** descrever como publicar, observar, atualizar e reverter a solução.

- Diagrama e responsabilidades das camadas, publicação em homologação e liberação controlada.
- Monitoramento de serviço, entradas e desempenho quando houver preços reais.
- Release que associa código, modelo, contrato, demografia e ambiente.
- Operação da API de LLM: credenciais, latência, custo, qualidade do texto, versões de prompt/modelo e desativação independente da previsão numérica.
- Associação das previsões às vendas, preservando datas de disponibilidade e histórico.
- Reentreinamento em lotes, avaliação com dados novos, critérios prévios de promoção e reversão do pacote completo.
- Pendências de dados e tolerâncias de negócio explicitadas; sem limiares operacionais inventados.

**Entregáveis concluídos:** `notebooks/08_deploy_monitoramento.ipynb` e `docs/deployment.md`. Infraestrutura, publicação e reentreinamento automático não foram executados.

**Critério de conclusão:** explicar o ciclo operacional e de atualização, incluindo falhas e responsabilidades, distinguindo artefatos locais de componentes futuros.

## 9. Comunicação para stakeholders

**Objetivo:** apresentar o valor e as limitações da solução de forma útil para decisões de negócio.

- Abrir com o problema, o público consumidor e o uso pretendido da estimativa de preço.
- Explicar o ganho frente ao baseline usando métricas observadas, sem inventar economia ou retorno financeiro.
- Traduzir MAE como erro absoluto médio em USD, conforme a premissa de moeda adotada no projeto. MAE não é garantia de erro máximo por imóvel.
- Mostrar poucos gráficos: real versus previsto, erros por faixa de preço ou região e principais fatores preditivos.
- Apresentar exemplos com boa previsão e com erro alto, explicando limitações de cobertura e dos dados históricos.
- Relatar o ganho observado dos dados demográficos e os casos em que a estimativa exige maior cuidado.
- Resumir o funcionamento da API, a estratégia de operação e os próximos passos com prioridade e justificativa.
- Organizar README final com reprodução dos notebooks, previsões em lote e links para resultados e propostas de API, Docker e deploy, identificadas como documentação.

**Entregáveis:** resumo executivo, relatório técnico e README reproduzível.

**Execução da etapa 09:** concluída em `notebooks/09_comunicacao_stakeholders.ipynb` e `docs/executive_summary.md`. Inclui dois gráficos, métricas em linguagem de negócio, exemplos reais selecionados por regra explícita e roteiro de apresentação. Não houve novo treinamento; as propostas de API, Docker, LLM e MLflow continuam identificadas como não implementadas.

**Critério de conclusão:** uma pessoa de negócio entende o resultado e seus limites, e uma pessoa técnica consegue reproduzir a entrega.

## Estrutura atual de arquivos

O desenvolvimento analítico é feito em Python com Jupyter Notebook. Os notebooks 01 a 06 contêm análises e modelagem executadas. Os notebooks 07 e 08 documentam a operação proposta; o 09 apresenta a comunicação dos resultados. Os CSVs originais estão em `data/raw/`, sem alterações no conteúdo. A EDA gerou resultados do histórico completo em `reports/`; a etapa 04 gerou manifestos ativos de split e validação cruzada em `data/processed/`. As dependências do ambiente estão fixadas em `requirements.txt`.

```text
.
├── README.md
├── PLANO_PROJETO.md
├── .gitignore
├── notebooks/
│   ├── README.md
│   ├── 01_eda.ipynb
│   ├── 02_merge_eda_complementar.ipynb
│   ├── 03_feature_engineering.ipynb
│   ├── 04_split_baseline.ipynb
│   ├── 05_selecao_modelo.ipynb
│   ├── 06_avaliacao_explicabilidade.ipynb
│   ├── 07_empacotamento_api.ipynb
│   ├── 08_deploy_monitoramento.ipynb
│   └── 09_comunicacao_stakeholders.ipynb
├── data/
│   ├── raw/
│   │   ├── kc_house_data.csv
│   │   ├── zipcode_demographics.csv
│   │   └── future_unseen_examples.csv
│   └── processed/
├── artifacts/
├── reports/
│   ├── figures/
│   ├── metrics/
│   └── predictions/
└── docs/
    ├── data_dictionary.md
    ├── model_card.md
    ├── deployment.md
    └── executive_summary.md
```

Os notebooks 07 a 09 organizam o planejamento operacional e a comunicação. Os documentos em `docs/` consolidam as análises e a documentação da entrega. Código de API, módulos de serviço e Dockerfile ficam fora da implementação desta entrega; sua organização e comportamento estão documentados.

## Sequência prática e pontos de decisão

1. Executar diagnóstico e EDA geral dos arquivos completos.
2. Implementar o merge validado e a EDA complementar das tabelas combinadas.
3. Preparar as variáveis; definir o split na modelagem e construir pipeline, baselines e validação reproduzível.
4. Comparar modelos, medir a contribuição demográfica e congelar a configuração escolhida.
5. Executar avaliação final, explicabilidade e previsão dos exemplos futuros.
6. Documentar o empacotamento, a API e o contêiner, incluindo como seriam verificados.
7. Documentar deploy, observabilidade, versionamento e aprendizado contínuo.
8. Consolidar comunicação, revisar a reprodução e preparar a entrega pública.

Modelo, conjunto de features e hiperparâmetros foram escolhidos na validação e registrados na etapa 05. Metas operacionais e tolerâncias de negócio permanecem pendentes; não foram deduzidas das métricas do teste.

