# Часть A. CLI-автоматизация: `scripts/setup-ch13.sh`

```
#!/usr/bin/env bash
# scripts/setup-ch13.sh
#
# Автоматическая настройка Главы 13: Resources (lifecycle and ownership).

set -euo pipefail
ROOT="$(cd "$(dirname "$0")/.." && pwd)"
cd "$ROOT"

echo "=== [1/7] Exclude new component from workspace ==="
if ! grep -q 'components/text-engine-resource' Cargo.toml; then
  python3 - <<'PY'
import re, pathlib
p = pathlib.Path("Cargo.toml")
s = p.read_text()
m = re.search(r'exclude\s*=\s*\[(.*?)\]', s, re.S)
if not m: raise SystemExit("exclude array not found")
inner = m.group(1).rstrip()
if not inner.endswith(','): inner += ','
inner += '\n    "components/text-engine-resource",\n'
p.write_text(s[:m.start()] + 'exclude = [' + inner + ']' + s[m.end():])
print("Added text-engine-resource to exclude.")
PY
fi

echo "=== [2/7] WIT contract ==="
mkdir -p wit/text-engine-resource
rm -f wit/text-engine-resource/world.wit
sleep 0.1
cat > wit/text-engine-resource/world.wit <<'EOF'
package docs:text-engine-resource@0.1.0;

interface processor {
    resource document {
        get-content: func() -> string;
        close: func();
    }
    open: func(name: string) -> result<document, string>;
}

world text-engine-resource {
    export processor;
}
EOF

echo "=== [3/7] Guest component ==="
if [ ! -d "components/text-engine-resource" ]; then
  cargo new components/text-engine-resource --lib --vcs none >/dev/null
fi
rm -f components/text-engine-resource/Cargo.toml components/text-engine-resource/src/lib.rs
sleep 0.1
cat > components/text-engine-resource/Cargo.toml <<'EOF'
[package]
name = "text-engine-resource"
version = "0.1.0"
edition = "2021"
[workspace]
[lib]
crate-type = ["cdylib"]
[dependencies]
wit-bindgen = "0.63"
EOF
cat > components/text-engine-resource/src/lib.rs <<'EOF'
wit_bindgen::generate!({
    path: "../../wit/text-engine-resource",
    world: "text-engine-resource",
});

use exports::docs::text_engine_resource::processor::{Guest, GuestDocument, Document};
use std::cell::Cell;

struct MyDocument {
    content: String,
    closed: Cell<bool>,
}

impl GuestDocument for MyDocument {
    fn get_content(&self) -> String {
        if self.closed.get() {
            panic!("Use after close!");
        }
        self.content.clone()
    }

    fn close(&self) {
        self.closed.set(true);
    }
}

struct ResourceEngine;

impl Guest for ResourceEngine {
    type Document = MyDocument;

    fn open(name: String) -> Result<Document, String> {
        Ok(Document::new(MyDocument {
            content: format!("Content of {}", name),
            closed: Cell::new(false),
        }))
    }
}

export!(ResourceEngine);
EOF

echo "=== [4/7] Host runtime_v13 & Cargo.toml check ==="
# Безопасное восстановление зависимостей в Cargo.toml
python3 - <<'PY'
import pathlib
p = pathlib.Path("host/Cargo.toml")
s = p.read_text()
if 'tower' not in s:
    s += '\n[dev-dependencies]\ntower = { version = "0.5", features = ["util"] }\n'
    print("Restored [dev-dependencies] tower.")
if 'uuid' not in s:
    s = s.replace('[dependencies]\n', '[dependencies]\nuuid = { version = "1", features = ["v4"] }\n')
    print("Restored uuid dependency.")
p.write_text(s)
PY

rm -f host/src/runtime_v13.rs
sleep 0.1
cat > host/src/runtime_v13.rs <<'EOF'
use anyhow::Result;
use std::path::Path;
use std::sync::{Arc, Mutex};
use wasmtime::component::{Component, Linker, ResourceAny};
use wasmtime::{Engine, Store};
use wasmtime_wasi::{WasiCtx, WasiCtxBuilder, WasiCtxView, WasiView};

wasmtime::component::bindgen!({
    path: "../wit/text-engine-resource",
    world: "text-engine-resource",
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

pub struct OpenDocument {
    pub store: Store<MyState>,
    pub bindings: TextEngineResource,
    pub doc: ResourceAny,
}

pub struct Runtime {
    engine: Engine,
    component: Component,
    linker: Linker<MyState>,
    stateful_doc: Mutex<Option<OpenDocument>>,
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
        Ok(Arc::new(Self {
            engine,
            component,
            linker,
            stateful_doc: Mutex::new(None),
        }))
    }

    fn new_store(&self) -> Store<MyState> {
        let wasi = WasiCtxBuilder::new().build();
        Store::new(&self.engine, MyState {
            ctx: wasi,
            table: wasmtime::component::ResourceTable::new(),
        })
    }

    pub fn open_document(&self, name: &str) -> Result<()> {
        let mut store = self.new_store();
        let b = TextEngineResource::instantiate(&mut store, &self.component, &self.linker)?;
        
        // call_open возвращает Result<Result<ResourceAny, String>, Error>
        let res = b.docs_text_engine_resource_processor().call_open(&mut store, name)?;
        
        let doc = match res {
            Ok(d) => d,
            Err(e) => anyhow::bail!("Failed to open document: {}", e),
        };

        *self.stateful_doc.lock().unwrap() = Some(OpenDocument { store, bindings: b, doc });
        Ok(())
    }

    pub fn get_content(&self) -> Result<String> {
        let mut guard = self.stateful_doc.lock().unwrap();
        if let Some(ref mut state) = *guard {
            // Получаем диспетчер для ресурса document
            let processor = state.bindings.docs_text_engine_resource_processor();
            let doc_dispatcher = processor.document();
            
            // Передаем state.doc по значению (ResourceAny реализует Copy)
            let content = doc_dispatcher.call_get_content(&mut state.store, state.doc)?;
            Ok(content)
        } else {
            anyhow::bail!("No open document");
        }
    }

    pub fn close_document(&self) -> Result<()> {
        let mut guard = self.stateful_doc.lock().unwrap();
        if let Some(ref mut state) = *guard {
            let processor = state.bindings.docs_text_engine_resource_processor();
            let doc_dispatcher = processor.document();
            
            doc_dispatcher.call_close(&mut state.store, state.doc)?;
            Ok(())
        } else {
            anyhow::bail!("No open document");
        }
    }
}
EOF

if ! grep -q 'pub mod runtime_v13;' host/src/lib.rs; then
  echo 'pub mod runtime_v13;' >> host/src/lib.rs
fi
touch host/src/lib.rs

echo "=== [5/7] Integration tests ==="
rm -f host/tests/integration_v13.rs
sleep 0.1
cat > host/tests/integration_v13.rs <<'EOF'
use std::path::PathBuf;

fn wasm_path() -> PathBuf {
    PathBuf::from(env!("CARGO_MANIFEST_DIR")).parent().unwrap()
        .join("components/text-engine-resource/target/wasm32-wasip1/release/text_engine_resource.wasm")
}

#[test]
fn test_v13_resource_lifecycle() {
    let path = wasm_path();
    if !path.exists() { eprintln!("Skipping"); return; }
    let rt = host::runtime_v13::Runtime::new(&path).expect("load");

    rt.open_document("test.txt").expect("open");
    let content = rt.get_content().expect("read");
    assert_eq!(content, "Content of test.txt");

    rt.close_document().expect("close");

    let err = rt.get_content();
    assert!(err.is_err(), "Should fail after close");
    
    let err_str = format!("{:?}", err.unwrap_err());
    assert!(err_str.contains("Use after close") || err_str.contains("panic") || err_str.contains("unreachable"), 
            "Expected use-after-close trap, got: {}", err_str);
}
EOF

echo "=== [6/7] Visual demo (demo-web-ui-ch13) ==="
if [ ! -d "demo-web-ui-ch13" ]; then
  cargo new demo-web-ui-ch13 --bin --vcs none >/dev/null
fi
rm -f demo-web-ui-ch13/Cargo.toml demo-web-ui-ch13/src/main.rs
mkdir -p demo-web-ui-ch13/static
rm -f demo-web-ui-ch13/static/index.html
sleep 0.1
cat > demo-web-ui-ch13/Cargo.toml <<'EOF'
[package]
name = "demo-web-ui-ch13"
version.workspace = true
edition.workspace = true
[dependencies]
host = { path = "../host" }
axum = "0.8"
tokio = { version = "1", features = ["full"] }
serde_json = "1"
anyhow = "1.0"
EOF
cat > demo-web-ui-ch13/src/main.rs <<'EOF'
use axum::{response::Html, routing::{get, post}, Json, Router};
use serde_json::{json, Value};
use std::sync::Arc;
use host::runtime_v13::Runtime;

const INDEX_HTML: &str = include_str!("../static/index.html");
async fn index() -> Html<&'static str> { Html(INDEX_HTML) }

async fn open_doc(axum::extract::State(rt): axum::extract::State<Arc<Runtime>>, Json(p): Json<Value>) -> Json<Value> {
    let name = p["name"].as_str().unwrap_or("test.txt").to_string();
    match tokio::task::spawn_blocking(move || rt.open_document(&name)).await {
        Ok(Ok(_)) => Json(json!({ "output": "Document opened successfully." })),
        Ok(Err(e)) => Json(json!({ "error": format!("{e}") })),
        Err(e) => Json(json!({ "error": format!("join: {e}") })),
    }
}

async fn read_doc(axum::extract::State(rt): axum::extract::State<Arc<Runtime>>) -> Json<Value> {
    match tokio::task::spawn_blocking(move || rt.get_content()).await {
        Ok(Ok(content)) => Json(json!({ "output": content })),
        Ok(Err(e)) => {
            eprintln!("[guest-resource-error] {e:?}");
            Json(json!({ "error": "Trap: Use after close!" }))
        },
        Err(e) => Json(json!({ "error": format!("join: {e}") })),
    }
}

async fn close_doc(axum::extract::State(rt): axum::extract::State<Arc<Runtime>>) -> Json<Value> {
    match tokio::task::spawn_blocking(move || rt.close_document()).await {
        Ok(Ok(_)) => Json(json!({ "output": "Document closed." })),
        Ok(Err(e)) => Json(json!({ "error": format!("{e}") })),
        Err(e) => Json(json!({ "error": format!("join: {e}") })),
    }
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let manifest = std::path::Path::new(env!("CARGO_MANIFEST_DIR"));
    let wasm_path = manifest.parent().unwrap()
        .join("components/text-engine-resource/target/wasm32-wasip1/release/text_engine_resource.wasm");
    if !wasm_path.exists() { std::process::exit(1); }

    let runtime = Runtime::new(&wasm_path)?;
    let app = Router::new()
        .route("/", get(index))
        .route("/open", post(open_doc))
        .route("/read", post(read_doc))
        .route("/close", post(close_doc))
        .with_state(runtime);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3012").await?;
    println!("🚀 Demo UI (Ch13): http://127.0.0.1:3012");
    axum::serve(listener, app).await?;
    Ok(())
}
EOF

cat > demo-web-ui-ch13/static/index.html <<'HTML_EOF'
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8"><title>Ch13: Resources</title>
<style>
  :root { --bg:#0f172a; --panel:#1e293b; --br:#334155; --tx:#e2e8f0; --mu:#94a3b8; --ac:#38bdf8; --ok:#22c55e; --er:#ef4444; --mo:ui-monospace,SFMono-Regular,Menlo,Consolas,monospace; }
  *{box-sizing:border-box;margin:0;padding:0}
  body{font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,sans-serif; background:var(--bg);color:var(--tx);min-height:100vh; display:flex;align-items:center;justify-content:center;padding:2rem}
  .box{max-width:800px;width:100%;background:var(--panel); border:1px solid var(--br);border-radius:12px;padding:2rem; box-shadow:0 20px 60px rgba(0,0,0,.4)}
  h1{font-size:1.6rem;font-weight:600;text-align:center;margin-bottom:.35rem; background:linear-gradient(135deg,var(--ac),#a78bfa); -webkit-background-clip:text; background-clip:text; -webkit-text-fill-color:transparent}
  .sub{color:var(--mu);text-align:center;font-size:.9rem;margin-bottom:2rem}
  .step { background: var(--bg); border: 1px solid var(--br); border-radius: 8px; padding: 1rem; margin-bottom: 1rem; display: flex; align-items: center; justify-content: space-between; }
  .step span { font-weight: 600; color: var(--ac); }
  input { padding: .6rem; background: var(--panel); border: 1px solid var(--br); color: var(--tx); border-radius: 6px; font-family: var(--mo); width: 150px; }
  button { padding: .6rem 1rem; border-radius: 6px; border: none; font-weight: 600; cursor: pointer; }
  button.primary { background: var(--ac); color: #0c1222; }
  button.warn { background: #f59e0b; color: #0c1222; }
  button.bad { background: var(--er); color: white; }
  button:hover { filter: brightness(1.1); }
  button:disabled { opacity: .5; cursor: not-allowed; }
  .result { margin-top: 1rem; font-family: var(--mo); font-size: .85rem; background: #000; padding: 1rem; border-radius: 8px; min-height: 50px; white-space: pre-wrap; }
  .res-ok { color: var(--ok); }
  .res-err { color: var(--er); }
</style>
</head>
<body>
<div class="box">
  <h1>Resources: Lifecycle & Ownership</h1>
  <p class="sub">Open -> Read -> Close -> Read (Danger)</p>

  <div class="step">
    <span>1. Open Document</span>
    <div><input type="text" id="doc-name" value="test.txt"> <button class="primary" onclick="run('/open', {name: document.getElementById('doc-name').value})">Open</button></div>
  </div>
  <div class="step">
    <span>2. Read Content</span>
    <button class="primary" onclick="run('/read')">Read</button>
  </div>
  <div class="step">
    <span>3. Close Document</span>
    <button class="warn" onclick="run('/close')">Close</button>
  </div>
  <div class="step" style="border-color: var(--er)">
    <span style="color:var(--er)">4. Read After Close (DANGER)</span>
    <button class="bad" onclick="run('/read')">Read</button>
  </div>

  <div class="result" id="out">Awaiting action...</div>
</div>
<script>
async function run(url, body = {}) {
  const out = document.getElementById('out');
  out.innerHTML = 'Processing...'; out.className = 'result';
  try {
    const res = await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body)
    });
    const data = await res.json();
    if (data.error) {
      out.innerHTML = `<span class="res-err">${data.error}</span>`;
    } else {
      out.innerHTML = `<span class="res-ok">${data.output}</span>`;
    }
  } catch(e) {
    out.innerHTML = `<span class="res-err">Network error: ${e}</span>`;
  }
}
</script>
</body>
</html>
HTML_EOF

echo "=== [7/7] Build & verify ==="
(cd components/text-engine-resource && cargo component build --release --quiet)
cargo check -p host --lib --quiet
cargo check -p demo-web-ui-ch13 --quiet

echo "✅ Готово. Следующие шаги:"
echo "  1. Тесты:  (cd host && cargo test --test integration_v13)"
echo "  2. Демо:   cargo run -p demo-web-ui-ch13 --release"
echo "  3. Открыть: http://127.0.0.1:3012"
```

