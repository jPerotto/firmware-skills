---
name: firmware-error-handling
description: Tratamento de erros para firmware embarcado (ESP8266/ESP32, PlatformIO/Arduino). Aplique sempre que implementar retry, timeout, reconexao, recovery, degradacao graciosa, validacao de dados, ou qualquer mecanismo de resiliencia. Cobre NullPointerException, retry com fila, timeout com millis(), backoff adaptativo, reconnexao MQTT/WiFi, watchdog, backup/fallback e logging de erros.
---

# Tratamento de Erros — Firmware C++ (PlatformIO/Arduino)

Guia de boas praticas para tratamento de erros e resiliencia em firmware embarcado ESP8266/ESP32.

**Principio central:** firmware IoT roda 24/7 sem supervisao. Todo erro deve ser tratado,
logado e, quando possivel, recuperado automaticamente. O sistema nunca deve travar silenciosamente.

---

## 1. NullPointerException — PADRAO BASE

Todo acesso a ponteiro deve ser precedido de validacao. A mensagem padrao e
**sempre** `"NullPointerException"`.

```cpp
// PADRAO: verificacao antes de uso
if(_driver == nullptr)
{
    e_LOG_MODULE("NullPointerException");
}
else
{
    _driver->execute(channel, value);
}

// PADRAO: multiplos ponteiros agrupados
if((_mqttProtocol == nullptr) || (_subscribeMQTT == nullptr))
{
    e_LOG_MODULE("NullPointerException");
}
else
{
    // logica segura
}
```

Regras:
- Mensagem **sempre** `"NullPointerException"` — padrao do codebase
- Nivel **sempre** `e_LOG_X` (error)
- Verificacao em **todo** metodo que usa ponteiro membro
- Agrupamento de verificacoes em um unico `if` quando possivel

---

## 2. RETRY — TENTATIVA COM CONTADOR

Para operacoes que podem falhar pontualmente (envio MQTT, escrita em flash),
usar retry com contador limitado.

```cpp
// PADRAO: do-while com contador
bool mqttManager::sendPayload(const char* topic, const char* buf, bool retained)
{
    uint8_t waitSend = NULL;
    bool sendSucess = false;

    if((topic == nullptr) || (buf == nullptr) || (_mqttProtocol == nullptr))
    {
        e_LOG_MQTT("NullPointerException");
    }
    else
    {
        if(_mqttProtocol->state() == statusConnectMqtt_e::MQTT_CONNECTED)
        {
            do
            {
                waitSend++;
                sendSucess = _mqttProtocol->publish(topic, (uint8_t*)buf, strlen(buf), retained);
            } while((waitSend < RETRIES_ACK_MQTT) && (sendSucess == false));
        }
        else
        {
            sendSucess = false;
            w_LOG_MQTT("Broker desconectado");
        }
    }

    return sendSucess;
}
```

Regras:
- Contador de tentativas com limite via constante (`RETRIES_ACK_MQTT`)
- Verificar pre-condicao antes de tentar (ex: broker conectado)
- Logar resultado final se falhar
- **Nunca** retry infinito — sempre ter limite

---

## 3. RETRY COM FILA — EXECUCAO DIFERIDA

Para comandos que podem ser reagendados, usar `commandQueue` com delay.

```cpp
// PADRAO: enfileirar se ha delay ou se fila ja tem itens
void processCommand(DynamicJsonDocument& jsonDoc)
{
    uint16_t delayCommand = jsonDoc["delay"];

    if((delayCommand > NULL) || (queueCommand->checkProcessQueue() == true))
    {
        // Enfileirar para execucao posterior
        if(queueCommand->insertQueue(jsonDoc, delayCommand) == false)
        {
            e_LOG_MODULE("Falha ao adicionar a fila");
            logManager->sendLog("error: Falha ao adicionar a fila");
        }
    }
    else
    {
        // Executar imediatamente
        if(executeCommand(jsonDoc) == false)
        {
            e_LOG_MODULE("Falha de execucao");
            logManager->sendLog("error: Falha de execucao");
        }
        else
        {
            sendReturn(jsonDoc["id"], false);
            sendDataLog(jsonDoc);
        }
    }
}
```

### Processamento da fila no loop

```cpp
void checkQueueCommand(void)
{
    if(queueCommand == nullptr)
    {
        e_LOG_MODULE("NullPointerException");
    }
    else
    {
        if(queueCommand->checkProcessQueue() == true)
        {
            DynamicJsonDocument jsonDoc(BUFFER_MQTT_8KB);

            if(queueCommand->executeQueue(jsonDoc) == false)
            {
                w_LOG_MODULE("QUEUE %s", queueCommand->getError().toString().c_str());
                jsonDoc.garbageCollect();
                jsonDoc.shrinkToFit();
            }
            else
            {
                if(executeCommand(jsonDoc) == false)
                {
                    w_LOG_MODULE("error: Falha de execucao");
                    logManager->sendLog("error: Falha de execucao");
                }
                else
                {
                    const char* id = jsonDoc["id"];
                    sendReturn(id, false);
                    sendDataLog(jsonDoc);
                }
            }
        }
    }
}
```

