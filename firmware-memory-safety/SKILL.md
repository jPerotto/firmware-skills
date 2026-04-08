---
name: firmware-memory-safety
description: Seguranca de memoria para firmware embarcado (ESP8266/ESP32, PlatformIO/Arduino). Aplique sempre que criar classes com ponteiros, alocar memoria dinamica, gerenciar lifecycle de objetos, ou revisar ownership de recursos. Cobre ownership, construtores, destrutores, checkDelete, inicializacao, stack vs heap, DynamicJsonDocument e strings.
---

# Seguranca de Memoria — Firmware C++ (PlatformIO/Arduino)

Guia de boas praticas para gerenciamento de memoria em firmware embarcado ESP8266/ESP32.

**Principio central:** todo ponteiro tem um dono. Quem cria, destroi.
Em sistemas de longa execucao (IoT 24/7), um unico memory leak e uma bomba-relogio.

---

## 1. OWNERSHIP — REGRA FUNDAMENTAL

Cada ponteiro alocado com `new` tem **UM unico dono** — a classe que o criou.
O dono e responsavel por:
- Alocar (`new`) no construtor ou `setup()`
- Validar (`!= nullptr`) antes de qualquer uso
- Liberar (`delete` + `= nullptr`) no destrutor via `checkDelete[X]()`

```
REGRA: quem faz new, faz delete
REGRA: quem recebe ponteiro por parametro, NAO faz delete
REGRA: ponteiro recebido = dependencia injetada = nao e seu
```

```cpp
// CORRETO — ownership claro
class myModule
{
  public:
    myModule(dataManager* dm);  // dm = injetado, NAO e nosso
    ~myModule();

  protected:
    void checkDeleteDriver(void);  // _driver = nosso, criamos com new

  private:
    dataManager* _dm = nullptr;       // INJETADO — nao deletar
    myDriver* _driver = nullptr;      // PROPRIO — deletar no destrutor
};

myModule::myModule(dataManager* dm)
{
    if(dm == nullptr)
    {
        e_LOG_MODULE("NullPointerException");
    }
    else
    {
        _dm = dm;                    // apenas armazena referencia
        _driver = new myDriver;     // criamos — somos donos

        if(_driver == nullptr)
        {
            e_LOG_MODULE("NullPointerException");
        }
    }
}

myModule::~myModule()
{
    checkDeleteDriver();    // deleta apenas o que criamos
    // NAO deletar _dm — nao e nosso
}
```

---

## 2. CONSTRUTOR — PADRAO OBRIGATORIO

Todo construtor que recebe ponteiros segue este fluxo:

```
1. Verificar TODOS os ponteiros recebidos em um unico if
2. Se null: logar "NullPointerException" e parar
3. Se valido: atribuir aos membros _
4. Criar objetos internos com new
5. Verificar CADA new contra nullptr
```

```cpp
// PADRAO: construtor com validacao completa
mqttManager::mqttManager(const char* domain, MQTT_CALLBACK_SIGNATURE)
{
    if((domain == nullptr) || (callback == nullptr))
    {
        e_LOG_MQTT("NullPointerException");
    }
    else
    {
        _domain = domain;
        _port = MQTT_BROKER_PORT;
        _mqttProtocol = new mqttProtocol(_domain, _port, callback);
        _subscribeMQTT = new mqttSubscribe(_mqttProtocol);

        if((_mqttProtocol == nullptr) || (_subscribeMQTT == nullptr))
        {
            e_LOG_MQTT("NullPointerException");
        }
    }
}
```

Regras do construtor:

| Regra | Descricao |
|-------|-----------|
| Validacao agrupada | `if((p1 == nullptr) \|\| (p2 == nullptr))` — um unico if |
| Mensagem padrao | `e_LOG_X("NullPointerException")` — sempre esta string |
| Sem init list | Usar corpo do construtor — permite validacao antes de atribuicao |
| Ordem de atribuicao | Parametros injetados primeiro, `new` depois |
| Check apos new | Agrupar todos os new e verificar juntos |

---

## 3. DESTRUTOR — PADRAO OBRIGATORIO

O destrutor **apenas chama** funcoes `checkDelete[X]()`. Sem logica, sem verificacoes,
sem nada mais.

```cpp
// CORRETO — destrutor limpo
mqttProtocol::~mqttProtocol()
{
    checkDeleteBuffer();
    checkDeleteWiFiClient();
}

// CORRETO — destrutor com multiplos checkDelete
ERC2102::~ERC2102()
{
    checkDeleteInterface();
    checkDeleteQueueControl();
    checkDeleteQueueDelay();
}

// INCORRETO — logica no destrutor
myModule::~myModule()
{
    if(_driver != nullptr)   // NAO — isso vai no checkDelete
    {
        _driver->shutdown();
        delete _driver;
        _driver = nullptr;
    }
}
```

