#include <WiFi.h>
#include <WebServer.h>         // Using the standard WebServer library
#include <Wire.h>
#include <Adafruit_Sensor.h>
#include <Adafruit_BMP280.h>
#include "DHT.h"               // Make sure DHT library is correctly installed
#include <ESPmDNS.h>           // For hostname resolution
#include <esp_chip_info.h>     // For detailed chip info

// --- Network & Server Settings ---
const char* staSSID = "Smeet";     // Your phone's hotspot name
const char* staPassword = NULL;    // Use NULL or "" for open networks

const char* mdnsName = "esp32-sensors"; // Hostname (http://esp32-sensors.local)
const int webPort = 80;
WebServer server(webPort);

// --- Pins ---
#define LED_BUILTIN 2          // Standard ESP32 built-in LED pin

// --- Sensor Settings ---
#define SDA_PIN 21
#define SCL_PIN 22
#define BMP280_ADDRESS 0x76
#define SEALEVELPRESSURE_HPA (1013.25) // Default Sea Level Pressure
Adafruit_BMP280 bmp;

#define DPIN 4
#define DTYPE DHT11
DHT dht(DPIN, DTYPE);

// --- Alert Thresholds (Customize these values) ---
const float TEMP_HIGH_LIMIT = 35.0;
const float TEMP_LOW_LIMIT = 15.0;
const float HUM_HIGH_LIMIT = 70.0;
const float HUM_LOW_LIMIT = 30.0;
const float PRESS_HIGH_LIMIT = 1030.0;
const float PRESS_LOW_LIMIT = 980.0;

// --- Global Variables ---
float currentSeaLevelPressure = SEALEVELPRESSURE_HPA; // Variable for adjustable SLP
bool ledState = LOW; // Track LED state

// --- Function Declarations ---
void handleMainPage();
void handleBarGraphPage();
void handleGaugesPage();
void handleStatusPage();
void handleSettingsPage();    // *** NEW: Handler for "/settings" ***
void handleSetSettings();   // *** NEW: Handler for saving settings ***
void handleLedControl();    // *** NEW: Handler for LED control ***
void handleSensorJson();
void handleNotFound();
void sendHTMLHeader(const char* title, const char* activeNav, bool includeCharts, bool includeGauges);
void sendHTMLFooter();
void connectToWiFi();
String getSensorStatusString(float value, bool& alertHigh, bool& alertLow, float highLimit, float lowLimit); // Helper for alerts
String getSensorStatusString(float value); // Overload for sensors without limits

// --- Setup ---
void setup() {
    Serial.begin(115200);
    Serial.println("\n\nESP32 Enhanced Data Acquisition System - WiFi STA");

    pinMode(LED_BUILTIN, OUTPUT); // Configure built-in LED pin
    digitalWrite(LED_BUILTIN, ledState); // Set initial LED state

    connectToWiFi(); // Connect to Hotspot

    Wire.begin(SDA_PIN, SCL_PIN); // Init I2C

    // Init BMP280
    if (!bmp.begin(BMP280_ADDRESS)) { Serial.println("!!! BMP280 Init Failed!"); }
    else { Serial.println("BMP280 Initialized."); bmp.setSampling(Adafruit_BMP280::MODE_FORCED, Adafruit_BMP280::SAMPLING_X2, Adafruit_BMP280::SAMPLING_X16, Adafruit_BMP280::FILTER_X16); }

    // Init DHT
    dht.begin(); Serial.println("DHT Initialized.");

    // --- Define Web Server Routes ---
    server.on("/", HTTP_GET, handleMainPage);
    server.on("/bargraph", HTTP_GET, handleBarGraphPage);
    server.on("/gauges", HTTP_GET, handleGaugesPage);
    server.on("/status", HTTP_GET, handleStatusPage);
    server.on("/settings", HTTP_GET, handleSettingsPage);     // *** NEW Route ***
    server.on("/setsettings", HTTP_POST, handleSetSettings); // *** NEW Route (using POST) ***
    server.on("/led", HTTP_GET, handleLedControl);       // *** NEW Route ***
    server.on("/data.json", HTTP_GET, handleSensorJson);
    server.onNotFound(handleNotFound);

    // Start mDNS
    if (MDNS.begin(mdnsName)) { Serial.print("mDNS started: http://"); Serial.print(mdnsName); Serial.println(".local"); MDNS.addService("http", "tcp", 80); }
    else { Serial.println("!!! Error starting mDNS!"); }

    server.begin(); // Start Web Server
    Serial.println("Web server started.");
}

// --- Main Loop ---
void loop() {
    server.handleClient();
}

// --- WiFi Connection Helper ---
void connectToWiFi() {
    Serial.print("Connecting to WiFi: "); Serial.println(staSSID); WiFi.mode(WIFI_STA); WiFi.begin(staSSID, staPassword); unsigned long startTime = millis(); while (WiFi.status() != WL_CONNECTED) { delay(500); Serial.print("."); if (millis() - startTime > 30000) { Serial.println("\n!!! Failed to connect to WiFi."); ESP.restart(); return; } } Serial.println("\nWiFi connected!"); Serial.print("IP address: "); Serial.println(WiFi.localIP());
}

// --- Route Handlers ---

