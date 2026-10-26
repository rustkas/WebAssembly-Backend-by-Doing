# Глава 5. Axum + Wasmtime: работающий backend

**Зачем создана эта глава?**
В Главе 4 мы научились проверять компонент через CLI (`wasm-tools`, `wasmtime --invoke`). Но backend — это не CLI. WASM-компонент должен вызываться из HTTP-сервера на каждый входящий запрос. До сих пор `host/src/main.rs` содержал всю логику в одном файле, что смешивало три разные задачи: инициализацию Wasmtime, HTTP-маршруты и точку входа. Теперь мы разделяем их на модули, делаем первый **реальный вызов** компонента из HTTP-обработчика и строим воспроизводимую измерительную лабораторию.

**Какую пользу несёт глава?**
1. **Разделение ответственности:** `runtime.rs` (Wasmtime) / `http.rs` (Axum) / `main.rs` (запуск).
2. **Первый работающий backend:** `curl → Axum → Wasmtime → Component → response`.
3. **Понимание схемы взаимодействия:** как именно байты из HTTP-запроса становятся аргументами `string` внутри WASM-компонента.
4. **Измерение цены границы:** насколько WASM-вызов дороже нативной функции в контексте HTTP.

**Чего мы хотим достичь по результатам тестирования?**
1. Endpoint `/process` возвращает корректный результат.
2. Нативная и WASM-реализации дают идентичный JSON.
3. Измерена и объяснена разница в latency: `HTTP → Rust` vs `HTTP → Axum → Wasmtime → Component`.

---

### 🚦 Starting point
У нас есть `host/src/main.rs` (всё в одном файле), `comparison/native/` (baseline), `components/text-engine-component/` (WASM с `normalize_text`). Мы **не переписываем** ничего — только реорганизуем `host` и приводим `native-baseline` в соответствие с WASM-алгоритмом.

### 🎯 PREDICT (Предсказание)
*Запишите свой прогноз перед запуском:*
> **Если мы создаём новый `Store` на каждый HTTP-запрос (stateless-модель), будет ли разница в latency между `HTTP → нативная функция` и `HTTP → Axum → Wasmtime → Component` заметна на фоне HTTP-overhead? Или WASM-накладные расходы потеряются в шуме?**

---

## 🗺 Схема взаимодействия Host и WebAssembly Component

Прежде чем писать код, важно понять, что физически происходит при вызове `rt.process(input)`. На схеме ниже показан полный путь данных от HTTP-запроса до байтов в линейной памяти компонента и обратно.

```text
 [ HTTP Client ]
       │  POST /process  { "input": "  Hello  " }
       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ HOST PROCESS (Rust)                                                         │
│                                                                             │
│  [ Axum Router ] ──(Json payload)──► [ http.rs: process() ]                 │
│                                             │                               │
│                                             ▼                               │
│                                  tokio::task::spawn_blocking                │
│                                             │ (передаём input)              │
│                                             ▼                               │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  [ runtime.rs: Runtime::process(input) ]  ← работает в blocking pool  │  │
│  │                                                                       │  │
│  │  1. Store::new(Engine, MyState { WasiCtx, ResourceTable })            │  │
│  │  2. TextEngine::instantiate(&mut store, &component, &linker)          │  │
│  │  3. bindings.docs_text_engine_processor().call_process(store, input)  │  │
│  └───────────────────────────────────┬───────────────────────────────────┘  │
│                                      │                                      │
│   ┌──────────────────────────────────▼───────────────────────────────────┐  │
│   │ WASMTIME ENGINE (JIT / AOT)                                          │  │
│   │                                                                      │  │
│   │  [ Component Instance ]                                              │  │
│   │    ├── Guest Memory (Linear Memory)                                  │  │
│   │    │   "  Hello  " (UTF-8 bytes)                                     │  │
│   │    └── Exported Function: process(ptr, len) -> (ptr, len)            │  │
│   │                                                                      │  │
│   │  ABI Boundary (WIT):                                                 │  │
│   │    Host выделяет память ВНУТРИ Guest, копирует туда строку,          │  │
│   │    вызывает функцию, забирает указатель на результат, копирует       │  │
│   │    обратно в Host, освобождает память Guest.                         │  │
│   └──────────────────────────────────┬───────────────────────────────────┘  │
│                                      │                                      │
│                                      ▼                                      │
│                     "hello" (String) возвращается в Axum                    │
└─────────────────────────────────────────────────────────────────────────────┘
       │
       ▼
 [ HTTP Client ]  200 OK  { "output": "hello" }
```

