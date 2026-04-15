import pandas as pd
import folium
import os
import random

# 1. VERİ OKUMA VE ÖLÇEKLENDİRME
koordinat_listesi = []
try:
    df = pd.read_csv('Book.csv', sep=';', decimal=',')
    df.columns = df.columns.str.strip()
    df = df.dropna(subset=['ENLEM', 'BOYLAM'])
    
    for i, row in df.iterrows():
        try:
            lat, lng = float(row['ENLEM']), float(row['BOYLAM'])
            if not (40.0 <= lat <= 42.0 and 27.0 <= lng <= 30.0): continue
            
            nufus = int(row['NÜFUS']) if pd.notnull(row['NÜFUS']) else 0
            
            if nufus > 60000: guc, base = "25 MVA", 18500
            elif nufus > 45000: guc, base = "15 MVA", 12000
            elif nufus > 30000: guc, base = "10 MVA", 8500
            elif nufus > 15000: guc, base = "5 MVA", 4200
            else: guc, base = "2.5 MVA", 1800
            
            c_phi = round(random.uniform(0.85, 0.98), 2)
                
            koordinat_listesi.append({
                'id': i, 'mahalle': str(row['SEMT MAHALLE']).strip().upper(),
                'nufus': nufus, 'guc': guc, 'base_kw': base, 'lat': lat, 'lng': lng,
                'cos_phi': c_phi, 'wear': 0 
            })
        except: continue
except Exception as e:
    print(f"Hata: {e}"); exit()

koordinat_listesi = sorted(koordinat_listesi, key=lambda x: x['nufus'], reverse=True)

# 2. HARİTA AYARLARI
m = folium.Map(location=[41.0082, 28.9784], zoom_start=11, tiles="cartodbpositron", zoom_control=False)

# 3. CSS
ui_styles = """
<style>
    body { margin: 0; padding: 0; overflow: hidden; background: #f0f0f0; }
    #side-panel { position: absolute; top: 0; left: 0; width: 330px; height: 100%; background: #f8f9fa; z-index: 1000; overflow-y: auto; border-right: 3px solid #2c3e50; font-family: sans-serif; box-shadow: 2px 0 10px rgba(0,0,0,0.1); }
    #top-bar { position: absolute; top: 0; left: 330px; right: 300px; height: 75px; background: rgba(28, 40, 51, 0.98); z-index: 999; display: flex; align-items: center; justify-content: space-around; font-family: sans-serif; color: white; border-bottom: 3px solid #e67e22; backdrop-filter: blur(10px); }
    #field-terminal { position: absolute; top: 0; right: 0; width: 300px; height: 100%; background: #1e272e; z-index: 1000; border-left: 4px solid #2ecc71; font-family: sans-serif; color: white; display: flex; flex-direction: column; }
    .item { padding: 12px; border-bottom: 1px solid #ddd; background: white; cursor: pointer; transition: 0.2s; }
    .item:hover { background: #f1f2f6; }
    .task-box { background: #2f3542; padding: 12px; border-radius: 6px; margin-bottom: 10px; border-left: 5px solid #2ecc71; animation: slideIn 0.3s; position: relative; }
    .checklist { list-style: none; padding: 10px 0; margin: 0; font-size: 11px; }
    .checklist li { padding: 4px 0; display: flex; align-items: center; color: #ecf0f1; transition: 0.3s; }
    .tech-item.locked { opacity: 0.3; pointer-events: none; }
    .checklist input { margin-right: 8px; cursor: pointer; }
    .finish-btn { width:100%; margin-top:10px; border:none; padding:8px; font-weight:bold; border-radius:4px; color:black; cursor:not-allowed; opacity: 0.4; transition: 0.3s; }
    .finish-btn.ready { background:#2ecc71; cursor:pointer; opacity: 1; }
    .reaktif-risk { color: #ff4757; font-weight: bold; animation: blinker 1s linear infinite; font-size: 10px; }
    #sim-clock { position: absolute; top: 85px; right: 315px; z-index: 1001; background: #2c3e50; color: #f1c40f; padding: 8px 15px; font-weight: bold; border-radius: 5px; font-family: monospace; font-size: 16px; border: 2px solid #e67e22; }
    .travel-bar { width: 100%; background: #444; height: 8px; border-radius: 4px; margin: 10px 0; overflow: hidden; display: none; }
    .travel-fill { height: 100%; background: #e67e22; width: 0%; transition: linear; }
    @keyframes blinker { 50% { opacity: 0; } }
    @keyframes slideIn { from { transform: translateX(30px); opacity: 0; } to { transform: translateX(0); opacity: 1; } }
    .popup-content { font-family: sans-serif; min-width: 180px; }
</style>
"""