// Main Dashboard Page (Line Graphs) - MODIFIED FOR 1-SECOND UPDATES
void handleMainPage() {
    Serial.println("Serving main page / (1s update)");
    sendHTMLHeader("Dashboard", "Dashboard", true, false); // Title, ActiveNav, IncludeCharts, No Gauges

    // HTML Structure remains the same
    String content = R"html(
    <div class="row row-cols-1 row-cols-md-2 row-cols-xl-3 g-4" id="dashboard-cards">
        <div class="col"><div class="card h-100 shadow-sm border-light anim-fade-in" data-status="dht_temp"><div class="card-body"><h5 class="card-title"><i class="bi bi-thermometer-half text-danger"></i> DHT Temp <span class="alert-icon float-end"></span></h5><p class="card-text display-4 text-center mb-0"><span class="sensor-value" id="dhtTempValue">--</span><small class="h4">&deg;C</small></p><p class="text-muted text-center small mb-2">Min: <span id="dhtTempMin">--</span>&deg;C / Max: <span id='dhtTempMax'>--</span>&deg;C</p><div class="chart-container-sm"><canvas id="dhtTempChart"></canvas></div><div class="sensor-status small text-muted mt-2">Status: <span id="dht_temp_status"></span></div></div><div class="card-footer text-muted small">DHT11 Sensor</div></div></div>
        <div class="col"><div class="card h-100 shadow-sm border-light anim-fade-in" style="animation-delay: 0.05s;" data-status="dht_hum"><div class="card-body"><h5 class="card-title"><i class="bi bi-droplet-fill text-primary"></i> DHT Humidity <span class="alert-icon float-end"></span></h5><p class="card-text display-4 text-center mb-0"><span class="sensor-value" id="dhtHumValue">--</span><small class="h4">%</small></p><p class="text-muted text-center small mb-2">Min: <span id="dhtHumMin">--</span>% / Max: <span id='dhtHumMax'>--</span>%</p><div class="chart-container-sm"><canvas id="dhtHumChart"></canvas></div><div class="sensor-status small text-muted mt-2">Status: <span id="dht_hum_status"></span></div></div><div class="card-footer text-muted small">DHT11 Sensor</div></div></div>
        <div class="col"><div class="card h-100 shadow-sm border-light anim-fade-in" style="animation-delay: 0.1s;" data-status="bmp_temp"><div class="card-body"><h5 class="card-title"><i class="bi bi-thermometer text-warning"></i> BMP Temp <span class="alert-icon float-end"></span></h5><p class="card-text display-4 text-center mb-0"><span class="sensor-value" id="bmpTempValue">--</span><small class="h4">&deg;C</small></p><p class="text-muted text-center small mb-2">Min: <span id="bmpTempMin">--</span>&deg;C / Max: <span id='bmpTempMax'>--</span>&deg;C</p><div class="chart-container-sm"><canvas id="bmpTempChart"></canvas></div><div class="sensor-status small text-muted mt-2">Status: <span id="bmp_temp_status"></span></div></div><div class="card-footer text-muted small">BMP280 Sensor</div></div></div>
        <div class="col"><div class="card h-100 shadow-sm border-light anim-fade-in" style="animation-delay: 0.15s;" data-status="bmp_press"><div class="card-body"><h5 class="card-title"><i class="bi bi-speedometer2 text-info"></i> BMP Pressure <span class="alert-icon float-end"></span></h5><p class="card-text display-4 text-center mb-0"><span class="sensor-value" id="bmpPressValue">--</span><small class="h5"> hPa</small></p><p class="text-muted text-center small mb-2">Min: <span id="bmpPressMin">--</span>hPa / Max: <span id='bmpPressMax'>--</span>hPa</p><div class="chart-container-sm"><canvas id="bmpPressChart"></canvas></div><div class="sensor-status small text-muted mt-2">Status: <span id="bmp_press_status"></span></div></div><div class="card-footer text-muted small">BMP280 Sensor</div></div></div>
        <div class="col"><div class="card h-100 shadow-sm border-light anim-fade-in" style="animation-delay: 0.2s;" data-status="bmp_alt"><div class="card-body"><h5 class="card-title"><i class="bi bi-bar-chart-fill text-success"></i> BMP Altitude</h5><p class="card-text display-4 text-center mb-0"><span class="sensor-value" id="bmpAltValue">--</span><small class="h5"> m</small></p><p class="text-muted text-center small mb-2">(Approx. @ <span id="slpDisplay"></span> hPa)</p><div class="chart-container-sm"><canvas id="bmpAltChart"></canvas></div><div class="sensor-status small text-muted mt-2">Status: <span id="bmp_alt_status"></span></div></div><div class="card-footer text-muted small">BMP280 Sensor</div></div></div>
        <div class="col"><div class="card h-100 shadow-sm border-light anim-fade-in" style="animation-delay: 0.25s;"><div class="card-body"><h5 class="card-title"><i class="bi bi-cpu-fill text-secondary"></i> System Info</h5><ul class="list-group list-group-flush"><li class="list-group-item d-flex justify-content-between align-items-center small py-1 px-0">Uptime:<span class="badge bg-secondary rounded-pill" id="sysUptime">-- s</span></li><li class="list-group-item d-flex justify-content-between align-items-center small py-1 px-0">Free Heap:<span class="badge bg-secondary rounded-pill" id="sysHeap">-- B</span></li><li class="list-group-item d-flex justify-content-between align-items-center small py-1 px-0">WiFi RSSI:<span class="badge bg-secondary rounded-pill" id="sysRSSI">-- dBm</span></li><li class="list-group-item d-flex justify-content-between align-items-center small py-1 px-0">Last Update:<span class="badge bg-light text-dark rounded-pill" id="lastUpdate">--</span></li></ul></div><div class="card-footer text-muted small">ESP32 System</div></div></div>
    </div>

    <script> // --- JavaScript for Main Dashboard (MODIFIED FOR 1-SECOND UPDATES) ---
        const UPDATE_INTERVAL_MS = 1000;      // *** Update interval set to 1 second ***
        const MAX_DATA_POINTS = 30;           // Keep 30 seconds of history
        let dhtTempChart, dhtHumChart, bmpTempChart, bmpPressChart, bmpAltChart;
        let minMax = {dhtTemp:{min:null,max:null},dhtHum:{min:null,max:null},bmpTemp:{min:null,max:null},bmpPress:{min:null,max:null}};

        // *** Modified to disable animation for faster updates ***
        function createLineChartConfigBase() {
             return { responsive: true, maintainAspectRatio: false, animation: { duration: 0 }, /* Disabled animation */ scales: { y: { beginAtZero: false, grid: { color: 'rgba(200,200,200,0.1)' }, ticks: { font: { size: 9 }, color: '#aaa' } }, x: { display: false } }, plugins: { legend: { display: false }, tooltip: { enabled: true, mode: 'index', intersect: false, bodySpacing: 4, displayColors: false } } };
        }
        function createLineChartDataset(label, color1, color2) { return [{ label: label, data: [], borderColor: color1, backgroundColor: color2, borderWidth: 1.5, pointRadius: 0, fill: true, tension: 0.3 }]; }

        function initLineCharts(){ try{ if(typeof Chart==='undefined'){console.error('Chart object not found!'); return;} const opts = createLineChartConfigBase(); const dsConf = createLineChartDataset; const getCtx = (id) => document.getElementById(id)?.getContext('2d'); const dhtTempCtx = getCtx('dhtTempChart'); if (dhtTempCtx) dhtTempChart = new Chart(dhtTempCtx, { type: 'line', data: { labels:[], datasets: dsConf('DHT Temp','rgba(255, 99, 132, 0.8)','rgba(255, 99, 132, 0.1)') }, options: opts }); const dhtHumCtx = getCtx('dhtHumChart'); if (dhtHumCtx) dhtHumChart = new Chart(dhtHumCtx, { type: 'line', data: { labels:[], datasets: dsConf('DHT Hum','rgba(153, 102, 255, 0.8)','rgba(153, 102, 255, 0.1)') }, options: opts }); const bmpTempCtx = getCtx('bmpTempChart'); if (bmpTempCtx) bmpTempChart = new Chart(bmpTempCtx, { type: 'line', data: { labels:[], datasets: dsConf('BMP Temp','rgba(255, 159, 64, 0.8)','rgba(255, 159, 64, 0.1)') }, options: opts }); const bmpPressCtx = getCtx('bmpPressChart'); if (bmpPressCtx) bmpPressChart = new Chart(bmpPressCtx, { type: 'line', data: { labels:[], datasets: dsConf('BMP Press','rgba(54, 162, 235, 0.8)','rgba(54, 162, 235, 0.1)') }, options: opts }); const bmpAltCtx = getCtx('bmpAltChart'); if (bmpAltCtx) bmpAltChart = new Chart(bmpAltCtx, { type: 'line', data: { labels:[], datasets: dsConf('BMP Alt','rgba(75, 192, 192, 0.8)','rgba(75, 192, 192, 0.1)') }, options: opts }); console.log("Line charts init attempted."); } catch(e){ console.error("Line chart init failed:", e); } }

        // Update line chart data using shift (rolling window)
        function updateLineChartData(chart, label, newData) { if(!chart||typeof newData!=='number'||isNaN(newData))return; const d=chart.data; d.labels.push(label); d.datasets[0].data.push(newData); while(d.labels.length > MAX_DATA_POINTS) { d.labels.shift(); d.datasets[0].data.shift(); } chart.update('none'); }

        function updateMinMax(key, value) { /* ... same as before ... */ if(typeof value!=='number'||isNaN(value))return;const t=minMax[key];if(t.min===null||value<t.min)t.min=value;if(t.max===null||value>t.max)t.max=value;}
        function displayMinMax() { /* ... same as before ... */ const f=(v,d)=>(v!==null?v.toFixed(d):'--'); document.getElementById('dhtTempMin').textContent=f(minMax.dhtTemp.min,1);document.getElementById('dhtTempMax').textContent=f(minMax.dhtTemp.max,1);document.getElementById('dhtHumMin').textContent=f(minMax.dhtHum.min,1);document.getElementById('dhtHumMax').textContent=f(minMax.dhtHum.max,1);document.getElementById('bmpTempMin').textContent=f(minMax.bmpTemp.min,2);document.getElementById('bmpTempMax').textContent=f(minMax.bmpTemp.max,2);document.getElementById('bmpPressMin').textContent=f(minMax.bmpPress.min,2);document.getElementById('bmpPressMax').textContent=f(minMax.bmpPress.max,2);}
        function applyAlertStatus(cardElement, alertHigh, alertLow, statusText) { /* ... same as before ... */ if(!cardElement)return;const i=cardElement.querySelector('.alert-icon');const s=cardElement.querySelector('.sensor-status span');const v=cardElement.querySelector('.sensor-value');cardElement.classList.remove('border-danger','border-warning','border-info');if(i)i.innerHTML='';if(statusText!=='OK'){cardElement.classList.add('border-danger');if(i)i.innerHTML='<i class="bi bi-x-octagon-fill text-danger"></i>';if(v)v.classList.add('text-danger');}else if(alertHigh||alertLow){cardElement.classList.add('border-warning');if(i)i.innerHTML='<i class="bi bi-exclamation-triangle-fill text-warning"></i>';if(v)v.classList.remove('text-danger');}else{if(v)v.classList.remove('text-danger');}if(s)s.textContent=statusText+(alertHigh?' (High)':'')+(alertLow?' (Low)':'');}

        // Main data update function
        async function updateDashboardData() {
            const elements = { dhtTempV: document.getElementById('dhtTempValue'), dhtHumV: document.getElementById('dhtHumValue'), bmpTempV: document.getElementById('bmpTempValue'), bmpPressV: document.getElementById('bmpPressValue'), bmpAltV: document.getElementById('bmpAltValue'), sysUptime: document.getElementById('sysUptime'), sysRSSI: document.getElementById('sysRSSI'), sysHeap: document.getElementById('sysHeap'), lastUpdate: document.getElementById('lastUpdate'), slpDisplay: document.getElementById('slpDisplay') };
            const cards = { dht_temp: document.querySelector('[data-status="dht_temp"]'), dht_hum: document.querySelector('[data-status="dht_hum"]'), bmp_temp: document.querySelector('[data-status="bmp_temp"]'), bmp_press: document.querySelector('[data-status="bmp_press"]'), bmp_alt: document.querySelector('[data-status="bmp_alt"]') };
            const statusEls = { dht_temp_status: document.getElementById('dht_temp_status'), dht_hum_status: document.getElementById('dht_hum_status'), bmp_temp_status: document.getElementById('bmp_temp_status'), bmp_press_status: document.getElementById('bmp_press_status'), bmp_alt_status: document.getElementById('bmp_alt_status') };

             // Show Loading (only needed once, really)
             if(!elements.dhtTempV.textContent || elements.dhtTempV.textContent === '--' || elements.dhtTempV.textContent === '...') {
                Object.values(elements).forEach(el => { if(el && el.classList.contains('sensor-value')) el.textContent = '...'; });
             }

            try { const response = await fetch('/data.json'); if (!response.ok) throw new Error(`HTTP error ${response.status}`); const data = await response.json(); if (!data || typeof data.timestamp === 'undefined') throw new Error("Invalid data format");
                const timeLabel = new Date(data.timestamp * 1000).toLocaleTimeString();
                const setText = (el, val, digits, suffix='') => { if(el){ if(val!==null&&typeof val==='number'&&!isNaN(val)){el.textContent=val.toFixed(digits)+suffix;} else{el.textContent='--';}}};

                // Update text values and apply alert styles
                setText(elements.dhtTempV, data.dht_temp, 1); applyAlertStatus(cards.dht_temp, data.dht_temp_alert_high, data.dht_temp_alert_low, data.dht_status);
                setText(elements.dhtHumV, data.dht_hum, 1); applyAlertStatus(cards.dht_hum, data.dht_hum_alert_high, data.dht_hum_alert_low, data.dht_status);
                setText(elements.bmpTempV, data.bmp_temp, 2); applyAlertStatus(cards.bmp_temp, data.bmp_temp_alert_high, data.bmp_temp_alert_low, data.bmp_status);
                setText(elements.bmpPressV, data.bmp_press, 2); applyAlertStatus(cards.bmp_press, data.bmp_press_alert_high, data.bmp_press_alert_low, data.bmp_status);
                setText(elements.bmpAltV, data.bmp_alt, 2); applyAlertStatus(cards.bmp_alt, false, false, data.bmp_status);
                setText(elements.sysUptime, data.uptime, 0, ' s'); setText(elements.sysRSSI, data.rssi, 0, ' dBm'); setText(elements.sysHeap, data.heap, 0, ' B');
                if (elements.lastUpdate) elements.lastUpdate.textContent = timeLabel; if (elements.slpDisplay) elements.slpDisplay.textContent = data.slp ? data.slp.toFixed(2) : 'N/A';

                // Update Min/Max only if value is valid
                if (data.dht_status === 'OK') { updateMinMax('dhtTemp', data.dht_temp); updateMinMax('dhtHum', data.dht_hum); } if (data.bmp_status === 'OK') { updateMinMax('bmpTemp', data.bmp_temp); updateMinMax('bmpPress', data.bmp_press); } displayMinMax();

                // Update charts only if value is valid (use NaN for errors)
                updateLineChartData(dhtTempChart, timeLabel, data.dht_status === 'OK' ? data.dht_temp : NaN);
                updateLineChartData(dhtHumChart, timeLabel, data.dht_status === 'OK' ? data.dht_hum : NaN);
                updateLineChartData(bmpTempChart, timeLabel, data.bmp_status === 'OK' ? data.bmp_temp : NaN);
                updateLineChartData(bmpPressChart, timeLabel, data.bmp_status === 'OK' ? data.bmp_press : NaN);
                updateLineChartData(bmpAltChart, timeLabel, data.bmp_status === 'OK' ? data.bmp_alt : NaN);

                document.querySelectorAll('.col .card.border-danger').forEach(el => { if(!el.querySelector('.sensor-status span')?.textContent.includes('Error')) el.classList.remove('border-danger'); });

            } catch (error) { console.error("Error updating dashboard:", error); Object.values(elements).forEach(el => { if (el && el.classList.contains('sensor-value')) el.textContent = 'ERR'; }); Object.values(cards).forEach(el => { if(el) el.classList.add('border-danger'); }); Object.values(statusEls).forEach(el => {if(el) el.textContent = 'Update Error'; }); }
        }
        document.addEventListener('DOMContentLoaded', () => { initLineCharts(); updateDashboardData(); setInterval(updateDashboardData, UPDATE_INTERVAL_MS); });
    </script>
    )html";
    server.sendContent(content);
    sendHTMLFooter();
}

