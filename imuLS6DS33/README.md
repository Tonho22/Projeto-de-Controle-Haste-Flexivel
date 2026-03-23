# Controle de Haste Flexível 

Este projeto contém o firmware base para o controle de uma haste utilizando um microcontrolador STM32 e um sensor inercial (IMU) LSM6DS33. 

O código atual realiza a leitura contínua dos dados inerciais em alta frequência e os transmite para um computador via USB para controle, análise ou supervisão.

## 📌 Arquitetura e Funcionamento Atual

Neste momento, o firmware atua como um sistema de aquisição de dados otimizado (*Data Logger* / *Streamer*). Veja como as partes interagem:

1. **Inicialização (Setup):**
   - O microcontrolador configura os barramentos USB (CDC - Porta COM Virtual) e I2C1 (Pinos PB8 para SCL e PB9 para SDA).
   - O IMU (no endereço `0xD4` para escrita) é configurado para operar numa amostragem (*Output Data Rate - ODR*) de 1666 Hz.
   - O acelerômetro é configurado para escala de `±16g` e o giroscópio para `±2000 dps`.

2. **Laço de Aquisição (Timer + DMA):**
   - Um timer de hardware (`TIM11`) está configurado para gerar uma interrupção a cada **10 milissegundos** (Frequência de 100 Hz).
   - Quando ativado, a cada interrupção do timer, o STM32 dispara um pedido de leitura I2C utilizando **DMA** (Direct Memory Access).
   - Ele lê 14 bytes a partir do registrador `0x20` do IMU (que engloba temperatura, eixos X, Y e Z do giroscópio e do acelerômetro) sem travar o processador.

3. **Transmissão (USB CDC):**
   - Assim que o DMA termina de transferir os 14 bytes do IMU para a memória RAM do STM32, a interrupção de RX completo (`HAL_I2C_MemRxCpltCallback`) é acionada.
   - Nesse exato instante, os dados são despachados para o PC via porta USB.

4. **Comandos de Controle do PC:**
   - A aquisição não é iniciada automaticamente. O STM32 escuta a porta USB.
   - Enviar a letra `r` (run) pela serial liga a flag `startReading`, permitindo a leitura.
   - Enviar a letra `s` (stop) pela serial desliga a flag, interrompendo o fluxo.

---

## 🚀 Como Expandir e Adicionar Novos Sensores

O código atual foi escrito de forma direta (tudo no `main.c`) e lida com apenas um sensor de forma acoplada às interrupções. Para evoluir esse projeto para contemplar múltiplos sensores (como encoders, outros IMUs ou sensores de força na haste), **as seguintes modificações arquiteturais são recomendadas:**

### 1. Modularização do Código
Retire o código solto do `main.c`. Crie arquivos separados como `lsm6ds33.c / .h`, `motor_control.c / .h` e `sensor_manager.c / .h`. Isso evitará que o `main` vire um "código espaguete".

### 2. Gerenciamento do Barramento I2C (Evitando Colisões)
Se você for adicionar novos sensores I2C no mesmo barramento (I2C1), **não grave leituras DMA simultâneas**. O barramento I2C só consegue falar com um dispositivo por vez.
- **Solução:** Em vez de realizar a leitura diretamente no callback do `TIM11`, use o Timer apenas para "levantar uma flag" avisando que é hora de ler. No laço `while (1)` infinito do `main`, você cria uma **Máquina de Estados**:
  1. Inicia o DMA para o Sensor 1.
  2. Aguarda o término (via callback).
  3. Inicia o DMA para o Sensor 2.
  4. Aguarda o término.
  5. Empacota e envia tudo pela USB e aguarda o próximo ciclo do Timer.

### 3. Protocolo de Envio (Framing)
Atualmente o firmware simplesmente "cospe" 14 bytes na porta USB. Com múltiplos sensores, o PC não vai saber diferenciar o que é o Sensor A do Sensor B se perder algum byte.
- **Solução:** Crie uma estrutura (*Struct*) de pacote de dados contendo um Cabeçalho, os Dados e um Checksum:
  ```c
  // Exemplo
  [0xAA, 0x55] // Cabeçalho (Sync)
  [ID do Pacote] 
  [Tamanho do payload]
  [ ... Dados do Sensor 1 ... ]
  [ ... Dados do Sensor 2 ... ]
  [Checksum/CRC]
  ```

### 4. Controle Ativo da Haste (Malha Fechada)
Quando implementar a atuação na haste (ex: controle de um motor DC via PWM + Ponte H):
- O controle (como um bloco PID) deve rodar de forma síncrona aos dados do IMU.
- Assim que todos os sensores forem lidos no ciclo de 10ms (100 Hz), o algoritmo calcula o erro, define a nova saída do controlador PWM e **só então** envia os dados por USB para telemetria no computador. A transmissão USB deve ter a rotina de menor prioridade do controle do motor.

---

## 🛠️ Esquemático e Ligações (Atual)

- **Placa:** STM32F411CEU6 (Black Pill)
- **Comunicação PC:** USB nativa / CDC
- **Sensor Base:** IMU LSM6DS33
  - **SCL (I2C1):** Pino `PB8`
  - **SDA (I2C1):** Pino `PB9`
  - **VCC/GND:** 3.3V
  *(Usar resistores pull-up de ~4.7kΩ nas linhas SCL e SDA se o módulo do sensor não os possuir nativamente).*
