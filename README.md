# HACKATON-LAND-DEGRADATION-UPGRADE
# Hackathon-Update-Land-Degradation-Hackathon

# 🌱 **Project Title: LandSight — Regenerating the Earth with Data**

## 🧩 **Hackathon Problem Fit**

**Problem:** Smallholder farmers and land stewards lack actionable, affordable data to monitor vegetation health and soil recovery.
**Solution:** LandSight empowers them with NDVI-based vegetation insights, soil recommendations, and easy visual reports — using free satellite imagery, open APIs, and Supabase as a lean data backend.

---

# ⚙️ **Architecture Overview**

**Data Flow:**

1. **Input:** Sentinel-2 imagery (Red + NIR bands) or uploaded drone imagery.
2. **Processing:** Python script computes NDVI, creates a GeoTIFF + JSON summary.
3. **Storage:** Supabase (PostgreSQL + Storage) holds raster + metadata.
4. **API Layer:** FastAPI serves NDVI maps, stats, and insights.
5. **Frontend:** React + Leaflet displays NDVI map and recommendations.

```
[Sentinel-2/Drone Data]
          ↓
     [Python NDVI Processor]
          ↓
  [Supabase Storage + DB]
          ↓
      [FastAPI Backend]
          ↓
     [React Map UI]
```

---

# 🧠 **Smart Additions (Hackathon Polish)**

* **Soil Health Insights:** NDVI thresholds trigger simple text insights (e.g., “Low NDVI → consider mulching or replanting legumes”).
* **Regeneration Score:** Mean NDVI mapped to a 0-100 index.
* **Community Layer (optional):** Crowd-submitted soil samples pinned to map.
* **Offline-ready:** GeoJSON + local caching in browser for field use.

---

## 🗄️ 1) Supabase Schema (`supabase_schema.sql`)

```sql
create schema if not exists landsight;

create table if not exists landsight.projects (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  owner text,
  created_at timestamptz default now()
);

create table if not exists landsight.ndvi_maps (
  id uuid primary key default gen_random_uuid(),
  project_id uuid references landsight.projects(id) on delete cascade,
  area_name text,
  storage_path text,
  bbox geometry,
  min_ndvi double precision,
  max_ndvi double precision,
  mean_ndvi double precision,
  regen_score int,
  insight text,
  computed_at timestamptz default now()
);
```

---

## 🐍 2) Python NDVI Processor (`landsight_ndvi.py`)

```python
import os, numpy as np, rasterio, uuid
from datetime import datetime
from supabase import create_client

SUPABASE_URL = os.getenv("SUPABASE_URL")
SUPABASE_KEY = os.getenv("SUPABASE_KEY")
SUPABASE_BUCKET = "ndvi"
PROJECT_ID = None
AREA_NAME = "Demo Plot"

def compute_ndvi(nir, red):
    ndvi = (nir - red) / (nir + red + 1e-6)
    return np.clip(ndvi, -1, 1)

def regen_insight(mean_ndvi):
    if mean_ndvi < 0.2:
        return "Critical: Vegetation stress. Introduce cover crops or mulching."
    elif mean_ndvi < 0.5:
        return "Fair: Recovery in progress. Maintain soil moisture and organic matter."
    else:
        return "Healthy: Strong vegetation cover. Continue regenerative practices."

def main():
    supabase = create_client(SUPABASE_URL, SUPABASE_KEY)
    red = rasterio.open("B04_red.tif").read(1).astype("float32")
    nir = rasterio.open("B08_nir.tif").read(1).astype("float32")

    ndvi = compute_ndvi(nir, red)
    mean = float(np.nanmean(ndvi))
    score = int((mean + 1) * 50)  # scale [-1,1] → [0,100]
    insight = regen_insight(mean)

    # Save GeoTIFF
    out_name = f"ndvi_{datetime.utcnow().strftime('%Y%m%dT%H%M%S')}.tif"
    with rasterio.open("B04_red.tif") as src:
        profile = src.profile
        profile.update(dtype="float32", count=1)
        with rasterio.open(out_name, "w", **profile) as dst:
            dst.write(ndvi, 1)

    storage_path = f"ndvi/{uuid.uuid4()}/{out_name}"
    supabase.storage().from_("ndvi").upload(storage_path, open(out_name, "rb"))
    supabase.table("landsight.ndvi_maps").insert({
        "project_id": PROJECT_ID,
        "area_name": AREA_NAME,
        "storage_path": storage_path,
        "min_ndvi": float(ndvi.min()),
        "max_ndvi": float(ndvi.max()),
        "mean_ndvi": mean,
        "regen_score": score,
        "insight": insight
    }).execute()
    print(f"Uploaded {out_name} | Regen score {score} | {insight}")

if __name__ == "__main__":
    main()
```

