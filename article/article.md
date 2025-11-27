# Oxidieren Schritt für Schritt

Refactoring und Erweiterung bestehender Embedded-Anwendungen mit Rust

**Rust ist da[^ov-erip][^fs-rwww][^cd-rip]. Rust ist heute vom Mikrocontroller bis zum Hyperscaler zu finden. Die  Sicherheitsgarantien der Sprache sind ein wichtiger Faktor, der zu dieser Verbreitung beiträgt. Aber selbst wenn man diesen außer Acht lässt, trägt Rust als ausdrucksstarke Sprache mit zeitgemäßer Werkzeugunterstützung, einem soliden Ökosystem und  einer offenen, aktiven Gemeinschaft viel, von dem ein Projekt profitieren kann. Fast alles davon lässt sich auf Mikrocontrollern nutzen und bestehende C-Anwendungen damit modernisieren.**

Im Gegensatz zu einer Neuimplementierung kann eine schrittweise Modernisierung einer Anwendung Aufwand und Risiko geringer halten. Einsatzerprobter Code kann weiterverwendet werden, bestehende Zertifizierungen bleiben erhalten und Erfahrung mit Rust kann sukzessiv erworben werden. Dadurch, dass die Anwendung erlebbar bleibt, bestehende Tests und Benchmarks weiterbenutzt werden können, wird es auch leichter, erreichten Fortschritt zu bewerten und mit dem gewonnenen Wissen weitere Schritte zu planen.

Selbst im Mix mit anderen Sprachen kann Rust seine Stärken ausspielen: Neuer Code in Rust wird weniger Fehler beinhalten als der gleich Code in C[^ms-bsoc][^rc-cppiop]. Während der erste Schritt noch viel Arbeit für die Integration der neuen Sprache erfordert, rücken bereits beim Zweiten die Arbeit mit der Sprache und das Erleben der Änderungen in den Vordergrund.

Die folgenden Abschnitte zeigen die Schritte zur Integration. Sie sollen dazu motivieren, es selbst auszuprobieren und unter [^online-example] steht ein vollständiges Beispiel dazu bereit.

## C-ABI als Grundlage

Die Integration von Rust mit anderen Sprachen erfolgt typischerweise über das C Application-Binary-Interface (C-ABI). Dieses legt unter anderem die Konventionen für Funktionsaufrufe und das Speicherlayout für Daten festlegt. Das C-ABI ist nicht Rusts natürliches ABI - aber Rusts Foreign-Function-Interface[^rn-ffi] (FFI) kann dazu kompatiblen Code erzeugen und über diesen Aufrufe zwischen den Sprachen in beide Richtungen ermöglichen.

Eine zum C-ABI kompatible Funktion `add` zum Addieren zweiter Integer-Werte kann zum Beispiel in Rust wie folgt definiert werden:

```rust
use core::ffi::c_int;

#[unsafe(no_mangle)]
pub extern "C" fn add(left: c_int, right: c_int) -> c_int {
    left + right
}
```

Für den plattformspezifischen C-Datentypen `int` stellt Rust die kompatible Typdefinition `c_int`[^drs-core-ffi] bereit. Die Annotation `#[unsafe(no_mangle)]` deaktiviert das sogenannte Name-Mangling[^1] und sorgt damit dafür, dass ein C-Linker sie auch als `add` findet. Das Schlüsselwort `unsafe`[^rb-unsafe] dient dabei zum Ausweisen der Stellen, an denen Rusts Sicherheitsgarantien nicht mehr vom Compiler geprüft werden können und macht es explizit, dass deren Einhaltung vom Entwickler sichergestellt werden muss. Durch `extern "C"` erhält die Funktion einen zum C-ABI kompatiblen Prolog und Epilog, sodass sie direkt von C-Code mit dessen Konventionen aufgerufen werden kann.

Die Rust-Funktion `add` kann damit aus C-Code heraus wie jede normale C-Funktion aufgerufen werden:

```c
extern int add(int left, int right);

int main(void) {
    return add(1, 2);
}
```

Für die Gegenrichtung - also von Rust zu C - liegt die Aufgabe des Erzeugens kompatibler Aufrufe ebenfalls beim Rust-Compiler und kann über die Definition eines Funktionsprototypen als `extern "C"` festgelegt werden:

```rust
extern "C" {
    pub fn square(value: c_int) -> c_int;
}

pub fn main() {
    let let squared = unsafe { square(42) };
}
```

Für Datenstrukturen kann ein zum C-ABI kompatibles Speicherlayout über eine Annotation mit `#[repr(C)]` erzeugt werden. Das folgende Beispiel zeigt die Rust-Stuktur `Point`. Diese besitzt ein C-kompatibles Speicherlayout und ist gleichzeitig ein vollwertiger Rust-Datentyp mit allem gewohntem Komfort, wie automatisch erzeugten Vergleichsoperationen oder einer Darstellung für Debugausgaben:

```rust
#[derive(Clone, Copy, Debug, Eq, PartialEq)]
#[repr(C)]
pub struct Point {
    x: i32,
    y: i32,
} 
```

Eine äquivalente Definition in C ist:

```c
typedef struct {
    int32_t x;
    int32_t y;
} Point;
```

Damit ist der Grundstein für die Integration auf der Ebene der Sprache gelegt. In der Praxis fallen Typen und Funktionssignaturen deutlich komplexer aus - doch dieses Fundament bleibt stets Dasselbe. Die nun folgenden Abschnitte beschreiben das daraus entstehende Grundrezept und die weiteren Zutaten zu einer Integration.

## Integration als Bibliothek

Für die fertige Anwendung muss der Code aus beiden Welten noch integriert werden. Im Fall der Integration vor Rust-Code in eine C-Mikrocontroller-Anwendung geschieht dies über eine statische, abgeschlossene Bibliothek, die alle Rust-Abhängigkeiten enthält. 

Rusts kleinste Übersetzungseinheit ist eine sogenannte Crate. Für diese kann der Typ des zu erzeugenden Artefakts in ihrem Manifest `Cargo.toml` ausgewählt werden. Für den hier beschriebenen Anwendungsfall wird eine `staticlib`[^linkage] benötigt:

```toml
[lib]
crate-type = ["staticlib"]
```

Soll Rust-Code aus mehr als einer Crate integriert werden, so kann dies über eine dedizierte Crate zur Integration erfolgen, die:

* Alle benötigten Crates als Abhängigkeiten importiert
* FFI-Bindings für diese gegebenenfalls als separate Crates importiert
* Die Integrationen mit der Rust-Laufzeitumgebung selbst bereitstellt oder importiert

Die Integration in den bestehenden Bau ist so vielfältig, wie die eingesetzten Build-Systeme: Vom Vendoring der binären Bibliothek mit ihren Headern für Eclipse-Varianten, über die manuellen Integration mit Make, bis hin zur Integration in CMake mit Hilfe eines Plugins wie Corrosion[^corrosion]. Letzteres zeigt auch das Online-Beispiel[^online-example].

## Automatisches Erzeugen von FFI-Bindings

Die Prototypen für Datentypen und Funktionen werden auch FFI-Bindings genannt. In den hier gezeigten Beispielen wurden sie manuell erstellt. Bei einer realen Anwendung wird dies jedoch schnell so komplex, dass es sich anbietet, diese zu generieren. Das reduziert zum einen das Potential für Fehler. Zum anderen kann es helfen, FFI-Bindings immer auf dem aktuellen Stand zu halten, indem man sie bei jedem Bau neu generiert. Inkompatible Änderungen am Code, für den die Bindings erzeugt werden, fallen so zur Übersetzungszeit und nicht erst als Fehler zur Laufzeit auf.

Die am weitest verbreiteten Werkzeuge dafür sind bindgen[^bindgen] zum Erzeugen der Prototypen aus C-Code für Rust und cbindgen[^cbindgen] zum Erzeugen von Definitionen und Prototypen aus Rust-Code für C.

Dokumentation zu ihrer Nutzung gibt es online beim jeweiligen Werkzeug und das Online-Beispiel zum Artikel[^online-example] zeigt ihre Nutzung.

## Fehlerbehandlung

Rust modelliert Fehler, die auch in korrekten Anwendungen auftreten können, wie zum Beispiel das Parsen von Eingaben, zur expliziten Behandlung über den Rückgabetyp `Result`[^trpl-result].

Fehler, die hingegen auf eine inkorrekte Anwendung zurückzuführen sind, werden oft nicht explizit modelliert. Stattdessen führen sie zu einer sogenannten Panik und dem Abbruch der Anwendung, wie sie zum Beispiel bei einer Division durch Null entsteht:

```rust
fn div(nominator: u32, denominator: u32) -> u32 {
    nominator / denominator
}
```

Während Rusts Standard-Laufzeitumgebung auf eine Panik im Normalfall mit Aufräumen des Aufrufstacks, der Ausgabe eine Fehlermeldung und dem Beenden des Prozesses reagiert, so erfordert die eingeschränkte Umgebung auf einem Mikrocontroller einen individuellen Panik-Handler und die Konfiguration zum Abbruch ohne Aufräumen des Stacks über `Cargo.toml`:

```toml
[profile.debug]
panic = "abort"

[profile.release]
panic = "abort"
```

Zum Benachrichtigen der bestehenden Anwendung (für einen geordneten Rückzug) könnte der Panik-Handler auf einem Cortex-M-Controller wie folgt aussehen: 

```rust
extern "C" {
    pub fn notify_about_rust_panic();
}

#[panic_handler]
fn panic(_: &PanicInfo) -> ! {
    unsafe { notify_about_rust_panic() }; 
    cortex_m::asm::udf();
}
```