// Handler for the Bar Graph Page
// Handler for the Enhanced Bar Graph Page (Bar + Lines)
// Handler for the Enhanced Bar Graph Page (Persistent Session History)
// Handler for the Enhanced Bar Graph Page (DEBUG - sessionStorage Disabled)
void handleBarGraphPage() {
    Serial.println("Serving enhanced bar graph page /bargraph (DEBUG - sessionStorage disabled)");
    sendHTMLHeader("Graphs", "Bar Graph", true, false); // Title, ActiveNav, IncludeCharts, No Gauges

    // HTML: Bar Chart Card + Row for Line Chart Cards (Same as before)
    String content = R"html(
    <div class="card shadow-sm border-light mb-4">
        <div class="card-header">Current Sensor Readings (Bar Graph - Updates ~1s)</div>
        <div class="card-body"><div class="bar-chart-container-lg"><canvas id="sensorBarChart"></canvas></div></div>
        <div class="card-footer text-muted small text-end">Last Update: <span id="lastUpdateBar">--</span></div>
    </div>
    <h3 class="mt-4 mb-3">Sensor History (Line Graphs - Last 30 Readings, Updates ~1s)</h3>
    <div class="row g-4">
       <div class="col-12"><div class="card h-100 shadow-sm border-light anim-fade-in"><div class="card-body"><h5 class="card-title small"><i class="bi bi-thermometer-half text-danger"></i> DHT Temp History</h5><div class="chart-container-med"><canvas id="lineChartDhtTemp"></canvas></div></div></div></div>
       <div class="col-12"><div class="card h-100 shadow-sm border-light anim-fade-in" style="animation-delay:0.05s;"><div class="card-body"><h5 class="card-title small"><i class="bi bi-droplet-fill text-primary"></i> DHT Humidity History</h5><div class="chart-container-sm"><canvas id="lineChartDhtHum"></canvas></div></div></div></div>
       <div class="col-12"><div class="card h-100 shadow-sm border-light anim-fade-in" style="animation-delay:0.1s;"><div class="card-body"><h5 class="card-title small"><i class="bi bi-thermometer text-warning"></i> BMP Temp History</h5><div class="chart-container-med"><canvas id="lineChartBmpTemp"></canvas></div></div></div></div>
       <div class="col-12"><div class="card h-100 shadow-sm border-light anim-fade-in" style="animation-delay:0.15s;"><div class="card-body"><h5 class="card-title small"><i class="bi bi-speedometer2 text-info"></i> BMP Pressure History</h5><div class="chart-container-sm"><canvas id="lineChartBmpPress"></canvas></div></div></div></div>
       <div class="col-12"><div class="card h-100 shadow-sm border-light anim-fade-in" style="animation-delay:0.2s;"><div class="card-body"><h5 class="card-title small"><i class="bi bi-bar-chart-fill text-success"></i> BMP Altitude History</h5><div class="chart-container-sm"><canvas id="lineChartBmpAlt"></canvas></div></div></div></div>
    </div>
    <div class="text-center text-muted small mt-3">Line Graph Last Update: <span id="lastUpdateLine">--</span></div>
    <style>.chart-container-med{position:relative;height:150px;width:100%;margin-top:5px;}.chart-container-sm{position:relative;height:120px;width:100%;margin-top:5px;}</style>

    <script> // --- JS with sessionStorage DISABLED for Debugging ---
        const UPDATE_INTERVAL_MS = 1000;
        const MAX_LINE_DATA_POINTS = 30;
        let sensorBarChart;
        let lineChartDhtTemp, lineChartDhtHum, lineChartBmpTemp, lineChartBmpPress, lineChartBmpAlt;

        function initBarChart() { /* ... same as before ... */ const c=document.getElementById('sensorBarChart');if(!c){console.error("Bar Canvas not found!");return null;}const x=c.getContext('2d');if(!x){console.error("Bar 2D Ctx failed!");return null;}try{return new Chart(x,{type:'bar',data:{labels:['DHT Temp (°C)','DHT Hum (%)','BMP Temp (°C)','BMP Press (hPa)','BMP Alt (m)'],datasets:[{label:'Value',data:[0,0,0,0,0],backgroundColor:['rgba(255,99,132,.7)','rgba(153,102,255,.7)','rgba(255,159,64,.7)','rgba(54,162,235,.7)','rgba(75,192,192,.7)'],borderColor:['#fff'],borderWidth:1}]},options:{responsive:!0,maintainAspectRatio:!1,indexAxis:'y',scales:{x:{beginAtZero:!0,grid:{color:'rgba(0,0,0,.05)'}},y:{grid:{display:!1}}},plugins:{legend:{display:!1},title:{display:!0,text:'Current Sensor Readings',font:{size:16}}},animation:{duration:150}}});}catch(e){console.error("Bar chart init failed:",e);return null;} }
        function createLineChartConfigBase() { /* ... same as before ... */ return{responsive:!0,maintainAspectRatio:!1,animation:{duration:0},scales:{y:{beginAtZero:!1,grid:{color:'rgba(200,200,200,0.1)'},ticks:{font:{size:9},color:'#aaa'}},x:{display:false}},plugins:{legend:{display:!1},tooltip:{enabled:!0,mode:'index',intersect:!1,bodySpacing:4,displayColors:!1}}};}
        function createLineChartDataset(label, color1, color2) { /* ... same as before ... */ return[{label:label,data:[],borderColor:color1,backgroundColor:color2,borderWidth:1.5,pointRadius:0,fill:true,tension:0.3}];}

        // *** MODIFIED: Initialize line charts WITHOUT loading from sessionStorage ***
        function initLineCharts() {
            const opts = createLineChartConfigBase();
            const getCtx = (id) => document.getElementById(id)?.getContext('2d');
             try {
                if(typeof Chart === 'undefined'){ console.error('Chart object not found!'); return; }
                // Initialize directly with empty data
                const ctx1 = getCtx('lineChartDhtTemp'); if(ctx1) lineChartDhtTemp = new Chart(ctx1, { type:'line', data:{ labels:[], datasets: createLineChartDataset('DHT T', 'rgba(255,99,132,.8)', 'rgba(255,99,132,.1)') }, options: opts });
                const ctx2 = getCtx('lineChartDhtHum'); if(ctx2) lineChartDhtHum = new Chart(ctx2, { type:'line', data:{ labels:[], datasets: createLineChartDataset('DHT H', 'rgba(153,102,255,.8)', 'rgba(153,102,255,.1)') }, options: opts });
                const ctx3 = getCtx('lineChartBmpTemp'); if(ctx3) lineChartBmpTemp = new Chart(ctx3, { type:'line', data:{ labels:[], datasets: createLineChartDataset('BMP T', 'rgba(255,159,64,.8)', 'rgba(255,159,64,.1)') }, options: opts });
                const ctx4 = getCtx('lineChartBmpPress'); if(ctx4) lineChartBmpPress = new Chart(ctx4, { type:'line', data:{ labels:[], datasets: createLineChartDataset('BMP P', 'rgba(54,162,235,.8)', 'rgba(54,162,235,.1)') }, options: opts });
                const ctx5 = getCtx('lineChartBmpAlt'); if(ctx5) lineChartBmpAlt = new Chart(ctx5, { type:'line', data:{ labels:[], datasets: createLineChartDataset('BMP A', 'rgba(75,192,192,.8)', 'rgba(75,192,192,.1)') }, options: opts });
                console.log("Line charts init attempted (sessionStorage loading disabled).");
            } catch(e) { console.error("Line chart init failed:", e); }
        }

        function updateBarChart(data) { /* ... same as before ... */ if(!sensorBarChart)return;const dt=data.dht_status==='OK'?data.dht_temp:0;const dh=data.dht_status==='OK'?data.dht_hum:0;const bt=data.bmp_status==='OK'?data.bmp_temp:0;const bp=data.bmp_status==='OK'?data.bmp_press:0;const ba=data.bmp_status==='OK'?data.bmp_alt:0;sensorBarChart.data.datasets[0].data=[dt,dh,bt,bp,ba];sensorBarChart.update();}

        // *** MODIFIED: Updates line charts WITHOUT saving state to sessionStorage ***
        function updateSingleLineChart(chart, storageKey, label, newData) { // storageKey param ignored now
             if (!chart) return;
             const value = (newData !== null && typeof newData === 'number' && !isNaN(newData)) ? newData : NaN;
             const data = chart.data;
             data.labels.push(label);
             data.datasets[0].data.push(value);
             while (data.labels.length > MAX_LINE_DATA_POINTS) {
                 data.labels.shift();
                 data.datasets[0].data.shift();
             }
             chart.update('none');
             // --- Saving to sessionStorage is commented out ---
             /*
             try {
                const dataToSave = { labels: data.labels, datasets: [{ data: data.datasets[0].data }] };
                sessionStorage.setItem(storageKey, JSON.stringify(dataToSave));
             } catch (e) { console.error(`Error saving data to sessionStorage for ${storageKey}:`, e); }
             */
        }

        async function updateAllChartsData() { /* ... same fetch/update logic as before ... */ try{const r=await fetch('/data.json');if(!r.ok)throw new Error(`HTTP ${r.status}`);const d=await r.json();if(!d||typeof d.timestamp==='undefined')throw new Error("Invalid data format");const timeLabel=new Date(d.timestamp*1000).toLocaleTimeString();updateBarChart(d);updateSingleLineChart(lineChartDhtTemp,'graphPage_lineData_dhtTemp',timeLabel,d.dht_status==='OK'?d.dht_temp:NaN);updateSingleLineChart(lineChartDhtHum,'graphPage_lineData_dhtHum',timeLabel,d.dht_status==='OK'?d.dht_hum:NaN);updateSingleLineChart(lineChartBmpTemp,'graphPage_lineData_bmpTemp',timeLabel,d.bmp_status==='OK'?d.bmp_temp:NaN);updateSingleLineChart(lineChartBmpPress,'graphPage_lineData_bmpPress',timeLabel,d.bmp_status==='OK'?d.bmp_press:NaN);updateSingleLineChart(lineChartBmpAlt,'graphPage_lineData_bmpAlt',timeLabel,d.bmp_status==='OK'?d.bmp_alt:NaN);const uB=document.getElementById('lastUpdateBar');if(uB)uB.textContent=timeLabel;const uL=document.getElementById('lastUpdateLine');if(uL)uL.textContent=timeLabel;}catch(e){console.error("Error updating charts data:",e);const uB=document.getElementById('lastUpdateBar');if(uB)uB.textContent="Error";const uL=document.getElementById('lastUpdateLine');if(uL)uL.textContent="Error";}}

        // --- Initial Setup ---
        document.addEventListener('DOMContentLoaded', () => {
             console.log("DOM Loaded. Initializing charts for /bargraph (sessionStorage disabled)...");
             if(typeof Chart==='undefined'){console.error("Chart object not defined!");return;}
             sensorBarChart = initBarChart();
             initLineCharts(); // Initialize line charts without loading
             updateAllChartsData();
             setInterval(updateAllChartsData, UPDATE_INTERVAL_MS);
        });
    </script>
    )html";
    server.sendContent(content);
    sendHTMLFooter();
}

