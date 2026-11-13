<!doctype html>
<html lang="ro">
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Brick Breaker — Administrare jucători</title>
<style>
  :root { color-scheme: dark; font: 16px/1.5 system-ui, sans-serif; color: #edf0fb; background: #10131d; }
  body { margin: auto; padding: 2rem 1rem; max-width: 55rem; }
  h1, h2 { margin-top: 0; line-height: 1.15; }
  section { background: #1a2030; border: 1px solid #364159; border-radius: 14px; padding: 1.25rem; margin: 1rem 0; }
  label, legend { font-weight: 600; }
  .field { display: grid; gap: .3rem; margin: .75rem 0; }
  .row { display: flex; flex-wrap: wrap; gap: .6rem; align-items: center; }
  input, select, textarea, button, .upload { font: inherit; border: 1px solid #465573; border-radius: 7px; background: #121827; color: inherit; padding: .55rem .75rem; }
  .upload, button { cursor: pointer; }
  button.primary { background: #b7c8ff; color: #11182b; border-color: transparent; font-weight: 700; }
  button:disabled { cursor: default; opacity: .45; }
  [hidden] { display: none !important; }
  fieldset { border: 1px solid #465573; border-radius: 7px; margin: .8rem 0; }
  #resetOptions { display: grid; grid-template-columns: repeat(auto-fit, minmax(13rem, 1fr)); gap: .4rem; }
  #resetOptions label { font-weight: 400; }
  #status { min-height: 1.5em; margin-bottom: 0; color: #d1dcff; }
  #status[data-error="true"] { color: #ffb9b9; }
  #items { padding-left: 1.3rem; overflow-wrap: anywhere; }
  #items li { margin: .6rem 0; }
  #items button { margin-left: .5rem; }
  small, .muted { color: #b9c3da; }
</style>
<h1>Administrare jucători</h1>
<p>Brick Breaker Remastered · Editor pentru <code>player_commands.v1.json</code></p>
<section>
  <h2>Comandă nouă</h2>
  <form id="commandForm">
    <label class="field">Acțiune
      <select id="action">
        <option value="inbox">Mesaj în inbox</option>
        <option value="ban">Ban temporar</option>
        <option value="unban">Ridică banul</option>
        <option value="reset">Reset selectiv</option>
        <option value="tester_title">Titlu Tester</option>
      </select>
    </label>
    <label class="field">Destinatar
      <select id="scope"><option value="one">Un jucător</option><option value="all">Toți jucătorii</option></select>
    </label>
    <label class="field" id="targetField">Player ID
      <input id="target" maxlength="80" required placeholder="Copiază ID-ul din Settings">
    </label>
    <label class="field">Comanda poate fi primită până la
      <input id="expiry" type="datetime-local" required>
      <small>Ora locală. Lasă comanda în fișier cât timp vrei să ajungă la jucătorii care se conectează.</small>
    </label>
    <div data-action="inbox">
      <label class="field">Titlu <input id="title" maxlength="64" required></label>
      <label class="field">Mesaj <textarea id="message" rows="4" maxlength="700" required></textarea></label>
    </div>
    <div data-action="ban" hidden>
      <label class="field">Ban până la <input id="until" type="datetime-local" required></label>
      <label class="field">Motiv afișat jucătorului <textarea id="reason" rows="2" maxlength="280"></textarea></label>
    </div>
    <fieldset data-action="reset" hidden>
      <legend>Date de resetat</legend>
      <div id="resetOptions"></div>
    </fieldset>
    <label class="field" data-action="tester_title" hidden>Modificare
      <select id="tester"><option value="true">Acordă titlul</option><option value="false">Retrage titlul</option></select>
    </label>
    <label id="confirmField" hidden><input type="checkbox" id="confirmAll" required> Confirm acțiunea pentru toți jucătorii</label>
    <div class="row"><button type="submit" class="primary">Adaugă la listă</button></div>
  </form>
</section>
<section>
  <h2>Comenzi din feed</h2>
  <p class="muted">GitHub Pages publică acest fișier. Folosește texte fără date personale. Importă feedul publicat înainte să adaugi comenzi într-o sesiune nouă.</p>
  <div class="row">
    <label class="upload" for="import">Importă JSON</label>
    <input type="file" id="import" accept=".json,application/json" hidden>
    <button id="prune" type="button">Elimină comenzile expirate</button>
    <button id="export" type="button" class="primary">Descarcă feedul</button>
  </div>
  <ol id="items"></ol>
  <p id="status" role="status" aria-live="polite"></p>
</section>
<script>
  'use strict';
  const get = (id) => document.getElementById(id);
  const actions = {
    inbox: 'Mesaj în inbox', ban: 'Ban temporar', unban: 'Ridică banul',
    reset: 'Reset selectiv', tester_title: 'Titlu Tester'
  };
  const fields = {
    score: 'Record de scor', coins: 'Sold coins', gems: 'Sold gems', name: 'Nume',
    wave_record: 'Record Waves', waves_run: 'Run Waves salvat', arcade: 'Progres și stele Arcade',
    xp: 'Nivel și XP', daily_streak: 'Streak zilnic', brick_drops: 'Brick Drops disponibile',
    permanent_upgrades: 'Upgrade-uri permanente'
  };
  const confirmActions = ['ban', 'reset', 'tester_title'];
  const toLocalInput = (date) => new Date(date - date.getTimezoneOffset() * 60000).toISOString().slice(0, 16);
  const now = () => Math.floor(Date.now() / 1000);
  const timestamp = (seconds) => Number.isInteger(seconds) && seconds > 0 && seconds <= 4102444800;
  const secondsFrom = (id) => Math.floor(new Date(get(id).value).getTime() / 1000);
  const dateLabel = (seconds) => new Date(seconds * 1000).toLocaleString('ro-RO');
  const sized = (value, min, max) => typeof value === 'string' && [...value].length >= min && [...value].length <= max;
  let commands = [];

  function status(message, error = false) {
    get('status').textContent = message;
    get('status').dataset.error = error;
  }
  for (const [field, label] of Object.entries(fields)) {
    const node = document.createElement('label');
    const checkbox = document.createElement('input');
    checkbox.type = 'checkbox';
    checkbox.value = field;
    node.append(checkbox, ' ' + label);
    get('resetOptions').append(node);
  }
  get('expiry').value = toLocalInput(new Date(Date.now() + 7 * 86400000));
  get('until').value = toLocalInput(new Date(Date.now() + 86400000));

  function refreshForm() {
    const action = get('action').value;
    const all = get('scope').value === 'all';
    for (const group of document.querySelectorAll('[data-action]')) {
      group.hidden = group.dataset.action !== action;
      for (const input of group.querySelectorAll('input, select, textarea')) input.disabled = group.hidden;
    }
    get('targetField').hidden = get('target').disabled = all;
    get('confirmField').hidden = get('confirmAll').disabled = !all || !confirmActions.includes(action);
    get('confirmAll').checked = false;
  }

  function valid(command) {
    if (!command || !sized(command.id, 1, 64) || !sized(command.target, 1, 80)
        || command.target === 'NONE' || !Object.hasOwn(actions, command.action) || !timestamp(command.expires_at)) return false;
    if (command.target === 'all' && confirmActions.includes(command.action) && command.confirm_all !== true) return false;
    switch (command.action) {
      case 'inbox': return sized(command.title, 1, 64) && sized(command.text, 1, 700);
      case 'ban': return timestamp(command.until) && sized(command.reason, 0, 280);
      case 'unban': return true;
      case 'reset': return Array.isArray(command.fields) && command.fields.length > 0
        && new Set(command.fields).size === command.fields.length && command.fields.every((field) => Object.hasOwn(fields, field));
      case 'tester_title': return typeof command.enabled === 'boolean';
    }
  }
  function render() {
    get('items').replaceChildren();
    if (commands.length === 0) {
      const empty = document.createElement('li');
      empty.textContent = 'Lista este goală.';
      get('items').append(empty);
    }
    for (const command of commands) {
      const row = document.createElement('li');
      const target = command.target === 'all' ? 'Toți' : command.target;
      const detail = command.action === 'reset' ? command.fields.map((field) => fields[field]).join(', ')
        : command.action === 'inbox' ? command.title : command.action === 'ban' ? 'până la ' + dateLabel(command.until)
        : command.action === 'tester_title' ? (command.enabled ? 'acordă' : 'retrage') : '';
      const message = [actions[command.action], target, detail, 'expiră ' + dateLabel(command.expires_at)].filter(Boolean).join(' · ');
      const remove = document.createElement('button');
      remove.type = 'button';
      remove.textContent = 'Scoate';
      remove.addEventListener('click', () => {
        commands = commands.filter(({ id }) => id !== command.id);
        render();
        status('Comanda a fost scoasă din listă. Efectele aplicate în joc rămân salvate.');
      });
      row.append(message, remove);
      get('items').append(row);
    }
  }
  get('action').addEventListener('change', refreshForm);
  get('scope').addEventListener('change', refreshForm);
  get('commandForm').addEventListener('submit', (event) => {
    event.preventDefault();
    if (commands.length >= 100) return status('Feedul permite 100 de comenzi. Elimină comenzile expirate.', true);
    const action = get('action').value;
    const all = get('scope').value === 'all';
    const target = all ? 'all' : get('target').value.trim();
    if (!all && target.toLowerCase() === 'all') return status('Selectează explicit „Toți jucătorii” pentru acest destinatar.', true);
    const id = 'studio-' + [...crypto.getRandomValues(new Uint8Array(16))].map((byte) => byte.toString(16).padStart(2, '0')).join('');
    const command = { id, target, action, expires_at: secondsFrom('expiry') };
    if (all && confirmActions.includes(action)) command.confirm_all = get('confirmAll').checked;
    if (action === 'inbox') Object.assign(command, { title: get('title').value.trim(), text: get('message').value.trim() });
    if (action === 'ban') Object.assign(command, { until: secondsFrom('until'), reason: get('reason').value.trim() });
    if (action === 'reset') command.fields = [...get('resetOptions').querySelectorAll(':checked')].map((checkbox) => checkbox.value);
    if (action === 'tester_title') command.enabled = get('tester').value === 'true';
    if (!valid(command) || command.expires_at <= now() || (action === 'ban' && command.until <= now())) {
      return status('Verifică destinatarul, datele viitoare și câmpurile obligatorii.', true);
    }
    commands.push(command);
    render();
    refreshForm();
    status('Comanda a fost adăugată. Descarcă feedul când lista este gata.');
  });
  get('import').addEventListener('change', async (event) => {
    const file = event.target.files[0];
    if (!file) return;
    try {
      const feed = JSON.parse(await file.text());
      if (feed.version !== 1 || !Array.isArray(feed.commands) || feed.commands.length > 100
          || !feed.commands.every(valid) || new Set(feed.commands.map(({ id }) => id)).size !== feed.commands.length) throw Error();
      commands = feed.commands;
      render();
      status('Feedul a fost importat.');
    } catch {
      status('Fișierul nu respectă formatul feedului.', true);
    } finally {
      event.target.value = '';
    }
  });
  get('prune').addEventListener('click', () => {
    const before = commands.length;
    const time = now();
    commands = commands.filter((command) => command.expires_at > time && (command.action !== 'ban' || command.until > time));
    render();
    status('Comenzi expirate eliminate: ' + (before - commands.length) + '.');
  });
  get('export').addEventListener('click', () => {
    const json = JSON.stringify({ version: 1, commands }, null, 2) + '\n';
    if ([...json].length > 262144) return status('Feedul e prea mare. Redu numărul sau lungimea mesajelor.', true);
    const link = document.createElement('a');
    link.href = URL.createObjectURL(new Blob([json], { type: 'application/json' }));
    link.download = 'player_commands.v1.json';
    link.click();
    setTimeout(() => URL.revokeObjectURL(link.href), 1000);
    status('Feedul a fost descărcat.');
  });
  refreshForm();
  render();
</script>
</html>
