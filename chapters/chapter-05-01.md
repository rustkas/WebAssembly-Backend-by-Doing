# Приложение к Главе 5. Интерактивное демо-приложение (Web UI)

**Зачем создано это приложение?**
В Главе 5 мы доказали работоспособность компонента через `curl` и скрипт `measure.sh`. Но для полноты картины и наглядной демонстрации полезно иметь **визуальный интерфейс**, который можно открыть в браузере. Это приложение не только служит отличным инструментом для презентации проекта, но и доказывает важную архитектурную мысль: `host` — это переиспользуемая библиотека. Мы можем поднять *новый* независимый веб-сервер, который вообще ничего не знает про `wasmtime` или `wit-bindgen`, и заставить его вызывать наш WASM-компонент.

**Какую пользу несёт приложение?**
1. **Визуальное доказательство:** Вы своими глазами видите, как текст из `<textarea>` проходит через HTTP, попадает в WASM-компонент и возвращается обратно в виде зеленого результата.
2. **Демонстрация переиспользования:** Крейт `demo-web-ui` зависит только от `host`, `axum` и `tokio`. Никаких зависимостей от Wasmtime.
3. **Замер round-trip из браузера:** Мы видим реальное время ответа (3–4 мс), которое включает в себя HTTP-оверхед, работу `spawn_blocking`, создание `Store` и сам вызов WASM.

---

## Структура подпроекта

```text
wasm-backend-lab/
├── Cargo.toml                     ← + одна строка в members
├── host/                          ← без изменений
├── components/                    ← без изменений
└── demo-web-ui/                   ← НОВЫЙ проект
    ├── Cargo.toml
    ├── src/
    │   └── main.rs
    └── static/
        └── index.html
```

### 1. `demo-web-ui/Cargo.toml`

```toml
[package]
name = "demo-web-ui"
version.workspace = true
edition.workspace = true

[dependencies]
host = { path = "../host" }
axum = "0.8"
tokio = { version = "1", features = ["full"] }
anyhow = "1.0"
```

### 2. `demo-web-ui/src/main.rs`

Сервер занимает всего 30 строк. Он добавляет маршрут `GET /` к уже готовому `host::http::router`.

```rust
use axum::{response::Html, routing::get, Router};
use host::Runtime;

const INDEX_HTML: &str = include_str!("../static/index.html");

async fn index() -> Html<&'static str> {
    Html(INDEX_HTML)
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let wasm_path = host::default_wasm_path();
    if !wasm_path.exists() {
        eprintln!("Error: WASM component not found at `{}`.", wasm_path.display());
        eprintln!("Run `cd components/text-engine-component && cargo component build --release` first.");
        std::process::exit(1);
    }

    println!("Loading component: {}", wasm_path.display());
    let runtime = Runtime::new(&wasm_path)?;

    // host::http::router уже содержит POST /process; добавляем только GET /
    let app: Router = host::http::router(runtime).route("/", get(index));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3003").await?;
    println!("🚀 Demo UI: http://127.0.0.1:3003");
    axum::serve(listener, app).await?;
    Ok(())
}
```

### 3. `demo-web-ui/static/index.html`

