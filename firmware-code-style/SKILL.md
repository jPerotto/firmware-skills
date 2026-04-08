---
name: firmware-code-style
description: Guia de estilo de codigo C++ para firmware embarcado (ESP8266/ESP32, PlatformIO/Arduino). Aplique sempre que criar ou revisar qualquer arquivo .cpp, .h ou .hpp do firmware. Cobre header guards, Doxygen, estrutura de classes, construtores, destrutores, nomenclatura, enums, structs, logging, braces e padroes de codificacao.
---

# Code Style Guide — Firmware C++ (PlatformIO/Arduino)

Guia de boas praticas de estilo para codigo C++ em firmware embarcado ESP8266/ESP32.

---

## 1. CABECALHO DE ARQUIVO (Doxygen)

Todo arquivo `.h` e `.cpp` comeca com este bloco — sem excecao:

```cpp
/**
 * @file nome-do-arquivo.h
 * @author <AUTHOR_NAME> (<AUTHOR_EMAIL>)
 * @brief DESCRICAO RESUMIDA EM MAIUSCULAS SEM ACENTOS
 * @warning AVISOS IMPORTANTES (opcional)
 * @version X.Y.Z
 * @date DD-MM-YYYY
 * @copyright Copyright (c) YYYY \b <COMPANY_NAME>
 */
```

Regras:
- `@brief` e `@warning` sempre em **MAIUSCULAS SEM ACENTUACAO** (compatibilidade Doxygen)
- Data no formato `DD-MM-YYYY`
- Versao alinhada com a versao do projeto

---

## 2. HEADER GUARDS

Padrao: `_[ESCOPO]_[NOME]_H` — tudo em maiusculas com underscores.

```cpp
// arquivo: wifi-manager.h  →  _WIFI_MANAGER_H
#ifndef _WIFI_MANAGER_H
#define _WIFI_MANAGER_H
// conteudo
#endif

// arquivo: log-wifi.h  →  _LOG_WIFI_MANAGER_H
#ifndef _LOG_WIFI_MANAGER_H
#define _LOG_WIFI_MANAGER_H
// conteudo
#endif

// arquivo: principais.h de uma lib  →  _PRINCIPAIS_[LIBNAME]_H
#ifndef _PRINCIPAIS_WIFI_PROTOCOL_H
#define _PRINCIPAIS_WIFI_PROTOCOL_H
// conteudo
#endif
```

**Nunca usar `#pragma once`.**

---

## 3. INCLUDES — COM DOCUMENTACAO OBRIGATORIA

Todo `#include` deve ter um `/** @brief ... */` acima dele:

```cpp
/**
 * @brief BIBLIOTECA BASE DE CONFIGURACOES E DEFINICOES
 */
#include <firmware-config.h>

/**
 * @brief BIBLIOTECA PARA PERSISTENCIA DE DADOS
 */
#include <data-manager.h>

/**
 * @brief SUB BIBLIOTECA PARA CONTROLE DO DRIVER DE INTERFACE
 */
#include "wifi-driver.h"

/**
 * @brief BIBLIOTECA PARA LOGS DO MODULO
 */
#include "log-wifi.h"
```

Ordem dos includes:
1. Bibliotecas externas/sistema (`<angle-brackets>`)
2. Sub-bibliotecas locais (`"quotes"`) — da mais especifica para a mais geral
3. `log-[nome].h` — sempre por ultimo antes de `principais.h`
4. `principais.h` — sempre o ultimo include

---

## 4. ESTRUTURA DE CLASSE

```cpp
class wifiManager : public wifiDebug
{
  public:
    // Construtor e destrutor primeiro
    wifiManager(dataManager* dataManager, dataConfig_t* dataConfig);
    ~wifiManager();

    // API publica
    void setup(void);
    bool loop(void);
    void enablePortal(void);
    bool enableDistribute(void);

  protected:
    // checkDelete para cada ponteiro membro proprio
    void checkDeleteDriver(void);
    void checkDeleteWiFiConnect(void);

    // helpers de setup e configuracao
    void setupModeConnection(void);
    void setupModeOperation(operationMode_e operationMode);

    // getters e setters internos
    void setOperationMode(operationMode_e operationMode);
    operationMode_e getOperationMode(void);

    // status checks que retornam bool
    bool statusConnection(void);
    bool checkValidCredentials(void);

  private:
    /**
     * @brief MODO DE OPERACAO DO WIFI
     */
    operationMode_e _operationMode = operationMode_e::WIFI_UNDEFINED;

    /**
     * @brief CLASE PARA SALVAR OS DADOS DE CONFIGURACOES
     */
    dataManager* _dataManager = nullptr;

    /**
     * @brief CONTROLE DA INTERFACE DE RADIO DO WIFI
     */
    wifiDriver* _wifiDriver = nullptr;
};
```