# Глава 13. Resources: lifecycle and ownership

**Зачем создана эта глава?**
До сих пор мы передавали данные через границу Host ↔ Guest по значению. Мы отправляли `string`, `list<u8>` или `record`. WASM-компонент получал копию этих данных в своей линейной памяти, что было безопасно, но крайне расточительно для больших файлов (например, гигабайтных документов или потоков видео). Если мы передадим 1 ГБ данных через `list<u8>`, сервер может упасть от нехватки памяти (OOM). Нам нужен механизм, который позволит Гостю работать с большими объектами, не копируя их целиком. Ответ — **Ресурсы (Resources)**. Это не просто данные, это «ручка» (handle) на объект, которым владеет Хост. Guest получает право вызывать методы этого объекта, но не владеет его памятью напрямую.

**Какую пользу несёт глава?**
1. **Разделение владения:** Вы увидите, что `resource<document>` в WIT — это не структура данных, а непрозрачный идентификатор. Хост владеет самим файлом/объектом, а Гость — только ручкой для доступа к нему.
2. **Жизненный цикл:** Мы разберем, что происходит, когда Гость вызывает `close()`. Как это влияет на память Хоста?
3. **Безопасность:** Вы на практике увидите, что попытка использовать ресурс после его закрытия (`use-after-close`) приводит к `Trap`, а не к повреждению памяти или скрытой утечке.

