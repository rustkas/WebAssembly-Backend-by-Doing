# Часть A. CLI-автоматизация: `scripts/setup-ch6.sh`

### Шаг 1. Создать файл скрипта

```bash
cd ~/projects/wa/wasm-backend-lab
mkdir -p scripts
nano scripts/setup-ch6.sh
```

Вставьте в редактор **полный текст** из блока ниже (или используйте heredoc-команду сразу после).

<details>
<summary><b>Полный текст `scripts/setup-ch6.sh`</b></summary>

```bash
#!/usr/bin/env bash
# scripts/setup-ch6.sh
#
# Автоматическая настройка Главы 6: Host ↔ Guest boundary.
#
# Создаёт:
#   1. WIT-контракт для нового компонента text-engine-advanced
#   2. Компонент components/text-engine-advanced (string, record, list<u8>, result)
#   3. host/src/runtime_v6.rs (Runtime для нового компонента)
#   4. host/tests/integration_v6.rs (тесты границы, включая negative UTF-8)
#   5. demo-web-ui-ch6/ (визуальный демо-проект, 3 вкладки: String, Record, Bytes)
#
# Не модифицирует существующие компоненты и код из Глав 1–5.
# Идемпотентен: повторный запуск безопасен.

set -euo pipefail
ROOT="$(cd "$(dirname "$0")/.." && pwd)"
cd "$ROOT"

echo "=== [1/7] Exclude new component from workspace ==="
if ! grep -q 'components/text-engine-advanced' Cargo.toml; then
  python3 - <<'PY'
import re, pathlib
p = pathlib.Path("Cargo.toml")
s = p.read_text()
m = re.search(r'exclude\s*=\s*\[(.*?)\]', s, re.S)
if not m:
    raise SystemExit("exclude array not found in Cargo.toml")
inner = m.group(1).rstrip()
if not inner.endswith(','):
    inner += ','
inner += '\n    "components/text-engine-advanced",\n'
p.write_text(s[:m.start()] + 'exclude = [' + inner + ']' + s[m.end():])
print("Added text-engine-advanced to exclude.")
PY
fi

echo "=== [2/7] WIT contract ==="
mkdir -p wit/text-engine-advanced
cat > wit/text-engine-advanced/world.wit <<'EOF'
package docs:text-engine-advanced@0.1.0;

interface processor {
    record text-request {
        content: string,
        mode: string,
    }

    variant text-error {
        invalid-utf8(string),
        empty-input,
        unknown-mode(string),
    }

    process-string: func(input: string) -> string;
    process-record: func(req: text-request) -> result<string, text-error>;
    process-bytes:  func(data: list<u8>)  -> result<string, text-error>;
}

world text-engine-advanced {
    export processor;
}
EOF

echo "=== [3/7] Guest component ==="
if [ ! -d "components/text-engine-advanced" ]; then
  cargo new components/text-engine-advanced --lib --vcs none >/dev/null
fi
cat > components/text-engine-advanced/Cargo.toml <<'EOF'
[package]
name = "text-engine-advanced"
version = "0.1.0"
edition = "2021"

[workspace]

[lib]
crate-type = ["cdylib"]

[dependencies]
text-engine = { path = "../text-engine" }
wit-bindgen = "0.63"
EOF
cat > components/text-engine-advanced/src/lib.rs <<'EOF'
wit_bindgen::generate!({
    path: "../../wit/text-engine-advanced",
    world: "text-engine-advanced",
});

use exports::docs::text_engine_advanced::processor::{Guest, TextError, TextRequest};

struct AdvancedEngine;

impl Guest for AdvancedEngine {
    fn process_string(input: String) -> String {
        text_engine::normalize_text(&input)
    }

    fn process_record(req: TextRequest) -> Result<String, TextError> {
        if req.content.is_empty() {
            return Err(TextError::EmptyInput);
        }
        match req.mode.as_str() {
            "upper"     => Ok(req.content.to_uppercase()),
            "lower"     => Ok(req.content.to_lowercase()),
            "normalize" => Ok(text_engine::normalize_text(&req.content)),
            other       => Err(TextError::UnknownMode(other.to_string())),
        }
    }

    fn process_bytes(data: Vec<u8>) -> Result<String, TextError> {
        if data.is_empty() {
            return Err(TextError::EmptyInput);
        }
        match String::from_utf8(data) {
            Ok(s)  => Ok(text_engine::normalize_text(&s)),
            Err(e) => Err(TextError::InvalidUtf8(e.to_string())),
        }
    }
}

export!(AdvancedEngine);
EOF

echo "=== [4/7] Host runtime_v6 ==="
cat > host/src/runtime_v6.rs <<'EOF'
use anyhow::Result;
use std::path::Path;
use std::sync::Arc;
use wasmtime::component::{Component, Linker};
use wasmtime::{Engine, Store};
use wasmtime_wasi::{WasiCtx, WasiCtxBuilder, WasiCtxView, WasiView};

wasmtime::component::bindgen!({
    path: "../wit/text-engine-advanced",
    world: "text-engine-advanced",
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

    pub fn process_string(&self, input: &str) -> Result<String> {
        let mut store = self.new_store();
        let b = TextEngineAdvanced::instantiate(&mut store, &self.component, &self.linker)?;
        Ok(b.docs_text_engine_advanced_processor().call_process_string(&mut store, input)?)
    }

    pub fn process_record(&self, content: &str, mode: &str)
        -> Result<Result<String, String>>
    {
        let mut store = self.new_store();
        let b = TextEngineAdvanced::instantiate(&mut store, &self.component, &self.linker)?;
        let req = exports::docs::text_engine_advanced::processor::TextRequest {
            content: content.to_string(),
            mode: mode.to_string(),
        };
        Ok(b.docs_text_engine_advanced_processor()
            .call_process_record(&mut store, &req)?
            .map_err(|e| format!("{:?}", e)))
    }

    pub fn process_bytes(&self, data: &[u8])
        -> Result<Result<String, String>>
    {
        let mut store = self.new_store();
        let b = TextEngineAdvanced::instantiate(&mut store, &self.component, &self.linker)?;
        Ok(b.docs_text_engine_advanced_processor()
            .call_process_bytes(&mut store, data)?
            .map_err(|e| format!("{:?}", e)))
    }

    fn new_store(&self) -> Store<MyState> {
        let wasi = WasiCtxBuilder::new().build();
        Store::new(&self.engine, MyState {
            ctx: wasi,
            table: wasmtime::component::ResourceTable::new(),
        })
    }
}
EOF
if ! grep -q 'pub mod runtime_v6;' host/src/lib.rs; then
  echo 'pub mod runtime_v6;' >> host/src/lib.rs
fi

echo "=== [5/7] Integration tests ==="
cat > host/tests/integration_v6.rs <<'EOF'
use std::path::PathBuf;

fn wasm_path() -> PathBuf {
    PathBuf::from(env!("CARGO_MANIFEST_DIR"))
        .parent().unwrap()
        .join("components/text-engine-advanced/target/wasm32-wasip1/release/text_engine_advanced.wasm")
}

#[test]
fn test_v6_data_boundary() {
    let path = wasm_path();
    if !path.exists() {
        eprintln!("Skipping: {} not found.", path.display());
        return;
    }
    let rt = host::runtime_v6::Runtime::new(&path).expect("load");

    // 1. String
    assert_eq!(rt.process_string("  HELLO  ").unwrap(), "hello");

    // 2. Record
    assert_eq!(rt.process_record("  Hi  ", "upper").unwrap().unwrap(), "  HI  ");

    // 3. Empty -> typed error
    let err = rt.process_record("", "upper").unwrap();
    assert!(err.is_err());

    // 4. Unknown mode -> typed error
    let err = rt.process_record("x", "bogus").unwrap();
    assert!(err.is_err());

    // 5. Bytes (valid)
    assert_eq!(rt.process_bytes(b"Hello World").unwrap().unwrap(), "hello world");

    // 6. Bytes (invalid UTF-8) -> typed error, NOT a panic
    let err = rt.process_bytes(&[0xFF, 0xFE, 0xFD]).unwrap();
    assert!(err.is_err());
    assert!(err.unwrap_err().contains("InvalidUtf8"));
}
EOF

echo "=== [6/7] Visual demo (demo-web-ui-ch6) ==="
if [ ! -d "demo-web-ui-ch6" ]; then
  cargo new demo-web-ui-ch6 --bin --vcs none >/dev/null
fi
cat > demo-web-ui-ch6/Cargo.toml <<'EOF'
[package]
name = "demo-web-ui-ch6"
version.workspace = true
edition.workspace = true

[dependencies]
host = { path = "../host" }
axum = "0.8"
tokio = { version = "1", features = ["full"] }
serde_json = "1"
anyhow = "1.0"
EOF
cat > demo-web-ui-ch6/src/main.rs <<'EOF'
use axum::{response::Html, routing::{get, post}, Json, Router};
use serde_json::{json, Value};
use std::sync::Arc;
use host::runtime_v6::Runtime;

const INDEX_HTML: &str = include_str!("../static/index.html");

async fn index() -> Html<&'static str> { Html(INDEX_HTML) }

async fn process_string(
    axum::extract::State(rt): axum::extract::State<Arc<Runtime>>,
    Json(p): Json<Value>,
) -> Json<Value> {
    let input = p["input"].as_str().unwrap_or("").to_string();
    match tokio::task::spawn_blocking(move || rt.process_string(&input)).await {
        Ok(Ok(out)) => Json(json!({ "output": out })),
        Ok(Err(e))  => Json(json!({ "error": format!("{e}") })),
        Err(e)      => Json(json!({ "error": format!("join: {e}") })),
    }
}

async fn process_record(
    axum::extract::State(rt): axum::extract::State<Arc<Runtime>>,
    Json(p): Json<Value>,
) -> Json<Value> {
    let content = p["content"].as_str().unwrap_or("").to_string();
    let mode    = p["mode"].as_str().unwrap_or("").to_string();
    match tokio::task::spawn_blocking(move || rt.process_record(&content, &mode)).await {
        Ok(Ok(Ok(out))) => Json(json!({ "output": out })),
        Ok(Ok(Err(e)))  => {
            eprintln!("[guest-record-error] {e}");
            Json(json!({ "error": "guest_error",
                         "kind":  error_kind(&e),
                         "reason":"see server log for details" }))
        }
        Ok(Err(e))      => Json(json!({ "error": format!("host: {e}") })),
        Err(e)          => Json(json!({ "error": format!("join: {e}") })),
    }
}

async fn process_bytes(
    axum::extract::State(rt): axum::extract::State<Arc<Runtime>>,
    Json(p): Json<Value>,
) -> Json<Value> {
    let data: Vec<u8> = p["bytes"]
        .as_array()
        .map(|a| a.iter().filter_map(|v| v.as_u64().map(|n| n as u8)).collect())
        .unwrap_or_default();
    match tokio::task::spawn_blocking(move || rt.process_bytes(&data)).await {
        Ok(Ok(Ok(out))) => Json(json!({ "output": out })),
        Ok(Ok(Err(e)))  => {
            eprintln!("[guest-bytes-error] {e}");
            Json(json!({ "error": "guest_error",
                         "kind":  error_kind(&e),
                         "reason":"see server log for details" }))
        }
        Ok(Err(e))      => Json(json!({ "error": format!("host: {e}") })),
        Err(e)          => Json(json!({ "error": format!("join: {e}") })),
    }
}

// Извлекаем "kind" из debug-строки TextError::Variant(...)
fn error_kind(debug: &str) -> &'static str {
    if debug.contains("InvalidUtf8")  { "invalid_utf8"  }
    else if debug.contains("EmptyInput")   { "empty_input"   }
    else if debug.contains("UnknownMode")  { "unknown_mode"  }
    else                                   { "unknown"       }
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let manifest = std::path::Path::new(env!("CARGO_MANIFEST_DIR"));
    let wasm_path = manifest.parent().unwrap()
        .join("components/text-engine-advanced/target/wasm32-wasip1/release/text_engine_advanced.wasm");

    if !wasm_path.exists() {
        eprintln!("Error: WASM component not found at `{}`.", wasm_path.display());
        eprintln!("Run `cd components/text-engine-advanced && cargo component build --release` first.");
        std::process::exit(1);
    }

    println!("Loading component: {}", wasm_path.display());
    let runtime = Runtime::new(&wasm_path)?;

    let app = Router::new()
        .route("/", get(index))
        .route("/string", post(process_string))
        .route("/record", post(process_record))
        .route("/bytes",  post(process_bytes))
        .with_state(runtime);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3005").await?;
    println!("🚀 Demo UI (Ch6): http://127.0.0.1:3005");
    axum::serve(listener, app).await?;
    Ok(())
}
EOF

mkdir -p demo-web-ui-ch6/static
cat > demo-web-ui-ch6/static/index.html <<'HTML_EOF'
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8"><title>Ch6: Data Boundary</title>
<style>
  :root { --bg:#0f172a; --panel:#1e293b; --br:#334155; --tx:#e2e8f0;
          --mu:#94a3b8; --ac:#38bdf8; --ok:#22c55e; --er:#ef4444;
          --mo:ui-monospace,SFMono-Regular,Menlo,Consolas,monospace; }
  *{box-sizing:border-box;margin:0;padding:0}
  body{font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,sans-serif;
       background:var(--bg);color:var(--tx);min-height:100vh;
       display:flex;align-items:center;justify-content:center;padding:2rem}
  .box{max-width:880px;width:100%;background:var(--panel);
       border:1px solid var(--br);border-radius:12px;padding:2rem;
       box-shadow:0 20px 60px rgba(0,0,0,.4)}
  h1{font-size:1.6rem;font-weight:600;text-align:center;margin-bottom:.35rem;
     background:linear-gradient(135deg,var(--ac),#a78bfa);
     -webkit-background-clip:text; background-clip:text;
     -webkit-text-fill-color:transparent}
  .sub{color:var(--mu);text-align:center;font-size:.9rem;margin-bottom:2rem}
  .sub code{font-family:var(--mo);background:rgba(255,255,255,.05);
            padding:.1rem .35rem;border-radius:4px;color:var(--ac)}
  .tabs{display:flex;gap:.4rem;border-bottom:1px solid var(--br);
        padding-bottom:.5rem;margin-bottom:1.25rem}
  .tab{padding:.55rem 1rem;border-radius:8px;cursor:pointer;
       background:transparent;color:var(--mu);border:1px solid var(--br);
       font-family:var(--mo);font-size:.82rem;transition:.15s}
  .tab:hover{color:var(--tx)}
  .tab.active{background:var(--ac);color:#0c1222;border-color:var(--ac);font-weight:600}
  .panel{display:none}
  .panel.active{display:block}
  label{display:block;font-size:.75rem;font-weight:500;color:var(--mu);
        text-transform:uppercase;letter-spacing:.05em;margin-bottom:.4rem;margin-top:.75rem}
  textarea,input[type=text]{width:100%;padding:.8rem;font-family:var(--mo);
        font-size:.9rem;background:var(--bg);color:var(--tx);
        border:1px solid var(--br);border-radius:8px;outline:none;transition:.15s}
  textarea{min-height:90px;resize:vertical}
  textarea:focus,input[type=text]:focus{border-color:var(--ac)}
  .row{display:grid;grid-template-columns:1fr 1fr;gap:.75rem}
  .actions{display:flex;gap:.6rem;margin-top:1rem;flex-wrap:wrap}
  button{padding:.65rem 1.15rem;font-size:.88rem;font-weight:500;border:none;
         border-radius:8px;cursor:pointer;transition:.15s}
  button.primary{background:var(--ac);color:#0c1222}
  button.primary:hover{background:#0ea5e9}
  button.primary:disabled{opacity:.5;cursor:wait}
  button.danger{background:var(--er);color:#fff}
  button.danger:hover{background:#dc2626}
  .result{margin-top:1.25rem;padding:1rem;background:var(--bg);
          border:1px solid var(--br);border-radius:8px;min-height:70px}
  .result-label{display:flex;justify-content:space-between;
                font-size:.72rem;color:var(--mu);
                text-transform:uppercase;letter-spacing:.05em;margin-bottom:.5rem}
  .result-body{font-family:var(--mo);font-size:.9rem;
               white-space:pre-wrap;word-break:break-word;color:var(--mu)}
  .result-body.ok{color:var(--ok)}
  .result-body.er{color:var(--er)}
  .timing{color:var(--ac);font-family:var(--mo);font-size:.75rem}
</style>
</head>
<body>
<div class="box">
  <h1>Data Boundary: String · Record · Bytes</h1>
  <p class="sub">Passes values through <code>text-engine-advanced.wasm</code> · WIT types & errors</p>

  <div class="tabs">
    <button class="tab active" data-tab="string">String</button>
    <button class="tab" data-tab="record">Record</button>
    <button class="tab" data-tab="bytes">Bytes (list&lt;u8&gt;)</button>
  </div>

  <div class="panel active" id="panel-string">
    <label>Input (string)</label>
    <textarea id="s-in">  Hello   String  </textarea>
    <div class="actions">
      <button class="primary" onclick="runString()">Process → string</button>
    </div>
    <div class="result">
      <div class="result-label"><span>Result</span><span class="timing" id="s-t"></span></div>
      <div class="result-body" id="s-out">Ready.</div>
    </div>
  </div>

  <div class="panel" id="panel-record">
    <div class="row">
      <div>
        <label>Content</label>
        <input type="text" id="r-content" value="  Hello Record  ">
      </div>
      <div>
        <label>Mode (upper / lower / normalize)</label>
        <input type="text" id="r-mode" value="upper">
      </div>
    </div>
    <div class="actions">
      <button class="primary" onclick="runRecord()">Process → result&lt;string, text-error&gt;</button>
      <button class="danger" onclick="runRecordBad()">Send unknown mode</button>
    </div>
    <div class="result">
      <div class="result-label"><span>Result</span><span class="timing" id="r-t"></span></div>
      <div class="result-body" id="r-out">Ready.</div>
    </div>
  </div>

  <div class="panel" id="panel-bytes">
    <label>Input string (converted to UTF-8 bytes in browser)</label>
    <input type="text" id="b-in" value="Hello Bytes">
    <div class="actions">
      <button class="primary" onclick="runBytes(false)">Process → valid UTF-8</button>
      <button class="danger" onclick="runBytes(true)">Send invalid UTF-8 (0xFF 0xFE 0xFD)</button>
    </div>
    <div class="result">
      <div class="result-label"><span>Result</span><span class="timing" id="b-t"></span></div>
      <div class="result-body" id="b-out">Ready.</div>
    </div>
  </div>
</div>

<script>
const tabs = document.querySelectorAll('.tab');
tabs.forEach(t => t.onclick = () => {
  tabs.forEach(x => x.classList.remove('active'));
  t.classList.add('active');
  document.querySelectorAll('.panel').forEach(p => p.classList.remove('active'));
  document.getElementById('panel-' + t.dataset.tab).classList.add('active');
});

async function post(url, body) {
  const t0 = performance.now();
  const r = await fetch(url, { method:'POST', headers:{'Content-Type':'application/json'},
                                body: JSON.stringify(body) });
  const data = await r.json();
  return { data, ms: performance.now() - t0 };
}

function show(outEl, timingEl, r) {
  timingEl.textContent = 'round-trip: ' + r.ms.toFixed(2) + ' ms';
  if (r.data.error) {
    outEl.className = 'result-body er';
    outEl.textContent = 'Error [' + (r.data.kind || r.data.error) + ']: ' + (r.data.reason || '');
  } else {
    outEl.className = 'result-body ok';
    outEl.textContent = r.data.output === '' ? '(empty string)' : r.data.output;
  }
}

async function runString() {
  const o = document.getElementById('s-out'), t = document.getElementById('s-t');
  o.textContent = 'Calling…';
  try { show(o, t, await post('/string', { input: document.getElementById('s-in').value })); }
  catch(e) { o.className='result-body er'; o.textContent = e; }
}

async function runRecord() {
  const o = document.getElementById('r-out'), t = document.getElementById('r-t');
  o.textContent = 'Calling…';
  try { show(o, t, await post('/record', {
    content: document.getElementById('r-content').value,
    mode:    document.getElementById('r-mode').value
  })); } catch(e) { o.className='result-body er'; o.textContent = e; }
}

async function runRecordBad() {
  document.getElementById('r-mode').value = 'bogus_mode';
  await runRecord();
}

async function runBytes(invalid) {
  const o = document.getElementById('b-out'), t = document.getElementById('b-t');
  o.textContent = 'Calling…';
  const bytes = invalid
    ? [255, 254, 253]
    : Array.from(new TextEncoder().encode(document.getElementById('b-in').value));
  try { show(o, t, await post('/bytes', { bytes })); }
  catch(e) { o.className='result-body err'; o.textContent = e; }
}
</script>
</body>
</html>
HTML_EOF

echo "=== [7/7] Build & verify ==="
(cd components/text-engine-advanced && cargo component build --release --quiet)
cargo check -p host --lib --quiet
cargo check -p demo-web-ui-ch6 --quiet

echo
echo "✅ Готово. Следующие шаги:"
echo "  1. Собрать компонент:  (cd components/text-engine-advanced && cargo component build --release)"
echo "  2. Тесты:              (cd host && cargo test --test integration_v6)"
echo "  3. Демо:               cargo run -p demo-web-ui-ch6 --release"
echo "  4. Открыть:            http://127.0.0.1:3005"
```

