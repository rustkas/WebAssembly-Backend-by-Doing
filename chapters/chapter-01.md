Вот обновленный текст Главы 1, написанный в строгом соответствии с методикой книги и вашими требованиями. В начале главы четко обозначены цели и польза, а исходные коды приведены в финальном, рабочем виде.

---

# Глава 1. Первая чистая вычислительная функция (Baseline)

**Зачем создана эта глава?**
Прежде чем добавлять WebAssembly, сложные контракты (WIT) и накладные расходы на среду исполнения (Wasmtime), мы должны создать **абсолютную точку отсчёта**. Если мы сразу начнем писать WASM-компоненты, то при возникновении ошибки или замедления мы не сможем понять, где именно она возникла: в нашем алгоритме или в механизмах пересечения границы Host/Guest.

**Какую пользу несет глава?**
Мы изолируем бизнес-логику (нормализацию текста) в чистом Rust. Это даст нам **Baseline (эталон)** — эталонную реализацию, которая работает с максимальной скоростью и минимальным потреблением памяти, не обремененная никакими накладными расходами. К этому эталону мы будем возвращаться на протяжении всей книги для сравнения.

**Чего мы хотим достичь по результатам тестирования?**
1. Написать чистую функцию, которая корректно обрабатывает текст (удаляет лишние пробелы, приводит к нижнему регистру).
2. Покрыть функцию модульными тестами, включая проверку граничных случаев (пустые строки, переносы, спецсимволы).
3. Создать автономный демо-пример, который наглядно показывает работу алгоритма на разных входных данных.

---

### 🚦 Starting point
Мы находимся в папке `components/text-engine`. Ранее здесь был сгенерирован каркас для WASM-компонента, но в этой главе мы **отключаем** WASM-зависимости, чтобы сосредоточиться на чистом Rust.

### 🎯 PREDICT (Предсказание)
*Перед запуском кода запишите свой прогноз:* 
Будет ли WASM-версия этой же функции работать медленнее, когда мы добавим её в следующих главах? Если да, то почему?

---

## Этап 1. Исходные коды

### Структура
*   `components/text-engine/Cargo.toml` — обновлён (настроен тип сборки).
*   `components/text-engine/src/lib.rs` — обновлён (чистая функция вместо WASM-экспорта).
*   `components/text-engine/examples/chapter_demo.rs` — создан (демо-пример).

### Новые / обновлённые файлы

**1. `components/text-engine/Cargo.toml`**
```toml
[package]
name = "text-engine"
version.workspace = true
edition.workspace = true

[lib]
# cdylib нужен для будущей сборки WASM-компонента, 
# rlib — для запуска юнит-тестов и демо-примеров.
crate-type = ["cdylib", "rlib"] 

[dependencies]
# Зависимостей нет. Мы используем только стандартную библиотеку Rust.
```

**2. `components/text-engine/src/lib.rs`**
```rust
// components/text-engine/src/lib.rs

/// Нормализует текст: удаляет лишние пробелы по краям,
/// приводит к нижнему регистру и убирает множественные пробелы внутри.
pub fn normalize_text(input: &str) -> String {
    // Оптимизация: заранее выделяем память под результат,
    // чтобы избежать частых реаллокаций. Это важно для будущих замеров памяти.
    let mut result = String::with_capacity(input.len());
    
    let mut first = true;
    for word in input.split_whitespace() {
        if !first {
            result.push(' ');
        }
        // Приводим слово к нижнему регистру
        let lower = word.to_lowercase();
        result.push_str(&lower);
        first = false;
    }
    
    result
}
```

### Подключение в lib.rs
Модулей нет, вся логика находится в корне `lib.rs`.

---

## Этап 2. Тесты исходных кодов

Поскольку мы пишем чистый Rust, используем стандартный встроенный фреймворк тестирования.

**`components/text-engine/src/lib.rs`** (добавляем в конец файла)
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_normalize_basic() {
        assert_eq!(normalize_text("Hello World"), "hello world");
    }

    #[test]
    fn test_normalize_extra_spaces() {
        // Проверяем удаление лишних пробелов
        assert_eq!(normalize_text("  Hello   World  "), "hello world");
    }

    #[test]
    fn test_normalize_empty() {
        // Проверяем граничные случаи
        assert_eq!(normalize_text(""), "");
        assert_eq!(normalize_text("   "), "");
    }

    #[test]
    fn test_normalize_newlines() {
        // Проверяем работу с переносами строк
        assert_eq!(normalize_text("Hello\nWorld\r\n"), "hello world");
    }
}
```

### Команды запуска

```bash
# Перейдите в папку с компонентом
cd components/text-engine