// Handler for Gauges Page
// *** CORRECTED Handler for Gauges Page ***
void handleGaugesPage() {
    Serial.println("Serving gauges page /gauges");
    sendHTMLHeader("Gauges", "Gauges", false, true); // Title, ActiveNav, No Charts, IncludeGauges

    // HTML structure using canvas-gauges elements (No changes needed here)
    String content = R"html(
     <div class="row row-cols-1 row-cols-md-2 row-cols-xl-3 g-4">
        <div class="col">
            <div class="card h-100 shadow-sm border-light text-center anim-fade-in">
                <div class="card-body">
                     <h5 class="card-title mb-3"><i class="bi bi-thermometer-half text-danger"></i> DHT Temp</h5>
                     <canvas data-type="radial-gauge" id="gauge-dht-temp"
                        data-width="220" data-height="220" data-units="°C"
                        data-min-value="-10" data-max-value="50"
                        data-major-ticks="-10,0,10,20,30,40,50" data-minor-ticks="5"
                        data-stroke-ticks="true" data-highlights='[ {"from": -10, "to": 0, "color": "rgba(0,0,255,.3)"}, {"from": 30, "to": 50, "color": "rgba(255,0,0,.3)"} ]'
                        data-color-plate="#fff" data-border-shadow-width="0" data-borders="false"
                        data-needle-type="arrow" data-needle-width="2"
                        data-needle-circle-size="7" data-needle-circle-outer="true" data-needle-circle-inner="false"
                        data-animation-duration="500" data-animation-rule="linear"
                        data-value="0" data-value-box="true" data-value-text-shadow="false" data-value-dec="1"
                     ></canvas>
                 </div>
                 <div class="card-footer text-muted small">DHT11 Sensor <span id="dht_temp_status"></span></div>
             </div>
         </div>
         <div class="col">
             <div class="card h-100 shadow-sm border-light text-center anim-fade-in" style="animation-delay: 0.05s;">
                 <div class="card-body">
                      <h5 class="card-title mb-3"><i class="bi bi-droplet-fill text-primary"></i> DHT Humidity</h5>
                      <canvas data-type="radial-gauge" id="gauge-dht-hum"
                         data-width="220" data-height="220" data-units="%"
                         data-min-value="0" data-max-value="100"
                         data-major-ticks="0,10,20,30,40,50,60,70,80,90,100" data-minor-ticks="5"
                         data-stroke-ticks="true" data-highlights='[ {"from": 80, "to": 100, "color": "rgba(0,100,255,.25)"} ]'
                         data-color-plate="#fff" data-border-shadow-width="0" data-borders="false"
                         data-needle-type="arrow" data-needle-width="2"
                         data-needle-circle-size="7" data-needle-circle-outer="true" data-needle-circle-inner="false"
                         data-animation-duration="500" data-animation-rule="linear"
                         data-value="0" data-value-box="true" data-value-text-shadow="false" data-value-dec="1"
                     ></canvas>
                  </div>
                  <div class="card-footer text-muted small">DHT11 Sensor <span id="dht_hum_status"></span></div>
              </div>
          </div>
           <div class="col">
              <div class="card h-100 shadow-sm border-light text-center anim-fade-in" style="animation-delay: 0.1s;">
                  <div class="card-body">
                       <h5 class="card-title mb-3"><i class="bi bi-thermometer text-warning"></i> BMP Temp</h5>
                       <canvas data-type="radial-gauge" id="gauge-bmp-temp"
                          data-width="220" data-height="220" data-units="°C"
                          data-min-value="-10" data-max-value="60"
                          data-major-ticks="-10,0,10,20,30,40,50,60" data-minor-ticks="5"
                          data-stroke-ticks="true" data-highlights='[ {"from": -10, "to": 0, "color": "rgba(0,0,255,.3)"}, {"from": 40, "to": 60, "color": "rgba(255,100,0,.3)"} ]'
                          data-color-plate="#fff" data-border-shadow-width="0" data-borders="false"
                          data-needle-type="arrow" data-needle-width="2"
                          data-needle-circle-size="7" data-needle-circle-outer="true" data-needle-circle-inner="false"
                          data-animation-duration="500" data-animation-rule="linear"
                          data-value="0" data-value-box="true" data-value-text-shadow="false" data-value-dec="2"
                      ></canvas>
                   </div>
                   <div class="card-footer text-muted small">BMP280 Sensor <span id="bmp_temp_status"></span></div>
               </div>
           </div>
            <div class="col">
               <div class="card h-100 shadow-sm border-light text-center anim-fade-in" style="animation-delay: 0.15s;">
                   <div class="card-body">
                        <h5 class="card-title mb-3"><i class="bi bi-speedometer2 text-info"></i> BMP Pressure</h5>
                        <canvas data-type="radial-gauge" id="gauge-bmp-press"
                           data-width="220" data-height="220" data-units="hPa"
                           data-min-value="900" data-max-value="1100"
                           data-major-ticks="900,950,1000,1050,1100" data-minor-ticks="10"
                           data-stroke-ticks="true" data-highlights='[ {"from": 900, "to": 960, "color": "rgba(200,200,0,.25)"}, {"from": 1040, "to": 1100, "color": "rgba(200,200,0,.25)"} ]'
                           data-color-plate="#fff" data-border-shadow-width="0" data-borders="false"
                           data-needle-type="arrow" data-needle-width="2"
                           data-needle-circle-size="7" data-needle-circle-outer="true" data-needle-circle-inner="false"
                           data-animation-duration="500" data-animation-rule="linear"
                           data-value="1000" data-value-box="true" data-value-text-shadow="false" data-value-dec="2"
                       ></canvas>
                    </div>
                    <div class="card-footer text-muted small">BMP280 Sensor <span id="bmp_press_status"></span></div>
                </div>
            </div>
     </div>
     <div class="text-center text-muted small mt-3">Last Update: <span id="lastUpdate">--</span></div>

     <script> // --- CORRECTED JavaScript for Gauges Page ---
        const GAUGE_UPDATE_INTERVAL_MS = 5000;

        // *** CORRECTED Function to update gauge values ***
        function updateGauge(gaugeId, newValue) {
            // Get the gauge INSTANCE using the library's collection
            // Use optional chaining (?.) in case document.gauges isn't ready yet
            const gaugeInstance = document.gauges?.get(gaugeId);

            if (gaugeInstance) { // Check if the gauge instance exists
                 let finalValue;
                 if (newValue !== null && typeof newValue === 'number' && !isNaN(newValue)) {
                     // Use the gauge's own options to format the number BEFORE assigning
                     const decimals = gaugeInstance.options.valueDec || 0;
                     // Ensure the value is parsed as float after toFixed() which returns a string
                     finalValue = parseFloat(newValue.toFixed(decimals));
                 } else {
                     // Set to the minimum value defined in the gauge's options on error/null
                     finalValue = gaugeInstance.options.minValue;
                 }
                 // Update the gauge instance's value property.
                 // The library will handle redrawing the canvas automatically.
                 gaugeInstance.value = finalValue;
            } else {
                 // This might happen briefly on first load, or if ID is wrong
                 // console.warn(`Gauge instance with ID ${gaugeId} not found or not ready.`);
            }
        }

        // Function to update the status text below gauges (same as before)
        function updateStatusText(elId, status, alertHigh, alertLow) {
             const el = document.getElementById(elId);
             if (el) {
                 let text = status;
                 let cssClass = 'text-muted'; // Default class
                 if (status === 'OK') {
                     if (alertHigh) { text += ' (High)'; cssClass = 'text-warning fw-bold'; }
                     else if (alertLow) { text += ' (Low)'; cssClass = 'text-warning fw-bold'; }
                     else { cssClass = 'text-success'; } // OK and within limits
                 } else {
                     cssClass = 'text-danger fw-bold'; // Error state
                 }
                 el.textContent = text;
                 el.className = `status-text ${cssClass}`; // Use a base class if needed
             }
        }

        // Async function to fetch data and trigger updates (same as before)
        async function updateGaugesData() {
            const updateEl = document.getElementById('lastUpdate');
             try {
                 const response = await fetch('/data.json');
                 if (!response.ok) throw new Error(`HTTP ${response.status}`);
                 const data = await response.json();
                 if (!data || typeof data.timestamp === 'undefined') throw new Error("Invalid data format");

                 // Update each gauge using the corrected function
                 updateGauge('gauge-dht-temp', data.dht_temp);
                 updateStatusText('dht_temp_status', data.dht_status, data.dht_temp_alert_high, data.dht_temp_alert_low);
                 updateGauge('gauge-dht-hum', data.dht_hum);
                 updateStatusText('dht_hum_status', data.dht_status, data.dht_hum_alert_high, data.dht_hum_alert_low);
                 updateGauge('gauge-bmp-temp', data.bmp_temp);
                 updateStatusText('bmp_temp_status', data.bmp_status, data.bmp_temp_alert_high, data.bmp_temp_alert_low);
                 updateGauge('gauge-bmp-press', data.bmp_press);
                 updateStatusText('bmp_press_status', data.bmp_status, data.bmp_press_alert_high, data.bmp_press_alert_low);
                 // Altitude gauge could be added here if desired

                 const timeLabel = new Date(data.timestamp * 1000).toLocaleTimeString();
                 if(updateEl) updateEl.textContent = timeLabel;

             } catch (error) {
                 console.error("Error updating gauges:", error);
                 if(updateEl) updateEl.textContent = "Error";
                 // Reset gauges on error
                 updateGauge('gauge-dht-temp', null); updateGauge('gauge-dht-hum', null);
                 updateGauge('gauge-bmp-temp', null); updateGauge('gauge-bmp-press', null);
                 // Update status text on error
                 updateStatusText('dht_temp_status','Error',false,false); updateStatusText('dht_hum_status','Error',false,false);
                 updateStatusText('bmp_temp_status','Error',false,false); updateStatusText('bmp_press_status','Error',false,false);
             }
         }

         // Event listener for DOMContentLoaded (same as before)
         document.addEventListener('DOMContentLoaded', () => {
             console.log("DOM Loaded, initializing gauges update...");
             // Gauge library initializes from data-* attributes automatically.
             // Add small delay before first fetch to ensure rendering.
             setTimeout(() => {
                 updateGaugesData(); // Initial data fetch
                 setInterval(updateGaugesData, GAUGE_UPDATE_INTERVAL_MS); // Set interval
             }, 500);
         });
     </script>
    )html";
    server.sendContent(content);
    sendHTMLFooter();
}

