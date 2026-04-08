---
name: firmware-doxygen-review
description: Padrao de documentacao Doxygen/Markdown para firmware embarcado (ESP8266/ESP32, PlatformIO/Arduino). Equilibra clareza e concisao — documenta apenas onde agrega valor. Use quando criar, revisar ou auditar documentacao em .h, .cpp, README.md, enums, structs ou tabelas de hardware. Reprova documentacao redundante, verbosa ou que descreve o obvio. Prioriza restricoes, riscos e contexto de uso.
---

# Documentacao — Padrao Doxygen/Markdown para Firmware Embarcado

Padrao de documentacao para projetos firmware ESP8266/ESP32 com PlatformIO.
Idioma: Portugues brasileiro. Doxygen em sentence case, sem acentos. `@warning` em MAIUSCULAS.

**Principio central:** documentacao existe para agregar valor, nao para preencher espaco.
Se o nome, tipo e contexto ja explicam o elemento, nao documente. Se ha restricao,
risco ou contexto nao obvio, documente de forma concisa.

---

## 1. README.md — FIRMWARE (TEMPLATE)

Cada firmware tem um README.md no formato Doxygen `@page`:

```markdown
@page subsystem{N} {firmware-name}
@tableofcontents

# Titulo do Dispositivo

Descricao do proposito do dispositivo e quais bibliotecas utiliza.

Utiliza as bibliotecas [library-a](#subsystemX), [library-b](#subsystemY)
e [library-c](#subsystemZ) para [descricao da funcionalidade].

@section subsystem{N}section1 Firmware

@subsection subsystem{N}subsection1 principais
O arquivo [principais.h](#firmware-name/principais.h) contem todas as
bibliotecas necessarias para o funcionamento do firmware.

@subsection subsystem{N}subsection2 variaveis
O arquivo [variaveis.h](#firmware-name/variaveis.h) contem todas as
variaveis e instancias de classe utilizadas no firmware.

@subsection subsystem{N}subsection3 Setup
O arquivo [main.cpp](#firmware-name/main.cpp) contem a funcao `setup()`
que inicializa todos os componentes do firmware.

@subsection subsystem{N}subsection4 Loop
O arquivo [loop.cpp](#firmware-name/loop.cpp) contem a funcao `loop()`
principal do firmware.

@subsection subsystem{N}subsection5 MQTT
O arquivo [mqtt.cpp](#firmware-name/mqtt.cpp) contem as funcoes de
configuracao e conexao com os brokers MQTT.

@section subsystem{N}section2 Hardware

@subsection subsystem{N}subsection6 Configuracao de Build

| CONFIG | VALUE |
| :----: | :----: |
| MODULO | <MODULE_NAME> |
| MCU | <MCU_TYPE> |
| F_cpu | 160MHz |
| Flash mode | DIO |
| Flash size | 4MB |
| File System | SPIFFS |

@subsection subsystem{N}subsection7 PINOUT

| CANAL | GPIO | FUNCAO |
| :---- | :---- | :---- |
| 1 | 12 | RELAY |
| 2 | 14 | LED |
```

### Regras do README de firmware:

- Sempre comecar com `@page subsystem{N} {firmware-name}`
- Sempre incluir `@tableofcontents`
- Secao 1 = Firmware (arquivos do projeto, um @subsection por arquivo)
- Secao 2 = Hardware (build config + pinout)
- Tabelas de build config usam `| :----: |` (centralizado)
- Tabelas de pinout usam `| :---- |` (alinhado a esquerda)
- Revisoes de hardware usam `^` para agrupar linhas sob a mesma revisao
- Links para arquivos: `[arquivo.h](#firmware-name/arquivo.h)`
- Links para bibliotecas: `[library-name](#subsystemX)`

---

## 2. README.md — BIBLIOTECA (TEMPLATE)

```markdown
@page subsystem{N} {library-name}
@tableofcontents

# Titulo da Biblioteca

Descricao do proposito da biblioteca.

@see className

@section subsystem{N}section1 Dependencias

* [firmware-config](#subsystem1) - Biblioteca base de configuracoes e definicoes
* [debug-log](#subsystem2) - Biblioteca para envio de LOGs via serial
* [utils](#subsystem3) - Biblioteca para diversas utilidades

@subsection subsystem{N}subsection1 Principais recursos

* [setup](#className::setup()) - CONFIGURACAO INICIAL DA BIBLIOTECA
* [loop](#className::loop()) - FUNCAO PRINCIPAL DE PROCESSAMENTO

@subsection subsystem{N}subsection2 Codigos de exemplos

* [BasicUsage](#library-name/examples/BasicUsage/BasicUsage.ino)
```

