---
layout: default
title: חיפוש AI - בגץ-AI
description: חיפוש סמנטי בפסקי דין של בית המשפט העליון לפי תיאור חופשי בעברית. בחרו תיאור של פסק דין וקבלו את 5 פסקי הדין הקרובים ביותר.
---

<style>
  .ai-search {
    max-width: 720px;
    margin: 0 auto;
  }
  .ai-search__intro {
    color: #555;
    margin-bottom: 1rem;
  }
  .ai-search__form {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }
  .ai-search__textarea {
    width: 100%;
    box-sizing: border-box;
    min-height: 130px;
    padding: 0.75rem;
    font: inherit;
    border: 1px solid var(--border-color);
    border-radius: 6px;
    resize: vertical;
  }
  .ai-search__textarea:disabled {
    background: #f0f0f0;
    color: #888;
  }
  .ai-search__meta {
    display: flex;
    justify-content: space-between;
    align-items: center;
    font-size: 0.9rem;
    color: #666;
  }
  .ai-search__counter--over {
    color: #b00020;
    font-weight: 600;
  }
  .ai-search__button {
    align-self: flex-start;
    background: var(--primary-color);
    color: #fff;
    border: 0;
    border-radius: 6px;
    padding: 0.6rem 1.4rem;
    font: inherit;
    cursor: pointer;
  }
  .ai-search__button:disabled {
    background: #999;
    cursor: not-allowed;
  }
  .ai-search__banner {
    margin: 1rem 0;
    padding: 0.75rem 1rem;
    border-radius: 6px;
    border: 1px solid transparent;
  }
  .ai-search__banner--error {
    background: #fdecea;
    border-color: #f5c2c0;
    color: #6b1815;
  }
  .ai-search__banner--info {
    background: #eef4fb;
    border-color: #c7d8ee;
    color: #1e3a5f;
  }
  .ai-search__results {
    list-style: none;
    padding: 0;
    margin: 1.5rem 0 0 0;
  }
  .ai-search__result {
    border: 1px solid var(--border-color);
    border-radius: 6px;
    padding: 0.9rem 1rem;
    margin-bottom: 0.75rem;
    background: #fff;
  }
  .ai-search__result a {
    color: var(--primary-color);
    text-decoration: none;
    font-weight: 700;
  }
  .ai-search__result a:hover {
    text-decoration: underline;
  }
  .ai-search__result-meta {
    display: flex;
    gap: 1rem;
    font-size: 0.85rem;
    color: #666;
    margin-top: 0.25rem;
  }
  .ai-search__similarity {
    font-weight: 600;
  }
  .ai-search__spinner {
    display: none;
    margin-inline-start: 0.5rem;
    width: 16px;
    height: 16px;
    border: 2px solid #ccc;
    border-top-color: var(--primary-color);
    border-radius: 50%;
    animation: ai-search-spin 0.7s linear infinite;
    vertical-align: middle;
  }
  .ai-search__spinner.is-active {
    display: inline-block;
  }
  @keyframes ai-search-spin {
    to { transform: rotate(360deg); }
  }
  .visually-hidden {
    position: absolute;
    width: 1px; height: 1px;
    padding: 0; margin: -1px;
    overflow: hidden; clip: rect(0,0,0,0);
    white-space: nowrap; border: 0;
  }
</style>

# חיפוש AI

<section class="ai-search">

  <p class="ai-search__intro">
    תארו במילים שלכם פסק דין שאתם מחפשים — את העובדות, את השאלה המשפטית או את התוצאה — ואנחנו נחזיר את 5 פסקי הדין הקרובים ביותר תוך שימוש בחיפוש סמנטי על סיכומי הניתוח שלנו.
  </p>

  <div id="ai-search-banner" class="ai-search__banner ai-search__banner--info" role="status" hidden></div>

  <form id="ai-search-form" class="ai-search__form" aria-busy="false">
    <label for="ai-search-input" class="visually-hidden">תיאור פסק הדין</label>
    <textarea
      id="ai-search-input"
      class="ai-search__textarea"
      placeholder="לדוגמה: עתירה כנגד החלטת רשות מקרקעי ישראל לסרב להאריך חוזה חכירה לחקלאי בנגב…"
      maxlength="800"
      minlength="20"
      required
      disabled
      aria-describedby="ai-search-counter"
    ></textarea>

    <div class="ai-search__meta">
      <span id="ai-search-counter">0 / 800 תווים</span>
      <span id="ai-search-remaining" aria-live="polite"></span>
    </div>

    <button id="ai-search-submit" type="submit" class="ai-search__button" disabled>
      חפש
      <span id="ai-search-spinner" class="ai-search__spinner" aria-hidden="true"></span>
    </button>
  </form>

  <ol id="ai-search-results" class="ai-search__results" aria-live="polite"></ol>

</section>

