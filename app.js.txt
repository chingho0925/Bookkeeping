// ==========================================
// 1. Firebase 核心初始化與離線引擎
// ==========================================
const firebaseConfig = {
    apiKey: "AIzaSyD-NlX22-qoK2M_wFV6YUIzNGvzQgFldv4",
    authDomain: "bookkeeping-app-cf680.firebaseapp.com",
    projectId: "bookkeeping-app-cf680",
    storageBucket: "bookkeeping-app-cf680.firebasestorage.app",
    messagingSenderId: "480039184522",
    appId: "1:480039184522:web:31658c3229b8fa0cef2573",
};

firebase.initializeApp(firebaseConfig);
const auth = firebase.auth();
const db = firebase.firestore();
const googleProvider = new firebase.auth.GoogleAuthProvider();

// 🔥 啟用專業級特性：離線持久化 (Offline Persistence)
db.enablePersistence({ synchronizeTabs: true }).catch((err) => {
    console.warn("離線模式警告:", err.code === 'failed-precondition' ? "多重分頁限制" : "瀏覽器不支援");
});

// ==========================================
// 2. 狀態管理 (State Management)
// ==========================================
let appData = null;
let currentFilter = 'ALL';
let currentSort = 'date-desc';
let isSubmitting = false;

// 監聽器退訂函數
let unsubUser = null;
let unsubTrips = null;

const defaultAppData = {
    activeTripId: "", customUserName: "", theme: "system",
    rates: { TWD: 1, JPY: 0.21, USD: 32.5, EUR: 35.0, KRW: 0.024 },
    categories: ["餐飲", "交通", "住宿", "購物", "娛樂", "其他"],
    folders: [], trips: []
};

// ==========================================
// 3. 身分驗證與即時資料綁定 (Real-time Sync)
// ==========================================
auth.onAuthStateChanged(user => {
    if (user) {
        document.getElementById('login-overlay').style.display = 'none';
        showLoading(true);
        setupRealtimeListeners(user);
    } else {
        if(unsubUser) unsubUser(); if(unsubTrips) unsubTrips();
        appData = null;
        document.getElementById('main-app').style.display = 'none';
        document.getElementById('login-overlay').style.display = 'flex';
        applyTheme('system');
    }
});

function setupRealtimeListeners(user) {
    // 監聽全域使用者設定
    unsubUser = db.collection("users").doc(user.uid).onSnapshot(doc => {
        let uData = doc.exists ? doc.data() : JSON.parse(JSON.stringify(defaultAppData));
        if(!appData) appData = { ...uData, trips: [] };
        else appData = { ...appData, ...uData };
        applyTheme(appData.theme);
        updateUI();
    });

    // 監聽有權限的共用帳本
    unsubTrips = db.collection("trips").where("members", "array-contains", user.uid).onSnapshot(snap => {
        let trips = [];
        snap.forEach(d => trips.push(d.data()));
        
        if (trips.length === 0) {
            createNewTripDirectly("我的新帳本", user.uid);
            return;
        }
        
        if (appData) {
            appData.trips = trips;
            if (!appData.activeTripId || !trips.find(t => t.id === appData.activeTripId)) {
                appData.activeTripId = trips[0].id;
                db.collection("users").doc(user.uid).set({ activeTripId: trips[0].id }, { merge: true });
            }
            updateUI();
            showLoading(false);
        }
    });
}