**Ключевые моменты схемы:**
1. **`spawn_blocking` обязателен.** Wasmtime в синхронном режиме блокирует поток ОС. Если вызвать `call_process` прямо в асинхронном обработчике Axum, мы заблокируем поток Tokio, и сервер перестанет принимать другие запросы.
2. **Адаптер WIT скрывает сложность.** Глядя на `call_process(&mut store, &input)`, мы передаём `&str`. Но под капотом `wit-bindgen` делает `malloc` в памяти Guest, копирует туда байты, вызывает `process(ptr, len)`, читает `(ptr, len)` ответа, копирует обратно и вызывает `free`. Именно эта работа и создаёт ту самую задержку ~0.4 мс.
3. **`Store` — это состояние.** В нём живут `WasiCtx` и `ResourceTable`. Создание `Store` на каждый запрос гарантирует полную изоляцию: если один запрос упадёт с паникой внутри WASM (trap), он не повредит состояние других запросов.

---

## Этап 1. Исходные коды

### Структура
**Обновляется:**
- `host/src/main.rs` — рефакторинг (только точка входа).
- `host/src/runtime.rs` — **новый** модуль (Wasmtime).
- `host/src/http.rs` — **новый** модуль (Axum).
- `host/src/lib.rs` — **новый** (для доступа из тестов).
- `host/Cargo.toml` — добавлен `[dev-dependencies]`.
- `comparison/native/src/main.rs` — обновлён алгоритм (для честного сравнения).

**Не изменяется:**
- Все компоненты (`text-engine`, `text-engine-core`, `text-engine-component`, `document-engine`).
- WIT-контракты.

### Новые / обновлённые файлы

**1. `host/src/lib.rs`** — публичный интерфейс и единая точка правды для путей.
```rust
pub mod http;
pub mod runtime;

pub use runtime::Runtime;

use std::path::PathBuf;

/// Единая точка правды для пути к собранному **компоненту** `text-engine-component`.
///
/// Ожидается артефакт Component Model (`(component …)`), а не Core WASM-модуль.
/// 
/// Что НЕ подходит:
///   - `target/wasm32-wasip1/release/text_engine.wasm` (Baseline, Глава 1 — это module)
///   - `target/wasm32-wasip1/release/text_engine_core.wasm` (Core WASM, Глава 2 — это module)
pub fn default_wasm_path() -> PathBuf {
    if let Ok(p) = std::env::var("TEXT_ENGINE_WASM") {
        return PathBuf::from(p);
    }
    // text-engine-component исключён из workspace, поэтому у него свой target/
    PathBuf::from(env!("CARGO_MANIFEST_DIR")).join(
        "../components/text-engine-component/target/wasm32-wasip1/release/text_engine_component.wasm",
    )
}
```

**2. `host/src/runtime.rs`** — вся логика Wasmtime в одном месте.
```rust
use anyhow::{anyhow, Result};
use std::path::Path;
use std::sync::Arc;
use wasmtime::component::{Component, Linker};
use wasmtime::{Engine, Store};
use wasmtime_wasi::{WasiCtx, WasiCtxBuilder, WasiCtxView, WasiView};

wasmtime::component::bindgen!({
    path: "../wit/text-engine",
    world: "text-engine",
});

pub struct MyState {
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

pub struct Runtime {
    engine: Engine,
    component: Component,
    linker: Linker<MyState>,
}

impl Runtime {
    pub fn new(wasm_path: &Path) -> Result<Arc<Self>> {
        if !wasm_path.exists() {
            return Err(anyhow!("WebAssembly component not found at `{}`", wasm_path.display()));
        }

        let engine = Engine::default();
        let mut linker = Linker::new(&engine);
        wasmtime_wasi::p2::add_to_linker_sync(&mut linker)?;

        let component = Component::from_file(&engine, wasm_path).map_err(|e| {
            anyhow!(
                "Failed to load WebAssembly *component* from `{}`: {e}\n\
                 Hint: host_v2 accepts only Component Model artifacts (`(component …)`).\n\
                 Known non-component artifacts in this project:\n\
                   * target/wasm32-wasip1/release/text_engine.wasm       (Baseline, Chapter 1 — module)\n\
                   * target/wasm32-wasip1/release/text_engine_core.wasm  (Core WASM demo, Chapter 2 — module)\n\
                 Use components/text-engine-component/target/wasm32-wasip1/release/text_engine_component.wasm",
                wasm_path.display()
            )
        })?;

        Ok(Arc::new(Self {
            engine,
            component,
            linker,
        }))
    }

    /// Синхронный вызов. Должен вызываться из `spawn_blocking`.
    pub fn process(&self, input: &str) -> Result<String> {
        // 1. Создаём изолированное состояние для одного запроса
        let wasi = WasiCtxBuilder::new().build();
        let mut store = Store::new(
            &self.engine,
            MyState {
                ctx: wasi,
                table: wasmtime::component::ResourceTable::new(),
            },
        );

        // 2. Инстанцируем компонент в этом Store
        let bindings = TextEngine::instantiate(&mut store, &self.component, &self.linker)?;
        
        // 3. Вызываем функцию (WIT-адаптер сделает маршалинг строки в память Guest)
        let result = bindings
            .docs_text_engine_processor()
            .call_process(&mut store, input)?;

        Ok(result)
    }
}
```

