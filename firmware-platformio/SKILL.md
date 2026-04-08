---
name: firmware-platformio
description: Configuracao de platformio.ini, library.properties, ambientes de build, e migracao do Arduino IDE para PlatformIO em projetos firmware embarcado (ESP8266/ESP32). Use quando criar ou editar platformio.ini, configurar ambientes, migrar projeto do Arduino IDE, definir lib_deps, build_flags, ou resolver problemas de build PlatformIO.
---

# PlatformIO — Configuracao de Projetos Firmware (ESP8266/ESP32)

Guia de boas praticas para configuracao de projetos PlatformIO com microcontroladores ESP.

---

## 1. ESTRUTURA PLATFORMIO DO PROJETO

```
project-name/
├── platformio.ini            ← configuracao de build
├── include/
│   ├── principals.h          ← UNICO arquivo com #include de libs
│   ├── variables.h           ← ponteiros globais (extern)
│   ├── defines.h             ← pinos, timings, constantes HW
│   └── prototypes.h          ← prototipos de funcoes cross-file
├── src/
│   ├── main.cpp              ← setup()
│   ├── globals.cpp           ← definicoes reais das variaveis (extern)
│   ├── loop.cpp              ← loop()
│   ├── mqtt.cpp              ← configuracao MQTT
│   ├── processMqtt.cpp       ← callbacks MQTT recebido
│   ├── executeCommand.cpp    ← execucao no hardware
│   ├── configDevice.cpp      ← configuracao de canais/HW
│   └── [especificos].cpp     ← modulos especificos do projeto
├── lib/                      ← libs locais do projeto (raramente)
├── boards/                   ← definicoes de boards customizados
│   └── custom-board.json     ← board customizado (opcional)
└── test/
```

---

## 2. PLATFORMIO.INI — TEMPLATE BASE

### 2.1 Projeto ESP32

```ini
; ============================================================
; PlatformIO - Projeto ESP32
; ============================================================

[platformio]
default_envs = release
src_dir = src
include_dir = include

; ============================================================
; Configuracoes compartilhadas entre ambientes
; ============================================================
[env]
framework = arduino
monitor_speed = 115200
upload_speed = 921600
lib_extra_dirs = ${PROJECT_DIR}/../libraries

; ============================================================
; [RELEASE] Compilacao para producao — sem logs
; ============================================================
[env:release]
platform = espressif32
board = esp32dev
board_build.f_cpu = 160000000L
board_build.flash_mode = dio
board_build.partitions = default.csv

build_flags =
    -D ARDUINO_ARCH_ESP32
    -D ESP32
    -D BOARD_REVISION='A'
    -D BOARD_SETUP=0
    -D BOARD_DEBUG_LEVEL=0
    -D F_CPU=160000000L
    -w

lib_deps =
    ; listar bibliotecas do projeto aqui
    bblanchon/ArduinoJson @ ^6.21.0

; ============================================================
; [DEBUG] Compilacao com logs verbose para desenvolvimento
; ============================================================
[env:debug]
extends = env:release

build_flags =
    ${env:release.build_flags}
    -D BOARD_DEBUG_LEVEL=5
    -D DEBUG_ESP_PORT=Serial
    -D CORE_DEBUG_LEVEL=0

monitor_filters = esp32_exception_decoder

; ============================================================
; [OTA] Upload via rede (Over-The-Air)
; ============================================================
[env:ota]
extends = env:release

upload_protocol = espota
upload_port = <DEVICE_IP>
upload_flags =
    --port=8266
```

### 2.2 Projeto ESP8266 (ESP12-F / 4MB)

