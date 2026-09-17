# Nó Tensiômetro – ESP32 Heltec LoRaWAN

## Objetivo

Desenvolver o firmware de um **ESP32 Heltec com LoRaWAN** para um nó de monitoramento de tensiômetro.

O equipamento deverá:

1. Ler os sensores;
2. Montar um payload binário;
3. Conectar à rede LoRaWAN utilizando **ABP**;
4. Transmitir periodicamente os dados para o **ChirpStack**.

---

## Dados medidos

O nó deverá transmitir:

* Pressão do tensiômetro em **kPa**;
* Temperatura do ar em **°C**;
* Umidade relativa do ar em **%**;
* Tensão da bateria em **V**.

---

## Configuração LoRaWAN

Utilizar ativação **ABP**.

```text
DevEUI:
3333ACE658799B4E

DevAddr:
7033E132

NwkSKey:
33338CC087775C9E83BA7C5E9A889699

AppSKey:
33287ADF52F1143E8482AAD7CC8DFAA4
```

Configurar a região LoRaWAN utilizada pela rede do ChirpStack.

Para a rede atual:

```text
Região: LA915
Modo: LoRaWAN
Ativação: ABP
```

---

## Estrutura do payload

O payload terá **8 bytes**.

| Bytes | Informação    | Tipo     | Conversão |
| ----- | ------------- | -------- | --------- |
| 0–1   | Pressão       | `int16`  | kPa × 10  |
| 2–3   | Temperatura   | `int16`  | °C × 100  |
| 4–5   | Umidade do ar | `uint16` | % × 10    |
| 6–7   | Bateria       | `uint16` | V × 1000  |

Os valores devem ser enviados em **Little Endian**.

### Exemplo

Para:

```text
Pressão      = 35,6 kPa
Temperatura  = 25,35 °C
Umidade      = 55,5 %
Bateria      = 3,300 V
```

Os valores enviados serão:

```text
Pressão:
35,6 × 10 = 356 = 0x0164
→ 64 01

Temperatura:
25,35 × 100 = 2535 = 0x09E7
→ E7 09

Umidade:
55,5 × 10 = 555 = 0x022B
→ 2B 02

Bateria:
3,300 × 1000 = 3300 = 0x0CE4
→ E4 0C
```

Payload final:

```text
64 01 E7 09 2B 02 E4 0C
```

ou:

```text
6401E7092B02E40C
```

---

## Montagem do payload no ESP32

Exemplo da lógica de montagem:

```cpp
int16_t pressaoRaw     = pressao_kPa * 10.0;
int16_t temperaturaRaw = temperatura_C * 100.0;
uint16_t umidadeRaw    = umidade_pct * 10.0;
uint16_t bateriaRaw    = bateria_V * 1000.0;

uint8_t payload[8];

payload[0] = pressaoRaw & 0xFF;
payload[1] = (pressaoRaw >> 8) & 0xFF;

payload[2] = temperaturaRaw & 0xFF;
payload[3] = (temperaturaRaw >> 8) & 0xFF;

payload[4] = umidadeRaw & 0xFF;
payload[5] = (umidadeRaw >> 8) & 0xFF;

payload[6] = bateriaRaw & 0xFF;
payload[7] = (bateriaRaw >> 8) & 0xFF;
```

---

## Funcionamento esperado

O programa deverá seguir aproximadamente esta sequência:

```text
Inicialização
      ↓
Configurar sensores
      ↓
Configurar LoRaWAN LA915
      ↓
Configurar ABP
      ↓
Ler pressão
      ↓
Ler temperatura e umidade
      ↓
Ler tensão da bateria
      ↓
Montar payload de 8 bytes
      ↓
Transmitir LoRaWAN
      ↓
Aguardar intervalo
      ↓
Nova leitura
```

---

## Organização sugerida do código

Separar o programa em funções:

```cpp
void configuraSensores();
void configuraLoRaWAN();

float lePressao();
float leTemperatura();
float leUmidade();
float leBateria();

void montaPayload();
void enviaLoRaWAN();
```

O `loop()` deve ficar simples:

```cpp
void loop()
{
    leSensores();
    montaPayload();
    enviaLoRaWAN();

    delay(INTERVALO_ENVIO);
}
```

Posteriormente o `delay()` poderá ser substituído por **Deep Sleep** para reduzir o consumo de bateria.

---

## Resultado esperado no ChirpStack

O codec do Device Profile deverá produzir:

```json
{
    "pressao": 35.6,
    "temperatura": 25.35,
    "umidade_ar": 55.5,
    "bateria": 3.3,
    "tamanho_payload": 8
}
```

## Requisitos do trabalho

O firmware final deverá apresentar no monitor serial:

```text
Pressao: xx.x kPa
Temperatura: xx.xx C
Umidade: xx.x %
Bateria: x.xxx V
Payload: XX XX XX XX XX XX XX XX
Enviando LoRaWAN...
```

Antes de integrar os sensores reais, testar inicialmente com valores fixos:

```cpp
pressao     = 35.6;
temperatura = 25.35;
umidade     = 55.5;
bateria     = 3.300;
```

Após confirmar que o **ChirpStack está recebendo e decodificando corretamente o payload**, substituir gradualmente os valores simulados pelas leituras reais dos sensores.


# Configuração no helteck

#include "Arduino.h"
#include "LoRaWan_APP.h"

// --------------------------------------------------
// LoRaWAN ABP
// --------------------------------------------------

uint8_t devEui[] = {
    0x33, 0x33, 0xAC, 0xE6,
    0x58, 0x79, 0x9B, 0x4E
};

uint32_t devAddr = 0x7033E132;

uint8_t nwkSKey[] = {
    0x33, 0x33, 0x8C, 0xC0,
    0x87, 0x77, 0x5C, 0x9E,
    0x83, 0xBA, 0x7C, 0x5E,
    0x9A, 0x88, 0x96, 0x99
};

uint8_t appSKey[] = {
    0x33, 0x28, 0x7A, 0xDF,
    0x52, 0xF1, 0x14, 0x3E,
    0x84, 0x82, 0xAA, 0xD7,
    0xCC, 0x8D, 0xFA, 0xA4
};


void montaPayload()
{
    int16_t  pressaoRaw     = (int16_t)(pressao * 10.0);
    int16_t  temperaturaRaw = (int16_t)(temperatura * 100.0);
    uint16_t umidadeRaw     = (uint16_t)(umidade * 10.0);
    uint16_t bateriaRaw     = (uint16_t)(bateria * 1000.0);

    // Pressao - bytes 0 e 1
    appData[0] = pressaoRaw & 0xFF;
    appData[1] = (pressaoRaw >> 8) & 0xFF;

    // Temperatura - bytes 2 e 3
    appData[2] = temperaturaRaw & 0xFF;
    appData[3] = (temperaturaRaw >> 8) & 0xFF;

    // Umidade - bytes 4 e 5
    appData[4] = umidadeRaw & 0xFF;
    appData[5] = (umidadeRaw >> 8) & 0xFF;

    // Bateria - bytes 6 e 7
    appData[6] = bateriaRaw & 0xFF;
    appData[7] = (bateriaRaw >> 8) & 0xFF;

    appDataSize = 8;
}
