# Antwort auf das Reviewer-Feedback (Stand 2026-04-29)

Hallo,

danke fürs genaue Drüberschauen. Hier die Antworten zu den vier Punkten;
die Stellen, an denen ich das Designdokument nachgeschärft habe, sind im
Diff nachvollziehbar.

## 1. „TRISTATE" bei den 74244 / 74540 — geht das mit Open-Collector?

Mit *Tri-State* ist gemeint, dass der Ausgang drei Zustände kennt: H, L
und hochohmig (Z). Das ist **nicht** dasselbe wie Open-Collector. Ein
Tri-State-Treiber ist ein Push-Pull-Treiber, der im aktiven Zustand
sowohl gegen H als auch gegen L treibt; nur im hochohmigen Zustand ist
er abgeschaltet. Open-Collector kann dagegen ausschließlich gegen L
treiben — H entsteht passiv über den Pull-up der Bus-Terminierung.

Für die Wired-OR-Signale des Omnibus (insbesondere `~INT_RQST`)
emuliere ich Open-Drain mit dem Tri-State-Treiber, indem der Daten-
eingang am 74540 fest auf den asserting Pegel verdrahtet wird und der
RP2350 ausschließlich das Output-Enable des 74540-Kanals umschaltet:
OE low → Bus wird low gezogen, OE high → Z, der Bus-Pull-up gewinnt.
Damit kann der Treiber den Bus nie aktiv gegen H treiben und steht
folglich nie in Konflikt mit einer anderen Karte, die `~INT_RQST`
parallel zieht. Das habe ich in §6.7 jetzt explizit dokumentiert.

### Anmerkung zur Anstiegsflanke

Der Hinweis ist berechtigt. AHC/AHCT hat typische Transition-Zeiten
von ~2 ns, das DEC-Backplane wurde aber für TTL aus der Ära 74H/74S
(~5–10 ns) ausgelegt und ist weder geziehlt terminiert noch impedanz-
kontrolliert. Auf einer langen unterminierten Leitung führt das
schnell zu Ringing und Reflexionen, die auf Nachbarkarten als
Fantom-Flanken sichtbar werden können.

Als Maßnahme habe ich in §10 (Open Questions) einen Eintrag ergänzt:
auf jedem 74540-Ausgangskanal, der direkt an einen Kartenrand-
Kontakt geht, wird ein Footprint für einen 33–47 Ω Serienwiderstand
vorgesehen. Bestückt wird nur, falls in Bring-up-Schritt 3 (Karte im
laufenden PDP-8/E, alle Treiber aus) Ringing am Oszilloskop sichtbar
ist. Falls AHC sich als zu aggressiv erweist, ist 74HC ein logischer
Drop-In (~6 ns Flanken) — reine BOM-Änderung, keine Schaltplan-
Änderung.

## 2. MD0..MD11 fehlen in §4.1

Stimmt — das war nicht eindeutig formuliert. MD0..MD11 (12 Signale)
und DATA0..DATA11 (12 Signale) liegen **nicht** als 24 separate Pins
am RP2350 an, sondern teilen sich einen gemeinsamen 12-Bit-Port
(`D0..D11`). Das funktioniert, weil Memory- und IOT-Cycles zeitlich
exklusiv sind. Die physikalische Trennung übernehmen die vier
74AHC574-Latches aus §5: MD-IN/MD-OUT für Memory, DATA-IN/DATA-OUT
für IOT.

Die 96 aktiven Signale aus §3.11 setzen sich nach wie vor zusammen:

* 15 MA + EMA (direkt am RP2350)
* 12 MD (über Latch + Shared D-Port)
* 12 DATA (über Latch + Shared D-Port)
* 8 TS/TP (TS1/TS3 direkt, TS2/TS4 in PIO abgeleitet, TPx über die
  §6.10 Trace-Kette — siehe Sektion 5)
* 8 Memory Control
* 10 Major State / IR / DMA Codes
* 10 Break / Interrupt / IOT
* 21 Front Panel / CPU State
* = 96

In §4.1 ist das jetzt expliziert: die Tabellenzeile nennt sowohl MD
als auch DATA, und ein zusätzlicher Absatz davor erklärt das Sharing
und verweist auf §5.

## 3. LINK und Interrupt-Steuerung — wirklich „slow"?