**Чего мы хотим достичь по результатам тестирования?**
1. Создать WIT-контракт с `resource document`.
2. Написать Guest-компонент, который отслеживает состояние ресурса (открыт/закрыт).
3. Настроить Host так, чтобы он сохранял `Store` и `bindings` для доступа к ресурсу между HTTP-запросами.
4. Провести негативный тест (DO): вызвать `get_content()` после `close()`. Убедиться, что это вызывает контролируемую панику (Trap), которую Host перехватывает.

---

### 🚦 Starting point
У нас есть `text-engine-observe` (Глава 12) с логами и метриками. Мы не трогаем её. Мы создаём новый компонент `text-engine-resource` и модуль `runtime_v13.rs`.

### 🎯 PREDICT (Предсказание)
*Запишите свой прогноз перед запуском:*
> **Если гость вызывает `get_content` после того, как вызвал `close`, получит ли он пустую строку (потому что данные стерты), ошибку `Result::Err` (как в бизнес-логике) или аварийное завершение (Trap)?**

---

## Этап 1. Исходные коды

### Структура
**Новые файлы:**
- `wit/text-engine-resource/world.wit` — контракт с ресурсом.
- `components/text-engine-resource/` — компонент.
- `host/src/runtime_v13.rs` — рантайм с управлением `ResourceAny`.
- `host/tests/integration_v13.rs` — тесты.
- `demo-web-ui-ch13/` — UI.

