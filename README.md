# Gêmeo Digital de Ocupação

Pipeline de Ciência de Dados que prevê, a partir de sensores IoT, se uma sala está **ocupada** ou **vazia**, para que o sistema de automação predial desligue sozinho o ar-condicionado (HVAC) e a iluminação dos ambientes vazios.

Projeto avaliativo do **SCTEC Aperfeiçoar, curso Integração de IA com Dados**.
Professor: Felipe Rodrigues Souza. Aluno: Anderson Roza.

**Vídeo de apresentação:** _(link a inserir)_

---

## O problema

Num complexo industrial e tecnológico, salas técnicas e escritórios ficam com ar e luz ligados mesmo vazios, principalmente fora do horário de pico. Com sensores ambientais já instalados, a pergunta é: **dá para saber, minuto a minuto, se há gente na sala, sem câmera e sem sensor de presença?**

A variável alvo é binária: `Occupancy = 1` (ocupado) e `Occupancy = 0` (vazio).

## A base de dados

[Occupancy Detection Dataset](https://archive.ics.uci.edu/dataset/357/occupancy+detection), da UCI Machine Learning Repository. Leituras por minuto de uma sala de escritório em fevereiro de 2015:

| Variável | Descrição |
|---|---|
| `Temperature` | Temperatura do ambiente (°C) |
| `Humidity` | Umidade relativa do ar (%) |
| `Light` | Luminosidade (lux) |
| `CO2` | Concentração de CO₂ (ppm) |
| `HumidityRatio` | Razão de umidade, derivada da temperatura e da umidade |
| `Occupancy` | **Alvo:** 0 = desocupado, 1 = ocupado |

O pipeline usa o arquivo `datatest.txt` (2.665 leituras) para treino e teste. Os arquivos `datatraining.txt` e `datatest2.txt` (17.895 leituras de dias posteriores) são usados apenas como **dias novos**, para medir a estabilidade dos modelos fora da amostra.

## O pipeline em 7 fases

| Fase | O que é feito | Decisão principal |
|---|---|---|
| 1. Análise exploratória | Dimensões, tipos, `describe()`, 4 gráficos e interpretação | A coluna `date` é descartada: o modelo deve ler os sensores, não decorar a agenda da sala |
| 2. Limpeza | 82 duplicadas removidas, imputação testada com 2% de ausentes simulados, boxplots | **Mediana**, porque todas as variáveis são assimétricas à direita; outliers de luz mantidos por serem reais |
| 3. Feature engineering | Nova variável `razao_co2_luz = CO2 / (Light + 1)` | O `+ 1` evita divisão por zero; correlação de -0,87 com a ocupação |
| 4. Divisão | 80/20 com `stratify=y` e `random_state=42` | Divisão antes de qualquer transformação, para evitar data leakage |
| 5. Escalonamento | `StandardScaler` só para o KNN: `fit_transform` no treino e `transform` no teste | A árvore dispensa escala, porque decide por limiares em uma variável de cada vez |
| 6. Hiperparâmetros | KNN com K = 1, 3, 5, 7, 9, 11, 15 e árvore com `max_depth` = 2, 3, 4, 5, 7, None | Escolha por validação cruzada no treino: **K = 5** e **max_depth = 4** |
| 7. Veredito | Comparação no teste, matrizes de confusão, diagnóstico da nova variável | **Árvore de Decisão com max_depth = 4** adotada |

## Resultados

| Modelo | Acurácia no teste | Acurácia em dias novos |
|---|---|---|
| KNN, K = 5 | 98,84% | 88,87% |
| **Árvore de Decisão, max_depth = 4** | **99,03%** | **95,67%** |

**Overfitting identificado:** o KNN com K = 1 e a árvore sem limite de profundidade chegam a 100% no treino e são os piores de cada família nos dias novos (86,2% e 93,2%).

**Por que a árvore foi escolhida:** maior acurácia no teste, muito mais estável em dias novos, explicável em no máximo 4 perguntas em lux e ppm e mais simples de manter em produção, já que não depende de um `StandardScaler`.

## Tecnologias

- **Python 3** e **Jupyter Notebook**
- **pandas** e **NumPy** para manipulação dos dados
- **Matplotlib** e **Seaborn** para os gráficos
- **scikit-learn** para divisão dos dados, `StandardScaler`, KNN, Árvore de Decisão, validação cruzada e métricas

## Estrutura do repositório

```
.
├── pipeline_ocupacao.ipynb   # notebook principal, com as 7 fases
├── data/
│   ├── occupancy.csv         # base de trabalho (datatest.txt)
│   └── raw/                  # arquivos originais da UCI
├── requirements.txt          # dependências com versões
└── README.md
```

Todos os caminhos do notebook são relativos à raiz do repositório. Se a pasta `data/` não existir, o próprio notebook baixa a base da UCI.

## Como executar

**Localmente**

```bash
git clone https://github.com/andersondiroza-dev/ocupacao-ia-pipeline.git
cd ocupacao-ia-pipeline
python -m venv .venv
source .venv/bin/activate          # no Windows: .venv\Scripts\activate
pip install -r requirements.txt
jupyter notebook pipeline_ocupacao.ipynb
```

Depois, no Jupyter, execute **Kernel > Restart & Run All**.

**No Google Colab:** abra o `pipeline_ocupacao.ipynb` no Colab e execute todas as células. A primeira célula de dados baixa a base da UCI automaticamente.

## Fluxo de trabalho no Git

- `main`: versão final entregue.
- `develop`: integração das funcionalidades durante o desenvolvimento.
- Uma feature branch por fase, criada a partir da `develop` e integrada por pull request:
  `feature/fase-1-eda`, `feature/fase-2-data-prep`, `feature/fase-3-feature-engineering`, `feature/fase-4-divisao-treino-teste`, `feature/fase-5-escalonamento`, `feature/fase-6-hiperparametros`, `feature/fase-7-veredito` e `feature/documentacao-readme`.

## Melhorias futuras

1. **Retirar a `razao_co2_luz` do KNN** ou aplicar uma transformação logarítmica nela: sem a variável, o KNN sobe de 88,9% para 97,7% em dias novos.
2. **Confirmar o estado "vazio" por alguns minutos seguidos** antes de desligar o ar, para evitar desligamentos com pessoas na sala.
3. **Retreinar com dados de cada sala do complexo** e de outras estações do ano, porque a luz natural muda ao longo do ano.
4. **Monitorar a acurácia em produção** mês a mês e retreinar quando ela cair.
