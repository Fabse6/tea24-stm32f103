# Fazit zur Übungsaufgabe

## 1. Werkzeugaufbau und GDB/Renode

Renode emuliert das STM32F103-Board mit CPU, Flash, SRAM und Peripherie. Das Renode-Skript lädt die ELF-Datei, richtet den GDB-Server auf Port 3333 ein und stellt die UART-Ausgabe bereit.

Mit `target remote localhost:3333` verbindet sich GDB über das Remote Serial Protocol mit diesem Server. GDB liest Symbole, Typen und Quelltext aus der ELF-Datei und steuert die Ausführung sowie Breakpoints. Renode stellt den emulierten Prozessor, seine Register und den Speicher bereit.

Die Arbeitsteilung ist damit: GDB kennt die Programmstruktur, Renode kennt den aktuellen Zustand der virtuellen Hardware.

## 2. Aufbau und Verbindungen

- `SP` direkt nach dem Start entspricht `0x20005000`, dem Anfang bzw. Ende des 20-KiB-RAM-Bereichs.
- `PC` entspricht der Reset-Handler-Adresse aus Wort 1 der Vektortabelle. Das gesetzte Thumb-Bit wird aus der zum Springen gespeicherten Vektoradresse entfernt; der tatsächliche Wert der nächsten Instruktion hat die Adresse minus 1.
- Wort 0 der Vektortabelle enthält den initialen Stack-Pointer `0x20005000`.
- Wort 1 enthält die Reset-Handler-Adresse mit gesetztem Thumb-Bit, hier `0x0800061d`; die reale Instruktionsadresse ist `0x0800061c`.
- `machine Reset` setzt den Prozessor zurück. Mit `flushregs` verwirft GDB zwischengespeicherte Registerwerte.

Die erste Ausgabe des UARTs wird durch die Zeile `uart_write("STM32F103RB startup OK\r\n");` ausgelöst. Beim freien Weiterlaufen erscheinen außerdem periodische `tick`-Ausgaben.

## 3. ELF, Architektur und Speichersegmente

Die ELF-Datei ist eine 32-Bit-ARM-Ausführungsdatei. Auf einem normalen x86-/x64-PC kann sie nicht direkt ausgeführt werden, weil sowohl die Prozessorarchitektur als auch der bare-metal-Ausführungskontext anders sind.

### Linker-Abschnitte

| Variable/Symbol | Adresse | Speicher | Abschnitt |
|---|---:|---|---|
| `g_data_counter` | `0x20000000` | RAM | `.data` |
| `g_bss_counter` | `0x20000174` | RAM | `.bss` |
| `g_bss_buffer` | `0x20000178` | RAM | `.bss` |
| `kRodataBanner` | Flash-Adresse | Flash | durch das Linker-Script in `.text` eingebunden |
| `main` | `0x08000570` | Flash | `.text` |
| `stack_probe` in `print_memory_layout()` | lokale Stack-Adresse | RAM | Stack |
| `current_heap_end` in `_sbrk()` | statische Adresse im BSS-/Stativbereich | RAM | `.bss` |

Der Name einer Variablen mit `nm -C` kann anzeigen, ob sie in `.data`, `.bss`, `.rodata` oder `.text` liegt. Für `.data` und `.bss` werden Laufzeitadressen im RAM benötigt, weil diese Variablen beim Programmablauf verändert werden können. Code und konstante Daten liegen im Flash, weil sie dort dauerhaft gespeichert werden können.

Für `.data` sind VMA und LMA unterschiedlich:

- VMA: Adresse im RAM zur Laufzeit
- LMA: Ladeadresse im Flash

Der Abschnitt `.bss` hat keine Startwerte im Flash und wird zur Laufzeit mit Null gefüllt.

Die Größe von `.data` beträgt:

```text
_edata - _sdata = 0x20000158 - 0x20000000
               = 344 Bytes
```

Die Größe von `.bss` beträgt:

```text
_ebss - _sbss = 0x20000470 - 0x20000158
              = 792 Bytes
```