```ini
; ============================================================
; PlatformIO - Projeto ESP8266
; ============================================================

[platformio]
default_envs = release
src_dir = src
include_dir = include

; ============================================================
; Configuracoes compartilhadas
; ============================================================
[env]
framework = arduino
monitor_speed = 115200
upload_speed = 921600
lib_extra_dirs = ${PROJECT_DIR}/../libraries

; ============================================================
; [RELEASE] Compilacao para producao — sem logs
; ============================================================
[env:release]
platform = espressif8266
board = esp12e
board_build.f_cpu = 160000000L
board_build.flash_mode = dio
board_build.filesystem = spiffs
board_build.ldscript = eagle.flash.4m1m.ld

build_flags =
    -D ARDUINO_ARCH_ESP8266
    -D ESP8266
    -D BOARD_REVISION='A'
    -D BOARD_SETUP=0
    -D BOARD_DEBUG_LEVEL=0
    -D NONOSDK22x_190703=1
    -D F_CPU=160000000L
    -D LWIP_OPEN_SRC
    -D TCP_MSS=536
    -D LWIP_FEATURES=1
    -D LWIP_IPV6=0
    -D FLASHMODE_DIO
    -w

lib_deps =
    ; listar bibliotecas do projeto aqui
    bblanchon/ArduinoJson @ ^6.21.0

; ============================================================
; [DEBUG] Compilacao com logs verbose
; ============================================================
[env:debug]
extends = env:release

build_flags =
    ${env:release.build_flags}
    -D BOARD_DEBUG_LEVEL=5
    -D DEBUG_ESP_PORT=Serial

monitor_filters = esp8266_exception_decoder
```

### 2.3 Projeto ESP8266 — 2MB flash (TYWE3S e similares)

```ini
[env:release]
platform = espressif8266
board = esp01_1m
board_build.f_cpu = 160000000L
board_build.flash_mode = dio
board_build.filesystem = spiffs
board_build.ldscript = eagle.flash.2m64.ld

build_flags =
    -D ARDUINO_ARCH_ESP8266
    -D ESP8266
    -D BOARD_REVISION='A'
    -D BOARD_SETUP=0
    -D BOARD_DEBUG_LEVEL=0
    -D F_CPU=160000000L
    -w
```

---

## 3. BUILD_FLAGS — REFERENCIA COMPLETA

### 3.1 Flags obrigatorias por plataforma

**ESP32:**
```
-D ARDUINO_ARCH_ESP32
-D ESP32
-D F_CPU=160000000L         (ou 240000000L se 240MHz)
```

**ESP8266:**
```
-D ARDUINO_ARCH_ESP8266
-D ESP8266
-D F_CPU=160000000L
-D NONOSDK22x_190703=1
-D LWIP_OPEN_SRC
-D TCP_MSS=536
-D LWIP_FEATURES=1
-D LWIP_IPV6=0
-D FLASHMODE_DIO
```

### 3.2 Flags customizadas do projeto

| Flag | Valores | Descricao |
|------|---------|-----------|
| `BOARD_DEBUG_LEVEL` | `0` (off) a `5` (verbose) | Nivel de log compilado |
| `BOARD_SETUP` | `0` (default), `1`, `2`... | Variante de setup (ex: 1ch, 3ch) |
| `BOARD_REVISION` | `'A'`, `'B'`, `'C'` | Revisao de hardware |
| `DEBUG_ESP_PORT` | `Serial`, `Serial1` | Porta serial para debug |
| `CORE_DEBUG_LEVEL` | `0` a `5` | Nivel de debug do ESP-IDF (ESP32) |

### 3.3 Niveis de log (BOARD_DEBUG_LEVEL)

| Valor | Nivel | Macros habilitadas |
|-------|-------|--------------------|
| `0` | NONE | nenhuma |
| `1` | ERROR | `e_LOG_*` |
| `2` | WARN | `e_LOG_*`, `w_LOG_*` |
| `3` | INFO | `e_LOG_*`, `w_LOG_*`, `i_LOG_*` |
| `4` | DEBUG | `e_LOG_*`, `w_LOG_*`, `i_LOG_*`, `d_LOG_*` |
| `5` | VERBOSE | todas (`e_`, `w_`, `i_`, `d_`, `v_LOG_*`) |

---

## 4. AMBIENTES POR REVISAO DE HARDWARE

Quando um projeto suporta multiplas revisoes de hardware, criar ambientes separados:

```ini
[env]
framework = arduino
lib_extra_dirs = ${PROJECT_DIR}/../libraries

; --- Revisao A ---
[env:revA-release]
platform = espressif32
board = esp32dev
build_flags =
    -D BOARD_REVISION='A'
    -D BOARD_SETUP=0
    -D BOARD_DEBUG_LEVEL=0
    ; ... demais flags

[env:revA-debug]
extends = env:revA-release
build_flags =
    ${env:revA-release.build_flags}
    -D BOARD_DEBUG_LEVEL=5
    -D DEBUG_ESP_PORT=Serial

; --- Revisao B ---
[env:revB-release]
platform = espressif32
board = esp32dev
build_flags =
    -D BOARD_REVISION='B'
    -D BOARD_SETUP=0
    -D BOARD_DEBUG_LEVEL=0
    ; ... demais flags

[env:revB-debug]
extends = env:revB-release
build_flags =
    ${env:revB-release.build_flags}
    -D BOARD_DEBUG_LEVEL=5
    -D DEBUG_ESP_PORT=Serial
```

---

## 5. AMBIENTES POR VARIANTE DE SETUP

Para projetos com multiplas configuracoes (ex: 1 canal, 3 canais):

```ini
; --- 1 Canal ---
[env:1ch-release]
extends = env:release
build_flags =
    ${env:release.build_flags}
    -D BOARD_SETUP=1

[env:1ch-debug]
extends = env:1ch-release
build_flags =
    ${env:1ch-release.build_flags}
    -D BOARD_DEBUG_LEVEL=5
    -D DEBUG_ESP_PORT=Serial

; --- 3 Canais ---
[env:3ch-release]
extends = env:release
build_flags =
    ${env:release.build_flags}
    -D BOARD_SETUP=3

[env:3ch-debug]
extends = env:3ch-release
build_flags =
    ${env:3ch-release.build_flags}
    -D BOARD_DEBUG_LEVEL=5
    -D DEBUG_ESP_PORT=Serial
```

---

## 6. LIB_DEPS — COMO REFERENCIAR BIBLIOTECAS

### 6.1 Bibliotecas locais (via lib_extra_dirs)

```ini
[env]
lib_extra_dirs = ${PROJECT_DIR}/../libraries

lib_deps =
    firmware-config          ; nome = campo "name" do library.properties
    debug-log
    utils
```

O PlatformIO resolve pelo `name=` do `library.properties` dentro de `lib_extra_dirs`.

### 6.2 Bibliotecas externas (PlatformIO Registry)

```ini
lib_deps =
    bblanchon/ArduinoJson @ ^6.21.0
    evert-arias/EasyButton @ ^2.0.1
```

### 6.3 Bibliotecas por path absoluto/relativo

```ini
lib_deps =
    ${PROJECT_DIR}/../libraries/firmware-config
    ${PROJECT_DIR}/../libraries/debug-log
```

**Preferencia:** Usar `lib_extra_dirs` + nome em vez de paths absolutos.

---

## 7. LIBRARY.PROPERTIES — TEMPLATE PADRAO

Toda biblioteca precisa de `library.properties` na raiz:

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

### Categorias validas

| Categoria | Quando usar |
|-----------|-------------|
| `Communication` | WiFi, MQTT, HTTP, RF, serial, protocolos |
| `Data Storage` | File system, SPIFFS, EEPROM, persistencia |
| `Signal Input/Output` | GPIO, touch, PWM, DAC, sensores |
| `Timing` | Timers, RTC, scheduling |
| `Device Control` | Controle de relays, dimmers, atuadores |
| `Other` | Utilitarios gerais |

### Arquiteturas

| Valor | Quando usar |
|-------|-------------|
| `architectures=esp8266,esp32` | Biblioteca compativel com ambas plataformas (padrao) |
| `architectures=esp32` | Biblioteca apenas ESP32 (ex: touch nativo) |
| `architectures=esp8266` | Biblioteca apenas ESP8266 (raro) |
| `architectures=*` | Qualquer plataforma Arduino |

---

## 8. FLASH E FILESYSTEM — CONFIGURACAO POR PLATAFORMA

### 8.1 ESP32 (4MB Flash)

```ini
board = esp32dev
board_build.flash_mode = dio
board_build.partitions = default.csv
```

Particoes padroes (`default.csv`):
| Particao | Tipo | Tamanho |
|----------|------|---------|
| nvs | data | 20KB |
| otadata | data | 8KB |
| app0 | app | 1280KB |
| app1 | app | 1280KB |
| spiffs | data | 1408KB |

Para SPIFFS menor:
```ini
board_build.partitions = min_spiffs.csv
```

### 8.2 ESP8266 — ESP12-F (4MB Flash, 1MB SPIFFS)

