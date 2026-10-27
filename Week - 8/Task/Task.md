# Task - Networking and Database Connection 

Build a rooftop water-tank monitor that reports its readings to the cloud and recovers automatically if Wi-Fi drops. You'll apply everything from this week's Activity to a new scenario.

**Scenario:**

An apartment complex's rooftop water tank needs remote monitoring from the building management office. A potentiometer stands in for a float-style level sensor. The device connects to Wi-Fi, reads the tank level, and saves it as a new row in a **Supabase** cloud database on a steady schedule, so the office can see the tank's history. If the level drops below a critical threshold, a local LED + buzzer alert must fire immediately, on its own timer separate from the cloud-reporting schedule, so building staff on-site are warned even if the network is down. Every Serial log line needs a real timestamp from the internet, not device uptime, and the device must reconnect on its own if Wi-Fi drops, without ever freezing.

**Components required:**
- ESP32-PICO development board
- 1 × potentiometer (stands in for the tank-level sensor)
- 1 × LED (low-level alert indicator)
- 1 × 220 Ω resistor
- Passive buzzer (alert tone)
- Jumper wires, breadboard

> **Schematic:**
>
> _[Schematic image placeholder: draw this circuit in [Circuit Studio](https://circuit-studio.pages.dev/), then paste your exported schematic image here (Export → Schematic image) and add your share link (Export → Share link) below it.]_
>
> <details><summary>Schematic link</summary>
>
> [Open the example schematic in Circuit Studio]()
>
> </details>
>
> **Your pin choices** (choose any suitable GPIO, following the pin rules, and use the same numbers in your schematic and your code):
>
> | Connection | Pin type needed | Your GPIO |
> |---|---|---|
> | Potentiometer wiper (outer legs → 3.3V and GND) | Analog input, **ADC1 only** | GPIO ___ |
> | Alert LED (through the 220 Ω resistor; LED cathode → GND) | Digital output | GPIO ___ |
> | Passive buzzer + (buzzer − → GND) | Digital output (`tone()`) | GPIO ___ |

**How to draw the schematic:** before building the circuit, draw it in [Circuit Studio](https://circuit-studio.pages.dev/).
- Include every component in the list above: the ESP32-PICO, the potentiometer, the alert LED with its 220 Ω resistor, and the passive buzzer.
- **Choose your own GPIO pins** and record them in the **Your pin choices** table above. Draw the same pins on the schematic and use the same numbers as named constants in your code (`potPin`, `alertLedPin`, `buzzerPin`). Connect every ground to GND.
- Follow the ESP32-PICO pin rules:
  - the potentiometer needs an **ADC1** pin (GPIO 32-36 or 39), because ADC2 stops working while Wi-Fi is on
  - the LED and buzzer can't use input-only GPIO 34-39
  - never use GPIO 1, 3, 6-11, 16 or 17
  - avoid boot-mode pins GPIO 0, 2, 5, 12 and 15
  - safe digital choices are GPIO 4, 13, 14, 18, 19, 21-23, 25-27, 32 and 33
- Export it with **Export → Schematic image** and **Export → Share link**, and paste both into the Schematic box above.

**Set up the Supabase database (one time):**
1. Create a free project at [supabase.com](https://supabase.com) (or reuse your Activity project).
2. In **SQL Editor**, run this SQL to create the `tank_levels` table and allow the device to insert rows only:
   ```sql
   create table tank_levels (
     id          bigint generated always as identity primary key,
     created_at  timestamptz not null default now(),
     level       integer,
     avg_level   real          -- only used by the extension
   );

   alter table tank_levels enable row level security;

   create policy "ESP32 can insert tank levels"
     on tank_levels for insert
     to anon
     with check (true);
   ```
3. Copy the **Project URL** and **publishable** (anon) key from **Project Settings → API Keys** into `SUPABASE_URL` and `SUPABASE_KEY`. Never use the `service_role` key on the device.
4. Check your rows in **Table Editor → tank_levels**, and include a screenshot of it with your evidence.

**Requirements:** the finished program must meet all of the following.
- Runs in **Station mode** (`WiFi.mode(WIFI_STA)`) and connects to a **2.4 GHz** Wi-Fi network using named `SSID`/`PASSWORD` constants, with a bounded blocking retry called exactly once, from `setup()`. Replace your real password, Supabase URL and key with `"YOUR_WIFI_PASSWORD"`, `"YOUR_PROJECT_ID"` and `"YOUR_SUPABASE_PUBLISHABLE_KEY"` before you submit the code.
- Syncs real time via NTP once, from `setup()`, only if the Wi-Fi connection actually succeeded.
- Reads the tank-level potentiometer no more than once every 2000 ms, non-blocking (`millis()`-gated, never `delay()`).
- `sendToSupabase()` inserts the level reading as a row in `tank_levels` (JSON `{"level":<reading>}`, HTTPS POST to `SUPABASE_URL + "/rest/v1/tank_levels"` with the `apikey` and `Authorization` headers) on its own non-blocking 15000 ms interval, only when `WiFi.status() == WL_CONNECTED` at the moment of sending. A send counts as successful only on response code `201`.
- Calls `http.end()` after every `http.begin()`/`http.POST()`, with no exceptions.
- If the level falls below `LOW_LEVEL_THRESHOLD`, a non-blocking LED + buzzer alert (`millis()`-timed, `tone()`/`noTone()`, never `digitalWrite()` on the buzzer) fires immediately. This alert must be timed separately from the 15-second cloud-send interval, and must still fire even while Wi-Fi is disconnected.
- A `maybe_reconnect()` function checks the connection on its own non-blocking interval and calls `WiFi.begin()` again (with no blocking `while`/`delay()` loop around it) whenever disconnected.
- Every Serial log line uses a real timestamp from `getTimestamp()`, never `millis()`.
- `loop()` contains no `delay()` anywhere.

**Extension (optional):**
- Also send `avg_level` in the same row, carrying a rolling average of the last 5 readings.
- Add a distinct "[RECOVERED]" log line the moment Wi-Fi reconnects after being down. Compare the current connection state against the last-known state each pass, rather than just logging "connected" on every pass while it stays up.
- Switch to **AP + Station mode** (`WIFI_AP_STA`) so on-site staff can join the device's own network (`"TankMonitor"`, 8+ character password) and see the current tank level on a web page at `http://192.168.4.1`, even when the building's Wi-Fi or internet is down. Cloud sends keep working through the station side whenever the router is reachable. `server.handleClient()` must run every pass of `loop()`.

**Testing on the real board:** while the program is running, switch off your router or phone hotspot. The tank-level readings and the low-level alert must keep working, `maybe_reconnect()` should log its reconnect attempts, and cloud sends should report as failed/offline (no new rows in `tank_levels` during that time). Switch the hotspot back on, and the device must reconnect and resume sending by itself, with no reset.

>
> Evidence: _(attach a photo or short video of your working circuit on the ESP32-PICO, plus a screenshot of the Serial Monitor output and of the `tank_levels` rows in Supabase)_

### Questions

- Why must the low-level alert be timed independently of the 15-second cloud-send interval, rather than only checked at the same time a cloud send happens?
  ```


  ```

- The device is required to keep firing the local alert even while Wi-Fi is disconnected. What in your design makes that true, and what would break it if the alert logic were placed inside `sendToSupabase()`'s success path?
  ```


  ```

- Why is `WiFi.status()` checked again immediately before sending, rather than trusting the result of `connectWiFi()` in `setup()`?
  ```


  ```

- What would a building manager actually see in the Supabase `tank_levels` table during an extended Wi-Fi outage, and why is the local LED + buzzer alert still necessary even though the cloud database exists?
  ```


  ```

- Describe, step by step, what happens in your program from the moment Wi-Fi drops to the moment it reconnects. Which functions run, on what schedule, and what does *not* happen while it's down?
  ```


  ```

- Your device runs in Station mode. Why would Access Point mode on its own **not** meet this brief? What would AP + Station mode add for the on-site building staff?
  ```


  ```

### Spot-the-Bug Worksheet

**Format:** For each round, read the snippet and write down what's wrong **before** revealing the answer.

**Round 1:**
```cpp
void loop() {
  int level = analogRead(potPin);
  WiFiClientSecure client;
  client.setInsecure();
  HTTPClient http;
  http.begin(client, String(SUPABASE_URL) + "/rest/v1/tank_levels");
  http.addHeader("Content-Type", "application/json");
  http.addHeader("apikey", SUPABASE_KEY);
  http.addHeader("Authorization", String("Bearer ") + SUPABASE_KEY);
  http.POST("{\"level\":" + String(level) + "}");
  http.end();
}
```
<details><summary>Answer</summary>There's no `WiFi.status() != WL_CONNECTED` check before building and sending the request. If the connection has dropped, this still attempts the HTTP call every pass instead of skipping it cleanly. The send must be gated on a confirmed connection first.</details>

**Round 2:**
```cpp
void loop() {
  int level = analogRead(potPin);
  sendToSupabase(level);
  delay(15000);
}
```
<details><summary>Answer</summary>`delay(15000)` freezes the entire `loop()` for 15 seconds every pass. The tank-level alert has to fire immediately and independently of the cloud send, and it can't run during that window. The 15-second cloud interval must be enforced with a non-blocking `millis()`-gated timer instead.</details>

**Round 3:**
```cpp
bool sendToSupabase(int level) {
  WiFiClientSecure client;
  client.setInsecure();
  HTTPClient http;
  http.begin(client, url);
  http.addHeader("Content-Type", "application/json");
  http.addHeader("apikey", SUPABASE_KEY);
  http.addHeader("Authorization", String("Bearer ") + SUPABASE_KEY);
  int code = http.POST("{\"level\":" + String(level) + "}");
  return code == 201;
}
```
<details><summary>Answer</summary>`http.end()` is never called, so the connection is never released, whether the request succeeds or fails. It must be called unconditionally right after `http.POST()`, before the function returns.</details>

**Round 4:**
```cpp
void triggerAlert() {
  digitalWrite(alertLedPin, HIGH);
  tone(buzzerPin, 1000);
  delay(500);
  digitalWrite(alertLedPin, LOW);
  noTone(buzzerPin);
}
```
<details><summary>Answer</summary>`triggerAlert()` uses a blocking `delay(500)`. While it runs, nothing else in `loop()` (the potentiometer read, the cloud send, the reconnect check) can execute. This needs the non-blocking `start_alert()`/`update_alert()` `millis()`-timed pattern used everywhere else in this course, not a single blocking function.</details>

**Round 5:**
```cpp
const char* SSID     = "BuildingWiFi-5G";
const char* PASSWORD = "rooftop2024";
```
<details><summary>Answer</summary>The network name ends in `-5G`, which almost always means a **5 GHz** network. The ESP32's radio only works on **2.4 GHz**, so it can't even see this network, and `connectWiFi()` times out with "Connection failed" every time, however correct the password and the rest of the code are. Fix: connect to the building's 2.4 GHz network instead (often the same name without `-5G`, or ending in `-2.4G`), or to a phone hotspot set to 2.4 GHz.</details>

**Round 6:**
```cpp
void setup() {
  connectWiFi();
  configTime(0, 0, "pool.ntp.org", "time.nist.gov");   // start the NTP sync (main + backup server)
  setenv("TZ", "AEST-10", 1);                          // local time zone: UTC+10 (AEST), no daylight saving
  tzset();
}

void loop() {
  Serial.println(getTimestamp());
}
```
<details><summary>Answer</summary>`getTimestamp()` is called from the very first pass of `loop()`, with no wait for `configTime()`'s NTP sync to complete first. The sync happens in the background over the following seconds. Every call before it finishes returns a timestamp near `1970-01-01`. A bounded wait loop confirming `time(nullptr)` exceeds a large sanity value must run in `setup()` before the clock can be trusted.</details>

**Round 7:**
```cpp
void maybe_reconnect() {
  while (WiFi.status() != WL_CONNECTED) {
    WiFi.begin(SSID, PASSWORD);
    delay(500);
  }
}
```
<details><summary>Answer</summary>This is a blocking `while`/`delay()` loop, not a non-blocking check. Called from `loop()`, it freezes the whole program (including the tank-level alert, which must keep working during an outage) for as long as reconnection takes. `maybe_reconnect()` must call `WiFi.begin()` once per check interval and return immediately, never waiting inside a loop for the result.</details>
