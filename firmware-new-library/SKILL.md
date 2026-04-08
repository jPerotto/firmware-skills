---
name: firmware-new-library
description: Como criar uma nova biblioteca C++ para PlatformIO/Arduino (ESP8266/ESP32). Use quando o usuario pedir para criar uma nova biblioteca, modulo ou componente reutilizavel. Gera todos os arquivos necessarios com os templates corretos: log-[nome].h, principais.h, [nome].h, [nome].cpp e library.properties.
---

# Criando Nova Biblioteca — PlatformIO/Arduino (ESP8266/ESP32)

Guia de boas praticas para criacao de bibliotecas C++ reutilizaveis para firmware embarcado.

---

## 1. ESTRUTURA DE DIRETORIOS

```
libraries/
└── minha-lib/
    ├── library.properties        ← metadados PlatformIO/Arduino
    ├── README.md
    ├── examples/
    │   └── BasicUsage/
    │       └── BasicUsage.ino
    └── src/
        ├── minha-lib.h           ← header publico (API da biblioteca)
        ├── minha-lib.cpp         ← implementacao
        ├── log-minhaLib.h        ← macros de log (sempre presente)
        ├── principais.h          ← enums e structs da biblioteca
        └── [sub-modulos].h/.cpp  ← se a lib tiver sub-componentes
```

**Regra:** Toda a implementacao fica em `src/`. O arquivo publico da biblioteca e `src/minha-lib.h`.

---

## 2. library.properties

```properties
name=minha-lib
version=1.0.0
author=<AUTHOR_SHORT>
maintainer=<AUTHOR_NAME> <<AUTHOR_EMAIL>>
sentence=Descricao curta do que a biblioteca faz
paragraph=Descricao detalhada da biblioteca e compatibilidade.
category=Communication
url=<REPOSITORY_URL>
architectures=esp8266,esp32
```

Categorias validas: `Communication`, `Data Storage`, `Signal Input/Output`, `Timing`, `Device Control`, `Other`

Para bibliotecas **apenas ESP32**: `architectures=esp32`

---

## 3. TEMPLATE: log-minhaLib.h

Sempre criado primeiro. Nome do arquivo: `log-[nome-da-lib].h`.
A classe de log tem o sufixo `LOG` ou `Debug`:

```cpp
#ifndef _LOG_MINHALIB_H
#define _LOG_MINHALIB_H

/**
 * @file log-minhaLib.h
 * @author <AUTHOR_NAME> (<AUTHOR_EMAIL>)
 * @brief CLASSE PARA CONTROLE DE ENVIO DE LOGS DA MINHA LIB
 * @version X.Y.Z
 * @date DD-MM-YYYY
 * @copyright Copyright (c) YYYY \b <COMPANY_NAME>
 */

/**
 * @brief BIBLIOTECA PADRAO DE LOGs
 */
#include <debug-log.h>

class minhaLibLOG
{
  protected:
#if BOARD_DEBUG_LEVEL >= LOG_LEVEL_ERROR
    #define e_LOG_MINHALIB(log, ...) LOG.printfln(CREATE_LOG(MINHA_LIB, log), ##__VA_ARGS__)
#else
    #define e_LOG_MINHALIB(log, ...) _NOP()
#endif

#if BOARD_DEBUG_LEVEL >= LOG_LEVEL_WARN
    #define w_LOG_MINHALIB(log, ...) LOG.printfln(CREATE_LOG(MINHA_LIB, log), ##__VA_ARGS__)
#else
    #define w_LOG_MINHALIB(log, ...) _NOP()
#endif

#if BOARD_DEBUG_LEVEL >= LOG_LEVEL_INFO
    #define i_LOG_MINHALIB(log, ...) LOG.printfln(CREATE_LOG(MINHA_LIB, log), ##__VA_ARGS__)
#else
    #define i_LOG_MINHALIB(log, ...) _NOP()
#endif

#if BOARD_DEBUG_LEVEL >= LOG_LEVEL_DEBUG
    #define d_LOG_MINHALIB(log, ...) LOG.printfln(CREATE_LOG(MINHA_LIB, log), ##__VA_ARGS__)
#else
    #define d_LOG_MINHALIB(log, ...) _NOP()
#endif

#if BOARD_DEBUG_LEVEL >= LOG_LEVEL_VERBOSE
    #define v_LOG_MINHALIB(log, ...) LOG.printfln(CREATE_LOG(MINHA_LIB, log), ##__VA_ARGS__)
#else
    #define v_LOG_MINHALIB(log, ...) _NOP()
#endif
};

#endif
```

