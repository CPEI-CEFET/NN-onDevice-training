# NN-onDevice-training

Treinamento online de redes neurais diretamente no **ESP32**, com implementação em C++ e scripts Python para simulação e análise dos experimentos.

O projeto compara duas redes MLP na predição de uma série temporal de temperatura: uma com treinamento convencional e outra com restrições sobre as normas dos pesos. O objetivo é investigar o comportamento do treinamento embarcado diante de valores extremos de entrada.

## Como funciona

Um script Python gera uma sequência de temperaturas simuladas e envia as amostras ao ESP32 pela interface serial. A cada amostra, o dispositivo:

1. Usa as duas temperaturas anteriores para predizer a temperatura atual.
2. Atualiza as duas redes por retropropagação e SGD.
3. Aplica a restrição de norma à rede regularizada.
4. Retorna as predições, os erros e as normas dos pesos.

O computador registra as respostas em CSV para análise posterior. **A inferência e o treinamento acontecem no ESP32.**

## Rede neural

As duas redes possuem a mesma arquitetura e são inicializadas com a mesma semente, `42`.

| Característica | Configuração |
|---|---|
| Arquitetura | 3 → 16 → 8 → 1 |
| Entradas | Duas temperaturas anteriores e uma constante `1.0` |
| Camadas ocultas | Leaky ReLU, com coeficiente `0.01` |
| Saída | Linear |
| Treinamento | SGD, uma amostra por atualização |
| Taxa de aprendizado atual | `0.25` |
| Taxa de atualização dos biases | Cinco vezes a taxa dos pesos |
| Normalização das temperaturas | Divisão por `100` |

A rede denominada **Lipschitz** no código aplica uma projeção por norma de Frobenius após cada atualização:

- Primeira camada: norma limitada a `L_max`.
- Segunda camada: norma limitada a `1.5 × L_max`.
- Valor atual de `L_max`: `1.0`.

A camada de saída não recebe essa restrição. Portanto, `L_max` representa o parâmetro de controle das camadas ocultas, e não um limite global garantido para a constante de Lipschitz da rede completa.

## Estrutura do projeto

```text
firmware/
├── CMakeLists.txt
├── sdkconfig
└── main/
    ├── CMakeLists.txt
    ├── main.cpp                # Comunicação serial e ciclo de treinamento
    ├── SimpleMLP.h             # Definição da rede
    └── SimpleMLP.cpp           # Inferência, SGD e restrições de norma

scripts/
├── simulador.py               # Geração de entradas e coleta via serial
├── graficoErro.py              # MSE acumulado ao longo do experimento
├── graficoCorrelacao.py        # Comparação entre temperaturas e predições
├── mseVSlr.py                  # Gráfico de MSE por taxa de aprendizado
├── resultados_artigo.csv
├── resultados_artigo004.csv
└── resultados_artigo025.csv
```

## Requisitos

- Placa ESP32 e conexão USB com comunicação serial.
- Ambiente ESP-IDF configurado. O `sdkconfig` incluído foi gerado com a versão **5.5.1**, para o alvo `esp32`.
- Python 3.
- Bibliotecas Python: `pyserial`, `numpy`, `pandas` e `matplotlib`.

Instale as dependências Python:

```bash
python -m pip install pyserial numpy pandas matplotlib
```

## Executando o experimento

### 1. Compilar e gravar o firmware

Com o ambiente ESP-IDF ativado, execute a partir da raiz do projeto:

```bash
cd firmware
idf.py build
idf.py -p PORTA_SERIAL flash
```

Substitua `PORTA_SERIAL` pela porta da placa, como `COM3` no Windows ou `/dev/ttyUSB0` no Linux.

### 2. Configurar a comunicação

Em `scripts/simulador.py`, ajuste a porta serial:

```python
PORTA = 'COM3'
BAUD = 115200
```

Feche qualquer monitor serial antes de iniciar a coleta, para liberar a porta.

### 3. Executar a simulação

A partir da raiz do projeto:

```bash
cd scripts
python simulador.py
```

O script envia **400 amostras** de uma série senoidal e injeta o valor `999.0` nos índices **200 a 204**, simulando leituras anômalas.

As respostas são salvas em `resultados_artigo.csv`, no diretório de execução. Uma nova execução sobrescreve esse arquivo; preserve uma cópia se quiser manter os resultados anteriores.

### 4. Gerar os gráficos

Dentro de `scripts/`, execute:

```bash
python graficoErro.py
python graficoCorrelacao.py
python mseVSlr.py
```

| Script | Dados utilizados | Saída |
|---|---|---|
| `graficoErro.py` | `resultados_artigo.csv` | `grafico_erro_artigo.pdf` e `mse_plot.csv` |
| `graficoCorrelacao.py` | `resultados_artigo004.csv` e `resultados_artigo025.csv` | `grafico_fidelidade.pdf` |
| `mseVSlr.py` | Valores definidos no próprio script | Gráfico exibido na tela |

O gráfico de correlação compara a rede padrão do arquivo `004` com a rede regularizada do arquivo `025`, identificadas nas legendas com taxas de aprendizado `0.04` e `0.25`, respectivamente.

O script `mseVSlr.py` apenas apresenta os valores já registrados no código; ele não executa uma varredura automática de taxas de aprendizado.

## Formato dos resultados

O CSV coletado contém as seguintes colunas:

| Coluna | Descrição |
|---|---|
| `Amostra` | Índice da amostra enviada |
| `Sensor` | Temperatura simulada enviada ao ESP32 |
| `PredPadrao` | Predição da rede padrão, em °C |
| `PredLipschitz` | Predição da rede regularizada, em °C |
| `ErroP` | Predição padrão menos temperatura enviada |
| `ErroL` | Predição regularizada menos temperatura enviada |
| `NormaP` | Norma conjunta dos pesos das duas camadas ocultas da rede padrão |
| `NormaL` | Norma conjunta dos pesos das duas camadas ocultas da rede regularizada |

As predições são calculadas **antes** da atualização com a amostra atual. As normas são registradas **depois** do treinamento e, na rede regularizada, da aplicação da restrição.

## Configuração dos experimentos

Os principais parâmetros podem ser ajustados em:

- `firmware/main/main.cpp`: taxa de aprendizado (`lr`) e limite de norma (`L_max`).
- `scripts/simulador.py`: porta serial, número de amostras, sinal de entrada e anomalias.
- `firmware/main/SimpleMLP.cpp`: inicialização, ativações e atualização dos parâmetros.

Alterações no firmware exigem nova compilação e gravação na placa.

## Observações sobre os resultados

O experimento utiliza temperaturas sintéticas enviadas pelo computador. A versão atual não realiza leitura direta de um sensor físico.

O MSE acumulado inclui as amostras anômalas, que podem dominar a métrica. Já o gráfico de correlação limita os eixos ao intervalo de 18 a 35 °C, ocultando pontos fora dessa faixa.

O coletor serial grava as linhas recebidas sem validar se contêm os sete campos numéricos esperados. Antes da análise, confira se mensagens de inicialização ou logs do dispositivo foram incluídos no CSV.

A comparação permite investigar estabilidade numérica nas condições testadas; conclusões sobre robustez devem considerar a taxa de aprendizado, as anomalias aplicadas e as limitações das visualizações.