# 4. SOL PANEL
side_panel_html = '<div style="padding: 15px; background: #2c3e50; color: white; position: sticky; top: 0; z-index: 11;"><b>⚡ İSTANBUL GÜÇ TRAFİĞİ</b><br><small>OPERATÖR KONTROL MERKEZİ</small></div>'
side_panel_html += '<div id="trafo-list-wrapper">'
for t in koordinat_listesi:
    nufus = t['nufus']
    color = 'red' if nufus > 60000 else 'orange' if nufus > 45000 else 'gold' if nufus > 30000 else 'yellow' if nufus > 15000 else 'green'
    hex_c = '#c0392b' if nufus > 60000 else '#d35400' if nufus > 45000 else '#f39c12' if nufus > 30000 else '#f1c40f' if nufus > 15000 else '#27ae60'
    label = 'GÜÇ KRİTİK' if nufus > 60000 else 'NORMALİN ÜSTÜNDE' if nufus > 45000 else 'NORMAL' if nufus > 30000 else 'HAFİF YÜK' if nufus > 15000 else 'DÜŞÜK YÜK'
    
    risk_html = f'<div id="risk-{t["id"]}" class="reaktif-risk" style="display:none;">⚠️ YÜKSEK ARIZA RİSKİ</div>'
    
    side_panel_html += f'''
    <div class="item" data-lat="{t['lat']}" data-lng="{t['lng']}" data-base="{t['base_kw']}" data-wear="0" onclick="focusTrafo({t['lat']}, {t['lng']})">
        <span id="dot-{t['id']}" style="height: 10px; width: 10px; background:{color}; border-radius:50%; display:inline-block;"></span>
        <b style="font-size: 12px;">{t['mahalle']}</b><br>
        <small>Yük: <span class="load-val" data-id="{t['id']}">{t['base_kw']}</span> kW | cos φ: <span class="cos-val" data-id="{t['id']}">{t['cos_phi']}</span></small><br>
        <div id="stat-{t['id']}" class="status-label" style="font-size: 10px; font-weight: bold; color:{hex_c};">DURUM: {label}</div>
        {risk_html}
        <button id="btn-{t['id']}" onclick="event.stopPropagation(); sendTask({t['id']}, '{t['mahalle']}', '{t['guc']}', {t['lat']}, {t['lng']})" style="width:100%; margin-top:5px; background:#e67e22; color:white; border:none; border-radius:3px; cursor:pointer; padding:6px; font-weight:bold;">EKİP SEVK ET</button>
    </div>'''
side_panel_html += '</div>'

