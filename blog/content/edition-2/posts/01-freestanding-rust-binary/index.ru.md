+++
title = "Независимый бинарный файл на Rust"
weight = 1
path = "ru/freestanding-rust-binary"
date = 2018-02-10

[extra]
translators = ["MrZloHex"]
+++

Первый шаг в создании собственного ядра операционной системы &mdash; это создание исполняемого файла на Rust, который не будет подключать стандартную библиотеку. Именно это дает возможность запускать Rust код на [голом железе][bare metal] без слоя операционной системы.

[bare metal]: https://en.wikipedia.org/wiki/Bare_machine

<!-- more -->

Этот блог открыто разрабатывается на [GitHub]. Если у вас возникли какие-либо проблемы или вопросы, пожалуйста, создайте _issue_. Также вы можете оставлять комментарии [в конце страницы][at the bottom]. Полный исходный код для этого поста вы можете найти в репозитории в ветке [`post-01`][post branch].

[GitHub]: https://github.com/phil-opp/blog_os
[at the bottom]: #comments
<!-- fix for zola anchor checker (target is in template): <a id="comments"> -->
[post branch]: https://github.com/phil-opp/blog_os/tree/post-01

<!-- toc -->

## Введение
Для того, чтобы написать ядро операционной системы, нужен код, который не зависит от операционной системы и ее возможностей. Это означает, что нельзя использовать потоки, файлы, [кучу][heap], сети, случайные числа, стандартный вывод или другие возможности, которые зависят от ОС или определённого железа.

[heap]: https://en.wikipedia.org/wiki/Heap_(data_structure)

Это значит, что нельзя использовать большую часть [стандартной библиотеки Rust][Rust Standard library], но остается множество других возможностей Rust, которые _можно использовать_. Например, [итераторы][iterators], [замыкания][closures], [сопоставление с образцом][pattern matching], [`Option`][option] и [`Result`][result], [форматирование строк][string formatting] и, конечно же, [систему владения][ownership system]. Эти функции дают возможность для написания ядра в очень выразительном и высоко-уровневом стиле, не беспокоясь о [неопределенном поведении][undefined behavior] или [сохранности памяти][memory safety].

[option]: https://doc.rust-lang.org/core/option/
[result]:https://doc.rust-lang.org/core/result/
[Rust standard library]: https://doc.rust-lang.org/std/
[iterators]: https://doc.rust-lang.org/book/ch13-02-iterators.html
[closures]: https://doc.rust-lang.org/book/ch13-01-closures.html
[pattern matching]: https://doc.rust-lang.org/book/ch06-00-enums.html
[string formatting]: https://doc.rust-lang.org/core/macro.write.html
[ownership system]: https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html
[undefined behavior]: https://www.nayuki.io/page/undefined-behavior-in-c-and-cplusplus-programs
[memory safety]: https://tonyarcieri.com/it-s-time-for-a-memory-safety-intervention

Чтобы создать ядро ОС на Rust, нужно создать исполняемый файл, который мог бы запускаться без ОС.