// Handler for System Status Page
void handleStatusPage() {
    // Added LED Control Button
    Serial.println("Serving status page /status");
    sendHTMLHeader("System Status", "Status", false, false);

    esp_chip_info_t chip_info; esp_chip_info(&chip_info);

    String content = R"html(
    <div class="row">
        <div class="col-md-8">
            <div class="card shadow-sm border-light mb-4">
                <div class="card-header">ESP32 System Information</div>
                <div class="card-body">
                    <ul class="list-group list-group-flush">
                        <li class='list-group-item d-flex justify-content-between align-items-center'>Chip Model:<span class='badge bg-info rounded-pill'>)html"; content += String(CONFIG_IDF_TARGET); content += R"html(</span></li>
                        <li class='list-group-item d-flex justify-content-between align-items-center'>Chip Revision:<span class='badge bg-info rounded-pill'>)html"; content += String(chip_info.revision); content += R"html(</span></li>
                        <li class='list-group-item d-flex justify-content-between align-items-center'>CPU Cores:<span class='badge bg-info rounded-pill'>)html"; content += String(chip_info.cores); content += R"html(</span></li>
                        <li class='list-group-item d-flex justify-content-between align-items-center'>Flash Size:<span class='badge bg-info rounded-pill'>)html"; content += String(ESP.getFlashChipSize()/(1024*1024)); content += R"html( MB</span></li>
                        <li class='list-group-item d-flex justify-content-between align-items-center'>MAC Address:<span class='badge bg-secondary rounded-pill'>)html"; content += WiFi.macAddress(); content += R"html(</span></li>
                        <li class='list-group-item d-flex justify-content-between align-items-center'>Current IP:<span class='badge bg-success rounded-pill'>)html"; content += WiFi.localIP().toString(); content += R"html(</span></li>
                        <li class='list-group-item d-flex justify-content-between align-items-center'>WiFi SSID:<span class='badge bg-primary rounded-pill'>)html"; content += String(staSSID); content += R"html(</span></li>
                        <li class='list-group-item d-flex justify-content-between align-items-center'>Signal Strength (RSSI):<span class='badge bg-warning text-dark rounded-pill'>)html"; content += String(WiFi.RSSI()); content += R"html( dBm</span></li>
                        <li class='list-group-item d-flex justify-content-between align-items-center'>Free Heap Memory:<span class='badge bg-secondary rounded-pill'>)html"; content += String(ESP.getFreeHeap()); content += R"html( Bytes</span></li>
                        <li class='list-group-item d-flex justify-content-between align-items-center'>Sketch Size:<span class='badge bg-secondary rounded-pill'>)html"; content += String(ESP.getSketchSize()); content += R"html( Bytes</span></li>
                        <li class='list-group-item d-flex justify-content-between align-items-center'>Free Sketch Space:<span class='badge bg-secondary rounded-pill'>)html"; content += String(ESP.getFreeSketchSpace()); content += R"html( Bytes</span></li>
                        <li class='list-group-item d-flex justify-content-between align-items-center'>System Uptime:<span class='badge bg-secondary rounded-pill' id='statusUptime'>--</span></li>
                    </ul>
                </div>
            </div>
        </div>
        <div class="col-md-4">
             <div class="card shadow-sm border-light mb-4">
                 <div class="card-header">Controls</div>
                 <div class="card-body text-center">
                      <h5 class="card-title mb-3"><i class="bi bi-lightbulb-fill"></i> Built-in LED</h5>
                      <button class="btn btn-lg btn-primary" id="ledButton" onclick="toggleLed()">Toggle LED</button>
                      <p class="small text-muted mt-2">Current State: <span id="ledStateText">Unknown</span></p>
                 </div>
             </div>
             <div class="card shadow-sm border-light">
                 <div class="card-header">Actions</div>
                 <div class="card-body text-center">
                     <button class="btn btn-danger" onclick="rebootEsp()">Reboot ESP32</button>
                 </div>
             </div>
        </div>
    </div>
    <script>
        let currentLedState = undefined; // undefined, 0 (off), 1 (on)

        function updateUptime(){ /* same as before */ const el=document.getElementById('statusUptime'); if(el){ let secsText = el.getAttribute('data-seconds') || Math.floor(Date.now()/1000 - pageLoadTime); let s = parseInt(secsText); s++; el.setAttribute('data-seconds', s); let h=Math.floor(s/3600); let m=Math.floor((s%3600)/60); let sec=s%60; el.textContent=`${h}h ${m}m ${sec}s`;}}

        function updateLedButton(state) {
            const btn = document.getElementById('ledButton');
            const txt = document.getElementById('ledStateText');
            if (!btn || !txt) return;
            currentLedState = state;
            if (state === 1) {
                btn.classList.remove('btn-primary');
                btn.classList.add('btn-warning');
                txt.textContent = 'ON';
            } else {
                 btn.classList.remove('btn-warning');
                 btn.classList.add('btn-primary');
                 txt.textContent = 'OFF';
            }
        }

        async function toggleLed() {
            const newState = (currentLedState === 1) ? 0 : 1;
            try {
                const response = await fetch(`/led?state=${newState}`);
                if (response.ok) {
                   const data = await response.json();
                   if (data.success) {
                       updateLedButton(data.ledState);
                   } else {
                       console.error('LED toggle failed on server:', data.message);
                       alert('Failed to toggle LED');
                   }
                } else {
                    console.error('LED toggle request failed:', response.status);
                    alert('Failed to send LED command');
                }
            } catch (error) {
                console.error('Error toggling LED:', error);
                alert('Error sending LED command');
            }
        }

       async function fetchInitialLedState() {
           // Optional: could add endpoint to just get LED state
           // For now, assume it starts OFF based on ESP32 code default
           updateLedButton(0); // Assume starts OFF
       }

       function rebootEsp() {
           if (confirm('Are you sure you want to reboot the ESP32?')) {
               // Send a reboot command - simple way is just navigating
               // We could create a dedicated /reboot endpoint, but this works too
               alert('Sending reboot command... connection will be lost.');
               // Create a dummy request or navigate - this might not always work perfectly
               fetch('/reboot-dummy-request').catch(()=>{}); // Send async, ignore result
               // Disable button to prevent multi-clicks
               event.target.disabled = true;
               event.target.textContent = 'Rebooting...';
           }
       }
       const pageLoadTime = Date.now()/1000; // Base time for uptime JS calc
       updateUptime(); setInterval(updateUptime, 1000);
       fetchInitialLedState();
    </script>
    )html";

    server.sendContent(content);
    sendHTMLFooter();
}

