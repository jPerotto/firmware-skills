---
name: firmware-safe-functions
description: Design de funcoes seguras para firmware embarcado (ESP8266/ESP32, PlatformIO/Arduino). Aplique sempre que criar, revisar ou refatorar qualquer funcao em .cpp ou .h. Cobre SRP, guard clauses, complexidade, aninhamento, tamanho, fluxo linear, retorno de bool, validacoes e limites de complexidade.
---

# Design de Funcoes Seguras — Firmware C++ (PlatformIO/Arduino)

Guia de boas praticas para escrita de funcoes seguras, legiveis e manteniveisem firmware embarcado ESP8266/ESP32.

**Principio central:** funcoes devem ser pequenas, previsíveis e com responsabilidade unica.
Se precisar da palavra "e" para descrever o que a funcao faz, ela deve ser dividida.

---

## 1. RESPONSABILIDADE UNICA (SRP)

Cada funcao faz **UMA coisa**. Se a funcao valida, processa e persiste — sao tres funcoes.

```cpp
// CORRETO — cada funcao com responsabilidade unica
void processCommand(DynamicJsonDocument& jsonDoc)
{
    if(validateCommand(jsonDoc) == false)
    {
        e_LOG_MODULE("Comando invalido");
    }
    else
    {
        executeCommand(jsonDoc);
        persistState();
    }
}

// INCORRETO — funcao faz tudo
void processCommand(DynamicJsonDocument& jsonDoc)
{
    // valida JSON, extrai campos, converte tipos,
    // executa comando, persiste estado, envia log,
    // atualiza MQTT — TUDO em 150 linhas
}
```

---

## 2. TAMANHO MAXIMO DE FUNCAO

| Faixa | Status | Acao |
|-------|--------|------|
| <= 60 linhas | IDEAL | Manter |
| 61-100 linhas | ACEITAVEL | Revisar se pode dividir |
| 101-120 linhas | ATENCAO | Dividir se possivel |
| > 120 linhas | OBRIGATORIO DIVIDIR | Extrair subfuncoes |

Contar linhas de codigo efetivo (sem comentarios, sem linhas em branco, sem chaves isoladas).

---

## 3. LIMITE DE ANINHAMENTO — MAXIMO 3 NIVEIS

Aninhamento profundo dificulta leitura e indica multiplas responsabilidades.

```cpp
// INCORRETO — 5 niveis de aninhamento
void countPulse(void)
{
    if((_startCapture == false) && (_recording == false))                // nivel 1
    {
        if((interval > START_THRESHOLD) && (interval < MAX_THRESHOLD))  // nivel 2
        {
            _startCapture = true;
        }
    }
    else
    {
        if(_recording == false)                                         // nivel 2
        {
            if(interval > START_THRESHOLD)                              // nivel 3
            {
                if(_totalPulses < MIN_PULSES)                           // nivel 4
                {
                    if(_totalPulses > SOME_VALUE)                       // nivel 5 !!
                    {
                        // logica enterrada
                    }
                }
            }
        }
    }
}

// CORRETO — extrair funcoes e usar early return
void countPulse(void)
{
    uint64_t interval = calculateInterval();

    if(isIdleState() == true)
    {
        handleIdlePulse(interval);
    }
    else if(_recording == false)
    {
        handleCapturePulse(interval);
    }
}

void handleCapturePulse(uint64_t interval)
{
    if(interval > START_THRESHOLD)
    {
        finalizeCapture();
    }
    else
    {
        appendPulse(interval);
    }
}
```

---

## 4. GUARD CLAUSES — ORDEM OBRIGATORIA

As validacoes no inicio da funcao seguem esta ordem fixa:

```
1. PONTEIROS / MEMORIA   → nullptr checks
2. PARAMETROS INVALIDOS   → range, tipo, formato
3. ESTADO DO SISTEMA      → conexao, modo de operacao
4. REGRAS DE NEGOCIO      → condicoes especificas do dominio
```

```cpp
// CORRETO — ordem de guard clauses
bool myModule::executeAction(uint8_t channel, uint8_t value)
{
    bool success = false;

    // 1. PONTEIROS
    if(_driver == nullptr)
    {
        success = false;
        e_LOG_MODULE("NullPointerException");
    }
    // 2. PARAMETROS
    else if(channel > MAX_CHANNELS)
    {
        success = false;
        w_LOG_MODULE("Canal invalido: %d", channel);
    }
    // 3. ESTADO
    else if(_operationMode == operationMode_e::UNDEFINED)
    {
        success = false;
        w_LOG_MODULE("Modo de operacao nao definido");
    }
    // 4. NEGOCIO
    else if(controlChannel[channel].active == false)
    {
        success = false;
        w_LOG_MODULE("Canal desativado");
    }
    else
    {
        // logica principal — sem aninhamento extra
        success = _driver->execute(channel, value);
    }

    return success;
}
```

