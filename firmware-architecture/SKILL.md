---
name: firmware-architecture
description: Arquitetura geral de firmware para ESP8266/ESP32 com PlatformIO/Arduino. Use quando estruturar um novo firmware, decidir a organizacao em camadas, definir a ordem de inicializacao, ou desenhar a arquitetura de um projeto embarcado.
---

# Arquitetura de Firmware — ESP8266/ESP32 (PlatformIO/Arduino)

Guia de boas praticas para arquitetura de firmware embarcado em microcontroladores ESP.

---

## 1. CAMADAS DO SISTEMA

Organize o firmware em camadas com responsabilidades claras e dependencias top-down:

```
┌──────────────────────────────────────────────────────────────────┐
│  APPLICATION LAYER  (projeto: logica de negocio)                 │
│  src/main.cpp  loop.cpp  mqtt.cpp  processMqtt.cpp  execute.cpp  │
└───────────────────────────┬──────────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────────┐
│  SERVICE LAYER  (servicos de conectividade e gerenciamento)      │
│  wifi-manager  mqtt-manager  ota-update  server-manager          │
│  log-manager   time-rtc      captive-portal                      │
└───────────────────────────┬──────────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────────┐
│  DATA LAYER  (persistencia e estruturas de dados)                │
│  data-manager  file-system  json-manager  status-control<T>      │
│  backup-system  queue-delay                                      │
└───────────────────────────┬──────────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────────┐
│  PROTOCOL / HARDWARE LAYER  (drivers e protocolos)               │
│  mqtt-protocol  http-protocol  serial-protocol  mdns-resolver    │
│  rf-driver  touch-driver  shift-register  watchdog               │
└───────────────────────────┬──────────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────────┐
│  FOUNDATION LAYER  (utilidades base, usadas por todas)           │
│  firmware-config  debug-log  utils  uuid-generator               │
└──────────────────────────────────────────────────────────────────┘
```

Regras:
- Camadas superiores dependem das inferiores, **nunca** o inverso
- Foundation Layer nao depende de nenhuma outra camada do projeto
- Cada biblioteca pertence a **uma unica camada**

---

## 2. ESTRUTURA DE ARQUIVOS DO PROJETO

```
project-name/
├── platformio.ini
├── include/
│   ├── principals.h      ← UNICO arquivo com #include de todas as libs
│   ├── variables.h       ← todos os ponteiros globais (= nullptr)
│   ├── defines.h         ← pinos, timings, constantes de hardware
│   └── prototypes.h      ← prototipos de funcoes cross-file
├── src/
│   ├── main.cpp          ← setup() — apenas chamadas de setup*()
│   ├── globals.cpp       ← definicoes reais das variaveis (extern)
│   ├── loop.cpp          ← loop() — watchdog + checks
│   ├── mqtt.cpp          ← configuracao e conexao MQTT
│   ├── processMqtt.cpp   ← callbacks de MQTT recebido
│   ├── executeCommand.cpp ← execucao de comandos no hardware
│   ├── configDevice.cpp  ← configuracao de canais/hardware
│   └── [especificos do projeto]
├── lib/                  ← libs locais do projeto (raramente usado)
└── test/
```

---

## 3. SEQUENCIA DE INICIALIZACAO

A ordem de inicializacao deve respeitar as dependencias entre componentes:

```cpp
void setup(void)
{
    setupDebugSerial();          // 1. SEMPRE PRIMEIRO — habilita log
    setupDeclaredClass();        // 2. Criar TODOS os objetos com new
    setupSystemData();           // 3. Montar filesystem e carregar config
    setupHardware();             // 4. Hardware especifico (relays, sensores, etc.)
    setupWiFiConnection();       // 5. WiFi
    setupMQTTConnections();      // 6. Broker(s) MQTT
    setupLogManager();           // 7. Fila de logs
}
```

### setupDeclaredClass() — padrao

Centraliza a criacao de todos os objetos com `new`, respeitando a ordem de dependencia:

```cpp
void setupDeclaredClass(void)
{
    systemData    = new fileSystem(&SPIFFS);
    dataConfig    = new dataConfig_t;
    saveConfig    = new dataManager(systemData, dataConfig);
    wifi          = new wifiManager(saveConfig, dataConfig);
    mqttLocal     = new mqttManager(MQTT_SERVER_LOCAL, processLocalMqtt);
    mqttRemote    = new mqttManager(MQTT_SERVER_REMOTE, processRemoteMqtt);
    logSync       = new syncLog(mqttRemote, systemData);
    logManager    = new managerLog(logSync);
}
```

Regras:
- Criar na ordem de dependencia: objetos base antes dos que os consomem
- Callbacks passados no construtor quando possivel
- Todo ponteiro inicializado como `nullptr` em `variables.h`

### setupDebugSerial()

```cpp
void setupDebugSerial(void)
{
#if BOARD_DEBUG_LEVEL > LOG_LEVEL_NONE
    DEBUG_ESP_PORT.begin(SERIAL_BAUD);
#endif
}
```

---

## 4. LOOP PRINCIPAL

```cpp
void loop(void)
{
    utils.checkWDT();            // SEMPRE PRIMEIRO — alimenta watchdog

    readInputs();                // leitura de entradas fisicas

    if(captivePortalEnabled)
    {
        wifi->enablePortal();    // modo AP para captura de credenciais
    }

    if(wifi->loop() == true)     // apenas processa se WiFi OK
    {
        checkMQTTLocal();
        checkMQTTRemote();
        checkCommandQueue();
    }
}
```