### Regras do README de biblioteca:

- Sempre comecar com `@page subsystem{N} {library-name}`
- `@see className` logo apos a descricao (referencia a classe principal)
- Secao 1 = Dependencias (lista com bullet points e links)
- Subsecao 1 = Principais recursos (lista de metodos da classe com links)
- Subsecao 2 = Codigos de exemplos (links para .ino de exemplo)
- Descricoes de metodos em sentence case, sem acentos
- Links para metodos: `[nome](#className::methodName())`

---

## 3. HEADER FILE (.h) — DOCUMENTACAO INLINE

### 3.1 Cabecalho do arquivo

Todo arquivo `.h` comeca com header guard + bloco Doxygen:

```cpp
#ifndef _CLASSNAME_H
#define _CLASSNAME_H

/**
 * @file filename.h
 * @author <AUTHOR_NAME> (<AUTHOR_EMAIL>)
 * @brief Descricao do proposito do arquivo em sentence case,
 *        sem acentos, pode ocupar multiplas linhas
 * @version X.Y.Z
 * @date DD-MM-YYYY
 * @copyright Copyright (c) YYYY \b <COMPANY_NAME>
 */
```

### 3.2 Includes documentados

Cada `#include` tem um `@brief` explicando o que a biblioteca faz:

```cpp
/**
 * @brief BIBLIOTECA BASE DE CONFIGURACOES E DEFINICOES
 */
#include <firmware-config.h>

/**
 * @brief BIBLIOTECA PARA DIVERSAS UTILIDADES
 */
#include <utils.h>

/**
 * @brief BIBLIOTECA PARA LOGS DO MODULO
 */
#include "log-module.h"
```

### 3.3 Classe documentada — membros privados

Documentar membros privados **apenas quando o nome + tipo nao sao suficientes**.
Agrupar membros relacionados sob um unico bloco de comentario para reduzir poluicao visual.

```cpp
class myModule : public myModuleLOG
{
  public:
    myModule(dependency* dep);
    ~myModule();

    void setup(void);
    bool loop(void);
    bool executeAction(uint8_t param);

  protected:
    void checkDeleteSubComponent(void);

  private:
    dependency* _dep = nullptr;
    operationMode_e _mode = operationMode_e::UNDEFINED;
    bool _status = false;

    /**
     * @brief TEMPO LIMITE PARA REENVIO APOS FALHA DE COMUNICACAO
     * @note VALOR EM MILISSEGUNDOS. ZERO DESABILITA O REENVIO
     */
    uint32_t _retryTimeout = NULL;
};
```

**Quando documentar membros privados:**
- Ponteiros cujo **proposito** nao e obvio pelo tipo/nome
- Variaveis com unidades nao evidentes (ms, us, bytes, dB)
- Flags ou contadores cujo significado depende de contexto externo
- Membros com restricoes de valor ou efeitos colaterais

**Quando NAO documentar:**
- `dependency* _dep` — tipo + nome ja explicam
- `bool _status` — obvio
- `operationMode_e _mode` — o enum ja descreve o dominio

### Regras do header:

- Membros `public` e `protected` NAO precisam de @brief no .h (documentados no .cpp)
- `@brief` em sentence case, sem acentos
- Ponteiros inicializados com `= nullptr`
- Tipos primitivos inicializados com `= {NULL}`, `= {false}`, `= {0}`
- **Preferir ausencia de documentacao a documentacao redundante**

---

## 4. SOURCE FILE (.cpp) — DOCUMENTACAO INLINE

### 4.1 Cabecalho do .cpp

```cpp
#include "my-module.h"
```

Arquivos `.cpp` NAO tem bloco `@file/@author/@version`. Apenas o `#include`.

### 4.2 Construtor