Этот пост описывает необходимые шаги для создания независимого исполняемого файла на Rust и объясняет, почему эти шаги нужны. Если вам интересен только минимальный пример, можете сразу перейти к __[итогам](#summary)__.

## Отключение стандартной библиотеки
По умолчанию, все Rust-крейты подключают [стандартную библиотеку][standard library], которая зависит от возможностей операционной системы, таких как потоки, файлы, сети. Она также зависит от стандартной библиотки C `libc`, которая очень тесно взаимодействует с возможностями ОС. Так как мы хотим написать операционную систему, мы не можем использовать библиотеки, которые зависят от операционной системы. Поэтому необходимо отключить автоматические подключение стандартной библиотеки через [атрибут `no_std`][attribute].

[standard library]: https://doc.rust-lang.org/std/
[attribute]: https://doc.rust-lang.org/1.30.0/book/first-edition/using-rust-without-the-standard-library.html

Мы начнем с создания нового проекта cargo. Самый простой способ сделать это &mdash; через командную строку:

```
cargo new blog_os --bin -- edition 2018
```

Я назвал этот проект `blog_os`, но вы можете назвать как вам угодно. Флаг `--bin` указывает на то, что мы хотим создать исполняемый файл (а не библиотеку), а флаг `--edition 2018` указывает, что мы хотим использовать [редакцию Rust 2018][edition] для нашего крейта. После выполнения команды cargo создаст каталог со следующей структурой:

[edition]: https://doc.rust-lang.org/nightly/edition-guide/rust-2018/index.html

```
blog_os
├── Cargo.toml
└── src
    └── main.rs
```

`Cargo.toml` содержит данные и конфигурацию крейта, такие как _название, автор, [семантическую версию][semantic version]_ и _зависимости_ от других крейтов. Файл `src/main.rs` содержит корневой модуль нашего крейта и функцию `main`. Можно скомпилировать крейт с помощью `cargo build` и запустить скомпилированную программу `blog_os` в поддиректории `target/debug`.

[semantic version]: https://semver.org/

### Атрибут `no_std`

В данный момент наш крейт неявно подключает стандартную библиотеку. Это можно исправить путем добавления [атрибута `no_std`][attribute]:

```rust
// main.rs

#![no_std]

fn main() {
    println!("Hello, world!");
}
```

Если сейчас попробовать скомпилировать программу (с помоцью команды `cargo build`), то появится следующая ошибка:

```
error: cannot find macro `println!` in this scope
 --> src/main.rs:4:5
  |
4 |     println!("Hello, world!");
  |     ^^^^^^^
```

Эта ошибка объясняется тем, что [макрос `println`][macro] &mdash; часть стандартной библиотеки, которая была отключена. Поэтому у нас больше нет возможность выводить что-либо на экран. Это логично, так как `println` печатает через [стандартный вывод][standard output], который, в свою очередь, является специальным файловым дескриптором, предоставляемым операционной системой.

[macro]: https://doc.rust-lang.org/std/macro.println.html
[standard output]: https://en.wikipedia.org/wiki/Standard_streams#Standard_output_.28stdout.29

Давайте уберем макрос `println` и попробуем скомпилировать еще раз:

```rust
// main.rs

#![no_std]

fn main() {}
```

```
> cargo build
error: `#[panic_handler]` function required, but not found
error: language item required, but not found: `eh_personality`
```

Сейчас компилятор не может найти функцию `#[panic_handler]` и «элемент языка».

## Реализация _паники_

Атрибут `pаnic_handler` определяет функцию, которая должна вызываться, когда происходит [паника (panic)][panic]. Стандартная библиотека предоставляет собственную функцию обработчика паники, но после отключения стандартной библиотеки мы должны написать собственный обработчик:

[panic]: https://doc.rust-lang.org/stable/book/ch09-01-unrecoverable-errors-with-panic.html

```rust
// in main.rs

use core::panic::PanicInfo;

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

Параметр [`PanicInfo`][PanicInfo] содержит название файла и строку, где произошла паника, и дополнительное сообщение с пояснением. Эта функция никогда не должна возвратиться, и такая функция называется [расходящейся][diverging functions] и она возращает [пустой тип]["never" type] `!`. Пока что мы ничего не можем сделать в этой функции, поэтому мы просто войдем в бесконечный цикл.

[PanicInfo]: https://doc.rust-lang.org/nightly/core/panic/struct.PanicInfo.html
[diverging functions]: https://doc.rust-lang.org/1.30.0/book/first-edition/functions.html#diverging-functions
["never" type]: https://doc.rust-lang.org/nightly/std/primitive.never.html

## Элемент языка `eh_personality`

Элементы языка &mdash; это специальные функции и типы, которые необходимы компилятору. Например, трейт [`Copy`] указывает компилятору, у каких типов есть [_семантика копирования_][`Copy`]. Если мы посмотрим на [реализацию][copy code] этого трейта, то увидим специальный атрибут `#[lang = "copy"]`, который говорит, что этот трейт является элементом языка.

[`Copy`]: https://doc.rust-lang.org/nightly/core/marker/trait.Copy.html
[copy code]: https://github.com/rust-lang/rust/blob/485397e49a02a3b7ff77c17e4a3f16c653925cb3/src/libcore/marker.rs#L296-L299

Несмотря на то, что можно предоставить свою реализацию элементов языка, это следует делать только в крайних случаях. Причина в том, что элементы языка являются крайне нестабильными деталями реализации, и компилятор даже не проверяет в них согласованность типов (поэтому он даже не проверяет, имеет ли функция правильные типы аргументов). К счастью, существует более стабильный способ исправить вышеупомянутую ошибку.

Элемент языка [`eh_personality`][language item] указывает на функцию, которая используется для реализации [раскрутки стека][stack unwinding]. По умолчанию, Rust использует раскрутку для запуска деструктуров для всех _живых_ переменных на стеке в случае [паники][panic]. Это гарантирует, что вся использованная память будет освобождена, и позволяет родительскому потоку перехватить панику и продолжить выполнение. Раскрутка &mdash; очень сложный процесс и требует некоторых специльных библиотек ОС (например, [libunwind] для Linux или [structured exception handling] для Windows), так что мы не должны использовать её для нашей операционной системы.

[language item]: https://github.com/rust-lang/rust/blob/edb368491551a77d77a48446d4ee88b35490c565/src/libpanic_unwind/gcc.rs#L11-L45
[stack unwinding]: https://www.bogotobogo.com/cplusplus/stackunwinding.php
[libunwind]: https://www.nongnu.org/libunwind/
[structured exception handling]: https://docs.microsoft.com/ru-ru/windows/win32/debug/structured-exception-handling

### Отключение раскрутки

Существуют и другие случаи использования, для которых раскрутка нежелательна, поэтому Rust предоставляет опцию [прерывания выполнения при панике][abort on panic]. Это отключает генерацию информации о символах раскрутки и, таким образом, значительно уменьшает размер бинарного файла. Есть несколько мест, где мы можем отключить раскрутку. Самый простой способ &mdash; добавить следующие строки в наш `Cargo.toml`:

```toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

Это устанавливает стратегию паники на `abort` (прерывание) как для профиля `dev` (используемого для `cargo build`), так и для профиля `release` (используемого для `cargo build --release`). Теперь элемент языка `eh_personality` больше не должен требоваться.

[abort on panic]: https://github.com/rust-lang/rust/pull/32900

Теперь мы исправили обе вышеуказанные ошибки. Однако, если мы сейчас попытаемся скомпилировать программу, возникнет другая ошибка:

```
> cargo build
error: requires `start` lang_item
```

В нашей программе отсутствует элемент языка `start`, который определяет начальную точку входа программы.

## Аттрибут `start`

Можно подумать, что функция `main` &mdash; это первая функция, вызываемая при запуске программы. Однако в большинстве языков есть [среда выполнения][runtime system], которая отвечает за такие вещи, как сборка мусора (например, в Java) или программные потоки (например, goroutines в Go). Эта система выполнения должна быть вызвана до `main`, поскольку ей необходимо инициализировать себя.

[runtime system]: https://en.wikipedia.org/wiki/Runtime_system

В типичном исполнимом файле Rust, который использует стандартную библиотеку, выполнение начинается в runtime-библиотеке C под названием `crt0` ("C runtime zero"), которая создает окружение для C-приложения. Это включает создание стека и размещение аргументов в нужных регистрах. Затем C runtime вызывает [точку входа для Rust-приложения][rt::lang_start], которая обозначается элементом языка `start`. Rust имеет очень маленький runtime, который заботится о некоторых мелочах, таких как установка защиты от переполнения стека или вывод сообщения при панике. Затем рантайм вызывает функцию `main`.

[rt::lang_start]: https://github.com/rust-lang/rust/blob/bb4d1491466d8239a7a5fd68bd605e3276e97afb/src/libstd/rt.rs#L32-L73

Наш независимый исполняемый файл не имеет доступа к runtime Rust и `crt0`, поэтому нам нужно определить собственную точку входа. Реализация языкового элемента `start` не поможет, поскольку он все равно потребует `crt0`. Вместо этого нам нужно напрямую переопределить точку входа `crt0`.

### Переопределение точки входа

Чтобы сообщить компилятору Rust, что мы не хотим использовать стандартную цепочку точек входа, мы добавляем атрибут `#![no_main]`.

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

Можно заметить, что мы удалили функцию `main`. Причина в том, что `main` не имеет смысла без стандартного runtime, которая ее вызывает. Вместо этого мы переопределим точку входа операционной системы с помощью нашей собственной функции `_start`:

```rust
#[no_mangle]
pub extern "C" fn _start() -> ! {
    loop {}
}
```

Используя атрибут `#[no_mangle]`, мы отключаем [искажение имен][name mangling], чтобы гарантировать, что компилятор Rust сгенерирует функцию с именем `_start`. Без этого атрибута компилятор генерировал бы какой-нибудь загадочный символ `_ZN3blog_os4_start7hb173fedf945531caE`, чтобы дать каждой функции уникальное имя. Атрибут необходим, потому что на следующем этапе нам нужно сообщить имя функции точки входа компоновщику.

Мы также должны пометить функцию как `extern "C"`, чтобы указать компилятору, что он должен использовать [соглашение о вызове C][C calling convention] для этой функции (вместо неопределенного соглашения о вызове Rust). Причина именования функции `_start` в том, что это имя точки входа по умолчанию для большинства систем.

[name mangling]: https://en.wikipedia.org/wiki/Name_mangling
[C calling convention]: https://en.wikipedia.org/wiki/Calling_convention

Возвращаемый `!` означает, что функция является расходящейся, т.е. не имеет права возвращаться. Это необходимо, поскольку точка входа не вызывается никакой функцией, а вызывается непосредственно операционной системой или загрузчиком. Поэтому вместо возврата точка входа должна, например, вызвать [системный вызов `exit`][`exit` system call] операционной системы. В нашем случае разумным действием может быть выключение машины, поскольку ничего не останется делать, если независимый исполнимый файл завершит исполнение. Пока что мы выполняем это требование путем бесконечного цикла.

[`exit` system call]: https://en.wikipedia.org/wiki/Exit_(system_call)

Если мы выполним `cargo build` сейчас, мы получим ошибку компоновщика (_linker_ error).

## Ошибки компоновщика

Компоновщик &mdash; это программа, которая объединяет сгенерированный код в исполняемый файл. Поскольку формат исполняемого файла отличается в Linux, Windows и macOS, в каждой системе есть свой компоновщик, и каждый покажет свою ошибку. Основная причина ошибок одна и та же: конфигурация компоновщика по умолчанию предполагает, что наша программа зависит от C runtime, а это не так.

Чтобы устранить ошибки, нам нужно сообщить компоновщику, что он не должен включать C runtime. Мы можем сделать это, передав компоновщику определенный набор аргументов или выполнив компиляцию для голого железа.

### Компиляция для голого железа

По умолчанию Rust пытается создать исполняемый файл, который может быть запущен в окружении вашей текущей системы. Например, если вы используете Windows на `x86_64`, Rust пытается создать исполняемый файл Windows `.exe`, который использует инструкции `x86_64`. Это окружение называется вашей "хост-системой".

Для описания различных окружений Rust использует строку [_target triple_]. Вы можете узнать тройку вашей хост-системы, выполнив команду `rustc --version --verbose`:

[_target triple_]: https://clang.llvm.org/docs/CrossCompilation.html#target-triple

```
rustc 1.35.0-nightly (474e7a648 2019-04-07)
binary: rustc
commit-hash: 474e7a6486758ea6fc761893b1a49cd9076fb0ab
commit-date: 2019-04-07
host: x86_64-unknown-linux-gnu
release: 1.35.0-nightly
LLVM version: 8.0
```

Приведенный выше результат получен от системы `x86_64` Linux. Мы видим, что тройка `host` &mdash; это `x86_64-unknown-linux-gnu`, которая включает архитектуру процессора (`x86_64`), производителя (`unknown`), операционную систему (`linux`) и [ABI] (`gnu`).

[ABI]: https://en.wikipedia.org/wiki/Application_binary_interface

Компилируя для тройки нашего хоста, компилятор Rust и компоновщик предполагают наличие базовой операционной системы, такой как Linux или Windows, которая по умолчанию использует C runtime, что вызывает ошибки компоновщика. Поэтому, чтобы избежать ошибок компоновщика, мы можем настроить компиляцию для другого окружения без базовой операционной системы.

Примером такого "голого" окружения является тройка `thumbv7em-none-eabihf`, которая описывает [ARM] архитектуру. Детали не важны, важно лишь то, что тройка не имеет базовой операционной системы, на что указывает `none` в тройке. Чтобы иметь возможность компилировать для этой системы, нам нужно добавить ее в rustup:

[ARM]: https://en.wikipedia.org/wiki/ARM_architecture

```
rustup target add thumbv7em-none-eabihf
```

Это загружает копию стандартной библиотеки (и `core`) для системы. Теперь мы можем собрать наш независимый исполняемый файл для этой системы:

```
cargo build --target thumbv7em-none-eabihf
```

Передавая аргумент `--target`, мы [кросс-компилируем][cross compile] наш исполняемый файл для голого железа. Поскольку система, под которую мы компилируем, не имеет операционной системы, компоновщик не пытается компоновать C runtime, и наша компиляция проходит успешно без каких-либо ошибок компоновщика.

[cross compile]: https://en.wikipedia.org/wiki/Cross_compiler

Именно этот подход мы будем использовать для сборки ядра нашей ОС. Вместо `thumbv7em-none-eabihf` мы будем использовать [custom target], который описывает окружение для архитектуры `x86_64`. Подробности будут описаны в следующем посте.

[custom target]: https://doc.rust-lang.org/rustc/targets/custom.html

### Аргументы компоновщика

Вместо компиляции под голое железо, ошибки компоновщика можно исправить, передав ему определенный набор аргументов. Мы не будем использовать этот подход для нашего ядра, поэтому данный раздел является необязательным и приводится только для полноты картины. Щелкните на _"Аргументы компоновщика"_ ниже, чтобы показать необязательное содержание.

<details>

<summary>Аргументы компоновщика</summary>

В этом разделе мы рассмотрим ошибки компоновщика, возникающие в Linux, Windows и macOS, и объясним, как их решить, передав компоновщику дополнительные аргументы. Обратите внимание, что формат исполняемого файла и компоновщик отличаются в разных операционных системах, поэтому для каждой системы требуется свой набор аргументов.

#### Linux

На Linux возникает следующая ошибка компоновщика (сокращенно):

```
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" […]
  = note: /usr/lib/gcc/../x86_64-linux-gnu/Scrt1.o: In function `_start':
          (.text+0x12): undefined reference to `__libc_csu_fini'
          /usr/lib/gcc/../x86_64-linux-gnu/Scrt1.o: In function `_start':
          (.text+0x19): undefined reference to `__libc_csu_init'
          /usr/lib/gcc/../x86_64-linux-gnu/Scrt1.o: In function `_start':
          (.text+0x25): undefined reference to `__libc_start_main'
          collect2: error: ld returned 1 exit status
```

Проблема заключается в том, что компоновщик по умолчанию включает процедуру запуска C runtime, которая также называется `_start`. Она требует некоторых символов стандартной библиотеки C `libc`, которые мы не включаем из-за атрибута `no_std`, поэтому компоновщик не может подключить эти библиотеки, поэтому появляются ошибки. Чтобы решить эту проблему, мы можем сказать компоновщику, что он не должен компоновать процедуру запуска C, передав флаг `-nostartfiles`.

Одним из способов передачи атрибутов компоновщика через cargo является команда `cargo rustc`. Команда ведет себя точно так же, как `cargo build`, но позволяет передавать опции `rustc`, базовому компилятору Rust. У `rustc` есть флаг `-C link-arg`, который передает аргумент компоновщику. В совокупности наша новая команда сборки выглядит следующим образом:

```
cargo rustc -- -C link-arg=-nostartfiles
```

Теперь наш крейт собирается как независимый исполняемый файл в Linux!

Нам не нужно было явно указывать имя нашей функции точки входа, поскольку компоновщик по умолчанию ищет функцию с именем `_start`.

#### Windows

В Windows возникает другая ошибка компоновщика (сокращенно):

```
error: linking with `link.exe` failed: exit code: 1561
  |
  = note: "C:\\Program Files (x86)\\…\\link.exe" […]
  = note: LINK : fatal error LNK1561: entry point must be defined
```

Ошибка "точка входа должна быть определена" (_"entry point must be defined"_) означает, что компоновщик не может найти точку входа. В Windows имя точки входа по умолчанию [зависит от используемой подсистемы][windows-subsystems]. Для подсистемы `CONSOLE` компоновщик ищет функцию с именем `mainCRTStartup`, а для подсистемы `WINDOWS` - функцию с именем `WinMainCRTStartup`. Чтобы переопределить названия точки входа на `_start`, мы можем передать компоновщику аргумент `/ENTRY`:

[windows-subsystems]: https://docs.microsoft.com/en-us/cpp/build/reference/entry-entry-point-symbol

```
cargo rustc -- -C link-arg=/ENTRY:_start
```

Из разного формата аргументов мы ясно видим, что компоновщик Windows - это совершенно другая программа, чем компоновщик Linux.

Теперь возникает другая ошибка компоновщика:

```
error: linking with `link.exe` failed: exit code: 1221
  |
  = note: "C:\\Program Files (x86)\\…\\link.exe" […]
  = note: LINK : fatal error LNK1221: a subsystem can't be inferred and must be
          defined
```

Эта ошибка возникает из-за того, что исполняемые файлы Windows могут использовать различные [подсистемы][windows-subsystems]. Для обычных программ они определяются в зависимости от имени точки входа: если точка входа называется `main`, то используется подсистема `CONSOLE`, а если точка входа называется `WinMain`, то используется подсистема `WINDOWS`. Поскольку наша функция `_start` имеет другое имя, нам нужно явно указать подсистему:

```
cargo rustc -- -C link-args="/ENTRY:_start /SUBSYSTEM:console"
```

Здесь мы используем подсистему `CONSOLE`, но подойдет и подсистема `WINDOWS`. Вместо того, чтобы передавать `-C link-arg` несколько раз, мы используем `-C link-args`, который принимает список аргументов, разделенных пробелами.

С помощью этой команды наш исполняемый файл должен успешно скомпилироваться под Windows.

#### macOS

На macOS возникает следующая ошибка компоновщика (сокращенно):

```
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" […]
  = note: ld: entry point (_main) undefined. for architecture x86_64
          clang: error: linker command failed with exit code 1 […]
```

Это сообщение об ошибке говорит нам, что компоновщик не может найти функцию точки входа с именем по умолчанию `main` (по какой-то причине в macOS все функции имеют префикс `_`). Чтобы установить точку входа в нашу функцию `_start`, мы передаем аргумент компоновщика `-e`:

```
cargo rustc -- -C link-args="-e __start"
```

Флаг `-e` задает имя функции точки входа. Поскольку в macOS все функции имеют дополнительный префикс `_`, нам нужно установить точку входа на `__start` вместо `_start`.

Теперь возникает следующая ошибка компоновщика:

```
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" […]
  = note: ld: dynamic main executables must link with libSystem.dylib
          for architecture x86_64
          clang: error: linker command failed with exit code 1 […]
```

macOS [официально не поддерживает статически скомпонованные исполняемые файлы][static binary] и по умолчанию требует от программ компоновки библиотеки `libSystem`. Чтобы переопределить это поведение и скомпоновать статический исполняемый файл, передадим компоновщику флаг `-static`:

[static binary]: https://developer.apple.com/library/archive/qa/qa1118/_index.html

```
cargo rustc -- -C link-args="-e __start -static"
```

Этого все равно недостаточно, так как возникает третья ошибка компоновщика:

```
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" […]
  = note: ld: library not found for -lcrt0.o
          clang: error: linker command failed with exit code 1 […]
```

Эта ошибка возникает из-за того, что программы на macOS по умолчанию ссылаются на `crt0` ("C runtime zero"). Она похожа на ошибку под Linux и тоже может быть решена добавлением аргумента компоновщика `-nostartfiles`:

```
cargo rustc -- -C link-args="-e __start -static -nostartfiles"
```

Теперь наша программа должна успешно скомпилироваться на macOS.

#### Объединение команд сборки

Сейчас у нас разные команды сборки в зависимости от платформы хоста, что не идеально. Чтобы избежать этого, мы можем создать файл с именем `.cargo/config.toml`, который будет содержать аргументы для конкретной платформы:

```toml
# in .cargo/config.toml

[target.'cfg(target_os = "linux")']
rustflags = ["-C", "link-arg=-nostartfiles"]

[target.'cfg(target_os = "windows")']
rustflags = ["-C", "link-args=/ENTRY:_start /SUBSYSTEM:console"]

[target.'cfg(target_os = "macos")']
rustflags = ["-C", "link-args=-e __start -static -nostartfiles"]
```

Ключ `rustflags` содержит аргументы, которые автоматически добавляются к каждому вызову `rustc`. Более подробную информацию о файле `.cargo/config.toml` можно найти в [официальной документации](https://doc.rust-lang.org/cargo/reference/config.html).

Теперь наша программа должна собираться на всех трех платформах с помощью простой `cargo build`.

#### Должны ли вы это делать?

Хотя можно создать независимый исполняемый файл для Linux, Windows и macOS, это, вероятно, не очень хорошая идея. Причина в том, что наш исполняемый файл все еще ожидает различных вещей, например, инициализации стека при вызове функции `_start`. Без C runtime некоторые из этих требований могут быть не выполнены, что может привести к сбою нашей программы, например, из-за ошибки сегментации.

Если вы хотите создать минимальный исполняемый файл, запускаемый поверх существующей операционной системы, то включение `libc` и установка атрибута `#[start]`, как описано [здесь] (https://doc.rust-lang.org/1.16.0/book/no-stdlib.html), вероятно, будет идеей получше.

</details>

## Итоги {#summary}

Минимальный независимый исполняемый бинарный файл Rust выглядит примерно так:

`src/main.rs`:

```rust
#![no_std] // don't link the Rust standard library
#![no_main] // disable all Rust-level entry points

use core::panic::PanicInfo;

#[no_mangle] // don't mangle the name of this function
pub extern "C" fn _start() -> ! {
    // this function is the entry point, since the linker looks for a function
    // named `_start` by default
    loop {}
}

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

`Cargo.toml`:

```toml
[package]
name = "crate_name"
version = "0.1.0"
authors = ["Author Name <author@example.com>"]

# the profile used for `cargo build`
[profile.dev]
panic = "abort" # disable stack unwinding on panic

# the profile used for `cargo build --release`
[profile.release]
panic = "abort" # disable stack unwinding on panic
```

Чтобы собрать этот исполняемый файл, его надо скомпилировать для голого железа, например, `thumbv7em-none-eabihf`:

```
cargo build --target thumbv7em-none-eabihf
```

В качестве альтернативы, мы можем скомпилировать его для хост-системы, передав дополнительные аргументы компоновщика:

```bash
# Linux
cargo rustc -- -C link-arg=-nostartfiles
# Windows
cargo rustc -- -C link-args="/ENTRY:_start /SUBSYSTEM:console"
# macOS
cargo rustc -- -C link-args="-e __start -static -nostartfiles"
```

Обратите внимание, что это лишь минимальный пример независимого бинарного файла Rust. Этот бинарник ожидает различных вещей, например, инициализацию стека при вызове функции `_start`. **Поэтому для любого реального использования такого бинарного файла потребуется совершить еще больше действий**.

## Что дальше?

В [следующем посте][next post] описаны шаги, необходимые для превращения нашего независимого бинарного файла в минимальное ядро операционной системы. Сюда входит создание custom target, объединение нашего исполняемого файла с загрузчиком и изучение, как вывести что-то на экран.

[next post]: @/edition-2/posts/02-minimal-rust-kernel/index.ru.md
