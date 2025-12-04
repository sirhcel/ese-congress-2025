---
# try also 'default' to start simple
theme: default
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
# background: https://cover.sli.dev
# some information about your slides (markdown enabled)
title: Oxidieren Schritt für Schritt - Refactoring und Erweiterung bestehender Embedded-Anwendungen mit Rust
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# apply UnoCSS classes to the current slide
# class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: fade-out
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# duration of the presentation
duration: 35min

selectable: true
fonts:
    mono: JetBrains Mono
---

Embedded Software Engineering Kongress 2025

# Oxidieren Schritt für Schritt

## Refactoring und Erweiterung bestehender Embedded-Anwendungen mit Rust

Christian Meusel

<!--
The last comment block of each slide will be treated as slide notes. It will be
visible and editable in Presenter Mode along with the slide. [Read more in the
docs](https://sli.dev/guide/syntax.html#notes)
-->

---

# Meine ersten Rust-Bindings

* MQTT-Klient des ESP-IDF von Espressif
* Konfiguration und Start
    ```c
    esp_mqtt_client_handle_t esp_mqtt_client_init(const esp_mqtt_client_config_t *config);
    esp_err_t esp_mqtt_client_start(esp_mqtt_client_handle_t client);
    ```
* Datenstruktur zur Konfiguration
    ```c
    typedef struct psk_key_hint {
        const uint8_t* key;                     /*!< key in PSK authentication mode in binary format */
        size_t   key_size;                      /*!< length of the key */
        const char* hint;                       /*!< hint in PSK authentication mode in string format */
    } psk_hint_key_t;

    typedef struct {
        const char *uri;
        mqtt_event_callback_t event_handle;
        // ...
        // The actual PSK data: key, length, hint.
        const struct psk_key_hint *psk_hint_key;
    }  esp_mqtt_client_config_t;
    ```
<!--
* Mikrocontroller
    * Xtensa
    * RISC-V
* FreeRTOS
-->

---

# Meine ersten Rust-Bindings: Rust-Abstraktion

* Rust-Abstraktion für MQTT-Klient
    ```rust
    let psk = PskConfiguration {
        key: &[0xde, 0xad, /* ... */ ];
        hint: "rust-esp32-std-demo",
    };

    let conf = MqttClientConfiguration {
        client_id: Some("rust-esp32-std-demo"),
        psk: Some(psk),
        ..Default::default()
    };

    let (mut client, mut connection) = EspMqttClient::new_with_conn("mqtts://broker.emqx.io:8883", &conf)?;
    ```
* Wenn es übersetzt, dann funktioniert es
* Das tat es auch

---

# Meine ersten Rust-Bindings: Zwei Wochen später

* Anwendung weiter in Entwicklung
* Sporadisch begannen Fehler aufzutauchen
    ```
    I (14339) rust_esp32_std_demo: MQTT Message: BeforeConnect
    E (14369) esp-tls-mbedtls: mbedtls_ssl_conf_psk returned -0x7100
    E (14369) esp-tls-mbedtls: Failed to set client configurations, returned [0x801B] (ESP_ERR_MBEDTLS_SSL_CONF_PSK_FAILED)
    E (14369) esp-tls: create_ssl_handle failed
    E (14379) esp-tls: Failed to open new connection
    E (14389) TRANSPORT_BASE: Failed to open a new connection
    E (14389) MQTT_CLIENT: Error transport connect
    I (14399) rust_esp32_std_demo: MQTT Message ERROR: ESP_FAIL
    I (14399) rust_esp32_std_demo: MQTT Message: Disconnected
    E (14409) MQTT_CLIENT: Client has not connected
    ```
* In Teilen, die ich seitdem nicht mehr angefasst hatte
* Auf zum Debugging ...

---

# Meine ersten Rust-Bindings: Was war geschehen?

* MQTT-Klient wird in einem separaten Thread ausgeführt
* Konfigurationsdaten werden kopiert
* Für referenzierte Daten in `psk_hint_key` jedoch nur der Zeiger
* Use-After-Free
    * Daten werden an Klienten übergeben
    * Klient wird gestartet
    * Stack-Frame mit Konfigurationsdaten wird abgebaut
    * Klient verarbeitet Konfiguration
* Undefiniertes Verhalten muss nicht unmittelbar Effekte zeigen

<!--
* Durch Tests schwer zu provozieren
    * Stack-Layout
    * Code im Thread der Initialisierung
    * Zeitfenster bis zur Verarbeitung der Konfiguration
-->

---

# Meine ersten Rust-Bindings: Wieso?

* Warum habe ich das in erster Instanz versucht?
    * Dokumentation der Funktionen wies nicht darauf hin
    * Kein Hinweis in den Datenstrukturen zur Konfiguration
* Code-Lesen bis in den MQTT-Klient hätte das offenbart
* Konfigurationsdaten waren im C-Beispielcode als `static` deklariert
* Riesiger-Kontext zum Schlussfolgern

<!--
* Über Ursachen kann ich nur spekulieren
    * Dokumentation Vergessen
    * Unterschiedliche Perspektive auf "offensichtlich"
* ESP-IDF ist OpenSource
    * Reichlich 16.000 Sterne bei GitHub
    * Reichlich 48.000 Commits
    * Knapp 1.000 Leute haben dazu beigetraten
    * Trotz großer Reichweite nicht aufgefallen
* Dokumentation enthält mittlerweile zumindest einen Hinweis
-->

---

# Über mich

* Christian Meusel
* Freiberufler: Eingebettete Systeme vom Scheitel bis zur Sohle
* Rust seit 2020
* Maintainer von serialport-rs und ein paar kleinen Embedded-Crates
* Mitglied der Rust-Embedded-Working-Group
* Arbeite am Neustart einer [Dresdner Rust-Community](https://github.com/rust-dresden) mit
* Kontakt
    * christian@christian-meusel.de
    * https://github.com/sirhcel

<!--
* Java in Eingebettenen Systemen
    * Arbeitshypothesen
    * Speichersicherheit
    * Onboarding
    * Lebendiges Ökosystem
* Rust heute
    * Gleiche Arbeitshypothesen
    * Echte Systemsprache
    * Lässt sich auch auf Mikrocontrollern einsetzen
-->

---
layout: image-left
image: rust-evangelism-strike-force-half-slide.jpg
backgroundSize: contain
---

# Cloudflare Rewrote Their Core in Rust, Then Half of the Internet Went Down

```rust
pub fn fetch_features(
    &mut self,
    input: &dyn BotsInput,
    features: &mut Features,
) -> Result<(), (ErrorFlags, i32)> {
    features.checksum &= 0xFFFF_FFFF_0000_0000;
    features.checksum |= u64::from(self.config.checksum);
    let (feature_values, _) = features
        .append_with_names(&self.config.feature_names)
        .unwrap();
    // ...
}
```

<!--
* Über Rust ;-)
* Guter Gründe abseits des Hypes
    * Ausdrucksstarke Sprache
    * Lokalität beim Schlussfolgern über Code
    * Sichere Defaults
    * Korrektheit bei Integration
    * Hilfreiche Fehlermeldungen
    * Allgemein akzeptierter Stil
* Ich möchte Sie einladen, Sich eine eigene Meinung zu bilden
-->

---

# Warum Erweitern oder Modernisieren?

* Erfahrung mit bekannter Codebasis sammeln
* Nicht alles auf einmal lösen müssen
* Anwendung erlebbar halten
* Bestehenden, bewährten Code behalten
    * Neu entdeckte Fehler stammen überwiegend aus neuem Code
    * Zertifizierung erhalten
    * Investition erhalten
* Externe Komponenten weiter nutzen
    * Adapter für Krypto-Beschleuniger
    * ...
* "Schwache Flanken" verbessern
* Kunden würdigen neue Funktionalität in der Regel höher als einen Rewrite

<!--
* Erlebbar für Tests, Bewertung und Diskussion
* Google: Neue Funktionalität in Android wird in Rust erstellt
    * Hinweis auf Blog-Artikel im Anhang
* Auch für Zertifizierer ist Rust oft Neuland
* Espressif
    * Zuerst Rust-Abstraktionen für ESP-IDF
    * Später reinen Rust-Support
    * Funktionsumfang des reinen Rust-Supports noch deutlich hinter
      ESP-IDF-basiertem
* Mainmatter und Redis
    * Rewrite "im laufenden Betrieb"
    * Parallel Schulung, Coaching und Erfahrungssammeln der Redis-Entwickler
    * Stückweise Transformation
    * Hinweis auf Votrag von Luca Palmieri
* Krypto-Beschleuniger
    * Expertise ist oft nicht vorhanden
-->

---
layout: image-right
image: bsm-ws36a-ed.png
background-size: contain
---

# Anwendungsbeispiel

* Energiezähler der Firma Bauer
* Erweiterung einer bestehenden Bare-Metal-Firmware in C
    * Verarbeitung neuer Eingabedaten
    * Dynamisches Erzeugen von 2D-Codes für ein Zusatzdisplay
    * Authentifizierte Kommunikation zwischen Zähler und Zusatzdisplay
* Projekt ist im Feldtest
* Zertifizierung läuft

---
layout: section
---

# Grundrezept

---

# Überblick

<div class="absolute bottom-5 left-0 right-0 p-2">
    <p align="center">
    <img width="800" src="/integration-overview.svg" />
    </p>
</div>

<!--
* Hinweis: Übertragbar auf Standardumgebungen
-->

---

# Grundlage: C-ABI

* Application-Binary-Interface der Sprache C
    * Konventionen für Funktionsaufrufe
    * Speicherlayout
    * Praktisch für alle Targets verfügbar
* Aufgabe der Integration damit liegt bei Rust
* Rusts Foreign-Function-Interface
    * Erzeugt kompatiblen Code für Aufrufe aus und Einsprungpunkte in Rust
    * Erzeugt kompatible Speicherdarstellungen
    * Einschränkung bei nutzbaren Sprachmerkmalen

<!--
* Speicherlayout
    * Ausrichtung
    * Größe
    * Darstellung von Daten
    * Intern und extern
-->

---

# Rust-FFI: Aufruf von Rust aus C

````md magic-move
```c
int add(const int left, const int right) {
    return left + right
}
```
```rust
use core::ffi::c_int;

#[unsafe(no_mangle)]
pub extern "C" fn add(left: c_int, right: c_int) -> c_int {
    left + right
}
```
````
<v-click>

* Modul `core::ffi` stellt plattformspezifische C-Datentypen bereit
* `#[unsafe(no_mangle)]` deaktiviert Name-Mangling des Rust-Compilers
* `extern "C"` erzeugt zum C-ABI kompatiblen Prolog und Epilog

</v-click>

<!--
* Name Mangling
    * Rust-Compiler erzeugt global eindeutiges Symbol
    * Bei C ist der Nutzer dafür verantwortlich
    * Freude bei Integration von Code
-->

---

# Rust-FFI: Aufruf von C aus Rust

````md magic-move
```c
int square(int value);
```
```rust
use core::ffi::c_int;

extern "C" {
    pub fn square(value: c_int) -> c_int;
}
```
```rust
use core::ffi::c_int;

extern "C" {
    pub fn square(value: c_int) -> c_int;
}

fn main() {
    let squared = unsafe { square(42) };
}
```
````

<v-click>

* Deklaration des Funktionsprototypen als `extern "C"`
* Compiler erzeugt zum C-ABI kompatiblen Code für einen Aufruf
* Aufruf muss über `unsafe` erfolgen
    * Rust-Compiler kann `square` nicht prüfen
    * Nutzer muss prüfen, dass `square` kein undefiniertes Verhalten enthält
    * Mit `unsafe` erklärt der Nutzer dies gegenüber dem Rust-Compiler
    * Zustimmung ist explizit und leicht auffindbar

</v-click>

---

# Rust-FFI: Datentypen

````md magic-move
```c
#include <stdint.h>

typedef struct {
    int32_t x;
    int32_t y;
} Point;
```
```rust
#[derive(Clone, Copy, Debug, Eq, Hash, PartialEq)]
#[repr(C)]
pub struct Point {
    pub x: i32,
    pub y: i32,
}
```
````
<v-click>

* `#[repr(C)]` erzeugt zum C-ABI kompatibles Layout
* In Rust weiterhin ein normaler Datentyp

</v-click>

---

# Panic-Handler

* Zwei Mechanismen zur Signalisierung von Fehlern
    * Im Datenfluss: `Result<T, E>`
    * Außerhalb von Kontroll- und Datenfluss: Panik
* Panik
    * Unmittelbarer Abbruch der Anwendung
    * Aufruf einer Handlerfunktion
* Beispiel: Prozessorausnahme zum Übergang zu bestehender Behandlung in C
    ```rust
    use core::panic::PanicInfo;

    #[panic_handler]
    fn panic(_: &PanicInfo) -> ! {
        cortex_m::asm::udf();
    }
    ```

<!--
* Später mehr zum Result
* `std`
    * Stack-Unwindung
    * Debug-Ausgabe
    * Mit Hilfe von Betriebssystemfunktionen
    * Stellt generischen Handler bereit
    * Handler nutzt Betriebssystem
* `no_std`
    * `no_std`-Umgebung: vielfältig
    * eigenen Handler bereitstellen
    * Keine Standardausgabe
    * Manöver des letzten Augenblicks durchführen (Fehlerzähler, ...)
    * Blockieren oder Reset
* Beispiel
    * Undefined Instruktion
    * Hard-Fault
    * Never-Type `!`: Funktion, die nicht zurückkehrt
-->

---

# Integration: Als Bibliothek

* C- und Rust-Code werden separat übersetzt
* Gesamten Rust-Code zu einer Bibliothek übersetzen
    * Rust-Runtime-Code würde sonst Namenskonflikte erzeugen
    * Gegebenenfalls dedizierte Crate dafür erstellen
    * Übersetzung als Bibliothek über `Cargo.toml` festlegen
        ```toml
        [lib]
        crate-type = ["staticlib"]
        ```
* Wie jede andere C-Bibliothek mit C-Compiler zur Anwendung linken
    ```sh
    $ gcc -o firmware.elf librusty.a ...
    ```
* Geschafft! &#x1f389;

<!--
* Für Mikrocontroller statische Bibliothek
* Lässt sich auch als Binärartefakt integrieren
    * Zum Beispiel bei IDEs
    * Zum Beispiel C-Developer-Tools
-->

---

# Zwischenbilanz

<v-clicks depth="2">

* Grundlage: C-ABI
    * Aufrufkompatible Rust-Funktionen
        ```rust
        #[unsafe(no_mangle)]
        extern "C" fn add(left: c_int, right: c_int) -> c_int
        ```
    * Passende C-Funktionsprototypen
        ```c
        int add(const int left, const int right);
        ```
    * Prototypen für C-Funktionen in Rust
        ```rust
        unsafe extern "C" { square(value: c_int) -> c_int; }
        ```
    * Kompatible Datentypen
        ```rust
        #[repr(C)]
        pub struct Point { x: i32, y: i32 }
        ```
* Gesamten Rust-Code zu einer (statischen) Bibliothek übersetzen 
* C- und Rust-Code zu einer Anwendung linken &#x1f389;

</v-clicks>

<!--
* Es gibt Potential für mehr Komfort
-->

---
layout: section
---

# Gewürze

---

# FFI-Bindings: Automatisches Erzeugen

* Manuelles Erzeugen ist fehleranfällig
    * Anpassung bei Änderungen
    * Komplexität der Schnittstelle
* Werkzeuge zum automatischen Erzeugen nutzen
    * `bindgen` erzeugt Rust-Bindungs für C/C++&#x200b;-Code
    * `cbindgen` erzeugt C/C++&#x200b;-Bindings für Rust-Code
    * Generierung in Build-Prozess integrieren
* Beispiele weiterer Werkzeuge
    * `cxx` für idiomatischere C++&#x200b;-Schnittstellen
    * `cxx-qt` für spezialisierte Integration mit Qt
    * ...
    * Vortrag von Kris van Rens: _Adopting Rust Means Talking to Rust – but
      how?_, heute 16:35 in diesem Saal

<!--
* Beispiel Komplexität: mbedTLS
* Werkzeuge haben Grenzen
    * Kritische Konstrukte müssen gegebenenfalls manuell übersetzen werden
-->

---

# bindgen

* Erzeugt Rust-Bindings für C/C++&#x200b;-Code
    * Aus Header-Datei
    * Für ein bestimmtes Build-Target
    * Gegebenenfalls mit Plattformdefinitionen des Targets (Sysroot)
    * Viele Parameter zum Steuern des Generierungsprozesses
* Nutzt LLVM für Analyse
* Generierung
    * Statisch und generierte Bindings dem Projekt hinzufügen
    * Dynamisch im Übersetzungsvorgang

<!--
* Sysroot:
    * Artefakte des Zielsystems
    * Zum Beispiel `stdio.h`
    * Stammt vom Compiler für Zielsystem oder Zielsystem selbst
* Statisch
    * Keine Integration in Build erforderlich
    * Bindings werden nicht automatisch aktuell gehalten
    * Gut genug für einen ersten Test
* Dynamisch
    * Integration in Build notwendig
    * Bindings automatisch aktuell
-->

---

# bindgen: Nutzung als Befehl

* Befehl als Wrapper um Bibliothek
* Für statische Generierung
* Für Integration in Nicht-Rust-Builds
* Beispiel
    ```sh
    $ bindgen --use-core \
        --newtype-enum mbedtls_md_type_t \
        bindgen/mbedtls_imports.h \
        -- \
        -target thumbv8m.main-none-eabihf \
        --sysroot /opt/arm-gnu-toolchain-13.2.rel1-darwin-arm64-arm-none-eabi/arm-none-eabi \
        -I mbedtls/include \
        > src/mbedtls.rs
    ```

---

# bindgen: Nutzung als Bibliothek

* Direkte Nutzung der Bibliothek in Cargo-Build-Skript `build.rs`
* Für dynamische Generierung
* Beispiel
    ```rust
    fn main() {
        let mbedtls_dir = std::env::var("MBEDTLS_DIR").expect("MBEDTLS_DIR missing");
        let sysroot_der = std::env::var("SYSROOT_DIR").expect("SYSROOT_DIR missing");

        bindgen::Builder::default()
            .use_core()
            .newtype_enum("mbedtls_md_type_t")
            .header("bindgen/mbedtls_imports.h")
            .clang_args(["--sysroot", &sysroot_dir])
            .clang_args(["-I", &mbedtls_dir])
            .generate()
            .expect("unable to generate bindings")
            .write_to_file("src/mbedtls.rs")
            .expect("unable to write mbedlts.rs");

    }
    ```

<!--
* TODO: Visualize with magic move?
-->

---

# bindgen: Header bündeln

* Bindgen nimmt eine einzelne Header-Datei entgegen
* Sollen FFI-Bindings für mehrere Header generiert werden - Wrapper erstellen
* Beispiel
    ```c
    #ifndef MBEDTLS_IMPORTS_H
    #define MBEDTLS_IMPORTS_H

    #include "mbedtls/ecdsa.h"
    #include "mbedtls/ecp.h"
    #include "mbedtls/entropy.h"
    #include "mbedtls/md.h"

    #endif
    ```

---
layout: two-cols-header
---

# bindgen: Beispiel

::left::

C-Funktion

```c {*}{class:'!children:text-[8px]/1'}
/**
 * \brief           This function computes [...]
 */
int mbedtls_ecdsa_write_signature(
    mbedtls_ecdsa_context *ctx,
    mbedtls_md_type_t md_alg,
    const unsigned char *hash,
    size_t hlen,
    unsigned char *sig,
    size_t sig_size,
    size_t *slen,
    int (*f_rng)(void *, unsigned char *, size_t),
    void *p_rng
);
```

::right::

Rust-FFI-Bindings
```rust {*}{class:'!children:text-[8px]/1'}
unsafe extern "C" {
    #[doc = " \\brief           This function computes [...]"]
    pub fn mbedtls_ecdsa_write_signature(
        ctx: *mut mbedtls_ecdsa_context,
        md_alg: mbedtls_md_type_t,
        hash: *const ::core::ffi::c_uchar,
        hlen: usize,
        sig: *mut ::core::ffi::c_uchar,
        sig_size: usize,
        slen: *mut usize,
        f_rng: ::core::option::Option<
            unsafe extern "C" fn(
                arg1: *mut ::core::ffi::c_void,
                arg2: *mut ::core::ffi::c_uchar,
                arg3: usize,
            ) -> ::core::ffi::c_int,
        >,
        p_rng: *mut ::core::ffi::c_void,
    ) -> ::core::ffi::c_int;
}
```

<style>
.two-cols-header {
  column-gap: 2em; /* Adjust the gap size as needed */
}
</style>

<!--
* Beispiel enthält noch keine Typen, wie `mbedtls_ecdsa_context`
-->

---

# bindgen: Limitierungen

* Übersetzt nicht alle Sprachkonstrukte
    * Zum Beispiel werden Definitionen mit Typumwandlung noch nicht unterstützt
        ```c
        typedef uint8_t flag_type;
        #define FLAG_TYPE_C(x) ((flag_type)(x))

        #define SOME_FLAG       FLAG_TYPE_C(0x1)
        #define ANOTHER_FLAG    FLAG_TYPE_C(0x2)
        ```
    * In solchen Fälle manuell erstellen und einbinden
        ```rust
        const SOME_FLAG: u8 = 0x1;
        const ANOTHER_FLAG: u8 = 0x2;
        ```
* bindgen nutzt LLVM - nicht den C-Compiler für das Zielsystem
    * Compiler kann zum Beispiel die Größe eines Enums frei wählen
    * Durch zusätzliche Tests absichern

<!--
* Bei Problemen können einzelne Elemente von Generierung ausgenommen werden
* Enums
    * Dieses Jahr durch Debuggung gelernt
    * Nach 25 Jahren C
        > Each enumerated type shall be compatible with char, a signed integer
        > type, or an unsigned integer type. The choice of type is
        > implementation-defined, but shall be capable of representing the
        > values of all the members of the enumeration.
-->

---

# cbindgen

* Erzeugt C/C++&#x200b;-Bindings für Rust-Code
* Einsatz analog zu bindgen
    * Statische/dynamisch
    * Befehl oder Bibliothek
* Konfiguration über Datei und Annotation im Code
    ```toml
    language = "C"
    cpp_compat = true

    autogen_warning = "// This file is autogenerated by cbindgen. Manual modifiation is futile."
    include_guard = "POINT_H"
    includes = ["some_dependency.h"]

    [enum]
    rename_variants = "QualifiedScreamingSnakeCase"
    ```
* Exportiert standardmäßig alle als `extern "C"` oder `#[repr(C)]`
  gekennzeichneten Elemente

---
layout: two-cols-header
---

# cbindgen: Beispiel

::left::

Rust-Funktion

```rust {*}{class:'!children:text-[8px]/1'}
use core::str::FromStr;
use core::ffi::{c_char, CStr};

#[derive(Clone, Copy, Debug, Eq, PartialEq, Hash)]
#[repr(C)]
pub struct Point {
    x: i32,
    y: i32,
}

#[unsafe(no_mangle)]
pub unsafe extern "C" fn parse_point(s: *const c_char, point: *mut Point) -> bool {
    // ...
}
```

::right::

C-FFI-Bindings

```c {*}{class:'!children:text-[8px]/1'}
#ifndef POINT_H
#define POINT_H

// This file is autogenerated by cbindgen. Manual modifiation is futile.

#include <stdarg.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdlib.h>
#include "some_dependency.h"

typedef struct Point {
  int32_t x;
  int32_t y;
} Point;

#ifdef __cplusplus
extern "C" {
#endif // __cplusplus

bool parse_point(const char *s, struct Point *point);

#ifdef __cplusplus
}  // extern "C"
#endif  // __cplusplus

#endif  /* POINT_H */
```

<style>
.two-cols-header {
  column-gap: 2em; /* Adjust the gap size as needed */
}
</style>

---

# cbindgen: Syntaxfehler und Diagnosemeldungen

* Syntaxfehler von cbindgen
    * Cargo-Build-Skript `build.rs` wird vor Rust-Compiler ausgeführt
    * Fehler brechen Rust-Build ab
    * Das unterdrückt die zugehörigen Fehlermeldungen und erschwert deren Diagnose
* Workaround: Syntaxfehler von cbindgen ignorieren
    * Siehe Dokumentation: [_Quick Start_ >
      _build.rs_](https://github.com/mozilla/cbindgen/blob/main/docs.md#buildrs)
    * Rust-Compiler wird diese später ebenfalls melden
    * Diagnosemeldungen des Rust-Compilers oder von `rust-analyzer` bleiben
      erhalten

<!--
* Ist bei aktiver Arbeit an Rust-Code störend
-->

---

# Fehlerbehandlung: Result

* Rusts erster Weg ist einen Enum als Standard-Rückgabetyp für Operationen, die
  fehlschlagen können
    ```rust
    enum Result<T, E> {
        Ok(T),
        Err(E),
    }
    ```
* Vorteil in Rust-Code
    * Explizite Behandlung der Fälle `Ok` und `Err` notwendig
    * Behandlung kann nicht vergessen werden
    * Fragezeichenoperator `?` zum komfortablen propagieren
* Nachteil beim FFI
    * Keine kompatible Darstellung
    * Explizite Konvertierung notwendig

<!--
* Generische Typen benötigen zwingend Name-Mangling
* Kein Name-Mangling für Funktionen, die von C aus aufrufbar sind
-->

---

# Fehlerbehandlung: Result Konvertieren

```rust {all|2|3-5|9-12|14|14-20|all}
#[unsafe(no_mangle)]
pub unsafe extern "C" fn parse_point(s: *const c_char, point: *mut Point) -> bool {
    if s.is_null() ||  point.is_null() {
        return false;
    }

    // SAFETY: We expect `s` to point to a valid null-terminated C string and
    // actually checked that it's not `NULL`.
    let s = unsafe { CStr::from_ptr(s) };
    let Ok(s) = s.to_str() else {
        return false;
    };

    match Point::from_str(s) {
        Ok(p) => {
            unsafe { point.write(p) };
            return true;
        }
        Err(_) => return false,
    }
}
```

---

# Zeiger

* Rust bietet dafür Raw-Pointer
    * `*const` für konstante Daten
    * `*mut` für veränderliche Daten
* Zugriff über Referenz, wenn Pointer und Daten die [notwendigen
  Anforderungen](https://doc.rust-lang.org/core/ptr/index.html#pointer-to-reference-conversion)
  dafür erfüllen
```rust {all|7|9|10|all}
/// # Safety
///
/// * `value` must be non-null, properly aligned and valid for reading and writing an `i32`.
/// * The memory `value` is pointing at must not be accessed concurrently to this function call.
///
#[unsafe(no_mangle)]
pub unsafe extern "C" fn add_one(value: *mut i32) {
    // SAFETY: The documentation states the requirements for `value` to the caller.
    if let Some(reference) = unsafe { value.as_mut() } {
        *reference += 1;
    }
}
```

* Falls nicht, ist das Erzeugen einer Referenz undefiniertes Verhalten

<!--
* Korrekte Ausrichtung (Alignment)
* Nicht `NULL`
* Dereferenzierbar
* Zeigt auf gültigen Wert des Datentyps
* Muss die Regeln für Aliasing erfüllen
-->

---

# Zeiger: Potentiell undefinierte Daten

* Referenz würde das Lesen der Daten über Safe-Rust ermöglichen
* Schreiben eines Wertes über entsprechende Methode des Raw-Pointers
```rust {all|7|9|all}
/// # Safety
///
/// * `value` must be non-null, properly aligned and valid for writing an `i32`.
/// * The memory `value` is pointing at must not be accessed concurrently to this function call.
///
#[unsafe(no_mangle)]
pub unsafe extern "C" fn init_value(value: *mut i32) {
    // SAFETY: The documentation states the requirements for `value` to the caller.
    unsafe { value.write(0) }
}
```

---

# Zeiger: Arrays

* Arrays aus C können in Rust-Slice gekapselt werden: `slice::from_raw_parts`

```rust {all|4|5-7|9|10-11|12-17|all}
use core::slice;

#[unsafe(no_mangle)]
unsafe extern "C" fn sum_up(values: *const i32, len: usize, sum: *mut i32) -> bool {
    if values.is_null() || sum.is_null() {
        return false;
    }

    let values = unsafe { slice::from_raw_parts(values, len) };
    let checked_sum: Option<i32> = values.iter()
        .try_fold(0i32, |acc, x| acc.checked_add(*x));
    match checked_sum {
        Some(s) => {
            unsafe { sum.write(s) };
            true
        }
        None => false,
    }
}
```

<!--
* TODO: Get line highlighting in code example to work when indented.
-->

---

# Zeiger: Ausgabeparameter

* Ausgabedaten einer Funktion werden in C oft über Ausgabezeiger bereitgestellt
* Raw-Pointer aus Referenz?
    * Werte in Rust müssen stets definiert sein
    * Dummy-Initialisierung vor Übergabe erzeugt unnötige Laufzeitkosten
* `MaybeUninit` kapselt uninitialisierte Daten
```rust {all|5|8-8|10|all}
use crate::sys::mbedtls::{mbedtls_ecdsa_context, mbedtls_ecdsa_init};
use core::mem::MaybeUninit;

fn main() {
    let mut context: MaybeUninit<mbedtls_ecdsa_context> = MaybeUninit::uninit();

    // SAFETY: We are passing in a valid pointer to uninitialized data.
    unsafe { mbedtls_ecdsa_init(context.as_mut_ptr()) };

    let mut context: mbedtls_ecdsa_context = context.assume_init();
    // ...
}
```

---

# FFI-Abstraktionen

* FFI-Bindings sind der Anfang für Interoperabilität
* Sind selten ergonomischer Rust-Code
* Ergonomische und sichere Abstraktion für Rust-Anwendungscode dafür erstellen
* Beispiel
```rust {all|1-3|5-11|all}
struct EcdsaContext {
    inner: mbedtls_ecdsa_context,
}

impl EcdsaContext {
    // ...

    pub fn sign(&self, md: &Digest) -> Result<Signature, Error> { /* ... */ }

    pub fn verify(&self, md: &Digest, sig: &Signature) -> Result<(), Error> { /* ... /* }
}
```

---
layout: section
---

# Garnitur

---

# Dynamischer Speicher

* Ist in Rusts `no_std`-Umgebung optional
* Kann für bestimmten Code erforderlich sein
* Globalen Allokator explizit bereitstellen
    * Implementierung von `core::alloc::GlobalAlloc`
    * Instanz durch Annotation mit `#[global_allocator]` als globalen
      Rust-Allokator setzen
* Kann über das Modul `alloc` genutzt werden
    ```rust
    use alloc::boxed::Box;
    let boxed_point: Box<Point> = Box::new(Point{ x: 1, y: -1 });
    ```
* &#x26A0;&#xFE0F; Fehler bei Allokation führt zu Panik!
* APIs für wählbaren Allokator und Allokationen mit Fehlschlag
    * Vorhanden
    * Aber noch experimentell

<!--
* Gründe
    * Algorithmus
    * Mit C-ABI inkompatible Daten von C halten lassen (Handle/Zeiger)
    * Abhängigkeiten
    * ...
* `core::alloc`
    * Allokator
    * Container
    * ...
* Globaler Allokator
    * Vielfältiges Terrain
    * Explizite Bereitstellung
* Wege gehen
    * Mehr Nutzung
    * Mehr Nachfragen
    * Stabilisierung beschleunigen
-->

---

# Dynamischer Speicher: libc-Allokator

* Oft steht bereits Speicherallokation über `malloc` und `free` in C-Codebasis
  bereit
* Zwingend notwendig, wenn in Rust allokierte Daten von C-Code mit `free`
  freigegeben werden sollen
    ``` rust
    #[unsafe(no_mangle)]
    pub unsafe extern "C" fn new_point(x: i32, y: i32) -> *mut Point {
        let point = Box::new(Point{ x, y });
        Box::leak(point) as *mut Point
    }
    ```
    &#x0020;

    ```c
    const Point *point = new_point(1, -1);
    free(point);
    ```
* Crate `libc_alloc` bietet einen schlüsselfertigen Adapter
    ```rust
    use libc_alloc::LibcAlloc;

    #[global_allocator]
    static ALLOCATOR: LibcAlloc = LibcAlloc;
    ```

<!--
* TODO: Vertikalen Abstand zwischen zweit Codeblöcken aufräumen
* Funktioniert nicht für Objekte, die `Drop` implementieren!
-->

---
layout: two-cols-header
---

# Statisch allokierte Datenstrukturen

* Dynamische Allokation ist in vielen Fällen nicht erforderlich
* Overhead kann vermieden werden
* Crate `heapless` stellt alternative Implementierungen bereit
    * Ähnliche Funktionen
    * Statische Kapazität
* Beispiel

::left::

```rust
extern crate alloc;
use alloc::vec::Vec;

let mut vec = Vec::new();
vec.push(42);
```

::right::

```rust
use heapless::Vec;


let mut vec = Vec::<i32, 3>::new();
vec.push(42)?;
```

<style>
.two-cols-header {
  column-gap: 2em; /* Adjust the gap size as needed */
}
</style>

---

# Ökosystem

* [Rust Embedded Working Group](https://github.com/rust-embedded)
* Matrix Chat: [#rust-embedded:matrix.org](https://matrix.to/#/#rust-embedded:matrix.org)
* Übersicht: [Awesome Embedded Rust](https://github.com/rust-embedded/awesome-embedded-rust)
* Mein zentralen Crates und Werkzeuge
    * Abstraktionen: `embedded-hal`, `heapless`
    * Logging: `defmt`, `log`, `tracing`
    * Datenaustausch: `serde`, `serde-json`, `postcard-rpc`
    * Treiber: ...
    * Build: `corrosion`
    * Debugging: `probe-rs`
* ...
* Danke! &#x2764;

<!--
* Wo fange ich an? Wo höre ich auf?
* Einladende Gemeinschaft
* Großartiges Ökosystem
* Unterstützung bei Vorbereitung des Vortrags
-->

---

# Integration mit anderen Sprachen

* Für Integration mit weiteren Anwendungen
    * Entwicklungswerkzeuge
    * Tests
    * Bedienoberflächen
    * ...
* Vortrag von Kris van Rens
    * _Adopting Rust Means Talking to Rust – but how?_
    * Heute 16:35 in diesem Saal

---

# Zum Mitnehmen

<v-clicks depth="2">

* Integration basiert auf C-ABI
    * Kompatible Funktionsaufrufe
    * Kompatible Datenstrukturen
    * Panic-Handler
    * Gegebnenfalls Allokator
    * Eine Rust-Bibliothek erstellen
    * Rust-Bibliothek in Anwendung linken
* Generatoren für FFI-Bindings nuzten
* Ökosystem erkunden und nutzen
* Trauen Sie sich, Fragen zu stellen
* Beginnen Sie mit kleinen Schritten

</v-clicks>

<!--
* Das hier gezeigte funktioniert 1:1 in reichhaltigerer Umgebung
-->

---

# Empfehlungen

* Übersichtsartikel von James Munns: [_Embedded Rust in Production
  2025_](https://onevariable.com/blog/embedded-rust-production/)
* Vortrag von Mark Russinovich: [_From Blue Screens to Orange Crabs:
  Microsoft’s Rusty Revolution_](https://youtu.be/uDtMuS7BExE?t=17)
* Blog-Artikel von Jeff Vander Stoep: [_Rust in Android: move fast and fix
  things_](https://security.googleblog.com/2025/11/rust-in-android-move-fast-fix-things.html)
* Vortrag von Florian Gilcher: [_Rust: Correctness at Large, Correctness in
  Motion_](https://www.youtube.com/watch?v=DZkW5uozWcw)

* Vortrag von Luca Palmieri: [_Rewrite, optimize, repeat: Our journey from porting a
  triemap from C to Rust_](https://www.youtube.com/watch?v=vv9MKcllekU)
* Blog-Artikel der Fish-Shell zur Migration zu Rust: [_Fish 4.0: The Fish Of
  Theseus_](https://fishshell.com/blog/rustport/)

* Blog-Artikel von Matthias Endler: [_Patterns for Defensive Programming in
  Rust_](https://corrode.dev/blog/defensive-programming/)
* Das Rustonomicon zu [FFI](https://doc.rust-lang.org/nomicon/ffi.html)

---

# Danke für Ihre Zeit!

<!-- TODO: Layout für Zitat anpassen und Workaround entfernen -->
<p>&nbsp;</p>
    <p>
    Die beste Zeit, eine Anwendung mit Rust zu modernisieren, war vor 10 Jahren.
    Die zweitbeste Zeit ist jetzt.
    <br />
    <i>Das Känguru</i>
</p>

<div class="absolute bottom-10 left-0 right-0 p-2">
    <p align="center">
    <img width="250" src="/qr-repo.svg" />
    https://github.com/sirhcel/ese-congress-2025
    </p>
</div>

<!--
* TODO: Layout _end_ anpassen und wieder nutzen?
* Vorher/nachher für Code-Animationen in PDF exportieren
-->

---
layout: section
---

# Dessert

---

# MaybeUninit: Partiell Initialisierte Arrays

<v-clicks depth="2">

* Übergabe eines uninitialisierten Puffers an C-Funktion
    ```c
    bool get_data(int32_t *data, size_t *len);
    ```
* Arbeit damit ist nicht trivial
    * Arrays lassen sich nicht nicht dynamisch zur Laufzeit teilen
    * `MaybeUninit::array_assert_init` wartet auf Stabilisierung
    * Für `[MaybeUninit::<T>; N]` lässt sich `assume_init` nicht partiell
      anwenden
    * Iterieren über initialisierte Elemente und `MaybeUninit::assume_init_ref`
      ist möglich
    * ...
    * &#x1F648;
* `heapless::Vec` bietet komfortable Schnittstelle

</v-clicks>

---

# MaybeUninit: Partiell Initialisierte Arrays

```rust {all|4|7|8-10|12|14-17|18-20|all}
use core::mem::MaybeUninit;
use heapless::Vec;

unsafe extern "C" { fn get_data(data: *mut i32, len: *mut usize) -> bool; }

fn main() {
    let mut data: Vec<i32, 32> = Vec::new();
    let buf: &mut [MaybeUninit<i32>] = data.spare_capacity_mut();
    let mut len = buf.len();

    // SAFETY: `buf` is a non-emty slice of uninitialized data and `len` passed in is taken from that very same slice.
    let good = unsafe { get_data(buf.as_mut_ptr() as *mut i32, &mut len as *mut usize) };

    if good && len <= data.capacity() {
        // SAFETY: We started with an empty `Vec` and therefor the legth of returned elements from `get_data` is the
        // lenght of initialized elements in this `Vec`.
        unsafe { data.set_len(len) }
        for item in data.iter() {
            // Process data ...
        }
    }
}
```