Damit beginnt `.data` bei `0x20000000` und `.bss` endet bei `0x20000470`.

## 4. `.rodata`

In der verlinkten ELF-Datei gibt es keinen eigenen Ausgabeabschnitt `.rodata`, weil das Linker-Script `*(.rodata*)` in den Ausgabeabschnitt `.text` einordnet. In der Objektdatei existiert `.rodata` jedoch noch.

Die Objektdatei enthält die Abschnitte `.text`, `.rodata`, `.data` und `.bss`; der Banner-Text liegt in `.rodata`, bevor der Linker ihn in den ausführbaren Ausgabeabschnitt `.text` übernimmt.

Für die Hardware macht eine Zusammenlegung mit `.text` keinen Unterschied, weil beide in den Flash geladen werden und dort unverändert bleiben. Ein eigener `.rodata`-Abschnitt ist dennoch übersichtlicher: Die Größenanalyse trennt Code und Konstanten, Speicherbereiche können gezielt beschrieben werden und Speicherschutz oder Leserechte lassen sich genauer festlegen.

## 5. Startup-Code und Initialisierung

Beim Reset arbeitet der Cortex-M zuerst hardwareseitig:

1. `SP` wird aus Wort 0 der Vektortabelle geladen.
2. `PC` wird aus Wort 1 der Vektortabelle geladen.

Danach läuft der Reset-Handler:

1. `.data` wird wortweise aus dem Flash (LMA) in den RAM (VMA) kopiert.
2. `.bss` wird wortweise mit Nullen gefüllt.
3. C-/C++-Initialisierer und globale Konstruktoren werden über `__libc_init_array()` aufgerufen.
4. `main()` wird aufgerufen.
5. Kehrt `main()` zurück, bleibt der Startup-Code in einer Endlosschleife stehen.

Die Initialisierung ist nötig, weil nicht initialisierter SRAM nach dem Einschalten zufällige Werte enthalten kann. `.data` braucht Startwerte aus dem Flash, und C/C++ garantiert für `.bss`-Variablen den Wert Null.

Der Reset-Handler verwendet die Schleifen `dst < &_edata` und `dst < &_ebss`. Die erste kopiert `.data`, die zweite löscht `.bss`. `uint32_t*` verwendet 32-Bit-Wörter statt einzelner Bytes und ist auf dem 32-Bit-ARM schneller. Das 4-Byte-Alignment aus dem Linker-Script sorgt dafür, dass die Quell- und Zieladressen für Wortzugriffe passend ausgerichtet sind.

Bei `.data` werden `344 / 4 = 86` Wörter kopiert.

`__libc_init_array()` ruft Konstruktoren und weitere C++-Initialisierer aus `.preinit_array` und `.init_array` auf. Dieser Schritt muss nach dem Kopieren und Nullsetzen erfolgen, damit Initialisierer bereits gültige globale Variablen vorfinden.

`Default_Handler` ist eine Endlosschleife als Standardbehandlung unerwarteter Exceptions und Interrupts. Schwache, auf ihn aliasierende `*_IRQHandler` können in `main.cpp` durch eine starke eigene Definition ersetzt werden. Damit verwendet die Vektortabelle den projektspezifischen Handler. Falls kein eigener Handler existiert, landet die CPU im `Default_Handler`.

### Experiment zur Initialisierung

- Direkt nach dem Reset ist `g_data_counter` noch nicht initialisiert und enthält in der Regel `0`.
- In `main` hat `g_data_counter` den Startwert `0x12345678`. Dieser Wert stammt aus dem Flash-Abbild von `.data` und wird vom Reset-Handler in den RAM kopiert.
- Auch wenn `g_bss_counter` vorher auf `0xDEADBEEF` gesetzt wurde, enthält es bei `main` wieder `0`.
- Das Nullen führt der Reset-Handler in der `.bss`-Schleife aus.
- Der Watchpoint auf `g_data_counter` hält beim Kopier-Schritt des Reset-Handlers, nicht in `main`.
- `main.cpp` überschreibt den schwachen `SysTick_Handler` mit einer eigenen Funktion.