---

## ⚡ 3) FastAPI Backend (`main.py`)

```python
from fastapi import FastAPI, HTTPException
from supabase import create_client
import os

app = FastAPI(title="LandSight API")
supabase = create_client(os.getenv("SUPABASE_URL"), os.getenv("SUPABASE_KEY"))
BUCKET = "ndvi"

@app.get("/ndvi")
def list_maps():
    res = supabase.table("landsight.ndvi_maps").select("*").order("computed_at", desc=True).execute()
    return res.data

@app.get("/ndvi/{id}")
def get_map(id: str):
    res = supabase.table("landsight.ndvi_maps").select("*").eq("id", id).single().execute()
    if not res.data:
        raise HTTPException(status_code=404, detail="Not found")
    signed = supabase.storage().from_(BUCKET).create_signed_url(res.data["storage_path"], 3600)
    res.data["signed_url"] = signed.get("signedURL", signed)
    return res.data
```

---

## 💻 4) React Frontend (`LandSightMap.jsx`)

```jsx
import React, { useEffect, useState } from "react";
import { MapContainer, TileLayer, ImageOverlay, useMap } from "react-leaflet";

function Fit({ bounds }) {
  const map = useMap();
  useEffect(() => { bounds && map.fitBounds(bounds); }, [bounds]);
  return null;
}

export default function LandSightMap() {
  const [maps, setMaps] = useState([]);
  const [selected, setSelected] = useState(null);
  const [url, setUrl] = useState(null);

  useEffect(() => {
    fetch("/api/ndvi").then(r => r.json()).then(setMaps);
  }, []);

  useEffect(() => {
    if (selected)
      fetch(`/api/ndvi/${selected}`).then(r => r.json()).then(d => setUrl(d.signed_url));
  }, [selected]);

  return (
    <div style={{ height: "90vh" }}>
      <MapContainer center={[0, 37]} zoom={6} style={{ height: "80%" }}>
        <TileLayer url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png" />
        {url && (
          <>
            <ImageOverlay url={url} bounds={[[-1.3, 36.8], [-1.28, 36.84]]} opacity={0.7} />
            <Fit bounds={[[-1.3, 36.8], [-1.28, 36.84]]} />
          </>
        )}
      </MapContainer>
      <div style={{ padding: 10 }}>
        <h3>NDVI Insights</h3>
        <select onChange={e => setSelected(e.target.value)} defaultValue="">
          <option value="" disabled>Select map</option>
          {maps.map(m => (
            <option key={m.id} value={m.id}>
              {m.area_name} — Regen Score {m.regen_score}
            </option>
          ))}
        </select>
        {selected && (
          <div style={{ marginTop: 10 }}>
            {maps.find(m => m.id === selected)?.insight}
          </div>
        )}
      </div>
    </div>
  );
}
```

---

## 🧭 **Judging-Ready Storyline**

1. **Inspiration:** 70% of Africa’s farmers rely on degraded soils; regenerating land needs local, affordable insight.
2. **Innovation:** LandSight fuses satellite NDVI, soil data, and open-source tech to visualize regeneration progress.
3. **Impact:** Helps farmers track vegetation recovery, plan reforestation, and verify carbon credits or land stewardship results.
4. **Scalability:** Serverless backend (Supabase + FastAPI) + free satellite data = zero-cost scaling.
5. **Future:** Integrate weather forecasts, carbon metrics, and SMS alerts in regional languages.

---

## 🏁 Quick Run

```bash
# 1. Apply schema in Supabase
# 2. Add .env with SUPABASE_URL and SUPABASE_KEY

pip install rasterio numpy fastapi uvicorn supabase
python landsight_ndvi.py
uvicorn main:app --port 8000

# In React app
npm install react-leaflet leaflet
npm start
```

