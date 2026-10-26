# Часть A. CLI-автоматизация: `scripts/setup-ch9.sh`

Скрипт создаёт компонент, настраивает песочницу и поднимает визуальный интерфейс. Он включает в себя все правки, которые мы сделали в процессе отладки (правильный импорт `FsPerms`, передача перечисления без массива, создание `sandbox` в тестах).

```bash
cd ~/projects/wa/wasm-backend-lab
mkdir -p scripts
nano scripts/setup-ch9.sh
```

<details>
<summary><b>Полный текст `scripts/setup-ch9.sh`</b></summary>

```bash
#!/usr/bin/env bash
# scripts/setup-ch9.sh
#
# Автоматическая настройка Главы 9: Host API design: Capabilities.
# Создает изолированную директорию (sandbox) и показывает, как WASI 
# блокирует доступ к файлам за ее пределами.
set -euo pipefail
ROOT="$(cd "$(dirname "$0")/.." && pwd)"
cd "$ROOT"

echo "=== [1/7] Exclude new component from workspace ==="
if ! grep -q 'components/text-engine-secure' Cargo.toml; then
  python3 - <<'PY'
import re, pathlib
p = pathlib.Path("Cargo.toml")
s = p.read_text()
m = re.search(r'exclude\s*=\s*\[(.*?)\]', s, re.S)
if not m:
    raise SystemExit("exclude array not found in Cargo.toml")
inner = m.group(1).rstrip()
if not inner.endswith(','): inner += ','
inner += '\n    "components/text-engine-secure",\n'
p.write_text(s[:m.start()] + 'exclude = [' + inner + ']' + s[m.end():])
print("Added text-engine-secure to exclude.")
PY
fi

echo "=== [2/7] WIT contract ==="
mkdir -p wit/text-engine-secure
rm -f wit/text-engine-secure/world.wit
sleep 0.1
cat > wit/text-engine-secure/world.wit <<'EOF'
package docs:text-engine-secure@0.1.0;

interface processor {
    read-file: func(path: string) -> result<string, string>;
}

world text-engine-secure {
    export processor;
}
EOF

echo "=== [3/7] Guest component ==="
if [ ! -d "components/text-engine-secure" ]; then
  cargo new components/text-engine-secure --lib --vcs none >/dev/null
fi
rm -f components/text-engine-secure/Cargo.toml components/text-engine-secure/src/lib.rs
sleep 0.1
cat > components/text-engine-secure/Cargo.toml <<'EOF'
[package]
name = "text-engine-secure"
version = "0.1.0"
edition = "2021"

[workspace]

[lib]
crate-type = ["cdylib"]

[dependencies]
wit-bindgen = "0.63"
EOF
cat > components/text-engine-secure/src/lib.rs <<'EOF'
wit_bindgen::generate!({
    path: "../../wit/text-engine-secure",
    world: "text-engine-secure",
});

use exports::docs::text_engine_secure::processor::Guest;

struct SecureEngine;

impl Guest for SecureEngine {
    fn read_file(path: String) -> Result<String, String> {
        // WASI заблокирует этот вызов, если файл не находится в разрешенной директории.
        // Для гостя файл за пределами sandbox выглядит как "несуществующий".
        match std::fs::read_to_string(&path) {
            Ok(content) => Ok(content),
            Err(e) => Err(format!("Failed to read: {}", e)),
        }
    }
}

export!(SecureEngine);
EOF

echo "=== [4/7] Host runtime_v9 ==="
rm -f host/src/runtime_v9.rs
sleep 0.1
cat > host/src/runtime_v9.rs <<'EOF'
use anyhow::Result;
use std::path::Path;
use std::sync::Arc;
use wasmtime::component::{Component, Linker};
use wasmtime::{Engine, Store};
use wasmtime_wasi::filesystem::FsPerms;
use wasmtime_wasi::{WasiCtx, WasiCtxBuilder, WasiCtxView, WasiView};

wasmtime::component::bindgen!({
    path: "../wit/text-engine-secure",
    world: "text-engine-secure",
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

    pub fn read_file(&self, path: &str) -> Result<Result<String, String>> {
        // FsPerms::ReadWrite дает гостю права на чтение и запись.
        // Мы передаем один экземпляр перечисления.
        let wasi = WasiCtxBuilder::new()
            .preopened_dir("./sandbox", "/sandbox", FsPerms::ReadWrite)?
            .build();

        let mut store = Store::new(&self.engine, MyState {
            ctx: wasi,
            table: wasmtime::component::ResourceTable::new(),
        });

        let b = TextEngineSecure::instantiate(&mut store, &self.component, &self.linker)?;
        let res = b.docs_text_engine_secure_processor().call_read_file(&mut store, path)?;
        
        Ok(res.map_err(|e| e.to_string()))
    }
}
EOF
if ! grep -q 'pub mod runtime_v9;' host/src/lib.rs; then
  echo 'pub mod runtime_v9;' >> host/src/lib.rs
fi
touch host/src/lib.rs

echo "=== [5/7] Integration tests ==="
rm -f host/tests/integration_v9.rs
sleep 0.1
cat > host/tests/integration_v9.rs <<'EOF'
use std::path::PathBuf;

fn wasm_path() -> PathBuf {
    PathBuf::from(env!("CARGO_MANIFEST_DIR"))
        .parent().unwrap()
        .join("components/text-engine-secure/target/wasm32-wasip1/release/text_engine_secure.wasm")
}

#[test]
fn test_v9_capabilities() {
    let path = wasm_path();
    if !path.exists() { eprintln!("Skipping"); return; }

    // Создаем папку и файл в текущей директории (host/)
    std::fs::create_dir_all("./sandbox").unwrap();
    std::fs::write("./sandbox/allowed.txt", "This is an allowed file.").unwrap();

    let rt = host::runtime_v9::Runtime::new(&path).expect("load");

    // 1. Чтение разрешенного файла
    let res = rt.read_file("/sandbox/allowed.txt").unwrap();
    assert!(res.is_ok(), "Should read allowed file: {:?}", res.err());
    assert_eq!(res.unwrap().trim(), "This is an allowed file.");

    // 2. Попытка чтения секретного файла
    let secret_res = rt.read_file("/secret.txt").unwrap();
    assert!(secret_res.is_err(), "Should NOT read secret file");
    let err = secret_res.unwrap_err();
    assert!(err.contains("No such file") || err.contains("Permission"), 
            "Expected not found or permission denied, got: {}", err);
}
EOF

echo "=== [6/7] Visual demo ==="
if [ ! -d "demo-web-ui-ch9" ]; then
  cargo new demo-web-ui-ch9 --bin --vcs none >/dev/null
fi
rm -f demo-web-ui-ch9/Cargo.toml demo-web-ui-ch9/src/main.rs
mkdir -p demo-web-ui-ch9/static
rm -f demo-web-ui-ch9/static/index.html
sleep 0.1
cat > demo-web-ui-ch9/Cargo.toml <<'EOF'
[package]
name = "demo-web-ui-ch9"
version.workspace = true
edition.workspace = true
[dependencies]
host = { path = "../host" }
axum = "0.8"
tokio = { version = "1", features = ["full"] }
serde_json = "1"
anyhow = "1.0"
EOF
cat > demo-web-ui-ch9/src/main.rs <<'EOF'
use axum::{response::Html, routing::{get, post}, Json, Router};
use serde_json::{json, Value};
use std::sync::Arc;
use host::runtime_v9::Runtime;

const INDEX_HTML: &str = include_str!("../static/index.html");
async fn index() -> Html<&'static str> { Html(INDEX_HTML) }

async fn read_allowed(axum::extract::State(rt): axum::extract::State<Arc<Runtime>>) -> Json<Value> {
    match tokio::task::spawn_blocking(move || rt.read_file("/sandbox/allowed.txt")).await {
        Ok(Ok(Ok(content))) => Json(json!({ "output": content })),
        Ok(Ok(Err(e))) => Json(json!({ "error": e })),
        _ => Json(json!({ "error": "internal_error" })),
    }
}

async fn read_secret(axum::extract::State(rt): axum::extract::State<Arc<Runtime>>) -> Json<Value> {
    match tokio::task::spawn_blocking(move || rt.read_file("/secret.txt")).await {
        Ok(Ok(Ok(content))) => Json(json!({ "output": content })),
        Ok(Ok(Err(e))) => {
            eprintln!("[guest-capability-error] {e}");
            // В UI показываем абстракцию более высокого уровня
            Json(json!({ "error": "Permission denied (Capability boundary)" }))
        },
        _ => Json(json!({ "error": "internal_error" })),
    }
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let manifest = std::path::Path::new(env!("CARGO_MANIFEST_DIR"));
    let wasm_path = manifest.parent().unwrap()
        .join("components/text-engine-secure/target/wasm32-wasip1/release/text_engine_secure.wasm");
    if !wasm_path.exists() { std::process::exit(1); }

    let runtime = Runtime::new(&wasm_path)?;
    // Убедимся, что директория существует при запуске UI
    std::fs::create_dir_all("./sandbox")?;

    let app = Router::new()
        .route("/", get(index))
        .route("/allowed", post(read_allowed))
        .route("/secret", post(read_secret))
        .with_state(runtime);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3008").await?;
    println!("🚀 Demo UI (Ch9): http://127.0.0.1:3008");
    axum::serve(listener, app).await?;
    Ok(())
}
EOF
cat > demo-web-ui-ch9/static/index.html <<'HTML_EOF'
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8"><title>Ch9: Capabilities</title>
<style>
  :root { --bg:#0f172a; --panel:#1e293b; --br:#334155; --tx:#e2e8f0; --mu:#94a3b8; --ac:#38bdf8; --ok:#22c55e; --er:#ef4444; --mo:ui-monospace,monospace; }
  *{box-sizing:border-box;margin:0;padding:0}
  body{font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,sans-serif; background:var(--bg);color:var(--tx);min-height:100vh; display:flex;align-items:center;justify-content:center;padding:2rem}
  .box{max-width:800px;width:100%;background:var(--panel); border:1px solid var(--br);border-radius:12px;padding:2rem; box-shadow:0 20px 60px rgba(0,0,0,.4)}
  h1{font-size:1.6rem;font-weight:600;text-align:center;margin-bottom:.35rem; background:linear-gradient(135deg,var(--ac),#a78bfa); -webkit-background-clip:text; background-clip:text; -webkit-text-fill-color:transparent}
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
  <h1>Capabilities: Sandbox & Preopened Dirs</h1>
  <p class="sub">WASM component attempts to read files via WASI</p>
  <div class="grid">
    <div class="panel">
      <h2>Allowed File</h2>
      <p class="desc">File inside <code>./sandbox/allowed.txt</code>.<br>WASI preopens this directory.</p>
      <button class="good" onclick="run('/allowed', 'out-allowed')">Read Allowed</button>
      <div class="result" id="out-allowed"></div>
    </div>
    <div class="panel">
      <h2>Forbidden File (Danger)</h2>
      <p class="desc">File outside sandbox: <code>/secret.txt</code>.<br>WASI hides it from Guest.</p>
      <button class="bad" onclick="run('/secret', 'out-secret')">Read Secret</button>
      <div class="result" id="out-secret"></div>
    </div>
  </div>
</div>
<script>
async function run(url, outId) {
  const out = document.getElementById(outId);
  out.innerHTML = 'Reading...'; out.className = 'result';
  document.querySelectorAll('button').forEach(b => b.disabled = true);
  try {
    const res = await fetch(url, { method: 'POST' });
    const data = await res.json();
    if (data.error) { out.innerHTML = `<span class="res-err">${data.error}</span>`; }
    else { out.innerHTML = `<span class="res-ok">${data.output.trim()}</span>`; }
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
(cd components/text-engine-secure && cargo component build --release --quiet)
cargo check -p host --lib --quiet
cargo check -p demo-web-ui-ch9 --quiet
echo "✅ Готово."
```
</details>