---

## 4. PADRAO checkDelete — UMA FUNCAO POR PONTEIRO

Cada ponteiro membro alocado com `new` tem **sua propria** funcao
`checkDelete[NomeMembro]()` em `protected:`.

```cpp
// PADRAO: checkDelete para cada ponteiro
void mqttProtocol::checkDeleteBuffer(void)
{
    if(_buffer != nullptr)
    {
        free(_buffer);
        _buffer = nullptr;
    }
}

void mqttProtocol::checkDeleteWiFiClient(void)
{
    if(_wifiClient != nullptr)
    {
        delete _wifiClient;
        _wifiClient = nullptr;
    }
}

void dataManager::checkDeleteJson(void)
{
    if(_configJson != nullptr)
    {
        delete _configJson;
        _configJson = nullptr;
    }
}
```

Regras do checkDelete:

| Regra | Descricao |
|-------|-----------|
| Nome | `checkDelete` + nome do membro (sem `_` prefixo): `checkDeleteDriver()` |
| Visibilidade | Sempre em `protected:` |
| Corpo | APENAS: `if != nullptr` → `delete` → `= nullptr` |
| Sem logica extra | Sem logs, sem efeitos colaterais, sem chamadas |
| `free()` vs `delete` | Usar `free()` para `malloc()`, `delete` para `new` — nunca misturar |
| Apos delete | **Sempre** atribuir `= nullptr` — previne double-free |

---

## 5. INICIALIZACAO DE MEMBROS PRIVADOS

Todo membro privado **deve ser inicializado inline** na declaracao:

```cpp
private:
    /**
     * @brief PONTEIRO PARA O PROTOCOLO MQTT
     */
    mqttProtocol* _mqttProtocol = nullptr;

    /**
     * @brief PONTEIRO PARA RESOLUCAO mDNS
     */
    resolverMDNS* _resolverMDNS = nullptr;

    /**
     * @brief PORTA DO BROKER
     */
    uint16_t _port = NULL;

    /**
     * @brief CONTADOR DE RECONEXOES
     */
    uint8_t _nReconnectBroker = NULL;

    /**
     * @brief TIMESTAMP DA ULTIMA CONEXAO
     */
    uint32_t _lastConnectMQTT = NULL;

    /**
     * @brief FLAG DE CONEXAO COM BROKER
     */
    bool _connectedBroker = false;

    /**
     * @brief ESTADO DE CONEXAO
     */
    statusConnect_e _state = statusConnect_e::DISCONNECTED;
```

| Tipo | Valor inicial | Exemplo |
|------|---------------|---------|
| Ponteiros | `= nullptr` | `myDriver* _driver = nullptr` |
| Tipos numericos | `= NULL` | `uint32_t _counter = NULL` |
| Booleanos | `= false` | `bool _active = false` |
| Enums | `= enum_e::VALOR_INICIAL` | `operationMode_e::UNDEFINED` |
| Strings (const char*) | `= nullptr` | `const char* _domain = nullptr` |
| Structs | `= {valores}` | `controlConnect_t _control = {}` |

**Nota:** `NULL` para tipos numericos e convencao do codebase. Manter por consistencia.

---

## 6. VERIFICACAO DE PONTEIROS EM METODOS

Antes de usar qualquer ponteiro membro, verificar se e valido.

### 6.1 Multiplos ponteiros — verificacao agrupada

```cpp
if((_driver == nullptr) || (_dataManager == nullptr) || (_dataConfig == nullptr))
{
    e_LOG_MODULE("NullPointerException");
}
else
{
    // logica segura — todos os ponteiros validos
}
```

### 6.2 Ponteiro unico com acao de recuperacao

```cpp
if(_connection == nullptr)
{
    status = false;
    resetManager();
}
else
{
    status = _connection->loop();
}
```

### 6.3 Ponteiro ja existe — reconfigurar

```cpp
if(_connection != nullptr)
{
    w_LOG_MODULE("Reconfigurar conexao");
    checkDeleteConnection();
}

_connection = new connectionHandler(_driver, _dataManager);

if(_connection == nullptr)
{
    e_LOG_MODULE("NullPointerException");
}
else
{
    _connection->setup(getOperationMode());
}
```

---

## 7. ALOCACAO CENTRALIZADA — setupDeclaredClass()

Nos firmwares, toda alocacao de objetos globais e centralizada em
`setupDeclaredClass()`, chamada uma unica vez no `setup()`.

