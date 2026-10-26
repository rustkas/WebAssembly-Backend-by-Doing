# Глава 3. WIT-контракт: contract before implementation

**Зачем создана эта глава?**
В Главе 2 мы убедились, что Component Model передает типы, а Core WASM — только числа. Но откуда берется информация о типах? Ответ — из WIT-контракта (WebAssembly Interface Types). WIT — это не просто описание функции. Это **строгий архитектурный контракт**. Если в традиционном backend вы можете передать JSON с неправильными полями и узнать об ошибке только во время выполнения (runtime), то WIT проверяет совместимость компонентов **до того, как они будут запущены**.

**Какую пользу несет глава?**
Мы поймем принцип **"Contract before implementation"** (Контракт до реализации). Мы создадим компонент `document-engine`, который принимает сложный тип `record document`. На практических поломках мы увидим, что WIT и `bindgen!` не позволяют собрать код, если реализация расходится с контрактом. Ошибки отсекаются на этапе компиляции.

**Чего мы хотим достичь по результатам тестирования?**
1. Создать новый WIT-контракт с высокоуровневыми типами (`record`).
2. Реализовать компонент, строго следующий этому контракту.
3. Провести три негативных эксперимента: изменение типа в WIT, подмену контракта в Host и удаление WIT-файла.

---

### 🚦 Starting point
У нас есть `components/text-engine-component` (принимает `string`) и `components/text-engine-core` (Core WASM). Мы **не трогаем** их. Мы создадим новый проект для `document-engine`.

### 🎯 PREDICT (Предсказание)
*Запишите свой прогноз перед запуском:*
1. Если мы изменим тип возвращаемого значения в WIT, но забудем изменить его в Rust, что произойдет: ошибка при сборке или падение во время выполнения?
2. Если Host-приложение ожидает `text-engine`, а мы подменим контракт на `document-engine`, на каком этапе это сломается?

---

## Этап 1. Исходные коды

### Структура
**Новые проекты:**
- `components/document-engine/` — новый компонент.
- `wit/document-engine/world.wit` — новый контракт.

**Обновляется:**
- Корневой `Cargo.toml` — добавляется `exclude` для нового компонента.

**Без изменений:**
- `components/text-engine/`, `components/text-engine-core/`, `components/text-engine-component/`.

### Создание новых проектов и каталогов

Выполните из корня `wasm-backend-lab`:

```bash
# 1. Создаем каталог для WIT-контракта
mkdir -p wit/document-engine

# 2. Создаем новый крейт для компонента
cd components
cargo new --lib document-engine
cd ..
```

### Обновление корневого `Cargo.toml`

Поскольку `document-engine` использует `wit-bindgen`, он не должен собираться нативно (из-за двоеточий в символах). Исключаем его из рабочего пространства:

```toml
# Cargo.toml
[workspace]
resolver = "2"
members = ["comparison/native", "host", "components/*"]
exclude = ["components/text-engine-component", "components/document-engine"]

[workspace.package]
version = "0.1.0"
edition = "2021"
```

### Новые файлы

**1. `wit/document-engine/world.wit`**
```wit
package docs:document-engine@0.1.0;

interface processor {
    // Новый тип данных: запись (record)
    record document {
        title: string,
        content: string,
    }

    // Функция принимает документ и возвращает строку
    process: func(doc: document) -> string;
}

world document-engine {
    export processor;
}
```

**2. `components/document-engine/Cargo.toml`**
```toml
[package]
name = "document-engine"
version = "0.1.0"
edition = "2021"

# Отдельный workspace, как и у text-engine-component
[workspace]

[lib]
crate-type = ["cdylib"]

[dependencies]
wit-bindgen = "0.63"
```

**3. `components/document-engine/src/lib.rs`**
```rust
wit_bindgen::generate!({
    path: "../../wit/document-engine",
    world: "document-engine",
});

struct DocumentEngine;

impl exports::docs::document_engine::processor::Guest for DocumentEngine {
    fn process(doc: exports::docs::document_engine::processor::Document) -> String {
        format!("Document '{}': {}", doc.title, doc.content)
    }
}

export!(DocumentEngine);
```

---

## Этап 2. Тесты исходных кодов

В этой главе мы не пишем юнит-тесты для `document-engine`, так как он компилируется только под WASM. Наша цель — проверить архитектурную совместимость.

### Команды запуска

```bash
# Проверяем, что старые тесты не сломались
cd ~/projects/wa/wasm-backend-lab/components/text-engine
cargo test

# Проверяем сборку нового компонента
cd ../document-engine
cargo component build --release

# Возвращаемся в корень
cd ../..
```

---

## Этап 3. Демо-пример

Демо-скрипт извлекает WIT-контракты из обоих компонентов и симулирует ситуацию: "У нас есть Host, который ждет строку, и мы пытаемся подсунуть ему новый `document-engine`".

