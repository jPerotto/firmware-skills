---
name: firmware-hardware-config
description: Abstracao de hardware para firmware embarcado (ESP8266/ESP32, PlatformIO/Arduino). Use quando configurar GPIO, pinos, variantes de placa, compilacao condicional, mapeamento de canais, PWM/DAC, ou qualquer interacao direta com hardware. Cobre defines.h, portable.h, BOARD_REVISION, BOARD_SETUP, preload de GPIO, channel mapping e configuracao condicional.
---

# Abstracao de Hardware — Firmware C++ (PlatformIO/Arduino)

Guia de boas praticas para abstracao de hardware em firmware embarcado ESP8266/ESP32.

**Principio central:** o hardware muda, o firmware se adapta.
Toda interacao com hardware deve ser parametrizada via defines, nunca hardcoded.

---

## 1. DEFINES.H — ORGANIZACAO

Cada firmware tem um `defines.h` em `include/` que centraliza todas as constantes
de hardware. Organizar por categoria:

```cpp
#ifndef _DEFINES_PROJECT_H
#define _DEFINES_PROJECT_H

// ============================================================
// VERSAO
// ============================================================
#define HARDWARE_VERSION "HW_1.0"
#define FIRMWARE_VERSION "FW_2.14.0"

// ============================================================
// PINOS — POR REVISAO DE HARDWARE
// ============================================================
#if BOARD_REVISION == 'A'
    #define PIN_RELAY_1     12
    #define PIN_RELAY_2     14
    #define PIN_RELAY_3     27
    #define PIN_RELAY_4     26
    #define PIN_LED_STATUS  2
    #define PIN_BUTTON      0
    #define PIN_TOUCH_1     T2
    #define PIN_TOUCH_2     T3
#elif BOARD_REVISION == 'B'
    #define PIN_RELAY_1     25
    #define PIN_RELAY_2     26
    #define PIN_LED_STATUS  2
    #define PIN_BUTTON      0
#else
    #error "BOARD_REVISION UNDEFINED"
#endif

// ============================================================
// SETUP — VARIANTES DE CANAL
// ============================================================
#define SETUP_RELAY_1CH     1
#define SETUP_RELAY_2CH     2
#define SETUP_RELAY_3CH     3
#define SETUP_RELAY_4CH     4

// ============================================================
// TIMINGS
// ============================================================
#define TIME_ENTER_AP_MODE          5000
#define TIME_CHECK_TOUCH            50
#define TIME_CHECK_TICKER           100
#define TIME_BLINK_LED              500

// ============================================================
// PWM / DAC (se aplicavel)
// ============================================================
#define FREQUENCY_PWM               10000
#define RESOLUTION_BITS_PWM         12
#define MIN_DAC_DIMMER              0
#define MAX_DAC_DIMMER              255
#define MIN_DIMMER_VALUE            0
#define MAX_DIMMER_VALUE            100

// ============================================================
// TOUCH CALIBRACAO (se aplicavel)
// ============================================================
#define N_SAMPLES                   100
#define CALIBRATION_PERIOD          PERIOD_1M
#define MAX_TOUCH_DEVIATION         50
#define TOUCH_MIN_VALUE_FACTOR      100

// ============================================================
// COMUNICACAO
// ============================================================
#define SERIAL_BAUD                 115200
#define BUFFER_MQTT_1KB             1024
#define BUFFER_MQTT_2KB             2048
#define BUFFER_MQTT_8KB             8192

#endif
```

Regras:
- Um unico `defines.h` por projeto — sem defines dispersos em outros arquivos
- Categorias separadas por comentarios `// ====`
- Pinos agrupados por revisao com `#if BOARD_REVISION`
- `#error` obrigatorio no `#else` final — impede compilacao sem revisao definida
- Sem numeros magicos — todo valor literal tem nome descritivo

---

## 2. PORTABLE.H — ABSTRACAO ESP32/ESP8266

A camada `portable.h` (em `firmware-master`) abstrai diferencas entre plataformas
usando macros com prefixo `p_`.

