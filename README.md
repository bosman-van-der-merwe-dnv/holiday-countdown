// Holiday Countdown — uses Nager.Date API
const API_BASE = 'https://date.nager.at/api/v3';
const countrySelect = document.getElementById('country');
const refreshBtn = document.getElementById('refresh');
const errorEl = document.getElementById('error');
const nextCard = document.getElementById('nextHoliday');
const upcomingEl = document.getElementById('upcoming');
const holidayNameEl = document.getElementById('holidayName');
const holidayDateEl = document.getElementById('holidayDate');
const localInfoEl = document.getElementById('localInfo');
const daysEl = document.getElementById('days');
const hoursEl = document.getElementById('hours');
const minutesEl = document.getElementById('minutes');
const secondsEl = document.getElementById('seconds');

let countdownTimer = null;

function showError(msg) {
  errorEl.textContent = msg;
  errorEl.classList.remove('hidden');
}

function clearError() {
  errorEl.classList.add('hidden');
  errorEl.textContent = '';
}

async function fetchJSON(url) {
  const res = await fetch(url);
  if (!res.ok) throw new Error(`API error: ${res.status}`);
  return res.json();
}

async function loadCountries() {
  try {
    const countries = await fetchJSON(`${API_BASE}/AvailableCountries`);
    countries.sort((a, b) => a.name.localeCompare(b.name));
    countrySelect.innerHTML = countries
      .map((c) => `<option value="${c.countryCode}">${c.name} (${c.countryCode})</option>`)
      .join('');

    let guess = 'US';
    const nav = (navigator.language || navigator.userLanguage || '').toLowerCase();
    const matching = countries.find((c) => c.countryCode.toLowerCase() === nav.split('-')[1]?.toLowerCase());
    if (matching) guess = matching.countryCode;
    else if (countries.some((c) => c.countryCode.toLowerCase() === nav.split('-')[0]?.toLowerCase())) {
      guess = nav.split('-')[0].toUpperCase();
    }

    countrySelect.value = guess;
  } catch (err) {
    console.error(err);
    showError('Failed to load country list. Try refreshing the page.');
  }
}

function formatLocalDate(isoDate) {
  const d = new Date(isoDate + 'T00:00:00');
  return d.toLocaleDateString(undefined, {
    weekday: 'long',
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  });
}

function plural(n, s) {
  return `${n} ${s}${n === 1 ? '' : 's'}`;
}

function startCountdown(targetDate) {
  if (countdownTimer) clearInterval(countdownTimer);

  function tick() {
    const now = new Date();
    let diffMs = targetDate - now;

    if (diffMs <= 0) {
      updateForCountry(countrySelect.value);
      return;
    }

    const totalSeconds = Math.floor(diffMs / 1000);
    const days = Math.floor(totalSeconds / 86400);
    let rem = totalSeconds - days * 86400;
    const hours = Math.floor(rem / 3600);
    rem -= hours * 3600;
    const minutes = Math.floor(rem / 60);
    const seconds = rem - minutes * 60;

    daysEl.textContent = days;
    hoursEl.textContent = String(hours).padStart(2, '0');
    minutesEl.textContent = String(minutes).padStart(2, '0');
    secondsEl.textContent = String(seconds).padStart(2, '0');
  }

  tick();
  countdownTimer = setInterval(tick, 1000);
}

async function updateForCountry(countryCode) {
  clearError();
  nextCard.classList.add('hidden');
  document.getElementById('list').classList.add('hidden');
  upcomingEl.innerHTML = '';

  try {
    const today = new Date();
    const year = today.getFullYear();

    const [holsThis, holsNext] = await Promise.all([
      fetchJSON(`${API_BASE}/PublicHolidays/${year}/${countryCode}`),
      fetchJSON(`${API_BASE}/PublicHolidays/${year + 1}/${countryCode}`)
    ]);

    const all = [...holsThis, ...holsNext]
      .map((h) => ({ ...h, dateObj: new Date(h.date + 'T00:00:00') }))
      .filter((h) => h.dateObj >= new Date(today.toDateString()));

    if (all.length === 0) {
      showError('No holiday data found for this country/timeframe.');
      return;
    }

    all.sort((a, b) => a.dateObj - b.dateObj);
    const next = all[0];

    holidayNameEl.textContent = `${next.localName} (${next.name})`;
    holidayDateEl.textContent = formatLocalDate(next.date);

    const target = new Date(next.date + 'T00:00:00');
    const now = new Date();
    if (target.toDateString() === now.toDateString()) {
      const endOfDay = new Date();
      endOfDay.setHours(23, 59, 59, 999);
      startCountdown(endOfDay);
      localInfoEl.textContent = 'Today — counting down to the end of the day in your local time.';
    } else {
      startCountdown(target);
      localInfoEl.textContent = `Countdown to the start of the holiday in your local time (${next.countryCode}).`;
    }

    nextCard.classList.remove('hidden');

    const max = Math.min(8, all.length);
    upcomingEl.innerHTML = all
      .slice(0, max)
      .map((h) => {
        const daysAway = Math.ceil((h.dateObj - new Date()) / 86400000);
        return `<li><strong>${h.localName}</strong> — ${formatLocalDate(h.date)} <span class="muted">(${plural(daysAway, 'day')} away)</span></li>`;
      })
      .join('');

    document.getElementById('list').classList.remove('hidden');
  } catch (err) {
    console.error(err);
    showError('Failed to load holiday data. Try Refresh. (Check network or API availability.)');
  }
}

countrySelect.addEventListener('change', () => updateForCountry(countrySelect.value));
refreshBtn.addEventListener('click', () => updateForCountry(countrySelect.value));

(async function init() {
  await loadCountries();
  updateForCountry(countrySelect.value);
})();
