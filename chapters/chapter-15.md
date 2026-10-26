# Часть A. CLI-автоматизация: `scripts/setup-ch15.sh`

```
#!/usr/bin/env bash
# scripts/setup-ch15.sh
#
# Автоматическая настройка Главы 15: The component becomes the unit of architecture.

set -euo pipefail
ROOT="$(cd "$(dirname "$0")/.." && pwd)"
cd "$ROOT"

echo "=== [1/7] Exclude new component from workspace ==="
if ! grep -q 'components/text-engine-v2' Cargo.toml; then
  python3 - <<'PY'
import re, pathlib
p = pathlib.Path("Cargo.toml")
s = p.read_text()
m = re.search(r'exclude\s*=\s*\[(.*?)\]', s, re.S)
if not m: raise SystemExit("exclude array not found")
inner = m.group(1).rstrip()
if not inner.endswith(','): inner += ','
inner += '\n    "components/text-engine-v2",\n'
p.write_text(s[:m.start()] + 'exclude = [' + inner + ']' + s[m.end():])
print("Added text-engine-v2 to exclude.")
PY
fi

echo "=== [2/7] Guest component v2 ==="
if [ ! -d "components/text-engine-v2" ]; then
  cargo new components/text-engine-v2 --lib --vcs none >/dev/null
fi
rm -f components/text-engine-v2/Cargo.toml components/text-engine-v2/src/lib.rs
sleep 0.1
cat > components/text-engine-v2/Cargo.toml <<'EOF'
[package]
name = "text-engine-v2"
version = "0.1.0"
edition = "2021"
[workspace]
[lib]
crate-type = ["cdylib"]
[dependencies]
wit-bindgen = "0.63"
EOF
cat > components/text-engine-v2/src/lib.rs <<'EOF'
wit_bindgen::generate!({
    path: "../../wit/text-engine",
    world: "text-engine",
});

struct TextEngineV2;

impl exports::docs::text_engine::processor::Guest for TextEngineV2 {
    fn process(input: String) -> String {
        // В отличие от v1 (которая делает to_lowercase), 
        // v2 переводит в верхний регистр и добавляет метку.
        format!("[V2] {}", input.to_uppercase())
    }
}

export!(TextEngineV2);
EOF

echo "=== [3/7] Host runtime_v15 ==="
rm -f host/src/runtime_v15.rs
sleep 0.1
cat > host/src/runtime_v15.rs <<'EOF'
use anyhow::Result;
use std::path::Path;
use std::sync::atomic::{AtomicU8, Ordering};
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
        WasiCtxView { ctx: &mut self.ctx, table: &mut self.table }
    }
}

pub struct Runtime {
    engine: Engine,
    component_v1: Component,
    component_v2: Component,
    linker: Linker<MyState>,
    active_version: AtomicU8,
}

impl Runtime {
    pub fn new(wasm_v1_path: &Path, wasm_v2_path: &Path) -> Result<Arc<Self>> {
        if !wasm_v1_path.exists() || !wasm_v2_path.exists() {
            anyhow::bail!("One of the WASM components is missing.");
        }
        let engine = Engine::default();
        let mut linker = Linker::new(&engine);
        wasmtime_wasi::p2::add_to_linker_sync(&mut linker)?;
        
        let component_v1 = Component::from_file(&engine, wasm_v1_path)?;
        let component_v2 = Component::from_file(&engine, wasm_v2_path)?;
        
        Ok(Arc::new(Self {
            engine,
            component_v1,
            component_v2,
            linker,
            active_version: AtomicU8::new(1), // По умолчанию v1
        }))
    }

    pub fn switch_version(&self, version: u8) {
        self.active_version.store(version, Ordering::SeqCst);
    }

    pub fn get_active_version(&self) -> u8 {
        self.active_version.load(Ordering::SeqCst)
    }

    pub fn process(&self, input: &str) -> Result<String> {
        let wasi = WasiCtxBuilder::new().build();
        let mut store = Store::new(&self.engine, MyState {
            ctx: wasi,
            table: wasmtime::component::ResourceTable::new(),
        });

        // Динамически выбираем компонент на основе атомарного флага
        let active = self.get_active_version();
        let component = if active == 2 { &self.component_v2 } else { &self.component_v1 };

        let bindings = TextEngine::instantiate(&mut store, component, &self.linker)?;
        let result = bindings
            .docs_text_engine_processor()
            .call_process(&mut store, input)?;

        Ok(result)
    }
}
EOF

if ! grep -q 'pub mod runtime_v15;' host/src/lib.rs; then
  echo 'pub mod runtime_v15;' >> host/src/lib.rs
fi
touch host/src/lib.rs

echo "=== [4/7] Integration tests ==="
rm -f host/tests/integration_v15.rs
sleep 0.1
cat > host/tests/integration_v15.rs <<'EOF'
use std::path::PathBuf;

fn wasm_v1_path() -> PathBuf {
    PathBuf::from(env!("CARGO_MANIFEST_DIR")).parent().unwrap()
        .join("components/text-engine-component/target/wasm32-wasip1/release/text_engine_component.wasm")
}

fn wasm_v2_path() -> PathBuf {
    PathBuf::from(env!("CARGO_MANIFEST_DIR")).parent().unwrap()
        .join("components/text-engine-v2/target/wasm32-wasip1/release/text_engine_v2.wasm")
}

#[test]
fn test_v15_hot_swap() {
    let v1 = wasm_v1_path();
    let v2 = wasm_v2_path();
    if !v1.exists() || !v2.exists() { eprintln!("Skipping"); return; }

    let rt = host::runtime_v15::Runtime::new(&v1, &v2).expect("load");
    
    // 1. По умолчанию v1 (lowercase)
    assert_eq!(rt.process("Hello World").unwrap(), "hello world");
    
    // 2. Переключаем на v2
    rt.switch_version(2);
    assert_eq!(rt.process("Hello World").unwrap(), "[V2] HELLO WORLD");
    
    // 3. Возвращаем v1
    rt.switch_version(1);
    assert_eq!(rt.process("Hello World").unwrap(), "hello world");
}
EOF

echo "=== [5/7] Visual demo (demo-web-ui-ch15) ==="
if [ ! -d "demo-web-ui-ch15" ]; then
  cargo new demo-web-ui-ch15 --bin --vcs none >/dev/null
fi
rm -f demo-web-ui-ch15/Cargo.toml demo-web-ui-ch15/src/main.rs
mkdir -p demo-web-ui-ch15/static
rm -f demo-web-ui-ch15/static/index.html
sleep 0.1
cat > demo-web-ui-ch15/Cargo.toml <<'EOF'
[package]
name = "demo-web-ui-ch15"
version.workspace = true
edition.workspace = true
[dependencies]
host = { path = "../host" }
axum = "0.8"
tokio = { version = "1", features = ["full"] }
serde_json = "1"
anyhow = "1.0"
EOF
cat > demo-web-ui-ch15/src/main.rs <<'EOF'
use axum::{response::Html, routing::{get, post}, Json, Router, extract::Path};
use serde_json::{json, Value};
use std::sync::Arc;
use host::runtime_v15::Runtime;

const INDEX_HTML: &str = include_str!("../static/index.html");
async fn index() -> Html<&'static str> { Html(INDEX_HTML) }

async fn process(axum::extract::State(rt): axum::extract::State<Arc<Runtime>>, Json(p): Json<Value>) -> Json<Value> {
    let input = p["input"].as_str().unwrap_or("").to_string();
    // ИСПРАВЛЕНО: Получаем версию ДО перемещения rt в spawn_blocking
    let version = rt.get_active_version();
    
    match tokio::task::spawn_blocking(move || rt.process(&input)).await {
        Ok(Ok(output)) => Json(json!({ "output": output, "version": version })),
        Ok(Err(e)) => Json(json!({ "error": format!("{e}") })),
        Err(e) => Json(json!({ "error": format!("join: {e}") })),
    }
}

async fn switch(axum::extract::State(rt): axum::extract::State<Arc<Runtime>>, Path(ver): Path<u8>) -> Json<Value> {
    rt.switch_version(ver);
    Json(json!({ "status": "switched", "version": ver }))
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let manifest = std::path::Path::new(env!("CARGO_MANIFEST_DIR"));
    let v1 = manifest.parent().unwrap().join("components/text-engine-component/target/wasm32-wasip1/release/text_engine_component.wasm");
    let v2 = manifest.parent().unwrap().join("components/text-engine-v2/target/wasm32-wasip1/release/text_engine_v2.wasm");
    
    if !v1.exists() || !v2.exists() { std::process::exit(1); }

    let runtime = Runtime::new(&v1, &v2)?;
    let app = Router::new()
        .route("/", get(index))
        .route("/process", post(process))
        .route("/switch/{ver}", post(switch))
        .with_state(runtime);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3014").await?;
    println!("🚀 Demo UI (Ch15): http://127.0.0.1:3014");
    axum::serve(listener, app).await?;
    Ok(())
}
EOF

cat > demo-web-ui-ch15/static/index.html <<'HTML_EOF'
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8"><title>Ch15: Replaceability</title>
<style>
  :root { --bg:#0f172a; --panel:#1e293b; --br:#334155; --tx:#e2e8f0; --mu:#94a3b8; --ac:#38bdf8; --ok:#22c55e; --er:#ef4444; --wr:#f59e0b; --mo:ui-monospace,SFMono-Regular,Menlo,Consolas,monospace; }
  *{box-sizing:border-box;margin:0;padding:0}
  body{font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,sans-serif; background:var(--bg);color:var(--tx);min-height:100vh; display:flex;align-items:center;justify-content:center;padding:2rem}
  .box{max-width:800px;width:100%;background:var(--panel); border:1px solid var(--br);border-radius:12px;padding:2rem; box-shadow:0 20px 60px rgba(0,0,0,.4)}
  h1{font-size:1.6rem;font-weight:600;text-align:center;margin-bottom:.35rem; background:linear-gradient(135deg,var(--ac),#a78bfa); -webkit-background-clip:text; background-clip:text; -webkit-text-fill-color:transparent}
  .sub{color:var(--mu);text-align:center;font-size:.9rem;margin-bottom:2rem}
  .versions { display: flex; gap: 1rem; justify-content: center; margin-bottom: 2rem; }
  .v-btn { padding: 0.8rem 1.5rem; border-radius: 8px; border: 2px solid var(--br); background: transparent; color: var(--tx); font-weight: 600; cursor: pointer; transition: 0.15s; }
  .v-btn.active { border-color: var(--ac); background: rgba(56,189,248,0.1); color: var(--ac); }
  .input-row { display: flex; gap: 1rem; margin-bottom: 1.5rem; }
  input[type=text] { flex-grow: 1; padding: .8rem; background: var(--bg); border: 1px solid var(--br); color: var(--tx); border-radius: 8px; font-family: var(--mo); }
  button.process { padding: .8rem 1.5rem; background: var(--ac); color: #0c1222; border: none; border-radius: 8px; font-weight: 600; cursor: pointer; }
  .result { background: #000; border-radius: 8px; padding: 1.5rem; font-family: var(--mo); font-size: 1rem; color: var(--ok); min-height: 60px; }
</style>
</head>
<body>
<div class="box">
  <h1>Component Replaceability</h1>
  <p class="sub">Switch between v1 and v2 without restarting the server</p>

  <div class="versions">
    <button class="v-btn active" id="btn-v1" onclick="switchVersion(1)">Version 1 (Lowercase)</button>
    <button class="v-btn" id="btn-v2" onclick="switchVersion(2)">Version 2 (Uppercase)</button>
  </div>

  <div class="input-row">
    <input type="text" id="input" value="Hello WASM">
    <button class="process" onclick="run()">Process</button>
  </div>

  <div class="result" id="out">Result will appear here...</div>
</div>
<script>
async function switchVersion(v) {
  await fetch(`/switch/${v}`, { method: 'POST' });
  document.getElementById('btn-v1').classList.toggle('active', v === 1);
  document.getElementById('btn-v2').classList.toggle('active', v === 2);
}

async function run() {
  const out = document.getElementById('out');
  out.textContent = 'Processing...';
  try {
    const res = await fetch('/process', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ input: document.getElementById('input').value })
    });
    const data = await res.json();
    if (data.error) { out.textContent = 'Error: ' + data.error; }
    else { out.textContent = data.output; }
  } catch(e) { out.textContent = 'Network error: ' + e; }
}
</script>
</body>
</html>
HTML_EOF

echo "=== [6/7] Ensure v1 is built ==="
(cd components/text-engine-component && cargo component build --release --quiet)

echo "=== [7/7] Build & verify ==="
(cd components/text-engine-v2 && cargo component build --release --quiet)
cargo check -p host --lib --quiet
cargo check -p demo-web-ui-ch15 --quiet

echo "✅ Готово. Следующие шаги:"
echo "  1. Тесты:  (cd host && cargo test --test integration_v15)"
echo "  2. Демо:   cargo run -p demo-web-ui-ch15 --release"
echo "  3. Открыть: http://127.0.0.1:3014"
```