```cpp
#if defined(ESP32)
    #define p_FILE_SYSTEM           SPIFFSFS
    #define p_WEBSERVER             WebServer
    #define p_HTTP_UPDATE           HTTPUpdate
    #define p_MAX_FREE_RAM          getMaxAllocHeap
    #define p_MIN_FREE_RAM          getMinFreeHeap
    #define p_SLEEP_WIFI            setSleep
    #define p_WAKEUP_WIFI           getSleep
    #define p_WIFI_ENCRYPT_NONE     WIFI_AUTH_OPEN
    #define p_SET_HOST_NAME_STA     setHostname
    #define p_FORMAT_IF_FAIL        true
    #define p_WIFIMULTI             WiFiMulti
    #define p_WIFI_POWER            setTxPower
    #define WIFI_POWER              WIFI_POWER_19_5dBm
    #define WIFI_CHECK_TIMEOUT      PERIOD_SEC(3)

#elif defined(ESP8266)
    #define p_FILE_SYSTEM           FS
    #define p_WEBSERVER             ESP8266WebServer
    #define p_HTTP_UPDATE           ESP8266HTTPUpdate
    #define p_MAX_FREE_RAM          getMaxFreeBlockSize
    #define p_MIN_FREE_RAM          getFreeContStack
    #define p_SLEEP_WIFI            forceSleepBegin
    #define p_WAKEUP_WIFI           forceSleepWake
    #define p_WIFI_ENCRYPT_NONE     ENC_TYPE_NONE
    #define p_SET_HOST_NAME_STA     hostname
    #define p_FORMAT_IF_FAIL
    #define p_WIFIMULTI             ESP8266WiFiMulti
    #define p_WIFI_POWER            setOutputPower
    #define WIFI_POWER              20
    #define WIFI_CHECK_TIMEOUT      1

#else
    #error "NON-EXISTENT CORE"
#endif
```

Regras:
- Prefixo `p_` em toda macro portavel
- Cada `/** @brief ... */` documentando o proposito
- `#error` no `#else` — impede compilacao em plataforma desconhecida
- O codigo da aplicacao usa apenas `p_FILE_SYSTEM`, `p_WEBSERVER`, etc.
- **Nunca** usar `#ifdef ESP32` no codigo da aplicacao — usar as macros `p_`

---

## 3. COMPILACAO CONDICIONAL — VARIANTES

### 3.1 Dois niveis: REVISAO + SETUP

```cpp
// Nivel 1: Revisao de hardware (PCB diferente)
#if SMARTEK_SWITCH_MULTI == 'A'
    // pinos da revisao A
#elif SMARTEK_SWITCH_MULTI == 'B'
    // pinos da revisao B
#else
    #error "HARDWARE UNDEFINED"
#endif

// Nivel 2: Variante de setup (mesmo PCB, config diferente)
#if SMARTEK_BOARD_SETUP == SETUP_RELAY_1CH
    // 1 canal
#elif SMARTEK_BOARD_SETUP == SETUP_RELAY_2CH
    // 2 canais
#elif SMARTEK_BOARD_SETUP == SETUP_RELAY_3CH
    // 3 canais
#elif SMARTEK_BOARD_SETUP == SETUP_RELAY_4CH
    // 4 canais
#else
    #error "SETUP UNDEFINED"
#endif
```

### 3.2 Aninhamento de condicoes

```cpp
void setupChannelSwitch(void)
{
#if(SMARTEK_SWITCH_DELTA == 'A') || (SMARTEK_SWITCH_DELTA == 'B')
    #if SMARTEK_BOARD_SETUP == SETUP_RELAY_1CH
        setupChannel(channelSetup_t{channelExecute_e::CHANNEL_1, PIN_RELAY_1});
    #elif SMARTEK_BOARD_SETUP == SETUP_RELAY_2CH
        setupChannel(channelSetup_t{channelExecute_e::CHANNEL_1, PIN_RELAY_1});
        setupChannel(channelSetup_t{channelExecute_e::CHANNEL_2, PIN_RELAY_2});
    #elif SMARTEK_BOARD_SETUP == SETUP_RELAY_3CH
        setupChannel(channelSetup_t{channelExecute_e::CHANNEL_1, PIN_RELAY_1});
        setupChannel(channelSetup_t{channelExecute_e::CHANNEL_2, PIN_RELAY_2});
        setupChannel(channelSetup_t{channelExecute_e::CHANNEL_3, PIN_RELAY_3});
    #else
        #error "SETUP UNDEFINED"
    #endif
#else
    #error "HARDWARE UNDEFINED"
#endif
}
```