```ini
board = esp12e
board_build.ldscript = eagle.flash.4m1m.ld
board_build.filesystem = spiffs
```

### 8.3 ESP8266 — 2MB Flash, 64KB SPIFFS

```ini
board = esp01_1m
board_build.ldscript = eagle.flash.2m64.ld
board_build.filesystem = spiffs
```

### 8.4 Upload de filesystem

```ini
; Comando: pio run -t uploadfs
[env:release]
board_build.filesystem = spiffs
```

---

## 9. BOARD CUSTOMIZADO (OPCIONAL)

Para registrar um board customizado no PlatformIO:

Criar `boards/custom-board.json`:
```json
{
    "build": {
        "arduino": {
            "ldscript": "esp32_out.ld"
        },
        "core": "esp32",
        "f_cpu": "160000000L",
        "f_flash": "80000000L",
        "flash_mode": "dio",
        "mcu": "esp32",
        "variant": "esp32"
    },
    "connectivity": ["wifi", "bluetooth"],
    "frameworks": ["arduino", "espidf"],
    "name": "Custom Board Name",
    "upload": {
        "flash_size": "4MB",
        "maximum_ram_size": 327680,
        "maximum_size": 1966080,
        "speed": 921600
    },
    "url": "<REPOSITORY_URL>",
    "vendor": "<COMPANY_NAME>"
}
```

Uso:
```ini
[env:release]
board = custom-board          ; referencia boards/custom-board.json
```

**Na maioria dos casos, usar `esp32dev` ou `esp12e` com build_flags e suficiente.**

---

## 10. MIGRACAO ARDUINO IDE -> PLATFORMIO

### 10.1 Mapeamento de configuracao

| Arduino IDE (arduino.json) | PlatformIO (platformio.ini) |
|---------------------------|---------------------------|
| `"board": "esp32:esp32:..."` | `board = esp32dev` + build_flags |
| `"board": "esp8266:esp8266:..."` | `board = esp12e` + build_flags |
| `"configuration": "Setup=1ch"` | `-D BOARD_SETUP=1` |
| `"configuration": "Revision=RevA"` | `-D BOARD_REVISION='A'` |
| `"configuration": "LogLevel=Verbose"` | `-D BOARD_DEBUG_LEVEL=5` |
| `"configuration": "LogLevel=None"` | `-D BOARD_DEBUG_LEVEL=0` |
| `"output": "build"` | `.pio/build/` (automatico) |
| `"port": "/dev/ttyUSB0"` | `upload_port = /dev/ttyUSB0` |

### 10.2 Mapeamento de arquivos

| Arduino IDE | PlatformIO |
|-------------|-----------|
| `projeto.ino` | `src/main.cpp` (com `#include <Arduino.h>`) |
| `loop.ino` | `src/loop.cpp` |
| `MQTT.ino` | `src/mqtt.cpp` |
| `processMQTT.ino` | `src/processMqtt.cpp` |
| `execute.ino` | `src/executeCommand.cpp` |
| `config.ino` | `src/configDevice.cpp` |
| `principais.h` (na raiz) | `include/principals.h` |
| `variaveis.h` (na raiz) | `include/variables.h` |
| `defines.h` (na raiz) | `include/defines.h` |
| `~/Arduino/libraries/` | `lib_extra_dirs` no platformio.ini |

### 10.3 Passos da migracao

1. Criar `platformio.ini` com template da secao 2
2. Criar diretorios `src/` e `include/`
3. Renomear `projeto.ino` -> `src/main.cpp`
   - Adicionar `#include <Arduino.h>` no topo
4. Renomear demais `.ino` -> `src/*.cpp`
   - Cada `.cpp` precisa de `#include "principals.h"` se usar libs
5. Mover headers `.h` da raiz -> `include/`
6. Configurar `lib_extra_dirs` apontando para diretorio de bibliotecas
7. Mapear configuracoes do `arduino.json` para `build_flags`
8. Remover diretorio `.vscode/` antigo (arduino.json, c_cpp_properties.json)
9. Compilar com `pio run` e corrigir includes

### 10.4 Diferenca critica: escopo de variaveis

No Arduino IDE, todos os `.ino` sao concatenados em um unico arquivo antes da compilacao. No PlatformIO, cada `.cpp` e compilado separadamente.