---

# Часть B. Глава 15. The component becomes the unit of architecture (The Pivot)

**Зачем создана эта глава?**
До сих пор мы строили инфраструктуру вокруг одного-единственного компонента. Но если мы просто заменим `text-engine.wasm` на другой файл в папке и перезапустим сервер, мы не докажем ничего нового — так работают обычные бинарники. Настоящая сила WebAssembly Component Model в том, что компонент можно заменить **во время работы системы (Hot Swap)**, при условии, что он реализует тот же самый WIT-контракт. Эта глава — поворотный момент (The Pivot). Здесь мы перестаем писать `monolith` и начинаем строить **платформу**, где реализация — это просто сменный артефакт.

**Какую пользу несёт глава?**
1. **Contract vs Implementation:** Вы увидите, что WIT-контракт — это единственный архитектурный закон. Всё, что внутри (v1, v2, v3) — это детали реализации, которые можно менять на лету.
2. **Динамическая маршрутизация:** Мы реализуем паттерн `Plugin Architecture`, где Host использует `AtomicU8` для мгновенного переключения между загруженными компонентами.
3. **Изоляция:** Вы поймете, как загружать несколько разных `.wasm` файлов в один `Engine` и выбирать нужный при создании нового `Store`.

**Чего мы хотим достичь по результатам тестирования?**
1. Создать `text-engine-v2`, который реализует тот же интерфейс `process(string) -> string`, но меняет текст в верхний регистр (в отличие от v1, которая приводит к нижнему).
2. Настроить `Runtime` так, чтобы он хранил обе версии и мог переключаться между ними по HTTP-запросу.
3. Убедиться, что при переключении с v1 на v2 и обратно выходные данные меняются, но API-контракт (`/process`) остается неизменным.

