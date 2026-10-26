# Предисловие. Настройка проекта

**Цель этой книги** — построить серверное приложение на Rust, в котором бизнес-логика вынесена в независимые WebAssembly-компоненты. Мы создадим **Host-приложение** (на Axum), которое управляет жизненным циклом, ресурсами и безопасностью WASM-компонентов (Guest).

К концу книги вы получите `wasm-backend-lab` — работающую платформу, где компоненты можно независимо собирать, тестировать, заменять и ограничивать в правах без перезапуска основного сервера.

### 1. Методика обучения

Книга построена по принципу **Learning by Doing**. Каждая глава — это инженерный эксперимент с фиксированной последовательностью:
1. **Predict** — сформулируйте гипотезу до запуска.
2. **Run** — соберите и запустите код.
3. **Observe** — зафиксируйте реальный результат.
4. **Break** — намеренно нарушьте систему, чтобы увидеть её защиту.
5. **Understand** — объясните поведение и зафиксируйте доказательство (Evidence).

### 2. Неизменяемый каркас и изменяемые детали

Чтобы вам было проще ориентироваться, весь код в книге делится на две категории:

*   **Неизменяемый каркас (Infrastructure)**: Это структура workspace, файл `Cargo.toml`, `rust-toolchain.toml`, настройка линковки WASI (`wasmtime_wasi::p2::add_to_linker_sync`) и базовый запуск HTTP-сервера. Вы настраиваете это один раз в Предисловии. В последующих главах мы *не переписываем* эти файлы с нуля, а только добавляем новые зависимости или маршруты.
*   **Изменяемая логика (Business Logic)**: Это WIT-контракты (`world.wit`), код внутри компонентов (`lib.rs`) и обработчики в Host (`main.rs`). Именно здесь мы будем добавлять новые типы данных, ошибки, состояния и политики безопасности (capabilities).

### 3. Архитектурная модель

В нашей архитектуре четко разделены роли:
*   **Host (Rust + Axum)** — HTTP-соединения, маршрутизация, политики безопасности.
*   **Guest (WASM Component)** — чистая бизнес-логика.
*   **Contract (WIT)** — строгий интерфейс взаимодействия.
*   **Runtime (Wasmtime)** — безопасная среда исполнения с изоляцией памяти.

### 4. Инструментарий

Зафиксированные версии, с которыми проверена книга:
*   **Rust**: `1.99.1` с target `wasm32-wasip2`
*   **Wasmtime**: `49.0.3`
*   **cargo-component**: `0.21.1`
*   **wasm-tools**: `1.262.0`

Установка:
```bash
rustup target add wasm32-wasip2
cargo install --locked wasm-tools cargo-component wasmtime-cli
```

Проверьте окружение:
```bash
rustc --version
wasmtime --version
cargo component --version
wasm-tools --version
```

### 5. Создание проекта

Создайте каталог проекта и перейдите в него:
```bash
mkdir wasm-backend-lab
cd wasm-backend-lab
```

Файл `Cargo.toml` объявляет workspace и связывает все пакеты проекта:
```toml
# Cargo.toml
[workspace]
resolver = "2"
members = [
    "comparison/native",
    "host",
    "components/*",
]

[workspace.package]
version = "0.1.0"
edition = "2021"
```

Файл `rust-toolchain.toml` фиксирует версию языка:
```toml
# rust-toolchain.toml
[toolchain]
channel = "1.99.1"
targets = ["wasm32-wasip2"]
```

### 6. Структура проекта

К концу Предисловия проект имеет следующую структуру:
```
wasm-backend-lab/
├── Cargo.toml
├── rust-toolchain.toml
│
├── host/
│   ├── Cargo.toml
│   └── src/
│       └── main.rs
│
├── components/
│   └── text-engine/
│       ├── Cargo.toml
│       └── src/
│           └── lib.rs
│
├── wit/
│   └── text-engine/
│       └── world.wit
│
├── comparison/
│   └── native/
│       ├── Cargo.toml
│       └── src/
│           └── main.rs
│
└── scripts/
    └── compare.sh
```

Каждый каталог появляется ровно тогда, когда в нём возникает файл. По мере роста проекта будут добавляться новые каталоги (`manifests/`, `tests/`, `examples/`) — но только в тех главах, где они действительно понадобятся.

### 7. Первый компонент: контракт WIT