### 3.3 Revisao com diferenca funcional

```cpp
// Revisao A = com relay fisico
#if SMARTEK_SWITCH_DELTA == 'A'
    setupChannel(channelSetup_t{channelExecute_e::CHANNEL_1, PIN_RELAY_1, PIN_LED_ON, PIN_LED_OFF});

// Revisao B = sem relay (controle por firmware)
#elif SMARTEK_SWITCH_DELTA == 'B'
    setupChannel(channelSetup_t{channelExecute_e::CHANNEL_1, OUTPUT_DISABLED, PIN_LED_ON, PIN_LED_OFF});
#endif
```

Regras:
- `#error` obrigatorio em todo `#else` final
- Revisao agrupa pinos — setup agrupa canais
- Cada combinacao revisao × setup deve ser explicitamente suportada
- `OUTPUT_DISABLED` para pinos ausentes na revisao

---

## 4. PRELOAD DE GPIO

Preservar o estado dos pinos de saida durante reboot (quando nao ha perda de energia).

```cpp
void preloadStatus(void)
{
#if SMARTEK_BOARD_SETUP == SETUP_RELAY_1CH
    pinMode(PIN_RELAY_1, OUTPUT);
    digitalWrite(PIN_RELAY_1, digitalRead(PIN_RELAY_1));
#elif SMARTEK_BOARD_SETUP == SETUP_RELAY_2CH
    pinMode(PIN_RELAY_1, OUTPUT);
    pinMode(PIN_RELAY_2, OUTPUT);
    digitalWrite(PIN_RELAY_1, digitalRead(PIN_RELAY_1));
    digitalWrite(PIN_RELAY_2, digitalRead(PIN_RELAY_2));
#else
    #error "SETUP UNDEFINED"
#endif

    d_LOG_SWITCH("Preload status");
}
```

Regras:
- Chamar `preloadStatus()` **antes** de `setupDeclaredClass()` no `setup()`
- Sequencia: `pinMode(OUTPUT)` → `digitalRead()` → `digitalWrite()` — nessa ordem
- O `digitalRead()` captura o estado atual do pino antes da reconfiguração
- Aplicar para todos os pinos de saida (relays, LEDs de potencia)
- Nao aplicar para pinos de entrada (touch, botoes, sensores)

---

## 5. MAPEAMENTO DE CANAIS

### 5.1 Enum de canais logicos

```cpp
enum channelExecute_e
{
    CHANNEL_NOP = -1,
    CHANNEL_1   = 0,
    CHANNEL_2   = 1,
    CHANNEL_3   = 2,
    CHANNEL_4   = 3,
    CHANNEL_5   = 4,
    CHANNEL_6   = 5
};
```

### 5.2 Struct de configuracao — varia por firmware

```cpp
// Switch simples (relay only)
typedef struct
{
    channelExecute_e channel;
    uint8_t pinRelay;
} channelSetup_t;

// Switch com LED indicador
typedef struct
{
    channelExecute_e channel;
    uint8_t pinRelay;
    uint8_t pinLedOn;
    uint8_t pinLedOff;
} channelSetup_t;

// Touch + channel
typedef struct
{
    channelExecute_e channel;
    uint8_t pinTouch;
} touchSetup_t;
```

### 5.3 Struct de controle — runtime

```cpp
// Controle de relay por canal
typedef struct
{
    uint8_t stateRelay = {LOW};
    uint8_t pinRelay   = {NULL};
} relayControl_t;

// Controle completo de canal
typedef struct
{
    bool active                       = {false};
    relayControl_t relayControl;
    touchControl_t touchControl;
} channelControl_t;
```

