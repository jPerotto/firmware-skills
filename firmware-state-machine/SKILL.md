---
name: firmware-state-machine
description: Padroes para maquinas de estado (FSM) em firmware embarcado (ESP8266/ESP32, PlatformIO/Arduino). Use quando implementar controle de estado, transicoes, timeout, persistencia de estado, ou qualquer componente com estados discretos (WiFi, MQTT, modos de operacao, controle de hardware).
---

# Maquinas de Estado — Firmware C++ (PlatformIO/Arduino)

Guia de boas praticas para implementacao de FSMs (Finite State Machines) em firmware embarcado ESP8266/ESP32.

**Principio central:** estados devem ser explicitos, transicoes rastreaveis e timeouts controlaveis.
Se o comportamento do sistema depende de "em que ponto estamos", use uma maquina de estado.

---

## 1. DEFINICAO DE ESTADOS — ENUM

Cada grupo de estados e um `enum` com sufixo `_e`, primeiro valor `UNDEFINED = 0`
ou equivalente semantico que represente "ainda nao inicializado".

```cpp
// WiFi — modos de operacao
enum operationMode_e
{
    WIFI_UNDEFINED  = 0,
    WIFI_CONNECT    = 1,
    WIFI_DISTRIBUTE = 2,
    WIFI_CAPTURE    = 3,
    WIFI_PORTAL     = 4
};

// WiFi — status de conexao
enum statusConnect_e
{
    STA_CONNECTED       = 1,
    STA_WAIT_CONNECT    = 2,
    STA_DISCONNECTED    = 3
};

// MQTT — estados de conexao (com erros negativos)
enum statusConnectMqtt_e
{
    MQTT_CONNECTION_TIMEOUT   = -4,
    MQTT_CONNECTION_LOST      = -3,
    MQTT_CONNECT_FAILED       = -2,
    MQTT_DISCONNECTED         = -1,
    MQTT_CONNECTED            =  0,
    MQTT_CONNECT_BAD_PROTOCOL =  1,
    MQTT_CONNECT_BAD_CLIENT   =  2,
    MQTT_CONNECT_UNAVAILABLE  =  3,
    MQTT_CONNECT_BAD_CRED     =  4,
    MQTT_CONNECT_UNAUTHORIZED =  5
};

// Queue — status de processamento
enum statusQueue_e
{
    NOT_STARTED       = 0,
    NO_PROCESS        = 1,
    WAIT_PROCESS      = 2,
    FAIL_DATA_COPY    = 3,
    SUCESS_TO_EXECUTE = 4
};
```

Regras:
- Sufixo `_e` no tipo
- Membros em `UPPER_SNAKE_CASE`
- Primeiro valor = estado indefinido/inicial (`= 0` ou `= -1` para erros)
- Valores explicitos para o primeiro membro e quando ha quebra de sequencia
- Usar com escopo: `operationMode_e::WIFI_CONNECT`

---

## 2. STRUCT DE CONTROLE — ESTADO + CONTEXTO

Agrupar o estado atual com flags, contadores e timestamps em uma struct `_t`.

```cpp
// Controle de conexao WiFi
typedef struct
{
    statusConnect_e statusConnect       = {statusConnect_e::STA_DISCONNECTED};
    uint32_t timeStartConnectWifi      = {NULL};
    uint8_t errorConnection            = {NULL};
    uint8_t failStartConnection        = {NULL};
    bool connectBackup                 = {false};
} controlConnect_t;

// Controle de canal dimmer
typedef struct
{
    uint8_t pinChannelDAC              = {NULL};
    uint8_t pinChannelReset            = {NULL};
    uint8_t dimmerPercent              = {NULL};
    uint8_t dimmerDAC                  = {NULL};
    uint32_t lastControlChannel        = {NULL};
    statusOperate_e operate            = {statusOperate_e::OPERATE_OFF};
    touchControlDimmer_e controlDimmer = {touchControlDimmer_e::DIMMER_NOP};
} controlChannel_t;

// Controle de touch
typedef struct
{
    bool statusTouch                   = {false};
    bool processTouch                  = {false};
    uint32_t timeProcessTouch          = {NULL};
} touchControl_t;
```