Regras da estrutura:
- Herdar da classe de log (`public wifiDebug`, `public mqttDebug`, etc.)
- Ordem: `public` → `protected` → `private`
- Indentacao com **tab** (nao espacos)
- `public:`, `protected:`, `private:` com **2 espacos** de recuo
- Membros privados: cada um com `/** @brief ... */`
- Funcoes sempre com `(void)` quando sem parametros

---

## 5. MEMBROS PRIVADOS — NOMENCLATURA E INICIALIZACAO

```cpp
private:
    /**
     * @brief MODO DE OPERACAO DO MODULO
     */
    operationMode_e _operationMode = operationMode_e::UNDEFINED;

    /**
     * @brief MODO DE CONEXAO
     */
    WiFiMode_t _connectMode = WiFiMode_t::WIFI_OFF;

    /**
     * @brief CLASSE PARA CONTROLE DE DADOS
     */
    dataManager* _dataManager = nullptr;

    /**
     * @brief FLAG DE CONTROLE
     */
    bool _activeDistribute = false;

    /**
     * @brief CONTADOR DE EVENTOS
     */
    uint32_t _counter = NULL;   // NULL para tipos numericos (padrao do codebase)
```

Regras:
- Prefixo `_` (underscore) em **todos** os membros privados
- camelCase apos o underscore: `_wifiDriver`, `_operationMode`
- Ponteiros sempre inicializados com `= nullptr`
- Enums inicializados com escopo: `operationMode_e::UNDEFINED`
- Tipos numericos inicializados com `NULL` (padrao do codebase — nao alterar para `0`)
- Booleanos inicializados com `false`
- Constantes de classe: sempre `static constexpr` — **nunca** `static const` para valores conhecidos em tempo de compilacao

```cpp
// CORRETO
static constexpr uint32_t TIMEOUT_MS = 10000;
static constexpr float VARIATION_RATIO = 0.30f;
static constexpr uint8_t MAX_RETRIES = 5;

// ERRADO — usar constexpr em vez de const
static const uint32_t TIMEOUT_MS = 10000;
static const float VARIATION_RATIO = 0.30f;
```

---

## 6. CONSTRUTOR — PADRAO OBRIGATORIO

```cpp
/**
 * @brief CONSTRUTOR DA CLASSE DE GERENCIAMENTO
 * @param dataManager CLASSE DE GERENCIAMENTO DE DADOS
 * @param dataConfig ESTRUTURA DE CONFIGURACAO
 */
myManager::myManager(dataManager* dataManager, dataConfig_t* dataConfig)
{
    if((dataManager == nullptr) || (dataConfig == nullptr))
    {
        e_LOG_MYMANAGER("NullPointerException");
    }
    else
    {
        _dataManager = dataManager;
        _dataConfig = dataConfig;
        _driver = new myDriver;

        if(_driver == nullptr)
        {
            e_LOG_MYMANAGER("NullPointerException");
        }
    }
}
```

Regras do construtor:
1. **Primeiro:** verificar TODOS os parametros ponteiro com `if((p1 == nullptr) || (p2 == nullptr))`
2. **Se null:** logar `e_LOG_X("NullPointerException")` — sem mais nada
3. **Se valido:** atribuir aos membros `_`, depois criar objetos internos com `new`
4. **Apos cada `new`:** verificar se retornou `nullptr` e logar se sim
5. Sem lista de inicializacao (`: member(value)`) — usar corpo do construtor
6. Abrir chave na **linha seguinte** (Allman style)

---

## 7. DESTRUTOR — PADRAO OBRIGATORIO

```cpp
/**
 * @brief DESTRUTOR DA CLASSE
 */
myManager::~myManager()
{
    checkDeleteDriver();
    checkDeleteConnection();
}
```

E para cada ponteiro membro, uma funcao `checkDelete[Membro]()` em `protected:`:

```cpp
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

Regra: cada ponteiro alocado com `new` dentro da classe tem **sua propria funcao** `checkDelete[X]()`. Nunca fazer `delete` diretamente no destrutor.

---

## 8. DOCUMENTACAO DE FUNCOES (DOXYGEN)

### Funcao void simples
```cpp
/**
 * @brief INICIA A CONFIGURACAO DO GERENCIADOR
 */
