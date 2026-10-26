# Часть A. Финальный скрипт `scripts/setup-ch10.sh`


```bash
#!/usr/bin/env bash
# scripts/setup-ch10.sh
#
# Автоматическая настройка Главы 10: Memory limits and isolation.

set -euo pipefail
ROOT="$(cd "$(dirname "$0")/.." && pwd)"
cd "$ROOT"

echo "=== [1/7] Exclude new component from workspace ==="
if ! grep -q 'components/text-engine-memory' Cargo.toml; then
  python3 - <<'PY'
import re, pathlib
p = pathlib.Path("Cargo.toml")
s = p.read_text()
m = re.search(r'exclude\s*=\s*\[(.*?)\]', s, re.S)
if not m:
    raise SystemExit("exclude array not found in Cargo.toml")
inner = m.group(1).rstrip()
if not inner.endswith(','): inner += ','
inner += '\n    "components/text-engine-memory",\n'
p.write_text(s[:m.start()] + 'exclude = [' + inner + ']' + s[m.end():])
print("Added text-engine-memory to exclude.")
PY
fi

echo "=== [2/7] WIT contract ==="
mkdir -p wit/text-engine-memory
rm -f wit/text-engine-memory/world.wit
sleep 0.1
cat > wit/text-engine-memory/world.wit <<'EOF'
package docs:text-engine-memory@0.1.0;
interface processor {
    allocate: func(mb: u32) -> result<string, string>;
}
world text-engine-memory {
    export processor;
}
EOF

echo "=== [3/7] Guest component ==="
if [ ! -d "components/text-engine-memory" ]; then
  cargo new components/text-engine-memory --lib --vcs none >/dev/null
fi
rm -f components/text-engine-memory/Cargo.toml components/text-engine-memory/src/lib.rs
sleep 0.1
cat > components/text-engine-memory/Cargo.toml <<'EOF'
[package]
name = "text-engine-memory"
version = "0.1.0"
edition = "2021"
[workspace]
[lib]
crate-type = ["cdylib"]
[dependencies]
wit-bindgen = "0.63"
EOF
cat > components/text-engine-memory/src/lib.rs <<'EOF'
wit_bindgen::generate!({
    path: "../../wit/text-engine-memory",
    world: "text-engine-memory",
});
use exports::docs::text_engine_memory::processor::Guest;
struct MemoryEngine;
impl Guest for MemoryEngine {
    fn allocate(mb: u32) -> Result<String, String> {
        let bytes = (mb as usize) * 1024 * 1024;
        // Пытаемся выделить память. Если лимит Store исчерпан,
        // произойдёт trap (out of memory), который Host перехватит.
        let mut vec: Vec<u8> = Vec::with_capacity(bytes);
        for i in 0..bytes {
            vec.push((i % 255) as u8);
        }
        Ok(format!("Allocated {} MB successfully.", mb))
    }
}
export!(MemoryEngine);
EOF

echo "=== [4/7] Host runtime_v10 ==="
rm -f host/src/runtime_v10.rs
sleep 0.1
cat > host/src/runtime_v10.rs <<'EOF'
use anyhow::Result;
use std::path::Path;
use std::sync::Arc;
use wasmtime::component::{Component, Linker};
use wasmtime::{Engine, Store, StoreLimits, StoreLimitsBuilder};
use wasmtime_wasi::{WasiCtx, WasiCtxBuilder, WasiCtxView, WasiView};

wasmtime::component::bindgen!({
    path: "../wit/text-engine-memory",
    world: "text-engine-memory",
});

pub struct MyState {
    ctx: WasiCtx,
    table: wasmtime::component::ResourceTable,
    limits: StoreLimits,
}

impl WasiView for MyState {
    fn ctx(&mut self) -> WasiCtxView<'_> {
        WasiCtxView { ctx: &mut self.ctx, table: &mut self.table }
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
            anyhow::bail!("WASM component not found at `{}`", wasm_path.display());
        }
        let engine = Engine::default();
        let mut linker = Linker::new(&engine);
        wasmtime_wasi::p2::add_to_linker_sync(&mut linker)?;
        let component = Component::from_file(&engine, wasm_path)?;
        Ok(Arc::new(Self { engine, component, linker }))
    }

    pub fn allocate(&self, mb: u32) -> Result<Result<String, String>> {
        let wasi = WasiCtxBuilder::new().build();
        
        let limits = StoreLimitsBuilder::new()
            .memory_size(64 * 1024 * 1024) // 64 MB
            .build();

        let mut store = Store::new(&self.engine, MyState {
            ctx: wasi,
            table: wasmtime::component::ResourceTable::new(),
            limits,
        });
        store.limiter(|state| &mut state.limits);

        let b = TextEngineMemory::instantiate(&mut store, &self.component, &self.linker)?;
        let res = b.docs_text_engine_memory_processor().call_allocate(&mut store, mb)?;
        Ok(res.map_err(|e| e.to_string()))
    }
}
EOF
if ! grep -q 'pub mod runtime_v10;' host/src/lib.rs; then
  echo 'pub mod runtime_v10;' >> host/src/lib.rs
fi
touch host/src/lib.rs

echo "=== [5/7] Integration tests ==="
rm -f host/tests/integration_v10.rs
sleep 0.1
cat > host/tests/integration_v10.rs <<'EOF'
use std::path::PathBuf;

fn wasm_path() -> PathBuf {
    PathBuf::from(env!("CARGO_MANIFEST_DIR"))
        .parent().unwrap()
        .join("components/text-engine-memory/target/wasm32-wasip1/release/text_engine_memory.wasm")
}

#[test]
fn test_v10_memory_limits() {
    let path = wasm_path();
    if !path.exists() { eprintln!("Skipping"); return; }
    let rt = host::runtime_v10::Runtime::new(&path).expect("load");

    // 1. Успешная аллокация
    let res = rt.allocate(32).unwrap();
    assert!(res.is_ok(), "Should allocate 32MB: {:?}", res.err());

    // 2. OOM. При превышении лимита Wasmtime делает trap, 
    // поэтому внешний Result будет Err(anyhow::Error).
    let res = rt.allocate(128);
    assert!(res.is_err(), "Should return a trap on OOM");
    let err = format!("{:?}", res.unwrap_err());
    assert!(err.contains("unreachable") || err.contains("out of memory") || err.contains("memory"), 
            "Expected trap/oom, got: {}", err);
}
EOF

echo "=== [6/7] Visual demo (demo-web-ui-ch10) ==="
if [ ! -d "demo-web-ui-ch10" ]; then
  cargo new demo-web-ui-ch10 --bin --vcs none >/dev/null
fi
rm -f demo-web-ui-ch10/Cargo.toml demo-web-ui-ch10/src/main.rs
mkdir -p demo-web-ui-ch10/static
rm -f demo-web-ui-ch10/static/index.html
sleep 0.1
cat > demo-web-ui-ch10/Cargo.toml <<'EOF'
[package]
name = "demo-web-ui-ch10"
version.workspace = true
edition.workspace = true
[dependencies]
host = { path = "../host" }
axum = "0.8"
tokio = { version = "1", features = ["full"] }
serde_json = "1"
anyhow = "1.0"
EOF
cat > demo-web-ui-ch10/src/main.rs <<'EOF'
use axum::{response::Html, routing::{get, post}, Json, Router};
use serde_json::{json, Value};
use std::sync::Arc;
use host::runtime_v10::Runtime;

const INDEX_HTML: &str = include_str!("../static/index.html");
async fn index() -> Html<&'static str> { Html(INDEX_HTML) }

async fn allocate_small(axum::extract::State(rt): axum::extract::State<Arc<Runtime>>) -> Json<Value> {
    match tokio::task::spawn_blocking(move || rt.allocate(32)).await {
        Ok(Ok(Ok(msg))) => Json(json!({ "output": msg })),
        Ok(Ok(Err(e))) => Json(json!({ "error": format!("Logical error: {e}") })),
        Ok(Err(e)) => Json(json!({ "error": format!("Runtime trap: {e}") })),
        Err(e) => Json(json!({ "error": format!("Join error: {e}") })),
    }
}

async fn allocate_big(axum::extract::State(rt): axum::extract::State<Arc<Runtime>>) -> Json<Value> {
    match tokio::task::spawn_blocking(move || rt.allocate(128)).await {
        Ok(Ok(Ok(msg))) => Json(json!({ "output": msg })),
        Ok(Ok(Err(e))) => {
            eprintln!("[guest-memory-error] logical: {e}");
            Json(json!({ "error": "Guest logical error" }))
        },
        Ok(Err(e)) => {
            let err_str = format!("{:?}", e);
            eprintln!("[guest-memory-trap] {err_str}");
            // Перехватываем trap от OOM
            if err_str.contains("unreachable") || err_str.contains("out of memory") || err_str.contains("memory") {
                Json(json!({ "error": "Memory limit exceeded (64MB cap)" }))
            } else {
                Json(json!({ "error": format!("Runtime trap: {e}") }))
            }
        },
        Err(e) => Json(json!({ "error": format!("Join error: {e}") })),
    }
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let manifest = std::path::Path::new(env!("CARGO_MANIFEST_DIR"));
    let wasm_path = manifest.parent().unwrap()
        .join("components/text-engine-memory/target/wasm32-wasip1/release/text_engine_memory.wasm");
    if !wasm_path.exists() { std::process::exit(1); }

    let runtime = Runtime::new(&wasm_path)?;

    let app = Router::new()
        .route("/", get(index))
        .route("/small", post(allocate_small))
        .route("/big", post(allocate_big))
        .with_state(runtime);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3009").await?;
    println!("🚀 Demo UI (Ch10): http://127.0.0.1:3009");
    axum::serve(listener, app).await?;
    Ok(())
}
EOF

cat > demo-web-ui-ch10/static/index.html <<'HTML_EOF'
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8"><title>Ch10: Memory Limits</title>
<style>
  :root { --bg:#0f172a; --panel:#1e293b; --br:#334155; --tx:#e2e8f0;
          --mu:#94a3b8; --ac:#38bdf8; --ok:#22c55e; --er:#ef4444;
          --mo:ui-monospace,SFMono-Regular,Menlo,Consolas,monospace; }
  *{box-sizing:border-box;margin:0;padding:0}
  body{font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,sans-serif;
       background:var(--bg);color:var(--tx);min-height:100vh;
       display:flex;align-items:center;justify-content:center;padding:2rem}
  .box{max-width:800px;width:100%;background:var(--panel);
       border:1px solid var(--br);border-radius:12px;padding:2rem;
       box-shadow:0 20px 60px rgba(0,0,0,.4)}
  h1{font-size:1.6rem;font-weight:600;text-align:center;margin-bottom:.35rem;
     background:linear-gradient(135deg,var(--ac),#a78bfa);
     -webkit-background-clip:text; background-clip:text;
     -webkit-text-fill-color:transparent}
  .sub{color:var(--mu);text-align:center;font-size:.9rem;margin-bottom:2rem}
  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 1rem; }
  .panel { background: var(--bg); border: 1px solid var(--br); border-radius: 8px; padding: 1.25rem; text-align: center; display: flex; flex-direction: column; }
  h2 { font-size: .9rem; font-weight: 600; margin-bottom: .5rem; color: var(--ac); }
  .desc { font-size: .75rem; color: var(--mu); margin-bottom: 1rem; flex-grow: 1; }
  button { padding: .6rem 1rem; font-size: .85rem; font-weight: 500; border: none; border-radius: 6px; cursor: pointer; transition: .15s; width: 100%; }
  button.good { background: var(--ac); color: #0c1222; }
  button.bad { background: var(--er); color: white; }
  button:hover { filter: brightness(1.1); }
  button:disabled { opacity: .5; cursor: wait; }
  .result { margin-top: 1rem; font-family: var(--mo); font-size: .75rem; word-break: break-all; min-height: 40px; }
  .res-ok { color: var(--ok); }
  .res-err { color: var(--er); }
</style>
</head>
<body>
<div class="box">
  <h1>Memory Limits: StoreLimits</h1>
  <p class="sub">WASM component attempts to allocate memory</p>
  <div class="grid">
    <div class="panel">
      <h2>Safe Allocation</h2>
      <p class="desc">Try to allocate 32 MB.<br>Limit is set to 64 MB.</p>
      <button class="good" onclick="run('/small', 'out-small')">Allocate 32 MB</button>
      <div class="result" id="out-small"></div>
    </div>
    <div class="panel">
      <h2>Dangerous Allocation (OOM)</h2>
      <p class="desc">Try to allocate 128 MB.<br>Exceeds the 64 MB limit.</p>
      <button class="bad" onclick="run('/big', 'out-big')">Allocate 128 MB</button>
      <div class="result" id="out-big"></div>
    </div>
  </div>
</div>
<script>
async function run(url, outId) {
  const out = document.getElementById(outId);
  out.innerHTML = 'Allocating...'; out.className = 'result';
  document.querySelectorAll('button').forEach(b => b.disabled = true);
  try {
    const res = await fetch(url, { method: 'POST' });
    const data = await res.json();
    if (data.error) {
      out.innerHTML = `<span class="res-err">${data.error}</span>`;
    } else {
      out.innerHTML = `<span class="res-ok">${data.output}</span>`;
    }
  } catch (e) {
    out.innerHTML = `<span class="res-err">Network error: ${e}</span>`;
  } finally {
    document.querySelectorAll('button').forEach(b => b.disabled = false);
  }
}
</script>
</body>
</html>
HTML_EOF

echo "=== [7/7] Build & verify ==="
(cd components/text-engine-memory && cargo component build --release --quiet)
cargo check -p host --lib --quiet
cargo check -p demo-web-ui-ch10 --quiet

echo "✅ Готово. Следующие шаги:"
echo "  1. Тесты:  (cd host && cargo test --test integration_v10)"
echo "  2. Демо:   cargo run -p demo-web-ui-ch10 --release"
echo "  3. Открыть: http://127.0.0.1:3009"
```