## 6. GDB-Befehle

- `break`: Setzt einen Haltepunkt an Funktion, Quellzeile oder Adresse.
- `continue`: Lässt das angehaltene Programm bis zum nächsten Breakpoint, Watchpoint oder manuellen Halt weiterlaufen.
- `next`: Führt die nächste Quellzeile aus und läuft über Funktionsaufrufe hinweg.
- `step`: Führt die nächste Quellzeile aus und steigt in aufgerufene Funktionen ein.
- `finish`: Führt die aktuelle Funktion bis zum Rücksprung aus.
- `x`: Liest rohen Speicher ab einer Adresse und formatiert ihn.
- `print`: Wertet Variablen, Ausdrücke oder Register aus.
- `info registers`: Zeigt die Prozessorregister.
- `bt`: Gibt die aktive Aufrufkette aus.
- `watch`: Hält an, wenn eine Variable oder Speicheradresse geschrieben oder gelesen wird.

Weitere nützliche Befehle sind `disassemble`, `info frame`, `info symbol`, `x/i` und `monitor`.

## 7. Stack

Die lokale Variable `local` liegt in einem Stack-Frame. Die Rekursionsebenen zeigen mit ihren Adressen, dass der Cortex-M-Stack zu niedrigeren Adressen wächst: Der Stack-Pointer wird bei Funktionsaufrufen und Prolog-Code dekrementiert.

Die Rekursion misst für jede Ebene die Adresse von `local`:

| Ebene (`depth`) | Adresse von `local` | Differenz zur vorherigen Ebene |
|---|---:|---:|
| 0 | Live-Wert aus GDB | – |
| 1 | Live-Wert aus GDB | Live-Differenz |
| 2 | Live-Wert aus GDB | Live-Differenz |
| 3 | Live-Wert aus GDB | Live-Differenz |
| 4 | Live-Wert aus GDB | Live-Differenz |
| 5 | Live-Wert aus GDB | Live-Differenz |

Bei einem typischen `stackDemo`-Prolog sichern `push {r7, lr}` acht Bytes, `sub sp, #16` reserviert Platz für lokale Daten und die gerundeten Frame-Elemente. Der konkrete Frame aus GDB muss jedoch zur Compiler-Version passend gemessen werden.

Ein Stack-Frame enthält gesicherte Register, die gespeicherte Rücksprungadresse `lr`, den Frame-Pointer `r7`, lokale Variablen und eventuelles Alignment-Padding. Der Epilog gibt den Platz mit `add sp` frei und stellt `r7` sowie `pc` aus den gesicherten Werten wieder her.

Der Stack wächst nach unten, weil Aufruf- und lokale Datenbereiche durch `push` oder `sub sp` zu kleineren Adressen angelegt werden. Bei einem Aufruf wird die Rücksprungadresse in `lr` gesichert und vom Prolog auf den Stack gelegt. Wird sie durch einen Pufferüberlauf überschrieben, kann ein späterer Funktionsrücksprung auf eine falsche oder manipulierte Adresse springen.

Zwischen `_stack_top` und dem SP in `main` liegt der Platz, den der Startup-Pfad und die aufgerufenen Funktionen bereits belegt haben. Der genaue Abstand wird mit `&_stack_top - $sp` gemessen.

### Stack Overflow

Bei einer Rekursionstiefe von 2000 wächst der Stack zuerst in den freien Heap-/RAM-Bereich und kann anschließend weiter in `.bss` und `.data` hineinwachsen. Die beobachtete Variable `g_data_counter` wird dabei nicht vom Quelltext, sondern durch Speicherüberschreibung verändert.

Auf einem Mikrocontroller gibt es kein Betriebssystem und keine MMU mit einer automatischen „Segmentation fault“-Meldung. Speicherbereiche werden deshalb still überschrieben, solange keine separate Schutzlogik eingreift. Folgefehler können sich erst später als falsche Werte oder HardFault zeigen.