void myManager::setup(void)
{
    checkConfig();
    setupModeConnection();
}
```

### Funcao com retorno bool
```cpp
/**
 * @brief INICIA A DISTRIBUICAO DE CREDENCIAIS
 * @return \c true SE CONSEGUIU INICIAR A DISTRIBUICAO
 * @return \c false SE FALHOU AO INICIAR A DISTRIBUICAO
 */
bool myManager::enableDistribute(void)
```

### Funcao com parametros
```cpp
/**
 * @brief CONFIGURA O MODO DE OPERACAO
 * @param operationMode MODO DE OPERACAO \see \c operationMode_e
 */
void myManager::setupModeOperation(operationMode_e operationMode)
```

### Getter
```cpp
/**
 * @brief RETORNA O MODO DE OPERACAO
 * @return \c operationMode_e COM O MODO DE OPERACAO
 */
operationMode_e myManager::getOperationMode(void)
{
    return _operationMode;
}
```

Regras:
- `@brief` em MAIUSCULAS SEM ACENTOS
- `@param` em MAIUSCULAS
- `@return \c true` e `@return \c false` em linhas separadas para bool
- Usar `\c` para nomes de tipos/variaveis inline
- Usar `\see` para referencias cruzadas

---

## 9. BRACE STYLE — ALLMAN

**Chave de abertura SEMPRE na linha seguinte**, em qualquer contexto:

```cpp
// Funcao
void myManager::setup(void)
{
    // corpo
}

// if/else
if((dataManager == nullptr) || (dataConfig == nullptr))
{
    e_LOG_MODULE("NullPointerException");
}
else
{
    _dataManager = dataManager;
}

// switch/case — cada case com chaves
switch(getOperationMode())
{
    case operationMode_e::MODE_A:
    {
        i_LOG_MODULE("MODE_A");
        activateModeA();
        break;
    }
    case operationMode_e::MODE_B:
    {
        i_LOG_MODULE("MODE_B");
        activateModeB();
        break;
    }
    default:
    {
        e_LOG_MODULE("UNKNOW");
        break;
    }
}
```

**Nunca omitir chaves**, mesmo em blocos de uma linha.

---

## 10. PADRAO PARA FUNCOES QUE RETORNAM BOOL

Sempre declarar variavel local de resultado, manipular, retornar ao final:

```cpp
bool myManager::checkValidCredentials(void)
{
    bool validCredentials = false;

    if(_dataConfig == nullptr)
    {
        validCredentials = false;
        e_LOG_MODULE("NullPointerException");
    }
    else
    {
        if(strlen(_dataConfig->credentials.ssid) > NULL)
        {
            validCredentials = true;
        }
        else
        {
            validCredentials = checkLoadBackupCredentials();
        }
    }

    return validCredentials;
}
```

**Nunca** usar `return true;` / `return false;` no meio da funcao — um unico `return` ao final.

---

## 11. VERIFICACAO DE PONTEIROS EM METODOS

Padrao para **multiplos ponteiros**:

```cpp
if((_driver == nullptr) || (_dataManager == nullptr) || (_dataConfig == nullptr))
{
    e_LOG_MODULE("NullPointerException");
}
else
{
    // logica
}
```

Padrao para **um ponteiro** com reset/recuperacao:

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

Padrao para **ja existe + reconfigurar**:

```cpp
if(_connection != nullptr)
{
    w_LOG_MODULE("Reconfigurar conexao");
    checkDeleteConnection();
    setup();
}
else
{
    _connection = new connectionHandler(_driver, _dataManager, _dataConfig);

    if(_connection == nullptr)
    {
        e_LOG_MODULE("NullPointerException");
    }
    else
    {
        _connection->setup(getOperationMode());
    }
}
```

---

## 12. ENUMERACOES

```cpp
/**
 * @brief MODOS DE OPERACAO DO MODULO
 */
enum operationMode_e
{
    MODE_UNDEFINED = 0, ///< MODO AINDA NAO DEFINIDO
    MODE_A         = 1, ///< MODO DE OPERACAO A
    MODE_B         = 2, ///< MODO DE OPERACAO B
    MODE_C         = 3, ///< MODO DE OPERACAO C
    MODE_D         = 4  ///< MODO DE OPERACAO D
};
```

Regras:
- Sufixo `_e` no nome do enum
- Membros em `UPPER_SNAKE_CASE` **sem** prefixo do enum
- Documentar cada membro com `///<` inline — nunca `@param` no bloco Doxygen
- Valores explicitos em **todos** os membros — para rapida visualizacao e entendimento
- Ao usar: **sempre** com escopo em todos os contextos — atribuicao, comparacao, switch/case, inicializacao:

```cpp
// CORRETO — escopo explicito em todos os contextos
errorCode_e _error = errorCode_e::LT_ERROR_NONE;       // inicializacao
_error = errorCode_e::LT_ERROR_TIMEOUT;                 // atribuicao
if(_error == errorCode_e::LT_ERROR_NONE)                 // comparacao
case errorCode_e::LT_ERROR_NONE:                         // switch/case

// ERRADO — sem escopo
errorCode_e _error = LT_ERROR_NONE;
_error = LT_ERROR_TIMEOUT;
if(_error == LT_ERROR_NONE)
case LT_ERROR_NONE:
```

---

## 13. STRUCTS (TYPEDEFS)

```cpp
/**
 * @brief ESTRUTURA PARA CONTROLE DE CONEXAO
 */
typedef struct
{
    statusConnect_e statusConnect    = {statusConnect_e::DISCONNECTED}; ///< CONTROLE DOS STATUS DE CONEXAO
    uint32_t timeStartConnect       = {NULL};                          ///< ULTIMA CONEXAO ESTAVEL EM MILISSEGUNDOS
    uint8_t errorConnection         = {NULL};                          ///< CONTADOR DE ERROS DE CONEXAO
    bool connectBackup              = {false};                         ///< INDICA USO DE CREDENCIAIS DE BACKUP
} controlConnect_t;
```

Regras:
- Sufixo `_t` no nome do typedef
- Membros em `camelCase`
- Inicializacao entre chaves em **todos** os membros: `= {valor}` — incluindo arrays (`= {0}`). Nenhum membro pode ficar sem inicializacao
- Enums com escopo: `statusConnect_e::DISCONNECTED`
- Documentar cada membro com `///<` inline — nunca `@param` no bloco Doxygen

---

## 14. CLASSE DE LOG — PADRAO LOG-[NOME].h

Cada biblioteca tem seu proprio arquivo de log com macros condicionais por nivel:

```cpp
// arquivo: log-module.h
#ifndef _LOG_MODULE_H
#define _LOG_MODULE_H

/**
 * @file log-module.h
 * @author <AUTHOR_NAME> (<AUTHOR_EMAIL>)
 * @brief CLASSE PARA REALIZAR OS LOGS DO MODULO
 * @version X.Y.Z
 * @date DD-MM-YYYY
 * @copyright Copyright (c) YYYY \b <COMPANY_NAME>
 */

/**
 * @brief BIBLIOTECA PADRAO DE LOGs
 */
#include <debug-log.h>

class moduleDebug
{
  protected:
#if BOARD_DEBUG_LEVEL >= LOG_LEVEL_ERROR
    #define e_LOG_MODULE(log, ...) LOG.printfln(CREATE_LOG(MODULE, log), ##__VA_ARGS__)
#else
    #define e_LOG_MODULE(log, ...) _NOP()
#endif

#if BOARD_DEBUG_LEVEL >= LOG_LEVEL_WARN
    #define w_LOG_MODULE(log, ...) LOG.printfln(CREATE_LOG(MODULE, log), ##__VA_ARGS__)
#else
    #define w_LOG_MODULE(log, ...) _NOP()
#endif

#if BOARD_DEBUG_LEVEL >= LOG_LEVEL_INFO
    #define i_LOG_MODULE(log, ...) LOG.printfln(CREATE_LOG(MODULE, log), ##__VA_ARGS__)
#else
    #define i_LOG_MODULE(log, ...) _NOP()
#endif

#if BOARD_DEBUG_LEVEL >= LOG_LEVEL_DEBUG
    #define d_LOG_MODULE(log, ...) LOG.printfln(CREATE_LOG(MODULE, log), ##__VA_ARGS__)
#else
    #define d_LOG_MODULE(log, ...) _NOP()
#endif

#if BOARD_DEBUG_LEVEL >= LOG_LEVEL_VERBOSE
    #define v_LOG_MODULE(log, ...) LOG.printfln(CREATE_LOG(MODULE, log), ##__VA_ARGS__)
#else
    #define v_LOG_MODULE(log, ...) _NOP()
#endif
};

#endif
```

Padrao de nomes das macros: `[nivel]_LOG_[MODULO](msg, ...)`:
- `e_LOG_X` — error
- `w_LOG_X` — warning
- `i_LOG_X` — info
- `d_LOG_X` — debug
- `v_LOG_X` — verbose

