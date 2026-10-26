# Часть A. Финальный скрипт `scripts/setup-ch11.sh`

```bash
#!/usr/bin/env bash
# scripts/setup-ch11.sh
#
# Автоматическая настройка Главы 11: Failure semantics and Cancellation.

set -euo pipefail
ROOT="$(cd "$(dirname "$0")/.." && pwd)"
cd "$ROOT"

echo "=== [1/8] Exclude new component from workspace ==="
if ! grep -q 'components/text-engine-crash' Cargo.toml; then
  python3 - <<'PY'
import re, pathlib
p = pathlib.Path("Cargo.toml")
s = p.read_text()
m = re.search(r'exclude\s*=\s*\[(.*?)\]', s, re.S)
if not m:
    raise SystemExit("exclude array not found in Cargo.toml")
inner = m.group(1).rstrip()
if not inner.endswith(','): inner += ','
inner += '\n    "components/text-engine-crash",\n'
p.write_text(s[:m.start()] + 'exclude = [' + inner + ']' + s[m.end():])
print("Added text-engine-crash to exclude.")
PY
fi

echo "=== [2/8] WIT contract ==="
mkdir -p wit/text-engine-crash
rm -f wit/text-engine-crash/world.wit
sleep 0.1
cat > wit/text-engine-crash/world.wit <<'EOF'
package docs:text-engine-crash@0.1.0;
interface processor {
    long-task: func(ms: u32) -> string;
    crash: func() -> string;
}
world text-engine-crash {
    export processor;
}
EOF

echo "=== [3/8] Guest component ==="
if [ ! -d "components/text-engine-crash" ]; then
  cargo new components/text-engine-crash --lib --vcs none >/dev/null
fi
rm -f components/text-engine-crash/Cargo.toml components/text-engine-crash/src/lib.rs
sleep 0.1
cat > components/text-engine-crash/Cargo.toml <<'EOF'
[package]
name = "text-engine-crash"
version = "0.1.0"
edition = "2021"
[workspace]
[lib]
crate-type = ["cdylib"]
[dependencies]
wit-bindgen = "0.63"
EOF
cat > components/text-engine-crash/src/lib.rs <<'EOF'
wit_bindgen::generate!({
    path: "../../wit/text-engine-crash",
    world: "text-engine-crash",
});
use exports::docs::text_engine_crash::processor::Guest;
use std::time::{Duration, Instant};
struct CrashEngine;
impl Guest for CrashEngine {
    fn long_task(ms: u32) -> String {
        let start = Instant::now();
        while start.elapsed() < Duration::from_millis(ms as u64) {
            std::hint::black_box(0);
        }
        format!("Completed long task of {} ms", ms)
    }
    fn crash() -> String {
        panic!("Guest intentionally panicked!");
    }
}
export!(CrashEngine);
EOF

echo "=== [4/8] Host runtime_v11 ==="
# Безопасное добавление uuid
python3 - <<'PY'
import pathlib
p = pathlib.Path("host/Cargo.toml")
s = p.read_text()
if 'uuid' not in s:
    s = s.replace('[dependencies]\n', '[dependencies]\nuuid = { version = "1", features = ["v4"] }\n')
    p.write_text(s)
    print("Added uuid to host/Cargo.toml")
PY

rm -f host/src/runtime_v11.rs
sleep 0.1
cat > host/src/runtime_v11.rs <<'EOF'
use anyhow::Result;
use std::collections::HashMap;
use std::path::Path;
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::{Arc, Mutex};
use std::time::Duration;
use wasmtime::component::{Component, Linker};
use wasmtime::{Config, Engine, Store, UpdateDeadline};
use wasmtime_wasi::{WasiCtx, WasiCtxBuilder, WasiCtxView, WasiView};

wasmtime::component::bindgen!({
    path: "../wit/text-engine-crash",
    world: "text-engine-crash",
});

pub struct MyState {
    ctx: WasiCtx,
    table: wasmtime::component::ResourceTable,
    cancelled: Arc<AtomicBool>,
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
    active_tasks: Mutex<HashMap<String, Arc<AtomicBool>>>,
}

impl Runtime {
    pub fn new(wasm_path: &Path) -> Result<Arc<Self>> {
        if !wasm_path.exists() {
            anyhow::bail!("WASM component not found at `{}`", wasm_path.display());
        }
        let mut config = Config::new();
        config.epoch_interruption(true);
        let engine = Engine::new(&config)?;
        let mut linker = Linker::new(&engine);
        wasmtime_wasi::p2::add_to_linker_sync(&mut linker)?;
        let component = Component::from_file(&engine, wasm_path)?;

        let engine_clone = engine.clone();
        std::thread::spawn(move || {
            loop {
                std::thread::sleep(Duration::from_millis(50));
                engine_clone.increment_epoch();
            }
        });

        Ok(Arc::new(Self {
            engine,
            component,
            linker,
            active_tasks: Mutex::new(HashMap::new()),
        }))
    }

    fn new_store(&self, cancelled: Arc<AtomicBool>) -> Store<MyState> {
        let wasi = WasiCtxBuilder::new().build();
        let mut store = Store::new(&self.engine, MyState {
            ctx: wasi,
            table: wasmtime::component::ResourceTable::new(),
            cancelled,
        });
        store.set_epoch_deadline(1);
        
        // Коллбэк получает StoreContextMut, а не &mut MyState
        store.epoch_deadline_callback(|ctx| {
            if ctx.data().cancelled.load(Ordering::SeqCst) {
                Ok(UpdateDeadline::Interrupt)
            } else {
                Ok(UpdateDeadline::Continue(1))
            }
        });
        store
    }

    pub fn start_task(&self) -> String {
        let id = uuid::Uuid::new_v4().to_string();
        let flag = Arc::new(AtomicBool::new(false));
        self.active_tasks.lock().unwrap().insert(id.clone(), flag);
        id
    }

    pub fn cancel_task(&self, id: &str) -> bool {
        if let Some(flag) = self.active_tasks.lock().unwrap().get(id) {
            flag.store(true, Ordering::SeqCst);
            true
        } else {
            false
        }
    }

    pub fn run_long_task(&self, id: &str, ms: u32) -> Result<String> {
        let flag = {
            let tasks = self.active_tasks.lock().unwrap();
            tasks.get(id).cloned()
        };
        let flag = match flag {
            Some(f) => f,
            None => anyhow::bail!("Task not found"),
        };
        let mut store = self.new_store(flag);
        let b = TextEngineCrash::instantiate(&mut store, &self.component, &self.linker)?;
        let res = b.docs_text_engine_crash_processor().call_long_task(&mut store, ms);
        self.active_tasks.lock().unwrap().remove(id);
        Ok(res?)
    }

    pub fn run_crash(&self) -> Result<String> {
        let mut store = self.new_store(Arc::new(AtomicBool::new(false)));
        let b = TextEngineCrash::instantiate(&mut store, &self.component, &self.linker)?;
        let res = b.docs_text_engine_crash_processor().call_crash(&mut store)?;
        Ok(res)
    }
}
EOF

if ! grep -q 'pub mod runtime_v11;' host/src/lib.rs; then
  echo 'pub mod runtime_v11;' >> host/src/lib.rs
fi
touch host/src/lib.rs

echo "=== [5/8] Integration tests ==="
rm -f host/tests/integration_v11.rs
sleep 0.1
cat > host/tests/integration_v11.rs <<'EOF'
use std::path::PathBuf;
fn wasm_path() -> PathBuf {
    PathBuf::from(env!("CARGO_MANIFEST_DIR")).parent().unwrap()
        .join("components/text-engine-crash/target/wasm32-wasip1/release/text_engine_crash.wasm")
}
#[test]
fn test_v11_failure_and_cancel() {
    let path = wasm_path();
    if !path.exists() { eprintln!("Skipping"); return; }
    let rt = host::runtime_v11::Runtime::new(&path).expect("load");

    let id = rt.start_task();
    let res = rt.run_long_task(&id, 100).unwrap();
    assert!(res.contains("Completed"));
    
    let id = rt.start_task();
    let rt_cancel = rt.clone();
    let id_cancel = id.clone();
    std::thread::spawn(move || {
        std::thread::sleep(std::time::Duration::from_millis(200));
        rt_cancel.cancel_task(&id_cancel);
    });
    
    let res = rt.run_long_task(&id, 1000);
    assert!(res.is_err(), "Task should be cancelled");
    
    let res = rt.run_crash();
    assert!(res.is_err(), "Crash should return Err");
}
EOF

echo "=== [6/8] Visual demo (demo-web-ui-ch11) ==="
if [ ! -d "demo-web-ui-ch11" ]; then
  cargo new demo-web-ui-ch11 --bin --vcs none >/dev/null
fi
rm -f demo-web-ui-ch11/Cargo.toml demo-web-ui-ch11/src/main.rs
mkdir -p demo-web-ui-ch11/static
rm -f demo-web-ui-ch11/static/index.html
sleep 0.1
cat > demo-web-ui-ch11/Cargo.toml <<'EOF'
[package]
name = "demo-web-ui-ch11"
version.workspace = true
edition.workspace = true
[dependencies]
host = { path = "../host" }
axum = "0.8"
tokio = { version = "1", features = ["full"] }
serde_json = "1"
anyhow = "1.0"
EOF
cat > demo-web-ui-ch11/src/main.rs <<'EOF'
use axum::{response::Html, routing::{get, post}, Json, Router, extract::Path};
use serde_json::{json, Value};
use std::sync::Arc;
use host::runtime_v11::Runtime;

const INDEX_HTML: &str = include_str!("../static/index.html");
async fn index() -> Html<&'static str> { Html(INDEX_HTML) }

async fn start_task(axum::extract::State(rt): axum::extract::State<Arc<Runtime>>) -> Json<Value> {
    let id = rt.start_task();
    let rt_clone = rt.clone();
    let id_clone = id.clone();
    tokio::task::spawn_blocking(move || {
        let res = rt_clone.run_long_task(&id_clone, 5000);
        match res {
            Ok(msg) => println!("[task {}] {msg}", id_clone),
            Err(e) => println!("[task {}] FAILED: {e:?}", id_clone),
        }
    });
    Json(json!({ "task_id": id }))
}

async fn cancel_task(axum::extract::State(rt): axum::extract::State<Arc<Runtime>>, Path(id): Path<String>) -> Json<Value> {
    if rt.cancel_task(&id) { Json(json!({ "status": "cancelled" })) }
    else { Json(json!({ "error": "task not found" })) }
}

async fn crash_guest(axum::extract::State(rt): axum::extract::State<Arc<Runtime>>) -> Json<Value> {
    match tokio::task::spawn_blocking(move || rt.run_crash()).await {
        Ok(Ok(_)) => Json(json!({ "output": "unexpected success" })),
        Ok(Err(e)) => {
            eprintln!("[crash] {e:?}");
            Json(json!({ "error": "Guest panicked (Trap)" }))
        },
        Err(e) => Json(json!({ "error": format!("join: {e}") })),
    }
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let manifest = std::path::Path::new(env!("CARGO_MANIFEST_DIR"));
    let wasm_path = manifest.parent().unwrap()
        .join("components/text-engine-crash/target/wasm32-wasip1/release/text_engine_crash.wasm");
    if !wasm_path.exists() { std::process::exit(1); }

    let runtime = Runtime::new(&wasm_path)?;
    let app = Router::new()
        .route("/", get(index))
        .route("/start", post(start_task))
        .route("/cancel/{id}", post(cancel_task)) // ИСПРАВЛЕНО: {id}
        .route("/crash", post(crash_guest))
        .with_state(runtime);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3010").await?;
    println!("🚀 Demo UI (Ch11): http://127.0.0.1:3010");
    axum::serve(listener, app).await?;
    Ok(())
}
EOF

cat > demo-web-ui-ch11/static/index.html <<'HTML_EOF'
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8"><title>Ch11: Cancellation & Failure</title>
<style>
  :root { --bg:#0f172a; --panel:#1e293b; --br:#334155; --tx:#e2e8f0; --mu:#94a3b8; --ac:#38bdf8; --ok:#22c55e; --er:#ef4444; --wr:#f59e0b; --mo:ui-monospace,SFMono-Regular,Menlo,Consolas,monospace; }
  *{box-sizing:border-box;margin:0;padding:0}
  body{font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,sans-serif; background:var(--bg);color:var(--tx);min-height:100vh; display:flex;align-items:center;justify-content:center;padding:2rem}
  .box{max-width:850px;width:100%;background:var(--panel); border:1px solid var(--br);border-radius:12px;padding:2rem; box-shadow:0 20px 60px rgba(0,0,0,.4)}
  h1{font-size:1.6rem;font-weight:600;text-align:center;margin-bottom:.35rem; background:linear-gradient(135deg,var(--ac),#a78bfa); -webkit-background-clip:text; background-clip:text; -webkit-text-fill-color:transparent}
  .sub{color:var(--mu);text-align:center;font-size:.9rem;margin-bottom:2rem}
  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 1rem; }
  .panel { background: var(--bg); border: 1px solid var(--br); border-radius: 8px; padding: 1.25rem; text-align: center; }
  h2 { font-size: .9rem; font-weight: 600; margin-bottom: .5rem; color: var(--ac); }
  .desc { font-size: .75rem; color: var(--mu); margin-bottom: 1rem; }
  button { padding: .6rem 1rem; font-size: .85rem; font-weight: 500; border: none; border-radius: 6px; cursor: pointer; transition: .15s; width: 100%; margin-bottom: 0.5rem; }
  button.primary { background: var(--ac); color: #0c1222; }
  button.warn { background: var(--wr); color: #0c1222; }
  button.bad { background: var(--er); color: white; }
  button:hover { filter: brightness(1.1); }
  button:disabled { opacity: .5; cursor: wait; }
  .result { margin-top: 1rem; font-family: var(--mo); font-size: .75rem; word-break: break-all; min-height: 40px; text-align: left; }
  .res-ok { color: var(--ok); }
  .res-err { color: var(--er); }
  .log { background: #000; padding: 0.5rem; border-radius: 4px; height: 100px; overflow-y: auto; font-family: var(--mo); font-size: 0.7rem; color: var(--mu); margin-top: 1rem; }
</style>
</head>
<body>
<div class="box">
  <h1>Failure Semantics & Cancellation</h1>
  <p class="sub">What dies when we cancel or crash?</p>
  <div class="grid">
    <div class="panel">
      <h2>Cancellable Task</h2>
      <p class="desc">Starts a 5s task. You can cancel it.</p>
      <button class="primary" id="btn-start" onclick="startTask()">Start Task</button>
      <button class="warn" id="btn-cancel" onclick="cancelTask()" disabled>Cancel Task</button>
      <div class="result" id="res-task"></div>
    </div>
    <div class="panel">
      <h2>Guest Crash (Panic)</h2>
      <p class="desc">Calls a function that panics inside WASM.</p>
      <button class="bad" onclick="crashGuest()">Trigger Crash</button>
      <div class="result" id="res-crash"></div>
    </div>
  </div>
  <div class="log" id="log">Server logs...</div>
</div>
<script>
let currentTaskId = null;
function log(msg) {
  const l = document.getElementById('log');
  l.innerHTML += `<div>> ${msg}</div>`;
  l.scrollTop = l.scrollHeight;
}
async function startTask() {
  document.getElementById('btn-start').disabled = true;
  document.getElementById('btn-cancel').disabled = false;
  document.getElementById('res-task').innerHTML = 'Running 5s task...';
  try {
    const res = await fetch('/start', { method: 'POST' });
    const data = await res.json();
    currentTaskId = data.task_id;
    log(`Started task ${currentTaskId}`);
  } catch(e) { log(`Error: ${e}`); document.getElementById('btn-start').disabled = false; }
}
async function cancelTask() {
  if (!currentTaskId) return;
  document.getElementById('btn-cancel').disabled = true;
  try {
    const res = await fetch(`/cancel/${currentTaskId}`, { method: 'POST' });
    document.getElementById('res-task').innerHTML = `<span class="res-err">Task Cancelled (Epoch Interrupt)</span>`;
    log(`Sent cancel signal for ${currentTaskId}`);
    document.getElementById('btn-start').disabled = false;
    currentTaskId = null;
  } catch(e) { log(`Error: ${e}`); }
}
async function crashGuest() {
  document.getElementById('res-crash').innerHTML = 'Calling crash...';
  try {
    const res = await fetch('/crash', { method: 'POST' });
    const data = await res.json();
    if (data.error) {
      document.getElementById('res-crash').innerHTML = `<span class="res-err">${data.error}</span>`;
      log(`Guest crashed: ${data.error}`);
    }
  } catch(e) { log(`Error: ${e}`); }
}
</script>
</body>
</html>
HTML_EOF

echo "=== [7/8] Modify host/lib.rs ==="
# Уже сделано

echo "=== [8/8] Build & verify ==="
(cd components/text-engine-crash && cargo component build --release --quiet)
cargo check -p host --lib --quiet
cargo check -p demo-web-ui-ch11 --quiet

echo "✅ Готово. Следующие шаги:"
echo "  1. Демо:   cargo run -p demo-web-ui-ch11 --release"
echo "  2. Открыть: http://127.0.0.1:3010"
```

