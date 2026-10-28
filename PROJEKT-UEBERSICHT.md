# Gratiswaren-Kontrolle – Projektübersicht

Übergabe-Dokument, um im normalen Claude-Chat weiterzuarbeiten.
**Im neuen Chat bitte immer die aktuelle `index.html` (und bei Bedarf `sw.js`) mit hochladen.** Ohne die Datei kann Claude nichts gezielt ändern.

Stand: 28.10.2026

---

## 1. Worum es geht

- **Nutzer:** Dan, Abteilungsleiter Getränke bei Edeka Wehrmann.
- **Zweck:** Aktionsflächen (Kassenregale, Stirnseiten) werden wochenweise an Lieferanten vergeben, als Gegenleistung gibt es Gratisware. Die App hält fest, was vereinbart ist, was geliefert wurde, was verbucht ist und was noch offen ist.
- **Geräte:** iPad (Hauptgerät) und iPhone, als Web-App auf dem Home-Bildschirm. Funktioniert offline.

## 2. Wo liegt was

| Was | Wo |
|---|---|
| App (öffentliche Adresse) | https://dant82.github.io/gratisware/ |
| Quellcode / Hosting | GitHub, Konto `DanT82`, Repository `gratisware`, GitHub Pages (Branch `main`, Ordner `/`) |
| Cloud-Datenbank | Firebase-Projekt `gratisware-ba0db`, Firestore, Standort europe-west3 (Frankfurt), kostenloser Spark-Tarif |
| Anmeldung | Firebase Authentication, nur **E-Mail/Passwort** (Google-Anmeldung bewusst nicht, weil sie in iPad-Home-Bildschirm-Apps oft hakt) |

### Dateien im Repository (alle direkt im Hauptordner, kein Unterordner)
- `index.html` – die komplette App (HTML, CSS, JavaScript in einer Datei, ca. 140 KB)
- `sw.js` – Service Worker, hält die Dateien für den Offline-Betrieb auf dem Gerät
- `manifest.webmanifest` – Name, Symbol und Farben für „Zum Home-Bildschirm“
- `icon-180.png`, `icon-192.png`, `icon-512.png` – App-Symbol (grünes Feld, gelbes Preisschild „GRATIS“)
- `fb-app.js`, `fb-auth.js`, `fb-firestore.js` – Firebase JavaScript SDK **10.14.1, Compat-Version**, lokal mitgeliefert (nicht vom CDN), damit die App offline startet

