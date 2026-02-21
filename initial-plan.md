Absolut. Hier ist eine vollständige Zusammenfassung unserer Diskussion über den Bau eines privaten Regelwerk-Editors mit ProseMirror als Kern – ohne die zuletzt besprochenen Offline‑ und Authelia‑Details. Du kannst diesen Text als Grundlage für deine Planung und Umsetzung verwenden.

---

📘 Zusammenfassung: Regelwerk‑Editor mit ProseMirror

1. Zielsetzung

Eine private Webseite für ein Pen‑&‑Paper‑Regelwerk, auf der berechtigte Nutzer (mehrere Editoren) Kapitel und Unterkapitel erstellen und bearbeiten können. Die Inhalte sollen im Browser mit einem WYSIWYG‑Editor gepflegt werden, der sauber strukturierte Daten liefert. Zusätzlich benötigt werden interne Links zwischen Kapiteln und Tooltips für Verweise.

2. Warum ProseMirror?

· Open Source (MIT‑Lizenz) – kostenlos, auch für kommerzielle Projekte.
· Strukturiertes Dokumentmodell – liefert JSON statt wildem HTML, perfekt für konsistente Darstellung und Weiterverarbeitung.
· Volle Kontrolle – man definiert genau, welche Elemente erlaubt sind (Überschriften, Absätze, Listen, spezielle Kästen).
· Erweiterbar – eigene Nodes (z.B. für interne Links) lassen sich problemlos integrieren.
· Kollaborationstauglich – durch Bibliotheken wie y-prosemirror wird Echtzeit‑Mehrbenutzer‑Bearbeitung möglich.

3. Grundlegende Architektur

Frontend (Browser)

· Framework nach Wahl: Vue.js oder React – beides funktioniert gut mit ProseMirror.
· Der Editor wird mit prosemirror-view, prosemirror-state und einem eigenen Schema aufgebaut.
· Für interne Links wird ein eigener Node-Typ (internalLink) definiert, der eine chapterId speichert.
· Bei Online‑Verbindung wird die Kollaborations‑Bibliothek aktiviert (siehe Punkt 5).

Backend (API)

· Node.js mit Express (oder Python FastAPI) – leichtgewichtig und einfach.
· Stellt Endpunkte bereit:
  · GET /api/chapters – liefert die gesamte Kapitelstruktur (Baum).
  · GET /api/chapters/:id – liefert ein einzelnes Kapitel inkl. Inhalt (JSON).
  · POST /api/chapters – neues Kapitel anlegen.
  · PUT /api/chapters/:id – Kapitel aktualisieren.
  · DELETE /api/chapters/:id – Kapitel löschen.
  · POST /api/login – Authentifizierung (JWT).
· Optional: WebSocket‑Server für Kollaboration (siehe Punkt 5).

Datenbank

· SQLite (für den Start) oder PostgreSQL (später).
· Tabelle chapters:
  ```sql
  CREATE TABLE chapters (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      title TEXT NOT NULL,
      parent_id INTEGER,           -- NULL für Hauptkapitel, sonst ID des Elternkapitels
      content JSON,                 -- ProseMirror‑Dokumentenbaum
      position INTEGER,              -- für Sortierung innerhalb der Geschwister
      created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
      updated_at DATETIME,
      FOREIGN KEY (parent_id) REFERENCES chapters(id) ON DELETE CASCADE
  );
  ```
· Tabelle users für mehrere Editoren:
  ```sql
  CREATE TABLE users (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      username TEXT UNIQUE NOT NULL,
      display_name TEXT,             -- Anzeigename für Cursor
      color TEXT,                     -- Wunschfarbe für Cursor
      password_hash TEXT NOT NULL,
      role TEXT DEFAULT 'editor'      -- z.B. 'admin', 'editor'
  );
  ```

4. Mehrbenutzer‑Authentifizierung

· JWT (JSON Web Tokens) – nach Login erhält der Client ein Token, das bei allen schreibenden API‑Aufrufen mitgesendet wird.
· Passwörter werden mit bcrypt oder argon2 gehasht.
· Jeder authentifizierte Nutzer darf Kapitel bearbeiten (keine feingranulare Rechteverwaltung nötig, da privat).

5. Gleichzeitiges Bearbeiten (Echtzeit‑Kollaboration)

Warum Yjs?

· Basiert auf CRDTs, löst Konflikte automatisch auf.
· Ermöglicht Offline‑Änderungen (später nutzbar) und getrennte Undo‑Verläufe pro Nutzer.
· Gut integriert mit ProseMirror über y-prosemirror.

Komponenten:

· Yjs‑Dokument (ydoc) im Frontend, das den aktuellen Editor‑State hält.
· WebSocket‑Server zur Verteilung der Änderungen: empfehlenswert ist Hocuspocus (speziell für Yjs entwickelt) oder der einfache y-websocket‑Server.
· y-prosemirror‑Plugins:
  · ySyncPlugin – synchronisiert ProseMirror mit Yjs.
  · yCursorPlugin – zeigt Cursor und Auswahlen anderer Nutzer an.
  · yUndoPlugin – ermöglicht nutzerindividuelles Rückgängigmachen.