// ==========================================
// 4. 極致渲染引擎 (DocumentFragment)
// ==========================================
function updateUI() {
    if (!appData || appData.trips.length === 0) return;
    const trip = appData.trips.find(t => t.id === appData.activeTripId) || appData.trips[0];
    
    // Header
    document.getElementById('header-trip-name').innerText = trip.name;
    document.getElementById('header-trip-period').innerText = `${trip.startDate} ~ ${trip.endDate}`;
    document.getElementById('settings-trip-name').innerText = trip.name;
    
    // Filter Rendering
    const usedCats = new Set(trip.expenses.map(e => e.category));
    const filterBox = document.getElementById('filterContainer');
    filterBox.innerHTML = `<div class="filter-chip ${currentFilter === 'ALL' ? 'active' : ''}" onclick="setFilter('ALL')">全部</div>`;
    usedCats.forEach(cat => { filterBox.innerHTML += `<div class="filter-chip ${currentFilter === cat ? 'active' : ''}" onclick="setFilter('${cat}')">${cat}</div>`; });

    // 🔥 渲染優化：使用 DocumentFragment 處理大量節點
    const listEl = document.getElementById('expense-list');
    listEl.innerHTML = ''; 
    const fragment = document.createDocumentFragment();
    
    let sortedExps = [...trip.expenses].sort((a, b) => {
        let vA = Math.round(a.amount * (a.rate || appData.rates[a.currency] || 1));
        let vB = Math.round(b.amount * (b.rate || appData.rates[b.currency] || 1));
        if (currentSort === 'date-desc') return new Date(b.date) - new Date(a.date) || b.id - a.id;
        if (currentSort === 'date-asc') return new Date(a.date) - new Date(b.date) || a.id - b.id;
        if (currentSort === 'amount-desc') return vB - vA || b.id - a.id;
        return vA - vB || a.id - b.id;
    });

    const displayExps = currentFilter === 'ALL' ? sortedExps : sortedExps.filter(e => e.category === currentFilter);
    let totalTWD = 0, filteredTWD = 0; let lastDate = ""; const payers = {};

    trip.expenses.forEach(e => { const v = getTwd(e); totalTWD += v; payers[e.payer||'我'] = (payers[e.payer||'我']||0) + v; });

    if (displayExps.length === 0) {
        const empty = document.createElement('div'); empty.className = 'empty-state'; empty.innerHTML = `<div class="empty-state-icon" style="font-size: 40px; margin-bottom:10px;">✨</div><div style="font-size:16px; font-weight:bold;">目前沒有任何紀錄</div>`; fragment.appendChild(empty);
    } else {
        displayExps.forEach(exp => {
            const v = getTwd(exp); filteredTWD += v;
            if (currentSort.startsWith('date-') && exp.date !== lastDate) {
                lastDate = exp.date;
                const dHead = document.createElement('div'); dHead.className = 'date-header'; dHead.innerText = exp.date; fragment.appendChild(dHead);
            }
            
            const div = document.createElement('div'); div.className = 'expense-item';
            div.onclick = () => openEditExpenseModal(exp.id);
            div.innerHTML = `
                <div class="flex-1">
                    <strong style="font-size:16px;">${exp.name}</strong> 
                    <span style="background:var(--theme-secondary); padding:2px 8px; border-radius:6px; font-size:12px; margin-left:6px;">${exp.payer||'我'}</span>
                    ${exp.details ? `<br><small style="color:var(--theme-text-muted);">📝 ${exp.details}</small>` : ''}
                    <br><small style="color:var(--theme-primary); font-weight:bold;">${exp.category}</small>
                </div>
                <div style="text-align: right;">
                    ${exp.currency === 'TWD' ? `<strong>TWD ${exp.amount.toLocaleString()}</strong>` : `<strong style="color:var(--theme-accent);">${exp.currency}${exp.amount}</strong><br><small style="color:var(--theme-text-muted);">≈ NT$ ${v.toLocaleString()}</small>`}
                </div>`;
            fragment.appendChild(div);
        });
    }
    
    listEl.appendChild(fragment);

    // Stats Updates
    document.getElementById('record-total').innerText = `NT$ ${filteredTWD.toLocaleString()}`;
    document.getElementById('total-spent').innerText = totalTWD.toLocaleString();
    document.getElementById('total-budget').innerText = (trip.budget||0).toLocaleString();
    
    let pct = trip.budget > 0 ? Math.min((totalTWD / trip.budget) * 100, 100) : 0;
    document.getElementById('budget-fill').style.width = pct + '%';
    document.getElementById('budget-fill').style.background = pct >= 100 ? 'var(--theme-danger)' : 'var(--theme-primary)';
    document.getElementById('budget-text').innerText = `${Math.round(pct)}%`;
    
    renderSelectOptions('itemCurrency', appData.rates, 'TWD');
    renderSelectOptions('itemCategory', appData.categories, appData.categories[0], true);
}

// ==========================================
// 5. 互動邏輯 (算式解析、防連點)
// ==========================================
function parseAmount(input) {
    try {
        const safeStr = input.replace(/[^0-9\.\+\-\*\/]/g, '');
        if(!safeStr) return NaN;
        return Math.round(Function(`'use strict'; return (${safeStr})`)() * 100) / 100;
    } catch(e) { return NaN; }
}

document.getElementById('itemAmount').addEventListener('blur', function() {
    let val = parseAmount(this.value);
    if(!isNaN(val)) this.value = val;
});

