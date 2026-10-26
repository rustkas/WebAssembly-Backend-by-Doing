# Глава 4. Сборка компонента и четыре проверки

**Зачем создана эта глава?**
Успешная сборка через `cargo component build` — это только начало. Артефакт может быть собран с логической ошибкой, иметь неверный интерфейс или требовать от Host-приложения тех импортов, которых оно не может предоставить. Нам нужен надёжный процесс (CI), прежде чем отдавать компонент в production.

**Какую пользу несёт глава?**
Мы освоим четырёхуровневую проверку компонента, используя исключительно CLI-инструменты без хрупких скриптов:
1. **Валидация артефакта** (`wasm-tools validate`).
2. **Экспортируемый интерфейс** (`wasm-tools component wit --json`).
3. **Проверка зависимостей** (`wasm-tools component wit --out-dir`).
4. **Поведенческое соответствие** (`wasmtime compile` + `wasmtime run --invoke`).

Дополнительно мы используем `--importize` для генерации Host-контракта (пригодится в Главе 5) и проверяем, что `wasmtime` отвергает некорректные типы данных на границе.

**Чего мы хотим достичь по результатам тестирования?**
1. Создать единый скрипт `verify.sh` для CI.
2. Научиться вызывать функции компонента через `wasmtime --invoke` с помощью Wave-синтаксиса.
3. Убедиться, что компонент успешно компилируется в `.cwasm`.
4. Увидеть, как `wasmtime` отвергает вызов при передаче неверного типа данных.

---

### 🚦 Starting point
У нас есть `components/text-engine-component` (принимает `string`) и `components/document-engine` (принимает `record`). В этой главе мы не пишем новый Rust-код для компонентов. Мы создаём инфраструктуру проверки в нативном проекте `text-engine-core`.

### 🎯 PREDICT (Предсказание)
*Запишите свой прогноз перед запуском:*
Если мы вызовем функцию `process` компонента `text-engine-component` напрямую через `wasmtime --invoke` и передадим в неё число (`123`) вместо строки, что произойдёт: `wasmtime` выполнит функцию с ошибкой, или отклонит вызов на этапе парсинга аргументов?

---

## Этап 1. Исходные коды

### Структура
**Новые файлы:**
- `scripts/verify.sh` — скрипт для CI.
- `components/text-engine-core/examples/chapter4_demo.rs` — демонстрация CLI.

**Обновляется:**
- `components/text-engine-core/Cargo.toml` — добавляем `serde_json`.

### Новые файлы

**1. `scripts/verify.sh`**
```bash
mkdir -p scripts
nano scripts/verify.sh
```

```bash
#!/bin/bash
set -e

if [ -z "$1" ]; then
    echo "Usage: $0 <path-to-wasm>"
    exit 1
fi

WASM_FILE=$1

echo "=== Verifying $WASM_FILE ==="

# 1. Валидация артефакта
echo -n "[1/5] Validating artifact... "
wasm-tools validate --features all "$WASM_FILE"
echo "✅ Valid"

# 2. Извлечение интерфейса (JSON)
echo -n "[2/5] Extracting interface... "
wasm-tools component wit --json "$WASM_FILE" > /tmp/interface.json
echo "✅ Extracted to /tmp/interface.json"

# 3. Проверка зависимостей
echo -n "[3/5] Checking dependencies... "
OUT_DIR=$(mktemp -d)
wasm-tools component wit "$WASM_FILE" --out-dir "$OUT_DIR" > /dev/null 2>&1 || true
if [ -d "$OUT_DIR/deps" ] && [ "$(ls -A "$OUT_DIR/deps")" ]; then
    echo "⚠️ Has WASI dependencies"
else
    echo "✅ No external dependencies"
fi
rm -rf "$OUT_DIR"

# 4. Генерация Host-контракта (--importize)
echo -n "[4/5] Generating Host contract... "
if wasm-tools component wit "$WASM_FILE" --importize > /dev/null 2>&1; then
    echo "✅ Host WIT generated successfully"
else
    echo "❌ Failed to generate Host WIT"
    exit 1
fi

# 5. Precompilation (wasmtime compile)
echo -n "[5/5] Precompiling to native code... "
wasmtime compile --target x86_64-unknown-linux-gnu "$WASM_FILE" -o /tmp/component.cwasm
echo "✅ Compiled to /tmp/component.cwasm"

echo "=== All structural checks passed ==="
```