Страница написана на чистом HTML/CSS/JS без сборщиков. Для отображения результата используется `fetch()`, а время round-trip замеряется через `performance.now()`. CSS-стили используют современные свойства (с фоллбэками для совместимости).

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>WASM Component Demo</title>
  <style>
    :root {
      --bg: #0f172a; --panel: #1e293b; --border: #334155;
      --text: #e2e8f0; --muted: #94a3b8; --accent: #38bdf8;
      --accent-hover: #0ea5e9; --success: #22c55e; --error: #ef4444;
      --mono: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
    }
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
      background: var(--bg); color: var(--text);
      min-height: 100vh; display: flex; align-items: center; justify-content: center; padding: 2rem;
    }
    .container {
      max-width: 820px; width: 100%; background: var(--panel);
      border: 1px solid var(--border); border-radius: 12px;
      padding: 2rem; box-shadow: 0 20px 60px rgba(0,0,0,0.4);
    }
    header { margin-bottom: 2rem; text-align: center; }
    h1 {
      font-size: 1.75rem; font-weight: 600; margin-bottom: 0.5rem;
      background: linear-gradient(135deg, var(--accent), #a78bfa);
      -webkit-background-clip: text;
              background-clip: text;
      -webkit-text-fill-color: transparent;
    }
    .subtitle { color: var(--muted); font-size: 0.9rem; }
    .subtitle code {
      font-family: var(--mono); background: rgba(255,255,255,0.05);
      padding: 0.15rem 0.4rem; border-radius: 4px; color: var(--accent);
    }
    label {
      display: block; font-size: 0.8rem; font-weight: 500; color: var(--muted);
      margin-bottom: 0.5rem; text-transform: uppercase; letter-spacing: 0.05em;
    }
    textarea {
      width: 100%; min-height: 120px; padding: 1rem; font-family: var(--mono); font-size: 0.9rem;
      background: var(--bg); color: var(--text); border: 1px solid var(--border); border-radius: 8px;
      resize: vertical; outline: none; transition: border-color 0.15s;
    }
    textarea:focus { border-color: var(--accent); }
    .presets { display: flex; gap: 0.5rem; flex-wrap: wrap; align-items: center; margin-top: 0.75rem; }
    .presets > span { font-size: 0.75rem; color: var(--muted); margin-right: 0.25rem; }
    .actions { display: flex; gap: 0.75rem; margin-top: 1.25rem; flex-wrap: wrap; }
    button {
      padding: 0.65rem 1.25rem; font-size: 0.9rem; font-weight: 500;
      border: none; border-radius: 8px; cursor: pointer; transition: all 0.15s;
    }
    button.primary { background: var(--accent); color: #0c1222; }
    button.primary:hover { background: var(--accent-hover); }
    button.primary:disabled { opacity: 0.5; cursor: wait; }
    button.secondary {
      background: transparent; color: var(--muted); border: 1px solid var(--border);
      font-family: var(--mono); font-size: 0.78rem;
    }
    button.secondary:hover { color: var(--text); border-color: var(--accent); }
    .result {
      margin-top: 1.5rem; padding: 1rem; background: var(--bg);
      border: 1px solid var(--border); border-radius: 8px; min-height: 70px;
    }
    .result-label {
      font-size: 0.72rem; color: var(--muted); text-transform: uppercase;
      letter-spacing: 0.05em; margin-bottom: 0.5rem;
      display: flex; justify-content: space-between; align-items: baseline;
    }
    .result-body {
      font-family: var(--mono); font-size: 0.9rem; white-space: pre-wrap;
      word-break: break-word; color: var(--muted);
    }
    .result-body.success { color: var(--success); }
    .result-body.error   { color: var(--error); }
    .timing { font-family: var(--mono); font-size: 0.75rem; color: var(--accent); }
    .badge {
      display: inline-block; font-size: 0.68rem; font-weight: 600; padding: 0.15rem 0.45rem;
      border-radius: 4px; margin-left: 0.5rem; background: rgba(56,189,248,0.15);
      color: var(--accent); text-transform: uppercase; vertical-align: middle;
    }
  </style>
</head>
<body>
  <div class="container">
    <header>
      <h1>WASM Component Demo</h1>
      <p class="subtitle">
        Calls <code>text-engine-component.wasm</code> via Wasmtime + Component Model
      </p>
    </header>

    <label for="input">Input text <span class="badge">POST /process</span></label>
    <textarea id="input" spellcheck="false">  Hello   WASM   World  </textarea>

    <div class="presets">
      <span>Presets:</span>
      <button class="secondary" data-input="  Hello   WASM   World  ">extra spaces</button>
      <button class="secondary" data-input="RUST &#10; is &#13;&#10; AWESOME">newlines</button>
      <button class="secondary" data-input="!!! ???">specials</button>
      <button class="secondary" data-input="">empty</button>
    </div>

    <div class="actions">
      <button class="primary" id="run-btn">Process via WASM →</button>
    </div>

    <div class="result">
      <div class="result-label">
        <span>Result</span>
        <span class="timing" id="timing"></span>
      </div>
      <div class="result-body" id="output">Ready. Click "Process via WASM".</div>
    </div>
  </div>

  <script>
    const inputEl  = document.getElementById('input');
    const outputEl = document.getElementById('output');
    const timingEl = document.getElementById('timing');
    const runBtn   = document.getElementById('run-btn');

    document.querySelectorAll('.presets button').forEach(btn => {
      btn.addEventListener('click', () => { inputEl.value = btn.dataset.input; inputEl.focus(); });
    });

    async function run() {
      runBtn.disabled = true;
      outputEl.className = 'result-body';
      outputEl.textContent = 'Calling WASM component...';
      timingEl.textContent = '';

      const t0 = performance.now();
      try {
        const resp = await fetch('/process', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ input: inputEl.value }),
        });
        const data = await resp.json();
        const ms = performance.now() - t0;
        timingEl.textContent = 'round-trip: ' + ms.toFixed(2) + ' ms';

        if (data.error) {
          outputEl.className = 'result-body error';
          outputEl.textContent = 'Error: ' + data.error;
        } else {
          outputEl.className = 'result-body success';
          outputEl.textContent = data.output === '' ? '(empty string)' : data.output;
        }
      } catch (e) {
        outputEl.className = 'result-body error';
        outputEl.textContent = 'Network error: ' + e;
      } finally {
        runBtn.disabled = false;
      }
    }

    runBtn.addEventListener('click', run);
    inputEl.addEventListener('keydown', (e) => {
      if ((e.ctrlKey || e.metaKey) && e.key === 'Enter') { e.preventDefault(); run(); }
    });
  </script>