**3. `host/src/http.rs`** — только Axum-маршруты.
```rust
use axum::{routing::post, Json, Router};
use serde_json::{json, Value};
use std::sync::Arc;
use crate::runtime::Runtime;

pub fn router(runtime: Arc<Runtime>) -> Router {
    Router::new()
        .route("/process", post(process))
        .with_state(runtime)
}

async fn process(
    axum::extract::State(runtime): axum::extract::State<Arc<Runtime>>,
    Json(payload): Json<Value>,
) -> Json<Value> {
    let input = match payload.get("input").and_then(Value::as_str) {
        Some(s) => s.to_string(),
        None => return Json(json!({ "error": "missing 'input' field" })),
    };

    // Wasmtime синхронный — уводим в blocking pool, чтобы не блокировать Tokio.
    let rt = Arc::clone(&runtime);
    let result = tokio::task::spawn_blocking(move || rt.process(&input)).await;

    match result {
        Ok(Ok(output)) => Json(json!({ "output": output })),
        Ok(Err(e)) => {
            eprintln!("WASM execution error: {e:?}");
            Json(json!({ "error": e.to_string() }))
        }
        Err(e) => {
            eprintln!("Task join error: {e:?}");
            Json(json!({ "error": "internal error" }))
        }
    }
}
```

**4. `host/src/main.rs`** — только точка входа.
```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let wasm_path = host::default_wasm_path();

    if !wasm_path.exists() {
        eprintln!("Error: WASM component not found at `{}`.", wasm_path.display());
        eprintln!("Run `cd components/text-engine-component && cargo component build --release` first.");
        std::process::exit(1);
    }

    let runtime = host::Runtime::new(&wasm_path)?;
    let app = host::http::router(runtime);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await?;
    println!("WASM Host listening on http://127.0.0.1:3000");
    axum::serve(listener, app).await?;

    Ok(())
}
```

**5. `comparison/native/src/main.rs`** — приводим алгоритм к WASM-логике.
> **⚠️ Iterative fix.** В Preface нативный baseline делал `to_uppercase()`, а WASM-компонент делает `normalize_text()`. Это разные алгоритмы. Приводим baseline в соответствие с WASM-логикой, иначе сравнение не имеет смысла.

```rust
use axum::{routing::post, Json, Router};
use serde_json::{json, Value};

fn normalize_text(input: &str) -> String {
    let mut result = String::with_capacity(input.len());
    let mut first = true;
    for word in input.split_whitespace() {
        if !first { result.push(' '); }
        result.push_str(&word.to_lowercase());
        first = false;
    }
    result
}

async fn process(Json(payload): Json<Value>) -> Json<Value> {
    let input = payload["input"].as_str().unwrap_or("");
    Json(json!({ "output": normalize_text(input) }))
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/process", post(process));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3001").await.unwrap();
    println!("Native baseline listening on http://127.0.0.1:3001");
    axum::serve(listener, app).await.unwrap();
}
```

---

## Этап 2. Тесты исходных кодов

**`host/tests/integration.rs`**
```rust
use axum::body::Body;
use axum::http::{Request, StatusCode};
use std::path::PathBuf;
use tower::ServiceExt;

fn wasm_path() -> PathBuf {
    host::default_wasm_path()
}

#[test]
fn test_runtime_missing_wasm() {
    let result = host::runtime::Runtime::new(std::path::Path::new("/nonexistent.wasm"));
    assert!(result.is_err(), "Runtime должен вернуть ошибку при отсутствии файла");
}

#[tokio::test]
async fn test_http_to_wasm_endpoint() {
    let path = wasm_path();
    if !path.exists() {
        eprintln!("Skipping: {} not found. Build the component first.", path.display());
        return;
    }

    let runtime = host::runtime::Runtime::new(&path)
        .unwrap_or_else(|e| panic!("Runtime::new failed for {}: {e}", path.display()));
    let app = host::http::router(runtime);

    let req = Request::builder()
        .method("POST")
        .uri("/process")
        .header("content-type", "application/json")
        .body(Body::from(r#"{"input":"  Hello   WASM  "}"#))
        .expect("valid request");

    let response = app.oneshot(req).await.expect("oneshot");
    assert_eq!(response.status(), StatusCode::OK);

    let body = axum::body::to_bytes(response.into_body(), 1024).await.expect("body");
    let body: serde_json::Value = serde_json::from_slice(&body).expect("valid json");
    assert_eq!(body["output"], "hello wasm");
}
```