```cpp
/**
 * @brief CONSTRUTOR DA CLASSE DO MODULO
 * @param dep PONTEIRO PARA A DEPENDENCIA EXTERNA
 */
myModule::myModule(dependency* dep)
{
    if(dep == nullptr)
    {
        e_LOG_MYMODULE("NullPointerException");
    }
    else
    {
        _dep = dep;
    }
}
```

### 4.3 Destrutor

```cpp
/**
 * @brief DESTRUTOR DA CLASSE DO MODULO
 */
myModule::~myModule()
{
    checkDeleteSubComponent();
}
```

### 4.4 Funcao void

```cpp
/**
 * @brief INICIA A CONFIGURACAO DO MODULO
 */
void myModule::setup(void)
{
    // implementacao
}
```

### 4.5 Funcao bool

```cpp
/**
 * @brief FUNCAO PRINCIPAL DE PROCESSAMENTO
 * @return \c true SE O PROCESSAMENTO OCORREU COM SUCESSO
 * @return \c false SE OCORREU ALGUM ERRO
 */
bool myModule::loop(void)
{
    bool resultado = false;
    // implementacao
    return resultado;
}
```

### 4.6 Funcao com parametros

```cpp
/**
 * @brief EXECUTA UMA ACAO ESPECIFICA
 * @param param VALOR DO PARAMETRO DE EXECUCAO
 * @return \c true SE A ACAO FOI EXECUTADA COM SUCESSO
 * @return \c false SE FALHOU AO EXECUTAR A ACAO
 */
bool myModule::executeAction(uint8_t param)
{
    bool sucesso = false;
    // implementacao
    return sucesso;
}
```

### Regras do source file:

- Toda funcao tem `/** @brief ... */` acima
- `@brief` em sentence case, sem acentos
- `@param` em sentence case: `@param nomeParam Descricao do parametro`
- `@return` usa `\c true` / `\c false` (com `\c` para monospace)
- Funcoes bool tem DOIS `@return`: um para true, outro para false
- Formato do @return: `@return \c true SE [CONDICAO]` / `@return \c false SE [CONDICAO]`
- Construtores documentam todos os `@param`
- Destrutores tem apenas `@brief`

---

## 5. ENUM E STRUCT — DOCUMENTACAO

### Principio: membros de enums e structs documentados com `///<` inline

O bloco Doxygen contem apenas `@brief` do tipo. Cada membro e documentado
com `///<` inline na mesma linha — menos verboso, leitura direta no codigo.

### 5.1 Enum

```cpp
/**
 * @brief ESTADOS DE OPERACAO DO MODULO
 */
enum operationMode_e
{
    UNDEFINED = 0, ///< MODO AINDA NAO DEFINIDO
    MODE_STA  = 1, ///< MODO ESTACAO (CLIENTE)
    MODE_AP   = 2, ///< MODO ACCESS POINT
    MODE_MESH = 3  ///< MODO MESH NETWORK
};
```

### 5.2 Struct

```cpp
/**
 * @brief CONTROLE DE CONEXAO COM O BROKER MQTT
 */
typedef struct
{
    statusConnect_e statusConnect = {statusConnect_e::DISCONNECTED}; ///< STATUS ATUAL DA CONEXAO
    uint32_t timeStartConnect    = {NULL};                          ///< TIMESTAMP DA ULTIMA CONEXAO ESTAVEL (ms)
    uint8_t errorConnection      = {NULL};                          ///< CONTADOR DE ERROS — RESET APOS RECONEXAO
    bool connectBackup           = {false};                         ///< INDICA USO DE CREDENCIAIS DE BACKUP
} controlConnect_t;
```

### Regras de enum/struct:

- `@brief` descreve o proposito geral do tipo
- Membros documentados com `///<` inline — **nunca** `@param` no bloco Doxygen
- Campos de struct inicializados com `= {valor}` — incluindo arrays (`= {0}`)
- Enums com valores explicitos em **todos** os membros
- Enums usados sempre com escopo: `operationMode_e::MODE_STA`

---

## 6. VARIAVEIS GLOBAIS — AGRUPAMENTO

### Principio: agrupar variaveis relacionadas, documentar individualmente apenas as criticas

Em vez de um `@brief` para cada variavel, agrupar por funcionalidade com um bloco unico.
Documentacao individual reservada para ponteiros criticos (comunicacao, hardware, estado do sistema).

