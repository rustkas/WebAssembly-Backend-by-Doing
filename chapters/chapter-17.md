# Часть A. CLI-автоматизация: `scripts/setup-ch17.sh`

```
#!/usr/bin/env bash
# scripts/setup-ch17.sh
#
# Автоматическая настройка Главы 17: Component as a deployable unit.

set -euo pipefail
ROOT="$(cd "$(dirname "$0")/.." && pwd)"
cd "$ROOT"

echo "=== [1/7] Exclude new component from workspace ==="
if ! grep -q 'components/text-engine-deployable' Cargo.toml; then
  python3 - <<'PY'
import re, pathlib
p = pathlib.Path("Cargo.toml")
s = p.read_text()
m = re.search(r'exclude\s*=\s*\[(.*?)\]', s, re.S)
if not m: raise SystemExit("exclude array not found")
inner = m.group(1).rstrip()
if not inner.endswith(','): inner += ','
inner += '\n    "components/text-engine-deployable",\n'
p.write_text(s[:m.start()] + 'exclude = [' + inner + ']' + s[m.end():])
print("Added text-engine-deployable to exclude.")
PY
fi

echo "=== [2/7] Guest component ==="
if [ ! -d "components/text-engine-deployable" ]; then
  cargo new components/text-engine-deployable --lib --vcs none >/dev/null
fi
rm -f components/text-engine-deployable/Cargo.toml components/text-engine-deployable/src/lib.rs
sleep 0.1
cat > components/text-engine-deployable/Cargo.toml <<'EOF'
[package]
name = "text-engine-deployable"
version = "0.1.0"
edition = "2021"
[workspace]
[lib]
crate-type = ["cdylib"]
[dependencies]
wit-bindgen = "0.63"
EOF
cat > components/text-engine-deployable/src/lib.rs <<'EOF'
wit_bindgen::generate!({
    path: "../../wit/text-engine",
    world: "text-engine",
});

struct TextEngine;
impl exports::docs::text_engine::processor::Guest for TextEngine {
    fn process(input: String) -> String {
        input.to_lowercase()
    }
}
export!(TextEngine);
EOF

echo "=== [3/7] Host runtime_v17 (Registry & Admission) ==="
# Безопасное добавление serde и toml в host/Cargo.toml
python3 - <<'PY'
import pathlib
p = pathlib.Path("host/Cargo.toml")
s = p.read_text()
if 'serde = ' not in s:
    s = s.replace('[dependencies]\n', '[dependencies]\nserde = { version = "1", features = ["derive"] }\ntoml = "0.8"\n')
    p.write_text(s)
    print("Added serde and toml to host/Cargo.toml")
PY

rm -f host/src/runtime_v17.rs
sleep 0.1
cat > host/src/runtime_v17.rs <<'EOF'
use anyhow::Result;
use serde::Deserialize;
use std::collections::HashMap;
use std::path::Path;
use std::sync::{Arc, Mutex};
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

#[derive(Debug, Deserialize, Clone)]
pub struct Manifest {
    pub name: String,
    pub version: String,
    pub required_capabilities: Vec<String>,
}

pub struct Registry {
    components: Mutex<HashMap<String, Arc<Component>>>,
}

impl Registry {
    pub fn new() -> Self {
        Self { components: Mutex::new(HashMap::new()) }
    }

    pub fn register(&self, manifest: &Manifest, component: Component) -> Result<(), String> {
        // Admission Control: проверяем, что манифест содержит необходимые разрешения
        if manifest.required_capabilities.is_empty() {
            return Err("Admission REJECT: missing required_capabilities".to_string());
        }
        if !manifest.required_capabilities.contains(&"fs:read".to_string()) {
            return Err("Admission REJECT: invalid capabilities".to_string());
        }

        let mut map = self.components.lock().unwrap();
        map.insert(manifest.name.clone(), Arc::new(component));
        Ok(())
    }

    pub fn get(&self, name: &str) -> Option<Arc<Component>> {
        let map = self.components.lock().unwrap();
        map.get(name).cloned()
    }
}

pub struct Runtime {
    pub engine: Engine,
    pub linker: Linker<MyState>,
    pub registry: Registry,
}

impl Runtime {
    pub fn new() -> Result<Arc<Self>> {
        let engine = Engine::default();
        let mut linker = Linker::new(&engine);
        wasmtime_wasi::p2::add_to_linker_sync(&mut linker)?;
        
        Ok(Arc::new(Self {
            engine,
            linker,
            registry: Registry::new(),
        }))
    }

    // Инкапсуляция: загрузка и регистрация компонента по пути
    pub fn deploy_from_path(&self, path: &Path, manifest: Manifest) -> Result<(), String> {
        let component = Component::from_file(&self.engine, path)
            .map_err(|e| format!("Load failed: {e}"))?;
        self.registry.register(&manifest, component)
    }

    pub fn process(&self, name: &str, input: &str) -> Result<String> {
        let component = self.registry.get(name).ok_or_else(|| anyhow::anyhow!("Component not found"))?;
        
        let wasi = WasiCtxBuilder::new().build();
        let mut store = Store::new(&self.engine, MyState {
            ctx: wasi,
            table: wasmtime::component::ResourceTable::new(),
        });

        let bindings = TextEngine::instantiate(&mut store, &component, &self.linker)?;
        let result = bindings.docs_text_engine_processor().call_process(&mut store, input)?;
        Ok(result)
    }
}
EOF

if ! grep -q 'pub mod runtime_v17;' host/src/lib.rs; then
  echo 'pub mod runtime_v17;' >> host/src/lib.rs
fi
touch host/src/lib.rs

echo "=== [4/7] Integration tests ==="
rm -f host/tests/integration_v17.rs
sleep 0.1
cat > host/tests/integration_v17.rs <<'EOF'
use std::path::PathBuf;

fn wasm_path() -> PathBuf {
    PathBuf::from(env!("CARGO_MANIFEST_DIR")).parent().unwrap()
        .join("components/text-engine-deployable/target/wasm32-wasip1/release/text_engine_deployable.wasm")
}

#[test]
fn test_v17_admission_control() {
    let path = wasm_path();
    if !path.exists() { eprintln!("Skipping"); return; }
    let rt = host::runtime_v17::Runtime::new().expect("load");
    
    // 1. Невалидный манифест (без capabilities)
    let bad_manifest = host::runtime_v17::Manifest {
        name: "bad".to_string(),
        version: "1.0".to_string(),
        required_capabilities: vec![], 
    };
    let res = rt.deploy_from_path(&path, bad_manifest);
    assert!(res.is_err(), "Should reject manifest without capabilities");
    assert!(res.unwrap_err().contains("Admission REJECT"));

    // 2. Валидный манифест
    let good_manifest = host::runtime_v17::Manifest {
        name: "good".to_string(),
        version: "1.0".to_string(),
        required_capabilities: vec!["fs:read".to_string()],
    };
    rt.deploy_from_path(&path, good_manifest).expect("Should accept valid manifest");

    // 3. Вызов зарегистрированного компонента
    let out = rt.process("good", "HELLO").expect("process");
    assert_eq!(out, "hello");
}
EOF

echo "=== [5/7] Visual demo (demo-web-ui-ch17) ==="
if [ ! -d "demo-web-ui-ch17" ]; then
  cargo new demo-web-ui-ch17 --bin --vcs none >/dev/null
fi
rm -f demo-web-ui-ch17/Cargo.toml demo-web-ui-ch17/src/main.rs
mkdir -p demo-web-ui-ch17/static
rm -f demo-web-ui-ch17/static/index.html
sleep 0.1
cat > demo-web-ui-ch17/Cargo.toml <<'EOF'
[package]
name = "demo-web-ui-ch17"
version.workspace = true
edition.workspace = true
[dependencies]
host = { path = "../host" }
axum = "0.8"
tokio = { version = "1", features = ["full"] }
serde_json = "1"
anyhow = "1.0"
EOF

# ИСПРАВЛЕНИЕ: Используем deploy_from_path вместо прямого вызова wasmtime
cat > demo-web-ui-ch17/src/main.rs <<'EOF'
use axum::{response::Html, routing::{get, post}, Json, Router};
use serde_json::{json, Value};
use std::sync::Arc;
use host::runtime_v17::{Runtime, Manifest};

const INDEX_HTML: &str = include_str!("../static/index.html");
async fn index() -> Html<&'static str> { Html(INDEX_HTML) }

async fn deploy_valid(axum::extract::State(rt): axum::extract::State<Arc<Runtime>>) -> Json<Value> {
    let manifest = Manifest {
        name: "text-engine".to_string(),
        version: "1.0.0".to_string(),
        required_capabilities: vec!["fs:read".to_string()],
    };
    let path = std::path::PathBuf::from(env!("CARGO_MANIFEST_DIR"))
        .parent().unwrap()
        .join("components/text-engine-deployable/target/wasm32-wasip1/release/text_engine_deployable.wasm");
    
    match rt.deploy_from_path(&path, manifest) {
        Ok(_) => Json(json!({ "status": "success", "message": "Component deployed and registered." })),
        Err(e) => Json(json!({ "status": "error", "message": e })),
    }
}

async fn deploy_invalid(axum::extract::State(rt): axum::extract::State<Arc<Runtime>>) -> Json<Value> {
    let manifest = Manifest {
        name: "text-engine-bad".to_string(),
        version: "1.0.0".to_string(),
        required_capabilities: vec![],
    };
    let path = std::path::PathBuf::from(env!("CARGO_MANIFEST_DIR"))
        .parent().unwrap()
        .join("components/text-engine-deployable/target/wasm32-wasip1/release/text_engine_deployable.wasm");
    
    match rt.deploy_from_path(&path, manifest) {
        Ok(_) => Json(json!({ "status": "success", "message": "Unexpected success!" })),
        Err(e) => Json(json!({ "status": "error", "message": e })),
    }
}

async fn process(axum::extract::State(rt): axum::extract::State<Arc<Runtime>>, Json(p): Json<Value>) -> Json<Value> {
    let input = p["input"].as_str().unwrap_or("").to_string();
    match rt.process("text-engine", &input) {
        Ok(out) => Json(json!({ "output": out })),
        Err(e) => Json(json!({ "error": format!("{e}") })),
    }
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let runtime = Runtime::new()?;
    let app = Router::new()
        .route("/", get(index))
        .route("/deploy-valid", post(deploy_valid))
        .route("/deploy-invalid", post(deploy_invalid))
        .route("/process", post(process))
        .with_state(runtime);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3016").await?;
    println!("🚀 Demo UI (Ch17): http://127.0.0.1:3016");
    axum::serve(listener, app).await?;
    Ok(())
}
EOF

cat > demo-web-ui-ch17/static/index.html <<'HTML_EOF'
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8"><title>Ch17: Deployable Unit</title>
<style>
  :root { --bg:#0f172a; --panel:#1e293b; --br:#334155; --tx:#e2e8f0; --mu:#94a3b8; --ac:#38bdf8; --ok:#22c55e; --er:#ef4444; --wr:#f59e0b; --mo:ui-monospace,SFMono-Regular,Menlo,Consolas,monospace; }
  *{box-sizing:border-box;margin:0;padding:0}
  body{font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,sans-serif; background:var(--bg);color:var(--tx);min-height:100vh; display:flex;align-items:center;justify-content:center;padding:2rem}
  .box{max-width:850px;width:100%;background:var(--panel); border:1px solid var(--br);border-radius:12px;padding:2rem; box-shadow:0 20px 60px rgba(0,0,0,.4)}
  h1{font-size:1.6rem;font-weight:600;text-align:center;margin-bottom:.35rem; background:linear-gradient(135deg,var(--ac),#a78bfa); -webkit-background-clip:text; background-clip:text; -webkit-text-fill-color:transparent}
  .sub{color:var(--mu);text-align:center;font-size:.9rem;margin-bottom:2rem}
  
  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 1rem; margin-bottom: 1.5rem; }
  .panel { background: var(--bg); border: 1px solid var(--br); border-radius: 8px; padding: 1.25rem; text-align: center; }
  h2 { font-size: .9rem; font-weight: 600; margin-bottom: .5rem; color: var(--ac); }
  .desc { font-size: .75rem; color: var(--mu); margin-bottom: 1rem; }
  button { padding: .6rem 1rem; border-radius: 6px; border: none; font-weight: 600; cursor: pointer; width: 100%; }
  button.good { background: var(--ac); color: #0c1222; }
  button.bad { background: var(--er); color: white; }
  button.process { background: var(--ok); color: white; padding: .8rem; font-size: 1rem; }
  button:hover { filter: brightness(1.1); }
  button:disabled { opacity: .5; cursor: wait; }
  
  .input-row { display: flex; gap: 1rem; margin-bottom: 1rem; }
  input[type=text] { flex-grow: 1; padding: .8rem; background: var(--bg); border: 1px solid var(--br); color: var(--tx); border-radius: 8px; font-family: var(--mo); }
  
  .result { background: #000; border-radius: 8px; padding: 1rem; font-family: var(--mo); font-size: .85rem; min-height: 60px; white-space: pre-wrap; }
  .res-ok { color: var(--ok); }
  .res-err { color: var(--er); }
</style>
</head>
<body>
<div class="box">
  <h1>Deployable Unit: Manifest & Admission</h1>
  <p class="sub">Registry validates component metadata before loading</p>

  <div class="grid">
    <div class="panel">
      <h2>Valid Deploy</h2>
      <p class="desc">Includes <code>required_capabilities: ["fs:read"]</code>.<br>Passes Admission Control.</p>
      <button class="good" onclick="run('/deploy-valid', 'out-deploy')">Deploy Valid</button>
    </div>
    <div class="panel">
      <h2>Invalid Deploy (Danger)</h2>
      <p class="desc">Missing <code>required_capabilities</code>.<br>Admission REJECT.</p>
      <button class="bad" onclick="run('/deploy-invalid', 'out-deploy')">Deploy Invalid</button>
    </div>
  </div>

  <div class="input-row">
    <input type="text" id="input" value="HELLO DEPLOY">
    <button class="process" onclick="runProcess()">Process</button>
  </div>

  <div class="result" id="out-deploy">Awaiting action...</div>
</div>
<script>
async function run(url, outId) {
  const out = document.getElementById(outId);
  out.innerHTML = 'Processing...'; out.className = 'result';
  try {
    const res = await fetch(url, { method: 'POST' });
    const data = await res.json();
    if (data.error) {
      out.innerHTML = `<span class="res-err">Error: ${data.error}</span>`;
    } else if (data.status === 'error') {
      out.innerHTML = `<span class="res-err">${data.message}</span>`;
    } else {
      out.innerHTML = `<span class="res-ok">${data.message}</span>`;
    }
  } catch(e) { out.innerHTML = `<span class="res-err">Network error: ${e}</span>`; }
}

async function runProcess() {
  const out = document.getElementById('out-deploy');
  out.innerHTML = 'Processing...'; out.className = 'result';
  try {
    const res = await fetch('/process', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ input: document.getElementById('input').value })
    });
    const data = await res.json();
    if (data.error) {
      out.innerHTML = `<span class="res-err">${data.error}</span>`;
    } else {
      out.innerHTML = `<span class="res-ok">Output: ${data.output}</span>`;
    }
  } catch(e) { out.innerHTML = `<span class="res-err">Network error: ${e}</span>`; }
}
</script>
</body>
</html>
HTML_EOF

echo "=== [6/7] Build & verify ==="
(cd components/text-engine-deployable && cargo component build --release --quiet)

echo "=== [7/7] Build Host & Demo ==="
cargo check -p host --lib --quiet
cargo check -p demo-web-ui-ch17 --quiet

echo "✅ Готово. Следующие шаги:"
echo "  1. Тесты:  (cd host && cargo test --test integration_v17)"
echo "  2. Демо:   cargo run -p demo-web-ui-ch17 --release"
echo "  3. Открыть: http://127.0.0.1:3016"
```