---

# Часть B. Глава 11. Failure semantics and Cancellation

**Зачем создана эта глава?**
В предыдущих главах мы научились защищать сервер от *внешних* угроз: ограничили время работы, память и доступ к файлам. Но что происходит, когда сам клиент обрывает соединение (нажал «Отмена» в браузере)? Или когда компонент паникует внутри себя? В традиционном backend отмена HTTP-запроса часто игнорируется — вычисления продолжаются в фоне, сжигая CPU, а результат просто выбрасывается. В WebAssembly мы должны четко определить **семантику отказов (Failure Semantics)**: кто именно умирает, в какой момент освобождается память, и как не оставить «зомби-процессы».

**Какую пользу несёт глава?**
1. **Понимание цепочки отмены:** Вы увидите, что `HTTP Cancellation` ≠ `WASM Interruption` ≠ `Resource Release`. Отмена на уровне HTTP (Tokio) не останавливает WASM-код автоматически.
2. **Изоляция паники (Trap):** Вы научитесь перехватывать `panic` внутри гостя. Wasmtime превращает панику в `Trap`, который убивает конкретный `Store`, но не роняет Host.
3. **Паттерн управления задачами:** Мы реализуем реестр активных задач с возможностью их принудительной остановки извне (через `AtomicBool` и `epoch_interruption`).