### 6.1 REPROVADO — documentacao individual excessiva

```cpp
// REPROVADO: cada variavel com @brief individual, maioria redundante
/**
 * @brief GERENCIAMENTO DE ARMAZENAMENTO DE DADOS
 */
fileSystem* systemData = nullptr;

/**
 * @brief ESTRUTURA DE CONFIGURACOES
 */
dataConfig_t* dataConfig = nullptr;

/**
 * @brief GERENCIAMENTO DE DADOS
 */
dataManager* saveConfig = nullptr;

/**
 * @brief PROTOCOLO DE COMUNICACAO DO WIFI
 */
wifiManager* wifi = nullptr;

/**
 * @brief PROTOCOLO MQTT PARA BROKER LOCAL
 */
mqttManager* localMQTT = nullptr;

/**
 * @brief CONEXAO COM BROKER DE LOG
 */
mqttManager* remoteMQTT = nullptr;
```

### 6.2 APROVADO — agrupamento com documentacao proporcional

```cpp
// --- PERSISTENCIA DE DADOS ---
fileSystem* systemData    = nullptr;
dataConfig_t* dataConfig  = nullptr;
dataManager* saveConfig   = nullptr;

// --- COMUNICACAO ---
wifiManager* wifi = nullptr;

/**
 * @brief BROKER LOCAL — CONTROLE DO DISPOSITIVO VIA MQTT
 * @warning REQUER wifi INICIALIZADO ANTES DE CONECTAR
 */
mqttManager* localMQTT = nullptr;

/**
 * @brief BROKER REMOTO — EXCLUSIVO PARA ENVIO DE LOGS
 * @note NAO RECEBE COMANDOS, APENAS PUBLICA
 */
mqttManager* remoteMQTT = nullptr;

// --- LOG ---
syncLog* logSync       = nullptr;
managerLog* logManager = nullptr;

// --- HARDWARE / RF ---

/**
 * @brief PROTOCOLO SOMFY — CONTROLE DE PERSIANAS VIA RF 433MHz
 * @warning DEPENDE DE processRF PARA TIMING DOS PULSOS
 */
rfSomfyProtocol* somfyProtocol = nullptr;

commandQueue* queueCommand = nullptr;

/**
 * @brief TIMER PARA PROCESSAMENTO CICLICO DO RF
 * @warning OPERA VIA INTERRUPCAO — NAO ALOCAR MEMORIA DENTRO DO CALLBACK
 */
Ticker* processRF = nullptr;

// --- CONTROLE ---
bool enablePortalCaptive = false;
EasyButton* button = nullptr;
```

**Quando documentar uma variavel individual:**
- Ponteiros para modulos de comunicacao (MQTT, HTTP) — indicar proposito especifico
- Ponteiros para hardware (RF, GPIO, Ticker) — indicar restricoes
- Variaveis com relacoes de dependencia entre si (`localMQTT` depende de `wifi`)
- Variaveis com restricoes de uso (@warning)

**Quando agrupar sem documentacao:**
- Variaveis de persistencia (`fileSystem*`, `dataManager*`, `dataConfig_t*`)
- Variaveis de log (`syncLog*`, `managerLog*`)
- Flags simples (`bool enablePortalCaptive`)
- Qualquer variavel cujo tipo + nome explicam o proposito

---

## 7. ARQUIVO DE LOG — DOCUMENTACAO

```cpp
#ifndef _LOG_MODULE_H
#define _LOG_MODULE_H

/**
 * @file log-module.h
 * @author <AUTHOR_NAME> (<AUTHOR_EMAIL>)
 * @brief CLASSE PARA CONTROLE DE ENVIO DE LOGS DO MODULO
 * @version X.Y.Z
 * @date DD-MM-YYYY
 * @copyright Copyright (c) YYYY \b <COMPANY_NAME>
 */

/**
 * @brief BIBLIOTECA PADRAO DE LOGs
 */
#include <debug-log.h>
```

---

## 8. NUMERACAO DE SUBSISTEMAS (@page)

O sistema de numeracao `subsystem{N}` segue uma ordem fixa. Ao criar novo firmware ou biblioteca, usar o proximo numero disponivel na sequencia do projeto.

**Padrao recomendado:**
- `subsystem1` a `subsystem7` — bibliotecas de base (config, debug, utils, etc.)
- `subsystem8+` — firmwares e componentes especificos