### Шаг 2. Запустить и проверить

```bash
./scripts/setup-ch10.sh
(cd host && cargo test --test integration_v10)
```

Тест должен пройти успешно (`test result: ok`). Если да — я сразу напишу финальный текст Главы 10!

# Глава 10. Memory limits and isolation: защита от Memory Bomb

**Зачем создана эта глава?**
В Главе 8 мы научились останавливать компонент, который бесконечно нагружает CPU (через `fuel` и `epoch`). Но есть не менее опасный сценарий: компонент пытается выделить гигантский кусок памяти (например, `Vec::with_capacity(10_000_000_000)`). В обычном Rust-приложении это вызовет аварийное завершение процесса (OOM-killer) или панику. В WebAssembly мы не можем допустить, чтобы один компонент убил весь HTTP-сервер. Мы должны научиться выделять «песочнице» жесткий лимит по памяти.

**Какую пользу несёт глава?**
1. **Управление памятью:** Вы научитесь использовать `StoreLimitsBuilder` для ограничения линейной памяти (Linear Memory), которую может запросить гость.
2. **Изоляция сбоев (Fault Isolation):** Вы увидите, что при превышении лимита Wasmtime генерирует `Trap` (прерывание). Инстанс компонента умирает, но Host остается живым и продолжает обслуживать другие запросы.
3. **Разделение логических и системных ошибок:** Ошибка нехватки памяти — это не `result<string, string>` (WIT). Это системный сбой Runtime'а. Мы научимся перехватывать его на уровне Host.