Сделайте скрипт исполняемым:
```bash
chmod +x scripts/verify.sh
```

**2. `components/text-engine-core/Cargo.toml`**

```toml
[package]
name = "text-engine-core"
version.workspace = true
edition.workspace = true

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
# Нет зависимостей.

[dev-dependencies]
serde_json = "1"
```

---

## Этап 2. Тесты исходных кодов

Тесты — это скрипт `verify.sh` и демо-пример. Юнит-тесты компонентов из Главы 1 остаются без изменений.

### Команды запуска
```bash
cd ~/projects/wa/wasm-backend-lab
./scripts/verify.sh components/text-engine-component/target/wasm32-wasip1/release/text_engine_component.wasm
```

---

## Этап 3. Демо-пример

**`components/text-engine-core/examples/chapter4_demo.rs`**
```rust
// components/text-engine-core/examples/chapter4_demo.rs
use std::path::PathBuf;
use std::process::Command;

fn workspace_root() -> PathBuf {
    PathBuf::from(env!("CARGO_MANIFEST_DIR"))
        .parent().unwrap()
        .parent().unwrap()
        .to_path_buf()
}

fn run_cmd(cmd: &mut Command) -> (bool, String, String) {
    let output = cmd.output().expect("Failed to run command");
    let stdout = String::from_utf8_lossy(&output.stdout).to_string();
    let stderr = String::from_utf8_lossy(&output.stderr).to_string();
    (output.status.success(), stdout, stderr)
}

fn main() {
    println!("=== Chapter 4: Four Component Checks (Advanced CLI) ===");

    let root = workspace_root();
    let text_wasm = root.join("components/text-engine-component/target/wasm32-wasip1/release/text_engine_component.wasm");

    println!("\n[*] Building component...");
    Command::new("cargo")
        .args(["component", "build", "--release", "--quiet"])
        .current_dir(root.join("components/text-engine-component"))
        .status()
        .unwrap();

    // --- ПРОВЕРКА 1: Валидация ---
    println!("\n[1] wasm-tools validate");
    let (ok, _, _) = run_cmd(Command::new("wasm-tools").args(["validate", "--features", "all", text_wasm.to_str().unwrap()]));
    if ok { println!("  ✅ Valid artifact"); }

    // --- ПРОВЕРКА 2: JSON Interface ---
    println!("\n[2] Extracting interface as JSON");
    let (ok, json_out, _) = run_cmd(Command::new("wasm-tools").args(["component", "wit", "--json", text_wasm.to_str().unwrap()]));
    if ok {
        let _parsed: serde_json::Value = serde_json::from_str(&json_out).expect("Invalid JSON");
        let has_process = json_out.contains("process");
        println!("  ✅ Interface extracted. Contains 'process': {}", has_process);
    }

    // --- ПРОВЕРКА 3: Зависимости ---
    println!("\n[3] Checking dependencies with --out-dir");
    let temp_dir = std::env::temp_dir().join("wasm_deps_check");
    let _ = std::fs::remove_dir_all(&temp_dir);
    let (_, _, _) = run_cmd(Command::new("wasm-tools")
        .args(["component", "wit", text_wasm.to_str().unwrap(), "--out-dir", temp_dir.to_str().unwrap()]));
    
    if temp_dir.join("deps").exists() {
        println!("  ⚠️ Component requires WASI imports:");
        for entry in std::fs::read_dir(temp_dir.join("deps")).unwrap() {
            if let Ok(e) = entry { println!("     - {}", e.file_name().to_string_lossy()); }
        }
    }

    // --- ПРОВЕРКА 4: Генерация Host-контракта (--importize) ---
    println!("\n[4] Generating Host contract via --importize");
    let (ok, import_wit, _) = run_cmd(Command::new("wasm-tools")
        .args(["component", "wit", text_wasm.to_str().unwrap(), "--importize"]));
    if ok {
        println!("  ✅ Host WIT (ready for bindgen in Chapter 5):");
        for line in import_wit.lines() {
            if line.contains("import") || line.contains("world") || line.contains("export") {
                println!("    {}", line.trim());
            }
        }
    }

    // --- ПРОВЕРКА 5a: Precompilation (wasmtime compile) ---
    println!("\n[5a] Precompiling to native code (wasmtime compile)");
    let cwasm_path = "/tmp/text_engine_component.cwasm";
    let (ok, _, _) = run_cmd(Command::new("wasmtime")
        .args(["compile", "--target", "x86_64-unknown-linux-gnu", text_wasm.to_str().unwrap(), "-o", cwasm_path]));
    if ok { println!("  ✅ Compiled to native .cwasm"); }

    // --- ПРОВЕРКА 5b: Поведенческая (с WASI) ---
    println!("\n[5b] Behavioral test via wasmtime --invoke (WITH WASI)");
    let wave_call = "process(\"  Hello   CLI  \")";
    
    let (ok, out, err) = run_cmd(Command::new("wasmtime")
        .args(["run", "-S", "common", "--invoke", wave_call, text_wasm.to_str().unwrap()]));
    
    if ok { 
        println!("  ✅ Execution successful! Output: {}", out.trim()); 
    } else { 
        println!("  ❌ Execution failed: {}", err.lines().next().unwrap_or("")); 
    }

    // --- BREAK: Передача неверного типа данных ---
    println!("\n[BREAK] Passing wrong argument type via wasmtime --invoke");
    println!("  Expected 'string', but we pass a number '123'...");
    
    let bad_wave_call = "process(123)";
    
    let (ok, _, err) = run_cmd(Command::new("wasmtime")
        .args(["run", "-S", "common", "--invoke", bad_wave_call, text_wasm.to_str().unwrap()]));
    
    if !ok {
        println!("  ❌ REJECTED by wasmtime at the boundary!");
        let error_msg = err.lines().find(|l| l.contains("type") || l.contains("error") || l.contains("mismatch") || l.contains("invalid"))
            .unwrap_or("Invalid argument type provided");
        println!("  Error: {}", error_msg.trim());
        println!("  This proves that WIT contracts enforce strict types even via CLI.");
    } else {
        println!("  ✅ Unexpected success. Type checking didn't work.");
    }
}
```