**Problema:** Variaveis globais declaradas em `variables.h` serao duplicadas se incluidas em multiplos `.cpp`.

**Solucao:** Usar `extern` nos headers e definir em um unico `.cpp`:

```cpp
// include/variables.h — DECLARACOES (extern)
#ifndef _VARIABLES_PROJECT_H
#define _VARIABLES_PROJECT_H

#include "principals.h"

extern fileSystem*    systemData;
extern wifiManager*   wifi;
extern mqttManager*   mqttLocal;
extern bool           enablePortal;

#endif

// src/globals.cpp — DEFINICOES (unico arquivo)
#include "variables.h"

fileSystem*    systemData     = nullptr;
wifiManager*   wifi           = nullptr;
mqttManager*   mqttLocal      = nullptr;
bool           enablePortal   = false;
```

### 10.5 Diferenca critica: prototipos de funcao

No Arduino IDE, prototipos sao gerados automaticamente. No PlatformIO, funcoes definidas em outros `.cpp` precisam de prototipos explicitos.

**Solucao:** Criar header de prototipos:

```cpp
// include/prototypes.h
#ifndef _PROTOTYPES_PROJECT_H
#define _PROTOTYPES_PROJECT_H

// loop.cpp
void readInputs(void);

// mqtt.cpp
void setupLocalMQTTConnection(void);
void setupRemoteMQTTConnection(void);
void checkMQTTLocal(void);
void checkMQTTRemote(void);

// processMqtt.cpp
void processLocalMqtt(const char* topic, DynamicJsonDocument& jsonDoc);
void processRemoteMqtt(const char* topic, DynamicJsonDocument& jsonDoc);

// executeCommand.cpp
bool executeCommand(DynamicJsonDocument& jsonDoc);

// configDevice.cpp
void setupHardware(void);
void setupDeclaredClass(void);

#endif
```

Incluir em `principals.h`:
```cpp
#include "prototypes.h"
```

---

## 11. MONITOR SERIAL

```ini
[env:debug]
monitor_speed = 115200
monitor_filters = esp32_exception_decoder    ; ESP32
; monitor_filters = esp8266_exception_decoder  ; ESP8266

; Porta especifica (opcional)
monitor_port = /dev/ttyUSB0
; monitor_port = COM3                         ; Windows
```

---

## 12. UPLOAD

```ini
; --- Serial (padrao) ---
upload_speed = 921600
upload_port = /dev/ttyUSB0
; upload_port = COM3                          ; Windows

; --- OTA ---
upload_protocol = espota
upload_port = 192.168.1.100
upload_flags =
    --port=8266
```

---

## 13. CHECKLIST — NOVO PROJETO PLATFORMIO

- [ ] `platformio.ini` criado com template correto (ESP32 ou ESP8266)
- [ ] `lib_extra_dirs` apontando para diretorio de bibliotecas
- [ ] Todos os `lib_deps` listados (conferir com `principals.h`)
- [ ] `build_flags` com plataforma (`ESP32`/`ESP8266`) + flags do projeto
- [ ] Ambiente `release` com `BOARD_DEBUG_LEVEL=0`
- [ ] Ambiente `debug` com `BOARD_DEBUG_LEVEL=5` + `DEBUG_ESP_PORT=Serial`
- [ ] `src/main.cpp` com `#include <Arduino.h>` no topo
- [ ] `include/variables.h` usando `extern` (nao definicao direta)
- [ ] `src/globals.cpp` com definicoes reais das variaveis globais
- [ ] `include/prototypes.h` com prototipos de todas as funcoes cross-file
- [ ] Flash/filesystem configurado corretamente (ldscript ou partitions)
- [ ] `monitor_speed = 115200`
- [ ] Compilacao OK com `pio run`
- [ ] Upload OK com `pio run -t upload`

---

## 14. COMANDOS UTEIS

```bash
# Compilar ambiente padrao
pio run

# Compilar ambiente especifico
pio run -e debug
pio run -e revA-release

# Upload
pio run -t upload
pio run -e debug -t upload

# Upload filesystem (SPIFFS)
pio run -t uploadfs

# Monitor serial
pio device monitor

# Limpar build
pio run -t clean

# Listar boards disponiveis
pio boards esp32
pio boards esp8266

# Verificar dependencias
pio lib list
```