---

### 🚦 Starting point
У нас есть `text-engine-component` (Глава 2, делает lowercase) и полностью рабочая инфраструктура рантаймов. Мы создаём новый компонент `text-engine-v2` и модуль `runtime_v15.rs`.

### 🎯 PREDICT (Предсказание)
*Запишите свой прогноз перед запуском:*
> **Если мы загрузим два разных компонента (v1 и v2) в один `Engine`, займёт ли переключение между ними много времени? Что произойдет с уже запущенными в этот момент запросами?**

---

## Этап 1. Исходные коды

### Структура
**Новые файлы:**
- `components/text-engine-v2/` — вторая реализация (Uppercase).
- `host/src/runtime_v15.rs` — рантайм с динамической маршрутизацией.
- `host/tests/integration_v15.rs` — тесты.
- `demo-web-ui-ch15/` — UI.

### Ключевые фрагменты кода

**1. `components/text-engine-v2/src/lib.rs`**
```rust
wit_bindgen::generate!({
    path: "../../wit/text-engine", // Тот же самый WIT-контракт!
    world: "text-engine",
});

impl exports::docs::text_engine::processor::Guest for TextEngineV2 {
    fn process(input: String) -> String {
        // Изменена только логика. Контракт остался тем же.
        format!("[V2] {}", input.to_uppercase())
    }
}
```

