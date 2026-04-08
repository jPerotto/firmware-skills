---
name: firmware-templates
description: Templates de codigo para firmware embarcado (ESP8266/ESP32, PlatformIO/Arduino). Use quando criar um novo modulo, driver, state machine, command handler ou firmware completo. Cada template segue todas as skills de code-style, safe-functions, memory-safety e doxygen-review.
---

# Templates de Codigo — Firmware C++ (PlatformIO/Arduino)

Templates prontos para os componentes mais comuns de firmware embarcado ESP8266/ESP32.
Todos seguem os padroes das skills existentes.

---

## 1. MODULO PADRAO (setup/loop)

Componente com ciclo de vida `setup()` + `loop()`.

### modulo.h

```cpp
#ifndef _MODULO_H
#define _MODULO_H

/**
 * @file modulo.h
 * @author <AUTHOR_NAME> (<AUTHOR_EMAIL>)
 * @brief Biblioteca para [descricao em sentence case, sem acentos]
 * @version 1.0.0
 * @date DD-MM-YYYY
 * @copyright Copyright (c) YYYY \b <COMPANY_NAME>
 */

/**
 * @brief BIBLIOTECA BASE DE CONFIGURACOES E DEFINICOES
 */
#include <firmware-config.h>

/**
 * @brief BIBLIOTECA PARA LOGS DO MODULO
 */
#include "log-modulo.h"

/**
 * @brief PRINCIPAIS DEFINICOES E ESTRUTURAS DO MODULO
 */
#include "principais.h"

class modulo : public moduloLOG
{
  public:
    modulo(dependency* dep);
    ~modulo();

    void setup(void);
    bool loop(void);

  protected:
    void checkDeleteSubComponent(void);

  private:
    dependency* _dep = nullptr;
    moduleState_e _state = moduleState_e::STATE_UNDEFINED;
    bool _initialized = false;
};

#endif
```

### modulo.cpp

```cpp
#include "modulo.h"

/**
 * @brief CONSTRUTOR DO MODULO
 * @param dep PONTEIRO PARA DEPENDENCIA EXTERNA
 */
modulo::modulo(dependency* dep)
{
    if(dep == nullptr)
    {
        e_LOG_MODULO("NullPointerException");
    }
    else
    {
        _dep = dep;
    }
}

/**
 * @brief DESTRUTOR DO MODULO
 */
modulo::~modulo()
{
    checkDeleteSubComponent();
}

/**
 * @brief VERIFICA E DELETA O SUB COMPONENTE
 */
void modulo::checkDeleteSubComponent(void)
{
    // implementar para cada ponteiro proprio
}

/**
 * @brief INICIA A CONFIGURACAO DO MODULO
 */
void modulo::setup(void)
{
    if(_dep == nullptr)
    {
        e_LOG_MODULO("NullPointerException");
    }
    else
    {
        _initialized = true;
        _state = moduleState_e::STATE_IDLE;
        i_LOG_MODULO("Setup concluido");
    }
}

/**
 * @brief FUNCAO PRINCIPAL DE PROCESSAMENTO
 * @return \c true SE O PROCESSAMENTO OCORREU COM SUCESSO
 * @return \c false SE OCORREU ALGUM ERRO
 */
bool modulo::loop(void)
{
    bool resultado = false;

    if(_dep == nullptr)
    {
        resultado = false;
        e_LOG_MODULO("NullPointerException");
    }
    else
    {
        resultado = true;
        // logica principal
    }

    return resultado;
}
```

---

## 2. DRIVER DE HARDWARE

Componente que controla GPIO, PWM, DAC ou shift register.

### driver.h

```cpp
#ifndef _MY_DRIVER_H
#define _MY_DRIVER_H

/**
 * @file my-driver.h
 * @author <AUTHOR_NAME> (<AUTHOR_EMAIL>)
 * @brief Driver para controle de [hardware em sentence case]
 * @version 1.0.0
 * @date DD-MM-YYYY
 * @copyright Copyright (c) YYYY \b <COMPANY_NAME>
 */

#include <firmware-config.h>
#include "log-myDriver.h"
#include "principais.h"

class myDriver : public myDriverLOG
{
  public:
    myDriver();
    ~myDriver() {}

    void setup(uint8_t pin);
    bool execute(uint8_t channel, uint8_t value);
    uint8_t read(uint8_t channel);

  private:
    uint8_t _pin = NULL;
    bool _configured = false;
};

#endif
```

### driver.cpp