---

## 🌍 **Pitch Summary (30-sec version)**

> “We built LandSight — a tool that empowers farmers to regenerate the Earth.
> With one satellite image, we reveal how alive their land really is.
> Our NDVI-driven Regeneration Score turns pixels into purpose —
> helping local communities measure recovery, not just yield.”

---

### 🚀 Full Script: *LandReGen_Hackathon.py*

```python
# LandReGen_Hackathon.py
# Creates NDVI tile + 3-slide pitch deck automatically
# Requirements: pip install matplotlib numpy python-pptx

import numpy as np
import matplotlib.pyplot as plt
from pptx import Presentation
from pptx.util import Inches
from io import BytesIO

# -----------------------------
# STEP 1: Generate NDVI Map Tile
# -----------------------------
def generate_ndvi_tile(filename="ndvi_tile.png"):
    # Simulate NDVI data (-1 to 1)
    ndvi = np.random.uniform(-1, 1, (256, 256))

    # Use a green-to-red gradient
    cmap = plt.get_cmap('RdYlGn')

    fig, ax = plt.subplots(figsize=(3, 3), dpi=100)
    ax.imshow(ndvi, cmap=cmap, vmin=-1, vmax=1)
    ax.axis('off')
    plt.tight_layout(pad=0)

    # Save the NDVI image
    plt.savefig(filename, bbox_inches='tight', pad_inches=0)
    plt.close(fig)
    print(f"✅ NDVI tile saved as {filename}")

# -----------------------------
# STEP 2: Create Pitch Deck
# -----------------------------
def create_pitch_deck(ndvi_image="ndvi_tile.png", output="LandReGen_PitchDeck.pptx"):
    prs = Presentation()

    slides = [
        {
            "title": "Problem",
            "content": (
                "Land degradation is accelerating across sub-Saharan regions due to "
                "climate stress, overgrazing, and deforestation. "
                "Communities lack timely, accurate data to guide regeneration efforts."
            )
        },
        {
            "title": "Solution",
            "content": (
                "Land ReGen combines satellite imagery and NDVI analytics to map vegetation "
                "health in near-real time. This visualization helps prioritize degraded zones "
                "for reforestation and sustainable farming."
            )
        },
        {
            "title": "Impact",
            "content": (
                "Empowering local stakeholders to restore ecosystems, improve food security, "
                "and sequester carbon — creating data-driven pathways toward land resilience."
            )
        }
    ]

    # Create slides
    for i, s in enumerate(slides):
        slide_layout = prs.slide_layouts[1]  # Title and Content
        slide = prs.slides.add_slide(slide_layout)
        slide.shapes.title.text = s["title"]
        slide.placeholders[1].text = s["content"]

        # Add NDVI map to Slide 2 (the 'Solution' slide)
        if i == 1:
            left = Inches(5)
            top = Inches(1.5)
            height = Inches(3)
            slide.shapes.add_picture(ndvi_image, left, top, height=height)

    prs.save(output)
    print(f"✅ Pitch deck created: {output}")

# -----------------------------
# STEP 3: Run Everything
# -----------------------------
if __name__ == "__main__":
    generate_ndvi_tile()
    create_pitch_deck()
    print("\n🌍 LandReGen Hackathon package ready! Submit both files:")
    print(" - ndvi_tile.png")
    print(" - LandReGen_PitchDeck.pptx")
```

---

### 💡 How to Run

1. Save as `LandReGen_Hackathon.py`
2. Install dependencies:

   ```bash
   pip install matplotlib numpy python-pptx
   ```
3. Run:

   ```bash
   python LandReGen_Hackathon.py
   ```
4. Output:

   * `ndvi_tile.png` → your NDVI visualization (green → red)
   * `LandReGen_PitchDeck.pptx` → 3-slide deck ready for upload

---

### 🌿 Optional Custom Touches

* Replace simulated NDVI with real satellite data:

  ```python
  ndvi = (nir_band - red_band) / (nir_band + red_band + 1e-6)
  ```
* Add logos or your hackathon team name on Slide 3.
* Use a real-world map snippet as a background for extra polish.

---





