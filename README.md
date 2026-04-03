# Classificação de Sentimentos em Texto com KNIME

Pipeline de Text Mining com Naive Bayes e TF-IDF para classificação binária de sentimentos, desenvolvido como trabalho da disciplina COS738 - 2026/1 - Busca e Mineração de Texto no Programa de Engenharia de Sistemas e Computação (COPPE/UFRJ).

## Visão Geral

Este projeto implementa um pipeline completo de classificação de sentimentos utilizando o KNIME Analytics Platform (v5.11.0). A partir de avaliações textuais rotuladas provenientes de três domínios distintos (Amazon, IMDB e Yelp), o workflow cobre desde a ingestão e limpeza dos dados até a avaliação do modelo com métricas de acurácia, precisão, recall e F1.

## Dataset

O dataset utilizado é o [UCI Sentiment Labelled Sentences Dataset](https://archive.ics.uci.edu/ml/datasets/Sentiment+Labelled+Sentences), composto por três arquivos CSV sem cabeçalho:

| Arquivo | Fonte | Descrição |
|---|---|---|
| `amazon_cells_labelled.csv` | Amazon | Avaliações de produtos |
| `imdb_labelled.csv` | IMDB | Críticas de filmes |
| `yelp_labelled.csv` | Yelp | Avaliações de restaurantes |

Cada linha contém uma frase e um label binário: **1** (positivo) ou **0** (negativo). Após limpeza e remoção de linhas com labels inválidos (causados por delimitador misto no arquivo IMDB), o dataset consolidado ficou com **1.490 instâncias**, distribuídas de forma equilibrada (50,2% negativos / 49,8% positivos).

## Estrutura do Repositório

```
bmt261_knime_binary_classification/
├── BMT/
│   ├── BMT.knwf               # Arquivo exportável do workflow KNIME
│   ├── workflow.knime          # Configuração do workflow
│   ├── workflow.svg            # Diagrama visual do fluxo
│   └── <Nó (#ID)>/            # Dados e configurações de cada nó
├── data/
│   └── sentiment labelled sentences/
│       ├── amazon_cells_labelled.txt
│       ├── imdb_labelled.txt
│       ├── yelp_labelled.txt
│       └── csv/               # Versões em CSV dos datasets
├── report/
│   ├── report_knime_bmt.tex   # Fonte LaTeX do relatório
│   └── report_knime_bmt.pdf   # Relatório técnico completo
└── README.md
```

## Pipeline

O workflow KNIME é composto pelas seguintes etapas:

### 1. Leitura e Verificação dos Dados

Três nós **CSV Reader** carregam os arquivos com delimitador vírgula, sem cabeçalho e encoding UTF-8. Em seguida, **Column Filter** retém apenas as colunas de texto (`Column0`) e label (`Column1`), e **Missing Value** remove linhas nulas antes da concatenação. Os três fluxos são unidos pelo nó **Concatenate**.

A consistência dos dados foi verificada com **Data Explorer**, que identificou 531 linhas com labels inválidos no arquivo IMDB. Um nó **Rule Engine** mapeou os valores válidos e um **Row Filter** eliminou as linhas problemáticas.

### 2. Pré-processamento de Texto

Os nós de Text Processing do KNIME foram aplicados na seguinte ordem:

1. **Strings to Document** — converte o texto bruto para o tipo `Document` do KNIME, com tokenização via OpenNLP WhitespaceTokenizer
2. **Case Converter** — converte todo o texto para minúsculas
3. **Stop Word Filter** — remove palavras funcionais sem valor discriminativo (lista built-in para inglês)
4. **Snowball Stemmer** — reduz palavras à sua raiz morfológica (e.g., "charger" → "charger[]")

> O stemming foi preferido à lematização por ser computacionalmente mais eficiente e suficiente para datasets de pequena escala.

### 3. Representação Vetorial (TF-IDF)

Foi utilizada a representação TF-IDF em vez do Bag-of-Words simples, uma vez que o BoW atribui peso igual a todos os termos. O TF-IDF penaliza termos muito frequentes e pouco informativos, calculado como:

```
TF-IDF(t, d) = TF(t, d) × log(N / DF(t))
```

No KNIME 5.11.0, o cálculo foi realizado com os nós **Bag of Words Creator**, **TF** e **IDF** em sequência, resultando em 5.765 linhas de termos expandidos.

### 4. Divisão Treino/Teste

O nó **Table Partitioner** dividiu os dados em **80% treino / 20% teste**, com random sampling e seed fixo **42** para garantir reprodutibilidade.

### 5. Treinamento e Avaliação

O nó **Naive Bayes Learner** treina o classificador com a coluna `label` como target, e o **Naive Bayes Predictor** realiza as predições. O nó **Scorer** calcula as métricas finais de avaliação.

O Naive Bayes foi escolhido por ser um baseline clássico para classificação de texto — eficiente, interpretável e apropriado para representações esparsas como TF-IDF.

## Resultados

### Comparação de Abordagens

| Abordagem | Acurácia | Observação |
|---|---|---|
| Texto bruto | 100% | Overfitting severo — modelo memoriza as frases |
| TF-IDF com stemming | **53,6%** | Generaliza, mas com underfit |

### Métricas — Naive Bayes com TF-IDF

| Classe | Precisão | Recall | F1 |
|---|---|---|---|
| 0 (negativo) | Alta | 86,5% | — |
| 1 (positivo) | — | **17,8%** | **0,268** |
| **Geral** | | | **Acurácia: 53,6%** |

### Análise de Erros

O erro predominante são os **Falsos Negativos** (454 casos): o modelo classifica reviews positivas como negativas. Reviews positivas tendem a usar vocabulário mais neutro e moderado, enquanto reviews negativas utilizam linguagem mais expressiva — padrão mais fácil de capturar pelo Naive Bayes.

### Limitações Identificadas

- Dataset pequeno (1.490 instâncias após limpeza)
- Heterogeneidade de domínios (Amazon, IMDB e Yelp combinados)
- Suposição de independência entre termos (limitação intrínseca do Naive Bayes)

Para melhorar o desempenho, recomenda-se o uso de Logistic Regression, SVM ou embeddings como Word2Vec.

## Como Executar

**Pré-requisitos:**
- [KNIME Analytics Platform](https://www.knime.com/downloads) versão **5.11.0** ou superior
- Extensões: *KNIME Text Processing*, *KNIME Base Nodes*

**Passos:**

1. Clone o repositório:
   ```bash
   git clone https://github.com/joaopedro-bs/bmt261_knime_binary_classification.git
   ```

2. Abra o KNIME e importe o workspace ou abra diretamente o arquivo `BMT/BMT.knwf`

3. Ajuste os caminhos dos arquivos CSV nos nós **CSV Reader** (#1, #3 e #5) para apontar para a pasta `data/sentiment labelled sentences/csv/`

4. Execute o workflow completo (botão *Execute All*)

## Relatório

O relatório técnico completo com análise crítica dos resultados está disponível em [`report/report_knime_bmt.pdf`](report/report_knime_bmt.pdf).

## Autor

**João Pedro Barbosa Martins**
Programa de Engenharia de Sistemas e Computação — COPPE/UFRJ
[joaopedro@cos.ufrj.br](mailto:joaopedro@cos.ufrj.br)

---

> **Nota sobre uso de IA:** Este trabalho contou com o auxílio da ferramenta Claude (Anthropic, modelo Claude Sonnet 4.6) exclusivamente para revisão textual e sugestões de estrutura. A concepção do pipeline, a execução das análises no KNIME e todas as decisões técnicas foram realizadas pelo autor.
