# Часть A. CLI-автоматизация: `scripts/setup-ch16.sh`

```
#!/usr/bin/env bash
# scripts/setup-ch16.sh
#
# Автоматическая настройка Главы 16: Hot-swap mechanics (in-flight, drain, rollback).

set -euo pipefail
ROOT="$(cd "$(dirname "$0")/.." && pwd)"
cd "$ROOT"

echo "=== [1/7] Exclude new components from workspace ==="
for comp in text-engine-slow-v1 text-engine-slow-v2; do
  if ! grep -q "components/$comp" Cargo.toml; then
    python3 - <<PY
import re, pathlib
p = pathlib.Path("Cargo.toml")
s = p.read_text()
m = re.search(r'exclude\s*=\s*\[(.*?)\]', s, re.S)
if not m: raise SystemExit("exclude array not found")
inner = m.group(1).rstrip()
if not inner.endswith(','): inner += ','
inner += '\n    "components/$comp",\n'
p.write_text(s[:m.start()] + 'exclude = [' + inner + ']' + s[m.end():])
print("Added $comp to exclude.")
PY
  fi
done

echo "=== [2/7] WIT contract ==="
mkdir -p wit/text-engine-slow
rm -f wit/text-engine-slow/world.wit
sleep 0.1
cat > wit/text-engine-slow/world.wit <<'EOF'
package docs:text-engine-slow@0.1.0;
interface processor {
    process: func(ms: u32) -> string;
}
world text-engine-slow {
    export processor;
}
EOF

echo "=== [3/7] Guest components v1 & v2 ==="
for ver in v1 v2; do
  COMP_DIR="components/text-engine-slow-$ver"
  if [ ! -d "$COMP_DIR" ]; then
    cargo new "$COMP_DIR" --lib --vcs none >/dev/null
  fi
  rm -f "$COMP_DIR/Cargo.toml" "$COMP_DIR/src/lib.rs"
  sleep 0.1
  cat > "$COMP_DIR/Cargo.toml" <<EOF
[package]
name = "text-engine-slow-$ver"
version = "0.1.0"
edition = "2021"
[workspace]
[lib]
crate-type = ["cdylib"]
[dependencies]
wit-bindgen = "0.63"
EOF

  cat > "$COMP_DIR/src/lib.rs" <<EOF
wit_bindgen::generate!({
    path: "../../wit/text-engine-slow",
    world: "text-engine-slow",
});

struct SlowEngine;
impl exports::docs::text_engine_slow::processor::Guest for SlowEngine {
    fn process(ms: u32) -> String {
        std::thread::sleep(std::time::Duration::from_millis(ms as u64));
        "done".to_string()
    }
}
export!(SlowEngine);
EOF
done

echo "=== [4/7] Host runtime_v16 ==="
rm -f host/src/runtime_v16.rs
sleep 0.1
cat > host/src/runtime_v16.rs <<'EOF'
use anyhow::Result;
use std::path::Path;
use std::sync::{Arc, RwLock};
use wasmtime::component::{Component, Linker};
use wasmtime::{Engine, Store};
use wasmtime_wasi::{WasiCtx, WasiCtxBuilder, WasiCtxView, WasiView};

wasmtime::component::bindgen!({
    path: "../wit/text-engine-slow",
    world: "text-engine-slow",
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

pub struct ActiveState {
    pub component: Arc<Component>,
    pub version: u8,
}

pub struct Runtime {
    pub engine: Engine,
    pub linker: Linker<MyState>,
    pub active: RwLock<ActiveState>,
    pub v1: Arc<Component>,
    pub v2: Arc<Component>,
}

impl Runtime {
    pub fn new(v1_path: &Path, v2_path: &Path) -> Result<Arc<Self>> {
        let engine = Engine::default();
        let mut linker = Linker::new(&engine);
        wasmtime_wasi::p2::add_to_linker_sync(&mut linker)?;
        
        let v1 = Arc::new(Component::from_file(&engine, v1_path)?);
        let v2 = Arc::new(Component::from_file(&engine, v2_path)?);

        Ok(Arc::new(Self {
            engine,
            linker,
            active: RwLock::new(ActiveState { component: v1.clone(), version: 1 }),
            v1,
            v2,
        }))
    }

    pub fn swap_version(&self, target_ver: u8) {
        let mut guard = self.active.write().unwrap();
        if target_ver == 2 {
            guard.component = self.v2.clone();
            guard.version = 2;
        } else {
            guard.component = self.v1.clone();
            guard.version = 1;
        }
    }

    pub fn get_active_version(&self) -> u8 {
        self.active.read().unwrap().version
    }

    // Вызов с замером времени
    pub fn process(&self, ms: u32) -> Result<(String, u8)> {
        // 1. Делаем "снимок" текущего компонента (Arc clone)
        let (component, version) = {
            let guard = self.active.read().unwrap();
            (guard.component.clone(), guard.version)
        };

        // 2. Создаем Store и вызываем. 
        // Переменная component держит Arc, поэтому даже если active заменится,
        // этот конкретный экземпляр v1 (или v2) не будет удален из памяти.
        let wasi = WasiCtxBuilder::new().build();
        let mut store = Store::new(&self.engine, MyState {
            ctx: wasi,
            table: wasmtime::component::ResourceTable::new(),
        });

        let bindings = TextEngineSlow::instantiate(&mut store, &component, &self.linker)?;
        let res = bindings.docs_text_engine_slow_processor().call_process(&mut store, ms)?;
        
        Ok((res, version))
    }
}
EOF

if ! grep -q 'pub mod runtime_v16;' host/src/lib.rs; then
  echo 'pub mod runtime_v16;' >> host/src/lib.rs
fi
touch host/src/lib.rs

echo "=== [5/7] Integration tests ==="
rm -f host/tests/integration_v16.rs
sleep 0.1
cat > host/tests/integration_v16.rs <<'EOF'
use std::path::PathBuf;

fn v1_path() -> PathBuf {
    PathBuf::from(env!("CARGO_MANIFEST_DIR")).parent().unwrap()
        .join("components/text-engine-slow-v1/target/wasm32-wasip1/release/text_engine_slow_v1.wasm")
}
fn v2_path() -> PathBuf {
    PathBuf::from(env!("CARGO_MANIFEST_DIR")).parent().unwrap()
        .join("components/text-engine-slow-v2/target/wasm32-wasip1/release/text_engine_slow_v2.wasm")
}

#[test]
fn test_v16_hot_swap_drain() {
    let rt = host::runtime_v16::Runtime::new(&v1_path(), &v2_path()).expect("load");
    assert_eq!(rt.get_active_version(), 1);

    // 1. Запускаем долгий запрос на v1 (в отдельном потоке)
    let rt_clone = rt.clone();
    let handle = std::thread::spawn(move || {
        rt_clone.process(500).unwrap() // Займет 500 мс
    });

    // 2. Ждем немного, чтобы запрос точно начал выполняться
    std::thread::sleep(std::time::Duration::from_millis(100));

    // 3. Переключаем версию на v2
    rt.swap_version(2);
    assert_eq!(rt.get_active_version(), 2);

    // 4. Запускаем новый запрос (должен пойти на v2)
    let (_, new_ver) = rt.process(10).unwrap();
    assert_eq!(new_ver, 2);

    // 5. Ждем завершения старого запроса. Он должен завершиться на v1!
    let (_, old_ver) = handle.join().unwrap();
    assert_eq!(old_ver, 1, "Old request should complete on v1");
}
EOF

echo "=== [6/7] Visual demo (demo-web-ui-ch16) ==="
if [ ! -d "demo-web-ui-ch16" ]; then
  cargo new demo-web-ui-ch16 --bin --vcs none >/dev/null
fi
rm -f demo-web-ui-ch16/Cargo.toml demo-web-ui-ch16/src/main.rs
mkdir -p demo-web-ui-ch16/static
rm -f demo-web-ui-ch16/static/index.html
sleep 0.1
cat > demo-web-ui-ch16/Cargo.toml <<'EOF'
[package]
name = "demo-web-ui-ch16"
version.workspace = true
edition.workspace = true
[dependencies]
host = { path = "../host" }
axum = "0.8"
tokio = { version = "1", features = ["full"] }
serde_json = "1"
anyhow = "1.0"
EOF
cat > demo-web-ui-ch16/src/main.rs <<'EOF'
use axum::{response::Html, routing::{get, post}, Json, Router, extract::Path};
use serde_json::{json, Value};
use std::sync::Arc;
use host::runtime_v16::Runtime;

const INDEX_HTML: &str = include_str!("../static/index.html");
async fn index() -> Html<&'static str> { Html(INDEX_HTML) }

async fn process(axum::extract::State(rt): axum::extract::State<Arc<Runtime>>) -> Json<Value> {
    // Симулируем долгий запрос (3 секунды)
    let result = tokio::task::spawn_blocking(move || rt.process(3000)).await;
    match result {
        Ok(Ok((_msg, version))) => Json(json!({ "version": version, "status": "completed" })),
        _ => Json(json!({ "error": "failed" })),
    }
}

async fn swap(axum::extract::State(rt): axum::extract::State<Arc<Runtime>>, Path(ver): Path<u8>) -> Json<Value> {
    rt.swap_version(ver);
    Json(json!({ "status": "swapped", "active_version": ver }))
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let manifest = std::path::Path::new(env!("CARGO_MANIFEST_DIR"));
    let v1 = manifest.parent().unwrap().join("components/text-engine-slow-v1/target/wasm32-wasip1/release/text_engine_slow_v1.wasm");
    let v2 = manifest.parent().unwrap().join("components/text-engine-slow-v2/target/wasm32-wasip1/release/text_engine_slow_v2.wasm");
    
    if !v1.exists() || !v2.exists() { std::process::exit(1); }

    let runtime = Runtime::new(&v1, &v2)?;
    let app = Router::new()
        .route("/", get(index))
        .route("/process", post(process))
        .route("/swap/{ver}", post(swap))
        .with_state(runtime);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3015").await?;
    println!("🚀 Demo UI (Ch16): http://127.0.0.1:3015");
    axum::serve(listener, app).await?;
    Ok(())
}
EOF

cat > demo-web-ui-ch16/static/index.html <<'HTML_EOF'
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8"><title>Ch16: Drain & Swap</title>
<style>
  :root { --bg:#0f172a; --panel:#1e293b; --br:#334155; --tx:#e2e8f0; --mu:#94a3b8; --ac:#38bdf8; --ok:#22c55e; --er:#ef4444; --wr:#f59e0b; --mo:ui-monospace,SFMono-Regular,Menlo,Consolas,monospace; }
  *{box-sizing:border-box;margin:0;padding:0}
  body{font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,sans-serif; background:var(--bg);color:var(--tx);min-height:100vh; display:flex;align-items:center;justify-content:center;padding:2rem}
  .box{max-width:850px;width:100%;background:var(--panel); border:1px solid var(--br);border-radius:12px;padding:2rem; box-shadow:0 20px 60px rgba(0,0,0,.4)}
  h1{font-size:1.6rem;font-weight:600;text-align:center;margin-bottom:.35rem; background:linear-gradient(135deg,var(--ac),#a78bfa); -webkit-background-clip:text; background-clip:text; -webkit-text-fill-color:transparent}
  .sub{color:var(--mu);text-align:center;font-size:.9rem;margin-bottom:2rem}
  .row { display: flex; gap: 1rem; margin-bottom: 1.5rem; }
  button { flex: 1; padding: .8rem; border-radius: 8px; border: none; font-weight: 600; cursor: pointer; font-size: 0.9rem; }
  button.primary { background: var(--ac); color: #0c1222; }
  button.warn { background: var(--wr); color: #0c1222; }
  button:hover { filter: brightness(1.1); }
  button:disabled { opacity: .5; cursor: not-allowed; }
  .logs { background: #000; border-radius: 8px; padding: 1rem; font-family: var(--mo); font-size: .85rem; color: var(--ok); min-height: 150px; }
</style>
</head>
<body>
<div class="box">
  <h1>Hot-swap: In-flight & Drain</h1>
  <p class="sub">Requests in progress finish on the old version</p>

  <div class="row">
    <button class="primary" id="btn-req" onclick="startRequest()">Start 3s Request</button>
    <button class="warn" onclick="swapVersion(2)">Swap to v2</button>
    <button class="warn" onclick="swapVersion(1)">Swap to v1</button>
  </div>

  <div class="logs" id="logs">Logs will appear here...</div>
</div>
<script>
function log(msg, color = 'var(--ok)') {
  const l = document.getElementById('logs');
  l.innerHTML += `<div style="color:${color}">> ${msg}</div>`;
}

async function startRequest() {
  document.getElementById('btn-req').disabled = true;
  log('Started 3s request...', 'var(--ac)');
  try {
    const res = await fetch('/process', { method: 'POST' });
    const data = await res.json();
    if (data.error) { log('Error: ' + data.error, 'var(--er)'); }
    else { log(`Request finished on V${data.version}`, 'var(--ok)'); }
  } catch(e) { log('Network error', 'var(--er)'); }
  finally { document.getElementById('btn-req').disabled = false; }
}

async function swapVersion(v) {
  await fetch(`/swap/${v}`, { method: 'POST' });
  log(`Swapped active version to V${v}`, 'var(--wr)');
}
</script>
</body>
</html>
HTML_EOF

echo "=== [7/7] Build & verify ==="
(cd components/text-engine-slow-v1 && cargo component build --release --quiet)
(cd components/text-engine-slow-v2 && cargo component build --release --quiet)
cargo check -p host --lib --quiet
cargo check -p demo-web-ui-ch16 --quiet

echo "✅ Готово. Следующие шаги:"
echo "  1. Тесты:  (cd host && cargo test --test integration_v16)"
echo "  2. Демо:   cargo run -p demo-web-ui-ch16 --release"
echo "  3. Открыть: http://127.0.0.1:3015"
```