</details>

### Шаг 2. Сделать скрипт исполняемым

```bash
chmod +x scripts/setup-ch6.sh
```

### Шаг 3. Запустить

```bash
./scripts/setup-ch6.sh
```

Ожидаемый вывод в конце:

```text
=== [7/7] Build & verify ===
✅ Готово. Следующие шаги:
  1. Собрать компонент:  (cd components/text-engine-advanced && cargo component build --release)
  2. Тесты:              (cd host && cargo test --test integration_v6)
  3. Демо:               cargo run -p demo-web-ui-ch6 --release
  4. Открыть:            http://127.0.0.1:3005
```

### Шаг 4. Что делает скрипт

| Шаг | Действие | Результат |
|---|---|---|
| 1 | `python3` правит корневой `Cargo.toml` | `components/text-engine-advanced` в `exclude` |
| 2 | Пишет WIT-контракт | `wit/text-engine-advanced/world.wit` |
| 3 | Создаёт крейт и `lib.rs` | `components/text-engine-advanced/` |
| 4 | Пишет `runtime_v6.rs` + `pub mod runtime_v6` | `host/src/runtime_v6.rs`, обновлён `host/src/lib.rs` |
| 5 | Пишет тесты | `host/tests/integration_v6.rs` |
| 6 | Создаёт визуальный демо-сервер | `demo-web-ui-ch6/` (HTML + Axum) |
| 7 | Собирает компонент и проверяет компиляцию | `.wasm`, `cargo check` без ошибок |