Substituicoes para nova biblioteca:
- `_LOG_MINHALIB_H` → `_LOG_[NOME]_H`
- `minhaLibLOG` → `[NomeDaLib]LOG` ou `[NomeDaLib]Debug`
- `e_LOG_MINHALIB` → `e_LOG_[NOME]` (em MAIUSCULAS)
- `MINHA_LIB` → `[NOME_DA_LIB]` em MAIUSCULAS (sem acentos, sem hifen — usar underscore)

---

## 4. TEMPLATE: principais.h

Contem **enums e structs** da biblioteca (nao includes):

```cpp
#ifndef _PRINCIPAIS_MINHALIB_H
#define _PRINCIPAIS_MINHALIB_H

/**
 * @file principais.h
 * @author <AUTHOR_NAME> (<AUTHOR_EMAIL>)
 * @brief DEFINICOES DE STATUS E ESTRUTURAS DE DADOS DA MINHA LIB
 * @version X.Y.Z
 * @date DD-MM-YYYY
 * @copyright Copyright (c) YYYY \b <COMPANY_NAME>
 */

/**
 * @brief ESTADOS DE OPERACAO DA MINHA LIB
 * @param ESTADO_A DESCRICAO DO ESTADO A
 * @param ESTADO_B DESCRICAO DO ESTADO B
 */
enum operacaoMinhaLib_e
{
    ESTADO_A = 0,
    ESTADO_B = 1
};

/**
 * @brief ESTRUTURA DE CONTROLE DA MINHA LIB
 * @param operacao ESTADO ATUAL DE OPERACAO
 * @param contador CONTADOR DE EVENTOS
 * @param ativo CONTROLE DE ATIVACAO
 */
typedef struct
{
    operacaoMinhaLib_e operacao = {operacaoMinhaLib_e::ESTADO_A};
    uint32_t contador           = {NULL};
    bool ativo                  = {false};
} controlMinhaLib_t;

#endif
```

---

## 5. TEMPLATE: minha-lib.h

```cpp
#ifndef _MINHA_LIB_H
#define _MINHA_LIB_H

/**
 * @file minha-lib.h
 * @author <AUTHOR_NAME> (<AUTHOR_EMAIL>)
 * @brief BIBLIOTECA PARA [DESCRICAO DO PROPOSITO EM MAIUSCULAS]
 * @version X.Y.Z
 * @date DD-MM-YYYY
 * @copyright Copyright (c) YYYY \b <COMPANY_NAME>
 */

/**
 * @brief BIBLIOTECA BASE DE CONFIGURACOES E DEFINICOES
 */
#include <firmware-config.h>

/**
 * @brief BIBLIOTECA PARA DIVERSAS UTILIDADES
 */
#include <utils.h>

/**
 * @brief [INCLUIR DEMAIS DEPENDENCIAS AQUI — cada uma com @brief]
 */
// #include <data-manager.h>

/**
 * @brief BIBLIOTECA PARA LOGS DA MINHA LIB
 */
#include "log-minhaLib.h"

/**
 * @brief PRINCIPAIS DEFINICOES E ESTRUTURAS DA MINHA LIB
 */
#include "principais.h"

class minhaLib : public minhaLibLOG
{
  public:
    minhaLib(/* dependencias como ponteiros */);
    ~minhaLib();

    void setup(void);
    bool loop(void);

    // API publica especifica da biblioteca
    bool executeAction(uint8_t param);
    bool getStatus(void);

  protected:
    void checkDeleteSubComponent(void);
    void setupInternal(void);
    bool checkCondition(void);

  private:
    /**
     * @brief [DESCRICAO DO MEMBRO EM MAIUSCULAS]
     */
    dependency* _dep = nullptr;

    /**
     * @brief ESTADO ATUAL DE OPERACAO
     */
    operacaoMinhaLib_e _operacao = operacaoMinhaLib_e::ESTADO_A;

    /**
     * @brief FLAG DE STATUS
     */
    bool _status = false;

    /**
     * @brief CONTADOR INTERNO
     */
    uint32_t _contador = NULL;
};

#endif
```