Любой компонент начинается с контракта. Создайте каталог и файл:
```bash
mkdir -p wit/text-engine
```

```wit
// wit/text-engine/world.wit
package docs:text-engine@0.1.0;

interface processor {
    process: func(input: string) -> string;
}

world text-engine {
    export processor;
}
```

Контракт определяет единственную функцию `process`, принимающую строку и возвращающую строку. Всё остальное — детали реализации, которые могут меняться без изменения контракта.

### 8. Сборка компонента

Создайте каталог компонента:
```bash
mkdir -p components/text-engine/src
```

```toml
# components/text-engine/Cargo.toml
[package]
name = "text-engine"
version.workspace = true
edition.workspace = true

[lib]
crate-type = ["cdylib"]

[dependencies]
wit-bindgen = "0.63"
```

Реализация генерируется из WIT и подключается через макрос `wit_bindgen::generate!`:
```rust
// components/text-engine/src/lib.rs
wit_bindgen::generate!({
    path: "../../wit/text-engine",
    world: "text-engine",
});

struct TextEngine;

impl exports::docs::text_engine::processor::Guest for TextEngine {
    fn process(input: String) -> String {
        input.to_uppercase()
    }
}

export!(TextEngine);
```

Соберите компонент:
```bash
cd components/text-engine
cargo component build
cd ../..
```
После сборки в `target/wasm32-wasip1/debug/` появится файл `text_engine.wasm`.

### 9. Baseline на чистом Rust

Чтобы убедиться, что наш компонент делает именно то, что заявлено в контракте, создадим эквивалентную реализацию без WASM. Она послужит эталоном для сравнения.

```bash
mkdir -p comparison/native/src
```

```toml
# comparison/native/Cargo.toml
[package]
name = "native-baseline"
version.workspace = true
edition.workspace = true

[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
serde_json = "1"
```

```rust
// comparison/native/src/main.rs
use axum::{routing::post, Json, Router};
use serde_json::{json, Value};

async fn process(Json(payload): Json<Value>) -> Json<Value> {
    let input = payload["input"].as_str().unwrap_or("");
    let output = input.to_uppercase();
    Json(json!({ "output": output }))
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/process", post(process));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3001").await.unwrap();
    println!("Native baseline listening on http://127.0.0.1:3001");
    axum::serve(listener, app).await.unwrap();
}
```

### 10. Host-приложение

Host принимает HTTP-запрос, загружает WASM-компонент через Wasmtime и возвращает результат.

```bash
mkdir -p host/src
```

```toml
# host/Cargo.toml
[package]
name = "host"
version.workspace = true
edition.workspace = true

[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
serde_json = "1"
wasmtime = { version = "49.0.3", features = ["component-model"] }
wasmtime-wasi = "49.0.3"
anyhow = "1.0"
```

```rust
// host/src/main.rs
use axum::{routing::post, Json, Router};
use serde_json::{json, Value};
use std::sync::Arc;
use wasmtime::component::{Component, Linker};
use wasmtime::{Engine, Store};
use wasmtime_wasi::{WasiCtx, WasiCtxBuilder, WasiCtxView, WasiView};

wasmtime::component::bindgen!({
    path: "../wit/text-engine",
    world: "text-engine",
});

struct MyState {
    ctx: WasiCtx,
    table: wasmtime::component::ResourceTable,
}

impl WasiView for MyState {
    fn ctx(&mut self) -> WasiCtxView<'_> {
        WasiCtxView {
            ctx: &mut self.ctx,
            table: &mut self.table,
        }
    }
}

struct AppState {
    engine: Engine,
    component: Component,
    linker: Linker<MyState>,
}

async fn process(
    axum::extract::State(state): axum::extract::State<Arc<AppState>>,
    Json(payload): Json<Value>,
) -> Json<Value> {
    let input = payload["input"].as_str().unwrap_or("").to_string();

    let wasi = WasiCtxBuilder::new().build();
    let mut store = Store::new(
        &state.engine,
        MyState {
            ctx: wasi,
            table: wasmtime::component::ResourceTable::new(),
        },
    );

    let bindings = match TextEngine::instantiate(&mut store, &state.component, &state.linker) {
        Ok(b) => b,
        Err(e) => {
            eprintln!("Failed to instantiate: {e:?}");
            return Json(json!({ "error": "Instantiation failed" }));
        }
    };

    let result = bindings
        .docs_text_engine_processor()
        .call_process(&mut store, &input);

    match result {
        Ok(output) => Json(json!({ "output": output })),
        Err(e) => {
            eprintln!("WASM execution error: {e:?}");
            Json(json!({ "error": "Execution failed" }))
        }
    }
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let engine = Engine::default();
    let mut linker = Linker::new(&engine);
    wasmtime_wasi::p2::add_to_linker_sync(&mut linker)?;

    // В Предисловии используем debug-сборку для скорости. В главах про производительность переключимся на release.
    let wasm_path = std::path::Path::new("target/wasm32-wasip1/debug/text_engine.wasm");

    if !wasm_path.exists() {
        eprintln!("Error: WASM component not found at {:?}.", wasm_path);
        eprintln!("Run `cd components/text-engine && cargo component build` first.");
        std::process::exit(1);
    }

    let component = Component::from_file(&engine, wasm_path)?;
    let state = Arc::new(AppState { engine, component, linker });

    let app = Router::new().route("/process", post(process)).with_state(state);
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await?;
    println!("WASM Host listening on http://127.0.0.1:3000");
    axum::serve(listener, app).await?;

    Ok(())
}
```