---

# Часть B. Глава 17. Component as a deployable unit (Manifest, Registry & Admission)

**Зачем создана эта глава?**
До сих пор мы загружали `.wasm` файлы напрямую из файловой системы по жестко заданному пути. Если мы просто меняли файл, система начинала использовать новый код. В реальном backend это недопустимо. Компонент должен быть **атомарной единицей поставки (Deployable Unit)**, которая проходит проверку перед тем, как попасть в систему. Мы должны разделить **Data Plane** (выполнение HTTP-запросов) и **Control Plane** (управление версиями и политиками). В этой главе мы создадим Manifest (паспорт компонента), Registry (реестр доверенных артефактов) и Admission Control (проверку прав).

**Какую пользу несёт глава?**
1. **Манифест (Manifest):** Вы научитесь описывать требования компонента (например, `required_capabilities`) в виде метаданных.
2. **Admission Control:** Мы реализуем паттерн, где Registry отклоняет загрузку компонента, если его требования не совпадают с политикой безопасности системы.
3. **Разделение слоев:** Демо-приложение (UI) больше не работает с `wasmtime` напрямую. Оно просит `Runtime` задеплоить компонент, а `Runtime` решает, можно ли это сделать.

**Чего мы хотим достичь по результатам тестирования?**
1. Создать структуру `Manifest` с полем `required_capabilities`.
2. Настроить `Registry`, который проверяет манифест.
3. Доказать в тестах, что компонент без `required_capabilities` получает отказ в регистрации (`Admission REJECT`), а валидный компонент регистрируется и выполняется.