---

## 9. CROSS-REFERENCES (LINKS DOXYGEN)

| Tipo | Formato | Exemplo |
|------|---------|---------|
| Pagina | `[nome](#subsystemN)` | `[wifi-manager](#subsystem5)` |
| Secao | `[nome](#subsystemNsectionN)` | `[Firmware](#subsystem8section1)` |
| Subsecao | `[nome](#subsystemNsubsectionN)` | `[Setup](#subsystem8subsection3)` |
| Classe | `#className` | `@see wifiManager` |
| Metodo | `[nome](#className::method())` | `[setup](#wifiManager::setup())` |
| Arquivo | `[nome](#path/file.h)` | `[principals.h](#project/principals.h)` |

---

## 10. TABELAS DE HARDWARE — REFERENCIA

### 10.1 Build Config (centralizado)

```markdown
| CONFIG | VALUE |
| :----: | :----: |
| MODULO | ESP-WROOM-32 |
| MCU | ESP32 |
```

### 10.2 Pinout (alinhado a esquerda)

```markdown
| CANAL | GPIO | FUNCAO |
| :---- | :---- | :---- |
| 1 | 12 | RELAY |
```

### 10.3 Revisoes de hardware (agrupamento com ^)

```markdown
| Revision A | BOARD_v1.0 |
| ^ | Flash size 4MB |
| ^ | File System 196KB |
| Revision B | BOARD_v1.1 |
| ^ | Flash size 4MB |
| ^ | File System 196KB |
```

O caracter `^` na primeira coluna indica que a linha pertence ao mesmo grupo da revisao acima.

---

## 11. CLASSIFICACAO POR CRITICIDADE — FUNCOES

Toda funcao em `.cpp` tem no minimo `@brief`. A profundidade da documentacao
e proporcional a criticidade, nao ao tamanho da funcao.

### 11.1 Funcao Simples

Funcoes autoexplicativas com logica linear, sem efeitos colaterais relevantes.

**Exemplos:** getters, setters, conversoes simples, checkDelete, destrutores.

**Documentacao:** apenas `@brief`. `@param` e `@return` apenas se o nome do
parametro ou tipo de retorno nao forem autoexplicativos.

```cpp
/**
 * @brief RETORNA O MODO DE OPERACAO DO WIFI
 * @return \c WiFiMode_t COM O MODO DE OPERACAO
 */
WiFiMode_t wifiDriver::getWifiMode(void)
{
    return _WiFiMode;
}

/**
 * @brief VERIFICA E DELETA A INSTANCIA DO DRIVER
 */
void myManager::checkDeleteDriver(void)
{
    if(_driver != nullptr)
    {
        delete _driver;
        _driver = nullptr;
    }
}
```

### 11.2 Funcao Moderada

Funcoes com validacao de parametros, logica de decisao, alocacao de memoria,
ou que podem falhar por multiplas razoes.

**Exemplos:** construtores com validacao, `setup()`, `loadConfig()`, funcoes
de fila/buffer, funcoes com fallback.

**Documentacao:** `@brief` + `@param`/`@return` obrigatorios.
Usar `@note` apenas se houver contexto nao obvio pelo codigo.

```cpp
/**
 * @brief CARREGA CONFIGURACOES DO DISPOSITIVO A PARTIR DO FILESYSTEM
 * @return \c true SE CONSEGUIU CARREGAR CONFIGURACAO VALIDA
 * @return \c false SE O ARQUIVO NAO EXISTE, ESTA CORROMPIDO,
 *         OU O FILESYSTEM NAO ESTA MONTADO
 * @note TENTA MIGRAR FORMATO ANTERIOR SE O JSON NAO EXISTIR
 */
bool dataManager::loadConfig(void)
```

```cpp
/**
 * @brief ADICIONA UM REGISTRO NA FILA DE COMANDOS
 * @param timeExecute TEMPO DE ESPERA APOS O COMANDO ANTERIOR (ms)
 * @param input IDENTIFICADOR DO REGISTRO NA FILA
 * @return \c true SE O REGISTRO FOI ADICIONADO COM SUCESSO
 * @return \c false SE FALHOU NA ALOCACAO DE MEMORIA PARA O REGISTRO
 */
bool queueDelay::addReg(uint16_t timeExecute, uint8_t input)
```