---

## 6. TEMPLATE: minha-lib.cpp

```cpp
#include "minha-lib.h"

/**
 * @brief CONSTRUTOR DA CLASSE DA MINHA LIB
 * @param dep [DESCRICAO DA DEPENDENCIA]
 */
minhaLib::minhaLib(dependency* dep)
{
    if(dep == nullptr)
    {
        e_LOG_MINHALIB("NullPointerException");
    }
    else
    {
        _dep = dep;
    }
}

/**
 * @brief DESTRUTOR DA CLASSE DA MINHA LIB
 */
minhaLib::~minhaLib()
{
    checkDeleteSubComponent();
}

/**
 * @brief VERIFICA E DELETA O SUB COMPONENTE
 */
void minhaLib::checkDeleteSubComponent(void)
{
    if(_subComponent != nullptr)
    {
        delete _subComponent;
        _subComponent = nullptr;
    }
}

/**
 * @brief INICIA A CONFIGURACAO DA MINHA LIB
 */
void minhaLib::setup(void)
{
    if(_dep == nullptr)
    {
        e_LOG_MINHALIB("NullPointerException");
    }
    else
    {
        setupInternal();
        i_LOG_MINHALIB("Setup concluido");
    }
}

/**
 * @brief CONFIGURACAO INTERNA DA MINHA LIB
 */
void minhaLib::setupInternal(void)
{
    // implementacao
    v_LOG_MINHALIB("Setup interno");
}

/**
 * @brief FUNCAO PRINCIPAL DE PROCESSAMENTO
 * @return \c true SE O PROCESSAMENTO OCORREU COM SUCESSO
 * @return \c false SE OCORREU ALGUM ERRO
 */
bool minhaLib::loop(void)
{
    bool resultado = false;

    if(_dep == nullptr)
    {
        resultado = false;
        e_LOG_MINHALIB("NullPointerException");
    }
    else
    {
        resultado = checkCondition();
    }

    return resultado;
}

/**
 * @brief VERIFICA A CONDICAO DE OPERACAO
 * @return \c true SE A CONDICAO FOR VALIDA
 * @return \c false SE A CONDICAO FOR INVALIDA
 */
bool minhaLib::checkCondition(void)
{
    bool condicao = false;

    // implementacao

    return condicao;
}

/**
 * @brief EXECUTA UMA ACAO
 * @param param VALOR DO PARAMETRO DE EXECUCAO
 * @return \c true SE A ACAO FOI EXECUTADA COM SUCESSO
 * @return \c false SE FALHOU AO EXECUTAR A ACAO
 */
bool minhaLib::executeAction(uint8_t param)
{
    bool sucesso = false;

    if(_dep == nullptr)
    {
        sucesso = false;
        e_LOG_MINHALIB("NullPointerException");
    }
    else
    {
        // logica de execucao
        i_LOG_MINHALIB("Acao executada com parametro %d", param);
        sucesso = true;
    }

    return sucesso;
}

/**
 * @brief RETORNA O STATUS ATUAL DA MINHA LIB
 * @return \c true SE ESTIVER OPERACIONAL
 * @return \c false SE ESTIVER COM PROBLEMA
 */
bool minhaLib::getStatus(void)
{
    return _status;
}
```

---

## 7. SUB-MODULOS (para libs mais complexas)

Se a biblioteca precisar de sub-modulos:

```
src/
├── minha-lib.h/.cpp          ← orquestrador principal
├── log-minhaLib.h            ← logs compartilhados por todos
├── principais.h              ← enums/structs compartilhados
├── sub-modulo-a.h/.cpp       ← sub-componente A
└── sub-modulo-b.h/.cpp       ← sub-componente B
```