```cpp
#include "my-driver.h"

/**
 * @brief CONSTRUTOR DO DRIVER
 */
myDriver::myDriver() {}

/**
 * @brief CONFIGURA O PINO DO DRIVER
 * @param pin PINO GPIO PARA CONTROLE
 * @warning CHAMAR ANTES DE QUALQUER OPERACAO DE execute() OU read()
 */
void myDriver::setup(uint8_t pin)
{
    _pin = pin;
    pinMode(_pin, OUTPUT);
    _configured = true;
    i_LOG_MYDRIVER("Configurado pino %d", _pin);
}

/**
 * @brief ACIONA O HARDWARE NO CANAL ESPECIFICADO
 * @param channel CANAL LOGICO DE ACIONAMENTO
 * @param value VALOR DE ACIONAMENTO (HIGH/LOW OU 0-255)
 * @return \c true SE O ACIONAMENTO FOI REALIZADO
 * @return \c false SE O DRIVER NAO ESTA CONFIGURADO
 * @warning VERIFICAR LIMITES DE CORRENTE ANTES DE ACIONAR
 */
bool myDriver::execute(uint8_t channel, uint8_t value)
{
    bool success = false;

    if(_configured == false)
    {
        success = false;
        e_LOG_MYDRIVER("Driver nao configurado");
    }
    else
    {
        digitalWrite(_pin, value);
        success = true;
        d_LOG_MYDRIVER("Channel %d value %d", channel, value);
    }

    return success;
}

/**
 * @brief LE O ESTADO ATUAL DO PINO
 * @param channel CANAL LOGICO DE LEITURA
 * @return VALOR ATUAL DO PINO (HIGH/LOW)
 */
uint8_t myDriver::read(uint8_t channel)
{
    return digitalRead(_pin);
}
```

---

## 3. STATE MACHINE

Componente com estados discretos e transicoes.

### state-machine.h

```cpp
#ifndef _MY_FSM_H
#define _MY_FSM_H

#include <firmware-config.h>
#include "log-myFsm.h"

enum fsmState_e
{
    FSM_UNDEFINED = 0,
    FSM_IDLE      = 1,
    FSM_RUNNING   = 2,
    FSM_ERROR     = 3
};

typedef struct
{
    fsmState_e state          = {fsmState_e::FSM_UNDEFINED};
    uint32_t lastStateChange  = {NULL};
    uint8_t errorCount        = {NULL};
} fsmControl_t;

class myFSM : public myFsmLOG
{
  public:
    myFSM(dependency* dep);
    ~myFSM();

    void setup(void);
    bool loop(void);
    fsmState_e getState(void);

  protected:
    void checkDeleteDep(void);
    void setState(fsmState_e newState);
    void handleIdle(void);
    void handleRunning(void);
    void handleError(void);
    void resetState(void);

  private:
    dependency* _dep = nullptr;
    fsmControl_t _control;
};

#endif
```

### state-machine.cpp