Sollte die Benachrichtigung der Anwendung in diesem Beispiel zurückkehren, wird eine Prozessorausnahme ausgelöst und die dafür bereits bestehenden Mechanismen in Gang gesetzt.

## Statisch allokierte Collections

Rusts Core-Laufzeitumgebung enthält keine der Collection-Datentypen der Standard-Laufzeitumgebung, wie zum Beispiel einen Vektor oder eine Map. Dies liegt daran, dass die Standard-Collections dynamische Speicherallokationen nutzen. Als Ersatz stellt die Crate `heapless`[^heapless] alternative Collections bereit, die sich auch in der Core-Laufzeitumgebung nutzen lassen.

## Dynamischer Speicher

Bis hierher erfolgten alle Speicherallokationen statisch. Sollen zusätzliche dynamische Allokationen erfolgen, benötigt die Core-Laufzeitumgebung einen explizit bereitgestellten Allokator[^global-allocator]. In vielen Fällen bringt bereits die C-Laufzeitumgebung einen solchen mit und er lässt sich mit Hilfe des Wrappers `LibcAlloc`[^crate-libc-alloc] nutzen:

```rust
use libc_alloc::LibcAlloc;

#[global_allocator]
static ALLOCATOR: LibcAlloc = LibcAlloc;
```

Die Nutzung desselben Allokators aus C und Rust ist auch die Grundlage dafür, um Speicherobjekte, die in einer der Sprachen allokiert wurden mit der Bordmitteln der jeweils anderen Sprache wieder freizugeben.

## Zusammenfassung und Ausblick

Die erste Schritt ist oft der schwierigste - nicht zuletzt deshalb, weil dabei viele unterschiedliche Probleme auf einmal gelöst werden müssen. Das ist auch bei der Integration von Rust in eine bestehende Anwendung nicht anders. Ist er einmal geschafft, werden die nächsten Schritte einfacher.

Rust kann an strategisch gewählten Stellen seine Sicherheitsgarantien und seine Produktivität unter Beweis stellen. Ist Rust erst einmal integriert, öffnet sich für die betreffende Anwendung auch der Zugang zu Rusts Ökosystem - über das Foreign-Function-Interface auch für bestehenden Code in C. Ein Entwicklungsteam kann mit einer vertrauten Codebasis arbeiten und schrittweise weitere Erfahrung sammeln.

_Die beste Zeit, eine Anwendung mit Rust zu modernisieren, war vor 10 Jahren.
Die zweitbeste Zeit ist jetzt._ - Das Känguru

Es lohnt sich. In mehrfacher Hinsicht.

## Literatur- und Quellenverzeichnis

[^ov-erip]: https://onevariable.com/blog/embedded-rust-production/
[^fs-rwww]: https://ferrous-systems.com/blog/rust-who-what-why/
[^cd-rip]: https://corrode.dev/podcast/

[^1]: Die Funktion würde sonst beispielsweise als `my_crate::add::h6f743390b012cb87` in der Symboltabelle auftauchen

[^ms-bsoc]: https://youtu.be/uDtMuS7BExE?t=17
[^mm-rop]: https://www.youtube.com/watch?v=H0AUP2OgppE
[^rc-cppiop]: https://youtu.be/Z5M4NIWoMJQ?t=80

[^drs-core-ffi]: https://doc.rust-lang.org/core/ffi/index.html
[^rn-ffi]: https://doc.rust-lang.org/nomicon/ffi.html

[^rb-unsafe]: https://doc.rust-lang.org/book/ch20-01-unsafe-rust.html

[^corrosion]: https://github.com/corrosion-rs/corrosion
[^bindgen]: https://github.com/rust-lang/rust-bindgen
[^cbindgen]: https://github.com/mozilla/cbindgen
[^online-example]: https://github.com/sirhcel/ese-congress-2025
[^trpl-result]: https://doc.rust-lang.org/book/ch09-02-recoverable-errors-with-result.html
[^linkage]: https://doc.rust-lang.org/reference/linkage.html
[^heapless]: https://github.com/rust-embedded/heapless
[^global-allocator]: https://doc.rust-lang.org/core/alloc/trait.GlobalAlloc.html
[^crate-libc-alloc]: https://docs.rs/libc_alloc

## Autor

![[portrait-github.jpg]]

Christian Meusel ist Diplom-Informatiker und entwickelt Eingebettete Systeme mit Leidenschaft. Er hat in vielen Rollen und Gewerken gearbeitet: Von Schwarzmagie für EMV-Zertifizierungen, Hardwareinbetriebnahmen, Software vom Mikrocontroller bis zum Userspace, Sicherheitsaudits und Produktzertifizierungen. Seit 2016 hilft er als Freiberufler seinen Kunden, aus Ideen Produkte werden zu lassen und diese im Feld zu betreuen. Rust spielt dabei eine immer größere Rolle.

## Kontakt

christian@christian-meusel.de
https://christian-meusel.de