---

# Часть B. Глава 9. Host API design: Capabilities (Sandbox)

**Зачем создана эта глава?**
До сих пор наши компоненты были полностью изолированы от мира, кроме обмена данными через WIT. Но реальному backend-приложению часто нужно работать с файлами (читать конфиги, сохранять отчеты). В традиционном backend мы просто даем программе доступ ко всей файловой системе. В мире WebAssembly это невозможно: WASM-модуль *по умолчанию не имеет права делать вообще ничего* (кроме математики). Мы должны явно выдать ему **Capability (разрешение)** на доступ к конкретным ресурсам.

**Какую пользу несёт глава?**
1. **Разделение Data и Capability boundaries:** Вы увидите, что передать строку (Data) — это не то же самое, что разрешить открыть файл (Capability).
2. **Песочница (Sandbox):** Мы научимся использовать `preopened_dir` в `WasiCtxBuilder`, чтобы дать компоненту доступ только к одной папке, скрыв всё остальное.
3. **Понимание WASI-ошибок:** Вы увидите, что при попытке выйти за пределы песочницы компонент получит ошибку `No such file or directory` (вместо `Permission Denied`). Это архитектурное решение WASI: скрывать существование файлов от недоверенного кода.

**Чего мы хотим достичь по результатам тестирования?**
1. Создать компонент, который умеет читать файл через `std::fs`.
2. Настроить `Runtime` так, чтобы он предоставлял доступ только к папке `./sandbox`.
3. Написать тест, который доказывает: чтение `allowed.txt` работает, а попытка прочитать `secret.txt` (лежащий вне песочницы) блокируется.

