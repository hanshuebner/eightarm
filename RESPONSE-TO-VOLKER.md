# Antwort an Volker — fast/slow-Klassifizierung vs. Kapitel 9

## Kurzantwort

Volker hat recht mit dem **Charakter** von Kapitel 9 — auf dem Omnibus
ist praktisch alles synchron zu TS-/TP-Flanken, und mit kleinen Margen
(≤ 70 ns für die INTERNAL-I/O-Dekodierung, ≤ 100 ns für C-Lines/SKIP,
20 ns D-FF-Setup, ~50 ns Clock-Pulse-Breiten, der gesamte Bus auf einer
6-MHz-Worst-Case-Logikkette aufgebaut, S. 9‑61).

Er hat **nicht** recht mit der Schlussfolgerung, dass das damit den
Plan in DESIGN.md kippt — denn er liest „fast / slow" so, als würde
das Dokument Signale in „synchron zu einer Bus-Flanke" vs. „asynchron"
einteilen. Das tut es nicht.

## Was „fast" / „slow" in DESIGN.md tatsächlich bedeutet

Die Klassifizierung (§4.1, präzisiert in §3 von
REVIEW_REPLY_2026-04-29) ist **nicht** über Flankentiming, sondern
über die Frage:

> Muss die Karte innerhalb derselben Bus-Cycle (≤ 1,5 µs) auf das
> Signal reagieren, um korrekt mitzuspielen?

Daraus ergeben sich drei Klassen:

* **fast** — Karte muss innerhalb der Cycle beobachten *oder treiben*:
  MA, MD/DATA, MEM_START, IO_PAUSE, WRITE, BUS_STROBE, INT_RQST, IOT-
  Antwort.
* **slow** — Karte braucht das Signal nur in menschlicher Reaktions­
  zeit *und* es ist bidirektional oder rein Ausgabe: Konsolen-Set-and-
  Hold, Status-LEDs, Board-ID-Straps.
* **observe-only synchron** (die dritte Klasse, mit §6.10 in der
  letzten Reviewrunde nachgezogen) — Karte treibt nie und hat keine
  Cycle-Deadline, das Signal ändert sich aber im Bus-Takt: F/D/E, IR,
  LINK, INT_IN_PROG/STROBE, TP1‑4, INHIBIT, RETURN usw.

Die dritte Klasse ist die, um die Volker sich Sorgen macht — und genau
für die wurde §6.10 eingebaut. Diese Signale werden **nicht** über
I²C abgetastet und **nicht** vom M33 gepollt. Sie laufen auf TS1
selbst in das 74HC597-Storage-Register (RCK = TS1‑rising); kurze
Pulse fängt eine 74HC273-Pulse-Latch ab, deren /CLR aus TS1‑rising
RC-differenziert wird. Asynchron zum Bus ist nur das *Auslesen* (PIO
schiebt QH der Daisy-Chain mit 75 MHz raus), und das ist harmlos,
weil das zweistufige 74HC597 Capture und Shift via /STR = ~TS1
entkoppelt.

Anders gesagt: die Karte hält genau die synchrone Bus-Disziplin ein,
die Kapitel 9 fordert; sie tut das nur nicht immer im RP2350 selbst —
zum Teil im 74HC597 / 74HC273 / 74AHC574 *vor* dem RP2350.

## Steht trotzdem etwas aus Kapitel 9 dem Design entgegen?

Ein paar Punkte verdienen einen zweiten Blick — **keiner ist tödlich**,
aber wir sollten sie zusammen durchgehen, damit Volker das nicht im
Vertrauen schlucken muss:

### 1. IOT-Response-Budget

Handbuch S. 9‑39: „A‑C ≤ 100 ns: Zeit, um den IOT zu dekodieren und
die C-Lines oder SKIP zu asserten."

DESIGN.md §6.6 nennt ~100–200 ns Ende zu Ende (PIO-Sample → M33-
Dispatch → Latch-Clock-to-Q → 74540-Propagation). Das Argument der
Karte ist, dass die **harte** Deadline TP2 ist (~600–800 ns nach
IO_PAUSE), und die 100 ns im Handbuch der typische Dekodier-Wert der
DEC-eigenen Peripherie sind, kein Bus-Spec-Maximum.

