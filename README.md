# Motor Health Monitor (ThingSpeak dashboard)

A single-file live dashboard for the ESP32 motor predictive-maintenance project.
It reads your ThingSpeak channel and shows NORMAL / CRITICAL / OFFLINE status,
current and vibration charts, and a list of recent critical events.
No server and no build step: it is plain HTML + JavaScript.

## 1. ThingSpeak channel setup

Create a channel with these fields:

| Field | Meaning | Values |
|-------|---------|--------|
| field1 | Motor state (A) | number |
| field2 |Motor current (g) | number |
| field3 | vibration | 0 = NORMAL, 1 = CRITICAL |
| field4 | Vib_peak (g), optional | number |

Then in the channel's **Sharing** tab choose **Share channel view with everyone** (public).
This is the simplest option because the dashboard then needs no key at all.

## 2. Configure the dashboard

Open `index.html` and edit only the `CONFIG` block:

```js
CHANNEL_ID: "3561474",   // your channel ID (number shown on the channel page)
READ_API_KEY: "MNMC0W2T5Z4AKVS9",        // only needed for a PRIVATE channel
```

## 3. Security: read this before pushing to GitHub

- **Never put your WRITE API key in this repository.** Anyone could then send fake data to your channel.
  The write key belongs only in the ESP32 sketch (and do not upload that sketch with the key to GitHub either).
- A **Read API key** in a public repo is visible to everyone. It only allows reading, so the risk is low,
  but if the data is private, keep the repo private or make the channel public instead.

## 4. Publish with GitHub Pages

1. Create a new repository on GitHub, for example `motor-dashboard`.
2. Upload `index.html` and `README.md` (Add file, Upload files, Commit).
3. Go to **Settings, Pages**. Under *Build and deployment* choose **Deploy from a branch**, branch `main`, folder `/ (root)`, Save.
4. After about a minute your dashboard is live at `https://<your-username>.github.io/motor-dashboard/`.

## 5. ESP32 side: send data to ThingSpeak

ThingSpeak's free plan accepts one update every 15 seconds, so send about every 20 seconds
(not every loop). Add this to your final sketch (Part 8). Keep your key in the sketch only.

```cpp
#include <WiFi.h>
#include <HTTPClient.h>

const char* WIFI_SSID = "YOUR_WIFI";
const char* WIFI_PASS = "YOUR_PASSWORD";
const char* TS_WRITE_KEY = "YOUR_WRITE_API_KEY";   // never commit this to GitHub

unsigned long lastSend = 0;

void connectWiFi() {
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  unsigned long t0 = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - t0 < 15000) delay(250);
}

// status: 0 = NORMAL, 1 = CRITICAL
void sendToThingSpeak(float current, float vib, int status, float peak) {
  if (WiFi.status() != WL_CONNECTED) { connectWiFi(); if (WiFi.status() != WL_CONNECTED) return; }
  HTTPClient http;
  String url = "https://api.thingspeak.com/update?api_key=" + String(TS_WRITE_KEY) +
               "&field1=" + String(current, 3) +
               "&field2=" + String(vib, 4) +
               "&field3=" + String(status) +
               "&field4=" + String(peak, 4);
  http.begin(url);
  http.GET();
  http.end();
}

// in setup():  connectWiFi();
// in loop(), after you have f[] and the prediction p:
//   if (millis() - lastSend > 20000) {
//     lastSend = millis();
//     float vib = sqrt(f[2]*f[2] + f[3]*f[3] + f[4]*f[4]);
//     sendToThingSpeak(f[0], vib, (p != 0) ? 1 : 0, f[5]);
//   }
```

Note: the HTTPS request blocks for a moment, which delays the next sensor window slightly. That is fine for monitoring.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| "Setup needed" banner | Set `CHANNEL_ID` in `index.html` |
| "Channel not found or private" | Make the channel public, or set `READ_API_KEY` |
| Status always OFFLINE | ESP32 not sending, or sending slower than `OFFLINE_AFTER_SECONDS` (default 90 s) |
| Charts empty but status shows | Check that field numbers in `CONFIG.FIELDS` match your channel |
| Page blank on GitHub Pages | Wait a minute, hard refresh (Ctrl+F5), check Pages is enabled |