### 11.3 Funcao Critica

Funcoes que interagem com hardware, comunicacao RF/MQTT/WiFi/HTTP, timers,
interrupcoes, ou que tem efeitos colaterais no sistema.

**Exemplos:** controle GPIO, envio/recepcao RF, conexao MQTT/WiFi,
manipulacao de registradores, `delayMicroseconds`, watchdog, sleep modes.

**Documentacao:** `@brief` + `@param`/`@return` + `@warning` obrigatorio.
Usar `@note` para contexto de uso e `@see` para referencia externa.

```cpp
/**
 * @brief TRANSMITE UM PAR DE PULSOS HIGH/LOW VIA GPIO DO RADIO RF
 * @param executePulse ESTRUTURA COM TIMINGS E VALORES DO PULSO (us)
 * @warning UTILIZA delayMicroseconds — NAO CHAMAR DENTRO DE ISR.
 *          TIMINGS INCORRETOS CAUSAM FALHA NA DECODIFICACAO PELO RECEPTOR
 */
void rfDriverTX::sendPulse(pulseProtocol_t* executePulse)
```

```cpp
/**
 * @brief CONFIGURA O PINO DO REGISTRADOR 74HC595 PARA ACIONAMENTO
 * @param pin POSICAO DO PINO NO REGISTRADOR (0-7)
 * @param mode MODO DE OPERACAO — SOMENTE \c OUTPUT E VALIDO
 * @return \c true SE A CONFIGURACAO FOI ACEITA
 * @return \c false SE O MODO NAO E \c OUTPUT (REJEITADO PELO CI)
 * @warning O CI 74HC595 NAO SUPORTA INPUT — OUTROS MODOS SAO IGNORADOS
 */
bool SR74HC595::pinMode(uint8_t pin, uint8_t mode)
```

```cpp
/**
 * @brief DESLIGA A INTERFACE WIFI PARA REDUZIR CONSUMO DE ENERGIA
 * @warning CHAMAR ANTES DE QUALQUER INICIALIZACAO WIFI PARA EVITAR BROWNOUT
 * @see https://github.com/esp8266/Arduino/issues/2111#issuecomment-224251391
 */
void wifiDriver::disableWifi(void)
```

### Tabela resumo — tags por criticidade

| Tag | Simples | Moderada | Critica |
|-----|---------|----------|---------|
| `@brief` | OBRIGATORIO | OBRIGATORIO | OBRIGATORIO |
| `@param` | SE NAO OBVIO | OBRIGATORIO | OBRIGATORIO |
| `@return` | SE NAO OBVIO | OBRIGATORIO (todos os cenarios) | OBRIGATORIO (todos os cenarios) |
| `@note` | - | SE HOUVER CONTEXTO | SE HOUVER CONTEXTO |
| `@warning` | - | - | OBRIGATORIO |
| `@see` | - | - | SE HOUVER REFERENCIA |

---

## 12. REGRAS DE QUALIDADE — O QUE REPROVAR

### 12.1 Redundancia (REPROVADO)

Comentarios que repetem o nome, tipo ou contexto ja obvio:

```cpp
// REPROVADO: @brief repete o nome
/**
 * @brief SEND PULSE
 */
void rfDriverTX::sendPulse(...)

// REPROVADO: @param repete o nome do parametro
/**
 * @param executePulse O EXECUTE PULSE
 */

// REPROVADO: @brief individual para variavel autoexplicativa
/**
 * @brief GERENCIAMENTO DE DADOS
 */
dataManager* saveConfig = nullptr;

// REPROVADO: enum sem @param nos membros
/**
 * @brief MODOS DE OPERACAO
 */
enum operationMode_e
{
    MODE_STA = 0,
    MODE_AP  = 1
};

// APROVADO: enum com @param em todos os membros
/**
 * @brief MODOS DE OPERACAO
 * @param MODE_STA MODO ESTACAO (CLIENTE)
 * @param MODE_AP MODO ACCESS POINT
 */
enum operationMode_e
{
    MODE_STA = 0,
    MODE_AP  = 1
};
```

### 12.2 Genericidade (REPROVADO)

Comentarios vagos que nao descrevem comportamento concreto:

