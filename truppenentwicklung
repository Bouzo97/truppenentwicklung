// ==UserScript==
// @name         Truppen-Verlauf (Die Stämme)
// @namespace    die-staemme-truppen-verlauf
// @version      1.0.0
// @description  Liest die Stammes-Truppenübersicht (Mitglieder → Truppen) einmal täglich ein, speichert den Verlauf und zeigt die Entwicklung; Export für Tabellen, Discord und Forum
// @match        https://*.die-staemme.de/game.php*
// @grant        none
// ==/UserScript==

/*
 * Truppen-Verlauf für Die Stämme
 *
 * Nutzung:
 *  - Schnellleiste: javascript:$.getScript('https://DEINE-URL/truppen-verlauf.js');void(0);
 *    Beim Öffnen wird die Truppenübersicht automatisch eingelesen und für heute gespeichert.
 *  - Tampermonkey: Datei als Userscript einfügen und USERSCRIPT_MODE auf true setzen
 *    (dann wird einmal pro Tag automatisch im Hintergrund gespeichert, ein Button öffnet das Fenster).
 *
 * Das Skript ruft nur die Seite "Stamm → Mitglieder → Truppen" auf (reines Lesen) und führt keine Aktionen aus.
 * Die gespeicherten Werte liegen nur im eigenen Browser (localStorage). Wer dort nichts freigegeben hat, steht mit "?" drin.
 */