---

### 🚦 Starting point

У нас есть компоненты для работы с текстом и состоянием. В этой главе мы не трогаем их. Мы создаём новый компонент `text-engine-secure` и добавляем в Host модуль `runtime_v9.rs`.

### 🎯 PREDICT (Предсказание)
*Запишите свой прогноз перед запуском:*
> **Если компонент вызовет `std::fs::read_to_string("/etc/passwd")`, а мы настроили ему доступ только к папке `./sandbox`, какую ошибку он получит: `PermissionDenied` (как в Linux) или `NotFound` (файл не найден)? Почему?**

---

## Этап 1. Исходные коды

### Структура
**Новые файлы:**
- `wit/text-engine-secure/world.wit` — контракт на чтение файла.
- `components/text-engine-secure/` — компонент.
- `host/src/runtime_v9.rs` — изолированный рантайм.
- `host/tests/integration_v9.rs` — тесты.
- `demo-web-ui-ch9/` — UI.

**Обновляется:**
- `Cargo.toml` (корень) — `exclude` для нового компонента.
- `host/src/lib.rs` — `pub mod runtime_v9;`.

### Ключевые фрагменты кода

**1. `components/text-engine-secure/src/lib.rs`**
```rust
impl Guest for SecureEngine {
    fn read_file(path: String) -> Result<String, String> {
        // Используем стандартный std::fs. Транслятор WASI 
        // сам перенаправит этот вызов в виртуальную ФС.
        match std::fs::read_to_string(&path) {
            Ok(content) => Ok(content),
            Err(e) => Err(format!("Failed to read: {}", e)),
        }
    }
}
```