document.getElementById('btn-submit-expense').onclick = async function() {
    if (isSubmitting) return;
    
    const name = document.getElementById('itemName').value.trim();
    const rawAmt = document.getElementById('itemAmount').value;
    const amount = parseAmount(rawAmt);
    const date = document.getElementById('itemDate').value;
    const curr = document.getElementById('itemCurrency').value;
    
    if (!name || isNaN(amount) || amount <= 0 || !date) return showToast('請填寫完整資訊 (金額支援簡易算式)', 'error');
    
    isSubmitting = true;
    this.innerText = '⏳ 儲存中...';
    this.disabled = true;

    try {
        const trip = appData.trips.find(t => t.id === appData.activeTripId);
        const newExp = {
            id: Date.now(), name, amount, date, currency: curr,
            category: document.getElementById('itemCategory').value,
            payer: document.getElementById('itemPayer').value.trim() || '我',
            details: document.getElementById('enableDetailsBtn').checked ? document.getElementById('itemDetails').value.trim() : "",
            rate: curr === 'TWD' ? 1 : (appData.rates[curr] || 1)
        };
        
        trip.expenses.unshift(newExp);
        await db.collection("trips").doc(trip.id).set({ expenses: trip.expenses }, { merge: true });
        
        document.getElementById('itemName').value = ''; document.getElementById('itemAmount').value = ''; 
        closeModal('addExpenseModal');
        showToast("✅ 紀錄已新增");
    } catch(e) {
        showToast("寫入失敗", "error");
    } finally {
        isSubmitting = false;
        this.innerText = '🚀 記上一筆';
        this.disabled = false;
    }
};

// ==========================================
// 6. 基礎工具與輔助 (Helpers)
// ==========================================
function getTwd(exp) { return Math.round(exp.amount * (exp.rate || appData.rates[exp.currency] || 1)); }
function showToast(msg, type='success') {
    const box = document.getElementById('toast-container');
    const t = document.createElement('div'); t.className='toast-msg';
    t.style.background = type==='error'?'var(--theme-danger)':'var(--theme-primary)';
    t.style.color = '#FFF'; t.innerText = msg; box.appendChild(t);
    setTimeout(()=>t.style.opacity='1', 10);
    setTimeout(()=>{ t.style.opacity='0'; setTimeout(()=>t.remove(), 300); }, 2500);
}
function setFilter(cat) { currentFilter = cat; updateUI(); }
function showLoading(show) { document.getElementById('loading-overlay').style.display = show ? 'flex' : 'none'; }
function openModal(id) { document.getElementById(id).classList.add('open'); }
function closeModal(id) { document.getElementById(id).classList.remove('open'); }

document.querySelectorAll('.close-btn').forEach(btn => {
    btn.onclick = function() { const target = this.getAttribute('data-target'); if(target) closeModal(target); }
});

document.querySelectorAll('.nav-item').forEach(el => {
    el.onclick = function() {
        document.querySelectorAll('.nav-item, .tab-view').forEach(e => e.classList.remove('active'));
        this.classList.add('active');
        document.getElementById('tab-' + this.getAttribute('data-tab')).classList.add('active');
        document.getElementById('fabBtn').style.display = this.getAttribute('data-tab') === 'records' ? 'flex' : 'none';
    }
});

document.getElementById('themeSelect').addEventListener('change', function() {
    appData.theme = this.value;
    db.collection("users").doc(auth.currentUser.uid).set({ theme: this.value }, { merge: true });
    applyTheme(this.value);
});

function applyTheme(theme) {
    if (theme === 'light') document.documentElement.setAttribute('data-theme', 'light');
    else if (theme === 'dark') document.documentElement.setAttribute('data-theme', 'dark');
    else document.documentElement.removeAttribute('data-theme');
}

document.getElementById('btn-login').onclick = () => auth.signInWithPopup(googleProvider);
document.getElementById('btn-logout').onclick = () => auth.signOut();
document.getElementById('fabBtn').onclick = () => openModal('addExpenseModal');
document.getElementById('enableDetailsBtn').onchange = (e) => document.getElementById('itemDetails').style.display = e.target.checked ? 'block' : 'none';

async function createNewTripDirectly(name, uid) {
    const t = { 
        id: 'trip_'+Date.now(), name, startDate: new Date().toISOString().slice(0,10), endDate: new Date().toISOString().slice(0,10), 
        budget: 0, expenses: [], folderId: null, order: Date.now(), members: [uid], inviteCode: Math.random().toString(36).substring(2, 8).toUpperCase()
    };
    await db.collection("trips").doc(t.id).set(t);
    await db.collection("users").doc(uid).set({ activeTripId: t.id }, { merge: true });
}

function renderSelectOptions(selectId, dataObj, defaultVal, isArray=false) {
    const sel = document.getElementById(selectId);
    if(!sel) return;
    sel.innerHTML = '';
    if(isArray) {
        dataObj.forEach(item => sel.innerHTML += `<option value="${item}">${item}</option>`);
    } else {
        sel.innerHTML = `<option value="TWD">台幣 (TWD)</option>`;
        Object.keys(dataObj).forEach(k => { if(k!=='TWD') sel.innerHTML += `<option value="${k}">${k} (${dataObj[k]})</option>`; });
    }
    sel.value = defaultVal;
}