---

## 5. LIMITE DE VALIDACOES — MAXIMO 5 POR FUNCAO

Se uma funcao precisa de mais de 5 validacoes no inicio, e um indicativo
de **quebra de SRP**. A funcao deve ser refatorada.

```cpp
// INCORRETO — 7 validacoes = funcao com responsabilidades demais
bool processData(data_t* data, config_t* config, driver_t* driver,
                 uint8_t channel, uint8_t mode, uint16_t timeout, bool force)
{
    if(data == nullptr) { /* ... */ }       // 1
    if(config == nullptr) { /* ... */ }     // 2
    if(driver == nullptr) { /* ... */ }     // 3
    if(channel > MAX) { /* ... */ }         // 4
    if(mode > MODE_MAX) { /* ... */ }       // 5
    if(timeout == NULL) { /* ... */ }       // 6
    if(force == true) { /* ... */ }         // 7
    // ...
}

// CORRETO — dividir responsabilidades
bool processData(processParams_t* params)
{
    if(params == nullptr)
    {
        e_LOG_MODULE("NullPointerException");
        return false; // early return permitido para nullptr critico
    }

    if(validateParams(params) == false)
    {
        return false;
    }

    return executeProcessing(params);
}
```

---

## 6. CADEIAS IF/ELSE IF — REGRAS

### 6.1 Else final obrigatorio

Toda cadeia `if` / `else if` deve terminar com um `else` final.
Isso garante que todos os caminhos sao tratados explicitamente — evita
falhas silenciosas onde nenhuma condicao e atendida.

```cpp
// CORRETO — else final explicito
if(limpa == true)
{
    result = command_e::LIMPA_BICO;
}
else if(sobe == true)
{
    result = command_e::SUBIR;
}
else
{
    result = command_e::IDLE;
}

// INCORRETO — sem else final
if(limpa == true)
{
    result = command_e::LIMPA_BICO;
}
else if(sobe == true)
{
    result = command_e::SUBIR;
}
// e se nenhum for true? falha silenciosa
```

### 6.2 Maximo 4 verificacoes

Cadeias longas de `if/else if` devem ser convertidas em `switch/case`,
tabela de decisao ou funcoes separadas.

```cpp
// INCORRETO — 6 if/else if
if(strcmp(type, "COMMAND") == NULL)
{
    executeCommand(jsonDoc);
}
else if(strcmp(type, "CONFIG") == NULL)
{
    applyConfig(jsonDoc);
}
else if(strcmp(type, "STATUS") == NULL)
{
    sendStatus(jsonDoc);
}
else if(strcmp(type, "RESET") == NULL)
{
    resetDevice();
}
else if(strcmp(type, "UPDATE") == NULL)
{
    startUpdate(jsonDoc);
}
else if(strcmp(type, "LOG") == NULL)
{
    sendLog(jsonDoc);
}

// CORRETO — switch ou funcao dedicada
void processMessage(const char* type, DynamicJsonDocument& jsonDoc)
{
    if(type == nullptr)
    {
        e_LOG_MODULE("NullPointerException");
    }
    else
    {
        dispatchCommand(type, jsonDoc);
    }
}
```

---

## 7. FLUXO LINEAR — VALIDACOES PRIMEIRO, LOGICA DEPOIS

Nunca misturar validacao com processamento. A funcao deve ter duas secoes claras:

```
1. SECAO DE VALIDACAO  — guard clauses, verificacoes
2. SECAO DE LOGICA     — processamento principal
```

```cpp
// CORRETO — fluxo linear
bool dataConfigJson::setDataConfigJson(fileSystem* systemData)
{
    bool setJsonSucess = true;

    // === VALIDACAO ===
    if(systemData == nullptr)
    {
        setJsonSucess = false;
        e_LOG_DATA("NullPointerException");
    }
    else
    {
        // === LOGICA PRINCIPAL ===
        DynamicJsonDocument configJson(jsonManager.getSizeJsonDataConfig());

        if(setJsonSucess == true)
        {
            setJsonSucess = setJsonOtaControl(configJson);
        }

        if(setJsonSucess == true)
        {
            setJsonSucess = setJsonCredentials(configJson);
        }

        if(setJsonSucess == true)
        {
            systemData->saveFile(FS_CONFIG_DATA_JSON, configJson);
        }
    }

    return setJsonSucess;
}

// INCORRETO — validacao misturada com logica
bool processAndSave(data_t* data)
{
    loadData();                        // logica
    if(data == nullptr) return false;  // validacao no meio
    processData(data);                 // logica
    if(!isValid(data)) return false;   // validacao no meio
    saveData(data);                    // logica
}
```