Das ist korrekt, sollte aber explizit gesagt werden — die Karte
**erfüllt nicht** die 100 ns A‑C-Angabe; sie erfüllt den TP2-Sample-
Punkt. Wenn Volker auf die 100 ns A‑C besteht, muss die IOT-
Dekodierung in PIO laufen, nicht in der M33-Firmware. Diese
Entscheidung explizit zu fällen wäre besser, als sie implizit zu
lassen.

### 2. PIO-abgeleitetes TS2 / TS4

§4.1 schreibt: „TS2/TS4 derived in PIO". Kapitel 9 (Fig. 9‑14, 9‑17)
behandelt TS2 und TS4 als physikalisch bus-getriebene Signale, nicht
als aus TS1+TS3 ableitbar. Für nominale Cycles geht ein idealer Phasen-
Zähler in PIO; TS-Längen können sich aber dehnen
(NOT_LAST_XFER stallt TS3, was sich auf die Position von TS4 auswirkt
— S. 9‑39).

Sicherer wäre, TS2 und TS4 ebenfalls auf direkte GPIO zu legen. Im
Pin-Budget (§4.3) steht zwar „0 spare", aber TS2/TS4 sind als „derived"
gelistet — irgendwo lässt sich ein tradebares Signal finden. Das
sollten wir konkret klären, bevor das Layout final geht.

### 3. D-FF-Setup-Zeiten und AHC-Flanken

S. 9‑59: 20 ns Setup für die DEC-Original-Parts; AHC bringt ~5 ns.
Die 74AHC574-Latches der Karte haben gegen die nominalen 50 ns
MD-Hold-Zeit am WRITE genug Marge — aber die Bus-eigenen Timing-
Pulse sind selbst nur ~50 ns breit (S. 9‑61), und AHC mit ~2 ns
Flanken auf einem unterminierten Backplane ist die Klingel-/
Reflexions-Sorge, die in §10 schon notiert ist.

Offen ist eigentlich nur, ob jemand den realen Backplane mit Scope
gegen die angenommenen Setup-Margen vermessen hat. Bring-up-Schritt 4
(„Karte passiv im laufenden 8/E, alle 74540 aus") ist dafür der
richtige Moment.

### 4. EMA-Pfad-Delay-Match

Steht schon als offene Frage in §10. Kapitel 9 spricht das nicht
separat an, aber die Logikketten-Regeln (S. 9‑61: „20 ns für System-
Variationen einkalkulieren") würden so einen Mismatch fangen.

## Empfehlung für das Gespräch mit Volker

Der Knackpunkt ist, das Gespräch umzuformulieren: Volker fragt „sind
diese Signale alle synchron?" (ja) und folgert daraus, dass der Slow-
Pfad nicht funktionieren kann. Das Design antwortet: „der Slow-Pfad
ist **nur** für Signale ohne Cycle-Deadline — die synchronen
Observe-Only-Signale liegen alle in der §6.10 Trace-Kette, und die
hält Bus-Timing ein."

Wenn §6.10 nicht in der Version war, die er gelesen hat, ist da
vermutlich die Diskrepanz — REVIEW_REPLY_2026-04-29 hat genau diesen
Pfad als Antwort auf seine vorige Reviewrunde eingebaut.

Falls er §6.10 gelesen hat und immer noch unwohl ist, sind die
konkreten Punkte zum gemeinsamen Klären (mit Scope am echten 8/E,
nicht auf Papier):

1. **IOT-Antwort: A‑C 100 ns oder nur TP2?** — Dispatch in PIO oder
   in M33-Firmware?
2. **TS2/TS4 in PIO ableiten oder direkt GPIO?** — was passiert bei
   gestreckten Cycles (NOT_LAST_XFER-Stall)?

Das sind die zwei Stellen, an denen das Design auf Annahmen baut, die
das Handbuch nicht vollständig deckt. Alles andere — Trace-Chain,
Slow-Pfad für Konsole/Housekeeping, Fast-Pfad für Memory- und IOT-
Kernfunktionen — passt sauber zu Kapitel 9.