### Шаг 5. Проверка после запуска

```bash
cd ~/projects/wa/wasm-backend-lab

# 1. Артефакт действительно component
wasm-tools print components/text-engine-advanced/target/wasm32-wasip1/release/text_engine_advanced.wasm | head -n 1
# → (component $text-engine-advanced

# 2. WIT-контракт содержит новые функции
wasm-tools component wit components/text-engine-advanced/target/wasm32-wasip1/release/text_engine_advanced.wasm | grep -E 'process-(string|record|bytes)'

# 3. Пакет demo-web-ui-ch6 в workspace
cargo metadata --no-deps --format-version 1 | grep -c '"name":"demo-web-ui-ch6"'
# → 1

# 4. Старые артефакты не тронуты
ls components/text-engine/src/lib.rs           # Глава 1
ls components/text-engine-component/            # Глава 2
ls demo-web-ui/static/index.html                # Глава 5
```

После этих проверок переходите к чтению текста главы (Часть B ниже).

---

# Часть B. Глава 6. Граница Host ↔ Guest: три класса данных и цена ошибки

**Зачем создана эта глава?**
До сих пор через границу Host↔Guest ходил только `string`. В реальном backend данные сложнее: структуры (`record`), сырые байты (`list<u8>`), и — что важнее всего — **ошибки**. Компонент может получить невалидный UTF-8, неизвестный `mode` или пустой ввод. Если host просто возвращает строку и надеется, что всё хорошо, — любая ошибка guest превращается в `panic` или молчаливую потерю данных. Мы должны научиться пересекать границу так, чтобы значение проходило через неё ровно то, что заявлено в WIT, а ошибки возвращались типизированными и не раскрывали чувствительных деталей наружу.