```cpp
// CORRETO — alocacao centralizada, ordem de dependencia
void setupDeclaredClass(void)
{
    // 1. Base — sem dependencias
    systemData    = new fileSystem(&SPIFFS);
    dataConfig    = new dataConfig_t;

    // 2. Dados — depende de base
    saveConfig    = new dataManager(systemData, dataConfig);

    // 3. Comunicacao — depende de dados
    wifi          = new wifiManager(saveConfig, dataConfig);
    mqttLocal     = new mqttManager(MQTT_SERVER_LOCAL, processLocalMqtt);
    mqttRemote    = new mqttManager(MQTT_SERVER_REMOTE, processRemoteMqtt);

    // 4. Servicos — depende de comunicacao
    logSync       = new syncLog(mqttRemote, systemData);
    logManager    = new managerLog(logSync);
}

// INCORRETO — alocacao dispersa em varios arquivos
void setupWiFi(void)
{
    wifi = new wifiManager(saveConfig, dataConfig);  // new fora do ponto central
}
```

Regras:
- **Uma unica funcao** centraliza todos os `new` de ponteiros globais
- Ordem de criacao respeita **dependencias** (objetos base antes dos que os consomem)
- Todo ponteiro em `variables.h` com `= nullptr` e inicializado aqui
- Callbacks passados no construtor quando possivel

---

## 8. STACK vs HEAP

| Usar Stack | Usar Heap |
|-----------|-----------|
| Variaveis locais de funcao | Objetos que vivem alem do escopo |
| Structs pequenas (<256 bytes) | Buffers grandes |
| Resultados temporarios | Objetos globais do firmware |
| Contadores, flags, timestamps | Objetos com lifecycle gerenciado |

```cpp
// STACK — preferir para variaveis locais
void processCommand(DynamicJsonDocument& jsonDoc)
{
    bool success = false;                        // stack
    uint8_t channel = jsonDoc["channel"];         // stack
    channelSetup_t setup = {channel, PIN_RELAY};  // stack — struct pequena
    // ...
}

// HEAP — apenas quando necessario
void setupDeclaredClass(void)
{
    wifi = new wifiManager(saveConfig, dataConfig);  // heap — lifecycle global
}
```

**Regra:** se nao precisa de `new`, nao use `new`.

---

## 9. DynamicJsonDocument — USO SEGURO

O `DynamicJsonDocument` aloca no heap. Usar com cuidado.

### 9.1 Sempre com tamanho explicito

```cpp
// CORRETO — tamanho explicito e controlado
DynamicJsonDocument configJson(jsonManager.getSizeJsonDataConfig());
DynamicJsonDocument jsonDoc(BUFFER_MQTT_8KB);
DynamicJsonDocument jsonLog(BUFFER_MQTT_1KB);

// INCORRETO — tamanho magico
DynamicJsonDocument doc(4096);
DynamicJsonDocument doc(2048);
```

### 9.2 Limpeza apos uso

```cpp
// CORRETO — cleanup explicito
DynamicJsonDocument jsonDoc(BUFFER_MQTT_8KB);

if(queueCommand->executeQueue(jsonDoc) == false)
{
    w_LOG_MODULE("QUEUE %s", queueCommand->getError().toString().c_str());
    jsonDoc.garbageCollect();
    jsonDoc.shrinkToFit();
}
```

### 9.3 Validacao de campos antes de uso

```cpp
// CORRETO — validar tipo e null antes de usar
JsonVariant cmd = jsonDoc["cmd"].as<JsonVariant>();

if((cmd.isNull() == true) || (cmd.is<const char*>() == false))
{
    e_LOG_MODULE("Comando invalido");
}
else
{
    processCmd(cmd);
}

// INCORRETO — usar campo sem validar
const char* cmd = jsonDoc["cmd"];  // crash se null
processCmd(cmd);
```

---

## 10. STRINGS — CONSTANTES ESTATICAS

Usar `constexpr const char*` para strings que nao mudam. Evitar `String`
do Arduino quando possivel.

```cpp
// CORRETO — strings estaticas
constexpr const char* FS_CONFIG_DATA_JSON = "/config.json";
constexpr const char* JSON_PARAM_OTA_RETAINED = "otaRetained";
#define MQTT_SERVER_LOCAL "mqtt.local"

// EVITAR — String do Arduino em loops ou funcoes chamadas frequentemente
void checkTopic(void)
{
    String topic = String(MQTT_PREFIX) + "/" + deviceId;  // aloca no heap a cada chamada
}

// PREFERIR — const char* com strcmp
void checkTopic(const char* topic)
{
    if(strcmp(topic, expectedTopic) == NULL)
    {
        // processar
    }
}
```

| Contexto | Recomendacao |
|----------|-------------|
| Constantes de path, nome, topico | `constexpr const char*` ou `#define` |
| Comparacao de strings | `strcmp()` com `const char*` |
| Concatenacao pontual (setup) | `String` do Arduino e aceitavel |
| Concatenacao em loop | PROIBIDO — usar buffer fixo ou `snprintf` |
| Logs | Strings literais em macros (ja estaticas) |