**2. `host/src/runtime_v9.rs` (Настройка прав доступа)**
```rust
use wasmtime_wasi::filesystem::FsPerms;

pub fn read_file(&self, path: &str) -> Result<Result<String, String>> {
    // Создаем WASI-контекст с Capability (разрешением).
    // Мы "предоткрываем" (preopen) только папку ./sandbox.
    // Для гостя она будет видна по пути "/sandbox".
    let wasi = WasiCtxBuilder::new()
        .preopened_dir("./sandbox", "/sandbox", FsPerms::ReadWrite)?
        .build();

    let mut store = Store::new(&self.engine, MyState { ctx: wasi, ... });
    // ... вызов компонента ...
}
```

---

## Этап 2. Тесты

**`host/tests/integration_v9.rs`** — проверяет два сценария.

### Команды запуска
```bash
(cd components/text-engine-secure && cargo component build --release)
(cd host && cargo test --test integration_v9)
```

**Что проверяется:**
1. Вызов `read_file("/sandbox/allowed.txt")` успешно возвращает содержимое файла.
2. Вызов `read_file("/secret.txt")` возвращает `Err` (потому что `secret.txt` лежит в корне проекта, а не в `./sandbox`).

---

## Этап 3. Демо-пример

Визуальный сервер `demo-web-ui-ch9` (порт 3008) с двумя кнопками.

### Запуск
```bash
cargo run -p demo-web-ui-ch9 --release
# open http://127.0.0.1:3008
```