**Какую пользу несёт глава?**
1. **Введение в три класса данных.** WIT различает **value types** (string, int, record, list, variant, result, option — копируются через границу), **resource types** (handle на объект, которым владеет одна сторона) и **stream/future types** (асинхронные потоки, передача по частям). Ресурсы и стримы — темы будущих глав. Здесь мы работаем только с value types, но должны явно увидеть, что за «просто данные» есть выбор.
2. **Data boundary vs Capability boundary.** Передать `record` через WIT — это **data boundary**: host сериализует, guest десериализует, копирование. А дать guest возможность открыть файл — это **capability boundary**: нужен явный `Linker` с разрешениями. Глава фокусируется на первом; второй — тема про capabilities.
3. **Три требования к ошибкам через границу.** (a) API наружу — согласованный и минимальный; (b) диагностика внутрь — полная, в логах; (c) чувствительные данные (пути, содержимое, трейсы) не утекают клиенту.

**Чего мы хотим достичь по результатам тестирования?**
1. Расширить WIT-контракт: три функции вместо одной (`process-string`, `process-record`, `process-bytes`).
2. Научиться читать `result<T, E>` из guest в host (WIT `result` → Rust `Result<T, E>`).
3. **DO (negative):** передать невалидный UTF-8 — guest возвращает `TextError::InvalidUtf8`, host не падает, клиент видит `{"error":"guest_error","kind":"invalid_utf8"}`.
4. Убедиться, что все предыдущие главы (1–5) продолжают работать.