# 5. JAVASCRIPT
js_logic = """
<script>
let simHour = 12;
let completedRepairs = 0; // MÜDAHALE SAYACI EKLENDİ
const loadProfile = [0.4, 0.3, 0.25, 0.3, 0.4, 0.6, 0.8, 0.9, 0.85, 0.8, 0.75, 0.8, 0.85, 0.85, 0.85, 0.95, 1.1, 1.3, 1.4, 1.35, 1.2, 0.9, 0.7, 0.5];

const TECH_LEVELS = [
    {label: 'GÜÇ KRİTİK', color: 'red', hex: '#c0392b'},
    {label: 'NORMALİN ÜSTÜNDE', color: 'orange', hex: '#d35400'},
    {label: 'NORMAL', color: 'gold', hex: '#f39c12'},
    {label: 'HAFİF YÜK', color: 'yellow', hex: '#f1c40f'},
    {label: 'DÜŞÜK YÜK', color: 'green', hex: '#27ae60'}
];

function getMap() {
    if(window.MAP_REF) return window.MAP_REF;
    for(let key in window) {
        if(key.startsWith('map_') && window[key] && window[key].flyTo) {
            window.MAP_REF = window[key];
            return window[key];
        }
    }
    return null;
}

function resortList() {
    const wrapper = document.getElementById('trafo-list-wrapper');
    const items = Array.from(wrapper.getElementsByClassName('item'));
    items.sort((a, b) => {
        const statA = a.querySelector('.status-label').innerText;
        const statB = b.querySelector('.status-label').innerText;
        let pA = statA.includes('DEVRE DIŞI') ? 2 : (statA.includes('KRİTİK') ? 1 : 0);
        let pB = statB.includes('DEVRE DIŞI') ? 2 : (statB.includes('KRİTİK') ? 1 : 0);
        return pB - pA;
    });
    items.forEach(it => wrapper.appendChild(it));
}

function updateLotoStatus(id, lotoCheck) {
    const container = document.getElementById('task-' + id);
    const steps = container.querySelectorAll('.tech-item');
    
    // LOTO TİKLENDİĞİNDE ENERJİYİ KES (SİYAH NOKTA)
    if(lotoCheck.checked) {
        const stat = document.getElementById('stat-'+id);
        const dot = document.getElementById('dot-'+id);
        const loadEl = document.querySelector('.load-val[data-id="'+id+'"]');
        const item = loadEl.closest('.item');
        
        stat.innerHTML = "DURUM: TRAFO DEVRE DIŞI (İSG)"; stat.style.color = "black";
        dot.style.background = "black"; loadEl.innerText = "0.0";
        updateMarkerColor(id, "black", parseFloat(item.dataset.lat), parseFloat(item.dataset.lng));
        
        const popLoad = document.getElementById('pop-load-' + id);
        const popStat = document.getElementById('pop-stat-' + id);
        if(popLoad) popLoad.innerText = "0.0";
        if(popStat) popStat.innerText = "TRAFO DEVRE DIŞI (İSG)";
        
        resortList();
    }

    steps.forEach(li => {
        if(lotoCheck.checked) {
            li.classList.remove('locked');
            li.querySelector('input').disabled = false;
        } else {
            li.classList.add('locked');
            li.querySelector('input').disabled = true;
            li.querySelector('input').checked = false;
        }
    });
    validateTask(id);
}

function validateTask(id) {
    const container = document.getElementById('task-' + id);
    const checks = container.querySelectorAll('input[type="checkbox"]');
    const btn = container.querySelector('.finish-btn');
    const allChecked = Array.from(checks).every(c => c.checked);
    if(allChecked) {
        btn.disabled = false;
        btn.classList.add('ready');
    } else {
        btn.disabled = true;
        btn.classList.remove('ready');
    }
}

function sendTask(id, name, cap, lat, lng) {
    document.getElementById('empty-msg').style.display = 'none';
    const btn = document.getElementById('btn-'+id);
    btn.innerText = "EKİP YOLDA..."; btn.disabled = true; btn.style.background = "#2c3e50";
    
    const taskBox = document.createElement('div');
    taskBox.id = "task-" + id; taskBox.className = "task-box";
    const travelTime = Math.floor(Math.random() * 4000) + 3000;
    
    taskBox.innerHTML = `
        <b>${name}</b><br>
        <div id="travel-ui-${id}">
            <small style="color:#e67e22">Ekip trafikte ilerliyor...</small>
            <div class="travel-bar" style="display:block"><div id="fill-${id}" class="travel-fill"></div></div>
        </div>
        <div id="checklist-ui-${id}" style="display:none">
            <ul class="checklist">
                <li style="border-bottom: 1px solid #555; padding-bottom: 5px; margin-bottom: 5px;">
                    <input type="checkbox" onchange="updateLotoStatus(${id}, this)"><b>[İSG] LOTO / Enerji İzolasyonu</b>
                </li>
                <li class="tech-item locked"><input disabled type="checkbox" onchange="validateTask(${id})">Arıza Onarım / Hat Kazanımı</li>
                <li class="tech-item locked"><input disabled type="checkbox" onchange="validateTask(${id})">Yük Dengeleme (Faz Aktarımı)</li>
                <li class="tech-item locked"><input disabled type="checkbox" onchange="validateTask(${id})">Kompanzasyon Onarımı</li>
                <li class="tech-item locked"><input disabled type="checkbox" onchange="validateTask(${id})">Kademe Ayarı / Yağ Kontrolü</li>
            </ul>
            <button disabled class="finish-btn" onclick="finishTask(${id}, ${lat}, ${lng})">İŞİ TAMAMLA</button>
        </div>`;
    
    document.getElementById('task-area').prepend(taskBox);
    setTimeout(() => { document.getElementById('fill-'+id).style.width = "100%"; document.getElementById('fill-'+id).style.transition = "width "+(travelTime/1000)+"s linear"; }, 100);
    
    setTimeout(() => {
        document.getElementById('travel-ui-'+id).style.display = 'none';
        document.getElementById('checklist-ui-'+id).style.display = 'block';
        btn.innerText = "MÜDAHALE SÜRÜYOR";
    }, travelTime);
    
    updateDashboard();
}

function updateMarkerColor(trafoId, newColor, lat, lng) {
    const map = getMap();
    if(!map) return;
    map.eachLayer(function(layer) {
        if (layer._latlng && Math.abs(layer._latlng.lat - lat) < 0.0001 && Math.abs(layer._latlng.lng - lng) < 0.0001) {
            if(layer.setStyle) layer.setStyle({ fillColor: newColor, color: newColor });
        }
    });
}

function finishTask(id, lat, lng) {
    document.getElementById('task-'+id).remove();
    if(document.querySelectorAll('.task-box').length == 0) document.getElementById('empty-msg').style.display = 'block';
    
    const stat = document.getElementById('stat-'+id);
    const dot = document.getElementById('dot-'+id);
    const loadEl = document.querySelector('.load-val[data-id="'+id+'"]');
    const item = loadEl.closest('.item');
    
    item.dataset.wear = "0"; 
    let riskEl = item.querySelector('.reaktif-risk');
    if(riskEl) riskEl.style.display = 'none';
    
    // LOTO SONRASI VEYA ARIZA SONRASI BİR ALT RİSK GRUBUNA (DENGELENMİŞ YÜKE) DÖNÜŞ
    const currTxt = stat.innerText.includes("DEVRE DIŞI") ? "GÜÇ KRİTİK" : stat.innerText.replace('DURUM: ', '');
    let idx = TECH_LEVELS.findIndex(l => l.label === currTxt);
    let nextIdx = Math.min(idx + 1, TECH_LEVELS.length - 1);
    let next = TECH_LEVELS[nextIdx];
    
    stat.innerHTML = "DURUM: " + next.label; stat.style.color = next.hex; dot.style.background = next.color;
    
    // Yükü hesaplanan çarpanla değil, baz gücün dengelenmiş (daha düşük) haliyle başlatır
    let baseVal = parseFloat(item.dataset.base);
    loadEl.innerText = (baseVal * 0.75).toFixed(1); 

    const btn = document.getElementById('btn-'+id);
    btn.innerText = "EKİP SEVK ET"; btn.disabled = false; btn.style.background = "#e67e22";
    document.querySelector('.cos-val[data-id="'+id+'"]').innerText = "0.98";
    
    updateMarkerColor(id, dot.style.background, lat, lng);
    resortList();
    
    completedRepairs++; // SAYACI ARTIR
    updateDashboard();
}

setInterval(() => {
    simHour = (simHour + 1) % 24;
    document.getElementById('sim-time').innerText = simHour.toString().padStart(2, '0') + ":00";
    const multiplier = loadProfile[simHour];
    
    const brightness = (simHour > 20 || simHour < 6) ? 0.75 : 1.0;
    document.querySelector('.leaflet-container').style.filter = `brightness(${brightness})`;

    let needsSort = false;
    document.querySelectorAll('.load-val').forEach(el => {
        const id = el.getAttribute('data-id');
        const stat = document.getElementById('stat-'+id);
        const dot = document.getElementById('dot-'+id);
        const item = el.closest('.item');

        if(!stat.innerText.includes("DEVRE DIŞI")) {
            let baseVal = parseFloat(item.dataset.base);
            let currentLoad = (baseVal * multiplier + (Math.random() * 40 - 20)).toFixed(1);
            el.innerText = Math.max(0, currentLoad);
            
            // Yıpranma Artışı
            if(stat.innerText.includes("KRİTİK")) {
                item.dataset.wear = parseInt(item.dataset.wear) + 5;
                if(parseInt(item.dataset.wear) > 40) {
                    let riskEl = item.querySelector('.reaktif-risk');
                    if(riskEl) riskEl.style.display = 'block';
                }
            }
            
            // Arıza Olasılığı 1/3 Oranında Artırıldı (0.000166)
            let failProb = 0.000166 + (parseInt(item.dataset.wear) / 4000);
            if(Math.random() < failProb) {
                stat.innerHTML = "DURUM: TRAFO DEVRE DIŞI"; stat.style.color = "black";
                dot.style.background = "black"; el.innerText = "0.0";
                updateMarkerColor(id, "black", parseFloat(item.dataset.lat), parseFloat(item.dataset.lng));
                needsSort = true;
            }
        }

        const popLoad = document.getElementById('pop-load-' + id);
        const popStat = document.getElementById('pop-stat-' + id);
        if(popLoad) popLoad.innerText = el.innerText;
        if(popStat) popStat.innerText = stat.innerText.replace('DURUM: ', '');
    });

    document.querySelectorAll('.cos-val').forEach(el => {
        const id = el.getAttribute('data-id');
        const stat = document.getElementById('stat-'+id);
        if(!stat.innerText.includes("DEVRE DIŞI")) {
            let currentCos = parseFloat(el.innerText);
            let nextCos = (currentCos + (Math.random() * 0.02 - 0.011));
            nextCos = Math.min(0.99, Math.max(0.82, nextCos)).toFixed(2);
            el.innerText = nextCos;
        } else { el.innerText = "0.00"; }
        const popCos = document.getElementById('pop-cos-' + id);
        if(popCos) popCos.innerText = el.innerText;
    });

    if(needsSort) resortList();
    updateDashboard();
}, 16000);

function focusTrafo(lat, lng) { 
    const map = getMap();
    if(map) map.flyTo([lat, lng], 16, {animate: true, duration: 1.5}); 
}

function updateDashboard() {
    let blackoutCount = 0;
    document.querySelectorAll('.status-label').forEach(el => {
        if(el.innerText.includes('DEVRE DIŞI')) blackoutCount++;
    });
    
    // --- GERÇEKÇİ İSTANBUL YÜK HESAPLAMASI (2.000 - 3.500 MW) ---
    const multiplier = loadProfile[simHour];
    let realisticCityLoadMW = 1650 + (1300 * multiplier) + (Math.random() * 40 - 20);
    realisticCityLoadMW -= (blackoutCount * 25); 
    
    document.getElementById('total-kw').innerText = Math.max(0, realisticCityLoadMW).toLocaleString('tr-TR', {minimumFractionDigits: 1, maximumFractionDigits: 1});
    document.getElementById('active-teams').innerText = document.querySelectorAll('.task-box').length;
    
    // --- YENİ SİSTEM DURUMU MANTIĞI (MÜDAHALE SAYISINA GÖRE) ---
    const sys = document.getElementById('sys-status');
    if (completedRepairs >= 6) {
        sys.innerText = "STABİL"; sys.style.color = "#2ed573";
    } else if (completedRepairs >= 3) {
        sys.innerText = "AZ RİSKLİ"; sys.style.color = "#f39c12"; // Turuncu/Sarı
    } else {
        sys.innerText = "RİSKLİ"; sys.style.color = "#ff4757"; // Kırmızı
    }
}

window.addEventListener('load', function() { resortList(); setTimeout(updateDashboard, 1500); });
</script>
"""

