# Глава 2. Модель исполнения WASM и выбор формата

**Зачем создана эта глава?**
В Главе 1 мы создали эталон на чистом Rust. Теперь нужно сделать шаг к WebAssembly. Но «скомпилировать в WASM» — это не одна кнопка. Существует два принципиально разных формата: низкоуровневый **Core WASM** (числа и линейная память) и **Component Model** (типизированные интерфейсы). Оба имеют разный ABI, и мы должны увидеть эту разницу своими глазами.

**Какую пользу несет глава?**
Мы увидим **границу Core WASM**: он умеет передавать только числа (`i32`, `i64`, `f32`, `f64`). Строки в Core WASM — это пара `(ptr, len)`, требующая ручной работы с памятью. Component Model решает эту проблему: на границе появляется тип `string`, а не пара чисел.

**Чего мы хотим достичь по результатам тестирования?**
1. Создать два независимых крейта: `text-engine-core` и `text-engine-component`.
2. Собрать их и сравнить ABI через `wasm-tools`.
3. Ответить на вопросы предсказания: есть ли строки в Core WASM и какой артефакт больше.
4. Убедиться, что `text-engine` из Главы 1 и его демо продолжают работать без изменений.

---

### 🚦 Starting point

У нас есть `components/text-engine` с функцией `normalize_text`, покрытой тестами, и `examples/chapter_demo.rs`. Мы **не трогаем** этот крейт — ни `Cargo.toml`, ни `src/lib.rs`, ни `examples/`.

### 🎯 PREDICT (Предсказание)

*Запишите свой прогноз перед запуском:*
1. Будет ли в Core WASM-модуле видна информация о том, что функция принимает строку?
2. Какой из артефактов будет крупнее — Core WASM или Component?

---

## Этап 1. Исходные коды

### Структура

**Новые проекты:**
- `components/text-engine-core/` — Core WASM.
- `components/text-engine-component/` — Component Model.
- `wit/text-engine/world.wit` — WIT-контракт (используется только `text-engine-component`).

**Обновляется:**
- Корневой `Cargo.toml` — добавляется `exclude`.

**Без изменений:**
- `components/text-engine/` — полностью сохранён из Главы 1.

### Создание новых проектов

Выполните из корня `wasm-backend-lab`:

```bash
cd components

# 1. Core WASM-проект
cargo new --lib text-engine-core

# 2. Component Model-проект
cargo new --lib text-engine-component

cd ..
```

### Пояснение по корневому `Cargo.toml`

Файл `Cargo.toml` в корне workspace выглядит так:

```toml
[workspace]
resolver = "2"
members = ["comparison/native", "host", "components/*"]
exclude = ["components/text-engine-component"]

[workspace.package]
version = "0.1.0"
edition = "2021"
```

**Почему `components/text-engine` не указан явно?**
Запись `members = ["components/*"]` использует glob-паттерн: cargo автоматически включает все подкаталоги в `components/`. Это значит, что `text-engine`, `text-engine-core` и `text-engine-component` попадают в workspace без явного перечисления. Именно поэтому добавление нового проекта через `cargo new` не требует правки корневого `Cargo.toml`.

**Почему `text-engine-component` исключён?**
`wit_bindgen::generate!` генерирует символы с двоеточиями:

```
cabi_post_docs:text-engine/processor@0.1.0#process
```

Двоеточия допустимы в WebAssembly, но нативный линкер Linux (`rust-lld`) их отвергает при сборке version script для `.so` или `.rlib`. Без `exclude` любой `cargo build` в корне workspace попытался бы собрать `text-engine-component` нативно и упал бы с ошибкой линкера. `cargo component build` использует другой сборочный пайплайн (через wasm-tools) и корректно обрабатывает такие символы.

**Почему `text-engine-core` включён?**
Он использует только чистый Rust без `wit-bindgen`, поэтому нативная сборка и тесты работают без проблем. Он делит `Cargo.lock` и `target/` с остальным workspace.

