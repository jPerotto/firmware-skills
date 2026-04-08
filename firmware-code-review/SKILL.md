---
name: firmware-code-review
description: Checklist unificada de revisao de codigo para firmware embarcado (ESP8266/ESP32, PlatformIO/Arduino). Use para validar qualquer codigo antes de merge, deploy ou finalizacao. Consolida regras de todas as skills em uma lista objetiva e verificavel organizada por categoria.
---

# Checklist de Code Review — Firmware C++ (PlatformIO/Arduino)

Checklist unificada para revisao de codigo em firmware embarcado ESP8266/ESP32.
Cada item e verificavel objetivamente — sem julgamento subjetivo.

---

## 1. ESTILO DE CODIGO (firmware-code-style)

### 1.1 Formatacao

- [ ] Allman braces (chave na linha seguinte) em **todos** os contextos
- [ ] Indentacao com **tab** (4 espacos de largura)
- [ ] `public:`, `protected:`, `private:` com 2 espacos de recuo
- [ ] Linha em branco entre blocos logicos distintos
- [ ] `switch/case` com chaves `{}` em cada case
- [ ] Chaves obrigatorias em blocos de uma linha (`if`, `for`, `while`)

### 1.2 Nomenclatura

- [ ] Classes em camelCase: `wifiManager`, `dataHandler`
- [ ] Metodos publicos: verbo + substantivo em camelCase: `setup()`, `enablePortal()`
- [ ] Metodos protected: prefixo funcional: `checkDeleteDriver()`, `setupMode()`
- [ ] Membros privados com prefixo `_`: `_driver`, `_operationMode`
- [ ] Enums tipo com sufixo `_e`: `operationMode_e`
- [ ] Enum membros em `UPPER_SNAKE_CASE`: `MODE_CONNECT`
- [ ] Structs tipo com sufixo `_t`: `controlConnect_t`
- [ ] Struct membros em `camelCase`: `statusConnect`, `timeStartConnect`
- [ ] Defines/macros em `UPPER_SNAKE_CASE`: `TIME_RECONNECT`, `PIN_RELAY_1`
- [ ] Arquivos em `kebab-case`: `wifi-manager.h`, `log-wifi.h`
- [ ] Booleanos com prefixo `is`, `has`, `can` ou `should`

### 1.3 Declaracoes

- [ ] `(void)` explicito em funcoes sem parametros
- [ ] Enums usados com escopo: `operationMode_e::MODE_A`
- [ ] Structs inicializados com chaves: `= {valor}`
- [ ] Ponteiros inicializados com `= nullptr`
- [ ] Numericos inicializados com `= NULL`
- [ ] Booleanos inicializados com `= false`
- [ ] Constantes de classe com `static constexpr` (nunca `static const` para valores compile-time)
- [ ] Macros de log (`e_LOG_X`, `w_LOG_X`) em vez de `Serial.print()`

---

## 2. FUNCOES (firmware-safe-functions)

- [ ] Responsabilidade unica (SRP) — funcao faz UMA coisa
- [ ] Tamanho <= 100 linhas (ideal <= 60)
- [ ] Aninhamento <= 3 niveis
- [ ] Cadeia `if/else if` termina com `else` final obrigatorio
- [ ] Max 4 verificacoes `if/else if` em cadeia
- [ ] Max 5 validacoes no inicio (guard clauses)
- [ ] Guard clauses na ordem: `nullptr` → parametros → estado → negocio
- [ ] Fluxo linear: validacoes primeiro, logica depois
- [ ] Funcoes `bool`: early return em guard clauses + variavel local + unico `return` na logica principal
- [ ] Comparacoes explicitas: `== false`, `== true`, `== nullptr`
- [ ] Parenteses em cada sub-expressao: `if((a == b) || (c == d))`
- [ ] Sem numeros magicos (`constexpr` ou `#define`)
- [ ] Max 5 parametros (acima: usar struct)
- [ ] Sem `delay()` no loop principal

---

## 3. MEMORIA (firmware-memory-safety)

- [ ] Ownership claro: quem cria (`new`), destroi (`delete`)
- [ ] Ponteiros injetados no construtor: validados mas NAO deletados no destrutor
- [ ] Ponteiros criados com `new`: tem `checkDelete[X]()` em `protected:`
- [ ] Construtor valida TODOS os ponteiros em um unico `if`
- [ ] Apos cada `new`: verificacao de `== nullptr`
- [ ] Destrutor chama APENAS `checkDelete[X]()` — sem logica
- [ ] Cada `checkDelete`: `if != nullptr` → `delete` → `= nullptr`
- [ ] Membros privados inicializados inline
- [ ] Sem `new` disperso — centralizado em construtor ou `setupDeclaredClass()`
- [ ] `DynamicJsonDocument` com tamanho explicito (constante nomeada)
- [ ] JSON validado antes de uso: `isNull()`, `is<T>()`
- [ ] Strings constantes como `constexpr const char*` ou `#define`
- [ ] `free()` para `malloc()`, `delete` para `new` — nunca misturar
- [ ] Sem reatribuicao de ponteiro sem deletar o anterior