// *** NEW Handler for LED Control ***
void handleLedControl() {
    Serial.print("Handling /led request...");
    String stateArg = server.arg("state"); // Get "state" parameter from URL (?state=0 or ?state=1)
    bool success = false;

    if (stateArg == "1") {
        ledState = HIGH;
        digitalWrite(LED_BUILTIN, ledState);
        Serial.println(" Turning LED ON");
        success = true;
    } else if (stateArg == "0") {
        ledState = LOW;
        digitalWrite(LED_BUILTIN, ledState);
        Serial.println(" Turning LED OFF");
        success = true;
    } else {
        Serial.println(" Invalid state argument.");
    }

    // Send JSON response confirming action
    String json = "{\"success\":";
    json += success ? "true" : "false";
    json += ",\"ledState\":";
    json += ledState ? "1" : "0";
    json += success ? "}" : ",\"message\":\"Invalid state parameter\"}";

    server.send(200, "application/json", json);
}


// *** NEW Handler for Settings Page (GET) ***
void handleSettingsPage() {
     Serial.println("Serving settings page /settings");
     sendHTMLHeader("Settings", "Settings", false, false);

     String content = R"html(
     <div class="card shadow-sm border-light">
        <div class="card-header">Configuration Settings</div>
        <div class="card-body">
            <form method="POST" action="/setsettings">
                <div class="mb-3">
                    <label for="slpInput" class="form-label">Sea Level Pressure (hPa)</label>
                    <input type="number" step="0.01" class="form-control" id="slpInput" name="slp" value=")html";
                    content += String(currentSeaLevelPressure, 2); // Pre-fill with current value
                    content += R"html(" required>
                    <div class="form-text">Used for altitude calculation. Find local value for accuracy.</div>
                </div>
                <button type="submit" class="btn btn-primary">Save Settings</button>
            </form>
            <div id="settings-feedback" class="mt-3"></div>
        </div>
     </div>
     <script>
        // Optional: Add JS to handle form submission with Fetch API for feedback
        // For simplicity now, we rely on standard form POST and page reload/redirect
     </script>
     )html";

     server.sendContent(content);
     sendHTMLFooter();
}