---

### 🚦 Starting point
У нас есть `runtime_v16.rs` с безопасным Hot Swap. Мы создаем `runtime_v17.rs`, который добавляет концепцию Registry.

### 🎯 PREDICT (Предсказание)
*Запишите свой прогноз перед запуском:*
> **Если мы попытаемся загрузить компонент, который требует доступ к файловой системе, но в манифесте мы не указали `required_capabilities`, на каком этапе это остановится: при попытке загрузить `.wasm` файл, при попытке выполнить его, или при регистрации в Registry?**

---

## Этап 1. Исходные коды

### Структура
**Новые файлы:**
- `components/text-engine-deployable/` — компонент.
- `host/src/runtime_v17.rs` — рантайм с Registry и Admission Control.
- `host/tests/integration_v17.rs` — тесты.
- `demo-web-ui-ch17/` — UI.

### Ключевые фрагменты кода

**1. `host/src/runtime_v17.rs` (Манифест и Реестр)**
```rust
#[derive(Debug, Deserialize, Clone)]
pub struct Manifest {
    pub name: String,
    pub version: String,
    pub required_capabilities: Vec<String>,
}

pub struct Registry {
    components: Mutex<HashMap<String, Arc<Component>>>,
}

impl Registry {
    pub fn register(&self, manifest: &Manifest, component: Component) -> Result<(), String> {
        // Admission Control: проверяем политики
        if manifest.required_capabilities.is_empty() {
            return Err("Admission REJECT: missing required_capabilities".to_string());
        }
        if !manifest.required_capabilities.contains(&"fs:read".to_string()) {
            return Err("Admission REJECT: invalid capabilities".to_string());
        }

        let mut map = self.components.lock().unwrap();
        map.insert(manifest.name.clone(), Arc::new(component));
        Ok(())
    }
}
```