Regras:
- Sufixo `_t` no tipo
- Membros em `camelCase`
- Inicializacao com chaves `= {valor}`
- Enum inicializado com escopo: `statusConnect_e::STA_DISCONNECTED`
- Timestamps sempre como `uint32_t` (millis)
- Contadores de erro como `uint8_t` (economia de memoria)

---

## 3. TRANSICOES VIA SWITCH/CASE

Transicoes de estado usam `switch/case` com chaves em cada case e `default` obrigatorio.

```cpp
void wifiManager::setupModeOperation(operationMode_e operationMode)
{
    setOperationMode(operationMode);

    switch(operationMode)
    {
        case operationMode_e::WIFI_CONNECT:
        {
            activeConnectMode();
            i_LOG_MODULE("Modo CONNECT");
            break;
        }
        case operationMode_e::WIFI_DISTRIBUTE:
        {
            activeDistributeMode();
            i_LOG_MODULE("Modo DISTRIBUTE");
            break;
        }
        case operationMode_e::WIFI_PORTAL:
        {
            activeCaptureMode();
            i_LOG_MODULE("Modo PORTAL");
            break;
        }
        default:
        {
            e_LOG_MODULE("Modo de operacao nao definido");
            break;
        }
    }
}
```

Regras:
- Chaves `{}` em **cada** case — sem excecao
- `break;` obrigatorio em cada case
- `default:` obrigatorio com log de erro
- Logar transicoes em nivel `i_LOG` ou `d_LOG`

---

## 4. CONVERSAO ESTADO → STRING

Para logging e debug, cada enum de estado deve ter uma funcao de conversao:

```cpp
String managerLog::stateOfConnect(statusConnectMqtt_e stateOfConnection)
{
    String state;

    switch(stateOfConnection)
    {
        case statusConnectMqtt_e::MQTT_CONNECTION_TIMEOUT:
        {
            state = FPSTR("Timeout");
            break;
        }
        case statusConnectMqtt_e::MQTT_CONNECTION_LOST:
        {
            state = FPSTR("Perdeu a conexao");
            break;
        }
        case statusConnectMqtt_e::MQTT_DISCONNECTED:
        {
            state = FPSTR("Desconectou");
            break;
        }
        case statusConnectMqtt_e::MQTT_CONNECTED:
        {
            state = FPSTR("Conectado");
            break;
        }
        default:
        {
            state = FPSTR("Desconhecido");
            break;
        }
    }

    return state;
}
```

Regras:
- Usar `FPSTR()` para strings em flash (economia de RAM)
- `default` retorna string generica ("Desconhecido")
- Funcao retorna `String` — aceitavel aqui pois e chamada pontualmente para log

---

## 5. TIMEOUT — CONTROLE COM millis()

Timeouts usam comparacao de timestamps com `millis()`. **Nunca** usar `delay()`.

### 5.1 Padrao basico de timeout

```cpp
bool checkTimeout(uint32_t lastTime, uint32_t timeoutMs)
{
    bool expired = false;

    if((millis() - lastTime) > timeoutMs)
    {
        expired = true;
    }
    else
    {
        expired = false;
    }

    return expired;
}
```

### 5.2 Timeout em struct de controle

```cpp
bool queueControl::checkTimeout(inputData_t* reg)
{
    bool timeout = false;

    if(reg != nullptr)
    {
        if((millis() - reg->timeout) > _timeoutQueue)
        {
            timeout = true;
        }
        else
        {
            timeout = false;
        }
    }

    return timeout;
}
```

### 5.3 Timeout com envio periodico

```cpp
bool managerLog::checkTimeSendLogConnection(void)
{
    bool sendLogConnection = false;

    if((millis() - _lastSendLogConnection) > TIME_SEND_CONNECTION_LOG)
    {
        sendLogConnection = true;
    }
    else
    {
        sendLogConnection = false;
    }

    return sendLogConnection;
}
```

### 5.4 Timeout absoluto (deadline)