```cpp
// REPROVADO: @return generico — nao descreve as condicoes
/**
 * @return \c true SE SUCESSO
 * @return \c false SE ERRO
 */

// REPROVADO: @brief vago
/**
 * @brief PROCESSA OS DADOS
 */
```

**Correcao:** descrever a condicao especifica de sucesso/falha:

```cpp
/**
 * @return \c true SE A INTERFACE WIFI ESTA ATIVA OU FOI
 *         REATIVADA COM SUCESSO APOS SLEEP
 * @return \c false SE O WAKEUP FALHOU OU O RESET DA
 *         INTERFACE NAO CONSEGUIU RESTAURAR A CONEXAO
 */
```

### 12.3 Excesso de Documentacao (REPROVADO)

Documentacao que polui visualmente sem agregar valor:

```cpp
// REPROVADO: @details, @pre, @post, @sideeffects em funcao simples
/**
 * @brief RETORNA O MODO DE OPERACAO
 * @details ACESSA O MEMBRO PRIVADO _operationMode E RETORNA SEU VALOR
 * @pre OBJETO DEVE ESTAR INICIALIZADO
 * @return \c operationMode_e COM O MODO DE OPERACAO
 * @post NENHUM EFEITO COLATERAL
 */
operationMode_e getOperationMode(void)

// REPROVADO: @param que repete o tipo
/**
 * @param data PONTEIRO UINT8_T COM OS DADOS
 * @param size TAMANHO UINT8_T DO VETOR
 */
```

**Correcao:** documentar apenas o que nao e obvio:

```cpp
/**
 * @brief RETORNA O MODO DE OPERACAO
 */
operationMode_e getOperationMode(void)

/**
 * @param data VETOR DE BYTES PARA CALCULO — NAO PODE SER nullptr
 * @param size QUANTIDADE DE BYTES A PROCESSAR (> 0)
 */
```

### 12.4 Enum/Struct sem documentacao inline (REPROVADO)

```cpp
// REPROVADO: membros sem ///<
/**
 * @brief ESTADOS DE CONEXAO
 */
enum statusConnect_e
{
    DISCONNECTED  = 0,
    CONNECTED     = 1,
    RECONNECTING  = 2
};
```

**Correcao:** `///<` inline em todos os membros:

```cpp
/**
 * @brief ESTADOS DE CONEXAO
 */
enum statusConnect_e
{
    DISCONNECTED  = 0, ///< DESCONECTADO DO BROKER
    CONNECTED     = 1, ///< CONECTADO E OPERACIONAL
    RECONNECTING  = 2  ///< TENTATIVA AUTOMATICA APOS PERDA DE SINAL
};
```

---

## 13. ORDEM DAS TAGS DOXYGEN

Quando multiplas tags forem necessarias, respeitar a ordem:

1. `@brief`
2. `@param` (na ordem dos parametros da funcao)
3. `@return` (do mais comum para o menos comum)
4. `@warning`
5. `@note`
6. `@see`

---

## 14. ALINHAMENTO COM PADROES DO CODEBASE

### 14.1 Padrao de Retorno Bool

O codebase usa variavel local de resultado com unico `return` ao final.
Os `@return` devem mapear os cenarios principais:

```cpp
/**
 * @brief VERIFICA SE EXISTEM CREDENCIAIS WIFI VALIDAS
 * @return \c true SE HA SSID CONFIGURADO OU BACKUP DISPONIVEL
 * @return \c false SE NAO HA CREDENCIAIS E O BACKUP TAMBEM FALHOU
 */
bool myManager::checkValidCredentials(void)
```

### 14.2 Construtores e Destrutores

- **Construtores:** `@brief` + `@param` para cada parametro
- **Destrutores:** apenas `@brief`
- **checkDelete[X]():** apenas `@brief`

---

## 15. PROCEDIMENTO DE REVISAO

Ao revisar documentacao, siga este fluxo:

### Passo 1 — Classificar cada elemento

Para cada funcao, variavel, enum ou struct:
1. Ler o codigo (assinatura, corpo, contexto)
2. Classificar como **Simples**, **Moderado** ou **Critico** (Secoes 6 e 11)

### Passo 2 — Validar proporcionalidade

