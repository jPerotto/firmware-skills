# firmware-skills

Skills para Claude Code aplicadas ao desenvolvimento de firmware embarcado para ESP8266/ESP32 com PlatformIO/Arduino.

---

## Sobre

Este repositorio contem um conjunto de **12 skills** que padronizam e automatizam o desenvolvimento de firmware embarcado. Cada skill e um guia especializado que o Claude Code utiliza como referencia ao criar, revisar ou refatorar codigo.

As skills cobrem desde a arquitetura geral do projeto ate detalhes de estilo de codigo, seguranca de memoria e tratamento de erros — garantindo consistencia entre projetos e desenvolvedores.

---

## Instalacao

1. Clone este repositorio:
   ```bash
   git clone https://github.com/<user>/firmware-skills.git
   ```

2. Copie as pastas de skill para o diretorio de skills do Claude Code:
   - **Global:** `~/.claude/skills/`
   - **Projeto:** `.claude/skills/` na raiz do projeto

3. Cada pasta contem um arquivo `SKILL.md` que o Claude Code carrega automaticamente quando o contexto e relevante.

---

## Skills Disponiveis

### Arquitetura e Estrutura

| Skill | Descricao |
|-------|-----------|
| [firmware-architecture](#firmware-architecture) | Arquitetura em camadas, estrutura de arquivos, sequencia de inicializacao e loop principal |
| [firmware-platformio](#firmware-platformio) | Configuracao de `platformio.ini`, ambientes de build, migracao do Arduino IDE |
| [firmware-hardware-config](#firmware-hardware-config) | Abstracao de hardware, `defines.h`, compilacao condicional, mapeamento de canais |

### Qualidade de Codigo

| Skill | Descricao |
|-------|-----------|
| [firmware-code-style](#firmware-code-style) | Guia de estilo C++ — nomenclatura, braces, classes, enums, structs, logging |
| [firmware-safe-functions](#firmware-safe-functions) | Design de funcoes seguras — SRP, guard clauses, limites de complexidade |
| [firmware-memory-safety](#firmware-memory-safety) | Seguranca de memoria — ownership, construtores, destrutores, checkDelete |
| [firmware-error-handling](#firmware-error-handling) | Tratamento de erros — retry, timeout, reconexao, degradacao graciosa, watchdog |
| [firmware-state-machine](#firmware-state-machine) | Maquinas de estado — enums, transicoes, timeout com `millis()`, persistencia |

### Documentacao e Revisao

| Skill | Descricao |
|-------|-----------|
| [firmware-doxygen-review](#firmware-doxygen-review) | Padrao Doxygen/Markdown — cabecalhos, funcoes, enums, classificacao por criticidade |
| [firmware-code-review](#firmware-code-review) | Checklist unificada de revisao — consolida todas as skills em itens verificaveis |

### Criacao e Templates

| Skill | Descricao |
|-------|-----------|
| [firmware-new-library](#firmware-new-library) | Como criar uma nova biblioteca PlatformIO — estrutura, templates, checklist |
| [firmware-templates](#firmware-templates) | Templates prontos — modulo, driver, state machine, command handler, firmware completo |

---

## Detalhamento das Skills

### firmware-architecture

Arquitetura geral de firmware para ESP8266/ESP32 com PlatformIO/Arduino.

**Quando usar:** Ao estruturar um novo firmware, decidir a organizacao em camadas, definir a ordem de inicializacao, ou desenhar a arquitetura de um projeto embarcado.

**Topicos cobertos:**
- Camadas do sistema (Application → Service → Data → Protocol/Hardware → Foundation)
- Estrutura de arquivos do projeto (`principals.h`, `variables.h`, `defines.h`, `prototypes.h`)
- Sequencia de inicializacao (`setup()`) com ordem de dependencias
- Loop principal nao-bloqueante com watchdog
- Padrao de variaveis globais (`extern` + `globals.cpp`)
- Persistencia de dados com `statusControl<T>`
- Design patterns recomendados (DI, State Machine, Strategy, Callback, Template)
- Scheduling nao-bloqueante via Ticker
- Arquitetura dual-broker MQTT
- Fluxo OTA update

---

### firmware-code-style

Guia de estilo de codigo C++ para firmware embarcado.

**Quando usar:** Sempre que criar ou revisar qualquer arquivo `.cpp`, `.h` ou `.hpp` do firmware.

**Topicos cobertos:**
- Cabecalho Doxygen obrigatorio (`@file`, `@author`, `@brief`, `@version`, `@date`, `@copyright`)
- Header guards no padrao `_[ESCOPO]_[NOME]_H` (nunca `#pragma once`)
- Includes com documentacao `/** @brief ... */` obrigatoria
- Estrutura de classe: `public` → `protected` → `private`
- Membros privados com prefixo `_` e inicializacao inline
- Construtores com validacao de ponteiros + `e_LOG("NullPointerException")`
- Destrutores que chamam apenas `checkDelete[X]()`
- Brace style Allman (chave na linha seguinte em todos os contextos)
- Funcoes bool: early return em guard clauses + variavel local + unico `return`
- Enums com sufixo `_e`, structs com sufixo `_t`
- Classe de log `log-[nome].h` com 5 niveis de macro (`e_`, `w_`, `i_`, `d_`, `v_`)

**Nomenclatura:**

| Item | Convencao | Exemplo |
|------|-----------|---------|
| Classes | camelCase | `wifiManager` |
| Metodos publicos | camelCase | `setup()`, `enablePortal()` |
| Membros privados | `_camelCase` | `_driver`, `_operationMode` |
| Enums (tipo) | `camelCase_e` | `operationMode_e` |
| Enum (membros) | `UPPER_SNAKE_CASE` | `MODE_CONNECT` |
| Structs (tipo) | `camelCase_t` | `controlConnect_t` |
| Defines/macros | `UPPER_SNAKE_CASE` | `TIME_RECONNECT` |
| Arquivos | `kebab-case` | `wifi-manager.h` |

---

### firmware-safe-functions

Design de funcoes seguras e previsiveis.

**Quando usar:** Sempre que criar, revisar ou refatorar qualquer funcao em `.cpp` ou `.h`.

**Topicos cobertos:**
- Responsabilidade unica (SRP) — funcao faz UMA coisa
- Tamanho maximo: ideal <= 60 linhas, aceitavel <= 100
- Aninhamento maximo: 3 niveis
- Guard clauses na ordem: `nullptr` → parametros → estado → negocio
- Maximo 5 validacoes no inicio da funcao
- Cadeias `if/else if`: maximo 4, sempre com `else` final
- Fluxo linear: validacoes primeiro, logica depois
- Comparacoes explicitas: `== false`, `== true`, `== nullptr`
- Parenteses em cada sub-expressao condicional
- Sem numeros magicos — usar `constexpr` ou `#define`
- Maximo 5 parametros (acima: usar struct)
- `(void)` explicito em funcoes sem parametros
- Restricoes de linguagem: proibido exceptions/RTTI, evitar lambda/auto/STL pesada

---

### firmware-memory-safety

Seguranca de memoria para firmware embarcado.

**Quando usar:** Sempre que criar classes com ponteiros, alocar memoria dinamica, gerenciar lifecycle de objetos, ou revisar ownership de recursos.

**Topicos cobertos:**
- **Ownership:** quem cria (`new`), destroi (`delete`) — regra fundamental
- Ponteiros injetados: validados mas NAO deletados no destrutor
- Ponteiros proprios: `checkDelete[X]()` em `protected:`
- Construtor: validar todos os ponteiros → atribuir membros → `new` internos → verificar
- Destrutor: chama apenas `checkDelete[X]()` — sem logica
- `checkDelete`: `if != nullptr` → `delete` → `= nullptr`
- Inicializacao inline de membros privados
- Alocacao centralizada em `setupDeclaredClass()`
- Stack vs heap — preferir stack quando possivel
- `DynamicJsonDocument` com tamanho explicito + `garbageCollect()`
- Validacao JSON: `isNull()` + `is<T>()` antes de `as<T>()`
- Strings constantes: `constexpr const char*` ou `#define`
- Prevencao de double-free, use-after-free, memory leak

---

### firmware-error-handling

Tratamento de erros e resiliencia para firmware 24/7.

**Quando usar:** Ao implementar retry, timeout, reconexao, recovery, degradacao graciosa, validacao de dados, ou qualquer mecanismo de resiliencia.

**Topicos cobertos:**
- `NullPointerException` como mensagem padrao em `e_LOG`
- Retry com contador limitado (`do/while` + constante de limite)
- Retry com fila (`commandQueue`) para execucao diferida
- Reconexao adaptativa em duas fases: rapida (agressiva) + lenta (backoff)
- Degradacao graciosa — hardware continua mesmo sem comunicacao
- Recovery apos reconexao — flag `sendStatesReconnect` + reenvio de estados
- Validacao JSON: `isNull()`, `is<T>()`, operador `|` para fallback
- Watchdog com auto-recuperacao
- Backup de credenciais em EEPROM + quick connect com IP cacheado
- Dual logging: local (serial) + remoto (MQTT)
- Status de erro como enum com conversao para string

---

### firmware-state-machine

Padroes para maquinas de estado (FSM) em firmware embarcado.

**Quando usar:** Ao implementar controle de estado, transicoes, timeout, persistencia, ou qualquer componente com estados discretos.

**Topicos cobertos:**
- Definicao de estados em `enum _e` com `UNDEFINED = 0`
- Struct de controle `_t` agrupando estado + timestamps + contadores + flags
- Transicoes via `switch/case` com chaves e `default` obrigatorio
- Conversao estado → string para logging com `FPSTR()`
- Timeout com `millis()` — basico, em struct, periodico e absoluto (deadline)
- Persistencia via `statusControl<T>` — carregar no setup, salvar apos mudanca
- Funcoes dedicadas de `resetState()` e `resetError()`
- Ticker para processamento periodico nao-bloqueante

---

### firmware-hardware-config

Abstracao de hardware e compilacao condicional.

**Quando usar:** Ao configurar GPIO, pinos, variantes de placa, compilacao condicional, mapeamento de canais, PWM/DAC, ou qualquer interacao direta com hardware.

**Topicos cobertos:**
- `defines.h` unico por projeto — organizado por categorias
- Pinos por revisao com `#if BOARD_REVISION` + `#error` obrigatorio
- `portable.h` com macros `p_` para abstracao ESP32/ESP8266
- Dois niveis de compilacao condicional: REVISAO + SETUP
- Preload de GPIO para preservar estado durante reboot
- Mapeamento de canais: `channelExecute_e` → struct `channelSetup_t` → pinos
- Conversao usuario → interno com funcao dedicada
- Controle de GPIO: relay, LED dual, DAC com rampa
- Build flags no `platformio.ini` por combinacao revisao x setup
- Constantes de hardware sem numeros magicos (prefixos `PIN_`, `TIME_`, `MIN_`, `MAX_`)

---

### firmware-platformio

Configuracao de projetos PlatformIO para ESP8266/ESP32.

**Quando usar:** Ao criar ou editar `platformio.ini`, configurar ambientes de build, migrar do Arduino IDE, definir `lib_deps`, `build_flags`, ou resolver problemas de build.

**Topicos cobertos:**
- Templates de `platformio.ini` para ESP32 e ESP8266 (4MB e 2MB)
- Ambientes separados: release (log=0) e debug (log=5)
- Build flags obrigatorias por plataforma
- Flags customizadas: `BOARD_REVISION`, `BOARD_SETUP`, `BOARD_DEBUG_LEVEL`
- Ambientes por revisao de hardware e variante de setup
- `lib_deps` e `lib_extra_dirs` — referencia de bibliotecas
- `library.properties` com categorias e arquiteturas
- Configuracao de flash, filesystem e particoes
- Board customizado (opcional)
- Migracao Arduino IDE → PlatformIO (passo a passo)
- Diferenca critica: escopo de variaveis (`extern`) e prototipos de funcao

---

### firmware-doxygen-review

Padrao de documentacao Doxygen/Markdown equilibrando clareza e concisao.

**Quando usar:** Ao criar, revisar ou auditar documentacao em `.h`, `.cpp`, `README.md`, enums, structs ou tabelas de hardware.

**Principio central:** Documentacao existe para agregar valor, nao para preencher espaco.

**Topicos cobertos:**
- README.md de firmware: `@page`, `@tableofcontents`, secoes Firmware e Hardware
- README.md de biblioteca: `@page`, `@see`, Dependencias, Recursos, Exemplos
- Header (.h): cabecalho Doxygen, includes documentados, membros privados (apenas se nao obvios)
- Source (.cpp): `@brief` em toda funcao, `@param`, `@return \c true/false`
- Enums e structs: `///<` inline em todos os membros (nunca `@param` no bloco)
- Variaveis globais: agrupamento por funcionalidade, documentar individualmente apenas criticas
- Classificacao por criticidade: Simples, Moderada, Critica (tabela de tags obrigatorias)
- Regras de qualidade: reprovar redundancia, genericidade, excesso de documentacao
- Cross-references Doxygen e tabelas de hardware

---

### firmware-code-review

Checklist unificada de revisao de codigo que consolida todas as skills.

**Quando usar:** Para validar qualquer codigo antes de merge, deploy ou finalizacao.

**Categorias da checklist:**
1. **Estilo de codigo** — formatacao, nomenclatura, declaracoes
2. **Funcoes** — SRP, tamanho, aninhamento, guard clauses, fluxo
3. **Memoria** — ownership, construtores, destrutores, checkDelete, JSON
4. **Maquina de estado** — enums, structs, transicoes, timeouts, persistencia
5. **Tratamento de erros** — null check, retry, reconexao, degradacao, watchdog
6. **Hardware** — defines.h, revisoes, preload, canais, portable.h
7. **Arquitetura** — camadas, setup, loop, variaveis globais
8. **Documentacao** — Doxygen, @brief, @param, @return, @warning
9. **Build** — platformio.ini, envs, lib_deps, library.properties

**Prioridade de verificacao:**

| Prioridade | Categorias |
|------------|------------|
| CRITICA | Memoria, Funcoes, Erros |
| ALTA | Estilo, Arquitetura, Hardware |
| MEDIA | State Machine, Build |
| PADRAO | Documentacao |

---

### firmware-new-library

Guia para criacao de novas bibliotecas C++ reutilizaveis para PlatformIO/Arduino.

**Quando usar:** Quando pedir para criar uma nova biblioteca, modulo ou componente reutilizavel.

**Gera os seguintes arquivos:**
```
libraries/minha-lib/
├── library.properties
├── README.md
├── examples/
│   └── BasicUsage/
│       └── BasicUsage.ino
└── src/
    ├── minha-lib.h          (header publico)
    ├── minha-lib.cpp        (implementacao)
    ├── log-minhaLib.h       (macros de log)
    └── principais.h         (enums e structs)
```

**Todos os templates seguem as skills de:** code-style, safe-functions, memory-safety e doxygen-review.

---

### firmware-templates

Templates de codigo prontos para os componentes mais comuns.

**Quando usar:** Ao criar um novo modulo, driver, state machine, command handler ou firmware completo.

**Templates disponiveis:**

| Template | Quando usar |
|----------|-------------|
| Modulo Padrao | Componente com ciclo `setup()` + `loop()` |
| Driver de Hardware | Controle de GPIO, PWM, DAC |
| State Machine | Componente com estados discretos |
| Command Handler | Processamento de comandos JSON via MQTT |
| Firmware Completo | Esqueleto de novo projeto (principals, variables, globals, main, loop, prototypes) |

---

## Relacionamento entre Skills

```
┌─────────────────────────────────────────────────────────┐
│                    firmware-code-review                   │
│              (consolida TODAS as skills)                  │
└───────┬──────┬──────┬──────┬──────┬──────┬──────┬───────┘
        │      │      │      │      │      │      │
   ┌────▼──┐ ┌─▼───┐ ┌▼────┐ ┌▼────┐ ┌▼───┐ ┌▼────┐ ┌▼──────┐
   │code-  │ │safe-│ │memo-│ │error│ │sta-│ │hard-│ │doxygen│
   │style  │ │func │ │ry   │ │hand │ │te  │ │ware │ │review │
   └───────┘ └─────┘ └─────┘ └─────┘ └────┘ └─────┘ └───────┘
        │      │      │      │      │      │
   ┌────▼──────▼──────▼──────▼──────▼──────▼──────┐
   │        firmware-architecture                   │
   │        firmware-platformio                     │
   └──────────────────────────────────────────────┘
        │                                    │
   ┌────▼────────────┐            ┌──────────▼────┐
   │ firmware-        │            │ firmware-      │
   │ new-library      │            │ templates      │
   └──────────────────┘            └───────────────┘
```

- **firmware-code-review** referencia todas as demais skills como itens de checklist
- **firmware-architecture** e **firmware-platformio** definem a base estrutural
- **firmware-new-library** e **firmware-templates** aplicam todas as skills na geracao de codigo
- As skills de qualidade (code-style, safe-functions, memory-safety, error-handling, state-machine, hardware-config, doxygen-review) sao independentes entre si e complementares

---

## Plataformas Suportadas

| Plataforma | MCU | Framework |
|------------|-----|-----------|
| ESP8266 | ESP12-F, TYWE3S | Arduino |
| ESP32 | ESP-WROOM-32 | Arduino |

**Toolchain:** PlatformIO

---

## Licenca

Uso interno. Skills personalizadas para padronizacao de desenvolvimento de firmware embarcado.