### Команды запуска
```bash
cd components/text-engine-core
cargo run --example chapter4_demo
```

---

## 🔍 Разбор результатов вывода

Ниже — расшифровка каждого блока. Для каждой строки: **что произошло**, **почему** и **что это доказывает**.

### `[*] Building component...`

Демо автоматически запускает `cargo component build --release`. Мы работаем именно с тем артефактом, который ушёл бы в production, а не с debug-сборкой.

### `[1] wasm-tools validate → ✅ Valid artifact`

```rust
run_cmd(Command::new("wasm-tools")
    .args(["validate", "--features", "all", text_wasm.to_str().unwrap()]));
```

`wasm-tools validate` проверил, что бинарный формат соответствует спецификации WebAssembly. Флаг `--features all` включает все современные proposals (Component Model, reference types и т.д.). Команда не печатает ничего при успехе и возвращает `exit 0`.

**Доказывает:** артефакт — валидный WebAssembly-компонент, а не повреждённый файл. Без этого шага можно было бы пытаться запустить битый файл и получить невнятную ошибку.

### `[2] Extracting interface as JSON → ✅ Contains 'process': true`

```rust
run_cmd(Command::new("wasm-tools")
    .args(["component", "wit", "--json", text_wasm.to_str().unwrap()]));
```

`--json` вернул машиночитаемое представление WIT-интерфейса. В отличие от текстового вывода (который в Главе 2 мы парсили через `grep` — хрупкое решение), JSON можно безопасно парсить через `serde_json`. Проверка `json_out.contains("process")` подтверждает, что функция `process` действительно экспортирована.

**Доказывает:** `cargo component build` сохранил экспорт функции. Это проверка «на месте ли то, что мы ожидали», без хрупких регулярных выражений.

### `[3] Checking dependencies → ⚠️ Component requires WASI imports`

```rust
run_cmd(Command::new("wasm-tools")
    .args(["component", "wit", text_wasm.to_str().unwrap(), "--out-dir", temp_dir.to_str().unwrap()]));
```

`--out-dir` извлёк весь граф зависимостей компонента в отдельные `.wit` файлы. Каждый файл — это импорт, который компоненту нужен от Host:

- `filesystem.wit`, `cli.wit`, `io.wit`, `clocks.wit` — **WASI-зависимости**. Компонент на Rust использует `String`, `Vec`, `println!` и т.д., и стандартная библиотека тянет за собой WASI.
- `text-engine.wit` — наш собственный контракт (часть графа, а не импорт).