---

## 8. RETORNO DE FUNCOES — EARLY RETURN EM GUARD CLAUSES

Padrao hibrido baseado em MISRA (Advisory 15.5), CERT C e IEC 61508:

- **Guard clauses** (pre-condicoes) usam **early return** — falhar rapido antes de tocar em estado
- **Logica principal** usa **variavel local + unico return ao final**

A funcao tem duas secoes claras:
1. SECAO DE GUARD CLAUSES — early return para condicoes que impedem execucao
2. SECAO DE LOGICA — variavel local + unico return

### Funcoes bool

```cpp
// CORRETO — guard clauses com early return + logica com variavel local
bool backupSystem::begin(uint32_t size)
{
    // GUARD CLAUSES — pre-condicoes
    if(_backup == nullptr)
    {
        e_LOG_DATA("NullPointerException");
        return false;
    }

    // LOGICA PRINCIPAL — variavel local + unico return
    bool statusBegin = false;

    if(_backup->begin(size) == false)
    {
        statusBegin = false;
        e_LOG_DATA("failed to initialise backup");
    }
    else
    {
        statusBegin = true;
        i_LOG_DATA("Backup start sucess");
    }

    return statusBegin;
}

// INCORRETO — return espalhados na logica principal (nao sao guard clauses)
bool backupSystem::begin(uint32_t size)
{
    if(_backup == nullptr)
    {
        return false;
    }

    if(_backup->begin(size) == false)
    {
        return false;  // ERRADO — isso e logica, nao guard clause
    }

    return true;
}
```

### Funcoes void

```cpp
// CORRETO — early return em guard clause de nullptr
void myModule::setup(void)
{
    if(_dep == nullptr)
    {
        e_LOG_MODULE("NullPointerException");
        return;
    }

    setupInternal();
}
```

### O que e guard clause vs logica

| Tipo | Descricao | Early return |
|------|-----------|-------------|
| Guard clause | Validacao de pre-condicao (nullptr, parametro invalido, estado impossivel) | PERMITIDO |
| Logica principal | Processamento, decisoes de negocio, resultados | PROIBIDO — usar variavel local |

Justificativa tecnica (IEC 61508, ISO 26262, CERT C):
- Fail fast evita execucao com estado invalido (hard fault, watchdog reset)
- Reduz complexidade ciclomatica e aninhamento
- Guard clauses isoladas facilitam cobertura MC/DC
- Libera stack frame mais cedo em caminhos de erro

---

## 9. COMPARACAO EXPLICITA

Sempre usar comparacao explicita com booleanos. Nunca usar negacao implicita.

```cpp
// CORRETO — explicito
if(isConnected == false)
if(checkValid() == true)
if(statusBroker == false)

// INCORRETO — implicito
if(!isConnected)
if(checkValid())
if(!statusBroker)
```

Aplicar tambem em retornos de funcoes:

```cpp
// CORRETO
if(wifi->loop() == true)
{
    checkMQTT();
}

// INCORRETO
if(wifi->loop())
{
    checkMQTT();
}
```

---

## 10. PARENTESES EM CONDICOES COMPOSTAS

Cada sub-expressao deve estar entre parenteses, mesmo quando a precedencia
do operador tornaria desnecessario.

```cpp
// CORRETO — parenteses em cada verificacao
if((domain == nullptr) || (callback == nullptr))

if((channel > MAX_CHANNELS) && (channel != CHANNEL_NOP))

if((checkIPAddress(gatewayIP) == false) || (checkPort(port) == false) || (checkURI(uri) == false))

// INCORRETO — sem parenteses individuais
if(domain == nullptr || callback == nullptr)

if(channel > MAX_CHANNELS && channel != CHANNEL_NOP)
```

---

## 11. PARAMETROS DE FUNCAO

| Regra | Limite |
|-------|--------|
| Maximo de parametros | 5 |
| Acima de 5 | Agrupar em struct `_t` |
| Ponteiros | Sempre como primeiros parametros |
| Funcoes sem parametros | Usar `(void)` explicito |