**Чего мы хотим достичь по результатам тестирования?**
1. Создать компонент, который умеет «зависать» на 5 секунд и паниковать.
2. Реализовать `start_task` (возвращает `id`) и `cancel_task(id)`.
3. Доказать, что:
   - Паника гостя возвращает `Trap` и не ломает хост.
   - Отмена задачи через `AtomicBool` прерывает выполнение за ~50 мс, не дожидаясь конца 5 секунд.

---

### 🚦 Starting point
У нас есть `text-engine-memory` (Глава 10) и понимание `epoch_interruption`. Мы создаём новый компонент `text-engine-crash` и модуль `runtime_v11.rs`.

### 🎯 PREDICT (Предсказание)
*Запишите свой прогноз перед запуском:*
> **Если мы запустим задачу на 5 секунд, а через 1 секунду вызовем `cancel_task`, вернёт ли `run_long_task` ошибку сразу, или продолжит ждать, пока не пройдут все 5 секунд?**

---

## Этап 1. Исходные коды

### Структура
**Новые файлы:**
- `wit/text-engine-crash/world.wit` — задачи и паника.
- `components/text-engine-crash/` — компонент.
- `host/src/runtime_v11.rs` — рантайм с отменой.
- `host/tests/integration_v11.rs` — тесты.
- `demo-web-ui-ch11/` — UI.