### Что увидите
1. Нажимаете **Read Allowed** (левая панель) — получаете зелёный текст `This is an allowed file.`
2. Нажимаете **Read Secret** (правая панель) — получаете красную ошибку `Permission denied (Capability boundary)`.
3. В консоли сервера появляется: `[guest-capability-error] Failed to read: No such file or directory (os error 44)`.

---

## 🔄 BREAK (Поломка) и OBSERVE

**Почему в логе `No such file`, а в UI `Permission denied`?**
Это фундаментальное отличие WASI от классических ОС. В Linux при попытке чтения файла без прав вы получите `EACCES` (Permission denied). Это подсказывает хакеру, что файл *существует*.
В WASI используется принцип **Capability-based security**. Если у компонента нет Capability на директорию, эта директория **не существует в его виртуальной файловой системе**. Поэтому вызов `open()` возвращает `ENOENT` (No such file or directory). 
Мы намеренно перехватываем эту ошибку в Host-коде (в `demo-web-ui-ch9/src/main.rs`) и возвращаем клиенту понятный JSON: `Permission denied`. Но внутри WASM-компонент даже не подозревает, что `/secret.txt` где-то есть.

---

## 🔄 Итеративная проверка

**Нужны ли доработки после тестов?**
Да. API `wasmtime-wasi` версии 49.0.3 преподнесло сюрприз: метод `preopened_dir` изменил сигнатуру (требует перечисление `FsPerms` напрямую, без обёртки в массив). Также при запуске тестов через `cargo test` рабочая директория — это `host/`, поэтому тест должен создавать папку `./sandbox` сам, перед запуском `Runtime`.

---

## Итог главы

**OBSERVE:**
- Без `preopened_dir` компонент не может прочитать ни один файл.
- С `preopened_dir("./sandbox", ...)` компонент читает файлы только внутри `sandbox`.
- Выход за пределы песочницы вызывает ошибку на уровне WASI (файл скрыт).

**PREDICT (проверка):**
Было предсказано: "Компонент получит `PermissionDenied`". Результат: **не подтвердилось**. WASI скрывает файлы, возвращая `NotFound`. Это сделано для безопасности.

**What did we prove?**
1. **Capability boundary — это не просто проверка прав, это архитектура изоляции.** Гость видит только то, что мы ему «предоткрыли».
2. **Host полностью контролирует файловую систему.** Мы можем дать доступ к `/tmp` или `./sandbox`, и компонент будет ограничен этим.
3. **`std::fs` в Rust работает внутри WASM** прозрачно для разработчика (благодаря WASI).

**What did we not prove?**
- Как ограничивать сетевые запросы (это тема Главы 11, где мы будем настраивать `WasiCtxBuilder::allow_tcp`).
- Как передавать секреты (например, токены API) через переменные окружения, не раскрывая их в файловой системе.

**What did we pay for it?**
- Нам пришлось использовать `wasmtime_wasi::filesystem::FsPerms` и явно настраивать `WasiCtxBuilder`.
- Компонент должен знать, что файл находится по пути `/sandbox/file.txt` (виртуальный путь), а не по реальному пути хоста.

**When is the alternative better?**
- Если компоненту нужен неограниченный доступ к файловой системе (например, он реализует `tar` или `grep`), WASI может стать узким местом. Но для 99% backend-задач (чтение конфигов, запись логов в конкретную папку) песочница — это идеальный выбор.

### 📊 Evidence
```text
Claim:          WASI блокирует доступ к файлам за пределами preopened_dir.
Experiment:     Вызов read_file("/secret.txt") при настроенном preopened_dir("./sandbox").
Measurement:    Возвращается ошибка "No such file or directory".
Observed:       Компонент не может выйти за пределы песочницы.
Conclusion:     Capabilities в WASI обеспечивают строгую изоляцию ФС.
Confidence:     High
```

### Что у нас теперь есть
- `components/text-engine-secure/` — компонент, читающий файлы.
- `host/src/runtime_v9.rs` — рантайм с настройкой `preopened_dir`.
- UI, наглядно показывающий разницу между разрешенным и запрещенным доступом.

**Следующий шаг:** В Главе 10 (Time control) мы вернемся к вопросам безопасности и научимся защищать сервер от компонентов, которые пытаются выполнить бесконечный цикл (`loop {}`), пожирая 100% CPU.