**2. `host/src/runtime_v15.rs` (Динамический выбор)**
```rust
pub struct Runtime {
    engine: Engine,
    component_v1: Component,
    component_v2: Component,
    linker: Linker<MyState>,
    active_version: AtomicU8, // Атомарный переключатель
}

pub fn process(&self, input: &str) -> Result<String> {
    // ...
    let active = self.get_active_version();
    // Выбираем нужный Component (код) для инстанцирования
    let component = if active == 2 { &self.component_v2 } else { &self.component_v1 };

    let bindings = TextEngine::instantiate(&mut store, component, &self.linker)?;
    let result = bindings.docs_text_engine_processor().call_process(&mut store, input)?;
    Ok(result)
}
```

---

## Этап 2. Тесты

**`host/tests/integration_v15.rs`**

### Команды запуска
```bash
(cd components/text-engine-component && cargo component build --release)
(cd components/text-engine-v2 && cargo component build --release)
(cd host && cargo test --test integration_v15)
```

**Что проверяется:**
1. При активной v1 `process("Hello")` возвращает `"hello"`.
2. После вызова `switch_version(2)` возвращается `"[V2] HELLO"`.
3. После вызова `switch_version(1)` снова возвращается `"hello"`.

---

## Этап 3. Демо-пример

Визуальный сервер `demo-web-ui-ch15` (порт 3014) с переключателем версий.