```cpp
#include "my-fsm.h"

/**
 * @brief CONSTRUTOR DA MAQUINA DE ESTADOS
 * @param dep PONTEIRO PARA DEPENDENCIA EXTERNA
 */
myFSM::myFSM(dependency* dep)
{
    if(dep == nullptr)
    {
        e_LOG_MYFSM("NullPointerException");
    }
    else
    {
        _dep = dep;
    }
}

/**
 * @brief DESTRUTOR DA MAQUINA DE ESTADOS
 */
myFSM::~myFSM()
{
    checkDeleteDep();
}

/**
 * @brief VERIFICA E DELETA A DEPENDENCIA
 */
void myFSM::checkDeleteDep(void)
{
    // somente se _dep for proprio (criado com new)
}

/**
 * @brief INICIA A CONFIGURACAO DA FSM
 */
void myFSM::setup(void)
{
    setState(fsmState_e::FSM_IDLE);
    i_LOG_MYFSM("FSM inicializada");
}

/**
 * @brief ALTERA O ESTADO DA FSM
 * @param newState NOVO ESTADO DA MAQUINA
 */
void myFSM::setState(fsmState_e newState)
{
    if(_control.state != newState)
    {
        d_LOG_MYFSM("Estado %d -> %d", _control.state, newState);
        _control.state = newState;
        _control.lastStateChange = millis();
    }
}

/**
 * @brief RETORNA O ESTADO ATUAL DA FSM
 * @return \c fsmState_e COM O ESTADO ATUAL
 */
fsmState_e myFSM::getState(void)
{
    return _control.state;
}

/**
 * @brief LOOP PRINCIPAL DA MAQUINA DE ESTADOS
 * @return \c true SE O PROCESSAMENTO OCORREU COM SUCESSO
 * @return \c false SE A FSM ESTA EM ESTADO DE ERRO
 */
bool myFSM::loop(void)
{
    bool resultado = false;

    if(_dep == nullptr)
    {
        resultado = false;
        e_LOG_MYFSM("NullPointerException");
    }
    else
    {
        switch(_control.state)
        {
            case fsmState_e::FSM_IDLE:
            {
                handleIdle();
                resultado = true;
                break;
            }
            case fsmState_e::FSM_RUNNING:
            {
                handleRunning();
                resultado = true;
                break;
            }
            case fsmState_e::FSM_ERROR:
            {
                handleError();
                resultado = false;
                break;
            }
            default:
            {
                e_LOG_MYFSM("Estado indefinido");
                resetState();
                resultado = false;
                break;
            }
        }
    }

    return resultado;
}

/**
 * @brief PROCESSA O ESTADO IDLE
 */
void myFSM::handleIdle(void)
{
    // logica do estado idle
}

/**
 * @brief PROCESSA O ESTADO RUNNING
 */
void myFSM::handleRunning(void)
{
    // logica do estado running
}

/**
 * @brief PROCESSA O ESTADO DE ERRO
 */
void myFSM::handleError(void)
{
    _control.errorCount++;

    if(_control.errorCount >= MAX_ERRORS)
    {
        resetState();
    }
}

/**
 * @brief RESETA A FSM PARA O ESTADO INICIAL
 */
void myFSM::resetState(void)
{
    _control.state = fsmState_e::FSM_IDLE;
    _control.lastStateChange = millis();
    _control.errorCount = NULL;
    i_LOG_MYFSM("FSM resetada");
}
```

---

## 4. COMMAND HANDLER

Componente que processa comandos JSON recebidos via MQTT.

### command-handler padrao

```cpp
/**
 * @brief PROCESSA UM COMANDO RECEBIDO VIA MQTT
 * @param jsonDoc DOCUMENTO JSON COM O COMANDO
 * @return \c true SE O COMANDO FOI EXECUTADO COM SUCESSO
 * @return \c false SE O COMANDO FALHOU OU E INVALIDO
 */
bool executeCommand(DynamicJsonDocument& jsonDoc)
{
    bool executeStatus = false;
    JsonVariant type = jsonDoc["type"].as<JsonVariant>();

    if((type.isNull() == true) || (type.is<const char*>() == false))
    {
        executeStatus = false;
        e_LOG_MODULE("Tipo de comando invalido");
    }
    else if(strcmp(type, "COMMAND") == NULL)
    {
        uint8_t state = jsonDoc["value"];
        userOperate_e channel = (userOperate_e)jsonDoc["channel"].as<uint8_t>();
        channelExecute_e channelExecute = convertUserChannelToOperate(channel);

        if(executeChannelControl(channelExecute, state) == false)
        {
            executeStatus = false;
            e_LOG_MODULE("Canal desconfigurado");
        }
        else
        {
            executeStatus = true;
        }
    }
    else
    {
        executeStatus = false;
        e_LOG_MODULE("Comando desconhecido");
    }

    return executeStatus;
}
```

---

## 5. FIRMWARE COMPLETO — ESQUELETO

Estrutura minima de um novo projeto de firmware.

### include/principals.h

```cpp
#ifndef _PRINCIPALS_PROJECT_H
#define _PRINCIPALS_PROJECT_H

/**
 * @brief BIBLIOTECA BASE DO ARDUINO
 */
#include <Arduino.h>

/**
 * @brief BIBLIOTECA PARA CONTROLE DE MICRO TAREFAS
 */
#include <Ticker.h>

/**
 * @brief BIBLIOTECA BASE DE CONFIGURACOES
 */
#include <firmware-config.h>

/**
 * @brief BIBLIOTECA PARA DIVERSAS UTILIDADES
 */
#include <utils.h>

/**
 * @brief BIBLIOTECA PARA PERSISTENCIA DE DADOS
 */
#include <data-manager.h>

/**
 * @brief BIBLIOTECA DE GERENCIAMENTO WIFI
 */
#include <wifi-manager.h>

/**
 * @brief BIBLIOTECA DE GERENCIAMENTO MQTT
 */
#include <mqtt-manager.h>

/**
 * @brief BIBLIOTECA PARA CONTROLE DE STATUS
 */
#include <status-control.h>

/**
 * @brief DEFINICOES DE HARDWARE DO PROJETO
 */
#include "defines.h"

/**
 * @brief VARIAVEIS GLOBAIS DO PROJETO
 */
#include "variables.h"

/**
 * @brief PROTOTIPOS DE FUNCOES CROSS-FILE
 */
#include "prototypes.h"

#endif
```