Die Klassifizierung *fast* / *slow* bezieht sich nicht auf die Natur
des Signals, sondern auf folgende Frage:

> Muss die Karte das Signal innerhalb einer Bus-Cycle (~1.5 µs)
> beobachten oder treiben, um korrekt mitzuspielen?

Mit dieser Definition:

* **LINK / LINK_DATA / LINK_LOAD** sind Teil des AC-Statusregisters.
  Die Karte beobachtet sie nur als CPU-Zustand (für Konsole,
  Status-LEDs, Debug) — sie sind nicht zykluskritisch. Im
  ursprünglichen Entwurf lagen sie auf MCP23017 mit ~20 ms
  Update-Intervall; im finalen Entwurf werden sie über die
  §6.10 Trace-Kette zyklus-genau erfasst (siehe Sektion 5).
* **INT_IN_PROG / INT_STROBE** signalisieren den CPU-internen
  Interrupt-Acknowledge-Ablauf. Die Karte hat genau zwei zeit-
  kritische Verpflichtungen rund um Interrupts: `~INT_RQST` zu
  asserten, wenn das emulierte Gerät interrupten will, und es
  innerhalb der IOT-Cycle wieder zu deasserten, in der das OS das
  Device-Flag löscht (KCF, TCF …). Beide laufen auf dem schnellen
  Pfad — `~INT_RQST` direkt am GPIO, der IOT über §6.6.
  INT_IN_PROG und INT_STROBE sind dagegen rein CPU-interne
  Status-Signale, die dem Bus mitteilen „ich mache gerade die
  Interrupt-Entry-Sequenz" (PC → Adresse 0, Sprung Adresse 1, ION
  löschen). Die Karte hat keine Aktion, die durch ihre Flanken
  getriggert werden müsste; sie liest sie nur für die Konsolen-
  Statusanzeige („Interrupt wird gerade abgearbeitet") und für
  Debug-Traces. Das entspricht dem Standard-PDP-8-Modell:
  Interrupt-Dispatch erfolgt softwareseitig per Skip-on-Flag-IOTs,
  nicht hardware-vektoriert über INT_STROBE. Damit reicht eine
  edge-getriggerte Auswertung nicht zwingend nötig — ich habe das
  in §6.7 jetzt explizit so dokumentiert. (Im finalen Entwurf
  liegen INT_IN_PROG und INT_STROBE in der §6.10 Trace-Kette und
  sind damit ohnehin zyklus-genau beobachtbar — siehe Sektion 5.)
* **`~INT_RQST` dagegen ist fast** und liegt direkt am RP2350-GPIO,
  weil die *Deassertion* zeitlich an die acknowledging IOT-Cycle
  gekoppelt ist — würde sie länger als die Cycle dauern, würde der
  nächste Befehl wieder als Interrupt interpretiert. I²C-Latenz wäre
  dafür zu langsam.

In §4.1 habe ich das Klassifizierungs-Kriterium jetzt explizit als
Definition davorgestellt, sodass das nicht mehr aus dem Kontext
erschlossen werden muss.

## 4. Was bedeuten FAST und SLOW konkret? Was ist die 1 MHz?

Die 1 MHz im Dokument ist der **I²C-Bustakt** (Fast-Mode-Plus), nicht
eine Sampling-Rate der Slow-Signale. Konkret:

| Pfad | Latenz Bus-Edge → Firmware sichtbar |
|---|---|
| FAST direkt: 74244 → RP2350-GPIO → PIO | ~10–20 ns (1 PIO-Clock @ 150 MHz) |
| FAST über Latch: 74244 → 74AHC574 (CK = WRITE/STROBE) → RP2350 | ~15 ns Capture, M33-Read folgt nach ~50 ns |
| TRACE-Kette (§6.10): 74244 → 74HC597 (+ 74HC273 für Pulse) → PIO → DMA → SRAM | Sample bei jedem Bus-Cycle (~1.5 µs Auflösung); ~430 ns für die Schiebefolge |
| SLOW: MCP23017 → I²C @ 1 MHz → Schatten-Register | ~20 ms im Default-Polling (50 Hz), ~80 µs im sofort-Poll |

Das volle Auslesen der zwei (im finalen Entwurf verbliebenen)
MCP23017 — 2 × 16 Bit Datenbytes plus Adress-Overhead auf
1-MHz-I²C — dauert ~80 µs. Das Default-Polling läuft mit 50 Hz
(alle 20 ms) und ist deutlich schneller als jede menschliche
Reaktion — für Set-and-Hold-Konsolenausgaben, Status-LEDs und
Housekeeping reicht das problemlos. Eine Konsolen-Abfrage (`status`
über CDC0) liest stattdessen den letzten Trace-Eintrag aus dem
Ringpuffer (§6.10) und ist damit nahe-instantan.

Kein Slow-Signal wird je auf einer Bus-Cycle-Deadline gelesen.
Sollte sich das später ändern (z. B. BREAK-Cycle-Unterstützung),
beschreibt §4.3 den Trade-out-Mechanismus: einen Slow-Pin auf einen
freien RP2350-GPIO promoten und im Gegenzug einen weniger zeit-
kritischen Fast-Pin entweder auf MCP23017 #3 oder auf einen
freien Slot der Trace-Kette verlagern — Firmware- und Routing-
Änderung, keine BOM-Änderung.

§4.1 hat jetzt eine Latenz-Aufstellung in Prosa, §4.2 eine ergänzte
Erklärung, dass die 1 MHz die Bus-Frequenz ist und nicht die
Polling-Rate.

## 5. Konsequenz: vollständiges Bus-Tracing — jetzt eingebaut

Der Reviewer hat damit implizit einen wichtigen Punkt aufgemacht:
durch die ursprüngliche Aufteilung in Fast- und Slow-Path **konnte
die Karte keinen zyklus-genauen Trace aller 96 Omnibus-Signale
liefern** — Major-State (F/D/E), IR, LINK, BREAK-Signale,
INT_IN_PROG/STROBE, TPx etc. lagen alle auf MCP23017 und wurden nur
mit ~50 Hz abgetastet.

Anstatt das als bleibende Limitation stehen zu lassen, habe ich
**Option (b) aus meiner ersten Antwort in den Entwurf eingearbeitet**
(neuer §6.10): einen High-Speed-Shift-Register-Trace-Pfad, der die
Slow-Signale bei jedem Bus-Cycle einsammelt.

**Hardware**:

* 4× **74HC597** (8-Bit Parallel-Load Shift Register, *zweistufig*:
  getrenntes Storage- und Shift-Register), daisy-chained über
  QH→DS ⇒ 32 Bit pro Sample. Zweistufig ist wichtig, damit das
  Capture (synchron zum Bus) und das Auslesen (per PIO) entkoppelt
  laufen.
* **RCK = TS1** (positiv-true). Bei jeder TS1-steigenden Flanke
  klinkt das Storage-Register die Eingänge ein — also ein Sample
  am Anfang jeder Bus-Cycle, das den Zustand am Ende der
  vorherigen Cycle festhält.
* **/STR = ~TS1**. Bei TS1-fallender Flanke (Beginn TS2) wird das
  Storage-Register ins Shift-Register kopiert; PIO kann ab da in
  Ruhe schieben (~427 ns für 32 Bit) und ist lange vor dem
  nächsten TS1 fertig.
* 1× **74HC273** Octal-D-FF mit /CLR als **Pulse-Latch-Bank** für
  die kurzen Puls- und Phasensignale, die ein einzelnes End-of-
  Cycle-Sample sonst verfehlen würde:
  - **TP1, TP2, TP3, TP4** — ~100 ns Pulse an festen Phasen
    innerhalb der Cycle.
  - **INT_STROBE** — kurzer Puls beim Interrupt-Entry.
  - **LINK_LOAD** — kurzer Puls beim LINK-Update.
  - **INHIBIT, RETURN** — Memory-Cycle-Phasensignale, die zur
    Sample-Zeit längst wieder deassertiert wären.
  Jedes dieser 8 Signale taktet einen D-FF (D=high), dessen Q
  bis zum nächsten /CLR-Puls high bleibt. /CLR wird via RC-
  Differenzierer (1 kΩ + 100 pF) aus TS1-rising erzeugt — ein
  74AHC1G14 Schmitt-Trigger-Inverter glättet die Flanke.
  Damit liefert die 74HC273 für jede Cycle die Antwort
  „dieser Puls ist gefeuert / dieses Phasensignal war aktiv".
* Die 24 Level-Signal-Eingänge der 597-Kette und die 8 D-Eingänge
  der 273 sind allesamt „Y"-Abgriffe der vorhandenen 74244-
  Eingangspuffer — selbe Buffer-Stufe, matched delay.

**PIO + DMA**:

* PIO2.SM1 schiebt 32 Bit bei sysclk/2 (= 75 MHz, 13.3 ns/Bit) ⇒
  ~427 ns pro Sample. Beginnt bei TS1-fallender Flanke, fertig
  weit vor Ende von TS2.
* PIO2.SM2 nimmt parallel einen 32-Bit-Snapshot der Fast-Path-
  GPIOs auf derselben TS1-IRQ.
* DMA fasst beide zu einem 64-Bit-Eintrag (8 B) zusammen und legt
  ihn in einen Ringpuffer in RP2350-SRAM ab.
* Default-Ring 256 KB ⇒ ~32k Einträge ⇒ ~50 ms Trace bei vollem
  Bus-Tempo (660 kHz × 8 B = 5.3 MB/s).
* Bei stehender CPU (RUN low, kein TS1) friert die Trace-Kette
  ein. Das ist OK: die fraglichen Slow-Signale (F/D/E, IR, LINK,
  USER_MODE …) sind dann statisch — der jüngste Trace-Eintrag
  spiegelt den aktuellen Zustand korrekt wider.

**Trigger / Filter über CDC0**:

```
> trace start ring                      # Dauer-Capture
> trace start trigger 'MEM_START & MA == 07777'
> trace start match 'IO_PAUSE & device == 03'
> trace dump 1024 binary
> trace save snapshot.bin               # auf SD-Karte
```

**Pin-Bilanz**:

* Trace braucht 2 RP2350-GPIO (TRACE_SCK, TRACE_SDI).
* Frei werden 2 Pins durch Demotion zweier Fast-Path-Signale, die
  in der 597-Kette mitgeführt werden:
  - **INTERNAL_IO** ist redundant — die Karte erkennt interne
    IOTs am MD-Device-Code (0..7).
  - **SOURCE** wird in die 597-Kette gelegt, sofern §10 zu
    „RP2350-input-only" auflöst (TBD).
* Netto bleiben es 48 GPIO ⇒ 48 belegt, alles passt.

**BOM-Bilanz**:

* −2× MCP23017 (die Observe-Only-Expander #0/#1 fallen weg, alle
  ihre Signale sind jetzt in der Trace-Kette) = −$3.24
* +4× 74HC597 = +$2.00
* +1× 74HC273 (Pulse-Latch-Bank) = +$0.30
* +1× 74AHC1G14 (Schmitt-Trigger-Inverter für /CLR) = +$0.12
* + RC-Glied für /CLR (1 kΩ + 100 pF) — im Passives-Kit enthalten.
* **Netto: −$0.82 pro Board**, plus Reduktion der I²C-Buslast von
  8 Devices auf 6.

Konkret heißt das: die Karte wird **selbst zum Logic-Analyser**.
Major-State, IR, LINK, BREAK-Cycle-Verhalten, Interrupt-Entry-
Sequenz, alle TPx-Pulse — alles cycle-akkurat aufgezeichnet, alles
über CDC0 oder SD-Karte abgreifbar, alles ohne externe
Instrumentierung. Für Bring-up und für die Diagnose von Software
auf der echten PDP-8 ist das ein deutlich erweitertes Werkzeug
gegenüber dem ursprünglichen Entwurf — und kostet sogar etwas
weniger.

Konsequenzen im Dokument:

* Neuer §6.10 mit der vollständigen Beschreibung (Hardware, PIO,
  Ringpuffer, Trigger, Halted-CPU-Fallback, Kosten).
* §2.2 BOM, §4.1 GPIO, §4.2 Expander-Tabelle, §4.3 Pin-Budget,
  §6.3 PIO-Allocation, §6.9 BREAK-Path, §9 Bring-up, §11 Appendix A,
  §13 BOM — alle entsprechend angepasst.
* §10: der frühere „Bus-Tracing-Fidelity"-Eintrag ist gelöscht
  (durch §6.10 erledigt).

Viele Grüße,
Hans