### 11. Скрипт сравнения (ускоренная сборка)

Чтобы не ждать минуты при каждой итерации, скрипт использует `debug`-профиль. `release`-сборка понадобится позже, когда мы будем замерять производительность.

```bash
mkdir -p scripts
```

```bash
#!/bin/bash
# scripts/compare.sh
set -e

cd "$(dirname "$0")/.."

echo "=== 1. Building Native Baseline ==="
cargo build --package native-baseline --quiet

echo "=== 2. Building WASM Component ==="
(cd components/text-engine && cargo component build --quiet)

echo "=== 3. Building WASM Host ==="
cargo build --package host --quiet

echo "=== 4. Starting Servers ==="
./target/debug/native-baseline > /tmp/native.log 2>&1 &
NATIVE_PID=$!

./target/debug/host > /tmp/wasm.log 2>&1 &
WASM_PID=$!

sleep 2

cleanup() {
    kill $NATIVE_PID $WASM_PID 2>/dev/null || true
}
trap cleanup EXIT

echo "=== 5. Running Comparison ==="
INPUT_JSON='{"input": "hello wasm infrastructure"}'

echo "--- Native Baseline Response ---"
NATIVE_RESP=$(curl -s -X POST http://127.0.0.1:3001/process -H "Content-Type: application/json" -d "$INPUT_JSON")
echo "$NATIVE_RESP"

echo "--- WASM Host Response ---"
WASM_RESP=$(curl -s -X POST http://127.0.0.1:3000/process -H "Content-Type: application/json" -d "$INPUT_JSON")
echo "$WASM_RESP"

echo "=== 6. Verification ==="
if [ "$NATIVE_RESP" == "$WASM_RESP" ]; then
    echo "SUCCESS: Both implementations returned the same result."
else
    echo "FAILURE: Results differ."
    echo "--- WASM Host Log ---"
    cat /tmp/wasm.log
    exit 1
fi
```

Запустите скрипт:
```bash
chmod +x scripts/compare.sh
./scripts/compare.sh
```

**Ожидаемый вывод:**
```
=== 5. Running Comparison ===
--- Native Baseline Response ---
{"output":"HELLO WASM INFRASTRUCTURE"}
--- WASM Host Response ---
{"output":"HELLO WASM INFRASTRUCTURE"}
=== 6. Verification ===
SUCCESS: Both implementations returned the same result.
```

### 12. Проверка сборки

Убедитесь, что весь workspace собирается без предупреждений:
```bash
cargo check
```
Команда должна завершиться со статусом `Finished` без ошибок.

### Итог Предисловия

Мы построили фундамент:
* **Workspace** с фиксированной версией Rust.
* **WIT-контракт** — источник истины о возможностях компонента.
* **WASM-компонент** `text-engine`, собранный в `debug`.
* **Host-приложение** на Axum + Wasmtime.
* **Native baseline** — эталонная реализация.
* **Скрипт `compare.sh`**, доказывающий эквивалентность.

Теперь у нас есть неизменяемый каркас и измерительная лаборатория. Мы готовы к первому эксперименту: в следующей главе мы намеренно нарушим контракт и посмотрим, как система отреагирует на несовпадение типов.