# imuLS6DS33 — Firmware de Aquisição de Dados IMU

## Visão Geral

Firmware embarcado para o microcontrolador **STM32F411CEU6**, desenvolvido no contexto de controle de uma haste/estrutura flexível.

**Resumo de funcionalidades:**
- Lê dados brutos de uma IMU (acelerômetro + giroscópio, 3 eixos cada) via **I2C** com **DMA**
- Transmite os dados para o PC via **USB CDC** (porta serial virtual)
- Aceita comandos simples (`'r'` = iniciar, `'s'` = parar)


---

## Hardware

| Item | Valor |
|------|-------|
| MCU | STM32F411CEU6 (Cortex-M4, 512K Flash, 128K RAM) |
| SYSCLK | 60 MHz (HSE 25 MHz + PLL) |
| Sensor | LSM6DS33 (ou compatível), I2C addr `0x6A` | 
| I2C | I2C1 Fast Mode 400 kHz — PB8 (SCL), PB9 (SDA) |
| USB | OTG FS, CDC — PA11 (DM), PA12 (DP) |
| Debug | SWD — PA13, PA14 |

---

> **Nota:** A confirmar o modelo do sensor a ser utilizado

## Compilação e Gravação

### Pré-requisitos
- **STM32CubeIDE** (toolchain GCC ARM inclusa)
- Cabo USB para gravação (SWD) ou o próprio USB da placa

### Build
1. Abra o projeto no STM32CubeIDE (`File → Import → Existing Projects`)
2. `Project → Build All` (ou `Ctrl+B`)

### Gravação
1. Conecte via SWD (ST-Link ou J-Link)
2. `Run → Debug` (ou `Run → Run`)
3. O firmware inicia automaticamente após o flash

---

## Como Usar

### 1. Conecte a placa via USB
A placa aparece como porta serial virtual:
- **Linux:** `/dev/ttyACM0`
- **Windows:** `COMx` (pode precisar de driver STM32 VCP)

### 2. Comandos

| Comando | Byte enviado | Efeito |
|---------|-------------|--------|
| Iniciar aquisição | `'r'` (0x72) | Começa a enviar dados a ~100 Hz |
| Parar aquisição | `'s'` (0x73) | Para o envio, timer continua rodando |

> **Nota:** Apenas o primeiro byte é verificado. Não há ACK/NACK de confirmação.

### 3. Dados Transmitidos

A cada ~10 ms, são enviados **14 bytes brutos** (lidos a partir do registrador `0x20` do sensor):

| Bytes | Dado | Formato |
|-------|------|---------|
| 0–1 | Temperatura | int16, LE |
| 2–3 | Giroscópio X | int16, LE |
| 4–5 | Giroscópio Y | int16, LE |
| 6–7 | Giroscópio Z | int16, LE |
| 8–9 | Acelerômetro X | int16, LE |
| 10–11 | Acelerômetro Y | int16, LE |
| 12–13 | Acelerômetro Z | int16, LE |

**Conversão para unidades físicas (baseada no datasheet LSM6DS33):**
- Temperatura: `T(°C) = raw / 256 + 25`
- Giroscópio (±2000 dps): `ω(dps) = raw × 70.0 / 1000`
- Acelerômetro (±16g): `a(g) = raw × 0.488 / 1000`



## Estrutura do Projeto

```
imuLS6DS33/
├── Core/Src/
│   ├── main.c                  ★ Toda a lógica do firmware
│   ├── stm32f4xx_it.c          ISR handlers (auto-gerado)
│   ├── stm32f4xx_hal_msp.c     Init de hardware (auto-gerado)
│   └── system_stm32f4xx.c      Config do sistema (CMSIS)
├── USB_DEVICE/App/
│   ├── usbd_cdc_if.c           ★ Interface USB CDC (recepção/transmissão)
│   ├── usbd_desc.c             Descritores USB
│   └── usb_device.c            Inicialização do stack USB
├── Drivers/                    HAL + CMSIS (auto-gerado)
├── Middlewares/                 Stack USB da ST (auto-gerado)
├── imuLS6DS33.ioc              Configuração STM32CubeMX
└── STM32F411CEUX_FLASH.ld      Linker script
```



---

## Limitações

| # | Limitação | Impacto |
|---|-----------|---------|
| 1 | Sem delimitador de frame (header/footer) | Perda de sincronização se um byte for perdido |
| 2 | Sem checksum/CRC | Impossível detectar corrupção |
| 3 | Pacotes descartados silenciosamente | Se USB está BUSY, dado é perdido sem aviso |
| 4 | Sem ACK de comandos | O PC não sabe se `'r'`/`'s'` foi recebido |
| 5 | Sem tratamento de erros I2C | Retorno de `HAL_I2C_Mem_Read_DMA` não é verificado |
| 6 | Sem timestamp nos dados | Impossível correlação temporal precisa |
| 7 | Sem watchdog | Se o firmware travar, não há reset automático |
| 8 | Valor de CTRL2_G possivelmente incorreto | Código escreve `0x46` (ODR 104 Hz), comentário indica 1666 Hz |

---

## Documentação Adicional

- **Referência Técnica:** [`docs/technical-reference.md`](docs/technical-reference.md) — arquitetura, fluxo de execução, periféricos, formato dos dados
- **Code Review:** [`docs/code-review.md`](docs/code-review.md) — revisão do código, riscos, sugestões, perguntas para o autor

---

*Documentação gerada em março de 2026 por análise direta do código-fonte. Não validada em hardware.*