Regras:
- Fila processada no `loop()` — verificar com `checkProcessQueue()`
- Erro de fila logado com `getError().toString()`
- Limpar `DynamicJsonDocument` com `garbageCollect()` + `shrinkToFit()` em caso de falha
- Fila com recuperacao: se `insertQueue` falha, limpar fila corrompida

---

## 4. TIMEOUT — RECONEXAO ADAPTATIVA

Reconexao com duas fases: rapida (agressiva) e lenta (backoff).

```cpp
// PADRAO: reconexao adaptativa (fast → slow)
bool mqttManager::checkTimeReconnect(void)
{
    bool reconnectBroker = false;

    // Fase 1: reconexao rapida
    if((_nReconnectBroker == NULL) || (_nReconnectBroker <= FORCE_RECONNECT_BROKER))
    {
        if((millis() - _lastConnectMQTT) > TIME_FORCE_RECONNECT_MQTT)
        {
            reconnectBroker = true;
            _lastConnectMQTT = millis();
            i_LOG_MQTT("Forcando reconexao %d/%d", _nReconnectBroker, FORCE_RECONNECT_BROKER);
        }
    }
    // Fase 2: reconexao lenta (backoff)
    else
    {
        if((millis() - _lastConnectMQTT) > _timeReconnect)
        {
            reconnectBroker = true;
            _lastConnectMQTT = millis();
            i_LOG_MQTT("Tentando reconexao lenta");
        }
    }

    return reconnectBroker;
}
```

### Controle de estado na desconexao

```cpp
void mqttManager::setStatusDisconnect(void)
{
    _nReconnectBroker++;
    _lastConnectMQTT = millis();
    _connectedBroker = false;
    _sendNewDevice = false;
    _subscribeMQTT->setStatus(false);

    // Apos muitas tentativas, limpar cache de conexao rapida
    if(_nReconnectBroker >= FORCE_RECONNECT_BROKER)
    {
        d_LOG_MQTT("Redefinindo quick MQTT");
        _quickBrokerMQTT->configured = false;
        _quickBrokerMQTT->domainIP = INADDR_NONE;
    }
}

void mqttManager::setStatusConnected(void)
{
    _connectedBroker = true;
    _sendNewDevice = false;
    _nReconnectBroker = NULL;
    _lastConnectMQTT = NULL;
}
```

Regras:
- Duas fases: rapida (`TIME_FORCE_RECONNECT_MQTT`) e lenta (`_timeReconnect`)
- Contador de tentativas (`_nReconnectBroker`) incrementado a cada falha
- Resetar contadores ao reconectar com sucesso
- Limpar cache de conexao rapida apos muitas falhas (forca nova resolucao DNS)

---

## 5. DEGRADACAO GRACIOSA

O sistema deve continuar operando mesmo com falhas parciais.

```cpp
// PADRAO: loop principal com degradacao
void loop(void)
{
    utils.checkWDT();        // Watchdog SEMPRE primeiro
    verificaEntradas();       // Hardware continua independente

    if(enablePortalCaptive == true)
    {
        enablePortalCaptive = false;
        logManager->sendLog("Entrando em modo AP");
        wifi->enablePortal();
    }

    if(wifi->loop() == true)  // So processa rede se WiFi OK
    {
        checkQueueCommand();  // Fila sempre processada

        bool statusBrokerLocal = checkProcessMQTTLocal();
        bool statusBrokerLog = checkProcessMQTTLog();

        // Sync completo so com AMBOS os brokers
        if((statusBrokerLocal == true) && (statusBrokerLog == true))
        {
            checkSyncControl();
        }
    }
    // Se WiFi falhar: hardware continua, touch continua,
    // fila acumula comandos, reconexao automatica
}
```

Niveis de degradacao:

| Componente falho | O que continua | O que para |
|-----------------|----------------|-----------|
| Broker remoto | Hardware, WiFi, broker local, fila | Logs remotos, OTA |
| Broker local | Hardware, WiFi, broker remoto, fila | Comandos MQTT |
| WiFi | Hardware, touch, fila local | Toda comunicacao |
| Filesystem | Hardware | Persistencia, config |

---

## 6. RECOVERY APOS RECONEXAO

Apos reconectar, reenviar todos os estados para sincronizar com o broker.