### include/variables.h

```cpp
#ifndef _VARIABLES_PROJECT_H
#define _VARIABLES_PROJECT_H

#include "principals.h"

// --- PERSISTENCIA ---
extern fileSystem*    systemData;
extern dataConfig_t*  dataConfig;
extern dataManager*   saveConfig;

// --- COMUNICACAO ---
extern wifiManager*   wifi;
extern mqttManager*   localMQTT;
extern mqttManager*   remoteMQTT;

// --- LOG ---
extern syncLog*       logSync;
extern managerLog*    logManager;

// --- CONTROLE ---
extern bool           enablePortalCaptive;
extern bool           sendStatesReconnectLocal;
extern bool           sendStatesReconnectRemote;

#endif
```

### src/globals.cpp

```cpp
#include "variables.h"

// --- PERSISTENCIA ---
fileSystem*    systemData                = nullptr;
dataConfig_t*  dataConfig               = nullptr;
dataManager*   saveConfig               = nullptr;

// --- COMUNICACAO ---
wifiManager*   wifi                     = nullptr;
mqttManager*   localMQTT                = nullptr;
mqttManager*   remoteMQTT               = nullptr;

// --- LOG ---
syncLog*       logSync                  = nullptr;
managerLog*    logManager               = nullptr;

// --- CONTROLE ---
bool           enablePortalCaptive      = false;
bool           sendStatesReconnectLocal = false;
bool           sendStatesReconnectRemote = false;
```

### src/main.cpp

```cpp
#include "variables.h"

void setup(void)
{
    setupDebugSerial();
    preloadStatus();
    setupDeclaredClass();
    setupSystemData();
    setupHardware();
    setupWiFiConnection();
    setupLocalMQTTConnection();
    setupRemoteMQTTConnection();
    setupLogManager();

    v_LOG_SETUP("SETUP SUCESS %s %s", HARDWARE_VERSION, FIRMWARE_VERSION);
}
```

### src/loop.cpp

```cpp
#include "variables.h"

void loop(void)
{
    utils.checkWDT();

    if(enablePortalCaptive == true)
    {
        enablePortalCaptive = false;
        logManager->sendLog("Entrando em modo AP");
        wifi->enablePortal();
    }

    if(wifi->loop() == true)
    {
        checkQueueCommand();

        bool statusLocal = checkProcessMQTTLocal();
        bool statusRemote = checkProcessMQTTRemote();

        if((statusLocal == true) && (statusRemote == true))
        {
            checkSyncControl();
        }
    }
}
```

### include/prototypes.h

```cpp
#ifndef _PROTOTYPES_PROJECT_H
#define _PROTOTYPES_PROJECT_H

// main.cpp
void setupDebugSerial(void);
void setupDeclaredClass(void);
void setupSystemData(void);
void setupHardware(void);
void setupWiFiConnection(void);
void setupLocalMQTTConnection(void);
void setupRemoteMQTTConnection(void);
void setupLogManager(void);

// loop.cpp
void preloadStatus(void);
void checkQueueCommand(void);
bool checkProcessMQTTLocal(void);
bool checkProcessMQTTRemote(void);
void checkSyncControl(void);

// processMqtt.cpp
void processLocalMqtt(const char* topic, DynamicJsonDocument& jsonDoc);
void processRemoteMqtt(const char* topic, DynamicJsonDocument& jsonDoc);

// executeCommand.cpp
bool executeCommand(DynamicJsonDocument& jsonDoc);

#endif
```

---

## 6. QUANDO USAR CADA TEMPLATE

| Preciso de... | Template |
|---------------|----------|
| Novo componente com ciclo setup/loop | **Modulo Padrao** |
| Controle de GPIO, PWM, DAC | **Driver de Hardware** |
| Componente com estados discretos | **State Machine** |
| Processar comandos JSON do MQTT | **Command Handler** |
| Novo projeto de firmware completo | **Firmware Completo** |
| Nova biblioteca reutilizavel | Usar skill `firmware-new-library` |