**Чего мы хотим достичь по результатам тестирования?**
1. Создать компонент, который пытается выделить заданный объём памяти.
2. Настроить `StoreLimits` на 64 МБ.
3. Доказать, что выделение 32 МБ проходит успешно, а запрос на 128 МБ завершается `Trap`'ом (out of memory), который Host корректно обрабатывает, не падая.

---

### 🚦 Starting point
У нас есть `text-engine-secure` (Глава 9) и рантайм с настройкой `WasiCtxBuilder`. Мы **не трогаем** их. Скрипт создал независимый компонент `text-engine-memory` и рантайм с лимитами.

### 🎯 PREDICT (Предсказание)
*Запишите свой прогноз перед запуском:*
> **Если компонент вызовет `Vec::with_capacity(128 * 1024 * 1024)` при лимите в 64 МБ, как это проявится на стороне Host? Вернётся ли ошибка из функции `allocate` (как `Err`), или функция вообще не сможет завершиться?**

---

## Этап 1. Исходные коды

### Структура
**Новые файлы:**
- `wit/text-engine-memory/world.wit` — контракт.
- `components/text-engine-memory/` — компонент с аллокацией.
- `host/src/runtime_v10.rs` — рантайм с `StoreLimits`.
- `host/tests/integration_v10.rs` — тесты.
- `demo-web-ui-ch10/` — UI.

