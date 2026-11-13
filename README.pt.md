# Inversor Deye → Home Assistant por Bluetooth (ESPHome, ESP32)

🇬🇧 [English version](README.md)

Lê um **inversor híbrido trifásico Deye** em tempo real no Home Assistant, **localmente**. A leitura é feita pela interface Bluetooth LE do **logger WiBLE**, com um ESP32 barato a correr ESPHome.

Não usa cloud, não precisa de cabos Modbus e não mexe no inversor. Basta um ESP32 perto do inversor e um ficheiro YAML.

Testado com um **Deye SUN‑10K‑SG05LP3‑EU‑SM2** (trifásico, bateria de baixa tensão de 48 V), com um logger **WiBLE** (firmware `DYDA_WiBLE_1.6.0`) e um **ESP32‑C6‑DevKitC‑1**. Está em uso desde agosto de 2026 e lê o inversor a cada 30 s.

---

## Porque existe

Os loggers Deye **WiBLE** mais recentes só funcionam com a cloud:

- **Não** abrem o Modbus TCP (porta 8899) nem a porta UDP de descoberta (48899). Por isso o [ha‑solarman](https://github.com/davidrapan/ha-solarman) e outras integrações locais não conseguem falar com eles. No ha‑solarman isto aparece como `'NoneType' object has no attribute 'parser'`; a causa é a porta fechada, não a configuração.
- Nenhum downgrade de firmware abre essas portas.

O logger **tem**, no entanto, uma interface Bluetooth LE, que a app da Deye usa na instalação, e que passa tramas Modbus RTU diretamente ao inversor. Este projeto usa esse caminho.

## O que se obtém

61 sensores e um controlo no Home Assistant:

| Grupo | Entidades |
|---|---|
| **Contadores de energia** (mantidos pelo inversor, ideais para o painel de energia) | Produção, consumo da casa, energia comprada/vendida, carga/descarga da bateria (diário e total) |
| **Bateria** | SOC, tensão, corrente, potência (+ a carregar / − a descarregar), temperatura |
| **Rede** | Tensão L1–L3, frequência, potência do CT interno, CT externo L1–L3 e total, potência da rede L1–L3 e total |
| **Saída do inversor** | Tensão e potência L1–L3, potência total, frequência |
| **Consumo** | Tensão e potência L1–L3, consumo total, potência UPS |
| **Painéis (PV)** | Potência, tensão e corrente de PV1–PV4, potência PV total |
| **Temperaturas** | Temperatura DC (dissipador) e AC |
| **Controlo** | `BLE Reconnect Interval` (número, ver abaixo) |

## Material

- Qualquer **ESP32 com Bluetooth LE** suportado pelo ESPHome. A configuração é para um `esp32-c6-devkitc-1` (8 MB de flash, ESP‑IDF). Para outra placa, altera o bloco `esp32:`.
- Coloca-o com bom alcance Bluetooth do logger: um ou dois metros, e evita caixas metálicas.
- Alimentação USB.

## Protocolo

Vem da engenharia inversa feita pela comunidade ao tráfego Bluetooth da app da Deye:

| | |
|---|---|
| Serviço GATT | `0x0922` |
| Característica de escrita | `0xFEC7` |
| Característica de notificação | `0xFED8` |
| Pedido | `AT+INVDATA=8,<trama Modbus RTU em hex + CRC16>\r\n` |
| Resposta | `+ok=<resposta Modbus RTU em hex, com CRC>\r\n` |

O ESP monta uma trama Modbus RTU normal de *leitura de registos* (`01 03 <início> <quantidade>`) e calcula o CRC16 na altura. Depois envia-a dentro do comando `AT+INVDATA` e interpreta a resposta `+ok=`.

### Mapa de registos

Usa o **mapa do Deye híbrido trifásico** (o que o ha‑solarman chama `deye_p3`), **não** o mapa monofásico que aparece em configurações antigas partilhadas. Os registos são lidos em 9 blocos de no máximo 14 registos cada, para que cada resposta caiba no MTU do Bluetooth:

| # | Início | Qtd. | Conteúdo |
|---|---|---|---|
| 1 | `0x0202` | 12 | Carga/descarga da bateria, compra/venda à rede (diário + total) |
| 2 | `0x020E` | 10 | Consumo da casa, produção PV (diário + total) |
| 3 | `0x021C` | 2 | Temperatura DC / AC |
| 4 | `0x024A` | 6 | Bateria: temperatura, tensão, SOC, potência, corrente |
| 5 | `0x0256` | 12 | Tensão da rede L1–L3, potência do CT interno, frequência |
| 6 | `0x0265` | 13 | CT externo, fator de potência, potência da rede L1–L3 e total |
| 7 | `0x0273` | 12 | Saída do inversor L1–L3 V/I/P, total, frequência |
| 8 | `0x0280` | 14 | Potência UPS, tensão e potência do consumo L1–L3, consumo total |
| 9 | `0x02A0` | 12 | Potência, tensão e corrente de PV1–PV4 |

Como se interpretam os valores:

- Os contadores de 32 bits vêm com a **palavra baixa primeiro** (convenção da Deye).
- As temperaturas são `(valor − 1000) × 0,1 °C`.
- As potências são inteiros de 16 bits com sinal.
- Cada resposta é comparada com o número de bytes esperado para o bloco pedido. Se uma resposta chegar atrasada, já depois de o ESP ter passado ao bloco seguinte, é descartada. Assim nunca é lida com o mapa de registos errado. Isto acontece logo a seguir a ligar.

> **Inversor monofásico?** O protocolo é o mesmo, mas os registos não. Substitui as tramas `cmd_*` e os casos do parser pelo mapa monofásico (o `deye_hybrid.yaml` do ha‑solarman é uma boa referência).

## Instalação

1. Instala o ESPHome (`pip install esphome`, ou o ESPHome Device Builder no Home Assistant).
2. Copia o `deye-sun10k-ble.yaml` e o `secrets.yaml.example` para a pasta do ESPHome. Muda o nome do segundo para `secrets.yaml` e preenche-o.
3. No bloco `substitutions:`, põe em `ble_mac` o endereço Bluetooth do **teu** logger (ver abaixo).
4. Na primeira vez, grava por USB:
   ```bash
   esphome run deye-sun10k-ble.yaml
   ```
   A partir daí, as atualizações vão por Wi‑Fi (OTA).
5. O Home Assistant encontra o dispositivo sozinho. Adiciona-o com a chave de encriptação da API que está no `secrets.yaml`.

### Encontrar o endereço do logger

- Faz um scan com uma app de telemóvel como a **nRF Connect**. O logger anuncia o serviço `0x0922`; escolhe o dispositivo que o anuncia.
- Em alternativa, junta temporariamente ao ESP um `esp32_ble_tracker:` com `on_ble_advertise` a escrever no log, e procura-o nos registos.

## Usar no Home Assistant

### Painel de energia

Usa os **contadores do próprio inversor**. Não acumulam erro e não se perdem quando o Home Assistant reinicia:

| Campo do painel de energia | Entidade |
|---|---|
| Consumo da rede | `Total Energy Bought` |
| Devolução à rede | `Total Energy Sold` |
| Produção solar | `Total Production` |
| Bateria: entrada / saída | `Total Battery Charge` / `Total Battery Discharge` |

### Tempo real ou cloud Deye: `BLE Reconnect Interval`

O logger só aceita **uma ligação Bluetooth de cada vez**. Enquanto o ESP está ligado, o logger não envia dados para a cloud Deye, por isso a app da Deye e qualquer integração por cloud deixam de atualizar.

A escolha faz-se com a entidade `BLE Reconnect Interval`:

| Valor | Comportamento |
|---|---|
| `0` (por omissão) | **Ligação permanente.** Dados em tempo real a cada 30 s; a cloud Deye fica parada. |
| `> 0` segundos | **Lê e desliga.** O ESP lê todos os blocos, desliga e volta a ligar ao fim de N segundos. A cloud Deye continua a funcionar entre leituras. |

Se voltares a pôr `0` enquanto está desligado, liga logo.

Em modo de ligação permanente, se passarem 90 s sem uma resposta válida, o ESP volta a ligar sozinho.

## Notas sobre o ESP32‑C6

- O log está configurado com `hardware_uart: UART0`. No C6, o log vai por omissão para o USB‑JTAG, que não aparece em muitos adaptadores USB‑série.
- Se lês a porta série com um script teu (pyserial), solta o DTR/RTS. Caso contrário o chip fica preso em reset e parece morto.

## Resolução de problemas

| Sintoma | Causa provável |
|---|---|
| Nunca liga | `ble_mac` errado; ESP longe do logger; app da Deye ligada ao logger ao mesmo tempo. |
| Liga, mas todos os valores ficam `unknown` | Inversor monofásico ou com outro mapa de registos (ver acima). |
| Aparece de vez em quando `Byte count … dropping frame` no log | Normal logo a seguir a ligar: é uma resposta atrasada a ser descartada. |
| A app ou a cloud da Deye deixaram de atualizar | Normal com `BLE Reconnect Interval = 0`; aumenta o valor se precisares da cloud. |

## Créditos

O protocolo Bluetooth vem da engenharia inversa feita pela comunidade à app da Deye. O mapa de registos trifásico segue as definições `deye_p3` do [ha‑solarman](https://github.com/davidrapan/ha-solarman).

## Licença

MIT. Ver [LICENSE](LICENSE). Usa por tua conta e risco: o código só **lê** registos (função `0x03`), mas fala com equipamento ligado à rede elétrica e a baterias.