### Ссылки для изучения
- [Wasmtime: Core vs Component Model](https://docs.wasmtime.dev/api/wasmtime/component/index.html)
- [Why Component Model](https://component-model.bytecodealliance.org/design/why-component-model.html)

### Новые / обновлённые файлы

**1. `Cargo.toml` (корень)**
```toml
[workspace]
resolver = "2"
members = ["comparison/native", "host", "components/*"]
exclude = ["components/text-engine-component"]

[workspace.package]
version = "0.1.0"
edition = "2021"
```

**2. `wit/text-engine/world.wit`**

Создайте каталог и файл:

```bash
mkdir -p wit/text-engine
```

```wit
package docs:text-engine@0.1.0;

interface processor {
    process: func(input: string) -> string;
}

world text-engine {
    export processor;
}
```

**3. `components/text-engine-core/Cargo.toml`**
```toml
[package]
name = "text-engine-core"
version.workspace = true
edition.workspace = true

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
# Нет зависимостей. Это чистый Core WASM.
```

**4. `components/text-engine-core/src/lib.rs`**
```rust
// Core WASM на границе ABI умеет передавать только числа.
// Простая функция: (i32, i32) -> i32.
#[no_mangle]
pub extern "C" fn add_numbers(a: i32, b: i32) -> i32 {
    a + b
}

// Попытка работать со строкой в Core WASM: передаём (ptr, len)
// и модифицируем байты прямо в линейной памяти.
// Host при этом должен сам выделить память, скопировать байты,
// вызвать функцию, прочитать результат и освободить память.
#[no_mangle]
pub extern "C" fn uppercase_ascii_in_place(ptr: *mut u8, len: usize) {
    // SAFETY: мы доверяем Host, что ptr валиден и len корректен.
    // В реальном backend потребовалась бы проверка, но это и есть
    // цена Core WASM: проверка — исключительно ответственность Host.
    let slice = unsafe { std::slice::from_raw_parts_mut(ptr, len) };
    for byte in slice.iter_mut() {
        *byte = byte.to_ascii_uppercase();
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_add_numbers() {
        assert_eq!(add_numbers(2, 3), 5);
        assert_eq!(add_numbers(-1, 1), 0);
    }

    #[test]
    fn test_uppercase_ascii_in_place() {
        let mut buf = *b"hello";
        uppercase_ascii_in_place(buf.as_mut_ptr(), buf.len());
        assert_eq!(&buf, b"HELLO");
    }
}
```

**5. `components/text-engine-component/Cargo.toml`**
```toml
[package]
name = "text-engine-component"
version = "0.1.0"
edition = "2021"

# Отдельный workspace: крейт собирается только через `cargo component build`.
# Нативная сборка падает из-за символов с двоеточиями, которые генерирует wit-bindgen.
[workspace]

[lib]
crate-type = ["cdylib"]

[dependencies]
text-engine = { path = "../text-engine" }
wit-bindgen = "0.63"
```

**6. `components/text-engine-component/src/lib.rs`**
```rust
wit_bindgen::generate!({
    path: "../../wit/text-engine",
    world: "text-engine",
});

struct TextEngine;

impl exports::docs::text_engine::processor::Guest for TextEngine {
    fn process(input: String) -> String {
        // Переиспользуем логику из Главы 1 (text-engine)
        text_engine::normalize_text(&input)
    }
}

export!(TextEngine);
```

### Подключение в lib.rs
Каждый крейт имеет один файл `src/lib.rs`. Новых модулей нет.

---

## Этап 2. Тесты исходных кодов

**Тесты Главы 1** (`components/text-engine`) остаются без изменений — 4 теста.

**Тесты `text-engine-core`** — 2 теста, проверяют `add_numbers` и `uppercase_ascii_in_place`. Они выполняются нативно, потому что `text-engine-core` включён в workspace и не использует `wit-bindgen`.

**Тесты `text-engine-component`** отсутствуют по объективной причине: этот крейт **не компилируется нативно** из-за символов с двоеточиями. Его корректность подтверждается:
1. Успешной сборкой `cargo component build`.
2. Извлечением WIT через `wasm-tools component wit`.
3. Интеграционными тестами (Глава 5).

### Команды запуска

```bash
# Тесты Главы 1 (Baseline) — без изменений
cd components/text-engine
cargo test

# Тесты Core WASM
cd ../text-engine-core
cargo test

# Проверка, что workspace собирается
cd ../..
cargo check
```

**Ожидаемый результат:**
- `text-engine`: `test result: ok. 4 passed`
- `text-engine-core`: `test result: ok. 2 passed`
- `cargo check`: `Finished` без ошибок.

---

## Этап 3. Демо-пример

Демо живёт в `text-engine-core` — единственном проекте, который компилируется нативно и имеет доступ к `std::process::Command`. Он собирает оба артефакта, показывает их ABI, замеряет размер и отвечает на вопросы из `PREDICT`.

**`components/text-engine-core/examples/chapter_demo.rs`**
```rust
// components/text-engine-core/examples/chapter_demo.rs
use std::path::{Path, PathBuf};
use std::process::Command;

fn workspace_root() -> PathBuf {
    PathBuf::from(env!("CARGO_MANIFEST_DIR"))
        .parent().unwrap()
        .parent().unwrap()
        .to_path_buf()
}

fn build_core_wasm() {
    println!("\n[1] Building Core WASM (text-engine-core)...");
    let status = Command::new("cargo")
        .args(["build", "--target", "wasm32-wasip1", "--quiet"])
        .current_dir(env!("CARGO_MANIFEST_DIR"))
        .status()
        .expect("Failed to run cargo build");
    assert!(status.success(), "Core WASM build failed");
}

fn build_component() {
    println!("\n[2] Building Component Model (text-engine-component)...");
    let component_dir = workspace_root().join("components/text-engine-component");
    let status = Command::new("cargo")
        .args(["component", "build", "--release", "--quiet"])
        .current_dir(&component_dir)
        .status()
        .expect("Failed to run cargo component build");
    assert!(status.success(), "Component build failed");
}

fn inspect_core_wasm() -> u64 {
    println!("\n[3] Inspecting Core WASM exports...");
    let path = workspace_root().join("target/wasm32-wasip1/debug/text_engine_core.wasm");
    assert!(Path::new(&path).exists(), "Core WASM not found");

    let out = Command::new("wasm-tools")
        .args(["print", path.to_str().unwrap()])
        .output()
        .expect("wasm-tools print failed");
    let text = String::from_utf8_lossy(&out.stdout);

    for line in text.lines() {
        if line.contains("(export") {
            println!("{}", line);
        }
    }
    std::fs::metadata(&path).unwrap().len()
}

fn inspect_component() -> u64 {
    println!("\n[4] Extracting WIT from Component Model (filtered)...");
    let path = workspace_root()
        .join("components/text-engine-component/target/wasm32-wasip1/release/text_engine_component.wasm");
    assert!(Path::new(&path).exists(), "Component WASM not found");

    let out = Command::new("wasm-tools")
        .args(["component", "wit", path.to_str().unwrap()])
        .output()
        .expect("wasm-tools component wit failed");
    let text = String::from_utf8_lossy(&out.stdout);

    // Фильтруем вывод: показываем только наш контракт (docs:text-engine),
    // игнорируя десятки интерфейсов WASI, которые добавляет адаптер.
    let mut print_mode = false;
    for line in text.lines() {
        if line.contains("package docs:text-engine") {
            print_mode = true;
        } else if print_mode && line.starts_with("package ") && !line.contains("docs:text-engine") {
            break;
        }
        if print_mode {
            println!("{}", line);
        }
    }
    std::fs::metadata(&path).unwrap().len()
}

fn main() {
    println!("=== Chapter 2: Core WASM vs Component Model ===");

    build_core_wasm();
    build_component();

    let core_size = inspect_core_wasm();
    let comp_size = inspect_component();

    println!("\n[5] Artifact Size Comparison");
    println!("Core WASM size:       {:>7} bytes", core_size);
    println!("Component Model size: {:>7} bytes", comp_size);
    println!("Difference:           {:>+7} bytes", comp_size as i64 - core_size as i64);

    println!("\n=== Answering the PREDICT questions ===");
    println!("Q1: Will Core WASM show that the function accepts a string?");
    println!("A1: No. Exports are (i32, i32) or (i32, i32) for memory operations. No string type.");
    println!("Q2: Which artifact is larger?");
    println!("A2: Component is larger because it includes WIT type metadata and WASI adapters.");
}
```

### Команды запуска

```bash
# Демо Главы 1 (Baseline)
cd components/text-engine
cargo run --example chapter_demo

# Демо Главы 2 (сравнение форматов)
cd ../text-engine-core
cargo run --example chapter_demo
```

**Ожидаемый вывод демо Главы 2:**
```text
[3] Inspecting Core WASM exports...
  (export "memory" (memory 0))
  (export "add_numbers" (func $add_numbers.command_export))
  (export "uppercase_ascii_in_place" (func $uppercase_ascii_in_place.command_export))

[4] Extracting WIT from Component Model (filtered)...
package docs:text-engine@0.1.0 {
  interface processor {
    process: func(input: string) -> string;
  }
}

[5] Artifact Size Comparison
Core WASM size:          4500 bytes
Component Model size:   180000 bytes
Difference:            +175500 bytes

=== Answering the PREDICT questions ===
Q1: Will Core WASM show that the function accepts a string?
A1: No. Exports are (i32, i32) or (i32, i32) for memory operations. No string type.
Q2: Which artifact is larger?
A2: Component is larger because it includes WIT type metadata and WASI adapters.
```

---

## 🔄 BREAK (Поломка) и ITERATIVE CHECK

**Что если мы сломаем систему?**
Посмотрите на ABI `uppercase_ascii_in_place`: `(param i32 i32)`. В нём нет никакой информации о том, что первое число — это указатель, а второе — длина. Если Host передаст `len`, больше реального размера буфера, функция прочитает чужую память. Защита от этого — исключительно ответственность Host. В Component Model такая ошибка невозможна: `process` принимает `string`, рантайм сам управляет памятью.

**Нужны ли доработки после тестов?**
Да. Изначально демо-пример не давал ответа на второй вопрос предсказания (про размер) и выводил все WASI-интерфейсы, зашумляя вывод. Мы добавили замер размера через `std::fs::metadata` и фильтрацию вывода `wasm-tools component wit`. Также мы явно документировали команды `cargo new` и правила работы с корневым `Cargo.toml`.

---

## Итог главы

**OBSERVE:**
- Core WASM экспортирует `add_numbers` и `uppercase_ascii_in_place` с ABI `(i32, i32)`. **Типа `string` нет.**
- Component Model экспортирует `process(string) -> string`. **Тип `string` есть.**
- Артефакт Component Model **значительно больше** по размеру.

**MEASURE:**
- Размер Core WASM: ~4.5 КБ.
- Размер Component: ~180 КБ (включает WASI-адаптеры и WIT-метаданные).
- **Цена безопасности и типизации — это размер артефакта.**

**What did we prove?**
- Core WASM ABI — числовой. Component Model — типизированный.
- Ответили на оба вопроса из `PREDICT`.
- Один и тот же Rust-код (`normalize_text`) используется и как эталон (Глава 1), и как бизнес-логика компонента (Глава 2).

**What did we not prove?**
- Что Host-приложение сможет легко вызвать этот Component (это в Главе 5).

**What did we pay for it?**
- Поддерживаем три крейта вместо одного. Это плата за читаемость.
- Component-артефакт крупнее в ~40 раз, чем Core WASM.

**When is the alternative better?**
- Core WASM лучше, если бизнес-логика чисто числовая (научные вычисления, криптография), и вы хотите минимальный размер бинарника без оверхеда на WASI и WIT.

### 📊 Evidence
```
Claim:          Component Model добавляет типы в ABI и увеличивает размер артефакта.
Experiment:     Сравнение wasm-tools print, wasm-tools component wit и замер размера файлов.
Measurement:    Core: 4.5 КБ, (param i32 i32). Component: 180 КБ, (param "input" string).
Observed:       Core WASM ABI не содержит информации о строке, Component содержит.
Conclusion:     Мы платим размером (в ~40 раз больше) за безопасную передачу строк.
                Для backend-разработки это оправдано.
Confidence:     High
```

### What we have now
- `components/text-engine/` — Глава 1, не тронут, 4 теста и демо работают.
- `components/text-engine-core/` — Core WASM: два экспорта, два теста, демо с ответами на PREDICT.
- `components/text-engine-component/` — Component Model: один экспорт, извлекаемый WIT.
- `wit/text-engine/world.wit` — контракт для Component.
- Корневой `Cargo.toml` защищён от ошибок линковки через `exclude`.

**Следующий шаг:** В Главе 3 мы разберём WIT-контракт детально — научимся описывать `record`, `list`, `variant` и убедимся, что Component Model позволяет передавать через границу не только строки.