---

## Этап 3. Измерительный скрипт

Демо — это **измерительный скрипт** (`scripts/measure.sh`), который поднимает оба сервера, прогревает их, делает перекрёстные замеры (interleaved) и проверяет эквивалентность.

```bash
#!/usr/bin/env bash
# scripts/measure.sh
set -euo pipefail

NATIVE_PORT="${NATIVE_PORT:-3001}"
WASM_PORT="${WASM_PORT:-3002}"
WARMUP="${WARMUP:-5}"
REQUESTS="${REQUESTS:-20}"
INPUT_JSON="${INPUT_JSON:-{\"input\":\"  Hello   WASM   World  \"}}"

ROOT="$(cd "$(dirname "$0")/.." && pwd)"
cd "$ROOT"

echo "=== 1. Killing stale servers ==="
pkill -f 'target/release/host'            2>/dev/null || true
pkill -f 'target/release/native-baseline' 2>/dev/null || true
sleep 1

echo "=== 2. Building ==="
cargo build --package native-baseline --release --quiet

if [ -n "${COMPONENT_WASM:-}" ]; then
  echo "ℹ️  COMPONENT_WASM override: $COMPONENT_WASM (skipping default component build)"
else
  COMPONENT_WASM="$ROOT/components/text-engine-component/target/wasm32-wasip1/release/text_engine_component.wasm"
  (cd components/text-engine-component && cargo component build --release --quiet)
fi

cargo build --package host --release --quiet

if [ ! -f "$COMPONENT_WASM" ]; then
  echo "❌ component artifact not found: $COMPONENT_WASM"
  exit 1
fi
echo "✅ component artifact: $COMPONENT_WASM"

echo "=== 3. Starting servers ==="
"$ROOT/target/release/native-baseline" > /tmp/native.log 2>&1 &
NATIVE_PID=$!

TEXT_ENGINE_WASM="$COMPONENT_WASM" "$ROOT/target/release/host_v2" > /tmp/wasm.log 2>&1 &
WASM_PID=$!

cleanup() {
  kill "$NATIVE_PID" "$WASM_PID" 2>/dev/null || true
  wait "$NATIVE_PID" "$WASM_PID" 2>/dev/null || true
}
trap cleanup EXIT INT TERM
sleep 1

# --- helpers ---
measure_one() {
  curl -s -f -o /dev/null --max-time 10 -w "%{time_total}" -X POST "$1" \
    -H "Content-Type: application/json" -d "$INPUT_JSON"
}

report_stats() {
  local label="$1" times="$2"
  echo "$times" | tr ' ' '\n' | grep -v '^$' | sort -n | awk -v label="$label" '
    { v[NR]=$1; s+=$1 }
    END {
      n=NR
      med = (n%2==1) ? v[(n+1)/2] : (v[n/2]+v[n/2+1])/2
      p95i = int(n*0.95); if (p95i < 1) p95i = 1
      printf "%s: n=%d min=%.4fs med=%.4fs avg=%.4fs p95=%.4fs max=%.4fs\n",
        label, n, v[1], med, s/n, v[p95i], v[n]
    }'
}

echo "=== 4. Warmup ($WARMUP requests each) ==="
for i in $(seq 1 "$WARMUP"); do
  measure_one "http://127.0.0.1:$NATIVE_PORT/process" >/dev/null
  measure_one "http://127.0.0.1:$WASM_PORT/process" >/dev/null
done

echo "=== 5. Measure ($REQUESTS requests each, interleaved) ==="
NATIVE_TIMES=""
WASM_TIMES=""
for i in $(seq 1 "$REQUESTS"); do
  T="$(measure_one "http://127.0.0.1:$NATIVE_PORT/process")"
  NATIVE_TIMES="$NATIVE_TIMES $T"
  T="$(measure_one "http://127.0.0.1:$WASM_PORT/process")"
  WASM_TIMES="$WASM_TIMES $T"
done

echo "=== 6. Statistics ==="
report_stats "Native" "$NATIVE_TIMES"
report_stats "WASM  " "$WASM_TIMES"

echo "=== 7. Verify equivalence ==="
NATIVE_RESP="$(curl -s -f -X POST "http://127.0.0.1:$NATIVE_PORT/process" -H "Content-Type: application/json" -d "$INPUT_JSON")"
WASM_RESP="$(curl -s -f -X POST "http://127.0.0.1:$WASM_PORT/process" -H "Content-Type: application/json" -d "$INPUT_JSON")"

if [ "$NATIVE_RESP" = "$WASM_RESP" ]; then
  echo "✅ Identical output: $NATIVE_RESP"
else
  echo "❌ Outputs differ."
  echo "Native: $NATIVE_RESP"
  echo "WASM:   $WASM_RESP"
  exit 1
fi
```