```cpp
// PADRAO: flag de resync + reenvio
bool checkProcessMQTTLocal(void)
{
    bool statusBroker = false;

    if(localMQTT->loop() == false)
    {
        statusBroker = false;
        sendStatesReconnectLocal = true;  // MARCA para resync
    }
    else
    {
        statusBroker = true;

        // Resync apos reconexao
        if((sendStatesReconnectLocal == true) && (localMQTT->statusBroker() == true))
        {
            if(sendReturn("", true) == true)  // true = reconnection
            {
                sendStatesReconnectLocal = false;  // Limpa flag
            }
        }
    }

    return statusBroker;
}
```

Regras:
- Flag `sendStatesReconnect[Broker]` por broker — declarada em `variables.h`
- Setar `true` quando broker desconecta
- Enviar todos os estados quando broker reconecta (`sendReturn("", true)`)
- Limpar flag somente apos envio confirmado
- Padrão identico para broker local e remoto

---

## 7. VALIDACAO DE JSON

Campos JSON devem ser validados antes de uso para prevenir crashes.

```cpp
// PADRAO: validar isNull + tipo antes de usar
JsonVariant cmd = jsonDoc["cmd"].as<JsonVariant>();

if((cmd.isNull() == true) || (cmd.is<const char*>() == false))
{
    e_LOG_MODULE("Comando invalido");
}
else
{
    processCmd(cmd);
}

// PADRAO: valor com fallback
bool configured = jsonConfig[JSON_PARAM_BROKER_CONFIG] | false;  // default false

// PADRAO: campo aninhado com validacao
JsonVariant controlIndex = jsonLog["command"]["channel"].as<JsonVariant>();

if(controlIndex.isNull() == true)
{
    channel = NULL;
}
else
{
    channel = controlIndex.as<uint8_t>() - OFFSET_CHANNEL;
}

// PADRAO: IP address com fallback
if(jsonConfig[JSON_PARAM_BROKER_IP].as<JsonVariant>().isNull() == true)
{
    _dataConfig->quickRemoteMQTT.domainIP = INADDR_NONE;
}
else
{
    _dataConfig->quickRemoteMQTT.domainIP = utils.stringToIPAddress(
        jsonConfig[JSON_PARAM_BROKER_IP].as<String>()
    );
}
```

Regras:
- **Sempre** `isNull()` antes de `as<T>()`
- Usar operador `|` para valores com fallback: `jsonDoc["key"] | defaultValue`
- Validar tipo com `is<T>()` quando o campo pode ter tipo errado
- Campos obrigatorios: logar erro se ausente
- Campos opcionais: usar fallback silenciosamente

---

## 8. WATCHDOG — ULTIMO RECURSO

O watchdog previne travamento total. Se o firmware parar de alimentar o WDT,
o ESP reinicia automaticamente.

```cpp
// PADRAO: WDT com auto-recuperacao
void watchDogUtils::checkWDT(void)
{
    if(_watchDog == nullptr)
    {
        e_LOG_UTILS("NullPointerException");
    }
    else if(_watchDog->state() == false)
    {
        _watchDog->setup();   // Reinicializar se nao esta ativo
        _watchDog->loop();    // Primeiro feed
    }
    else
    {
        _watchDog->loop();    // Feed normal
    }
}

// ESP32: reset com verificacao
void watchDogTimer::loop(void)
{
#if defined(ESP32)
    esp_err_t wdt_status = esp_task_wdt_reset();

    if(wdt_status != ESP_OK)
    {
        w_LOG_WDT("Falha ao resetar WDT %d", wdt_status);
        setup();  // Reinicializar WDT se reset falhou
    }
#elif defined(ESP8266)
    ESP.wdtFeed();
#endif
}
```

Regras:
- `utils.checkWDT()` **sempre** primeira linha do `loop()`
- WDT timeout: 30s (ESP32), 8s (ESP8266) — suficiente para operacoes normais
- Se WDT reset falha: reinicializar o WDT, nao ignorar
- WDT e a **ultima linha de defesa** — nao depender dele para logica normal

---

## 9. BACKUP E FALLBACK

### 9.1 Credenciais WiFi em EEPROM

```cpp
// Salvar credenciais bem-sucedidas como backup
bool backupSystem::setBackupCredentials(credentials_t* credentials)
{
    bool backupSucess = false;

    if(_backup == nullptr)
    {
        backupSucess = false;
        e_LOG_DATA("NullPointerException");
    }
    else
    {
        _backup->put(NULL, *credentials);

        if(_backup->commit() == false)
        {
            backupSucess = false;
            e_LOG_DATA("Fail commit");
        }
        else
        {
            backupSucess = true;
            i_LOG_DATA("Commit sucess");
        }
    }

    return backupSucess;
}
```