**`components/document-engine/examples/chapter_demo.rs`**
```rust
// components/document-engine/examples/chapter_demo.rs
use std::path::{Path, PathBuf};
use std::process::Command;

fn workspace_root() -> PathBuf {
    PathBuf::from(env!("CARGO_MANIFEST_DIR"))
        .parent().unwrap()
        .parent().unwrap()
        .to_path_buf()
}

fn extract_wit(wasm_path: &Path) -> String {
    let out = Command::new("wasm-tools")
        .args(["component", "wit", wasm_path.to_str().unwrap()])
        .output()
        .expect("Failed to run wasm-tools component wit");
    
    let text = String::from_utf8_lossy(&out.stdout).to_string();
    
    let mut filtered = String::new();
    let mut in_package = false;
    for line in text.lines() {
        if line.contains("package docs:") {
            in_package = true;
        } else if in_package && line.starts_with("package ") {
            break;
        }
        if in_package {
            filtered.push_str(line);
            filtered.push('\n');
        }
    }
    filtered
}

fn main() {
    println!("=== Chapter 3: Contract before implementation ===");

    let root = workspace_root();
    
    // 1. Собираем оба компонента
    println!("\n[1] Building components...");
    Command::new("cargo")
        .args(["component", "build", "--release", "--quiet"])
        .current_dir(root.join("components/text-engine-component"))
        .status().expect("Failed to build text-engine");
    
    Command::new("cargo")
        .args(["component", "build", "--release", "--quiet"])
        .current_dir(root.join("components/document-engine"))
        .status().expect("Failed to build document-engine");

    // 2. Извлекаем контракты
    println!("\n[2] Extracting WIT contracts...");
    
    let text_wasm = root.join("components/text-engine-component/target/wasm32-wasip1/release/text_engine_component.wasm");
    let doc_wasm = root.join("components/document-engine/target/wasm32-wasip1/release/document_engine.wasm");

    let text_wit = extract_wit(&text_wasm);
    let doc_wit = extract_wit(&doc_wasm);

    println!("--- Contract of text-engine (Host expects this) ---");
    println!("{}", text_wit);
    
    println!("--- Contract of document-engine (What we have) ---");
    println!("{}", doc_wit);

    // 3. Негативный эксперимент (Симуляция)
    println!("\n[3] Negative Experiment: Trying to link Host (expects string) with Guest (expects document)...");
    
    let host_expects_string = text_wit.contains("process: func(input: string) -> string");
    let guest_provides_document = doc_wit.contains("process: func(doc: document) -> string");

    if host_expects_string && guest_provides_document {
        println!("❌ REJECTED: The Guest provides 'document', but the Host expects 'string'.");
        println!("This error happened BEFORE runtime. WIT enforced the contract.");
    } else {
        println!("✅ SUCCESS: Contracts match.");
    }
}
```

### Команды запуска

```bash
cd components/document-engine
cargo run --example chapter_demo
```

---

## 🔄 BREAK (Поломка) — детальные шаги

Проведем три поломки, чтобы увидеть систему в действии.

### 📍 Поломка №1: Несовместимый тип в WIT

**Цель:** убедиться, что Rust-компилятор не даст собрать код, если WIT противоречит реализации.

**Шаг 1. Перейдите в каталог компонента:**
```bash
cd ~/projects/wa/wasm-backend-lab/components/document-engine
```

**Шаг 2. Откройте WIT-файл и измените тип возвращаемого значения `process` со `string` на `u32`:**
```bash
nano ../../wit/document-engine/world.wit
```

```wit
    process: func(doc: document) -> u32;
```

**Шаг 3. Попробуйте собрать:**
```bash
cargo component build --release
```

**Ожидаемая ошибка:**
```text
error[E0053]: method `process` has an incompatible type for trait
  --> src/lib.rs:10:77
   |
10 |     fn process(doc: ...::Document) -> String {
   |                                       ^^^^^^ expected `u32`, found `String`
   |
   = note: expected signature `fn(Document) -> u32`
              found signature `fn(Document) -> String`
```

*Урок:* `bindgen!` генерирует трейт с типом `u32`. Наша реализация возвращает `String`. Компилятор не пропускает несовпадение.

**Шаг 4. Верните `string` обратно и пересоберите:**
```bash
nano ../../wit/document-engine/world.wit
# Вернуть: process: func(doc: document) -> string;
cargo component build --release
```

---

### 📍 Поломка №2: Попытка передать `document` туда, где ожидается `string`

**Цель:** увидеть, что `bindgen!` генерирует **новый тип**, и старые имена исчезают.

**Шаг 1. Перейдите в `host/`:**
```bash
cd ~/projects/wa/wasm-backend-lab/host
```

**Шаг 2. Откройте `host/src/main.rs` и замените в `bindgen!` путь на `document-engine`:**
```bash
nano src/main.rs
```

```rust
wasmtime::component::bindgen!({
    path: "../wit/document-engine",
    world: "document-engine",
});
```

**Шаг 3. Попробуйте собрать `host`:**
```bash
cargo check
```