· Awareness – jeder Nutzer teilt seinen Namen, seine Farbe und Cursorposition mit.

Integration im Frontend:

```javascript
import * as Y from 'yjs'
import { HocuspocusProvider } from '@hocuspocus/provider'
import { ySyncPlugin, yCursorPlugin, yUndoPlugin } from 'y-prosemirror'

const ydoc = new Y.Doc()
const provider = new HocuspocusProvider({
  url: 'ws://localhost:1234',
  name: 'dokument-id',   // z.B. chapter-123
  document: ydoc,
  token: user.jwt
})

const type = ydoc.get('prosemirror', Y.XmlFragment)

const view = new EditorView(editorElement, {
  state: EditorState.create({
    schema: mySchema,
    plugins: [
      ySyncPlugin(type),
      yCursorPlugin(provider.awareness),
      yUndoPlugin(),
      // eigene Plugins
    ]
  })
})

// Awareness mit Nutzerdaten füllen
provider.awareness.setLocalStateField('user', {
  name: user.display_name,
  color: user.color,
  id: user.id
})
```

Cursor‑Styling per CSS:

```css
.ProseMirror-yjs-cursor {
  border-left: 2px solid;
  border-color: inherit;
  position: relative;
}
.ProseMirror-yjs-cursor > div {
  position: absolute;
  top: -1.4em;
  left: -2px;
  background-color: inherit;
  color: white;
  padding: 2px 4px;
  border-radius: 3px;
  font-size: 12px;
  white-space: nowrap;
}
```

Persistenz auf dem Server:

· Hocuspocus kann Dokumente automatisch in einer Datenbank speichern (z.B. über onStoreDocument‑Hook).
· Alternativ: Regelmäßiges Speichern des Yjs‑Updates oder des gesamten JSON in der chapters‑Tabelle.

6. Interne Links und Tooltips

Schema‑Erweiterung:
Eigener Node für interne Verweise:

```javascript
import { Node } from 'prosemirror-model'

const internalLinkNode = new Node({
  name: 'internalLink',
  attrs: { chapterId: { default: null } },
  inline: true,
  group: 'inline',
  parseDOM: [{
    tag: 'a[data-chapter-id]',
    getAttrs: dom => ({ chapterId: dom.getAttribute('data-chapter-id') })
  }],
  toDOM: node => {
    const { chapterId } = node.attrs
    return ['a', { 'data-chapter-id': chapterId, href: `#/chapter/${chapterId}` }, 0]
  }
})
```

Link‑Dialog im Editor:

· Button in der Menüleiste öffnet ein Modal mit einer Baumansicht aller Kapitel.
· Nach Auswahl wird der markierte Text durch den internalLink‑Node ersetzt (oder der Node umschließt ihn).

Tooltips:

· Einfache Lösung: title‑Attribut mit dem Kapitelnamen.
· Erweiterte Lösung: Beim Hovern per API den Inhalt des Zielkapitels laden und in einem Popup anzeigen.

Rendering in der Leseansicht:

· Das vom Server gelieferte JSON wird mit einem einfachen Renderer (z.B. prosemirror-render) in HTML umgewandelt.
· Für den internalLink‑Node wird ein <a>‑Tag mit href auf die entsprechende Kapitelseite erzeugt – so navigiert der Leser direkt zum verlinkten Kapitel.

7. Entwicklungsfahrplan

1. Backend‑Grundgerüst mit Express, SQLite und JWT‑Authentifizierung.
2. Datenbank‑Tabellen (users, chapters) anlegen.
3. Login‑Maske und einfache Kapitelliste (Baum) im Frontend.
4. ProseMirror‑Editor einbinden (zunächst ohne Kollaboration) mit einfachem Schema.
5. Interne Links implementieren: eigenen Node definieren, Dialog zur Kapitelauswahl bauen.
6. Tooltips hinzufügen (erstes title‑Attribut).
7. Kollaboration mit Yjs und Hocuspocus integrieren: WebSocket‑Server aufsetzen, Frontend‑Anbindung, Cursor‑Farben.
8. Leseansicht bauen, die JSON rendert und Links als Navigation umsetzt.
9. Feinschliff: Drag‑Drop für Kapitelreihenfolge, Benutzerverwaltung, UI‑Optimierungen.

8. Nützliche Ressourcen

· ProseMirror‑Dokumentation: https://prosemirror.net/docs/guide/
· Yjs: https://yjs.dev/
· y‑prosemirror: https://github.com/yjs/y-prosemirror
· Hocuspocus: https://hocuspocus.dev/
· Tiptap (Wrapper um ProseMirror, erleichtert Einstieg): https://tiptap.dev/