### 9.2 Quick Connect — IP cacheado

```cpp
// Usar IP cacheado quando disponivel, DNS como fallback
bool mqttManager::checkSetConnection(void)
{
    bool quickConnect = false;

    if(_quickBrokerMQTT->configured == false)
    {
        // Sem cache — resolver DNS
        if(resolveDNS() == false)
        {
            quickConnect = _mqttProtocol->setServer(_domain, _port);  // fallback domain
        }
        else
        {
            quickConnect = _mqttProtocol->setServer(_quickBrokerMQTT->domainIP, _port);
        }
    }
    else
    {
        // Cache disponivel — conexao rapida
        quickConnect = _mqttProtocol->setServer(_quickBrokerMQTT->domainIP, _port);
    }

    return quickConnect;
}
```

Regras:
- Credenciais backup em EEPROM — sobrevive a formatacao do SPIFFS
- Quick connect: IP do broker cacheado apos resolucao DNS bem-sucedida
- Limpar cache apos N falhas consecutivas (forcar nova resolucao)
- Fallback sempre disponivel: domain name quando IP falha

---

## 10. LOGGING DE ERROS — DUAL (LOCAL + REMOTO)

Erros devem ser logados **localmente** (serial) E **remotamente** (MQTT) quando possivel.

```cpp
// PADRAO: dual logging
if(executeCommand(jsonDoc) == false)
{
    w_LOG_MODULE("error: Falha de execucao");          // LOCAL — serial
    logManager->sendLog("error: Falha de execucao");   // REMOTO — MQTT
}
```

| Nivel | Macro | Quando usar | Enviar remoto? |
|-------|-------|-------------|----------------|
| ERROR | `e_LOG_X` | NullPointerException, falha critica | SIM |
| WARN | `w_LOG_X` | Falha recuperavel, degradacao | SIM |
| INFO | `i_LOG_X` | Transicoes, reconexoes, setup | NAO |
| DEBUG | `d_LOG_X` | Detalhes internos | NAO |
| VERBOSE | `v_LOG_X` | Trace detalhado | NAO |

Regras:
- `e_LOG` e `w_LOG` de erros operacionais: **sempre** acompanhar com `logManager->sendLog()`
- `i_LOG` de transicoes: apenas local (serial)
- Prefixar mensagem remota com `"error: "` para facilitar filtragem
- `logManager` pode estar null (WiFi desconectado) — verificar antes de chamar

---

## 11. STATUS DE ERRO COM ENUM

Para componentes com multiplos estados de erro, usar enum + conversao para string.

```cpp
// Definicao
enum statusQueue_e
{
    NOT_STARTED       = 0,
    NO_PROCESS        = 1,
    WAIT_PROCESS      = 2,
    FAIL_DATA_COPY    = 3,
    SUCESS_TO_EXECUTE = 4
};

// Conversao para log
String errorQueue::toString(void) const
{
    String error;

    switch(_statusQueue)
    {
        case statusQueue_e::NOT_STARTED:
        {
            error = FPSTR("Nao iniciada");
            break;
        }
        case statusQueue_e::FAIL_DATA_COPY:
        {
            error = FPSTR("Falha ao copiar os dados para a fila");
            break;
        }
        case statusQueue_e::SUCESS_TO_EXECUTE:
        {
            error = FPSTR("Sucesso ao executar a fila");
            break;
        }
        default:
        {
            error = FPSTR("Desconhecido");
            break;
        }
    }

    return error;
}

// Uso no caller
w_LOG_MODULE("QUEUE %s", queueCommand->getError().toString().c_str());
```

---

## 12. CHECKLIST DE TRATAMENTO DE ERROS

- [ ] Todo ponteiro verificado com `!= nullptr` antes de uso
- [ ] NullPointerException logado com `e_LOG_X("NullPointerException")`
- [ ] Retry com contador limitado (`RETRIES_ACK_MQTT`, etc.)
- [ ] Fila de comandos para execucao diferida com delay
- [ ] Timeout com `millis()` — nunca `delay()`
- [ ] Reconexao adaptativa: fase rapida + fase lenta (backoff)
- [ ] Contadores de erro resetados apos reconexao bem-sucedida
- [ ] Degradacao graciosa: hardware continua se comunicacao falhar
- [ ] Flag de resync por broker (`sendStatesReconnectLocal/Remote`)
- [ ] JSON validado com `isNull()` antes de `as<T>()`
- [ ] Watchdog alimentado na primeira linha do `loop()`
- [ ] WDT com auto-recuperacao se reset falhar
- [ ] Credenciais backup em EEPROM
- [ ] Quick connect com IP cacheado + fallback DNS
- [ ] Erros operacionais logados local E remoto
- [ ] Status de erro como enum com conversao para string