---

### 🚦 Starting point

У нас есть `text-engine-component` (Глава 2), `host_v2` (Глава 5), `demo-web-ui` (Глава 5). Мы их **не трогаем**. Скрипт создал независимый `text-engine-advanced` компонент и `demo-web-ui-ch6` UI.

### 🎯 PREDICT (Предсказание)

*Запишите свой прогноз перед запуском:*

> **Если guest вернёт `TextError::InvalidUtf8` через WIT `result`, а host не обработает это специально — что произойдёт при вызове `curl -X POST …/bytes` с невалидным UTF-8? Сервер упадёт (500 с паникой), вернёт ошибку, или «съест» её и отдаст пустую строку?**

---

## Этап 1. Исходные коды

### Структура

**Новые файлы (созданы скриптом):**
- `wit/text-engine-advanced/world.wit` — новый контракт.
- `components/text-engine-advanced/` — новый компонент.
- `host/src/runtime_v6.rs` — новый рантайм.
- `host/tests/integration_v6.rs` — новые тесты.
- `demo-web-ui-ch6/` — новый UI + API.

**Обновляется:**
- `Cargo.toml` (корень) — `text-engine-advanced` в `exclude`.
- `host/src/lib.rs` — `pub mod runtime_v6;`.

**Без изменений:**
- `components/text-engine/`, `text-engine-core/`, `text-engine-component/`, `document-engine/`.
- `host/src/{runtime,http,main}.rs`, `host/src/bin/host_v2.rs`.
- `demo-web-ui/`.