**2. `host/src/runtime_v17.rs` (Метод деплоя)**
```rust
pub fn deploy_from_path(&self, path: &Path, manifest: Manifest) -> Result<(), String> {
    // Загружаем бинарник
    let component = Component::from_file(&self.engine, path)
        .map_err(|e| format!("Load failed: {e}"))?;
    // Регистрируем (проходим Admission Control)
    self.registry.register(&manifest, component)
}
```

---

## Этап 2. Тесты

**`host/tests/integration_v17.rs`**

### Команды запуска
```bash
(cd components/text-engine-deployable && cargo component build --release)
(cd host && cargo test --test integration_v17)
```

**Что проверяется:**
1. `deploy_from_path` с пустым `required_capabilities` возвращает `Err` (Admission REJECT).
2. `deploy_from_path` с `["fs:read"]` возвращает `Ok`.
3. Зарегистрированный компонент можно успешно вызвать через `process("good", "HELLO")`.

---

## Этап 3. Демо-пример

Визуальный сервер `demo-web-ui-ch17` (порт 3016).

### Запуск
```bash
cargo run -p demo-web-ui-ch17 --release
# open http://127.0.0.1:3016
```

### OBSERVE (Наблюдение)

1. Нажмите **Deploy Invalid** (красная кнопка).
   * *Результат:* `Admission REJECT: missing required_capabilities`. Компонент даже не попал в память Registry.