Regras:
- **Nunca** usar `delay()` no `loop()` principal
- Watchdog sempre alimentado primeiro
- Processos MQTT/rede so executam se WiFi estiver conectado

---

## 5. PADRAO DE VARIAVEIS GLOBAIS (variables.h)

```cpp
#ifndef _VARIABLES_PROJECT_H
#define _VARIABLES_PROJECT_H

#include "principals.h"

// Declaracoes com extern (definicoes em globals.cpp)
extern fileSystem*    systemData;
extern dataConfig_t*  dataConfig;
extern dataManager*   saveConfig;
extern wifiManager*   wifi;
extern mqttManager*   mqttLocal;
extern mqttManager*   mqttRemote;
extern bool           captivePortalEnabled;

#endif
```

```cpp
// globals.cpp — definicoes reais
#include "variables.h"

fileSystem*    systemData           = nullptr;
dataConfig_t*  dataConfig           = nullptr;
dataManager*   saveConfig           = nullptr;
wifiManager*   wifi                 = nullptr;
mqttManager*   mqttLocal            = nullptr;
mqttManager*   mqttRemote           = nullptr;
bool           captivePortalEnabled = false;
```

Regras:
- Headers usam `extern` — definicoes em um unico `.cpp`
- Ponteiros sempre inicializados com `= nullptr`
- Booleanos inicializados com `= false`

---

## 6. PERSISTENCIA DE DADOS

| Dado | Storage | Mecanismo |
|------|---------|-----------|
| Credenciais WiFi | SPIFFS/LittleFS | Arquivo JSON |
| Configuracao MQTT | SPIFFS/LittleFS | Arquivo JSON |
| Estado de atuadores | SPIFFS/LittleFS | `statusControl<T>` |
| Backup de credenciais | EEPROM/NVS | Redundancia |
| Logs de operacao | SPIFFS/LittleFS | Arquivo JSON rotativo |
| UUID do dispositivo | SPIFFS/LittleFS | Gerado uma vez |

### statusControl — uso com template

```cpp
// No setup, apos carregar filesystem
deviceStatus->setup(&controlData);
statusLoad_e result = deviceStatus->checkLoadStatus();

if(result == statusLoad_e::STATUS_LOAD_RESET)
{
    initDefaultStatus(&controlData);
}

// Apos qualquer mudanca de estado
deviceStatus->setUpdateStatus();
```

---

## 7. DESIGN PATTERNS RECOMENDADOS

| Pattern | Onde aplicar | Proposito |
|---------|-------------|-----------|
| **Dependency Injection** | Construtores de todas as classes | Desacoplamento e testabilidade |
| **State Machine** | WiFi, MQTT, modos de operacao | Controle explicito de estados |
| **Strategy** | Protocolos, modos de operacao | Trocar comportamento sem mudar cliente |
| **Callback/Observer** | MQTT, interrupcoes, eventos | Evento → handler desacoplado |
| **Template** | `statusControl<T>` | Reutilizacao com type-safety |
| **Facade** | Managers de servicos | Simplificar APIs complexas |
| **Singleton** | Utilitarios globais | Acesso global controlado |
| **Command** | Filas de execucao | Execucao diferida e enfileirada |

---

## 8. SCHEDULING NAO-BLOQUEANTE (Ticker)

```cpp
// No setup — apos criar o Ticker
processTicker->attach_ms(INTERVAL_MS, periodicTask);

// Funcao chamada periodicamente
void periodicTask(void)
{
    // logica rapida e nao-bloqueante
}
```

Regras:
- **Nunca** usar `delay()` no `loop()` principal
- Processos periodicos via `Ticker::attach_ms()`
- O `loop()` principal e apenas orquestrador de checks
- ISRs e callbacks de Ticker devem ser rapidos (sem I/O bloqueante)

---

## 9. ARQUITETURA DUAL-BROKER MQTT

Quando o projeto requer separacao entre comandos/controle e telemetria/logs:

```
                         ┌─────────────────┐
                         │     Device      │
                         └────────┬────────┘
                                  │
               ┌──────────────────┼──────────────────┐
               │                                     │
    ┌──────────▼──────────┐             ┌────────────▼────────────┐
    │    LOCAL BROKER      │             │    REMOTE BROKER        │
    │  (comandos/status)   │             │  (logs/OTA/telemetria)  │
    └──────────────────────┘             └─────────────────────────┘
```

- LOCAL: subscriptions de comandos, config, status query
- REMOTE: subscriptions de OTA, logs de operacao, heartbeat

---

## 10. FLUXO OTA UPDATE

```
Remote MQTT → topico OTA recebido
    ↓
processRemoteMqtt() → identificar payload
    ↓
otaUpdate::beginUpdate(url, key, type)
    ↓
Download HTTPS
    ↓
Verificacao + flash
    ↓
ESP.restart()
```

---

## 11. CHECKLIST AO CRIAR NOVO PROJETO

- [ ] `principals.h` criado com todos os `#include` documentados
- [ ] `variables.h` com `extern` para todos os ponteiros globais
- [ ] `globals.cpp` com definicoes reais das variaveis
- [ ] `defines.h` com pinos e timings usando `#if REVISION == 'X'`
- [ ] `prototypes.h` com prototipos de funcoes cross-file
- [ ] `setup()` na ordem: debug → declare → data → hardware → wifi → mqtt → log
- [ ] `loop()` comeca com watchdog feed
- [ ] `setupDeclaredClass()` cria na ordem de dependencia
- [ ] `statusControl<T>` para todo estado que precisa sobreviver a reboot
- [ ] Ticker para tarefas periodicas, nunca `delay()`
- [ ] `platformio.ini` com environments separados por plataforma/revisao
