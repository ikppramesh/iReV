# iReV — Charge Matrix

> **EV Charging Calculator & Live Station Finder for Indian Electric Vehicles**
> Deployed at → [https://ikppramesh.github.io/iReV](https://ikppramesh.github.io/iReV)

---

## Table of Contents

1. [Product Overview](#1-product-overview)
2. [Goals & Non-Goals](#2-goals--non-goals)
3. [User Personas](#3-user-personas)
4. [Architecture](#4-architecture)
5. [Feature Specification — Calculator Tab](#5-feature-specification--calculator-tab)
6. [Feature Specification — Live Stations Tab](#6-feature-specification--live-stations-tab)
7. [Feature Specification — Charging Apps Tab](#7-feature-specification--charging-apps-tab)
8. [Data Reference — Vehicles](#8-data-reference--vehicles)
9. [Data Reference — Charger Types](#9-data-reference--charger-types)
10. [Data Reference — Operator Pricing](#10-data-reference--operator-pricing)
11. [Data Reference — Curated Stations (Hyderabad)](#11-data-reference--curated-stations-hyderabad)
12. [Data Reference — Charging Apps](#12-data-reference--charging-apps)
13. [API Integrations](#13-api-integrations)
14. [Design System](#14-design-system)
15. [Responsive Behaviour](#15-responsive-behaviour)
16. [Key Calculations & Formulas](#16-key-calculations--formulas)
17. [Known Limitations](#17-known-limitations)
18. [Future Enhancements](#18-future-enhancements)

---

## 1. Product Overview

**iReV Charge Matrix** is a single-page progressive web app (no build toolchain, no server) designed for Indian EV owners — primarily the **Mahindra XEV 9e Cineluxe** — to:

- Calculate exact charging **time, cost, and range** for any charger speed and SOC range
- Discover **live nearby charging stations** with distance, pricing, plug availability, and one-tap navigation
- Browse **charging network apps** with direct Android / iOS store links

The entire product ships as a single `index.html` file hosted on GitHub Pages, with zero dependencies beyond Leaflet.js (map) loaded via CDN.

---

## 2. Goals & Non-Goals

### Goals
- Accurate, real-time-ish cost estimates with full GST 18% tax breakdown
- Support all 14 charger types available in India (CCS2, Type 2, Type 6, BHIM, 3-pin)
- Cover 40+ Indian EV models across 14 manufacturers
- Show nearby stations via OpenChargeMap API with graceful offline fallback
- Provide one-tap Google Maps navigation to any station (name-based search for accuracy)
- List every major Indian charging network app with verified store links

### Non-Goals
- Payment processing or in-app charging session management
- Real-time slot reservation
- Vehicle telematics or OBD integration
- Trip planning / route optimisation
- Push notifications for station availability

---

## 3. User Personas

| Persona | Need |
|---|---|
| **New EV Owner (XEV 9e)** | "How long and how much will it cost to charge from 20% to 80% at Lulu Mall?" |
| **Highway Traveller** | "Which CCS2 stations are within 50 km? How far is each?" |
| **Cost-conscious Daily Driver** | "What's the cheapest nearby charger right now, including GST?" |
| **Multi-EV Household** | "I have a Creta EV too — what's my charging cost for that battery?" |
| **New to Public Charging** | "Which app do I download to pay at Charge Zone / GOEC / Statiq?" |

---

## 4. Architecture

```
index.html  (single file — HTML + CSS + JS)
│
├── <style>          Dark-theme CSS variables, component styles, animations
├── <header>         Sticky brand bar — updates dynamically with selected vehicle
├── <div.tabs>       3 navigation tabs (sticky, scrollable on mobile)
│
├── panel-calc       ⚡ Calculator Tab
│   ├── Vehicle Selector Card   (cascading Make→Model→Variant)
│   ├── Battery State Card      (20-cell animated pack + SOC sliders)
│   ├── Select Charger Card     (DC / AC grouped grid)
│   ├── Settings Card           (price, battery kWh, efficiency, km/kWh)
│   └── Results Card            (time, cost, range, energy + breakdown)
│
├── panel-stations   📍 Live Stations Tab
│   ├── Controls Row            (Detect Location, Refresh, Radius selector)
│   ├── View Toggle             (List / Map)
│   ├── Filter Chips            (All, Malls, DC, AC, CCS2, Type 2, 50kW+, Available)
│   ├── Status Bar              (live / cached / loading)
│   ├── Map Container           (Leaflet + OpenStreetMap, toggleable)
│   └── Stations List           (cards with plugs, pricing, navigation)
│
├── panel-apps       📱 Charging Apps Tab
│   └── App Grid               (14 apps, Android + iOS links)
│
└── <script>
    ├── VEHICLES[]              40+ Indian EVs with battery/range/efficiency data
    ├── CHARGERS[]              14 charger type definitions
    ├── OP_RATES{}              20 operator pricing entries (base + GST)
    ├── FALLBACK_STATIONS[]     14 curated Hyderabad stations
    ├── APPS[]                  14 charging apps with store links
    └── Functions               ~30 JS functions (see §5–7)
```

**Dependencies:**
| Library | Version | Purpose |
|---|---|---|
| Leaflet.js | 1.9.4 | Interactive map (OpenStreetMap tiles) |
| OpenChargeMap API | v3 | Live station data |

**Hosting:** GitHub Pages — `main` branch, root `/index.html`

---

## 5. Feature Specification — Calculator Tab

### 5.1 Vehicle Selector

**Purpose:** Replace the fixed XEV 9e assumption with a configurable vehicle, auto-populating battery capacity and real-world efficiency.

**UI:** Three cascading `<select>` dropdowns — **Manufacturer → Model → Variant/Pack**

**Behaviour:**
1. On load, defaults to **Mahindra → XEV 9e → Cineluxe (79 kWh)**
2. Changing Make populates Model list and resets to first model
3. Changing Model populates Variant list and resets to first variant
4. Changing Variant:
   - Sets **Battery (kWh)** input to the variant's battery capacity
   - Sets **Range (km/kWh)** input to `variant.effKm × 0.70` (70% of WLTP as real-world estimate, capped at 8.5)
   - Updates **header badge** and **header subtitle**
   - Re-renders vehicle showcase card and recalculates results

**Vehicle Showcase Card** (rendered below dropdowns):
- Manufacturer name (small caps)
- Model name (large bold)
- Variant + body type label (SUV/Crossover, Sedan, Hatchback, 2-Wheeler)
- Specs row: Battery kWh · WLTP Range (km) · WLTP km/kWh · Estimated real km/kWh
- Styled badge pill with make + model + kWh

---

### 5.2 Battery State (Pack Visualisation)

**Visual:** A horizontal row of **20 cells**, each representing **5% SOC** of the battery.

| Cell State | Colour | Meaning |
|---|---|---|
| `c-charged` | Green (`#22c55e`) | Already stored charge (0% → From SOC) |
| `c-charging` | Cyan pulsing (`#00e5ff`) | Energy to be added (From SOC → To SOC) |
| `c-empty` | Near-black with subtle border | Remaining capacity (To SOC → 100%) |

Below the pack cells:
- **Left label**: From SOC% · stored kWh (green)
- **Right label**: To SOC% · target kWh (cyan)
- **Centre summary**: `{stored kWh} stored + charging {delta kWh} = {target kWh} target`

**Battery capacity terminal:** A small cap rectangle on the right side of the pack, styled to resemble a physical battery terminal.

**SOC Sliders** (below pack):
- **From SOC** (0–95%): Red-to-border gradient track, current percentage displayed live
- **To SOC** (5–100%): Cyan-to-border gradient track, current percentage displayed live
- Enforced constraint: To SOC always > From SOC (auto-corrects on conflict)

---

### 5.3 Charger Selection

**Layout:** Two sections — **DC (Direct Current)** and **AC (Alternating Current)** — each with a colour-coded section label and a responsive card grid.

**Charger Card States:**
- Default: dark card, cyan kW label
- Selected DC: blue border + background, blue kW
- Selected AC: purple border + background, purple kW

Each card shows:
- Power in kW (large bold)
- Plug type (CCS2, Type 2, BHIM, etc.)
- AC/DC badge (blue for DC, purple for AC)
- Speed badge (ULTRA / FAST / MED / SLOW)

Default selected charger: **CCS2 50 kW (DC)**

---

### 5.4 Settings

| Field | Default | Notes |
|---|---|---|
| Price (₹/kWh) | ₹19.00 | In-pocket price including GST; updates cost breakdown dynamically |
| Battery (kWh) | 79 | Auto-set by vehicle selection; manually editable |
| Efficiency | 92% — AC fast | Options: 85% (home slow), 92% (AC fast), 96% (DC fast) |
| Range (km/kWh) | 5.5 | Real-world estimate; auto-set to 70% of WLTP on vehicle select |

---

### 5.5 Results

**Progress bar:** Fills to `(To SOC − From SOC)%` width with purple-to-cyan gradient.

**Result Cards (4):**
| Card | Primary Value | Sub-value |
|---|---|---|
| ⏱️ Charge Time | Xh Ym | Total minutes |
| 💰 Total Cost | ₹XXXX | ₹X.XX/km |
| 🛣️ Range Added | XXX km | kWh consumed |
| ⚡ Energy Added | XX.X kWh | kWh from wall |

**Cost Breakdown Table:**
| Row | Description |
|---|---|
| Charger | kW · Plug Type |
| Current Type | DC Fast / AC · Speed label |
| Wall Draw (incl. losses) | kWh from wall + kW draw |
| Base rate (pre-tax) | ₹X.XX/kWh (price ÷ 1.18) |
| GST 18% | +₹X.XX/kWh (in orange) |
| In-pocket rate | ₹X.XX/kWh (in green) |
| Cost per km (with tax) | ₹X.XX/km |
| Petrol equivalent saving | ₹XXXX saved vs petrol (@₹105/L, 15 km/L) |
| Net energy added | XX.XX kWh net |

---

## 6. Feature Specification — Live Stations Tab

### 6.1 Controls

- **📍 Detect My Location** — Triggers browser Geolocation API; fallback to Hyderabad default (17.4435°N, 78.3772°E)
- **🔄 Refresh** — Re-fetches OCM data with current location and radius
- **Radius Selector** — 5 km / 10 km / 20 km / 50 km (default: 50 km)

### 6.2 View Toggle

- **☰ List View** — Station cards sorted by distance (default)
- **🗺 Map View** — Leaflet interactive map with colour-coded markers

### 6.3 Filter Chips

| Filter | Behaviour |
|---|---|
| All | Show all stations |
| 🏬 Malls | Stations tagged `isMall:true` (regex-detected or curated) |
| ⚡ DC Only | Stations with at least one DC plug |
| 〜 AC Only | Stations with only AC plugs |
| CCS2 | Stations with CCS2 plug type |
| Type 2 | Stations with Type 2 / IEC 62196 plug type |
| 50 kW+ | Stations with at least one plug ≥ 50 kW |
| 🟢 Available | Stations with at least one free slot |

### 6.4 Station Card

Each station card displays:

**Header:**
- Station name + 🏬 Mall tag (if applicable)
- Address · Operator name
- Pricing: `₹{base}/kWh + GST ₹{gst} = ₹{total}/kWh in pocket`
- Operational status (colour-coded: green = operational, red = unavailable)
- Distance (km or metres) + estimated drive time

**Plug Tiles (grouped by DC / AC):**
- Plug type name
- Power (kW)
- Slots: X free / Y total
- Per-plug pricing: base · GST · in-pocket
- DC badge (blue) or AC badge (purple)

**Footer:**
- ✔ Verified timestamp (last seen)
- 🧭 Navigate button → Opens **Google Maps Search** for station name + address
- ⚡ Use Xk W in Calc button → Switches to Calculator tab with that charger pre-selected

### 6.5 Station Data Pipeline

```
User taps "Detect Location"
    ↓
GPS coords obtained (or fallback to Hyderabad default)
    ↓
Fetch OCM API (max 100 results, within selected radius)
    ↓  [success]                    ↓  [network error / CORS]
Map OCM response to stations     Use FALLBACK_STATIONS
Apply operator pricing (OP_RATES)    Filter by radius
Merge with FALLBACK_STATIONS         Apply operator pricing
(dedup by name prefix, 12 chars)
Filter by radius
Sort by distance (haversine)
    ↓
renderStations() + plotMapMarkers()
    ↓
Auto-refresh in 3 minutes
```

**Status bar states:**
- 🔵 Loading: "Initialising…" (pulse dot)
- 🟢 Live: "Live · N stations within X km · HH:MM:SS"
- 🟡 Cached: "Curated · N Hyderabad stations · OCM API unavailable"

### 6.6 Leaflet Map

- **Tiles:** OpenStreetMap (`https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png`)
- **User marker:** Cyan circle (16px) with white border and glow
- **Station markers:** 13px circles
  - Green = has free slots
  - Red = busy / slot status unknown
  - Grey = not operational
- **Station popup:** Name · Address · Distance · Price · Max kW · Slot status · Navigate link · Calc shortcut
- **Auto-bounds:** Map fits all visible station markers + user location with 30px padding

---

## 7. Feature Specification — Charging Apps Tab

### 7.1 App Cards

Each app card displays:
- App icon (emoji-based)
- App name + network name
- Description of the network and primary use case
- Android store button (Google Play deep link)
- iOS store button (App Store deep link)
- Hyderabad station coverage note

**14 apps listed** — see §12 for full data.

---

## 8. Data Reference — Vehicles

**Total: 35 models, 40+ manufacturers, ~80 variants across 4-wheelers and 2-wheelers**

### 4-Wheelers (27 models)

#### Mahindra (3 models)
| Model | Variant | Battery (kWh) | WLTP Range (km) | WLTP km/kWh |
|---|---|---|---|---|
| XEV 9e | MX1 Pro | 59 | 542 | 9.2 |
| XEV 9e | MX2 Pro | 59 | 542 | 9.2 |
| XEV 9e | Pack One | 79 | 656 | 8.3 |
| XEV 9e | Pack Two | 79 | 656 | 8.3 |
| **XEV 9e** | **Cineluxe ★** | **79** | **656** | **8.3** |
| BE 6e | MX1 Pro | 59 | 535 | 9.1 |
| BE 6e | MX2 Pro | 59 | 535 | 9.1 |
| BE 6e | Pack One | 79 | 682 | 8.6 |
| BE 6e | Pack Two | 79 | 682 | 8.6 |
| XUV400 EV | EC Pro | 34.5 | 375 | 10.9 |
| XUV400 EV | EL Pro | 39.4 | 456 | 11.6 |

★ = App default on load

#### Tata (5 models)
| Model | Variant | Battery (kWh) | WLTP Range (km) | WLTP km/kWh |
|---|---|---|---|---|
| Nexon EV | Creative | 30 | 312 | 10.4 |
| Nexon EV | Fearless | 30 | 312 | 10.4 |
| Nexon EV | Creative+ LR | 40.5 | 465 | 11.5 |
| Nexon EV | Fearless+ LR | 40.5 | 465 | 11.5 |
| Punch EV | Smart | 25 | 315 | 12.6 |
| Punch EV | Smart+ | 25 | 315 | 12.6 |
| Punch EV | Adventure | 35 | 421 | 12.0 |
| Punch EV | Empowered | 35 | 421 | 12.0 |
| Tiago EV | XT | 19.2 | 250 | 13.0 |
| Tiago EV | XZ+ | 19.2 | 250 | 13.0 |
| Tiago EV | XZ+ LR | 24 | 315 | 13.1 |
| Tigor EV | XE / XM / XZ+ | 26 | 306 | 11.8 |
| Curvv EV | Creative | 45 | 502 | 11.2 |
| Curvv EV | Accomplished | 55 | 585 | 10.6 |
| Curvv EV | Empowered+ | 55 | 585 | 10.6 |

#### Hyundai (4 models)
| Model | Variant | Battery (kWh) | WLTP Range (km) | WLTP km/kWh |
|---|---|---|---|---|
| Creta EV | Executive | 45 | 473 | 10.5 |
| Creta EV | Smart / Premium / Excellence | 51.4 | 473 | 9.2 |
| Ioniq 5 | Standard Range | 58 | 385 | 6.6 |
| Ioniq 5 | Long Range | 72.6 | 631 | 8.7 |
| Ioniq 6 | RWD Standard | 53 | 429 | 8.1 |
| Ioniq 6 | RWD Long Range | 77.4 | 614 | 7.9 |
| Kona Electric | Standard | 39.2 | 452 | 11.5 |

#### MG (3 models)
| Model | Variant | Battery (kWh) | WLTP Range (km) | WLTP km/kWh |
|---|---|---|---|---|
| ZS EV | Excite | 44.5 | 461 | 10.4 |
| ZS EV | Exclusive | 50.3 | 461 | 9.2 |
| Comet EV | Pace / Spin | 17.3 | 230 | 13.3 |
| Windsor EV | Excite / Exclusive / Essence | 38 | 331 | 8.7 |

#### Kia (2 models)
| Model | Variant | Battery (kWh) | WLTP Range (km) | WLTP km/kWh |
|---|---|---|---|---|
| EV6 | RWD Standard | 58 | 394 | 6.8 |
| EV6 | RWD / AWD Long Range | 77.4 | 483–528 | 6.2–6.8 |
| EV9 | RWD / AWD Long Range | 99.8 | 505–561 | 5.1–5.6 |

#### BYD (2 models)
| Model | Variant | Battery (kWh) | WLTP Range (km) | WLTP km/kWh |
|---|---|---|---|---|
| Atto 3 | Standard | 60.5 | 521 | 8.6 |
| Seal | Dynamic / Performance | 82.56 | 580–650 | 7.0–7.9 |

#### BMW (4 models)
| Model | Variant | Battery (kWh) | WLTP Range (km) | WLTP km/kWh |
|---|---|---|---|---|
| i4 | eDrive35 | 66.5 | 590 | 8.9 |
| i4 | eDrive40 / M50 | 80.7 | 510–590 | 6.3–7.3 |
| iX1 | xDrive30 | 64.7 | 440 | 6.8 |
| iX3 | Inspiring | 74 | 460 | 6.2 |
| i7 | xDrive60 | 101.7 | 590 | 5.8 |

#### Mercedes-Benz (4 models)
| Model | Variant | Battery (kWh) | WLTP Range (km) | WLTP km/kWh |
|---|---|---|---|---|
| EQS | 450+ / 580 4M | 107.8 | 676–770 | 6.3–7.1 |
| EQE | 350+ | 90.6 | 660 | 7.3 |
| EQA | 250+ | 70.5 | 560 | 7.9 |
| EQB | 300 4M | 66.5 | 419 | 6.3 |

#### Audi (2 models)
| Model | Variant | Battery (kWh) | WLTP Range (km) | WLTP km/kWh |
|---|---|---|---|---|
| Q8 e-tron | 50 quattro | 95 | 491 | 5.2 |
| Q8 e-tron | 55 quattro | 114 | 582 | 5.1 |
| Q4 e-tron | 40 / 50 quattro | 77 | 488–520 | 6.3–6.8 |

#### Volvo (3 models)
| Model | Battery (kWh) | WLTP Range (km) | WLTP km/kWh |
|---|---|---|---|
| XC40 Recharge | 67 | 418 | 6.2 |
| C40 Recharge | 67 | 440 | 6.6 |
| EX90 | 107 | 580 | 5.4 |

#### Toyota (1 model)
| Model | Variant | Battery (kWh) | WLTP Range (km) | WLTP km/kWh |
|---|---|---|---|---|
| bZ4X | FWD / AWD | 71.4 | 470–516 | 6.6–7.2 |

---

### 2-Wheelers (8 models)

| Make | Model | Variant | Battery (kWh) | Range (km) | km/kWh |
|---|---|---|---|---|---|
| Ola Electric | S1 Pro | Gen 3 | 4.0 | 195 | 48.8 |
| Ola Electric | S1 Air | Gen 3 | 3.0 | 151 | 50.3 |
| Ola Electric | S1 X | 2 kWh / 3 kWh / 4 kWh | 2–4 | 91–190 | 45.5–50.3 |
| Ather Energy | Rizta | S (2.9) / Z (3.7) | 2.9–3.7 | 123–160 | 42.4–43.2 |
| Ather Energy | 450X | Gen 3 | 2.9 | 146 | 50.3 |
| TVS | iQube | S / ST | 3.04–5.1 | 100–145 | 28.4–32.9 |
| Bajaj | Chetak | Premium / Urbane | 3.2 | 113–128 | 35.3–40.0 |
| Simple Energy | Simple One | Standard | 4.8 | 212 | 44.2 |

---

## 9. Data Reference — Charger Types

All 14 charger types supported in the app:

| ID | Label | kW | Plug Type | Current | Speed Tier |
|---|---|---|---|---|---|
| ccs240 | 240 kW | 240 | CCS2 | DC | ULTRA |
| ccs175 | 175 kW | 175 | CCS2 | DC | ULTRA |
| ccs120 | 120 kW | 120 | CCS2 | DC | ULTRA |
| ccs90 | 90 kW | 90 | CCS2 | DC | FAST |
| ccs60 | 60 kW | 60 | CCS2 | DC | FAST |
| **ccs50** | **50 kW** | **50** | **CCS2** | **DC** | **FAST (Default)** |
| ccs30 | 30 kW | 30 | CCS2 | DC | MED |
| ccs25 | 25 kW | 25 | CCS2 | DC | MED |
| ac22 | 22 kW | 22 | Type 2 | AC | MED |
| ac11 | 11 kW | 11 | Type 2 | AC | MED |
| ac72 | 7.2 kW | 7.2 | Type 2 | AC | MED |
| t6 | 12 kW | 12 | Type 6 | AC | MED |
| bhim | 15 kW | 15 | BHIM | DC | MED |
| slow | 3.3 kW | 3.3 | 3-pin | AC | SLOW |

**Speed tier visual colours:**
- ULTRA → Purple (`#7c3aed`)
- FAST → Blue (`#2563eb`)
- MED → Amber (`#d97706`)
- SLOW → Slate (`#475569`)

---

## 10. Data Reference — Operator Pricing

All prices are per kWh. GST is 18% flat across all operators.

| Operator Key(s) | Base (₹/kWh) | GST | In-Pocket (₹/kWh) |
|---|---|---|---|
| go ec, goec | 20.00 | 18% | **23.60** |
| bpcl | 15.25 | 18% | **18.00** |
| thunder | 15.25 | 18% | **18.00** |
| tata power, tata, charge zone, chargezone, jio-bp, jio, pulse | 14.41 | 18% | **17.00** |
| statiq, mahindra, zeon, magenta | 13.56 | 18% | **16.00** |
| ather | 12.71 | 18% | **15.00** |
| iocl | 11.86 | 18% | **14.00** |
| eesl | 10.17 | 18% | **12.00** |
| shell | 16.95 | 18% | **20.00** |
| **default** (unrecognised operator) | **16.10** | **18%** | **19.00** |

> Lulu Mall (GOEC) is the most expensive at ₹23.60/kWh in-pocket.
> EESL government stations are the cheapest at ₹12.00/kWh in-pocket.

---

## 11. Data Reference — Curated Stations (Hyderabad)

14 hand-curated stations used as offline fallback when OCM API is unavailable. All filtered by the user's selected radius.

### Standalone Stations

| Name | Area | Operator | Fastest Plug |
|---|---|---|---|
| Thunder Plus — Izzathnagar | Izzathnagar | Thunder Plus | CCS2 240 kW |
| Tata Power EZ Charge — Hitech City | Madhapur | Tata Power | CCS2 50 kW |
| Charge Zone Hub — Gachibowli | Gachibowli | Charge Zone | CCS2 60 kW |
| EESL Charging Station — Jubilee Hills | Jubilee Hills | EESL | CCS2 25 kW |
| Mahindra e-Charge — Banjara Hills | Banjara Hills | Mahindra Electric | CCS2 50 kW |
| Statiq — Kondapur | Kondapur | Statiq | CCS2 50 kW |
| Jio-bp Pulse — Madhapur | Madhapur | Jio-bp | CCS2 60 kW |

### Mall Stations

| Mall | Location | Operator | In-Pocket Rate |
|---|---|---|---|
| Lulu Mall Hyderabad | Kukatpally | GO EC | ₹23.60/kWh |
| Inorbit Mall — Cyberabad | Mindspace, Madhapur | Charge Zone | ₹17.00/kWh |
| Sarath City Capital Mall | Kondapur | Statiq | ₹16.00/kWh |
| Forum Sujana City Mall | Kukatpally | Charge Zone | ₹17.00/kWh |
| GVK One Mall | Banjara Hills | Tata Power | ₹17.00/kWh |
| Manjeera Mall — KPHB | KPHB Colony | EESL | ₹12.00/kWh |
| Nexus Mall (Westend) | Begumpet | Jio-bp | ₹17.00/kWh |
| Southside Mall — Mehdipatnam | Mehdipatnam | BPCL | ₹18.00/kWh |
| City Centre Mall | Banjara Hills | Magenta ChargeGrid | ₹16.00/kWh |

---

## 12. Data Reference — Charging Apps

| App | Network | Android Package | iOS ID | Key Coverage (Hyderabad) |
|---|---|---|---|---|
| GOEC | GOEC Autotech | `com.namp.azadpower` | `id1600027947` | Lulu Mall Kukatpally |
| Tata Power EZ Charge | Tata Power | `com.tatapower.evapp` | `id1496350476` | GVK One, Hitech City |
| Charge Zone | Charge Zone | `com.chargezone` | `id1627426498` | Inorbit, Forum Sujana, Gachibowli |
| Statiq | Statiq EV | `com.statiq` | `id1520534378` | Sarath City, Kondapur, IT corridor |
| Jio-bp Pulse | Jio-bp | `com.namp.jiobppulse` | `id1600465639` | Nexus Mall, petrol pumps |
| eDrive BPCL | BPCL | `com.edrive.bpcl` | `id6502164392` | Southside Mall, BPCL outlets |
| Zeon Charging | Zeon | `com.namp.zeon` | `id1539882735` | Hotels, offices, residences |
| Magenta ChargeGrid | Magenta Power | `com.magenta.chargegrid` | `id1575081547` | City Centre Mall, societies |
| Ather Grid | Ather Energy | `com.atherenergy.aegridapp` | `id1356687521` | Jubilee Hills, Kondapur |
| Bolt.Earth | REVOSAUTO Tech | `com.revos.bolt.android` | `id1541141946` | 37,000+ points, 1,700+ cities |
| ThunderPlus | Trinity Cleantech | `com.trinity.thunderplus` | `id6451310643` | Thunder Plus Izzathnagar, 800+ |
| IndianOil e-Charge | Indian Oil Corp | `com.iocl.ev` | `id6450923369` | IOCL petrol pumps |
| Charge_IN by Mahindra | Mahindra | `com.mahendra.chargin` | `id6754163852` | Dealerships, ultra-fast network |
| Kazam EV | Kazam EV Tech | `com.kazam.ev` | `id1563868545` | Tier-2 cities, highways |

---

## 13. API Integrations

### OpenChargeMap (OCM) API v3

**Endpoint:**
```
GET https://api.openchargemap.io/v3/poi/?output=json
    &latitude={userLat}
    &longitude={userLng}
    &distance={radius}
    &distanceunit=km
    &maxresults=100
    &compact=false
    &verbose=false
```

**Auth Header:** `X-API-Key: <key>`

**Key Response Fields Mapped:**

| OCM Field | App Field | Notes |
|---|---|---|
| `AddressInfo.Title` | `name` | Station name |
| `AddressInfo.AddressLine1 + Town` | `addr` | Display address |
| `AddressInfo.Latitude/Longitude` | `lat`, `lng` | GPS coordinates |
| `OperatorInfo.Title` | `operator` | Used for OP_RATES lookup |
| `StatusType.ID` | `operational` | 0/50/75/100 = operational, 150 = temp unavailable, 200 = future |
| `DateLastVerified` | `verified` | Displayed as relative time |
| `Connections[].ConnectionType.Title` | plug `type` | Used for DC/AC detection |
| `Connections[].PowerKW` | plug `kw` | Charger speed |
| `Connections[].Quantity` | plug `qty` | Total slots |

**DC Detection Regex:** `/CCS|CHAdeMO|BHIM|GB.T DC/i`

**Status ID Logic:**
```
StatusType.ID === 0, 50, 75, 100  → operational = true
StatusType.ID === 150             → operational = false, label = "Temporarily Unavailable"
StatusType.ID === 200             → operational = false, label = "Coming Soon"
```

**Deduplication:** OCM results and curated fallback stations are merged by comparing the first 12 characters of the lowercase, whitespace-stripped station name. If a curated station name-prefix exists in OCM results, it is suppressed.

**Auto-refresh:** Every 3 minutes (`setInterval` equivalent via `setTimeout` recursion).

---

### Google Maps Navigation

Navigation links use **name + address search** (not lat/lng coordinates) to ensure Google Maps finds the correct location:

```
https://www.google.com/maps/search/?api=1&query={encodeURIComponent(name + ' ' + addr)}
```

This approach was chosen because OCM coordinate accuracy varies and raw lat/lng links were found to route to incorrect locations.

---

### Leaflet.js + OpenStreetMap

```javascript
L.map('map').setView([userLat, userLng], 12)
L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
  attribution: '© OpenStreetMap',
  maxZoom: 19
})
```

Map is hidden by default. Revealed when user clicks **🗺 Map View**. `leafMap.invalidateSize()` is called on reveal to prevent tile rendering bugs.

---

## 14. Design System

### Colour Tokens

| Token | Value | Usage |
|---|---|---|
| `--bg` | `#0d0d0f` | Page background |
| `--card` | `#16181c` | Primary card background |
| `--card2` | `#1e2026` | Secondary card / input background |
| `--border` | `#2a2d35` | Card borders, dividers |
| `--accent` | `#00e5ff` | Primary cyan — active states, SOC markers, DC charging cells |
| `--accent2` | `#7c3aed` | Purple — header gradient, ULTRA badge, progress bar start |
| `--dc` | `#3b82f6` | DC charger labels, plug tiles |
| `--ac` | `#a855f7` | AC charger labels, plug tiles |
| `--green` | `#22c55e` | Operational status, charged SOC cells, available slots |
| `--red` | `#ef4444` | Unavailable status, busy markers |
| `--orange` | `#f97316` | GST highlight in cost breakdown |
| `--yellow` | `#eab308` | Delta kWh highlight |
| `--text` | `#f0f2f5` | Primary text |
| `--muted` | `#8b8fa8` | Secondary text, labels |

### Typography
- Font stack: `-apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif`
- Card titles: 0.72rem, 700 weight, uppercase, letter-spacing 1px
- Result values: 1.4rem, 800 weight
- Labels: 0.67–0.72rem

### Component Classes
| Class | Purpose |
|---|---|
| `.card` | Standard content card |
| `.result-card` | Stat result tile |
| `.charger-btn` | Charger selection button |
| `.plug-tile` | Plug slot tile in station card |
| `.chip` | Filter chip |
| `.station-card` | Live station card |
| `.mall-card` | Mall station variant (left border accent) |
| `.app-card` | Charging app card |
| `.vehicle-showcase` | Vehicle info showcase |
| `.pack-outer` | Battery pack container |
| `.p-cell` | Individual battery cell |

---

## 15. Responsive Behaviour

**Breakpoint:** `max-width: 600px`

| Element | Desktop | Mobile |
|---|---|---|
| SOC sliders | 2 columns | 1 column (stacked) |
| Results grid | Auto-fill (4 wide) | 2 columns |
| Vehicle selector | 3 columns | 1 column (stacked) |
| Vehicle specs | 4 columns | 2 columns |
| App grid | 2 columns | 1 column |
| Map height | 420px | 300px |
| Header DC/AC tags | Visible | Hidden |
| Panel padding | 18px | 13px |

---

## 16. Key Calculations & Formulas

### Energy to Add
```
deltaKwh = battery × (toSOC − fromSOC) / 100
```

### Wall Energy (accounting for charger efficiency losses)
```
wallKwh = deltaKwh / efficiency
```
Where `efficiency` = 0.85 (home slow) / 0.92 (AC fast) / 0.96 (DC fast)

### Charge Time
```
hours = wallKwh / chargerKw
```

### Total Cost
```
cost = wallKwh × pricePerKwh
```

### Range Added
```
rangeKm = deltaKwh × kmPerKwh
```

### Cost per km
```
costPerKm = cost / rangeKm
```

### Petrol Equivalent Saving
```
saving = (rangeKm / 15) × 105
```
Assumes 15 km/L petrol mileage and ₹105/L petrol price.

### GST Breakdown (from in-pocket price)
```
baseRate = price / 1.18
gstAmt   = price − baseRate
```

### Haversine Distance (for station sorting)
```javascript
haversine(lat1, lng1, lat2, lng2) → km
```
Earth radius = 6371 km. Used for sorting all stations by distance from user.

### Real-World Range Estimate (on vehicle select)
```
realWorldEff = min(wltpEffKm × 0.70, 8.5)  km/kWh
```
70% of WLTP, capped at 8.5 km/kWh to avoid over-optimistic highway estimates.

---

## 17. Known Limitations

| Limitation | Detail |
|---|---|
| **Slot availability is simulated** | OCM does not provide real-time slot data. A random availability simulation is applied per plug. A warning is shown to users. |
| **OCM coordinate accuracy varies** | Some OCM station coordinates are slightly inaccurate. Navigation therefore uses name+address Google Maps search instead of raw coordinates. |
| **No account / history** | Calculator inputs reset on page reload. No persistent state. |
| **No trip planning** | The app calculates for a single charge session, not a multi-stop journey. |
| **Curated stations are Hyderabad-only** | Fallback dataset covers 14 stations in Hyderabad only. Users outside Hyderabad rely on OCM API being available. |
| **Pricing data may drift** | Operator per-kWh rates are hardcoded. Actual prices may change without app update. |
| **2-wheeler charger types not differentiated** | Scooters and bikes in the vehicle list use the same charger grid as 4-wheelers. In reality, 2-wheelers typically use 3-pin or proprietary connectors. |

---

## 18. Future Enhancements

| Feature | Priority | Notes |
|---|---|---|
| Real vehicle images | High | Replace showcase placeholder with official manufacturer press images via CDN |
| Persistent settings | Medium | Save last-used vehicle, charger, and price to `localStorage` |
| Trip planner | Medium | Multi-stop route with charge stops, range anxiety indicator |
| Real-time slot data | Medium | Integrate operator APIs (Statiq, Charge Zone, Tata Power) for live slot counts |
| Price alerts | Low | Notify when per-kWh rate changes at a saved station |
| Cost comparison table | Medium | Side-by-side cost at all nearby stations for a given charge session |
| Offline PWA | Low | Service worker + manifest for installable app experience |
| Multi-city station database | Medium | Expand curated fallback to cover Chennai, Bengaluru, Pune, Mumbai |
| Charging history log | Low | Log past sessions with date, cost, kWh, location |
| Dark/light theme toggle | Low | Currently dark-only |
| Night rate support | Medium | Some operators (BPCL, EESL) offer cheaper rates off-peak |

---

## Quick Start (Local Development)

```bash
git clone https://github.com/ikppramesh/iReV.git
cd iReV
# Open in browser directly — no build step needed
open index.html
```

To push updates:
```bash
git add index.html
git commit -m "Your change"
git push origin main
# GitHub Pages auto-deploys within ~60 seconds
```

---

*Built for the Mahindra XEV 9e Cineluxe — Stealth Black.*
*Covers the Indian EV ecosystem from Tiago to EQS.*