# Запустите тесты
cargo test
```

**Ожидаемый результат:**
```
running 4 tests
test tests::test_normalize_basic ... ok
test tests::test_normalize_empty ... ok
test tests::test_normalize_extra_spaces ... ok
test tests::test_normalize_newlines ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out;
```

---

## Этап 3. Демо-пример

Создадим автономный пример, который запускает нашу функцию и демонстрирует её работу на разных входных данных. Обратите внимание на выделенную функцию `print_result`: это правило проекта — изолировать вывод строк, чтобы избежать проблем с неявными возвращаемыми типами в Rust.

**`components/text-engine/examples/chapter_demo.rs`**
```rust
// components/text-engine/examples/chapter_demo.rs
use text_engine::normalize_text;

// Выносим вывод строк в отдельную функцию для безопасности типов
fn print_result(input: &str) {
    let output = normalize_text(input);
    println!("Input:  {:?}", input);
    println!("Output: {:?}", output);
    println!("---");
}

fn main() {
    let inputs = vec![
        "  Hello   WASM   World!  ",
        "RUST \n is \r\n AWESOME",
        "   ",
        "",
        "!!! ???", // Edge case: только спецсимволы
    ];

    println!("=== Text Engine Baseline Demo ===");
    for input in inputs {
        print_result(input);
    }
}
```

### Команды запуска демо-примера:
```bash
cd components/text-engine
cargo run --example chapter_demo
```

**Ожидаемый результат:**
```
=== Text Engine Baseline Demo ===
Input:  "  Hello   WASM   World!  "
Output: "hello wasm world!"
---
Input:  "RUST \n is \r\n AWESOME"
Output: "rust is awesome"
---
Input:  "   "
Output: ""
---
Input:  ""
Output: ""
---
Input:  "!!! ???"
Output: "!!! ???"
---
```

---

## 🔄 BREAK (Поломка) и ITERATIVE CHECK

**Что если мы сломаем систему?**
Давайте намеренно сломаем тест, чтобы убедиться, что `cargo test` действительно защищает нас. Измените в `lib.rs` в функции `normalize_text` строку `result.push_str(&lower);` на `result.push_str(&word.to_uppercase());` (то есть сделайте текст заглавными).

Запустите `cargo test`. Вы увидите, что все 4 теста упали (`FAILED`). Это доказывает, что наша тестовая инфраструктура работает. Верните код обратно (`lower`), чтобы тесты снова прошли.

**Нужны ли доработки после тестов?**
Изначально в демо-примере использовался `format!` в хвостовой позиции цикла, что могло вызвать ошибку типов. Мы вынесли логику вывода в `print_result`, тем самым сделав код надежным.

---

## Итог главы

**OBSERVE (Наблюдение):**
*   `cargo test` проходит успешно (4/4).
*   Демо-пример корректно обрабатывает пустые строки, лишние пробелы, переносы и спецсимволы.

**What did we prove?**
*   Алгоритм нормализации работает корректно.
*   Мы создали эталонный код, к которому будем возвращаться.

**What did we not prove?**
*   Мы ничего не доказали про WebAssembly. Мы пока не собирали `.wasm` файл.

**What did we pay for it?**
*   Мы потратили время на покрытие тестами, без которых можно было бы обойтись в простом скрипте, но без них невозможно построение надежного backend.

**When is the alternative better?**
*   Всегда, когда бизнес-логике не нужна изоляция, ограничение прав (capabilities) или независимое обновление. Чистый Rust будет работать быстрее и потреблять меньше памяти.

### 📊 Evidence
```
Claim:          Функция нормализации текста на чистом Rust работает корректно и покрыта тестами.
Experiment:     Запуск cargo test и демо-примера (cargo run --example).
Measurement:    4/4 теста пройдены, 5 сценариев в демо выполнены без паники.
Observed:       Функция возвращает корректно отформатированный текст.
Conclusion:     Baseline для будущих сравнений создан. Код готов к интеграции в WASM.
Confidence:     High
```

### What we have now
У нас есть чистая, протестированная функция `normalize_text`, которая работает как обычная Rust-библиотека. 

**Следующий шаг:** В Главе 2 мы вернём `wit-bindgen`, обернём эту функцию в WASM-компонент и заставим Host вызывать её. Именно тогда мы сможем проверить наше предсказание о том, насколько WASM медленнее чистого Rust.