<script>
(function () {
  // Endpoint host can be overridden via <meta name="ai-search-endpoint" content="..."> for dev/staging.
  var metaEndpoint = document.querySelector('meta[name="ai-search-endpoint"]');
  var ENDPOINT_BASE = (metaEndpoint && metaEndpoint.content) || 'https://middleware.bagats-poc.win';
  var INPUT_MIN = 20;
  var INPUT_MAX = 800;
  var HEALTH_TIMEOUT_MS = 3000;

  var form = document.getElementById('ai-search-form');
  var input = document.getElementById('ai-search-input');
  var counter = document.getElementById('ai-search-counter');
  var remaining = document.getElementById('ai-search-remaining');
  var submitBtn = document.getElementById('ai-search-submit');
  var spinner = document.getElementById('ai-search-spinner');
  var resultsEl = document.getElementById('ai-search-results');
  var banner = document.getElementById('ai-search-banner');

  function showBanner(message, level) {
    banner.textContent = message;
    banner.className = 'ai-search__banner ai-search__banner--' + (level || 'info');
    banner.hidden = false;
  }

  function clearBanner() {
    banner.hidden = true;
    banner.textContent = '';
  }

  function disableForm(reasonMessage) {
    input.disabled = true;
    submitBtn.disabled = true;
    if (reasonMessage) {
      showBanner(reasonMessage, 'error');
    }
  }

  function enableForm() {
    input.disabled = false;
    submitBtn.disabled = false;
  }

  function updateCounter() {
    var len = input.value.length;
    counter.textContent = len + ' / ' + INPUT_MAX + ' תווים';
    counter.classList.toggle('ai-search__counter--over', len > INPUT_MAX);
  }

  function setBusy(busy) {
    form.setAttribute('aria-busy', busy ? 'true' : 'false');
    submitBtn.disabled = busy;
    spinner.classList.toggle('is-active', busy);
  }

  function renderResults(results) {
    resultsEl.innerHTML = '';
    if (!results || results.length === 0) {
      var li = document.createElement('li');
      li.className = 'ai-search__result';
      li.textContent = 'לא נמצאו פסקי דין דומים. נסו לתאר את פסק הדין במילים אחרות.';
      resultsEl.appendChild(li);
      return;
    }
    results.forEach(function (r, idx) {
      var li = document.createElement('li');
      li.className = 'ai-search__result';

      var titleLine = document.createElement('div');
      var link = document.createElement('a');
      link.href = r.judgment_url;
      var titleText = (idx + 1) + '. ' + r.case_number + (r.title ? ' — ' + r.title : '');
      link.textContent = titleText;
      titleLine.appendChild(link);
      li.appendChild(titleLine);

      var meta = document.createElement('div');
      meta.className = 'ai-search__result-meta';
      var sim = document.createElement('span');
      sim.className = 'ai-search__similarity';
      var pct = Math.round((r.similarity || 0) * 100);
      sim.textContent = 'התאמה: ' + pct + '%';
      meta.appendChild(sim);
      if (r.judgment_date) {
        var d = document.createElement('span');
        d.textContent = 'תאריך פסק הדין: ' + r.judgment_date;
        meta.appendChild(d);
      }
      li.appendChild(meta);

      resultsEl.appendChild(li);
    });
  }

  function fetchWithTimeout(url, options, ms) {
    var controller = new AbortController();
    var t = setTimeout(function () { controller.abort(); }, ms);
    var opts = Object.assign({}, options || {}, { signal: controller.signal });
    return fetch(url, opts).finally(function () { clearTimeout(t); });
  }

  // 1. Health check on page load — form starts disabled, enable only on success.
  fetchWithTimeout(ENDPOINT_BASE + '/health', {}, HEALTH_TIMEOUT_MS)
    .then(function (resp) {
      if (!resp.ok) {
        throw new Error('health ' + resp.status);
      }
      clearBanner();
      enableForm();
    })
    .catch(function () {
      disableForm('שירות החיפוש אינו זמין כעת. אנא נסו שוב מאוחר יותר.');
    });

  // 2. Live char counter.
  input.addEventListener('input', updateCounter);
  updateCounter();

  // 3. Submit handler.
  form.addEventListener('submit', function (event) {
    event.preventDefault();
    var query = (input.value || '').trim();
    if (query.length < INPUT_MIN) {
      showBanner('יש להזין לפחות ' + INPUT_MIN + ' תווים.', 'error');
      return;
    }
    if (query.length > INPUT_MAX) {
      showBanner('הטקסט ארוך מדי — מקסימום ' + INPUT_MAX + ' תווים.', 'error');
      return;
    }

    clearBanner();
    setBusy(true);
    resultsEl.innerHTML = '';

    fetch(ENDPOINT_BASE + '/ai_search', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ query: query })
    })
      .then(function (resp) {
        return resp.json().then(function (data) { return { status: resp.status, data: data }; });
      })
      .then(function (env) {
        var data = env.data || {};
        if (env.status === 200) {
          renderResults(data.results || []);
          if (typeof data.remaining_today === 'number') {
            remaining.textContent = 'נותרו לך ' + data.remaining_today + ' חיפושים היום.';
          }
        } else if (env.status === 422) {
          showBanner(typeof data.detail === 'string' ? data.detail : 'הקלט אינו תקין.', 'error');
        } else if (env.status === 429 || env.status === 503) {
          showBanner(data.error || data.detail || 'השירות אינו זמין כעת. אנא נסו שוב מאוחר יותר.', 'error');
        } else {
          showBanner('אירעה שגיאה. אנא נסו שוב מאוחר יותר.', 'error');
        }
      })
      .catch(function () {
        showBanner('אין חיבור לשירות. אנא נסו שוב מאוחר יותר.', 'error');
      })
      .finally(function () {
        setBusy(false);
      });
  });
})();
</script>