---

## 11. FUNCTION POINTERS E CALLBACKS

Function pointers devem seguir o padrao de callback existente no codebase:

```cpp
// PADRAO: definicao do callback como macro
#define MQTT_CALLBACK_SIGNATURE std::function<void(const char*, DynamicJsonDocument&)> callback
#define INTERFACE_PROTOCOL_CALLBACK std::function<void(dataInterface_t*)> callback

// PADRAO: armazenamento como membro
class mqttProtocol
{
  private:
    MQTT_CALLBACK_SIGNATURE;  // expande para std::function<...> callback
};

// PADRAO: passagem no construtor
mqttManager::mqttManager(const char* domain, MQTT_CALLBACK_SIGNATURE)
{
    // ...
    _mqttProtocol = new mqttProtocol(_domain, _port, callback);
}
```

Regras:
- Callbacks passados no construtor e armazenados como membro
- Validar callback `!= nullptr` antes de invocar
- Callbacks definidos como `std::function` (excecao controlada ao veto de STL)
- Usar `std::bind` quando necessario para metodos de classe

---

## 12. PREVENCAO DE PROBLEMAS COMUNS

### 12.1 Double-free

```cpp
// CORRETO — nullptr apos delete previne double-free
void checkDeleteDriver(void)
{
    if(_driver != nullptr)
    {
        delete _driver;
        _driver = nullptr;  // OBRIGATORIO
    }
}

// INCORRETO — sem nullptr apos delete
void checkDeleteDriver(void)
{
    if(_driver != nullptr)
    {
        delete _driver;
        // _driver ainda aponta para memoria liberada!
    }
}
```

### 12.2 Use-after-free

```cpp
// CORRETO — verificar antes de usar
if(_driver != nullptr)
{
    _driver->execute(channel, value);
}

// INCORRETO — usar sem verificar
_driver->execute(channel, value);  // crash se _driver foi deletado
```

### 12.3 Memory leak

```cpp
// CORRETO — delete antes de reatribuir
if(_connection != nullptr)
{
    checkDeleteConnection();
}
_connection = new connectionHandler();

// INCORRETO — reatribuir sem deletar
_connection = new connectionHandler();  // leak do objeto anterior
```

### 12.4 free() vs delete

```cpp
// CORRETO — usar free para malloc, delete para new
void* buffer = malloc(size);
free(buffer);

myClass* obj = new myClass();
delete obj;

// INCORRETO — misturar
void* buffer = malloc(size);
delete buffer;  // comportamento indefinido

myClass* obj = new myClass();
free(obj);  // comportamento indefinido
```

---

## 13. CLASSES SEM PONTEIROS PROPRIOS

Classes simples sem alocacao dinamica tem construtor/destrutor minimos:

```cpp
// Classe sem ponteiros proprios — destrutor vazio e aceitavel
class myTimer : public myTimerLOG
{
  public:
    myTimer();
    ~myTimer() {}

    void start(uint32_t durationMs);
    bool check(void);
    void reset(void);

  private:
    uint32_t _startTime = NULL;
    uint32_t _duration = NULL;
    bool _active = false;
};

myTimer::myTimer() {}
```

Nao criar `checkDelete` para classes que nao alocam nada.

---

## 14. CHECKLIST DE SEGURANCA DE MEMORIA

Antes de finalizar qualquer classe:

- [ ] Ownership claro: quem cria (`new`), destroi (`delete`)
- [ ] Ponteiros injetados NO construtor: validados mas NAO deletados no destrutor
- [ ] Ponteiros criados com `new`: tem `checkDelete[X]()` em `protected:`
- [ ] Construtor valida TODOS os ponteiros recebidos em um unico `if`
- [ ] Apos cada `new`: verificacao de `nullptr`
- [ ] Destrutor chama APENAS `checkDelete[X]()` — sem logica
- [ ] Cada `checkDelete`: `if != nullptr` → `delete/free` → `= nullptr`
- [ ] Membros privados inicializados inline: ponteiros `= nullptr`, numericos `= NULL`, bool `= false`
- [ ] Sem `new` disperso — centralizado em construtor ou `setupDeclaredClass()`
- [ ] `DynamicJsonDocument` com tamanho explicito (constante nomeada)
- [ ] Campos JSON validados antes de uso (`isNull()`, `is<T>()`)
- [ ] Strings constantes como `constexpr const char*` ou `#define`
- [ ] Sem `String` do Arduino em loops ou funcoes chamadas frequentemente
- [ ] `free()` para `malloc()`, `delete` para `new` — nunca misturar
- [ ] Sem reatribuicao de ponteiro sem deletar o anterior