```cpp
// CORRETO — ate 5 parametros, ponteiros primeiro
bool executeAction(driver_t* driver, config_t* config, uint8_t channel, uint8_t value)

// CORRETO — agrupamento quando muitos parametros
bool executeAction(actionParams_t* params)

// CORRETO — void explicito
void setup(void)
bool loop(void)
operationMode_e getMode(void)
```

---

## 12. NUMEROS MAGICOS — PROIBIDOS

Todo valor literal deve ser substituido por constante nomeada.

```cpp
// CORRETO
#define TIME_RESET_TOUCH     2000
#define TIME_START_DIMMER    500
#define MIN_DIMMER_VALUE     0
#define MAX_DIMMER_VALUE     100
constexpr uint8_t MAX_CHANNELS = 4;

if((millis() - lastControl) > TIME_RESET_TOUCH)
if(dimmerValue >= MIN_DIMMER_VALUE)

// INCORRETO
if((millis() - lastControl) > 2000)
if(dimmerValue >= 0)
```

**Excecao:** valores 0 e 1 em contextos matematicos obvios (incremento,
indice inicial, multiplicador neutro) nao precisam de constante.

---

## 13. FUNCOES VOID COM SETUP

Funcoes `void` que configuram o sistema nao retornam erro, mas devem
**logar internamente** qualquer falha.

```cpp
// CORRETO — void com logging interno
void myModule::setup(void)
{
    if(_dep == nullptr)
    {
        e_LOG_MODULE("NullPointerException");
    }
    else
    {
        setupInternal();
        i_LOG_MODULE("Setup concluido");
    }
}

// INCORRETO — void silencioso
void myModule::setup(void)
{
    setupInternal();  // se _dep for nullptr, crash silencioso
}
```

---

## 14. ESPACAMENTO ENTRE BLOCOS LOGICOS

Manter **uma linha em branco** entre blocos logicos distintos dentro da funcao.
Nao usar linha em branco dentro do mesmo bloco.

```cpp
// CORRETO
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
        resultado = checkCondition();
    }

    return resultado;
}

// INCORRETO — sem separacao entre declaracao, validacao e retorno
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
        resultado = checkCondition();
    }
    return resultado;
}
```

---

## 15. RESTRICOES DE LINGUAGEM

Evitar recursos que aumentam complexidade sem beneficio proporcional:

| Recurso | Status | Motivo |
|---------|--------|--------|
| `lambda` | EVITAR | Dificulta debug e rastreamento |
| Templates complexos | EVITAR | Usar apenas `statusControl<T>` e similares simples |
| `auto` | EVITAR | Tipo explicito melhora legibilidade |
| STL pesada (vector, map) | EVITAR | Fragmentacao de heap |
| Exceptions | PROIBIDO | Overhead inaceitavel em firmware |
| RTTI | PROIBIDO | Overhead de memoria |
| Sobrecarga de operador | EVITAR | Preferir funcoes explicitas |
| Heranca multinivel | MAX 1 NIVEL | Alem do mixin de log |
| Callbacks | PERMITIDO | Uso controlado para eventos (MQTT, touch, serial) |
| Heranca simples | PERMITIDO | Para mixin de log e interface |

---

## 16. CHECKLIST DE FUNCAO SEGURA

Antes de finalizar qualquer funcao:

- [ ] Funcao tem responsabilidade unica (SRP)
- [ ] Tamanho <= 100 linhas (ideal <= 60)
- [ ] Aninhamento <= 3 niveis
- [ ] Cadeia if/else if termina com `else` final obrigatorio
- [ ] Max 4 verificacoes if/else if em cadeia
- [ ] Max 5 validacoes no inicio (guard clauses)
- [ ] Guard clauses na ordem: nullptr → params → estado → negocio
- [ ] Fluxo linear: validacoes primeiro, logica depois
- [ ] Funcoes bool: early return em guard clauses + variavel local + unico `return` na logica principal
- [ ] Comparacoes explicitas: `== false`, `== true`, `== nullptr`
- [ ] Parenteses em cada sub-expressao condicional
- [ ] Sem numeros magicos (constexpr ou #define)
- [ ] Max 5 parametros (acima: usar struct)
- [ ] `(void)` explicito em funcoes sem parametros
- [ ] Erros logados com macro de log (`e_LOG_X`, `w_LOG_X`)
- [ ] Sem `delay()` no loop principal (usar `millis()`)