```cpp
// Calcular deadline uma vez, verificar depois
uint32_t queueDelay::calculeNewTimeExecute(uint16_t timeExecute)
{
    uint32_t newTimeExecute = NULL;

    newTimeExecute = millis() + timeExecute;

    return newTimeExecute;
}

// Verificar se deadline passou
bool queueDelay::checkProcessQueue(void)
{
    bool process = false;
    dataQueue_t* first = firstDataQueue();

    if(first != nullptr)
    {
        if(millis() >= first->timeToExecute)
        {
            process = true;
        }
    }

    return process;
}
```

Regras:
- Usar `millis()` — nunca `delay()`
- Constantes de tempo nomeadas: `TIME_RECONNECT`, `TIME_RESET_TOUCH`
- Timestamps armazenados como `uint32_t`
- Considerar rollover de `millis()` a cada ~49 dias (a subtracao `millis() - lastTime` funciona corretamente com unsigned overflow)

---

## 6. PERSISTENCIA DE ESTADO

Estados que devem sobreviver a reboot sao persistidos via `statusControl<T>`.

### 6.1 Carregar estado na inicializacao

```cpp
// No setup, apos carregar filesystem
statusLoad_e statusLoad = switchStatus->checkLoadStatus();

switch(statusLoad)
{
    case statusLoad_e::STATUS_LOAD_RESET:
    {
        // Primeira execucao ou reset — usar valores padrao
        initDefaultStatus();
        break;
    }
    case statusLoad_e::STATUS_LOAD_SUCESS:
    {
        // Estado carregado com sucesso
        i_LOG_MODULE("Status carregado");
        break;
    }
    case statusLoad_e::STATUS_LOAD_PRELOAD:
    {
        // Arquivo existe mas falhou ao carregar — usar preload
        w_LOG_MODULE("Usando preload");
        break;
    }
    default:
    {
        e_LOG_MODULE("Status desconhecido");
        break;
    }
}
```

### 6.2 Salvar estado apos mudanca

```cpp
// Apos qualquer mudanca de estado no hardware
void controlActiveRelay(channelExecute_e channel, uint8_t status)
{
    controlChannel[channel].stateRelay = status;
    digitalWrite(controlChannel[channel].pinRelay, status);
    switchStatus->setUpdateStatus();  // persiste imediatamente
}
```

### 6.3 Resetar estado

```cpp
// Reset completo — limpa struct e salva
template<class structData_t>
void statusControl<structData_t>::resetStatus(void)
{
    d_LOG_SWITCH_CONTROL("Resetando status");
    memset(_structData, NULL, sizeof(structData_t));
    setUpdateStatus();
}
```

Regras:
- Carregar estado no `setup()` — antes de configurar hardware
- Salvar estado **imediatamente** apos cada mudanca
- Reset via `memset` + save — nunca apagar arquivo diretamente
- `setUpdateStatus()` retorna `bool` — logar se falhar

---

## 7. RESET DE ESTADO

Funcoes dedicadas para resetar partes do estado:

```cpp
// Reset de contadores de erro
void wifiStation::resetErrorConnection(void)
{
    controlConnect.errorConnection = NULL;
    i_LOG_WIFI("Reset error connection");
}

// Reset de timestamp
void wifiStation::resetErrorStartConnection(void)
{
    controlConnect.failStartConnection = NULL;
}

// Reset completo de controle
void resetTouchControl(channelExecute_e channel)
{
    control.touchControl.statusTouch = false;
    control.touchControl.processTouch = false;
    control.touchControl.timeProcessTouch = NULL;
    control.channelControl.controlDimmer = touchControlDimmer_e::DIMMER_NOP;
}
```

Regras:
- Uma funcao de reset por grupo logico
- Resetar para os mesmos valores da inicializacao inline
- Logar o reset em nivel `i_LOG` ou `d_LOG`

---

## 8. TICKER — PROCESSAMENTO PERIODICO

Para avaliacoes periodicas de estado, usar `Ticker` em vez de verificacoes no `loop()`.