// *** NEW Handler for Saving Settings (POST) ***
void handleSetSettings() {
    Serial.println("Handling POST /setsettings");
    bool success = false;
    String feedbackMessage = "";

    if (server.hasArg("slp")) { // Check if the 'slp' parameter was sent
        String slpArg = server.arg("slp");
        float newSLP = slpArg.toFloat();
        // Basic validation - ensure it's a reasonable pressure value
        if (newSLP > 800.0 && newSLP < 1200.0) {
             currentSeaLevelPressure = newSLP;
             Serial.print("Sea Level Pressure updated to: "); Serial.println(currentSeaLevelPressure);
             success = true;
             feedbackMessage = "Settings saved successfully!";
        } else {
            Serial.println("!!! Invalid SLP value received.");
            feedbackMessage = "Error: Invalid Sea Level Pressure value entered.";
        }
    } else {
         Serial.println("!!! No SLP argument received.");
         feedbackMessage = "Error: Missing Sea Level Pressure data.";
    }

    // Send response page with feedback (or could redirect)
     sendHTMLHeader("Settings Saved", "Settings", false, false);
     String content = "<div class='card shadow-sm border-light'><div class='card-header'>Settings Update</div><div class='card-body'>";
     if (success) {
        content += "<div class='alert alert-success' role='alert'>" + feedbackMessage + "</div>";
     } else {
        content += "<div class='alert alert-danger' role='alert'>" + feedbackMessage + "</div>";
     }
     content += "<p><a href='/settings' class='btn btn-secondary mt-2'>Back to Settings</a> ";
     content += "<a href='/' class='btn btn-primary mt-2'>Go to Dashboard</a></p>";
     content += "</div></div>";
     server.sendContent(content);
     sendHTMLFooter();

    // Alternative: Redirect back to settings page (might lose feedback message easily)
    // server.sendHeader("Location", "/settings", true);
    // server.send(302, "text/plain", "");
}