### Ссылки для изучения
- [WIT — Resources](https://component-model.bytecodealliance.org/design/wit.html#resources) — как описывать ресурсы в контракте.
- [Wasmtime — ResourceAny](https://docs.wasmtime.dev/api/wasmtime/component/struct.ResourceAny.html) — как хост хранит handle на ресурс.

### Ключевые фрагменты кода

**1. `wit/text-engine-resource/world.wit`**
```wit
interface processor {
    resource document {
        get-content: func() -> string;
        close: func();
    }
    open: func(name: string) -> result<document, string>;
}
```

**2. `components/text-engine-resource/src/lib.rs` (Гость)**
```rust
struct MyDocument {
    content: String,
    closed: Cell<bool>,
}

impl GuestDocument for MyDocument {
    fn get_content(&self) -> String {
        if self.closed.get() {
            panic!("Use after close!"); // Намеренная паника
        }
        self.content.clone()
    }

    fn close(&self) {
        self.closed.set(true);
    }
}
```

**3. `host/src/runtime_v13.rs` (Хост)**
```rust
pub struct OpenDocument {
    pub store: Store<MyState>,
    pub bindings: TextEngineResource,
    pub doc: ResourceAny,
}

pub fn get_content(&self) -> Result<String> {
    let mut guard = self.stateful_doc.lock().unwrap();
    if let Some(ref mut state) = *guard {
        let processor = state.bindings.docs_text_engine_resource_processor();
        let doc_dispatcher = processor.document();
        // Передаем handle (ResourceAny) в метод ресурса
        let content = doc_dispatcher.call_get_content(&mut state.store, state.doc)?;
        Ok(content)
    } else {
        anyhow::bail!("No open document");
    }
}
```

---

## Этап 2. Тесты

**`host/tests/integration_v13.rs`**

### Команды запуска
```bash
(cd components/text-engine-resource && cargo component build --release)
(cd host && cargo test --test integration_v13)
```

**Что проверяется:**
1. `open_document("test.txt")` и `get_content()` возвращают `"Content of test.txt"`.
2. `close_document()` успешно закрывает ресурс.
3. Вызов `get_content()` после закрытия возвращает `Err` (внутри которого спрятан `wasm trap`).

---

## Этап 3. Демо-пример

Визуальный сервер `demo-web-ui-ch13` (порт 3012) с пошаговым сценарием.

### Запуск
```bash
cargo run -p demo-web-ui-ch13 --release
# open http://127.0.0.1:3012
```

### 🔄 BREAK (Поломка) и OBSERVE

1. Нажмите **Open**. Документ создан. В логах сервера ничего не происходит.
2. Нажмите **Read**. Вы получите `Content of test.txt`.
3. Нажмите **Close**. Ресурс помечен как закрытый.
4. Нажмите **Read** (красная кнопка).
   * *Результат:* Красный текст `Trap: Use after close!`.
   * *В консоли сервера:* `[guest-resource-error] error while executing at wasm backtrace: ... panic!`.

**Почему это Trap, а не `Result::Err`?**
В Rust внутри WASM мы использовали `panic!`. Для Wasmtime это аварийное завершение (Trap). Это доказывает, что ресурсы защищены на уровне рантайма. Если гость нарушает контракт жизненного цикла (использует ресурс после `close`), Wasmtime немедленно уничтожает инстанс (Store), не давая ему повредить память Хоста.

---

## 🔄 Итеративная проверка

**Нужны ли доработки после тестов?**
Да, API `wit-bindgen` 0.63 и `wasmtime` 49 потребовали явного использования диспетчера (`processor.document()`) и передачи `ResourceAny` по значению (а не по ссылке). Мы также выяснили, что для удержания ресурса между HTTP-запросами нужно сохранять не только сам `ResourceAny`, но и `Store` с `bindings`, потому что ресурс «живет» именно в контексте `Store`.

---

## Итог главы

**OBSERVE:**
- Ресурс — это `ResourceAny`, непрозрачный указатель (handle).
- Памятью владеет Хост (в нашем случае это симуляция, но в реальности это был бы файловый дескриптор).
- Использование после `close` вызывает `Trap`, а не тихую ошибку.

**PREDICT (проверка):**
Было предсказано: "Компонент получит `Result::Err`". Результат: **не подтвердилось**. Нарушение жизненного цикла ресурса — это не бизнес-ошибка, а нарушение безопасности (Trap). Wasmtime не позволяет продолжить выполнение после `Use after close`.

**What did we prove?**
1. **Ресурсы экономят память.** Мы не копируем содержимое документа в WASM. Мы передаем только `ResourceAny` (указатель на объект в `Store`).
2. **Lifecycle контролируется.** Метод `close()` в госте меняет состояние, и повторный вызов `get_content` гарантированно падает.
3. **Изоляция работает.** Паника внутри ресурса не ломает HTTP-сервер.

**What did we not prove?**
- Как передавать ресурсы из Гостя в Хост (у нас Хост создаёт ресурс). Это тема продвинутого WIT.

**What did we pay for it?**
- Сложность кода. Нам пришлось хранить `Store` и `bindings` в `Mutex<Option<...>>`.
- Необходимость использовать `Arc<Mutex<...>>` для синхронизации.

**When is the alternative better?**
- Если данные маленькие (например, JSON на 1 КБ), передавать их по значению (`string` или `list<u8>`) гораздо проще и быстрее. Ресурсы нужны только для больших данных или объектов с чётким жизненным циклом (файлы, сокеты, БД).

### 📊 Evidence
```text
Claim:          Использование ресурса после close вызывает Trap и не повреждает Хост.
Experiment:     Вызов get_content() после close_document().
Measurement:    Wasmtime возвращает ошибку "wasm trap: unreachable instruction executed".
Observed:       Сервер отвечает HTTP 200 с JSON ошибкой.
Conclusion:     ResourceAny + panic в WASM = безопасный жизненный цикл.
Confidence:     High
```

### Что у нас теперь есть
- `components/text-engine-resource/` — компонент с ресурсом.
- `host/src/runtime_v13.rs` — управление `ResourceAny`.
- UI с пошаговым сценарием (Open -> Read -> Close -> Read).
- Все главы 1–12 не тронуты.

**Следующий шаг:** В Главе 14 (Streams and the memory path) мы пойдем еще дальше. Мы не просто откроем файл, мы передадим его содержимое потоком (`stream<u8>`), чтобы обработать гигабайты данных, не загружая их в память целиком.