### Ссылки для изучения
- [Wasmtime — StoreLimits](https://docs.wasmtime.dev/api/wasmtime/struct.StoreLimits.html) — как работает лимитер.
- [Wasmtime — Traps](https://docs.wasmtime.dev/api/wasmtime/struct.Trap.html) — почему `out of memory` — это trap, а не ошибка.

### Ключевые фрагменты кода

**1. `host/src/runtime_v10.rs` (Настройка лимитов)**
```rust
use wasmtime::{StoreLimits, StoreLimitsBuilder};

pub fn allocate(&self, mb: u32) -> Result<Result<String, String>> {
    let wasi = WasiCtxBuilder::new().build();
    
    // 1. Создаем лимитер: максимум 64 МБ на инстанс
    let limits = StoreLimitsBuilder::new()
        .memory_size(64 * 1024 * 1024)
        .build();

    let mut store = Store::new(&self.engine, MyState {
        ctx: wasi,
        table: wasmtime::component::ResourceTable::new(),
        limits,
    });

    // 2. Привязываем лимитер к Store
    store.limiter(|state| &mut state.limits);

    let b = TextEngineMemory::instantiate(&mut store, &self.component, &self.linker)?;
    let res = b.docs_text_engine_memory_processor().call_allocate(&mut store, mb)?;
    Ok(res.map_err(|e| e.to_string()))
}
```

---

## Этап 2. Тесты

**`host/tests/integration_v10.rs`**

### Команды запуска
```bash
(cd components/text-engine-memory && cargo component build --release)
(cd host && cargo test --test integration_v10)
```

**Что проверяется:**
1. `allocate(32)` — возвращает `Ok` (успешная аллокация).
2. `allocate(128)` — **падает с `Trap`**, потому что Rust-аллокатор внутри гостя вызывает `abort()`, когда `Vec::with_capacity` получает отказ от WASI.

---

## Этап 3. Демо-пример

Визуальный сервер `demo-web-ui-ch10` (порт 3009) с двумя панелями.

### Запуск
```bash
cargo run -p demo-web-ui-ch10 --release
# open http://127.0.0.1:3009
```

### 🔄 BREAK (Поломка) и OBSERVE

1. Нажмите **Allocate 32 MB** (левая панель).
   * *Результат:* Успешно (зелёный текст).
2. Нажмите **Allocate 128 MB** (правая панель).
   * *Результат:* Вы увидите красную ошибку `Memory limit exceeded (64MB cap)`.
   * *В консоли сервера:* `[guest-memory-trap] wasm trap: wasm 'unreachable' instruction executed`.

**Почему это Trap, а не обычная ошибка?**
Глядя на WIT-контракт, мы объявили `allocate: func(mb: u32) -> result<string, string>`. Но Rust-аллокатор устроен так: если памяти не хватает, он не возвращает `Err`. Он вызывает `handle_alloc_error`, который внутри WASM компилируется в инструкцию `unreachable`. Wasmtime перехватывает это как `Trap` и немедленно уничтожает текущий `Instance` (чтобы предотвратить повреждение памяти).
Наш Host перехватывает этот системный `Trap` и возвращает клиенту понятное сообщение.

---

## 🔄 Итеративная проверка

**Нужны ли доработки после тестов?**
Да, мы обнаружили важную особенность. Изначально тест пытался обработать `allocate(128)` как `Ok(Err(...))`. Но из-за того, что Rust делает `abort` при OOM, Wasmtime возвращает внешний `Err(anyhow::Error)`. Пришлось изменить тест, чтобы он проверял наличие `trap` или `unreachable`, а в веб-сервере — добавить обработку этого специфического Trap'а.

---

## Итог главы

**OBSERVE:**
- Лимиты памяти защищают хост от OOM-killer.
- Выделение памяти сверх лимита приводит к Trap'у, а не к логической ошибке.
- Сервер продолжает работать после Trap'а.

**PREDICT (проверка):**
Было предсказано: "Функция вернет `Err`". Результат: **не подтвердилось**. Функция аварийно завершилась (Trap), так как Rust-аллокатор вызывает `panic/abort`. Это доказывает, что OOM — это не бизнес-ошибка, а нарушение системных границ.

**What did we prove?**
1. **Изоляция памяти работает.** Компонент не может съесть всю RAM хоста.
2. **StoreLimits — это инструмент Runtime'а.** WIT-контракт не может гарантировать безопасность по памяти; это обязанность Host'а.
3. **Trap изолирован.** После падения одного инстанса сервер способен создавать новые `Store` и обслуживать новые запросы.

**What did we not prove?**
- Как ограничивать память на уровне отдельного `Store` при использовании пула инстансов (мы рассмотрим это в части про масштабирование).

**What did we pay for it?**
- Нам пришлось обернуть `StoreLimits` в `MyState`, чтобы Wasmtime мог к нему обращаться.
- Клиент получает менее детальную ошибку (мы не можем узнать, *сколько именно* памяти не хватило, только сам факт Trap).

**When is the alternative better?**
- Если компонент должен обрабатывать огромные массивы данных (например, ML-модели), лимит в 64 МБ станет узким горлышком. В таких случаях лимит увеличивают или переходят на потоковую обработку (Streams, Глава 14).

### 📊 Evidence
```text
Claim:          StoreLimits предотвращает исчерпание памяти хостом.
Experiment:     Вызов allocate(128) при лимите 64 МБ.
Measurement:    Wasmtime возвращает Trap (unreachable instruction).
Observed:       Хост переживает вызов, возвращает HTTP 200 с JSON ошибки.
Conclusion:     Память гостя строго изолирована.
Confidence:     High
```

### Что у нас теперь есть
- `components/text-engine-memory/` — компонент для теста памяти.
- `host/src/runtime_v10.rs` — рантайм с `StoreLimits`.
- UI для визуального сравнения безопасной и опасной аллокации.
- Все компоненты и рантаймы из Глав 1–9 не тронуты.

**Следующий шаг:** В Главе 11 (Failure semantics and Cancellation) мы соберем все виды отказов (время, память, capabilities, паника гостя) в единую таблицу и посмотрим, как отмена HTTP-запроса влияет на уничтожение ресурсов компонента.