// Handler for Sensor Data Endpoint (/data.json)
void handleSensorJson() {
    // *** Modified to include status and alert flags ***
    float bmp_t = NAN, bmp_p = NAN, bmp_a = NAN;
    float dht_h = NAN, dht_t = NAN;
    bool bmp_read_ok = false;
    String bmp_status = "Error";
    String dht_status = "Error";

    // --- Read BMP280 ---
    if (bmp.takeForcedMeasurement()) {
        bmp_t = bmp.readTemperature();
        bmp_p = bmp.readPressure();
        if (!isnan(bmp_p)) {
            // *** Use the globally adjustable sea level pressure ***
            bmp_a = bmp.readAltitude(currentSeaLevelPressure);
            bmp_read_ok = true; // Mark as OK only if pressure read succeeds
            bmp_status = "OK";
        } else { bmp_a = NAN; bmp_status = "Read Error (P)"; }
        if (isnan(bmp_t)) { bmp_status = "Read Error (T)"; bmp_read_ok = false; } // Temp failure also an error

    } else { Serial.println("!!! BMP280 takeForcedMeasurement() failed!"); bmp_status = "Comms Error"; }

    // --- Read DHT11 ---
    dht_h = dht.readHumidity();
    dht_t = dht.readTemperature();
    if (!isnan(dht_h) && !isnan(dht_t)) { dht_status = "OK"; }
    else { Serial.println("!!! Failed DHT read"); dht_status = "Read Error"; }

    // --- Check Thresholds ---
    bool dht_temp_alert_high = false, dht_temp_alert_low = false;
    bool dht_hum_alert_high = false, dht_hum_alert_low = false;
    bool bmp_temp_alert_high = false, bmp_temp_alert_low = false;
    bool bmp_press_alert_high = false, bmp_press_alert_low = false;

    if (dht_status == "OK") {
        if (dht_t > TEMP_HIGH_LIMIT) dht_temp_alert_high = true;
        if (dht_t < TEMP_LOW_LIMIT) dht_temp_alert_low = true;
        if (dht_h > HUM_HIGH_LIMIT) dht_hum_alert_high = true;
        if (dht_h < HUM_LOW_LIMIT) dht_hum_alert_low = true;
    }
     if (bmp_status == "OK") { // Only check BMP thresholds if read was OK
        if (bmp_t > TEMP_HIGH_LIMIT) bmp_temp_alert_high = true;
        if (bmp_t < TEMP_LOW_LIMIT) bmp_temp_alert_low = true;
        float bmp_p_hpa = bmp_p / 100.0F; // Convert to hPa for threshold check
        if (bmp_p_hpa > PRESS_HIGH_LIMIT) bmp_press_alert_high = true;
        if (bmp_p_hpa < PRESS_LOW_LIMIT) bmp_press_alert_low = true;
    }

    // --- Create JSON response ---
    String json = "{";
    unsigned long currentMillis = millis();
    json += "\"timestamp\":" + String(currentMillis / 1000);
    json += ",\"dht_status\":\"" + dht_status + "\""; // Include status
    json += ",\"dht_temp\":" + (dht_status=="OK" ? String(dht_t,1) : "null");
    json += ",\"dht_temp_alert_high\":" + String(dht_temp_alert_high ? "true" : "false");
    json += ",\"dht_temp_alert_low\":" + String(dht_temp_alert_low ? "true" : "false");
    json += ",\"dht_hum\":" + (dht_status=="OK" ? String(dht_h,1) : "null");
    json += ",\"dht_hum_alert_high\":" + String(dht_hum_alert_high ? "true" : "false");
    json += ",\"dht_hum_alert_low\":" + String(dht_hum_alert_low ? "true" : "false");
    json += ",\"bmp_status\":\"" + bmp_status + "\""; // Include status
    json += ",\"bmp_temp\":" + (bmp_read_ok ? String(bmp_t,2) : "null");
    json += ",\"bmp_temp_alert_high\":" + String(bmp_temp_alert_high ? "true" : "false");
    json += ",\"bmp_temp_alert_low\":" + String(bmp_temp_alert_low ? "true" : "false");
    json += ",\"bmp_press\":" + (bmp_read_ok ? String(bmp_p/100.0F,2) : "null");
    json += ",\"bmp_press_alert_high\":" + String(bmp_press_alert_high ? "true" : "false");
    json += ",\"bmp_press_alert_low\":" + String(bmp_press_alert_low ? "true" : "false");
    json += ",\"bmp_alt\":" + (bmp_read_ok && !isnan(bmp_a) ? String(bmp_a,2) : "null");
    json += ",\"slp\":" + String(currentSeaLevelPressure, 2); // Include current SLP used
    json += ",\"uptime\":" + String(currentMillis / 1000);
    json += ",\"rssi\":" + String(WiFi.RSSI());
    json += ",\"heap\":" + String(ESP.getFreeHeap());
    json += "}";

    server.sendHeader("Access-Control-Allow-Origin","*");
    server.sendHeader("Cache-Control","no-cache");
    server.sendHeader("Content-Type","application/json");
    server.send(200,"application/json",json);
}


// Handler for 404 Not Found errors
void handleNotFound() {
    Serial.println("Handling 404 Not Found for URI: " + server.uri());
    sendHTMLHeader("404 Not Found", "404", false, false);
    server.sendContent("<div class='alert alert-danger' role='alert'><h2>Page Not Found</h2><p>Sorry, the requested URL (<code>" + server.uri() + "</code>) was not found.</p><p><a href='/' class='alert-link'>Return to Dashboard</a></p></div>");
    sendHTMLFooter();
}


// --- HTML Helper Functions ---

// Sends the HTML header, includes libraries, CSS, Nav
void sendHTMLHeader(const char* title, const char* activeNav, bool includeCharts, bool includeGauges) {
    // (Content mostly same as previous, ensuring Bootstrap/Icons/Fonts/JS Libs are linked via CDN)
    String header = "<!DOCTYPE html><html lang='en'><head><meta charset='UTF-8'><meta name='viewport' content='width=device-width, initial-scale=1.0'><title>";
    header += title; header += " - Low cost DAQ system Dashboard</title>";
    header += "<link href='https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css' rel='stylesheet' integrity='sha384-QWTKZyjpPEjISv5WaRU9OFeRpok6YctnYmDr5pNlyT2bRjXh0JMhjY6hW+ALEwIH' crossorigin='anonymous'>";
    header += "<link rel='stylesheet' href='https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.3/font/bootstrap-icons.min.css'>";
    header += "<link href='https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;500;600&display=swap' rel='stylesheet'>";
    if (includeCharts) header += "<script src='https://cdn.jsdelivr.net/npm/chart.js@3.9.1/dist/chart.min.js'></script>";
    if (includeGauges) header += "<script src='https://cdn.jsdelivr.net/npm/canvas-gauges@2.1.7/gauge.min.js'></script>";
    header += R"html(<style> body{font-family:'Poppins',sans-serif;background-color:#f8f9fa;} .navbar{background:linear-gradient(135deg,#0056b3,#1a73e8);} .navbar-brand{font-weight:600;} .chart-container-sm{position:relative;height:100px;width:100%;margin-top:10px;} .bar-chart-container-lg{position:relative;height:400px;width:100%;} .card-title i{margin-right:8px;font-size:1.1em;} .display-4{font-weight:300;} .small{font-size:0.85em;} .list-group-item{background-color:transparent;border:0;padding-left:0;padding-right:0;} .list-group-flush>.list-group-item:last-child{border-bottom-width:1px;} canvas{max-width:100%;} @keyframes fadeIn{from{opacity:0;transform:translateY(10px);}to{opacity:1;transform:translateY(0);}} .anim-fade-in{opacity:0;animation:fadeIn 0.6s ease-out forwards;} .alert-icon{font-size:0.9em;} .card.border-danger{border-width:3px!important;} .card.border-warning{border-width:3px!important;} </style></head><body> <nav class="navbar navbar-expand-lg navbar-dark shadow-sm mb-4"> <div class="container-fluid"> <a class="navbar-brand" href="/"><i class="bi bi-speedometer2"></i> Low cost DAQ system Dashboard</a> <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarNav" aria-controls="navbarNav" aria-expanded="false" aria-label="Toggle navigation"><span class="navbar-toggler-icon"></span></button> <div class="collapse navbar-collapse" id="navbarNav"> <ul class="navbar-nav ms-auto"> <li class="nav-item"><a class="nav-link )html"; header += (strcmp(activeNav,"Dashboard")==0?"active":""); header += R"html(" href="/"><i class="bi bi-graph-up"></i> Dashboard</a></li> <li class="nav-item"><a class="nav-link )html"; header += (strcmp(activeNav,"Bar Graph")==0?"active":""); header += R"html(" href="/bargraph"><i class="bi bi-bar-chart-line-fill"></i> Bar Graph</a></li> <li class="nav-item"><a class="nav-link )html"; header += (strcmp(activeNav,"Gauges")==0?"active":""); header += R"html(" href="/gauges"><i class="bi bi-speedometer"></i> Gauges</a></li> <li class="nav-item"><a class="nav-link )html"; header += (strcmp(activeNav,"Status")==0?"active":""); header += R"html(" href="/status"><i class="bi bi-info-circle-fill"></i> Status</a></li> <li class="nav-item"><a class="nav-link )html"; header += (strcmp(activeNav,"Settings")==0?"active":""); header += R"html(" href="/settings"><i class="bi bi-gear-fill"></i> Settings</a></li> </ul> </div> </div> </nav> <main class="container-fluid mb-5"> )html";

    server.setContentLength(CONTENT_LENGTH_UNKNOWN);
    server.send(200, "text/html", header);
}

// Sends the HTML footer
void sendHTMLFooter() {
    // Includes Bootstrap JS Bundle
    String footer = R"html( </main> <footer class="py-3 my-4 border-top text-center text-muted"> <p class="mb-0 small">&copy; 2025 ESP32 Sensor Dashboard</p> </footer> <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js" integrity="sha384-YvpcrYf0tY3lHB60NNkmXc5s9fDVZLESaAA55NDzOxhy9GkcIdslK1eN7N6jIeHz" crossorigin="anonymous"></script> </body></html> )html";
    server.sendContent(footer);
    server.sendContent(""); // Finalize chunked response
}