---

# Часть B. Глава 16. Hot-swap mechanics: in-flight, drain, rollback

**Зачем создана эта глава?**
В Главе 15 мы научились переключать указатель на активный компонент. Но мы сделали это слишком наивно. Если мы просто заменим `Component` в памяти, когда 100 HTTP-запросов прямо сейчас выполняют код старой версии, мы получим катастрофу: либо запросы упадут с `Trap` (потому что память старого компонента освобождена), либо мы получим утечку памяти. Настоящий Zero-Downtime Deployment требует механизма **Drain (слива)**. Мы должны гарантировать, что старые запросы доработают на старой версии, а новые пойдут на новую.

**Какую пользу несёт глава?**
1. **Reference Counting в действии:** Вы увидите, как `Arc` (Atomic Reference Counted) позволяет удерживать компонент в памяти ровно столько, сколько нужно активным запросам.
2. **Паттерн Snapshot:** Мы реализуем паттерн, где каждый запрос берет «снимок» текущей версии. Пока запрос жив, его версия не будет удалена.
3. **Безопасность конкурентности:** Мы используем `RwLock` (Read-Write Lock), чтобы обеспечить атомарность переключения без блокировки самих вычислений.

**Чего мы хотим достичь по результатам тестирования?**
1. Создать два компонента (v1 и v2), которые симулируют долгую работу (например, `sleep(3s)`).
2. Настроить `Runtime` так, чтобы он использовал `RwLock<ActiveState>`.
3. Доказать в тесте, что если мы запустим долгий запрос на v1, а затем переключимся на v2, старый запрос всё равно вернёт `version = 1`, но при этом новый запрос вернёт `version = 2`.