Verificar se a documentacao e proporcional a criticidade:
- **Simples com excesso** → REPROVADO por verbosidade
- **Critico sem @warning** → REPROVADO por falta de informacao de risco
- **Moderado com @details/@pre/@post** → REPROVADO por excesso (usar @note se necessario)
- **Enum/struct sem `///<`** nos membros → REPROVADO

### Passo 3 — Validar qualidade do conteudo

- `@brief` descreve comportamento, nao repete o nome?
- `@param` agrega alem do nome/tipo (range, restricoes, unidade)?
- `@return` cobre cenarios com condicoes especificas (nao generico)?
- `@warning` descreve restricao real (hardware, timing, dependencia)?
- Variaveis relacionadas estao agrupadas?
- Campos de enum/struct com @param no bloco Doxygen?

### Passo 4 — Emitir relatorio

Para cada elemento, emitir um dos vereditos:

- **APROVADO** — documentacao proporcional e de qualidade
- **REPROVADO — EXCESSO** — documentacao verbosa, redundante ou excessiva para a criticidade
- **REPROVADO — FALTA** — informacao critica ausente (restricao, risco, contexto)

### Formato do relatorio

```
## [Elemento] — [CLASSIFICACAO]

**Veredito:** [APROVADO | REPROVADO — EXCESSO | REPROVADO — FALTA]

**Problema:** @brief redundante — "GERENCIAMENTO DE DADOS" para dataManager* saveConfig

**Correcao sugerida:**
[codigo corrigido ou indicacao de remocao]
```

---

## 16. CONVENCOES GERAIS

| Regra | Aplicacao |
|-------|-----------|
| Idioma | Portugues brasileiro |
| Case dos comentarios | Sentence case, sem acentos (`@warning` em MAIUSCULAS) |
| Formato de data | DD-MM-YYYY |
| @brief no .h | Apenas para membros private NAO obvios |
| @brief no .cpp | Para TODA funcao |
| @param | Sentence case, sem acentos, apenas se agrega valor |
| @return bool | Dois @return separados (true/false) com `\c` |
| Enum/struct campos | `///<` inline obrigatorio em todos os membros, nunca `@param` |
| Variaveis globais | Agrupar relacionadas, documentar individualmente apenas criticas |
| Links internos | Formato Doxygen `#anchor` |

---

## 17. CHECKLIST DE DOCUMENTACAO

### Novo firmware:
- [ ] `README.md` com `@page`, `@tableofcontents`, secoes Firmware e Hardware
- [ ] Tabela de Build Config com modulo, MCU, flash, filesystem
- [ ] Tabela de Pinout por revisao de hardware
- [ ] Cada arquivo `.cpp` referenciado em `@subsection`
- [ ] Links para bibliotecas usadas com `#subsystemN`

### Nova biblioteca:
- [ ] `README.md` com `@page`, `@see`, secao Dependencias, Principais recursos, Exemplos
- [ ] Todos os metodos publicos listados em "Principais recursos"
- [ ] Links para dependencias com `#subsystemN`

### Novo header (.h):
- [ ] Header guard `#ifndef _CLASSNAME_H`
- [ ] Bloco `@file`, `@author`, `@brief`, `@version`, `@date`, `@copyright`
- [ ] Cada `#include` com `/** @brief ... */`
- [ ] Membros private documentados APENAS se nao obvios

### Novo source (.cpp):
- [ ] Toda funcao com `/** @brief ... */`
- [ ] Construtores com `@param` para cada parametro
- [ ] Funcoes bool com dois `@return \c true/false` com condicoes especificas
- [ ] `@brief` e `@param` em sentence case, sem acentos

### Revisao de qualidade:
- [ ] Nenhuma documentacao repete o nome da funcao, variavel ou parametro
- [ ] Todos os membros de enum/struct documentados com `///<` inline (nunca `@param`)
- [ ] Variaveis globais agrupadas por funcionalidade
- [ ] Funcoes criticas (hardware, RF, comunicacao) tem `@warning`
- [ ] Funcoes simples NAO tem excesso de tags (@details, @pre, @post)
- [ ] Todo `@return` descreve condicoes especificas, nao generico
- [ ] Texto em portugues, sentence case, sem acentos (`@warning` em MAIUSCULAS)
- [ ] Documentacao proporcional a criticidade — nem mais, nem menos