---

## 15. ARQUIVO PRINCIPAIS.H DE BIBLIOTECA

Para bibliotecas, `principais.h` contem **enums e structs** (nao o arquivo de includes do projeto):

```cpp
#ifndef _PRINCIPAIS_LIBNAME_H
#define _PRINCIPAIS_LIBNAME_H

/**
 * @file principais.h
 * @author <AUTHOR_NAME> (<AUTHOR_EMAIL>)
 * @brief DEFINICOES DE STATUS E ESTRUTURAS DE DADOS
 * @version X.Y.Z
 * @date DD-MM-YYYY
 * @copyright Copyright (c) YYYY \b <COMPANY_NAME>
 */

/**
 * @brief DESCRICAO DO ENUM
 */
enum meuEnum_e
{
    MEMBRO_1 = 0, ///< DESCRICAO DO MEMBRO 1
    MEMBRO_2 = 1  ///< DESCRICAO DO MEMBRO 2
};

/**
 * @brief DESCRICAO DA STRUCT
 */
typedef struct
{
    meuEnum_e campo1 = {meuEnum_e::MEMBRO_1}; ///< DESCRICAO DO CAMPO 1
    uint32_t campo2  = {NULL};                 ///< DESCRICAO DO CAMPO 2
} minhaStruct_t;

#endif
```

---

## 16. NOMENCLATURA — RESUMO

| Item | Convencao | Exemplo |
|------|-----------|---------|
| Classes | PascalCase | `wifiManager`, `dataHandler` |
| Metodos publicos | camelCase, verbo + substantivo | `setup()`, `loop()`, `enablePortal()` |
| Metodos protected | prefixo funcional + camelCase | `checkDeleteDriver()`, `setupMode()`, `getMode()` |
| Membros privados | `_camelCase` | `_driver`, `_operationMode` |
| Enums (tipo) | `camelCase_e` | `operationMode_e`, `statusConnect_e` |
| Enum (membros) | `UPPER_SNAKE_CASE` | `MODE_CONNECT`, `STA_DISCONNECTED` |
| Structs (tipo) | `camelCase_t` | `controlConnect_t`, `dataConfig_t` |
| Struct (membros) | `camelCase` | `statusConnect`, `timeStartConnect` |
| Defines/macros | `UPPER_SNAKE_CASE` | `TIME_RECONNECT`, `N_MAX_RETRIES` |
| Arquivos | `kebab-case` | `wifi-manager.h`, `log-wifi.h` |

---

## 17. COMENTARIOS

```cpp
// Comentarios inline em portugues
// SECOES IMPORTANTES EM MAIUSCULAS

// Uso de \c para tipos em Doxygen inline
// @return \c true SE CONECTADO
// @return \c false SE DESCONECTADO
// @see \c operationMode_e
```

- Sempre em **portugues**
- Doxygen em MAIUSCULAS SEM ACENTOS
- Strings de erro em ingles tecnico: `"NullPointerException"`

---

## 18. CHECKLIST DE REVISAO

Antes de finalizar qualquer arquivo:

- [ ] Cabecalho Doxygen completo (`@file`, `@author`, `@brief`, `@version`, `@date`, `@copyright`)
- [ ] Header guard no padrao `_[ESCOPO]_[NOME]_H`
- [ ] Todo `#include` com `/** @brief ... */` acima
- [ ] Classe herda da classe de log correspondente
- [ ] Construtor verifica todos os ponteiros antes de usar
- [ ] Destrutor chama apenas `checkDelete[X]()` — sem logica direta
- [ ] Cada ponteiro membro tem sua `checkDelete[X]()` em `protected:`
- [ ] Brace style Allman (chave na linha seguinte) em **todos** os contextos
- [ ] `switch/case` com chaves em cada `case`
- [ ] Funcoes bool retornam variavel local, nao `return true/false` no meio
- [ ] Membros privados com prefixo `_` e inicializados inline
- [ ] Constantes de classe com `static constexpr` (nunca `static const` para valores compile-time)
- [ ] Enums usados com escopo: `operationMode_e::MODE_A`
- [ ] Structs com inicializacao entre chaves: `= {valor}`
- [ ] Macros de log em vez de `Serial.print()`
- [ ] Arquivo `log-[nome].h` com classe `[Nome]Debug` e 5 niveis de macro
- [ ] Comentarios e `@brief` em portugues, MAIUSCULAS, SEM ACENTOS
- [ ] `(void)` explicito em funcoes sem parametros