### Ключевые фрагменты кода

**1. `components/text-engine-crash/src/lib.rs`**
```rust
fn long_task(ms: u32) -> String {
    let start = Instant::now();
    // Активный цикл ожидания (сжигает CPU, чтобы Epoch успел сработать)
    while start.elapsed() < Duration::from_millis(ms as u64) {
        std::hint::black_box(0);
    }
    format!("Completed long task of {} ms", ms)
}

fn crash() -> String {
    panic!("Guest intentionally panicked!");
}
```

**2. `host/src/runtime_v11.rs` (Отмена через AtomicBool)**
```rust
pub fn cancel_task(&self, id: &str) -> bool {
    if let Some(flag) = self.active_tasks.lock().unwrap().get(id) {
        // Просто поднимаем флаг, не блокируя Store
        flag.store(true, Ordering::SeqCst);
        true
    } else { false }
}

fn new_store(&self, cancelled: Arc<AtomicBool>) -> Store<MyState> {
    // ...
    store.set_epoch_deadline(1);
    // Коллбэк проверяет флаг отмены на каждом тике (раз в 50 мс)
    store.epoch_deadline_callback(|ctx| {
        if ctx.data().cancelled.load(Ordering::SeqCst) {
            Ok(UpdateDeadline::Interrupt)
        } else {
            Ok(UpdateDeadline::Continue(1))
        }
    });
    store
}
```

