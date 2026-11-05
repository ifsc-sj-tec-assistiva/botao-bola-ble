# Documentação Técnica e Fundamentação de Arquitetura

Esta documentação detalha as decisões de engenharia, premissas de projeto e implementação do acionador assistivo pneumático via **BLE HID Mouse**, baseado no microcontrolador **ESP32-C3 Super Mini** e no conversor analógico-digital com sensor de pressão **HX710B**.

---

## 1. Contexto e Objetivos

O sistema atua como uma interface humana de tecnologia assistiva de baixo custo, convertendo picos de variação de pressão pneumática (como sopros acentuados ou pulsações manuais em um acionador pneu-mecânico tipo "botão bola") em comandos padrão de clique do mouse hospedeiro via Bluetooth Low Energy.

A arquitetura prioriza:

* **Determinismo e baixa latência:** resposta imediata ao estímulo físico sem bloqueios de tarefas.
* **Eficiência energética:** redução do footprint computacional e desativação de subsistemas ociosos para viabilizar operação prolongada em bateria LiPo.
* **Resiliência a ruído:** filtragem contínua do baseline analógico para evitar falsos positivos desencadeados por variações térmicas ou mecânicas lentas.

---

## 2. Decisões Arquiteturais e Otimizações

### 2.1. Arquitetura da Pilha Bluetooth (NimBLE-Arduino v2.x)
A comunicação sem fio é gerida via NimBLE-Arduino (v2.x), pilha otimizada nativamente para BLE (Bluetooth Low Energy). 
A estruturação direta dos serviços HID over GATT (HOGP) através desta biblioteca minimizou a alocação dinâmica de memória RAM e reduziu a carga computacional no ciclo de anúncios (advertising) e envio de relatórios.

### 2.2. Processamento do Sinal: Média Móvel em $\mathcal{O}(1)$

Para estabelecer o valor de referência (baseline) da pressão ambiente sem comprometer a frequência de amostragem, utiliza-se um buffer circular com atualização incremental da soma:

$$\text{somaLeituras}_{\text{nova}} = \text{somaLeituras}_{\text{anterior}} - x_{\text{antigo}} + x_{\text{novo}}$$

Isso elimina a necessidade de iterar sobre o array a cada leitura, reduzindo a complexidade temporal de $\mathcal{O}(N)$ para $\mathcal{O}(1)$ por ciclo de amostragem.

### 2.3. Gestão de Consumo Energético e Frequência de Operação

A otimização em modo de operação contínua fundamenta-se em:

1. Frequência da CPU: Ajustada em 80 MHz (setCpuFrequencyMhz(80)), garantindo estabilidade do subsistema de RF com margem em relação ao limite mínimo de operação estável do BLE (50 MHz).
2. Estabilidade Térmica: A redução da frequência e a eliminação de laços de repetição desnecessários minimizam a dissipação térmica do microcontrolador em uso prolongado, reduzindo o efeito do aumento da corrente de fuga (leakage current) do silício sob aquecimento sustentado.

---

## 3. Especificação de Parâmetros e Constantes do Firmware

| Parâmetro | Valor Configurado | Significado Físico / Funcional |
| --- | --- | --- |
| `PIN_OUT` | GPIO 2 | Pino de dados (*DOUT*) do HX710B |
| `PIN_SCK` | GPIO 3 | Pino de clock (*SCK*) do HX710B |
| `DELTA` | 60000 | Limiar absoluto de desvio da média ($\Delta P$) para validar um clique |
| `TAMANHO_BUFFER` | 20 amostras | Janela da média móvel (~2 segundos de histórico a 10 Hz) |
| `DEBOUNCE_MS` | 100 ms | Janela refratária mínima entre cliques consecutivos |
| `TIMEOUT_PRESSAO_MS` | 200 ms | Duração limite para travamento de estado em pressões prolongadas |
| `FATOR_RELEASE` | 0.35 | Histerese: a liberação ocorre quando o desvio cai a 35% do limiar de ativacão |

---

## 4. Fluxo Lógico e Máquina de Estados

### 4.1. Leitura Não-Bloqueante do HX710B (`lerHX710B`)

O conversor HX710B opera com amostragem nativa de 10 Hz. A função inspeciona a linha `PIN_OUT`:

* Se `PIN_OUT == HIGH`, a conversão atual ainda não finalizou. A função encerra retornando `0` sem manipular o clock, evitando dessincronizar ou colocar o sensor em *Power-Down* acidental.
* Se `PIN_OUT == LOW`, uma seção crítica (`portENTER_CRITICAL`) gera os 24 pulsos de clock para captura dos dados e o 25º pulso para manter o ganho/taxa do próximo ciclo.

### 4.2. Algoritmo de Detecção de Evento (Histerese Dinâmica)

```text
[ Leitura válida x_t ] 
          │
          ▼
┌──────────────────────────────────────────────┐
│  Atualiza soma acum. e calcula média móvel   │
│         diferença = | x_t - média |          │
└──────────────────────────────────────────────┘
          │
          ├─── (diferenca > DELTA) && !emEstadoPressionado ───► [ Dispara enviarClique() ]
          │                                                          │
          │                                                          ▼
          │                                              [ Liga emEstadoPressionado ]
          │                                              [ Guarda inicioPressaoMillis ]
          │
          └─── emEstadoPressionado == true ───────────────────► [ Avalia Condição de Rearme ]
                                                                     │
                                                                     ├── (diferenca < DELTA * 0.35)
                                                                     OU
                                                                     └── (tempo > TIMEOUT_PRESSAO_MS)
                                                                                 │
                                                                                 ▼
                                                                    [ Desliga emEstadoPressionado ]

```