(function () {
  'use strict';

  // ---------------------------------------------------------------- Einstellungen
  const USERSCRIPT_MODE = false; // true = Button einblenden + täglich automatisch speichern

  const CFG = {
    histTage: 400,   // so viele Tage Verlauf bleiben gespeichert
  };

  // Einheiten: Kurzname, Bauernhofplätze (BH), Gruppe für die Off-/Deff-Summe
  const UNITS = {
    spear: { k: 'Speer', n: 'Speerträger', bh: 1, g: 'def' },
    sword: { k: 'Schwert', n: 'Schwertkämpfer', bh: 1, g: 'def' },
    axe: { k: 'Axt', n: 'Axtkämpfer', bh: 1, g: 'off' },
    archer: { k: 'Bogen', n: 'Bogenschütze', bh: 1, g: 'def' },
    spy: { k: 'Späher', n: 'Späher', bh: 2, g: '' },
    light: { k: 'LKav', n: 'Leichte Kavallerie', bh: 4, g: 'off' },
    marcher: { k: 'BBogen', n: 'Berittener Bogenschütze', bh: 5, g: 'off' },
    heavy: { k: 'SKav', n: 'Schwere Kavallerie', bh: 6, g: 'def' },
    ram: { k: 'Ramme', n: 'Rammbock', bh: 5, g: 'off' },
    catapult: { k: 'Kata', n: 'Katapult', bh: 8, g: 'off' },
    knight: { k: 'Pala', n: 'Paladin', bh: 10, g: '' },
    snob: { k: 'AG', n: 'Adelsgeschlecht', bh: 100, g: '' },
  };
  const uInfo = (u) => UNITS[u] || { k: u, n: u, bh: 0, g: '' };
  const GREEN = '#2e7d32', RED = '#c62828';

  // ---------------------------------------------------------------- Grundlagen
  const gd = window.game_data;
  if (window.top !== window.self) return;
  if (!gd) { alert('Bitte im Spiel ausführen (game.php).'); return; }
  if (window.__tvVerlauf) { if (!USERSCRIPT_MODE) window.__tvVerlauf.open(); return; }

  const nf = new Intl.NumberFormat('de-DE');
  const fmt = (n) => (n == null ? '?' : nf.format(Math.round(n)));
  const esc = (s) => String(s).replace(/[&<>"']/g, (c) => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]));
  const q = (sel) => document.querySelector('#tv-box ' + sel);
  const HKEY = 'tv_hist_' + gd.world;

  const tagKey = (d) => `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-${String(d.getDate()).padStart(2, '0')}`;
  const anz = (k) => k.split('-').reverse().join('.');
  const dn = (k) => { const [y, m, d] = k.split('-').map(Number); return Date.UTC(y, m - 1, d) / 864e5; };

  // ---------------------------------------------------------------- Speicher
  // { "2026-11-20": { t: "20.11.26 10:34", u: ["spear", ...], p: { spielerId: [name, punkte, [werte je Einheit, null = nicht freigegeben]] } } }
  function ladeHist() {
    try { const h = JSON.parse(localStorage.getItem(HKEY)); return h && typeof h === 'object' ? h : {}; } catch (x) { return {}; }
  }
  function speichereHist(h) {
    const keys = Object.keys(h).sort();
    while (keys.length > CFG.histTage) delete h[keys.shift()];
    try { localStorage.setItem(HKEY, JSON.stringify(h)); return true; } catch (x) { return false; }
  }

  // ---------------------------------------------------------------- Einlesen
  async function holeText(url) {
    const r = await fetch(url, { credentials: 'same-origin' });
    if (!r.ok) throw new Error('HTTP ' + r.status);
    return r.text();
  }

  // Liest die Tabelle "Truppenübersicht" des Stammes: Spalten werden über die Einheiten-Symbole (unit_xxx) erkannt.
  function parseTruppen(html) {
    const doc = new DOMParser().parseFromString(html, 'text/html');
    const stand = ((doc.body.textContent || '').match(/Zuletzt aktualisiert:\s*(\d{2}\.\d{2}\.\d{2,4}\s+\d{1,2}:\d{2})/) || [])[1] || '';
    let tab = null;
    doc.querySelectorAll('table').forEach((t) => {
      if (tab || !t.rows.length) return;
      const cells = [...t.rows[0].cells];
      if (cells.length <= 25 && cells.filter((c) => c.querySelector('img[src*="unit_"]')).length >= 3) tab = t;
    });
    if (!tab) return { stand, u: [], rows: [] };
    const spalten = []; // Spaltenindex -> Einheit
    [...tab.rows[0].cells].forEach((c, i) => {
      const im = c.querySelector('img[src*="unit_"]');
      const m = im && (im.getAttribute('src') || '').match(/unit_([a-z]+)\./);
      if (m) spalten.push({ i, u: m[1] });
    });
    const zahl = (s) => { const z = String(s || '').replace(/[\s.]/g, ''); return /^\d+$/.test(z) ? parseInt(z, 10) : null; };
    const rows = [];
    [...tab.rows].slice(1).forEach((tr) => {
      const a = tr.querySelector('a');
      if (!a || /^zusammenfassung/i.test(tr.cells[0].textContent.trim())) return;
      const mid = (a.getAttribute('href') || '').match(/[?&]id=(\d+)/);
      const name = a.textContent.trim();
      rows.push({
        id: mid ? mid[1] : 'n:' + name, name,
        punkte: zahl(tr.cells[1] && tr.cells[1].textContent) || 0,
        vals: spalten.map((s) => zahl(tr.cells[s.i] && tr.cells[s.i].textContent)),
      });
    });
    return { stand, u: spalten.map((s) => s.u), rows };
  }

  // Liest die Übersicht und speichert sie als Messung für heute (mehrfach am Tag: letzte Messung gilt, bekannte Werte bleiben erhalten)
  async function einlesenUndSpeichern() {
    const url = `${location.origin}/game.php?${gd.village ? 'village=' + gd.village.id + '&' : ''}screen=ally&mode=members_troops`;
    const p = parseTruppen(await holeText(url));
    if (!p.rows.length) throw new Error('Keine Truppentabelle gefunden (Stamm → Mitglieder → Truppen erreichbar?).');
    const hist = ladeHist();
    const key = tagKey(new Date());
    const alt = hist[key];
    const snap = { t: p.stand, u: p.u, p: {} };
    p.rows.forEach((r) => {
      let vals = r.vals;
      const o = alt && alt.p[r.id];
      if (o && alt.u.join() === p.u.join()) vals = vals.map((v, i) => (v == null && o[2][i] != null ? o[2][i] : v));
      snap.p[r.id] = [r.name, r.punkte, vals];
    });
    hist[key] = snap;
    const ok = speichereHist(hist);
    return { key, stand: p.stand, n: p.rows.length, frei: p.rows.filter((r) => r.vals.some((v) => v != null)).length, ok, u: p.u };
  }

  // ---------------------------------------------------------------- Auswertung
  // Werte je Spieler: Punkte, jede Einheit, Off-/Deff-/Gesamt-Bauernhofplätze (null, wenn nicht freigegeben)
  function snapWerte(snap) {
    const out = new Map();
    Object.entries(snap.p).forEach(([id, [name, punkte, vals]]) => {
      const m = { punkte };
      snap.u.forEach((u, i) => { m[u] = vals[i]; });
      const grp = (f) => {
        let s = 0;
        for (let i = 0; i < snap.u.length; i++) {
          const U = uInfo(snap.u[i]);
          if (!f(U)) continue;
          if (vals[i] == null) return null;
          s += vals[i] * U.bh;
        }
        return s;
      };
      m.off = grp((U) => U.g === 'off'); m.def = grp((U) => U.g === 'def'); m.ges = grp(() => true);
      out.set(id, { name, m });
    });
    return out;
  }

  function findBase(keys, latest, modus) {
    const c = keys.filter((k) => k < latest);
    if (!c.length) return null;
    if (modus === 'alt') return c[0];
    const ziel = dn(latest) - Number(modus);
    return c.reduce((b, k) => (Math.abs(dn(k) - ziel) < Math.abs(dn(b) - ziel) ? k : b), c[0]);
  }

  function metriken(hist) {
    const vorh = new Set();
    Object.values(hist).forEach((s) => s.u.forEach((u) => vorh.add(u)));
    const einh = Object.keys(UNITS).filter((u) => vorh.has(u)).concat([...vorh].filter((u) => !UNITS[u]));
    return [{ k: 'punkte', t: 'Punkte' }]
      .concat(einh.map((u) => ({ k: u, t: uInfo(u).k, title: uInfo(u).n })))
      .concat([{ k: 'off', t: 'Off-BH', title: 'Bauernhofplätze aller Off-Einheiten (Axt, LKav, Ramme, Kata, ber. Bogen)' },
        { k: 'def', t: 'Deff-BH', title: 'Bauernhofplätze aller Deff-Einheiten (Speer, Schwert, Bogen, SKav)' },
        { k: 'ges', t: 'Gesamt-BH', title: 'Bauernhofplätze aller Truppen (inkl. Späher, Paladin, AG)' }]);
  }

  let sort = { k: 'ges', dir: -1 };

  // Gemeinsames Modell für Anzeige, Kopieren, BB-Code und Bild: { titel, kopf:[{t,num,delta}], zeilen:[[Zelle]], summe, info }
  // Zelle: Text (Textspalte) oder { v, d } (Zahl und Veränderung)
  function modell() {
    const hist = ladeHist();
    const keys = Object.keys(hist).sort();
    if (!keys.length) return null;
    const latest = keys[keys.length - 1];
    const ansicht = q('#tv-ansicht').value;
    const modus = q('#tv-vergleich').value;
    const mets = metriken(hist);
    const kopfZahl = mets.map((m) => ({ t: m.t, num: 1, delta: 1, k: m.k, title: m.title }));
    const welt = `Welt ${gd.world}`;

    if (ansicht === 'uebersicht') {
      const cur = snapWerte(hist[latest]);
      const bk = findBase(keys, latest, modus);
      const bas = bk ? snapWerte(hist[bk]) : null;
      let zeilen = [...cur.entries()].map(([id, e]) => {
        const b = bas && bas.get(id);
        const cells = {};
        mets.forEach((m) => {
          const v = e.m[m.k] == null ? null : e.m[m.k];
          const bv = b && b.m[m.k] != null ? b.m[m.k] : null;
          cells[m.k] = { v, d: v != null && bv != null ? v - bv : null };
        });
        return { name: e.name, cells };
      });
      const sk = sort.k, nachD = q('#tv-sortd').checked;
      zeilen.sort((a, b) => {
        let x, y;
        if (sk === 'name') return a.name.localeCompare(b.name, 'de') * sort.dir;
        x = a.cells[sk] ? (nachD ? a.cells[sk].d : a.cells[sk].v) : null;
        y = b.cells[sk] ? (nachD ? b.cells[sk].d : b.cells[sk].v) : null;
        if (x == null && y == null) return 0;
        if (x == null) return 1;
        if (y == null) return -1;
        return (x - y) * sort.dir;
      });
      const summe = ['Summe'];
      mets.forEach((m) => {
        let v = 0, d = 0, hatV = false, hatD = false;
        zeilen.forEach((z) => { const c = z.cells[m.k]; if (c.v != null) { v += c.v; hatV = true; } if (c.d != null) { d += c.d; hatD = true; } });
        summe.push({ v: hatV ? v : null, d: hatD ? d : null });
      });
      const frei = zeilen.filter((z) => z.cells.ges.v != null).length;
      return {
        titel: `Truppen-Übersicht ${welt} – Stand ${anz(latest)}${hist[latest].t ? ' (Spiel: ' + hist[latest].t + ')' : ''}`,
        kopf: [{ t: 'Spieler' }].concat(kopfZahl),
        zeilen: zeilen.map((z) => [z.name].concat(mets.map((m) => z.cells[m.k]))),
        summe, ansichtArt: 'uebersicht',
        info: `${zeilen.length} Mitglieder, bei ${frei} sind Truppen freigegeben („?“ = nicht freigegeben). ` +
          (bk ? `Veränderung gegenüber ${anz(bk)} (vor ${dn(latest) - dn(bk)} Tag${dn(latest) - dn(bk) === 1 ? '' : 'en'}).` : 'Noch kein Vergleichswert: Der heutige Stand ist gespeichert, Veränderungen erscheinen ab dem nächsten Tag.'),
        sortKeys: ['name'].concat(mets.map((m) => m.k)),
      };
    }

    // Verlauf: ein Spieler oder der ganze Stamm, ein Eintrag pro gespeichertem Tag (neueste oben)
    const gesamt = ansicht === 'gesamt';
    const pid = gesamt ? null : ansicht.slice(2);
    const reihe = [];
    let name = 'Stamm gesamt';
    keys.forEach((k) => {
      const w = snapWerte(hist[k]);
      let m;
      if (gesamt) {
        m = {}; let n = 0;
        mets.forEach((mt) => { let s = 0, has = false; w.forEach((e) => { if (e.m[mt.k] != null) { s += e.m[mt.k]; has = true; } }); m[mt.k] = has ? s : null; });
        w.forEach((e) => { if (e.m.ges != null) n++; });
        m.n = n;
      } else {
        const e = w.get(pid);
        if (!e) return;
        name = e.name; m = e.m;
      }
      reihe.push({ k, m });
    });
    const kopf = [{ t: 'Datum' }].concat(kopfZahl);
    if (gesamt) kopf.push({ t: 'Spieler mit Daten', num: 1, delta: 0 });
    const zeilen = reihe.map((r, i) => {
      const prev = i > 0 ? reihe[i - 1].m : null;
      const cells = mets.map((mt) => {
        const v = r.m[mt.k] == null ? null : r.m[mt.k];
        const pv = prev && prev[mt.k] != null ? prev[mt.k] : null;
        return { v, d: v != null && pv != null ? v - pv : null };
      });
      if (gesamt) cells.push({ v: r.m.n, d: null });
      return [anz(r.k)].concat(cells);
    }).reverse();
    return {
      titel: `Truppen-Verlauf ${name} – ${welt}`,
      kopf, zeilen, summe: null, ansichtArt: 'verlauf',
      info: `${reihe.length} gespeicherte Tage${gesamt ? ' (Summe aller Mitglieder mit freigegebenen Truppen)' : ''}. Veränderung jeweils gegenüber dem davor gespeicherten Tag.`,
    };
  }

  // ---------------------------------------------------------------- Darstellung der Zellen
  function zt(c, col) {
    if (!col.num) return { a: String(c), b: '', f: null };
    const a = c.v == null ? '?' : fmt(c.v);
    if (col.delta && c.d != null && Math.round(c.d) !== 0) return { a, b: ' (' + (c.d > 0 ? '+' : '−') + fmt(Math.abs(c.d)) + ')', f: c.d > 0 ? GREEN : RED };
    return { a, b: '', f: null };
  }

  function renderTabelle() {
    const m = modell();
    if (!m) { q('#tv-table').innerHTML = ''; q('#tv-info').textContent = 'Noch keine gespeicherten Daten.'; return; }
    q('#tv-info').textContent = m.info;
    const sortierbar = m.ansichtArt === 'uebersicht';
    let h = '<table class="tv-tab"><thead><tr>' + m.kopf.map((c, i) => {
      const key = sortierbar ? (i === 0 ? 'name' : c.k) : '';
      const pfeil = sortierbar && sort.k === key ? (sort.dir > 0 ? ' ▲' : ' ▼') : '';
      return `<th${key ? ` data-k="${key}" style="cursor:pointer"` : ''}${c.title ? ` title="${esc(c.title)}"` : ''}>${esc(c.t)}${pfeil}</th>`;
    }).join('') + '</tr></thead><tbody>';
    const zeile = (z, cls) => '<tr' + (cls ? ` class="${cls}"` : '') + '>' + z.map((c, i) => {
      const col = m.kopf[i], t = zt(c, col);
      const grau = col.num && c.v == null ? ' style="color:#999"' : '';
      return `<td${col.num ? ' class="n"' : ''}${grau}>${esc(t.a)}${t.b ? `<span style="color:${t.f}">${esc(t.b)}</span>` : ''}</td>`;
    }).join('') + '</tr>';
    m.zeilen.forEach((z) => { h += zeile(z); });
    if (m.summe) h += zeile(m.summe, 'tv-sum');
    q('#tv-table').innerHTML = h + '</tbody></table>';
    q('#tv-table').querySelectorAll('th[data-k]').forEach((th) => {
      th.onclick = () => { const k = th.dataset.k; sort = { k, dir: sort.k === k ? -sort.dir : (k === 'name' ? 1 : -1) }; renderTabelle(); };
    });
  }

  // ---------------------------------------------------------------- Export
  const zeilenMitSumme = (m) => m.zeilen.concat(m.summe ? [m.summe] : []);

  // Für Excel / Google Tabellen: Tabulator-getrennt, Zahlen ohne Tausenderpunkt, Veränderung in eigener Spalte
  function tsv() {
    const m = modell();
    const kopf = []; m.kopf.forEach((c) => { kopf.push(c.t); if (c.delta) kopf.push('Δ ' + c.t); });
    const lines = [kopf.join('\t')];
    zeilenMitSumme(m).forEach((z) => {
      const l = [];
      z.forEach((c, i) => {
        const col = m.kopf[i];
        if (!col.num) { l.push(String(c)); return; }
        l.push(c.v == null ? '' : Math.round(c.v));
        if (col.delta) l.push(c.d == null ? '' : Math.round(c.d));
      });
      lines.push(l.join('\t'));
    });
    return lines.join('\n');
  }

  function bbCode() {
    const m = modell();
    let t = `[b]${m.titel}[/b]\n[table]\n[**]${m.kopf.map((c) => c.t).join('[||]')}[/**]\n`;
    zeilenMitSumme(m).forEach((z, ri) => {
      const letzte = m.summe && ri === m.zeilen.length;
      t += '[*]' + z.map((c, i) => {
        const x = zt(c, m.kopf[i]);
        let s = x.a + (x.b ? `[color=${x.f}]${x.b}[/color]` : '');
        if (letzte || (i === 0 && m.ansichtArt === 'uebersicht')) s = `[b]${s}[/b]`;
        return s;
      }).join('[|]') + '\n';
    });
    return t + '[/table]';
  }

  // Alle gespeicherten Tage als lange Tabelle (ein Spieler pro Tag und Zeile) – zum Weiterarbeiten in Excel / Google Tabellen
  function csvAlle() {
    const hist = ladeHist();
    const keys = Object.keys(hist).sort();
    const mets = metriken(hist);
    const z = (v) => '"' + String(v == null ? '' : v).replace(/"/g, '""') + '"';
    const kopf = ['Datum', 'Stand_Spiel', 'Spieler_ID', 'Spieler'].concat(mets.map((m) => m.t));
    const lines = [kopf.map(z).join(';')];
    keys.forEach((k) => {
      const w = snapWerte(hist[k]);
      w.forEach((e, id) => {
        lines.push([anz(k), hist[k].t, id, e.name].concat(mets.map((m) => (e.m[m.k] == null ? '' : e.m[m.k]))).map(z).join(';'));
      });
    });
    return '﻿' + lines.join('\r\n');
  }

  // Tabelle als Bild (PNG) für Discord
  function bildCanvas() {
    const m = modell();
    const S = 2, H = 24, PAD = 8, FONT = '13px Arial, sans-serif', FB = 'bold 13px Arial, sans-serif';
    const mc = document.createElement('canvas').getContext('2d');
    const zeilen = zeilenMitSumme(m);
    const breiten = m.kopf.map((c, i) => {
      mc.font = FB;
      let w = mc.measureText(c.t).width;
      zeilen.forEach((z) => { const t = zt(z[i], c); mc.font = FONT; w = Math.max(w, mc.measureText(t.a + t.b).width); });
      return Math.ceil(w) + PAD * 2;
    });
    const breite = breiten.reduce((a, b) => a + b, 0);
    const kopfH = 32, fussH = 22;
    const W = Math.max(breite, 360), Ht = kopfH + H * (zeilen.length + 1) + fussH;
    const cv = document.createElement('canvas');
    cv.width = W * S; cv.height = Ht * S;
    const g = cv.getContext('2d');
    g.scale(S, S);
    g.fillStyle = '#f4e4bc'; g.fillRect(0, 0, W, Ht);
    g.textBaseline = 'middle'; g.textAlign = 'left'; g.fillStyle = '#000'; g.font = 'bold 15px Arial, sans-serif';
    g.fillText(m.titel, 10, kopfH / 2);
    g.fillStyle = '#c1a264'; g.fillRect(0, kopfH, W, H);
    const text = (i, x, y, t, fett, farbe) => {
      g.font = fett ? FB : FONT; g.fillStyle = farbe || '#000';
      if (m.kopf[i].num) { g.textAlign = 'right'; g.fillText(t, x + breiten[i] - PAD, y + H / 2); } else { g.textAlign = 'left'; g.fillText(t, x + PAD, y + H / 2); }
    };
    let x = 0;
    m.kopf.forEach((c, i) => { text(i, x, kopfH, c.t, true); x += breiten[i]; });
    zeilen.forEach((z, ri) => {
      const y = kopfH + H * (ri + 1);
      const summe = m.summe && ri === zeilen.length - 1;
      if (summe) { g.fillStyle = '#e6d19b'; g.fillRect(0, y, W, H); }
      g.fillStyle = '#d8c18a'; g.fillRect(0, y + H - 1, W, 1);
      x = 0;
      z.forEach((c, i) => {
        const col = m.kopf[i], t = zt(c, col);
        const grau = col.num && c.v == null ? '#999' : '#000';
        if (t.b) {
          g.font = FONT; const wb = g.measureText(t.b).width;
          g.textAlign = 'right'; g.font = FONT; g.fillStyle = t.f; g.fillText(t.b, x + breiten[i] - PAD, y + H / 2);
          g.fillStyle = '#000'; g.fillText(t.a, x + breiten[i] - PAD - wb, y + H / 2);
        } else text(i, x, y, t.a, summe, grau);
        x += breiten[i];
      });
    });
    g.fillStyle = '#555'; g.font = '11px Arial, sans-serif'; g.textAlign = 'left'; g.fillStyle = '#555';
    g.fillText(m.info, 10, Ht - fussH / 2);
    return cv;
  }
  const bildBlob = () => new Promise((res) => bildCanvas().toBlob(res, 'image/png'));

  function dateiLaden(name, blobOrText, typ) {
    const b = blobOrText instanceof Blob ? blobOrText : new Blob([blobOrText], { type: typ });
    const url = URL.createObjectURL(b);
    const a = document.createElement('a');
    a.href = url; a.download = name;
    document.body.appendChild(a); a.click(); a.remove();
    setTimeout(() => URL.revokeObjectURL(url), 2000);
  }
  async function kopieren(text, meldung) {
    try {
      if (navigator.clipboard && navigator.clipboard.writeText) await navigator.clipboard.writeText(text);
      else { const ta = q('#tv-bb'); q('#tv-bbwrap').style.display = 'block'; ta.value = text; ta.select(); document.execCommand('copy'); }
      meldungSetzen(meldung);
    } catch (e) { meldungSetzen('Kopieren nicht möglich – bitte CSV nutzen.', true); }
  }
  function meldungSetzen(t, fehler) { const el = q('#tv-msg'); el.textContent = t; el.style.color = fehler ? RED : GREEN; }
  function setStatus(t, fehler) { const el = q('#tv-status'); el.textContent = t; el.style.color = fehler ? RED : '#000'; }

  // ---------------------------------------------------------------- Fenster
  function fuelleAnsicht() {
    const hist = ladeHist();
    const sel = q('#tv-ansicht');
    const alt = sel.value;
    const namen = new Map();
    Object.keys(hist).sort().forEach((k) => Object.entries(hist[k].p).forEach(([id, e]) => namen.set(id, e[0])));
    sel.innerHTML = '<option value="uebersicht">Übersicht aller Spieler (heute)</option><option value="gesamt">Verlauf: Stamm gesamt</option>' +
      '<optgroup label="Verlauf eines Spielers">' + [...namen.entries()].sort((a, b) => a[1].localeCompare(b[1], 'de')).map(([id, n]) => `<option value="p:${esc(id)}">${esc(n)}</option>`).join('') + '</optgroup>';
    if ([...sel.options].some((o) => o.value === alt)) sel.value = alt;
  }

  async function lesen() {
    q('#tv-lesen').disabled = true;
    setStatus('Lese Truppenübersicht…');
    try {
      const r = await einlesenUndSpeichern();
      setStatus(`Gespeichert für ${anz(r.key)}: ${r.n} Mitglieder, bei ${r.frei} sind Truppen freigegeben.` + (r.stand ? ` Stand laut Spiel: ${r.stand}.` : '') + (r.ok ? '' : ' ACHTUNG: Speichern im Browser nicht möglich!'), !r.ok);
    } catch (e) {
      console.error(e);
      setStatus('Fehler: ' + e.message + ' – es werden die bereits gespeicherten Daten angezeigt.', true);
    } finally {
      q('#tv-lesen').disabled = false;
      fuelleAnsicht();
      renderTabelle();
    }
  }

  function bauen() {
    if (document.getElementById('tv-box')) return;
    const style = document.createElement('style');
    style.textContent = `
      #tv-box{position:fixed;top:4vh;left:50%;transform:translateX(-50%);width:min(1250px,96vw);max-height:92vh;overflow:auto;z-index:99999;background:#f4e4bc;border:2px solid #7d510f;padding:12px;font-size:12px;color:#000;box-shadow:0 4px 20px rgba(0,0,0,.5)}
      #tv-box h3{margin:0 0 8px}
      #tv-box .tv-close{position:absolute;right:8px;top:6px;cursor:pointer;font-size:18px;font-weight:bold}
      #tv-box textarea{width:100%;box-sizing:border-box}
      #tv-box .tv-tab{border-collapse:collapse;width:100%}
      #tv-box .tv-tab th{background:#c1a264;padding:3px 6px;text-align:left;white-space:nowrap}
      #tv-box .tv-tab td{padding:2px 6px;border-bottom:1px solid #d8c18a;white-space:nowrap}
      #tv-box .tv-tab td.n{text-align:right}
      #tv-box .tv-tab tr.tv-sum td{background:#e6d19b;font-weight:bold}
      #tv-box button{margin:4px 6px 4px 0;cursor:pointer}
      #tv-box select{margin-right:10px}
      #tv-table{max-height:58vh;overflow:auto;margin-top:6px}
    `;
    document.head.appendChild(style);
    const box = document.createElement('div');
    box.id = 'tv-box';
    box.innerHTML = `
      <span class="tv-close" title="Schließen">×</span>
      <h3>Truppen-Verlauf – Welt ${esc(gd.world)}</h3>
      <div id="tv-status" style="margin-bottom:6px"></div>
      <div>
        Ansicht: <select id="tv-ansicht"></select>
        Vergleich mit: <select id="tv-vergleich">
          <option value="1">Vortag</option><option value="7" selected>vor 7 Tagen</option><option value="14">vor 14 Tagen</option>
          <option value="30">vor 30 Tagen</option><option value="alt">ältestem gespeicherten Stand</option></select>
        <label><input type="checkbox" id="tv-sortd"> nach Veränderung sortieren</label>
      </div>
      <div>
        <button id="tv-lesen" title="Truppenübersicht neu einlesen und für heute speichern">Neu einlesen</button>
        <button id="tv-kopier" title="Tabelle tabulatorgetrennt kopieren, zum Einfügen in Excel oder Google Tabellen">Für Tabelle kopieren</button>
        <button id="tv-csv" title="Alle gespeicherten Tage als CSV (ein Spieler pro Tag und Zeile)">CSV (alle Tage)</button>
        <button id="tv-png">Bild (PNG)</button><button id="tv-pngcopy">Bild kopieren</button>
        <button id="tv-bbbtn">BB-Code</button>
        <button id="tv-loesch" title="Löscht alle im Browser gespeicherten Truppen-Verläufe">Verlauf löschen</button>
      </div>
      <div id="tv-msg" style="min-height:16px"></div>
      <div id="tv-info" style="margin:4px 0"></div>
      <div id="tv-bbwrap" style="display:none"><textarea id="tv-bb" rows="8" readonly></textarea><button id="tv-copy">In Zwischenablage kopieren</button></div>
      <div id="tv-table"></div>
      <p style="margin:8px 0 0;color:#555">BH = Bauernhofplätze (Speer/Schwert/Axt/Bogen 1, Späher 2, LKav 4, Ramme 5, ber. Bogen 5, SKav 6, Kata 8, Paladin 10, AG 100). Off-BH = Axt, LKav, Ramme, Kata, ber. Bogen; Deff-BH = Speer, Schwert, Bogen, SKav. Die Zahlen sind die Truppen jedes Spielers inklusive der Truppen außerhalb der Dörfer, so wie sie im Spiel für den Stamm freigegeben sind. Der Verlauf wird nur im eigenen Browser gespeichert: Wer Browserdaten löscht, verliert ihn, deshalb ab und zu „CSV (alle Tage)“ sichern.</p>`;
    document.body.appendChild(box);
    q('.tv-close').onclick = () => (box.style.display = 'none');
    q('#tv-lesen').onclick = lesen;
    q('#tv-ansicht').onchange = renderTabelle;
    q('#tv-vergleich').onchange = renderTabelle;
    q('#tv-sortd').onchange = renderTabelle;
    q('#tv-kopier').onclick = () => { if (modell()) kopieren(tsv(), 'Tabelle kopiert – in Excel oder Google Tabellen mit Strg+V einfügen.'); };
    q('#tv-csv').onclick = () => { if (modell()) dateiLaden(`truppen_${gd.world}_${tagKey(new Date())}.csv`, csvAlle(), 'text/csv;charset=utf-8'); };
    q('#tv-png').onclick = async () => { if (modell()) { dateiLaden(`truppen_${gd.world}_${tagKey(new Date())}.png`, await bildBlob()); meldungSetzen('Bild gespeichert.'); } };
    q('#tv-pngcopy').onclick = async () => {
      if (!modell()) return;
      try {
        if (!(navigator.clipboard && window.ClipboardItem)) throw new Error('nicht unterstützt');
        await navigator.clipboard.write([new ClipboardItem({ 'image/png': bildBlob() })]);
        meldungSetzen('Bild kopiert – in Discord mit Strg+V einfügen.');
      } catch (e) { meldungSetzen('Kopieren nicht möglich – bitte „Bild (PNG)“ nutzen.', true); }
    };
    q('#tv-bbbtn').onclick = () => { if (modell()) { q('#tv-bbwrap').style.display = 'block'; q('#tv-bb').value = bbCode(); } };
    q('#tv-copy').onclick = () => kopieren(q('#tv-bb').value, 'BB-Code kopiert.');
    let sicher = false;
    q('#tv-loesch').onclick = () => {
      const b = q('#tv-loesch');
      if (!sicher) { sicher = true; b.textContent = 'Wirklich alles löschen?'; setTimeout(() => { sicher = false; b.textContent = 'Verlauf löschen'; }, 5000); return; }
      sicher = false; b.textContent = 'Verlauf löschen';
      try { localStorage.removeItem(HKEY); meldungSetzen('Gespeicherter Verlauf gelöscht.'); } catch (x) { meldungSetzen('Löschen nicht möglich.', true); }
      fuelleAnsicht(); renderTabelle();
    };
    fuelleAnsicht();
    renderTabelle();
  }

  function open() {
    bauen();
    document.getElementById('tv-box').style.display = 'block';
    lesen();
  }

  window.__tvVerlauf = { open };

  if (USERSCRIPT_MODE) {
    const b = document.createElement('button');
    b.textContent = 'Truppen-Verlauf';
    b.style.cssText = 'position:fixed;left:8px;bottom:40px;z-index:99998;cursor:pointer';
    b.onclick = open;
    document.body.appendChild(b);
    // einmal pro Tag automatisch im Hintergrund speichern
    setTimeout(() => { if (!ladeHist()[tagKey(new Date())]) einlesenUndSpeichern().catch((e) => console.warn('Truppen-Verlauf:', e.message)); }, 3000);
  } else {
    open();
  }
})();