O header do sub-modulo **nao inclui** bibliotecas base diretamente — inclui o header principal que ja os traz, ou inclui apenas o necessario. O log e compartilhado via `log-minhaLib.h`.

---

## 8. INTEGRACAO NO PROJETO (platformio.ini)

```ini
[env:meu-projeto]
platform = espressif32
board = esp32dev
framework = arduino

lib_deps =
    # bibliotecas internas (path relativo)
    ${PROJECT_DIR}/../libraries/firmware-config
    ${PROJECT_DIR}/../libraries/debug-log
    ${PROJECT_DIR}/../libraries/utils
    ${PROJECT_DIR}/../libraries/minha-lib
    # bibliotecas externas
    bblanchon/ArduinoJson @ ^6.21.0
```

Para biblioteca no mesmo repositorio com multiplos ambientes, usar `lib_extra_dirs`:

```ini
[platformio]
lib_extra_dirs = ../libraries

[env:my-project]
platform = espressif32
board = esp32dev
framework = arduino
lib_deps =
    firmware-config
    debug-log
    utils
    minha-lib
```

---

## 9. CHECKLIST DE NOVA BIBLIOTECA

- [ ] Diretorio `libraries/minha-lib/src/` criado
- [ ] `library.properties` com `architectures=esp8266,esp32` (ou so `esp32`)
- [ ] `log-minhaLib.h` com 5 niveis de macro (`e_`, `w_`, `i_`, `d_`, `v_`)
- [ ] `principais.h` com enums (`_e`) e structs (`_t`) da biblioteca
- [ ] `minha-lib.h` herda da classe de log: `class minhaLib : public minhaLibLOG`
- [ ] Construtor verifica todos os ponteiros antes de usar
- [ ] Destrutor chama apenas `checkDelete[X]()` — sem logica direta
- [ ] Cada ponteiro membro com `checkDelete[X]()` em `protected:`
- [ ] Todos os membros privados com prefixo `_` e `/** @brief ... */`
- [ ] Toda funcao com Doxygen (`@brief`, `@param`, `@return \c true/false`)
- [ ] Allman braces em todos os contextos
- [ ] Funcoes bool com variavel local de resultado + unico `return` ao final
- [ ] `(void)` explicito em funcoes sem parametros
- [ ] Comentarios e `@brief` em portugues, MAIUSCULAS, SEM ACENTOS
- [ ] `library.properties` adicionado
- [ ] `examples/BasicUsage/BasicUsage.ino` criado

---

## 10. EXEMPLO COMPLETO — biblioteca simples sem dependencias

Para uma biblioteca utilitaria sem ponteiros injetados:

```cpp
// log-myTimer.h
class myTimerLOG { protected: /* macros */ };

// my-timer.h
class myTimer : public myTimerLOG
{
  public:
    myTimer();
    ~myTimer() {}

    void start(uint32_t durationMs);
    bool check(void);
    void reset(void);

  private:
    /**
     * @brief TEMPO DE INICIO DO TIMER
     */
    uint32_t _startTime = NULL;

    /**
     * @brief DURACAO CONFIGURADA DO TIMER
     */
    uint32_t _duration = NULL;

    /**
     * @brief INDICA SE O TIMER ESTA ATIVO
     */
    bool _active = false;
};

// my-timer.cpp
myTimer::myTimer() {}

void myTimer::start(uint32_t durationMs)
{
    _startTime = millis();
    _duration  = durationMs;
    _active    = true;
    v_LOG_MYTIMER("Timer iniciado: %d ms", durationMs);
}

bool myTimer::check(void)
{
    bool expired = false;

    if(_active == true)
    {
        if((millis() - _startTime) >= _duration)
        {
            expired = true;
            _active = false;
            d_LOG_MYTIMER("Timer expirado");
        }
    }

    return expired;
}

void myTimer::reset(void)
{
    _startTime = NULL;
    _duration  = NULL;
    _active    = false;
}
```