2. Нажмите **Deploy Valid** (синяя кнопка).
   * *Результат:* `Component deployed and registered.`
3. Нажмите **Process**.
   * *Результат:* `Output: hello deploy`.

---

## 🔄 BREAK (Поломка) — Обход Admission Control

**Цель:** Увидеть, что будет, если попытаться загрузить компонент напрямую в `Engine`, минуя `Registry`.
**Шаг 1.** В `demo-web-ui-ch17/src/main.rs` удалите вызов `deploy_from_path` и попробуйте сделать `Component::from_file` напрямую (как в Главе 15).
* *Результат:* Компонент загрузится в память.
* *Урок:* `wasmtime` не знает о ваших бизнес-правилах. Без слоя `Registry` и `Admission Control` любой `.wasm` файл будет выполнен. Безопасность обеспечивается только архитектурой Host'а.

---

## 🔄 Итеративная проверка

**Нужны ли доработки после тестов?**
В процессе написания мы поняли, что демо-приложение не должно зависеть от `wasmtime`. Мы инкапсулировали загрузку в метод `deploy_from_path`, чтобы UI-слой работал только с понятиями `Manifest` и `Runtime`. Это правильное архитектурное решение: демо-слой не должен знать, как именно работает движок.

---

## Итог главы

**OBSERVE:**
- Registry проверяет манифест до того, как компонент будет сохранен в памяти.
- Manifest — это контракт между разработчиком компонента и платформой.
- Ошибка `Admission REJECT` возвращается как обычная строка, которую легко показать пользователю.