```cpp
// Declaracao em variables.h
Ticker* processSwitch = nullptr;

// Criacao em setupDeclaredClass()
processSwitch = new Ticker;

// Configuracao no setup
processSwitch->attach_ms(TIME_CHECK_TOUCH, verificaEntradas);

// Callback — deve ser rapido e nao-bloqueante
void verificaEntradas(void)
{
    // leitura de touch, botoes, sensores
    // NAO fazer I/O pesado (sem WiFi, sem SPIFFS, sem serial longo)
}
```

Regras:
- Callback do Ticker deve ser **rapido** (< 1ms)
- **Nunca** alocar memoria dentro do callback
- **Nunca** fazer I/O bloqueante (WiFi, file system, serial longo)
- Usar flags para sinalizar ao `loop()` que algo precisa ser processado
- Constante de intervalo nomeada: `TIME_CHECK_TOUCH`, `TIME_CHECK_TICKER`

---

## 9. PADRAO COMPLETO — EXEMPLO DE FSM

Exemplo de como combinar todos os elementos em um componente:

```cpp
// === principais.h ===
enum moduleState_e
{
    STATE_UNDEFINED = 0,
    STATE_IDLE      = 1,
    STATE_ACTIVE    = 2,
    STATE_ERROR     = 3
};

typedef struct
{
    moduleState_e state       = {moduleState_e::STATE_UNDEFINED};
    uint32_t lastStateChange  = {NULL};
    uint8_t errorCount        = {NULL};
    bool needsSync            = {false};
} moduleControl_t;

// === modulo.h ===
class myModule : public myModuleLOG
{
  public:
    myModule(dependency* dep);
    ~myModule();

    void setup(void);
    bool loop(void);

  protected:
    void checkDeleteDep(void);
    void setState(moduleState_e newState);
    void handleIdle(void);
    void handleActive(void);
    void handleError(void);
    void resetState(void);

  private:
    dependency* _dep = nullptr;
    moduleControl_t _control;
};

// === modulo.cpp ===
void myModule::setState(moduleState_e newState)
{
    if(_control.state != newState)
    {
        d_LOG_MODULE("Estado %d -> %d", _control.state, newState);
        _control.state = newState;
        _control.lastStateChange = millis();
    }
}

bool myModule::loop(void)
{
    bool resultado = false;

    if(_dep == nullptr)
    {
        resultado = false;
        e_LOG_MODULE("NullPointerException");
    }
    else
    {
        switch(_control.state)
        {
            case moduleState_e::STATE_IDLE:
            {
                handleIdle();
                resultado = true;
                break;
            }
            case moduleState_e::STATE_ACTIVE:
            {
                handleActive();
                resultado = true;
                break;
            }
            case moduleState_e::STATE_ERROR:
            {
                handleError();
                resultado = false;
                break;
            }
            default:
            {
                e_LOG_MODULE("Estado indefinido");
                resetState();
                resultado = false;
                break;
            }
        }
    }

    return resultado;
}

void myModule::resetState(void)
{
    _control.state = moduleState_e::STATE_IDLE;
    _control.lastStateChange = millis();
    _control.errorCount = NULL;
    _control.needsSync = false;
    i_LOG_MODULE("Estado resetado");
}
```

---

## 10. CHECKLIST DE MAQUINA DE ESTADO

- [ ] Estados definidos em `enum` com sufixo `_e` e valor inicial `UNDEFINED = 0`
- [ ] Membros do enum em `UPPER_SNAKE_CASE` com valores explicitos
- [ ] Struct de controle `_t` agrupa estado + timestamps + contadores + flags
- [ ] Transicoes via `switch/case` com chaves em cada case
- [ ] `default:` presente em todo switch com log de erro
- [ ] Transicoes logadas em nivel `d_LOG` ou `i_LOG`
- [ ] Timeouts com `millis()` — nunca `delay()`
- [ ] Constantes de tempo nomeadas (`TIME_RECONNECT`, `TIME_RESET`)
- [ ] Estado persistido via `statusControl` quando deve sobreviver a reboot
- [ ] `setUpdateStatus()` chamado imediatamente apos cada mudanca
- [ ] Funcao `resetState()` dedicada para retornar ao estado inicial
- [ ] Ticker para verificacoes periodicas — callback rapido e nao-bloqueante
- [ ] Funcao de conversao estado → string para logging