---

## 4. MAQUINA DE ESTADO (firmware-state-machine)

- [ ] Estados em `enum` com sufixo `_e` e valor inicial (ex: `UNDEFINED = 0`)
- [ ] Struct de controle `_t` agrupa estado + timestamps + flags
- [ ] Transicoes via `switch/case` com `default` + log de erro
- [ ] Timeouts com `millis()` — nunca `delay()`
- [ ] Constantes de tempo nomeadas
- [ ] Estado persistido via `statusControl` quando deve sobreviver a reboot
- [ ] `setUpdateStatus()` apos cada mudanca de estado
- [ ] Funcao `resetState()` dedicada

---

## 5. TRATAMENTO DE ERROS (firmware-error-handling)

- [ ] `NullPointerException` logado com `e_LOG_X`
- [ ] Retry com contador limitado
- [ ] Fila de comandos para execucao diferida
- [ ] Reconexao adaptativa: fase rapida + backoff
- [ ] Contadores resetados apos reconexao
- [ ] Degradacao graciosa: hardware continua se comunicacao falhar
- [ ] Flag `sendStatesReconnect` por broker
- [ ] Erros operacionais logados local (`e_LOG`) E remoto (`logManager->sendLog`)
- [ ] Watchdog alimentado na primeira linha do `loop()`

---

## 6. HARDWARE (firmware-hardware-config)

- [ ] `defines.h` unico com constantes organizadas por categoria
- [ ] Pinos por revisao com `#if BOARD_REVISION`
- [ ] `#error` em todo `#else` final
- [ ] `preloadStatus()` antes de `setupDeclaredClass()`
- [ ] `channelExecute_e` como tipo logico para canais
- [ ] Struct `channelSetup_t` mapeia canal → pinos
- [ ] `portable.h` com macros `p_` para ESP32/ESP8266
- [ ] Build flags no `platformio.ini` para cada combinacao

---

## 7. ARQUITETURA (firmware-architecture)

- [ ] Dependencias top-down (camada superior depende da inferior)
- [ ] Setup na ordem: debug → declare → data → hw → wifi → mqtt → log
- [ ] Loop comeca com `utils.checkWDT()`
- [ ] Variaveis globais com `extern` em `variables.h`, definicao em `globals.cpp`
- [ ] `principals.h` com todos os `#include` documentados
- [ ] `prototypes.h` com prototipos de funcoes cross-file

---

## 8. DOCUMENTACAO (firmware-doxygen-review)

- [ ] Cabecalho Doxygen completo em todo `.h` (`@file`, `@author`, `@brief`, `@version`, `@date`, `@copyright`)
- [ ] Todo `#include` com `/** @brief ... */` acima
- [ ] Toda funcao no `.cpp` com `/** @brief ... */`
- [ ] `@brief` e `@param` em sentence case, sem acentos (`@warning` em MAIUSCULAS)
- [ ] Funcoes `bool` com dois `@return \c true/false` com condicoes especificas
- [ ] `@warning` em funcoes criticas (hardware, RF, timers)
- [ ] Sem documentacao redundante (nome != @brief)
- [ ] Enums e structs com `///<` inline em **todos** os membros (nunca `@param` no bloco)
- [ ] Documentacao proporcional a criticidade

---

## 9. BUILD (firmware-platformio)

- [ ] `platformio.ini` com envs `release` (level=0) e `debug` (level=5)
- [ ] `build_flags` com plataforma + `BOARD_REVISION` + `BOARD_SETUP`
- [ ] `lib_extra_dirs` apontando para `libraries/`
- [ ] `monitor_speed = 115200`
- [ ] `library.properties` em toda biblioteca
- [ ] Compilacao sem warnings (`-w` em release)

---

## 10. COMO USAR ESTA CHECKLIST

### Em code review (PR/merge):

1. Percorrer **cada arquivo modificado**
2. Verificar apenas os itens **relevantes** ao tipo de mudanca
3. Marcar items como APROVADO ou REPROVADO com justificativa
4. Para REPROVADO: indicar arquivo, linha e correcao sugerida

### Ao criar novo codigo:

1. Usar como referencia **antes** de escrever
2. Verificar ao finalizar, antes de considerar pronto

### Prioridade de verificacao:

| Prioridade | Categorias |
|------------|------------|
| **CRITICA** | Memoria (3), Funcoes (2), Erros (5) |
| **ALTA** | Estilo (1), Arquitetura (7), Hardware (6) |
| **MEDIA** | State Machine (4), Build (9) |
| **PADRAO** | Documentacao (8) |

### Formato de report:

```
## [Arquivo:Linha] — [APROVADO | REPROVADO]
**Categoria:** [numero da secao]
**Problema:** descricao objetiva
**Correcao:** codigo ou indicacao
```