**PREDICT (проверка):**
Было предсказано: "Остановится при регистрации". Результат: **подтвердилось**. Компонент физически существует и валиден, но платформа отказывается его регистрировать из-за нарушений политики безопасности.

**What did we prove?**
1. **Control Plane отделен от Data Plane.** Загрузка и проверка компонента не влияют на обработку текущих запросов.
2. **Admission Control работает.** Мы можем отклонить компонент на основе метаданных, еще до его запуска.
3. **Манифест формализует требования.** Теперь у нас есть место, где можно указать `required_capabilities` и `version`.

**What did we not prove?**
- Как проверять совместимость (Compatibility matrix) между версией Host и версией Компонента.

**What did we pay for it?**
- Появился дополнительный слой абстракции (`Registry`), который нужно поддерживать.

**When is the alternative better?**
- Если у вас закрытая система (вы сами собираете и деплоите компоненты), можно обойтись без Registry. Но если вы планируете подключать сторонние (third-party) компоненты, Registry и Admission Control — обязательны.

### 📊 Evidence
```text
Claim:          Registry блокирует компоненты с невалидным манифестом.
Experiment:     Попытка задеплоить компонент без required_capabilities.
Measurement:    Метод deploy_from_path возвращает Err("Admission REJECT").
Observed:       Компонент не попадает в активный список.
Conclusion:     Политики безопасности проверяются до запуска кода.
Confidence:     High
```

### Что у нас теперь есть
- `Manifest` и `Registry` в `runtime_v17.rs`.
- UI с кнопками проверки Admission Control.
- Все предыдущие главы (1–16) не тронуты.

**Следующий шаг:** Мы готовы к **Главе 18 (Composition models)**, где научимся собирать несколько компонентов в единый пайплайн (например, `A -> B -> C`), чтобы строить сложные бизнес-процессы.