# 6. HARİTA BİLGİ PENCERELERİ
for t in koordinat_listesi:
    nufus = t['nufus']
    c = 'red' if nufus > 60000 else 'orange' if nufus > 45000 else 'gold' if nufus > 30000 else 'yellow' if nufus > 15000 else 'green'
    status_init = 'NORMAL' if nufus <= 30000 else 'GÜÇ KRİTİK' if nufus > 60000 else 'NORMALİN ÜSTÜNDE'
    
    popup_html = f"""
    <div class='popup-content'>
        <b style='color:#2c3e50; font-size:14px;'>⚡ {t['mahalle']}</b><br>
        <hr style='margin:5px 0;'>
        <b>Durum:</b> <span id='pop-stat-{t['id']}'>{status_init}</span><br>
        <b>Kapasite:</b> {t['guc']}<br>
        <b>Anlık Yük:</b> <span id='pop-load-{t['id']}'>{t['base_kw']}</span> kW<br>
        <b>Güç Faktörü:</b> <span id='pop-cos-{t['id']}'>{t['cos_phi']}</span> cos φ
    </div>
    """
    
    marker = folium.CircleMarker(
        [t['lat'], t['lng']], 
        radius=8, 
        color=c, 
        fill=True, 
        fill_color=c, 
        fill_opacity=0.9, 
        popup=folium.Popup(popup_html, max_width=300)
    )
    marker.add_to(m)