</body>
</html>
```

---

## Автоматизация создания проекта

Чтобы не копировать файлы вручную, в репозитории добавлен скрипт `scripts/init-demo-web-ui.sh`. Он использует `cargo new` для корректной регистрации пакета в workspace, а затем перезаписывает конфигурацию и исходный код.

Запуск:
```bash
./scripts/init-demo-web-ui.sh
```

Скрипт идемпотентен: повторный запуск безопасно обновит `Cargo.toml`, `main.rs` и `index.html` до актуального состояния.

---

## 🔬 Evidence: Результаты реального запуска

Запускаем сервер:
```bash
cargo run -p demo-web-ui --release
# Loading component: .../text_engine_component.wasm
# 🚀 Demo UI: http://127.0.0.1:3003
```

Открываем `http://127.0.0.1:3003` в браузере и проверяем три сценария:

1. **Стандартный ввод с лишними пробелами:** 
   Вводим `  Hello   WASM   World  `.
   Результат: `hello wasm world` (зеленым). Round-trip: **4.00 ms**.
2. **Спецсимволы (крайний случай):**
   Нажимаем пресет `specials` (`!!! ???`).
   Результат: `!!! ???`. Round-trip: **3.00 ms**.
3. **Пустая строка:**
   Нажимаем пресет `empty`.
   Результат: `(empty string)`. Round-trip: **3.00 ms**.

**Что это доказывает:**
- **Инфраструктура изолирована.** `demo-web-ui` не знает, что под капотом `host::Runtime` работает Wasmtime. Он просто шлёт `fetch()` на `/process`.
- **WASM полностью функционален.** Он корректно обрабатывает граничные случаи (спецсимволы, пустые строки), возвращая валидный JSON.
- **Накладные расходы (round-trip).** В браузере мы видим 3–4 мс. Это время включает HTTP-сеть, парсинг JSON, `spawn_blocking`, создание `Store`, вызов WASM и обратный путь. Для интерактивного UI это мгновенно.

---

## 🎯 What did we prove?

1. **API `host` переиспользуем.** Один и тот же компонент `text-engine-component.wasm` успешно загружается и вызывается двумя разными бинарниками: `host_v2` (API-сервер) и `demo-web-ui` (UI-сервер).
2. **Изоляция WASM работает.** Каждый клик по кнопке создаёт новый `Store` и `WasiCtx`. Даже если компонент запаникует (trap), это не уронит сервер `demo-web-ui`.
3. **Визуальное подтверждение.** Мы получили работающий end-to-end сценарий: `UI -> Axum -> Wasmtime -> Component -> UI`.

Теперь у нас есть не только консольные метрики и скрипты, но и наглядное доказательство того, что созданная архитектура готова к использованию в реальных веб-приложениях.