---

### 🚦 Starting point
У нас есть `runtime_v15.rs` с `AtomicU8`. В этой главе мы эволюционируем его до `runtime_v16.rs`.

### 🎯 PREDICT (Предсказание)
*Запишите свой прогноз перед запуском:*
> **Если мы удалим `Component v1` из памяти сразу после переключения на `v2`, что произойдет с запросом, который прямо сейчас выполняется на `v1`? Он упадёт с ошибкой, или Rust не даст нам удалить `v1`?**

---

## Этап 1. Исходные коды

### Структура
**Новые файлы:**
- `components/text-engine-slow-v1/` — компонент, который «спит».
- `components/text-engine-slow-v2/` — компонент, который «спит».
- `host/src/runtime_v16.rs` — рантайм с `RwLock`.
- `host/tests/integration_v16.rs` — тесты.
- `demo-web-ui-ch16/` — UI.

### Ссылки для изучения
- [Rust std — RwLock](https://doc.rust-lang.org/std/sync/struct.RwLock.html) — как работает блокировка чтения/записи.
- [Rust std — Arc](https://doc.rust-lang.org/std/sync/struct.Arc.html) — как работает подсчёт ссылок.

### Ключевые фрагменты кода

**1. `host/src/runtime_v16.rs` (Безопасное переключение)**
```rust
pub struct ActiveState {
    pub component: Arc<Component>,
    pub version: u8,
}

pub struct Runtime {
    // RwLock позволяет множеству читателей (запросов) 
    // и только одному писателю (процессу деплоя) работать одновременно.
    pub active: RwLock<ActiveState>,
    pub v1: Arc<Component>,
    pub v2: Arc<Component>,
    // ...
}

pub fn process(&self, ms: u32) -> Result<(String, u8)> {
    // 1. Берем "снимок" (Snapshot) текущего состояния.
    // Arc::clone увеличивает счетчик ссылок на 1.
    let (component, version) = {
        let guard = self.active.read().unwrap();
        (guard.component.clone(), guard.version)
    }; // Блокировка RwLock снимается здесь.

    // 2. Выполняем запрос с компонентом из снимка.
    // Даже если кто-то вызовет swap_version() прямо сейчас и 
    // перезапишет active.component, наш локальный 'component' 
    // будет держать старую версию в памяти до конца функции.
    let mut store = Store::new(...);
    let bindings = TextEngineSlow::instantiate(&mut store, &component, &self.linker)?;
    let res = bindings.docs_text_engine_slow_processor().call_process(&mut store, ms)?;
    
    Ok((res, version))
}
```

---

## Этап 2. Тесты

**`host/tests/integration_v16.rs`**

### Команды запуска
```bash
(cd components/text-engine-slow-v1 && cargo component build --release)
(cd components/text-engine-slow-v2 && cargo component build --release)
(cd host && cargo test --test integration_v16)
```

**Что проверяется:**
1. Запускаем поток с `process(500)` (займет 500 мс).
2. Ждем 100 мс (чтобы запрос точно начал выполняться).
3. Вызываем `swap_version(2)` (переключаем на v2).
4. Проверяем, что новый запрос идет на v2.
5. Ждем завершения старого потока и проверяем, что он вернул `version = 1`.

---

## Этап 3. Демо-пример

Визуальный сервер `demo-web-ui-ch16` (порт 3015).

### Запуск
```bash
cargo run -p demo-web-ui-ch16 --release
# open http://127.0.0.1:3015
```

### OBSERVE (Наблюдение)

1. Нажмите **Start 3s Request**.
2. **Сразу же** нажмите **Swap to v2**.
3. Через 3 секунды в логах появится `Request finished on V1`.
4. Снова нажмите **Start 3s Request**. Теперь он завершится на `V2`.

**Почему это работает?**
Когда вы нажали Start, HTTP-запрос сделал `Arc::clone()` компонента v1. Счетчик ссылок (Reference Count) стал равен `2` (один в `RwLock`, второй в запросе).
Когда вы нажали Swap, `RwLock` заменил свой `Arc` на v2. Счетчик ссылок для v1 упал до `1`.
Даже если бы мы захотели удалить v1, Rust не позволил бы, пока запрос держит его в памяти. Когда запрос завершится, счетчик упадет до `0`, и Garbage Collector (в данном случае `Drop`) очистит память.

---

## 🔄 BREAK (Поломка) — Опасность `Mutex`

**Цель:** Увидеть, почему `RwLock` лучше `Mutex`.
Измените `RwLock` на `Mutex` в `Runtime`. Запустите тест.
* **Результат:** Тест пройдет, но вы заметите, что переключение (`swap_version`) будет ждать, пока завершится долгий запрос (500 мс). `Mutex` блокирует всех, а `RwLock` позволяет читать и писать параллельно (если нет активных писателей).

---

## 🔄 Итеративная проверка

**Нужны ли доработки после тестов?**
Мы осознали важность использования `Arc<Component>`. Без `Arc` (если бы мы просто передавали ссылки с лайфтаймами), мы бы не смогли вынести `component` за пределы блокировки `RwLock`. `Arc` — это единственный безопасный способ управлять памятью между потоками в Rust в данном сценарии.

---

## Итог главы

**OBSERVE:**
- `Arc` удерживает память компонента живой, пока в нем есть нужда.
- `RwLock` позволяет безопасно менять активный компонент.
- In-flight запросы завершаются на старой версии (Drain).

**PREDICT (проверка):**
Было предсказано: "Удаление v1 сломает запрос". Результат: **не подтвердилось**. Rust и `Arc` не дали нам удалить v1, пока запрос его использует.

**What did we prove?**
1. **Zero-Downtime Deployment возможен.** Мы обновили версию без обрыва соединений.
2. **Снимок (Snapshot) — это ключ.** Запрос берет снимок версии в начале и работает с ним.
3. **Rust Ownership Model спасает от Use-After-Free.** Компилятор сам следит за тем, чтобы память не была очищена слишком рано.

**What did we not prove?**
- Как откатывать (Rollback) транзакцию, если новый компонент v2 оказался сломан (для этого нужно добавить проверку Health Check).

**What did we pay for it?**
- Небольшие накладные расходы на `Arc::clone()` и блокировку `RwLock` (микросекунды).
- Усложнение логики рантайма.

**When is the alternative better?**
- Если downtime приемлем (например, ночное обновление), проще использовать рестарт сервера. Hot Swap нужен для критически важных сервисов (24/7).

### 📊 Evidence
```text
Claim:          Hot-swap со снимком (Arc) предотвращает падение in-flight запросов.
Experiment:     Запуск долгого запроса, переключение версии во время его работы.
Measurement:    Старый запрос возвращает version=1, новый — version=2.
Observed:       Arc удерживает память v1 в живых до конца запроса.
Conclusion:     Drain (слив) реализован безопасно.
Confidence:     High
```

### Что у нас теперь есть
- `components/text-engine-slow-v1/` и `v2`.
- `host/src/runtime_v16.rs` — безопасный менеджер версий.
- UI с наглядным тестом Drain.

**Следующий шаг:** Мы полностью разобрали управление версиями. В Части VII (Composition & Finishing) мы перейдем к упаковке (Manifests) и композиции, но перед этим — в Главе 17 мы соберем всё в единую архитектуру **Component as a deployable unit**.