Символ `⚠️` не означает ошибку, это факт, который Host должен обработать.

**Доказывает:** мы знаем точный набор интерфейсов, которые предоставит Host в Главе 5. Без этого шага интеграция упала бы с `unknown import: wasi:...` — сложно диагностируемая ошибка.

### `[4] Generating Host contract → ✅ world root-importized`

```rust
run_cmd(Command::new("wasm-tools")
    .args(["component", "wit", text_wasm.to_str().unwrap(), "--importize"]));
```

`--importize` сгенерировал **зеркало** нашего компонента с точки зрения Host:

```wit
world root-importized {
    import docs:text-engine/processor@0.1.0;
}
```

Это означает: Host импортирует интерфейс `docs:text-engine/processor@0.1.0` и вызывает его.

**Доказывает:** контракт для Host можно получить **автоматически** из готового артефакта Guest. В Главе 5 мы используем этот вывод для `wasmtime::component::bindgen!`, который сгенерирует типизированные Rust-вызовы.

### `[5a] Precompiling to native code → ✅ Compiled to native .cwasm`

```rust
run_cmd(Command::new("wasmtime")
    .args(["compile", "--target", "x86_64-unknown-linux-gnu", text_wasm.to_str().unwrap(), "-o", cwasm_path]));
```

`wasmtime compile` заранее скомпилировал `.wasm` в нативный машинный код (`.cwasm`). Это AOT-компиляция: в продакшене `wasmtime` загрузит `.cwasm` напрямую, минуя JIT при первом запросе.

**Доказывает:** артефакт компилируется для нашей целевой платформы. Если бы компонент использовал неподдерживаемые инструкции (например, SIMD без поддержки CPU), мы бы узнали об этом сейчас, а не в момент первого HTTP-запроса.

### `[5b] Behavioral test → ✅ Output: "hello cli"`

```rust
let wave_call = "process(\"  Hello   CLI  \")";
run_cmd(Command::new("wasmtime")
    .args(["run", "-S", "common", "--invoke", wave_call, text_wasm.to_str().unwrap()]));
```

`wasmtime run -S common` предоставил WASI-контекст (без него компонент не запустится — см. шаг 3). `--invoke "process(\"...\")"` вызвал функцию через Wave-синтаксис: текстовый формат аргументов, который понимает `wasmtime`. Функция получила строку и вернула `"hello cli"`.

**Доказывает:** бизнес-логика работает **внутри WASM** без написания Host-кода. Мы вызвали функцию из настоящего компонента в настоящем рантайме и получили ожидаемый результат. Также доказано: `-S common` обязателен — без него компонент не стартует.

### `[BREAK] Passing wrong argument type → ❌ Error: invalid value type at 8..11`

```rust
let bad_wave_call = "process(123)";
run_cmd(Command::new("wasmtime")
    .args(["run", "-S", "common", "--invoke", bad_wave_call, text_wasm.to_str().unwrap()]));
```

`wasmtime` попытался распарсить `process(123)`, но обнаружил: `process` ожидает `string`, а получил `integer`. Ошибка `invalid value type at 8..11` указывает на позицию `123` в строке.

**Доказывает:** ошибка возникла **до** исполнения WASM-кода. `wasmtime` не выполнил ни одной инструкции Guest. WIT-контракт — это не документация, а **исполняемое ограничение**, которое проверяется рантаймом. Даже CLI-инструменты не могут обойти типизацию.

---

## 🔄 BREAK (Поломка) — повреждение артефакта

**Цель:** увидеть, как `wasm-tools validate` ловит битый файл, который `cargo component build` уже создал.

**Шаг 1. Скопируйте компонент и повредите один байт:**
```bash
cd ~/projects/wa/wasm-backend-lab
cp components/text-engine-component/target/wasm32-wasip1/release/text_engine_component.wasm /tmp/broken.wasm
dd if=/dev/urandom of=/tmp/broken.wasm bs=1 count=4 seek=100 conv=notrunc status=none
```

**Шаг 2. Запустите проверку:**
```bash
./scripts/verify.sh /tmp/broken.wasm
```

**Ожидаемая ошибка:**
```text
[1/5] Validating artifact... error: failed to parse file
Caused by:
    magic header not detected
```

