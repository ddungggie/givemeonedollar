import { initializeApp } from "https://www.gstatic.com/firebasejs/10.14.1/firebase-app.js";
import {
  getAuth, onAuthStateChanged, signInWithPopup, GoogleAuthProvider, signOut
} from "https://www.gstatic.com/firebasejs/10.14.1/firebase-auth.js";
import {
  getFirestore, collection, addDoc, deleteDoc, doc, onSnapshot, query, orderBy, serverTimestamp
} from "https://www.gstatic.com/firebasejs/10.14.1/firebase-firestore.js";
import { firebaseConfig } from "./firebase-config.js";

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);

/* ---------- 유틸 ---------- */
const $ = id => document.getElementById(id);
const nf = new Intl.NumberFormat('ko-KR');
const won = n => nf.format(Math.round(n)) + '원';
const sgn = n => (n > 0 ? '+' : n < 0 ? '−' : '');
const wonS = n => sgn(n) + nf.format(Math.abs(Math.round(n))) + '원';
const pctS = n => sgn(n) + Math.abs(n).toFixed(2) + '%';
const cls = n => (n > 0 ? 'up' : n < 0 ? 'down' : '');
const esc = s => String(s).replace(/[&<>"']/g, c => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]));

function todayStr() {
  const d = new Date();
  return d.getFullYear() + '-' + String(d.getMonth() + 1).padStart(2, '0') + '-' + String(d.getDate()).padStart(2, '0');
}
function showErr(msg) {
  $('errBox').innerHTML = '<div class="msg err">' + esc(msg) + '</div>';
}
function clearErr() { $('errBox').innerHTML = ''; }

/* ---------- 상태 ---------- */
let purchases = [];
let currentRate = null;
let rateDate = '';
let history = [];       // [[date, rate], ...]
let historyStart = null;
let unsubscribe = null;
let rateChart = null, barChart = null;

/* ---------- 환율 (Frankfurter, 키 불필요) ---------- */
const API = 'https://api.frankfurter.dev/v1';

async function getJson(url) {
  const res = await fetch(url);
  if (!res.ok) throw new Error('환율 정보를 가져오지 못했습니다. (' + res.status + ')');
  return res.json();
}

async function loadRate() {
  const d = await getJson(API + '/latest?base=USD&symbols=KRW');
  currentRate = d.rates.KRW;
  rateDate = d.date;
}

async function loadHistory(startStr) {
  const s = new Date(startStr + 'T00:00:00Z');
  s.setUTCDate(s.getUTCDate() - 3);
  const from = s.toISOString().slice(0, 10);
  const d = await getJson(API + '/' + from + '..?base=USD&symbols=KRW');
  history = Object.keys(d.rates).sort().map(k => [k, d.rates[k].KRW]);
  historyStart = startStr;
}

async function reloadAll() {
  clearErr();
  $('refreshBtn').disabled = true;
  try {
    await loadRate();
    if (purchases.length) await loadHistory(purchases[0].date);
  } catch (e) {
    showErr(e.message);
  }
  $('refreshBtn').disabled = false;
  render();
}

/* ---------- 화면 ---------- */
function render() {
  const P = purchases;
  const big = $('plBig');

  if (currentRate == null) { big.className = 'big'; big.textContent = '불러오는 중…'; return; }

  const totalUsd = P.reduce((s, p) => s + p.usd, 0);
  const cost = P.reduce((s, p) => s + p.usd * p.rate, 0);
  const avg = totalUsd ? cost / totalUsd : 0;
  const value = totalUsd * currentRate;
  const pl = value - cost;
  const plPct = cost ? pl / cost * 100 : 0;

  if (!P.length) {
    big.className = 'big'; big.textContent = '내역 없음';
    $('plSub').className = 'sub'; $('plSub').textContent = '아래에서 첫 매수 내역을 추가해 주세요.';
  } else {
    big.className = 'big ' + cls(pl); big.textContent = wonS(pl);
    $('plSub').className = 'sub ' + cls(pl); $('plSub').textContent = pctS(plPct);
  }

  const gap = currentRate - avg;
  const stats = [
    ['현재 환율', currentRate.toFixed(2) + '원'],
    ['평균 매수환율', P.length ? avg.toFixed(2) + '원' : '–'],
    ['평균 대비', P.length ? '<span class="' + cls(gap) + '">' + sgn(gap) + Math.abs(gap).toFixed(2) + '원</span>' : '–'],
    ['보유 달러', P.length ? '$' + nf.format(Math.round(totalUsd * 100) / 100) : '–'],
    ['투자원금', P.length ? won(cost) : '–'],
    ['평가금액', P.length ? won(value) : '–']
  ];
  $('stats').innerHTML = stats.map(s => '<div class="stat"><dt>' + s[0] + '</dt><dd>' + s[1] + '</dd></div>').join('');

  $('rows').innerHTML = P.length ? P.map(p => {
    const c = p.usd * p.rate, g = p.usd * currentRate - c, gp = (currentRate / p.rate - 1) * 100;
    return '<tr><td>' + esc(p.date) + (p.memo ? '<small>' + esc(p.memo) + '</small>' : '') + '</td>' +
      '<td>' + p.rate.toFixed(2) + '</td><td>$' + nf.format(p.usd) + '</td><td>' + won(c) + '</td>' +
      '<td class="' + cls(g) + '">' + wonS(g) + '</td><td class="' + cls(g) + '">' + pctS(gp) + '</td>' +
      '<td><button class="del" data-id="' + esc(p.id) + '">삭제</button></td></tr>';
  }).join('') : '<tr><td colspan="7" style="text-align:center;color:var(--muted)">아직 입력한 내역이 없습니다.</td></tr>';

  $('updated').textContent = '시세 기준일: ' + rateDate + ' (유럽중앙은행 고시, 하루 1회 갱신)';
  drawCharts(avg);
}

function drawCharts(avg) {
  if (rateChart) { rateChart.destroy(); rateChart = null; }
  if (barChart) { barChart.destroy(); barChart = null; }
  if (!purchases.length || !history.length) return;

  const labels = history.map(r => r[0]);
  const rates = history.map(r => r[1]);
  const buyPts = labels.map(() => null);
  purchases.forEach(p => {
    let i = labels.findIndex(l => l >= p.date);
    if (i < 0) i = labels.length - 1;
    buyPts[i] = p.rate;
  });

  rateChart = new Chart($('rateChart'), {
    type: 'line',
    data: {
      labels,
      datasets: [
        { label: '환율', data: rates, borderColor: '#16241d', borderWidth: 2, pointRadius: 0, tension: 0.15 },
        { label: '평균 매수환율', data: labels.map(() => avg), borderColor: '#2f6b4f', borderDash: [6, 5], borderWidth: 2, pointRadius: 0 },
        { label: '내 매수', data: buyPts, showLine: false, pointRadius: 6, pointBackgroundColor: '#2f6b4f', pointBorderColor: '#fff', pointBorderWidth: 2 }
      ]
    },
    options: {
      responsive: true, maintainAspectRatio: false, interaction: { mode: 'index', intersect: false },
      plugins: { legend: { position: 'bottom' } },
      scales: { x: { ticks: { maxTicksLimit: 8 }, grid: { display: false } }, y: { ticks: { callback: v => nf.format(v) } } }
    }
  });

  const gains = purchases.map(p => (currentRate / p.rate - 1) * 100);
  barChart = new Chart($('barChart'), {
    type: 'bar',
    data: {
      labels: purchases.map(p => p.date),
      datasets: [{ label: '수익률(%)', data: gains, backgroundColor: gains.map(g => (g >= 0 ? '#d6453d' : '#2f6fd0')) }]
    },
    options: {
      responsive: true, maintainAspectRatio: false,
      plugins: { legend: { display: false }, tooltip: { callbacks: { label: c => pctS(c.parsed.y) } } },
      scales: { x: { grid: { display: false } }, y: { ticks: { callback: v => v + '%' } } }
    }
  });
}

/* ---------- 로그인 / Firestore ---------- */
onAuthStateChanged(auth, user => {
  if (unsubscribe) { unsubscribe(); unsubscribe = null; }
  $('login').hidden = !!user;
  $('app').hidden = !user;
  purchases = []; history = []; historyStart = null; currentRate = null;
  if (!user) return;

  $('userName').textContent = user.displayName || user.email || '';
  $('fDate').value = todayStr();
  render();
  reloadAll();

  const q = query(collection(db, 'users', user.uid, 'purchases'), orderBy('date'));
  unsubscribe = onSnapshot(q, async snap => {
    purchases = snap.docs.map(d => ({ id: d.id, ...d.data() }));
    try {
      if (purchases.length && purchases[0].date !== historyStart) await loadHistory(purchases[0].date);
    } catch (e) { showErr(e.message); }
    render();
  }, err => showErr('내역을 불러오지 못했습니다: ' + err.message));
});

$('loginBtn').addEventListener('click', async () => {
  $('loginErr').innerHTML = '';
  try {
    await signInWithPopup(auth, new GoogleAuthProvider());
  } catch (e) {
    $('loginErr').innerHTML = '<div class="msg err">' + esc(e.message) + '</div>';
  }
});
$('logoutBtn').addEventListener('click', () => signOut(auth));
$('refreshBtn').addEventListener('click', reloadAll);

$('addForm').addEventListener('submit', async e => {
  e.preventDefault();
  const user = auth.currentUser;
  if (!user) return;
  const btn = $('addBtn');
  const rate = Number($('fRate').value), usd = Number($('fUsd').value);
  if (!(rate > 0) || !(usd > 0)) { showErr('매수 환율과 금액을 확인해 주세요.'); return; }
  btn.disabled = true; btn.textContent = '저장 중…';
  try {
    await addDoc(collection(db, 'users', user.uid, 'purchases'), {
      date: $('fDate').value, rate, usd, memo: $('fMemo').value.trim(), createdAt: serverTimestamp()
    });
    $('fRate').value = ''; $('fUsd').value = ''; $('fMemo').value = '';
    clearErr();
  } catch (err) {
    showErr('저장하지 못했습니다: ' + err.message);
  }
  btn.disabled = false; btn.textContent = '매수 추가';
});

$('rows').addEventListener('click', async e => {
  const b = e.target.closest('button.del');
  const user = auth.currentUser;
  if (!b || !user || !confirm('이 매수 내역을 삭제할까요?')) return;
  b.disabled = true;
  try {
    await deleteDoc(doc(db, 'users', user.uid, 'purchases', b.dataset.id));
  } catch (err) {
    showErr('삭제하지 못했습니다: ' + err.message);
    b.disabled = false;
  }
});