# 7. UI HTML BİRLEŞTİRME
full_ui = f"""
{ui_styles}
<div id="sim-clock">🕒 Saat: <span id="sim-time">12:00</span></div>
<div id="side-panel">{side_panel_html}</div>
<div id="top-bar">
    <div style="text-align: center;">
        <div style="font-size: 11px; color: #bdc3c7;">İSTANBUL TOPLAM YÜK</div>
        <div style="font-size: 20px; font-weight: bold; color: #f1c40f;"><span id="total-kw">0</span> <small>MW</small></div>
    </div>
    <div style="text-align: center;">
        <div style="font-size: 11px; color: #bdc3c7;">AKTİF SAHA EKİBİ</div>
        <div style="font-size: 20px; font-weight: bold; color: #e67e22;"><span id="active-teams">0</span> <small>BİRİM</small></div>
    </div>
    <div style="text-align: center;">
        <div style="font-size: 11px; color: #bdc3c7;">SİSTEM DURUMU</div>
        <div id="sys-status" style="font-size: 18px; font-weight: bold; color: #3498db;">ANALİZ EDİLİYOR...</div>
    </div>
</div>
<div id="field-terminal">
    <div style="padding: 15px; background: #2ecc71; color: black; font-weight: bold;">🚛 SAHA TERMİNALİ</div>
    <div id="task-area" style="padding: 15px; flex-grow: 1; overflow-y: auto;">
        <div id="empty-msg" style="text-align:center; color: #576574; margin-top: 50px;">BEKLEMEDE...</div>
    </div>
</div>

<div id="dispatch-modal" style="display:none; position:fixed; top:50%; left:50%; transform:translate(-50%, -50%); background:#ffffff; padding:25px; border-radius:12px; z-index:99999; box-shadow:0 15px 30px rgba(0,0,0,0.5); text-align:center; color:#2c3e50; border-top: 6px solid #e67e22; font-family:sans-serif; min-width: 320px;">
    <div style="font-size: 50px; margin-bottom: 10px;">🚚</div>
    <h3 style="margin:0; color:#e67e22; font-size:20px;">EKİP YÖNLENDİRİLİYOR</h3>
    <p id="modal-desc" style="font-size:14px; color:#34495e; margin:15px 0; line-height:1.5;">...</p>
    <button onclick="document.getElementById('dispatch-modal').style.display='none'" style="background:#2ecc71; color:#fff; font-weight:bold; border:none; padding:10px 30px; border-radius:5px; cursor:pointer; font-size:14px;">TAMAM</button>
</div>

{js_logic}
"""

m.get_root().html.add_child(folium.Element(full_ui))

# 8. KAYDET
path = os.path.join(os.environ['USERPROFILE'], 'Desktop', 'İstanbul Trafo Yük Ve Kompanzasyon Takibi.html')
m.save(path)
print(f"Proje Tamamlandı Ve Masaustune Eklendi Dosya: {path}")