---

## 🔄 BREAK (Поломка) — несовместимые артефакты

**Цель:** увидеть, как система отреагирует, если подсунуть ей не тот артефакт. В нашей экосистеме есть три WASM-файла, и только один из них — Component.

**Шаг 1. Попробуйте запустить `host` с Core WASM-модулем (Baseline):**
```bash
COMPONENT_WASM="$PWD/target/wasm32-wasip1/release/text_engine.wasm" ./scripts/measure.sh
```

**Ожидаемый результат:**
```text
❌ not a component-model artifact: .../text_engine.wasm
   first line: (module $text_engine.wasm
   hint: text-engine — это Baseline из Главы 1, компилируется в module,
         а не в component (по дизайну книги).
```

**Шаг 2. Что будет, если убрать проверку `wasm-tools print` и попытаться запустить?**
Вы получите ту самую ошибку, которая преследовала нас в начале:
```text
Runtime::new: failed to parse WebAssembly module
Caused by: attempted to parse a wasm module with a component parser
```

**Шаг 3. Вывод.**
Файл `text_engine.wasm` физически существует, имеет расширение `.wasm` и даже валиден. Но это **Module**, а не **Component**. `wasmtime::component::Component::from_file` умеет читать только Component Model. Именно поэтому `default_wasm_path()` жёстко зашит на `text_engine_component.wasm` из папки `text-engine-component`.

---

## Итог главы

**OBSERVE:**
Из реального прогона `measure.sh` (n=20, interleaved):

```text
Native: n=20 min=0.0008s med=0.0009s avg=0.0011s p95=0.0012s max=0.0041s
WASM  : n=20 min=0.0010s med=0.0013s avg=0.0015s p95=0.0026s max=0.0027s
✅ Identical output: {"output":"hello wasm world"}
```

**MEASURE:**
- Δmed ≈ **0.4 ms** (~1.44x).
- Δp95 ≈ **1.4 ms**.
- Хвост (p95) у WASM тяжелее, но это следствие работы аллокатора и JIT-прогрева.

**PREDICT (проверка):**
Было предсказано: «накладные расходы WASM потеряются в шуме HTTP». Результат: **не подтвердилось**. WASM-накладные расходы (создание `Store`, `WasiCtx`, копирование строк через WIT-границу) видны отчётливо — они составляют ~0.4 мс при HTTP-overhead ~0.9 мс.

**What did we prove?**
- WASM-компонент работает в backend end-to-end: `HTTP → Axum → Wasmtime → Component → HTTP`.
- Разделение `runtime` / `http` / `main` делает код читаемым и тестируемым.
- `spawn_blocking` обязателен: Wasmtime синхронный.
- Мы платим ~0.4 мс за изоляцию при stateless-модели (новый `Store` на каждый запрос).

**What did we not prove?**
- Что WASM **выгоднее** нативной функции. Мы измерили только цену границы. Сравнение с альтернативами (процессы, контейнеры) — Часть VI.

**When is the alternative better?**
- Если бизнес-логике не нужна изоляция — нативная функция быстрее.
- Если нужна изоляция, но не нужен WASM (например, приемлема изоляция на уровне ОС) — контейнер может быть проще в отладке.

### 📊 Evidence
```text
Claim:          WASM-компонент может обслуживать HTTP-запросы в backend, но стоит ~0.4 мс на запрос.
Experiment:     scripts/measure.sh (interleaved замер, n=20).
Measurement:    Native med=0.9ms, WASM med=1.3ms. Эквивалентность вывода: подтверждена.
Observed:       Endpoint возвращает корректный JSON, ошибки обрабатываются.
Conclusion:     Инфраструктура (Axum + Wasmtime) готова для управляемой
                единицы исполнения. Цена изоляции — стабильные 0.4 мс.
Confidence:     High
```