Mögliche Schutzmaßnahmen:

- Rekursion und unnötig große lokale Puffer vermeiden.
- `_Min_Stack_Size` und damit reservierten Stack passend festlegen.
- Stack-Painting verwenden und den unveränderten Musterbereich messen.
- Eine Memory Protection Unit einsetzen, wenn sie verfügbar ist.
- Aufrufstiefe begrenzen und kritische Funktionen messen.

Die Änderung auf Rekursionstiefe 2000 wird wieder auf `5` zurückgesetzt.

## 8. Heap und `_sbrk`

Beim ersten Aufruf von `_sbrk(64)` gibt `_sbrk` die vorherige Heap-Grenze, also `_heap_start`, zurück. Danach wird `current_heap_end` um 64 Bytes erhöht.

Der erste Rückgabewert ist `0x20000490`, der mit `&_heap_start` übereinstimmt. Insgesamt stehen ungefähr:

```text
_heap_end - _heap_start
= 0x20004800 - 0x20000490
= 0x4370
= 17.264 Bytes
```

`_sbrk` gibt `(void*)-1` zurück, wenn der angeforderte Zuwachs negativ ist oder die neue Heap-Grenze über `_heap_end` hinausgehen würde. Bei einem Aufruf mit `0x10000` kann daher kein ausreichender Speicher bereitgestellt werden. Die Firmware meldet anschließend einen Heap-Allokationsfehler.

`current_heap_end` ist eine statische lokale Variable in `_sbrk()` und liegt im statischen RAM-Bereich, voraussichtlich `.bss`, da sie mit Null startet. Der Heap wächst nach oben, der Stack nach unten. `_sbrk` begrenzt den Heap nur mit `_heap_end`; es kann nicht verhindern, dass ein sehr großer Stack in den Heap hineinwächst.

Bei `malloc(32)` ruft `_malloc_r` beziehungsweise der C-Laufzeit-Allokator `_sbrk` auf. `malloc` fordert meist einen größeren Speicherblock an als die angefragten Nutzdaten, weil er Verwaltungsdaten und Alignment berücksichtigen und Speicher in größeren Chunks vorhalten möchte. `g_heap_ptr` liegt nicht im Heap: Die Zeigervariable liegt in `.bss`, der von ihr adressierte Speicherblock liegt im Heap.

### Vergleich Stack und Heap

| Eigenschaft | Stack | Heap |
|---|---|---|
| Verwaltung | CPU über `SP` und Funktionsaufrufe | Allocator wie `malloc`/`free` beziehungsweise `_sbrk` |
| Wachstumsrichtung | nach unten zu niedrigeren Adressen | nach oben zu höheren Adressen |
| Belegung | automatisch beim Aufruf von Funktionen und beim Anlegen lokaler Variablen | dynamisch durch explizite Speicheranforderung |
| Freigabe | automatisch beim Verlassen des Funktionsframes | durch `free` oder `delete` |
| Typische Risiken | Stack Overflow und Überschreiben gespeicherter Rücksprungadressen | Out-of-Memory, Speicherlecks und Fragmentierung |

## 9. Gesamtspeicherkarte

```text
0x20005000  +----------------------------+  _stack_top
            | Stack                      |  wächst nach unten
            |             ↓              |
            | freier Speicher            |
            |             ↑              |
            | Heap                       |  wächst nach oben
0x20000490  +----------------------------+  _heap_start
            | .noinit                    |
0x20000470  +----------------------------+  _ebss
            | .bss                       |
0x20000158  +----------------------------+  _sbss
            | .data                      |
0x20000000  +----------------------------+
```

Das Flash-Abbild enthält an niedrigen Adressen die Vektortabelle und danach `.text` beziehungsweise die darin eingebundenen Konstanten. Die Startwerte für `.data` liegen im Flashbereich mit LMA `0x080000EC` bis ungefähr `0x080000FE`:

```text
0x08000000  Vektortabelle / .isr_vector
            .text
            .rodata (ggf. in .text eingebunden)
            LMA der Startwerte von .data
```
