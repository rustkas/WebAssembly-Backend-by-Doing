# WebAssembly Backend by Doing
### Build a controlled runtime for WebAssembly Components

> **Сквозной вопрос:** *How do I build a backend around WebAssembly Components?*

**Обещание книги:** К концу книги вы сами построите работающий `wasm-backend-lab`, в котором WASM-компонент является управляемой единицей исполнения — с контрактом, capabilities, limits, lifecycle и возможностью горячей замены.

---

## 📖 О книге

Это практическое руководство по построению бэкенда вокруг WebAssembly-компонентов. Книга построена на методике **learning by doing**. Мы не просто изучаем теорию Component Model; мы шаг за шагом строим реальный проект `wasm-backend-lab`.

**Три принципа книги:**
1. **Don't explain the architecture** before you have built the mechanism that makes it necessary.
2. **Every important architectural claim** must be turned into something you can build, run, break, observe, and measure.
3. **The book should not tell you that WASM is useful.** It should put you in situations where WASM's advantages and disadvantages become observable.

Мы используем стек **Rust + Axum + Wasmtime + WIT**. 

---
## 📖 Исходный код книги

[Source code]([./chapters/chapter-01.md](https://github.com/rustkas/wasm-backend-lab))

---

## 🧪 Методика: PREDICT → RUN → OBSERVE → BREAK → FIX → MEASURE → UNDERSTAND → Evidence

Каждая глава заставляет вас не просто читать код, а проходить цикл:
* **PREDICT:** Записать ожидания перед запуском.
* **RUN:** Запустить код.
* **OBSERVE:** Увидеть, что произошло в реальности.
* **BREAK:** Сломать систему отрицательным экспериментом (negative scenario).
* **FIX:** Починить, добавив нужный слой абстракции.
* **MEASURE:** Сравнить с baseline (чистый Rust, Docker или другие альтернативы).
* **UNDERSTAND:** Объяснить, почему это работает именно так.
* **Evidence:** Зафиксировать Claim, Experiment и Confidence в конце главы.

---

## 📚 Оглавление

Тексты глав находятся в папке [`chapters/`](./chapters/).

### Предисловие. Настраиваем проект
- [Предисловие. Настраиваем проект](./chapters/chapter-00.md)
  *(Цель: работающий каркас `wasm-backend-lab` с CI и фиксацией версий)*

### Часть I. Foundation — «Как вообще запустить WASM-компонент из backend?»
- [Глава 1. Первая чистая вычислительная функция](./chapters/chapter-01.md)
- [Глава 2. Модель исполнения WASM и выбор формата](./chapters/chapter-02.md)
- [Глава 3. WIT-контракт: contract before implementation](./chapters/chapter-03.md)
- [Глава 4. Сборка компонента и четыре проверки](./chapters/chapter-04.md)
- [Глава 5. Axum + Wasmtime: работающий backend](./chapters/chapter-05.md)

### Часть II. Data, state, time — «Что пересекает границу и когда оно живёт?»
- [Глава 6. Host ↔ guest boundary: три класса данных и ошибки](./chapters/chapter-06.md)
- [Глава 7. State: operational model and ownership](./chapters/chapter-07.md)
- [Глава 8. Time control: fuel, epoch, cancellation](./chapters/chapter-08.md)

### Часть III. What can a guest do? — «Что именно разрешено компоненту?»
- [Глава 9. Host API design: Capabilities (Files, Network, Secrets)](./chapters/chapter-09.md)
- [Глава 10. Memory limits and isolation](./chapters/chapter-10.md)

### Часть IV. What if the guest misbehaves? — «Что именно умирает, когда что-то падает?»
- [Глава 11. Failure semantics and Cancellation](./chapters/chapter-11.md)
- [Глава 12. Observability: logs, metrics, traces](./chapters/chapter-12.md)

### Часть V. What about data at scale? — «Как работать с большими данными без копирования?»
- [Глава 13. Resources: lifecycle and ownership](./chapters/chapter-13.md)
- [Глава 14. Streams and the memory path](./chapters/chapter-14.md)

### Часть VI. The Pivot: Replaceability & Operations — «Как заменить компонент, не сломав систему?»
- [Глава 15. The component becomes the unit of architecture (The Pivot)](./chapters/chapter-15.md)
- [Глава 16. Hot-swap mechanics: in-flight, drain, rollback](./chapters/chapter-16.md)
- [Глава 17. Component as a deployable unit (Manifest, Registry & Admission)](./chapters/chapter-17.md)

### Часть VII. Composition & Finishing — «Собираем всё вместе»
- [Глава 18. Composition models](./chapters/chapter-18.md)
- [Глава 19. Test architecture and independent build](./chapters/chapter-19.md)
- [Глава 20. Component design patterns](./chapters/chapter-20.md)
- [Глава 21. Working managed runtime — финальная сборка](./chapters/chapter-21.md)
- [Глава 22. Итог книги: что мы построили и что дальше](./chapters/chapter-22.md)

---

## 🛠 Как работать с репозиторием

1. **Клонируйте репозиторий** и перейдите в директорию проекта.
2. **Установите инструменты** (версии зафиксированы в `rust-toolchain.toml`):
   ```bash
   rustup target add wasm32-wasip1 wasm32-wasip2
   cargo install cargo-component --locked