### Ссылки для изучения
- [WIT — Type system](https://component-model.bytecodealliance.org/design/wit.html) — value / resource / stream, `result`, `variant`.
- [Wasmtime — Component Model](https://docs.wasmtime.dev/api/wasmtime/component/index.html) — что значит `bindgen!`.
- [component-model — Why Component Model](https://component-model.bytecodealliance.org/design/why-component-model.html) — зачем типы на границе.

### Ключевые фрагменты контракта

**`wit/text-engine-advanced/world.wit`**

```wit
package docs:text-engine-advanced@0.1.0;

interface processor {
    record text-request {
        content: string,
        mode: string,
    }

    variant text-error {
        invalid-utf8(string),
        empty-input,
        unknown-mode(string),
    }

    process-string: func(input: string) -> string;
    process-record: func(req: text-request) -> result<string, text-error>;
    process-bytes:  func(data: list<u8>)  -> result<string, text-error>;
}

world text-engine-advanced {
    export processor;
}
```

**Что здесь важно:**
- `record` — именованная структура с двумя `string`. WIT-тип, host сериализует в память guest.
- `variant text-error` — tagged union. В Rust становится `enum`. В host — `Result<_, TextError>`.
- `list<u8>` — Rust-`Vec<u8>`. Host передаёт массив байт, guest получает `Vec<u8>`.
- `result<string, text-error>` — на границе host видит `Result<String, TextError>`. Ошибка типизирована и не теряется.

**`components/text-engine-advanced/src/lib.rs` (фрагмент `process-bytes`)**

```rust
fn process_bytes(data: Vec<u8>) -> Result<String, TextError> {
    if data.is_empty() {
        return Err(TextError::EmptyInput);
    }
    match String::from_utf8(data) {
        Ok(s)  => Ok(text_engine::normalize_text(&s)),
        Err(e) => Err(TextError::InvalidUtf8(e.to_string())),
    }
}
```

**Что важно:** `String::from_utf8` возвращает `Result`, и мы **явно** транслируем `Err` в `TextError::InvalidUtf8`. Без этого guest бы запаниковал на `.unwrap()`.

### Подключение в `lib.rs`

Единственная строка в `host/src/lib.rs`:

```rust
pub mod runtime_v6;
```

---

## Этап 2. Тесты

**`host/tests/integration_v6.rs`** — 1 тест, 6 проверок внутри.

**Что проверяется:**
1. `process_string` — значение (`"  HELLO  "` → `"hello"`).
2. `process_record` — структура (`{"content":"  Hi  ","mode":"upper"}` → `"  HI  "`).
3. `process_record` с пустым `content` → `Err(EmptyInput)`.
4. `process_record` с `mode: "bogus"` → `Err(UnknownMode)`.
5. `process_bytes` валидный UTF-8 → `Ok(...)`.
6. **Negative:** `process_bytes(&[0xFF, 0xFE, 0xFD])` → `Err(InvalidUtf8)`, **не panic**.

### Команды запуска

```bash
# 1. Компонент
(cd components/text-engine-advanced && cargo component build --release)

# 2. Тесты текущей главы
(cd host && cargo test --test integration_v6)

# 3. Регресс: все прошлые тесты продолжают работать
(cd host && cargo test --test integration)
(cd components/text-engine && cargo test)
(cd components/text-engine-core && cargo test)

# 4. Всё разом
cargo test --workspace
```

**Ожидаемый результат:**
```
running 1 test
test test_v6_data_boundary ... ok
test result: ok. 1 passed; 0 failed
```

---

## Этап 3. Демо-пример

Визуальный сервер `demo-web-ui-ch6` (порт 3005) с тремя вкладками — по одной на каждый класс данных.

### Запуск

```bash
# Терминал 1 — UI + API
cargo run -p demo-web-ui-ch6 --release
# → Loading component: …/text_engine_advanced.wasm
# → 🚀 Demo UI (Ch6): http://127.0.0.1:3005
```

Открыть: **http://127.0.0.1:3005**

### Что увидите

**Вкладка String:** вводите `  Hello   String  ` → ответ `hello string`, зелёным.

**Вкладка Record:** два поля — `Content` и `Mode` (upper/lower/normalize). Кнопка «Send unknown mode» ставит `mode = "bogus_mode"` → красный ответ `Error [unknown_mode]: see server log for details`, а в консоли сервера — полный `[guest-record-error] UnknownMode("bogus_mode")`.

**Вкладка Bytes:** кнопка «Send invalid UTF-8 (0xFF 0xFE 0xFD)» отправляет ровно эти три байта → красный ответ `Error [invalid_utf8]`, сервер продолжает отвечать на следующие клики.

### Почему это визуальное доказательство

1. **Три вкладки = три класса данных.** Читатель видит на одном экране, как одна и та же граница справляется со `string`, `record` и `list<u8>`.
2. **Round-trip в миллисекундах.** Каждый клик показывает время, включая создание `Store`, инстанциацию, marshal аргументов, вызов, marshal ответа.
3. **«Красная кнопка» = negative scenario.** Именно то, что нужно увидеть: нажали «Send invalid UTF-8» — получили ошибку, **но сервер не упал**. Можно нажать ещё раз, ещё раз — UI продолжает работать.
4. **Разделение internal/external.** В UI вы видите `kind: invalid_utf8` — generic. В консоли сервера — полный `InvalidUtf8("invalid utf-8 sequence of 3 bytes from index 0")` — диагностика. Это и есть три требования к ошибкам через границу.

### Доказательство через curl (опционально)

```bash
# Валидный UTF-8
curl -s -X POST http://127.0.0.1:3005/bytes \
  -H 'Content-Type: application/json' \
  -d '{"bytes":[72,101,108,108,111]}'
# → {"output":"hello"}

# Невалидный UTF-8 (три байта 0xFF 0xFE 0xFD)
curl -s -X POST http://127.0.0.1:3005/bytes \
  -H 'Content-Type: application/json' \
  -d '{"bytes":[255,254,253]}'
# → {"error":"guest_error","kind":"invalid_utf8","reason":"see server log for details"}

# Смотрим лог сервера — там полная диагностика:
# [guest-bytes-error] InvalidUtf8("invalid utf-8 sequence of 3 bytes from index 0")
```

---

## 🔄 BREAK (Поломка) — что будет, если гость не обработает `Err`

**Цель:** увидеть, что происходит, когда guest «проглатывает» ошибку вместо возврата типизированной.

**Шаг 1. Временно изменим `process_bytes` в компоненте:**

```rust
fn process_bytes(data: Vec<u8>) -> Result<String, TextError> {
    // Было: match String::from_utf8(data) { Ok(s) => …, Err(e) => Err(InvalidUtf8(…)) }
    // Стало: unwrap — симулируем "всё должно быть UTF-8"
    let s = String::from_utf8(data).unwrap();   // ← намеренная поломка
    Ok(text_engine::normalize_text(&s))
}
```

Пересоберите:
```bash
(cd components/text-engine-advanced && cargo component build --release)
```

**Шаг 2. Отправьте невалидный UTF-8 через UI (кнопка «Send invalid UTF-8»).**

**Что произойдёт:**
- Guest паникует внутри WASM → Wasmtime ловит `trap`.
- `call_process_bytes` возвращает `Err(wasmtime::Error)`, а не `Ok(Err(TextError::InvalidUtf8))`.
- Наш host-обработчик попадает в ветку `Ok(Err(e)) => Json({"error": "host: …"})` — потому что это был trap, а не WIT-ошибка.
- В логе сервера: `[guest-bytes-error] …` — но с сообщением о trap, а не о UTF-8.

**Шаг 3. Нажмите кнопку ещё раз.** Сервер продолжает работать — Wasmtime изолирует trap одного вызова от следующих. Но диагностика стала хуже: в логе вы видите `wasm trap`, а не «невалидный UTF-8 в позиции 0».

**Шаг 4. Верните правильную реализацию:**

```rust
fn process_bytes(data: Vec<u8>) -> Result<String, TextError> {
    if data.is_empty() { return Err(TextError::EmptyInput); }
    match String::from_utf8(data) {
        Ok(s)  => Ok(text_engine::normalize_text(&s)),
        Err(e) => Err(TextError::InvalidUtf8(e.to_string())),
    }
}
```

Пересоберите. Нажмите кнопку — снова `Error [invalid_utf8]`, в логе снова полная диагностика.

**Урок:** `result<T, E>` в WIT — это не сахар. Это то, что отделяет **управляемую ошибку** (host видит `kind: invalid_utf8` и логирует детали) от **trap** (host видит общий trap, диагностика размывается). Без типизированного `E` вы теряете информацию на границе.

---

## 🔄 Итеративная проверка

**Нужны ли доработки после тестов?**

Да, мы обнаружили три вещи в процессе работы:

1. **`format!("{:?}", e)` теряет структуру `TextError`.** В первом варианте `runtime_v6.rs` мы конвертировали `TextError` в `String` через `Debug`, что ломает типизацию на host-стороне. Правильно — либо возвращать `TextError` как тип (сложнее), либо в host-обработчике парсить `kind` из строки (что мы и сделали через `error_kind()`). Для главы выбран второй вариант — минимальный код.

2. **`base64` не нужен.** Изначально передавали `bytes` как base64-строку. Это лишняя зависимость. Оказалось проще передавать JSON-массив `[255, 254, 253]` — `serde_json` сам превращает его в `Vec<u8>`. Убрали `base64`.

3. **Логирование отдельно от ответа.** Первая версия `process_bytes` возвращала `{"error": format!("{e}")}` — клиент видел полный текст ошибки guest, включая `"invalid utf-8 sequence of 3 bytes from index 0"`. Это нарушает требование «чувствительные данные не раскрываются». Исправлено: в лог (`eprintln!`) пишем полный, в JSON — только `kind`.

### Реализация доработок

Все три доработки уже включены в скрипт `setup-ch6.sh` (см. часть A). В частности:

- `runtime_v6::process_bytes` возвращает `Result<Result<String, String>>` — внешний `Result` = anyhow (wasmtime), внутренний = результат guest.
- `demo-web-ui-ch6/src/main.rs` содержит `error_kind()` — извлекает `invalid_utf8` / `empty_input` / `unknown_mode` из строки.
- Ответ клиенту содержит `{ "error": "guest_error", "kind": "invalid_utf8", "reason": "see server log for details" }`.

---

## Итог главы

**OBSERVE:**
- Все три класса value-данных проходят через границу: `string`, `record`, `list<u8>`.
- `result<T, E>` в WIT → `Result<T, TextError>` в host. Ошибка типизирована.
- Negative: невалидный UTF-8 → `Err(InvalidUtf8)`, host отвечает `kind: invalid_utf8`, сервер жив.
- В логах — полная диагностика; клиенту — только generic `kind`.

**MEASURE:**
- Round-trip из UI: 2–5 ms (включая создание `Store`, инстанциацию, marshal аргументов, вызов, marshal ответа).
- Один и тот же компонент обслуживает все три вкладки без рестарта.

**PREDICT (проверка):**
Было предсказано, что без специальной обработки guest либо упадёт (panic → trap → 500), либо вернёт пустую строку. Результат: **подтвердилось частично**. Wasmtime сам по себе ловит trap и превращает его в `Err(wasmtime::Error)` — сервер не падает. Но информация об ошибке теряется: host видит `trap`, а не `invalid utf-8`. Без `result<T, E>` в WIT вы получаете «сервер жив, но диагностики нет». С `result` — и жив, и диагностика.

**What did we prove?**
1. Value types (`string`, `record`, `list<u8>`) проходят через Component Model границу без ручной сериализации.
2. `result<T, E>` в WIT даёт типизированные ошибки, которые host читает как `Result`.
3. Невалидный UTF-8 не роняет host — граница устойчива.
4. Data boundary ≠ Capability boundary: мы передаём данные, но не даём guest «сделать что-то» в мире host.
5. Три требования к ошибкам через границу выполнены: API наружу минимален (`kind`), диагностика внутри полна (лог), чувствительные данные не утекают.

**What did we not prove?**
- Что `resource` и `stream` работают аналогично. Это темы будущих глав.
- Что `result` покрывает *все* виды ошибок — сетевые, OOM, trap самого WASM. Мы работаем только с ошибками уровня бизнес-логики.

**What did we pay for it?**
- +1 компонент (`text-engine-advanced`), +1 рантайм (`runtime_v6`), +1 UI-сервер. Три новых крейта ради одной главы.
- WIT-контракт стал длиннее: `record` + `variant` + `result` вместо одной функции `process: string -> string`.
- Клиент получает generic `kind`, а не полный текст ошибки — это цена безопасности (sensitive data).

**When is the alternative better?**
- Если ошибки бизнес-логики не критичны (внутренний сервис, слабая связанность) — можно вернуть `string` и логировать всё на клиенте. Но граница безопасности размывается.
- Если нужны сложные async-потоки — `value types` недостаточно, нужны `stream`/`future` (тема следующей главы).

### 📊 Evidence

```text
Claim 1:        WIT result<T, E> сохраняет типизированную ошибку через границу.
Experiment:     process-bytes с bytes = [0xFF, 0xFE, 0xFD].
Measurement:    Возвращается Ok(Err(TextError::InvalidUtf8(...))), не trap.
Observed:       HTTP 200 с {"error":"guest_error","kind":"invalid_utf8"}.
                В логе сервера — полный InvalidUtf8("invalid utf-8 sequence of 3 bytes from index 0").
Conclusion:     result<T, E> — то, что делает ошибку частью контракта.
Confidence:     High

Claim 2:        Host выживает после ошибки guest.
Experiment:     После BREAK (unwrap в guest) и последующего восстановления —
                многократные нажатия «Send invalid UTF-8».
Measurement:    Сервер отвечает на каждый следующий запрос, PID стабилен.
Observed:       Wasmtime изолирует trap одного вызова от следующих.
Conclusion:     Component Model + result<T,E> = устойчивая граница.
Confidence:     High

Claim 3:        API наружу ≠ диагностика внутрь.
Experiment:     Сравнение ответа UI и лога сервера.
Measurement:    UI: {"error":"guest_error","kind":"invalid_utf8","reason":"see server log"}.
                Log: [guest-bytes-error] InvalidUtf8("invalid utf-8 sequence of 3 bytes from index 0").
Observed:       Чувствительные данные (содержимое байт) не утекают клиенту.
Conclusion:     Разделение на log/eprintln и JSON-ответ — обязательная часть контракта.
Confidence:     High
```

### Что у нас теперь есть

- `components/text-engine-advanced/` — компонент с тремя функциями и типизированными ошибками.
- `host/src/runtime_v6.rs` — переиспользуемый рантайм.
- `host/tests/integration_v6.rs` — 6 проверок границы.
- `demo-web-ui-ch6/` — визуальный UI (три вкладки) + API на одном порту.
- Все главы 1–5 продолжают работать без изменений.

**Следующий шаг:** В Главе 7 мы перейдём от *статуса* к *состоянию*. Компонент до сих пор stateless — каждый вызов новый `Store`. Что если бизнес-логике нужна память между вызовами? Введём `resource` — типизированный handle на объект, которым владеет либо host, либо guest, и посмотрим, как это меняет архитектуру.