### 5.4 Funcao de configuracao

```cpp
void setupChannel(channelSetup_t channelSetup)
{
    multiControl.controlChannel[channelSetup.channel].active = true;
    multiControl.controlChannel[channelSetup.channel].relayControl.pinRelay = channelSetup.pinRelay;
    v_LOG_SWITCH("setup channel %d relay %d", channelSetup.channel, channelSetup.pinRelay);
}
```

### 5.5 Conversao usuario → interno

```cpp
channelExecute_e convertUserChannelToOperate(userOperate_e channel)
{
    channelExecute_e channelOperate;

    switch(channel)
    {
        case userOperate_e::USER_CHANNEL_1:
        {
            channelOperate = channelExecute_e::CHANNEL_1;
            break;
        }
        case userOperate_e::USER_CHANNEL_2:
        {
            channelOperate = channelExecute_e::CHANNEL_2;
            break;
        }
        default:
        {
            channelOperate = channelExecute_e::CHANNEL_NOP;
            break;
        }
    }

    return channelOperate;
}
```

Regras:
- `channelExecute_e` e o tipo logico universal — usado em todo o firmware
- `CHANNEL_NOP = -1` indica canal invalido/desconfigurado
- A struct `channelSetup_t` varia por firmware conforme o hardware
- `setupChannel()` mapeia canal logico → pino fisico
- Funcao de conversao `user→internal` isola a API do hardware

---

## 6. CONTROLE DE GPIO

### 6.1 Relay — acionamento direto

```cpp
void controlActiveRelay(channelExecute_e channel, uint8_t status)
{
    multiControl.controlChannel[channel].relayControl.stateRelay = status;

    d_LOG_SWITCH("Comandar channel %d relay %d status %d",
                 channel + OFFSET_CHANNEL,
                 multiControl.controlChannel[channel].relayControl.pinRelay,
                 status);

    digitalWrite(multiControl.controlChannel[channel].relayControl.pinRelay, status);
    switchStatus->setUpdateStatus();  // persistir estado
}
```

### 6.2 LED — controle dual (on/off pins)

```cpp
void updateStatusLedChannel(channelExecute_e channel, uint8_t status)
{
    dimmerControl.controlChannel[channel].ledControl.statusLed = status;

    if(status == HIGH)
    {
        digitalWrite(dimmerControl.controlChannel[channel].ledControl.pinLedOn, HIGH);
        digitalWrite(dimmerControl.controlChannel[channel].ledControl.pinLedOff, LOW);
    }
    else
    {
        digitalWrite(dimmerControl.controlChannel[channel].ledControl.pinLedOff, HIGH);
        digitalWrite(dimmerControl.controlChannel[channel].ledControl.pinLedOn, LOW);
    }
}
```

### 6.3 DAC — controle analogico com rampa

```cpp
void executeControlDAC(channelExecute_e channel, int16_t startDAC, int16_t stopDAC)
{
    uint8_t timeRamp = calculeTimeRamp(startDAC, stopDAC);

    if(startDAC < stopDAC)
    {
        for(startDAC; startDAC < stopDAC; startDAC++)
        {
            delay(timeRamp);  // delay aceitavel aqui — rampa de hardware
            dacWrite(dimmerControl.controlChannel[channel].channelControl.pinChannelDAC, startDAC);
        }
    }
    else if(startDAC > stopDAC)
    {
        for(startDAC; startDAC > stopDAC; startDAC--)
        {
            delay(timeRamp);
            dacWrite(dimmerControl.controlChannel[channel].channelControl.pinChannelDAC, startDAC);
        }
    }
    else
    {
        dacWrite(dimmerControl.controlChannel[channel].channelControl.pinChannelDAC, stopDAC);
    }

    dimmerControl.controlChannel[channel].channelControl.dimmerDAC = stopDAC;
}
```

Regras:
- Logar toda mudanca de estado de GPIO em nivel `d_LOG`
- Persistir estado apos mudanca (`setUpdateStatus()`)
- `delay()` so e aceitavel em rampas de hardware — nunca no `loop()` principal
- Sempre atualizar a struct de controle alem do GPIO fisico