**Ожидаемая ошибка:**
```text
error[E0433]: cannot find type `TextEngine` in this scope
  --> host/src/main.rs:56:26
   |
56 |     let bindings = match TextEngine::instantiate(...) {
   |                          ^^^^^^^^^^ use of undeclared type `TextEngine`

error[E0599]: no method named `docs_text_engine_processor` found for type `!` in the current scope
  --> host/src/main.rs:66:10
   |
66 |         .docs_text_engine_processor()
   |          ^^^^^^^^^^^^^^^^^^^^^^^^^^ method not found in `!`
```

*Урок:* Макрос `bindgen!` сгенерировал тип `DocumentEngine`. Тип `TextEngine` **больше не существует**. Все обращения к нему — ошибка.

**Шаг 4. Верните `bindgen!` обратно и пересоберите:**
```bash
nano src/main.rs
# Вернуть: path: "../wit/text-engine", world: "text-engine",
cargo check
```

---

### 📍 Поломка №3: Удаление WIT-файла (устаревший кэш)

**Цель:** увидеть, как `bindgen!` читает WIT-файлы при компиляции и как rust-analyzer кэширует результат.

**Шаг 1. Временно переименуйте WIT-файл:**
```bash
cd ~/projects/wa/wasm-backend-lab
mv wit/text-engine/world.wit wit/text-engine/world.wit.bak
```

**Шаг 2. Запустите `cargo check` в `host/`:**
```bash
cd host
cargo check
```

**Ожидаемая ошибка:**
```text
error: failed to read path for WIT [/home/.../wit/text-engine]
Caused by: No such file or directory (os error 2)
```

**Шаг 3. Верните файл обратно и пересоберите:**
```bash
mv ../wit/text-engine/world.wit.bak ../wit/text-engine/world.wit
cargo check
```
*Ожидаемый результат:* `Finished` без ошибок.

**Шаг 4. Воспроизведите проблему устаревшего кэша rust-analyzer:**
1. Снова переименуйте файл:
   ```bash
   mv ../wit/text-engine/world.wit ../wit/text-engine/world.wit.bak
   ```
2. В VS Code откройте `host/src/main.rs` — увидите красные подчёркивания.
3. Верните файл:
   ```bash
   mv ../wit/text-engine/world.wit.bak ../wit/text-engine/world.wit
   ```
4. **Проблема:** rust-analyzer может **не заметить** восстановления. Красные подчёркивания остаются, хотя `cargo check` проходит.
5. **Решение:** `Ctrl+Shift+P` → `rust-analyzer: Restart server`. (Или `cargo clean -p host`).

*Урок:* `bindgen!` читает WIT-файл при компиляции. rust-analyzer кэширует результат. Файлы `.wit` не входят в `.rs`-граф. При перемещении WIT-файлов перезапускайте rust-analyzer.

---

### 📋 Чек-лист восстановления

После всех поломок проверьте, что проект в исходном состоянии:

```bash
cd ~/projects/wa/wasm-backend-lab

ls wit/text-engine/world.wit
ls wit/document-engine/world.wit

find . -name "*.bak" | grep -v target

cd host && cargo check && cd ..
cd components/text-engine-component && cargo component build --release --quiet && cd ../..
cd components/document-engine && cargo component build --release --quiet && cd ../..
```

Все команды должны завершиться успешно.

---

## Итог главы

**OBSERVE:**
- WIT-файл для `document-engine` описывает сложный тип `record`.
- `bindgen!` генерирует Rust-типы и трейты на основе WIT.
- Изменение WIT приводит к ошибке компиляции Rust-кода, если реализация расходится с контрактом.

**What did we prove?**
- WIT — это строгий контракт, проверяемый на этапе компиляции.
- Ошибки несовместимости отлавливаются до запуска.
- `bindgen!` полностью пересоздает типы при смене контракта.

**What did we not prove?**
- Как Host (Axum) обрабатывает `record` при передаче по HTTP.

**What did we pay for it?**
- Мы обязаны обновлять WIT *до* написания кода. Это требует дисциплины.
- Кэш rust-analyzer требует перезапуска после изменения WIT.

**When is the alternative better?**
- Если вы используете динамический JSON повсюду, вы не платите за WIT, но получаете ошибки в рантайме, которые в WASM могут привести к панике (trap).

### 📊 Evidence
```
Claim:          WIT гарантирует, что несовместимые типы не смогут быть слинкованы.
Experiment:     Изменение типа в WIT (Поломка №1) и подмена контракта в Host (Поломка №2).
Measurement:    Ошибки E0053, E0433, E0599 на этапе `cargo check`.
Observed:       Компилятор не пропускает код, расходящийся с WIT-контрактом.
Conclusion:     WIT переносит проверку типов из рантайма в этап компиляции.
                Это делает систему надежнее.
Confidence:     High
```

### What we have now
У нас есть два независимых компонента (`text-engine-component` и `document-engine`), каждый со своим WIT-контрактом. Мы на практике увидели, что WIT и `bindgen!` защищают архитектуру от случайных несовместимостей.

**Следующий шаг:** В Главе 4 мы вернемся к Host-приложению (Axum) и научимся передавать в него данные, чтобы в Главе 5 наконец-то запустить Wasmtime и вызвать компонент.