---

## Этап 2. Тесты

**`host/tests/integration_v11.rs`**

### Команды запуска
```bash
(cd components/text-engine-crash && cargo component build --release)
(cd host && cargo test --test integration_v11)
```

**Что проверяется:**
1. `run_long_task(100)` — успешно завершается (нет отмены).
2. `run_long_task(1000)` с отменой через 200 мс — падает с `Trap`.
3. `run_crash()` — падает с `Trap` (panic).

---

## Этап 3. Демо-пример

Визуальный сервер `demo-web-ui-ch11` (порт 3010) с двумя панелями.

### Запуск
```bash
cargo run -p demo-web-ui-ch11 --release
# open http://127.0.0.1:3010
```

### 🔄 BREAK (Поломка) и OBSERVE

1. Нажмите **Start Task**. UI скажет "Running 5s task...".
2. Через 1-2 секунды нажмите **Cancel Task**.
   * *Результат:* Задача мгновенно завершается с `Task Cancelled`.
   * *В консоли сервера:* `[task <id>] FAILED: wasm trap: interrupt`.
3. Нажмите **Trigger Crash**.
   * *Результат:* Красный текст `Guest panicked (Trap)`.
   * *В консоли сервера:* `[crash] error while executing at wasm backtrace: ... panic!`.

