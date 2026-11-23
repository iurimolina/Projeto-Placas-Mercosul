# Projeto — Placas Mercosul (Grupo C)

Classificação de recortes de placas veiculares em quatro categorias: `mercosul`, `tradicional`, `mercosul moto` e `tradicional moto`.

## Estrutura

```
├── 01 — Grupo C — Exploração inicial.ipynb               leitura do annotations.xml, visualização e conferência das anotações
├── 02 — Grupo C — Organização do dataset.ipynb           recorte das placas e divisão treino/validação (70/30)
├── 03 — Grupo C — Análise exploratória e CNN do zero.ipynb  análise exploratória, CNN do zero e métricas
├── dados/
│   ├── treino/<categoria>/*.jpg                          14.141 recortes
│   └── validacao/<categoria>/*.jpg                        6.064 recortes
├── resultados/                                           gráficos, tabelas e o modelo treinado (cnn_do_zero.pt)
├── requirements.txt
└── README.md
```

## Como executar

**Google Colab:** abrir os notebooks na ordem 01 → 02 → 03, com GPU (`Ambiente de execução → Alterar tipo → GPU`). Os caminhos apontam para o Drive compartilhado da turma (`COMPUTER VISION_TURMA 2026`).

**Local:** a pasta `dados` já está incluída, então o notebook 03 roda direto:

```
pip install -r requirements.txt
jupyter notebook
```

Os notebooks 01 e 02 leem o `annotations.xml` e as fotografias originais do Drive e só rodam no Colab.

O notebook 03 levou ~15 min numa RTX 3060. Sem GPU ele roda, mas bem mais devagar.

## Dados

| categoria | treino | validação | total | % |
|---|---|---|---|---|
| mercosul | 6.934 | 2.973 | 9.907 | 49,0 |
| tradicional | 5.123 | 2.196 | 7.319 | 36,2 |
| mercosul moto | 1.533 | 658 | 2.191 | 10,8 |
| tradicional moto | 551 | 237 | 788 | 3,9 |

A maior categoria tem 12,6x mais exemplos que a menor.

## Modelo

CNN construída do zero (sem pesos pré-treinados), montada incrementalmente — cada experimento adiciona uma coisa ao anterior (12 épocas, oversampling, sem aumento de dados):

| experimento | parâmetros | F1 macro |
|---|---|---|
| 1 bloco (16) | 516 | 0,596 |
| 2 blocos (16-32) | 5,2 mil | 0,737 |
| 3 blocos (16-32-64) | 23,8 mil | 0,849 |
| 4 blocos (16-32-64-128) | 98,0 mil | 0,880 |
| + BatchNorm | 98,4 mil | 0,879 |
| + 2 convs por bloco | 295 mil | 0,882 |
| + Dropout 0.3 | 295 mil | 0,888 |
| 5 blocos (…-256) | 1,18 mi | 0,882 |

As redes maiores decoram o treino (sem aumento de dados). O modelo final é uma rede **simples de 25 mil parâmetros**: 4 blocos `conv 3×3 → BatchNorm → ReLU → MaxPool` com canais 8-16-32-64, média global, dropout 0.3 e uma camada linear. Treino: AdamW (weight decay 1e-3) com OneCycle, por 15 épocas.

Entrada 64 × 128, com o recorte redimensionado sem distorcer (bordas pretas) e normalizado pela média e pelo desvio do treino. Sem aumento de dados.

## Balanceamento por reamostragem (rede simples)

| estratégia | imagens de treino | F1 macro | acurácia balanceada | recall tradicional moto |
|---|---|---|---|---|
| sem reamostragem | 14.141 | 0,873 | 0,891 | 0,798 |
| **oversampling** | 27.736 | **0,877** | **0,907** | **0,857** |
| undersampling | 2.204 | 0,798 | 0,840 | 0,810 |

O modelo final usa **oversampling**: as categorias menores são sorteadas com reposição até 6.934 imagens cada.

## Resultado final (validação)

| métrica | valor |
|---|---|
| acurácia | 0,914 |
| acurácia balanceada | 0,907 |
| F1 macro | 0,877 |

| categoria | precisão | recall | F1 |
|---|---|---|---|
| mercosul | 0,962 | 0,903 | 0,932 |
| tradicional | 0,905 | 0,927 | 0,916 |
| mercosul moto | 0,841 | 0,939 | 0,887 |
| tradicional moto | 0,702 | 0,857 | 0,772 |

## Overfitting

| | rede de 295 mil parâmetros | rede simples (25 mil) |
|---|---|---|
| acurácia treino / validação | 100% / 92,3% | 93,5% / 91,4% |
| perda treino / validação | 0,0005 / 0,56 | 0,21 / 0,31 |
| perda de validação | sobe desde a época 4 | cai até a última época |

A rede simples abre mão de ~0,01 de F1 macro para não decorar o treino.

## Dificuldades

- **Desbalanceamento:** `tradicional moto` tem só 551 imagens de treino e é a categoria com pior F1.
- **Recortes pequenos:** 20,7% têm menos de 20 px de altura; a troca mercosul × tradicional é o erro mais comum.
- **Proporções diferentes:** carro ~2,4, moto ~1,2 — redimensionar distorcendo apagaria essa diferença.
- **Fotos noturnas em infravermelho:** sem cor, a faixa azul do Mercosul não aparece.
- **Rótulos errados no dataset:** há motos anotadas como carro e Mercosul anotada como tradicional (estimados em ~3%); parte dos "erros" do modelo são anotações erradas.
- **Validação otimista:** o split é por recorte (a mesma fotografia pode ter placas no treino e na validação), e a arquitetura e a época foram escolhidas na própria validação. Um conjunto de teste separado por fotografia daria uma medida mais confiável.