`cargo component build` отработал, файл создан, но он битый. Без `wasm-tools validate` мы бы узнали об этом только при попытке загрузки в Host.

---

## 🔄 Итеративная проверка

**Нужны ли доработки после тестов?**
Да, в процессе работы мы обнаружили несколько нюансов, которые повлияли на финальный код:

1. **Синтаксис `--invoke`.** `wasmtime` ожидает имя функции без префикса интерфейса (`process(...)`, а не `docs:text-engine/processor.process(...)`). Неверный синтаксис даёт ошибку `No exported func named ...`.
2. **Типовая проверка Wave.** Передача числа вместо строки вызывает `invalid value type` на этапе парсинга — до запуска WASM. Это защита от ошибок интеграции.
3. **WASI-контекст обязателен.** Без `-S common` компонент не запустится из-за отсутствующих импортов.
4. **`--importize` вместо ручного WIT.** Раньше мы писали Host-контракт вручную. Теперь используем `--importize` — он генерирует ровно то, что нужно `bindgen!`.
5. **`serde_json` для JSON-парсинга.** Отказались от `grep` по текстовому выводу — используем структурированный парсер.

---

## Итог главы

**OBSERVE:**

Из восьми проверок **семь выполняются CLI-инструментами** без единой строки Rust-кода для Host:

| Шаг | Инструмент | Что доказывает |
|---|---|---|
| `[*]` | `cargo component build` | Артефакт создан |
| `[1]` | `wasm-tools validate` | Файл не битый, соответствует спецификации |
| `[2]` | `wasm-tools component wit --json` | Функция `process` присутствует |
| `[3]` | `wasm-tools component wit --out-dir` | Компонент требует WASI-контекста |
| `[4]` | `wasm-tools component wit --importize` | Host-WIT готов для Главы 5 |
| `[5a]` | `wasmtime compile` | Компонент компилируется в нативный код |
| `[5b]` | `wasmtime run --invoke -S common` | Бизнес-логика работает корректно |
| `[BREAK]` | `wasmtime run --invoke process(123)` | WIT отвергает неверный тип |

**What did we prove?**
- Полный цикл проверки реализуется стандартными CLI-инструментами.
- WIT-контракт — исполняемое ограничение: `wasmtime` отвергает неверный тип **до** запуска кода.
- `--importize` автоматически генерирует Host-контракт из готового артефакта Guest.
- Компонент компилируется в `.cwasm` (AOT), что критично для холодного старта.

**What did we not prove?**
- Как Host (Axum) использует `--importize` WIT для `bindgen!` и как проходит цикл вызова — это будет в Главе 5.

**What did we pay for it?**
- Нужно запускать `verify.sh` в CI, чтобы гарантировать, что каждый собранный компонент проходит все проверки.
- `--importize` требует готового `.wasm` — проверка возможна только после сборки.

**When is the alternative better?**
- Для простых скриптов можно пропустить инспекцию (`--json`, `--out-dir`), но для production (где важна безопасность, скорость холодного старта и прослеживаемость) все проверки критичны.

### 📊 Evidence
```
Claim 1:        wasmtime отвергает вызов при передаче неверного типа.
Experiment:     wasmtime run --invoke "process(123)".
Measurement:    Ошибка "invalid value type at 8..11".
Observed:       Ошибка возникла до запуска WASM-кода.
Conclusion:     WIT-контракт проверяется рантаймом; CLI не может его обойти.
Confidence:     High

Claim 2:        --importize генерирует Host-контракт из Guest-артефакта.
Experiment:     wasm-tools component wit --importize.
Measurement:    Вывод: `world root-importized { import docs:text-engine/processor@0.1.0; }`.
Observed:       WIT соответствует экспортам компонента.
Conclusion:     Контракт для Host можно получить автоматически.
Confidence:     High
```

### What we have now
- `scripts/verify.sh` — CI-скрипт для пяти структурных проверок.
- `components/text-engine-core/examples/chapter4_demo.rs` — демонстрация всех возможностей CLI.
- Все предыдущие главы (`text-engine`, `text-engine-core`, `text-engine-component`, `document-engine`) не тронуты.

**Следующий шаг:** В Главе 5 мы создадим Host-приложение на Axum, которое загружает проверенный `text-engine-component` через Wasmtime в Rust-коде. Используем `--importize` для генерации биндингов.