### Запуск
```bash
cargo run -p demo-web-ui-ch15 --release
# open http://127.0.0.1:3014
```

### OBSERVE (Наблюдение)

1. Выберите **Version 1**, введите "Hello WASM" и нажмите **Process**. Результат: `hello wasm`.
2. Переключитесь на **Version 2**, снова нажмите **Process**. Результат: `[V2] HELLO WASM`.
3. Сервер не перезапускался. Код HTTP-обработчика не менялся.

---

## 🔄 BREAK (Поломка) — что будет, если контракты не совпадут?

**Цель:** Увидеть, что Hot Swap работает только при строгом соблюдении WIT-контракта.
**Шаг 1.** Зайдите в `wit/text-engine/world.wit` и измените `process: func(input: string) -> string` на `process: func(input: u32) -> u32`.
**Шаг 2.** Попробуйте собрать `text-engine-v2`.
* *Результат:* Ошибка компиляции `E0053` (несовпадение типов).
**Урок:** WIT-контракт — это единственный архитектурный закон. Вы не можете заменить реализацию, если она нарушает закон, даже если она «лучше».

---

## 🔄 Итеративная проверка

**Нужны ли доработки после тестов?**
Да, при написании HTTP-обработчика мы наткнулись на проблему владения `Arc<Runtime>`. Поскольку `rt` перемещается в `spawn_blocking`, мы не могли запросить `version` после. Мы исправили это, запросив `version` (которая реализует `Copy`) **до** перемещения `rt` в замыкание. Это напоминание о том, что Rust строго следит за временем жизни данных.

---

## Итог главы

**OBSERVE:**
- v1 и v2 могут сосуществовать в одном `Engine`.
- Переключение между версиями — это просто смена `Component` (кода) при создании нового `Store`.
- HTTP-слой (`/process`) вообще ничего не знает о версиях.

**PREDICT (проверка):**
Было предсказано: "Переключение займёт много времени". Результат: **не подтвердилось**. Переключение — это просто запись `u8` в `AtomicU8` (наносекунды). Код компонента уже скомпилирован в нативный код (JIT), поэтому создание нового инстанса занимает микросекунды.

**What did we prove?**
1. **Компонент — это единица архитектуры.** Вы можете менять реализации, не переписывая Host.
2. **WIT — это контракт.** Пока контракт неизменен, система стабильна.
3. **Hot Swap на уровне роутинга работает.**

**What did we not prove?**
- Безопасное обновление (Safe Hot Swap) **под нагрузкой**. В нашем коде мы просто переключаем флаг. Но что, если в момент переключения уже есть 100 активных запросов на v1? Они должны завершиться на v1, а новые пойти на v2. Без `drain` (слива) это не сработает.

**What did we pay for it?**
- Хост вынужден хранить в памяти все версии компонентов (`component_v1` и `component_v2`).
- Появляется управляющий слой (Control Plane), который переключает версии.

**When is the alternative better?**
- Если обновления происходят раз в месяц, можно просто перезапустить сервер (Rolling Update). Hot Swap нужен для систем, где downtime недопустим (например, биржевые торги).

### 📊 Evidence
```text
Claim:          Компонент можно заменить в рантайме без перезапуска сервера.
Experiment:     Переключение версии в работающем приложении.
Measurement:    v1 -> "hello", v2 -> "[V2] HELLO".
Observed:       HTTP-слой не изменился.
Conclusion:     WIT-контракт позволяет отделить реализацию от Host.
Confidence:     High
```

### Что у нас теперь есть
- `components/text-engine-v2/` — вторая реализация.
- `host/src/runtime_v15.rs` — маршрутизатор компонентов.
- UI для переключения версий в реальном времени.
- Все главы 1–14 не тронуты.

**Следующий шаг:** В Главе 16 (Hot-swap mechanics) мы решим проблему `drain`. Мы научимся переключать версии так, чтобы старые запросы доходили до конца на старой версии, а новые шли на новую. Это основа безопасного развертывания (Zero-Downtime Deployment).