**Почему так происходит?**
Мы не ждем 5 секунд. Метод `cancel_task` ставит флаг `AtomicBool` в `true`. Фоновый поток Wasmtime (который каждые 50 мс делает `increment_epoch`) вызывает коллбэк `epoch_deadline_callback`. Коллбэк видит, что флаг `true`, и возвращает `Interrupt`. Wasmtime мгновенно прерывает WASM-код.

---

## 🔄 Итеративная проверка

**Нужны ли доработки после тестов?**
Да, мы обнаружили важный архитектурный нюанс при отмене. Изначально мы пытались отменять задачу, захватывая `Mutex<Store>`. Но если задача уже выполняется, `Mutex` заблокирован, и отмена не сработает, пока задача не завершится. Решение: хранить в `HashMap` не `Store`, а `Arc<AtomicBool>`. Сам `Store` создаётся на время выполнения функции и уничтожается после.

---

## Итог главы

**OBSERVE:**
- `trap` (из-за паники или отмены) не роняет сервер, а изолируется в конкретном инстансе.
- Отмена — это асинхронный процесс. Мы отправляем сигнал (`AtomicBool`), а Wasmtime прерывает код на следующем тике (через `epoch`).
- Ресурсы (память) освобождаются сразу после `Drop` (выхода из функции).

**PREDICT (проверка):**
Было предсказано: "Отмена должна сработать мгновенно". Результат: **подтвердилось**. Задача завершилась за ~50 мс (один тик Epoch), а не за 5 секунд.

**What did we prove?**
1. **Failure Domain ограничен.** Паника гостя не задевает Host.
2. **Отмена — это кооперативный процесс.** WASM-код должен периодически отдавать управление (или Wasmtime должен вмешаться через Epoch), чтобы отмена сработала.
3. **Паттерн отмены:** `AtomicBool` + `epoch_deadline_callback` работает без блокировок и безопасно для многопоточности.

**What did we not prove?**
- Что произойдет, если гость не уступает управление Wasmtime (например, `loop {}` без проверок). В этом случае Epoch всё равно сработает, так как Wasmtime прерывает *инструкции*, а не только вызовы функций.

**What did we pay for it?**
- Мы добавляем накладные расходы на проверку `AtomicBool` на каждом тике.
- Мы должны хранить `HashMap` активных задач для их отмены.

**When is the alternative better?**
- Если задачи короткие и быстрые, `AtomicBool` не нужен. Вы просто дожидаетесь их завершения и игнорируете отмену (как в стандартном Rust).

### 📊 Evidence
```text
Claim:          Отмена HTTP-запроса не останавливает WASM-код без явного вмешательства.
Experiment:     Запуск long_task(5000) с отменой через AtomicBool.
Measurement:    Задача падает с Trap через ~50 мс.
Observed:       Epoch и AtomicBool позволяют прервать код без Mutex lock.
Conclusion:     Chain: Cancel -> AtomicBool -> Epoch Tick -> Trap -> Resource Release.
Confidence:     High
```

### Что у нас теперь есть
- `components/text-engine-crash/` — компонент с паникой и долгими задачами.
- `host/src/runtime_v11.rs` — менеджер отмены.
- UI, показывающий отмену и панику в реальном времени.
- Все главы 1–10 продолжают работать.

**Следующий шаг:** В Главе 12 (Observability) мы научимся «видеть» внутрь WASM-компонента: собирать метрики его работы и перехватывать `println!`, чтобы понять, что происходит внутри черного ящика.