---

## 7. BUILD FLAGS — COMO DEFINIR VARIANTES

As macros de revisao e setup sao definidas no `platformio.ini`:

```ini
[env:revA-1ch-release]
build_flags =
    -D BOARD_REVISION='A'
    -D SMARTEK_BOARD_SETUP=1
    -D BOARD_DEBUG_LEVEL=0

[env:revA-2ch-release]
build_flags =
    -D BOARD_REVISION='A'
    -D SMARTEK_BOARD_SETUP=2
    -D BOARD_DEBUG_LEVEL=0

[env:revB-1ch-debug]
build_flags =
    -D BOARD_REVISION='B'
    -D SMARTEK_BOARD_SETUP=1
    -D BOARD_DEBUG_LEVEL=5
    -D DEBUG_ESP_PORT=Serial
```

Regras:
- Um ambiente por combinacao revisao × setup × modo (release/debug)
- `BOARD_REVISION` como char: `'A'`, `'B'`, `'C'`
- `SMARTEK_BOARD_SETUP` como int: `1`, `2`, `3`, `4`
- `BOARD_DEBUG_LEVEL`: `0` (release) ou `5` (debug verbose)

---

## 8. CONSTANTES DE HARDWARE — SEM NUMEROS MAGICOS

Toda constante de hardware deve ter nome descritivo:

```cpp
// CORRETO
#define FREQUENCY_PWM           10000
#define RESOLUTION_BITS_PWM     12
#define MIN_DAC_DIMMER          0
#define MAX_DAC_DIMMER          255
#define TIME_RESET_TOUCH        2000
#define TIME_START_DIMMER       500
#define TIME_RAMP_MAX           1300
#define TIME_RAMP_MIN           800
#define N_SAMPLES               100
#define OFFSET_CHANNEL          1

// INCORRETO
dacWrite(pin, 255);        // que e 255?
if(millis() - t > 2000)   // que e 2000?
channel + 1;              // que e 1?
```

Categorias de constantes:

| Categoria | Prefixo | Exemplos |
|-----------|---------|----------|
| Pinos | `PIN_` | `PIN_RELAY_1`, `PIN_LED_STATUS`, `PIN_DAC_1` |
| Tempo | `TIME_` | `TIME_RESET_TOUCH`, `TIME_CHECK_TICKER` |
| Limites | `MIN_`, `MAX_`, `N_` | `MIN_DAC_DIMMER`, `MAX_CHANNELS`, `N_SAMPLES` |
| Setup | `SETUP_` | `SETUP_RELAY_1CH`, `SETUP_DIMMER_2CH` |
| Frequencia | `FREQUENCY_` | `FREQUENCY_PWM` |
| Buffer | `BUFFER_` | `BUFFER_MQTT_8KB` |

---

## 9. CHECKLIST DE CONFIGURACAO DE HARDWARE

- [ ] `defines.h` unico por projeto com todas as constantes de hardware
- [ ] Pinos organizados por revisao com `#if BOARD_REVISION`
- [ ] `#error` em todo `#else` final de compilacao condicional
- [ ] Sem numeros magicos — todo valor literal com nome descritivo
- [ ] `preloadStatus()` chamada antes de `setupDeclaredClass()` no `setup()`
- [ ] GPIO preload: `pinMode(OUTPUT)` → `digitalRead()` → `digitalWrite()`
- [ ] `channelExecute_e` como tipo logico para canais
- [ ] `CHANNEL_NOP = -1` para canal invalido/desconfigurado
- [ ] Struct `channelSetup_t` mapeia canal logico → pinos fisicos
- [ ] Funcao `setupChannel()` configura cada canal individualmente
- [ ] Funcao de conversao `user→internal` isola API do hardware
- [ ] Toda mudanca de GPIO logada e persistida
- [ ] `portable.h` com macros `p_` para abstracao ESP32/ESP8266
- [ ] Build flags no `platformio.ini` para cada combinacao revisao × setup
- [ ] `delay()` proibido no loop — aceitavel apenas em rampas de hardware