### Firestore-Regeln (veröffentlicht)
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{uid}/{document=**} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
  }
}
```

### Firebase-Konfiguration (steht so in `index.html`, ist nicht geheim)
```
apiKey: "AIzaSyBNA7NhqkFdKXhTKXnEpj5cUPO9acbbAJ0"
authDomain: "gratisware-ba0db.firebaseapp.com"
projectId: "gratisware-ba0db"
storageBucket: "gratisware-ba0db.firebasestorage.app"
messagingSenderId: "1004419186622"
appId: "1:1004419186622:web:8227595c3e4b1f3691c3ef"
```

## 3. Technik in Kürze

- Reines HTML/CSS/JavaScript ohne Framework und ohne Build-Schritt. Alles steckt in `index.html` in einer großen Funktion `(function () { ... })()`.
- **Speicher:** Jede Liste liegt im `localStorage` des Geräts. Ist man angemeldet, wird zusätzlich nach Firestore gespiegelt (`users/{uid}/...`). Firestore hat Offline-Persistenz (IndexedDB) und überträgt Änderungen, sobald wieder Netz da ist.
- **Abgleich-Logik:** `pushCollection()` / `pushMeta()` vergleichen jedes Dokument mit dem zuletzt bekannten Cloud-Stand (sortiertes JSON) und schreiben nur Änderungen. `onSnapshot`-Listener übernehmen Änderungen vom anderen Gerät (`applyRemote`). Beim **ersten Anmelden pro Gerät** (`firstLink`) werden Gerät und Cloud zusammengeführt (bei gleicher ID gewinnt der neuere Stand `geaendert`/`erstellt`). Danach ist die Cloud maßgeblich.
- **Service Worker:** Erst Netz, ohne Netz aus dem Speicher. **Bei jedem Update die Versionsnummer in `sw.js` erhöhen** (aktuell `const CACHE = "gratiswaren-v9";` → nächstes Mal `v10` usw.), sonst kommt die neue Version nicht zuverlässig an.
- **Design:** Flaschengrün als Akzentfarbe, Preisetikett-Gelb, Systemschriften (Avenir Next Condensed für Überschriften), hell und dunkel.

## 4. Datenmodell

### localStorage-Schlüssel → Firestore-Pfad
| localStorage | Firestore | Inhalt |
|---|---|---|
| `gratiswaren-lokal` | `users/{uid}/buchungen/{id}` | Gratiswaren-Einträge |
| `gratiswaren-belegungen` | `users/{uid}/belegungen/{id}` | Belegungen im Aktionskalender |
| `gratiswaren-flaechen` | `users/{uid}/meta/flaechen` `{list}` | Aktionsflächen |
| `gratiswaren-artikel` | `users/{uid}/meta/artikel` `{list}` | gelernte Artikel mit Preisen |
| `gratiswaren-kontakte` | `users/{uid}/meta/kontakte` `{list}` | Telefonbuch |
| `gratiswaren-einstellungen` | `users/{uid}/meta/einstellungen` | Einstellungen |
| `gratiswaren-sync-uid` | – | merkt sich, ob dieses Gerät schon zusammengeführt wurde |
| `gratiswaren-ansicht` | – | zuletzt geöffneter Reiter |

### Felder
- **Buchung (Gratisware):** `id, lieferant, kontakt, datum (abgemacht am), bestellartikel, menge, einheit (Display/Kiste/…), ve (Einheiten je Display), veName (Sixpack/…), preis (je Einheit bzw. je VE), aktion (Kassenregal | Standard | Eigene), quote (nur bei Eigene, in %), gratisartikel, gEinheit, gPreis, notiz, lieferungen[], belegungId?, beispiel?, erstellt, geaendert`
- **Lieferung:** `id, datum, menge (in Gratis-Einheiten), lieferschein, verbucht (bool), verbuchtAm`
- **Belegung:** `id, flaecheId, start (Montag als JJJJ-MM-TT), wochen, lieferant, kontakt, artikel, notiz, buchungId?, erstellt`
- **Fläche:** `id, name, typ (Kassenregal | Standard)`. Standard am Anfang: Kassenregal 1, Kassenregal 2, Stirnseite Getränke.
- **Artikel:** `id, name, einheit, preis, ve?, veName?`
- **Kontakt:** `id, name, telefon, email, notiz, lieferant (Haupt-Zuordnung, darf leer sein), weitere[] (weitere Lieferanten), hauptFuer[] (Lieferantennamen in Kleinschrift, für die er Hauptansprechpartner ist)`
- **Einstellungen:** `wochen (Nachhaken nach … Wochen, Standard 3), absender (Standard „Edeka Wehrmann · Getränke“), land (Bundesland-Kürzel oder leer), anlaesse (bool)`

## 5. Rechenlogik

- **Quote:** Kassenregal 20 %, Standard 10 %, sonst eigene Quote.
- **Bestellwert:** Display: Menge × Einheiten je Display × Preis je Einheit; sonst Menge × Preis.
- **Gratiswert:** Bestellwert × Quote, auf Cent gerundet.
- **Anspruch:** abgerundet (Gratiswert ÷ Preis der Gratis-Einheit) in ganzen Einheiten. **Restwert** = Gratiswert − Anspruch × Preis.
- **Verbucht:** Summe der als verbucht markierten Lieferungen × Preis. **Noch offen** = Gratiswert − verbucht (der Restwert bleibt offen).
- **Beispiel:** 6 Displays × 60 Sixpacks × 6,49 € = 2.336,40 € Bestellwert → 20 % = 467,28 € → bei 17,99 € je Kiste 25 Kisten, 17,53 € Restwert.
- **Status:** Angaben fehlen / Zu verbuchen / Teilweise geliefert / Offen / Verbucht / **Überfällig** (Ende der Belegung bzw. Vereinbarungsdatum + Nachhak-Frist überschritten und noch nicht alles geliefert).

## 6. Funktionen (Stand jetzt)

**Kopfbereich:** Status-Anzeige Abgleich („Abgeglichen“, „Offline · n Änderungen warten“, „Nicht angemeldet“), Einstellungen, „+ Neu“-Knopf passend zum Reiter.

**Reiter Aktionskalender**
- Flächen als Zeilen, 12 Kalenderwochen als Spalten, Blättern um 4 Wochen, „Diese Woche“.
- „Springe zu“: KW + Jahr, oder Feiertag/Anlass aus einer Liste.
- Freies Feld antippen → Belegung anlegen (Fläche, Start-KW, Dauer, Lieferant, Ansprechpartner, Artikel). Doppelbelegung wird verhindert.
- „Speichern & Gratisware anlegen“ übernimmt die Belegung vorausgefüllt in die Gratiswaren-Kontrolle (Quote nach Flächen-Typ).
- Farben: schraffiert = noch keine Gratisware, blau = offen, gelb = zu verbuchen, grün = verbucht, rot = überfällig.
- Liste „Noch nicht übernommen“.
- **Feiertage & Anlässe** werden selbst berechnet (Gaußsche Osterformel): bundesweit, regional nach Bundesland, optional Anlässe (Karneval, Valentinstag, Muttertag, Oktoberfest, Heiligabend, Silvester). Anzeige in der Wochen-Kopfzeile, **Hinweis-Karten** für die nächsten 10 Wochen mit freien Flächen.
- Flächen verwalten (umbenennen, Typ, hinzufügen, entfernen, wenn nicht belegt).
- **Belegung kopieren:** Kopie landet in derselben KW im nächsten Jahr (änderbar). Hat die Vorlage eine Gratisware, übernimmt „Speichern & Gratisware anlegen“ deren Mengen und Preise.

**Reiter Gratisware**
- Drei Kennzahlen: Gratiswert gesamt, davon bereits verbucht, noch offen. Balken: verbucht / geliefert, nicht verbucht / offen.
- Roter Hinweis bei überfälligen Einträgen, Filter (Alle, Nachhaken, Offen, Zu verbuchen, Verbucht, Angaben fehlen), Suche.
- Formular mit Live-Rechnung („Kassenbon“), Artikel-Vorschläge mit automatischem Preis, Ansprechpartner-Vorschläge.
- Detail: Rechnung, Lieferungen eintragen (Menge vorgeschlagen), einzeln oder alle verbuchen, bearbeiten, löschen.
- **Gratisware kopieren:** Formular mit allen Angaben der Vorlage, ohne Lieferungen und Flächen-Verknüpfung, Datum = heute.

**Reiter Lieferanten**
- Je Lieferant: Gratiswert, verbucht, offen, Restwerte, Anzahl überfällig, nächste Fläche.
- Lieferanten-Fenster: Ansprechpartner (aus dem Telefonbuch hinzufügen, neu anlegen, als Haupt festlegen, entfernen), Summen, Hinweis wenn Restwerte zusammen eine ganze Kiste ergeben, alle Aktionen.
- **„Aufstellung teilen“:** fertiger Text mit Anrede („Hallo Herr Meier, …“) und offenen Mengen über das Teilen-Menü (WhatsApp, Mail).
- „Artikel & Preise“ verwalten.

**Reiter Telefonbuch**
- Alle Kontakte nach Lieferant gruppiert, „Ohne Zuordnung“ am Ende, Suche, Telefon/E-Mail antippbar, bearbeiten, löschen mit Rückfrage. Ein Kontakt kann für mehrere Lieferanten zuständig sein.

**Einstellungen**
- Nachhak-Frist, Absender für Aufstellungen, Bundesland, Anlässe an/aus, Artikel verwalten.
- Abgleich: anmelden, Konto anlegen, Passwort vergessen, abmelden.
- Sicherung als JSON-Datei speichern/laden (enthält alles), CSV-Export für OpenOffice Calc (Semikolon, Dezimalkomma).

## 7. So kommt ein Update aufs iPad

1. Geänderte Dateien (meist `index.html` und `sw.js`) bei GitHub hochladen: Repository `gratisware` → **Add file** → **Upload files** → **Commit changes**. Gleichnamige Dateien werden ersetzt.
2. 1–2 Minuten warten.
3. App mit Internet öffnen. Wenn nötig, App ganz schließen und neu öffnen.
4. Tipp: HTML-Dateien lassen sich am iPad schlecht herunterladen, deshalb besser **als ZIP** schicken lassen und in der Dateien-App entpacken.

## 8. Offene Punkte

- **Nicht von Claude getestet:** die Verbindung zum echten Firebase-Projekt (in der Arbeitsumgebung war das Netzwerk gesperrt). Getestet ist alles mit einer nachgebauten Datenbank. Nach dem Update prüfen, ob oben „Abgeglichen“ steht.
- Empfohlen, falls noch nicht gemacht: In Firebase **Authentication → Einstellungen → Nutzeraktionen** die Registrierung neuer Konten ausschalten, sobald iPad und iPhone angemeldet sind.
- In Firebase ist die Domain `dant82.github.io` als autorisierte Domain eingetragen (bzw. sollte es sein).
- Es gibt außerdem eine ältere **Online-Version auf claude.ai** (Artifact, ohne Kalender, Abgleich usw.). Sie ist nicht mehr aktuell und wird nicht weitergepflegt.

## 9. Ideen aus dem Brainstorming (noch nichts davon gebaut)

**Empfohlene Reihenfolge**
1. **Papierkorb** (Gelöschtes 30 Tage zurückholen; wichtig wegen Abgleich zwischen Geräten) und **Versionsanzeige** in der App („Version vom …“).
2. **Startseite „Diese Woche“:** was steht wo, was wird umgebaut, was ist zu verbuchen, was ist überfällig, welcher Feiertag kommt.
3. **Fotos als Nachweis** (Aufbau, Lieferschein) an der Aktion.
4. **Jahresauswertung pro Lieferant** für Jahresgespräche (Gratiswert, Flächenauslastung, Vorjahresvergleich).

**Weitere Ideen**
- Pfand der Gratisware mitrechnen (z. B. 3,10 € je Kiste).
- Vergütung in Geld statt Ware erfassen (Werbekostenzuschuss, Platzierungsgeld).
- Abverkauf nach der Aktion festhalten (verkauft / Rest).
- Warteliste für vergebene Flächen.
- Edeka-Handzettel / Werbewochen im Kalender.
- Umbau-Erinnerung montags, Aufgaben für Mitarbeiter.
- Textvorlagen: Reservierungsbestätigung, freundliche Mahnung.
- Belegungen in den iPhone-Kalender exportieren.
- Push-Benachrichtigungen (braucht kostenpflichtigen Firebase-Tarif + Hintergrunddienst, großer Aufwand).
- Weitere Nutzer (Stellvertreter, Marktleiter), ggf. nur lesend.
- Monatsplan als PDF zum Aushängen.

**Offene Fragen an Dan**
- Wird das Pfand der Gratisware extra gebucht?
- Soll später noch jemand anderes mit der App arbeiten?
- Gibt es von Lieferanten auch Geld statt Ware für die Flächen?

## 10. Hinweise für Claude im neuen Chat

- Die App ist **eine** Datei `index.html` mit Vanilla-JavaScript. Bitte im bestehenden Stil weiterbauen (deutsche Texte und Kommentare, Hilfsfunktionen wie `$()`, `esc()`, `fmt()`, `num()`, `r2()`, `dDE()`, `uid()`, `toast()`), kein Framework einführen.
- Jede neue gespeicherte Liste braucht: localStorage-Schlüssel, `pushMeta`/`pushCollection`-Aufruf, Übernahme in `applyRemote`/Listener, Zusammenführen in `firstLink`, Aufnahme in Sicherung und Wiederherstellung.
- Beim Laden zuerst **alle** Listen aus dem Speicher lesen, erst danach etwas speichern (sonst werden noch nicht geladene Listen überschrieben – dieser Fehler wurde schon einmal behoben).
- Dialoge: Knöpfe im Fußbereich nicht bei jeder Eingabe neu aufbauen (sonst geht ein Tipp beim Verlassen eines Felds verloren).
- Nach jeder Änderung `CACHE` in `sw.js` hochzählen und beide Dateien liefern, am besten als ZIP.
