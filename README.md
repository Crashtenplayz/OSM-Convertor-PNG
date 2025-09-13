# OSM Convertor PNG

An interactive Python utility that downloads OpenStreetMap (and compatible) raster tiles for a chosen map style, zoom level, and area size, then optionally splits each tile into a fine grid and saves the results to disk.

The tool respects common tile-server conventions (including subdomains for load balancing), supports caching to avoid re-downloading tiles, and performs a preflight check before large downloads.

NOTICE: Always respect the usage policy of the tile provider you choose. For large-scale or automated use, run your own tile server or use a commercial provider with appropriate API keys and terms.

---

## Features

- Interactive prompts (no command-line flags required)
- Choose area size (length × width in tiles)
- Choose zoom level (0–19 typical; checks capacity vs. requested area)
- Optional border cleaning (for “Standard” style, uses a no-labels variant)
- Choose from multiple basemap providers:
  - Standard (OSM)
  - CycloSM
  - Thunderforest: Cycle Map, Transport (require API key)
  - Tracestrack Topo (requires key)
  - OpenTopoMap (Topo)
  - Humanitarian (HOT)
  - Shortbread (may not be publicly available)
  - MapTiler OMT (requires key)
- Optional per-tile splitting (e.g., split each tile into 2×2, 4×4, … up to 64×64)
- Caching of downloaded tiles to speed up reruns
- Preflight URL check to validate provider/key before downloading many tiles

---

## Requirements

- Python 3.10+ (uses modern type hints like `list[str] | None`)
- Pip packages:
  - `requests`
  - `Pillow`

Install dependencies:
```bash
pip install requests Pillow
```

---

## Getting Started

1. Save the script as `main.py` (already provided in this repository).
2. Install the requirements (see above).
3. Run:
```bash
python3 main.py
```

You will be prompted for:
- Length (number of tiles in X)
- Width (number of tiles in Y)
- Whether to “clean border road” (for Standard layer, this uses Carto Light No-Labels tiles)
- Zoom level (0–19 typical)
  - The script verifies that your requested area can fit within the zoom level (since a zoom `z` has `2^z` tiles per side).
  - If the requested area exceeds `2^z × 2^z`, you’ll be offered to use a suitable zoom or the area will be clamped.
- Split factor per tile (1 = no split; e.g., 2 → 2×2, 4 → 4×4 … up to 64)
- Basemap layer (choose a number from the list)
  - If the selected provider requires an API key, you’ll be prompted to enter it.
- The script runs a preflight check for the first tile, then proceeds to download.

Example session (abridged):
```
Welcome to OSM Tile Downloader!
Enter the length (number of tiles in X): 4
Enter the width (number of tiles in Y): 3
Do you want to clean border road? (y/n): y
Enter the zoom level you want (0-19 typical): 4
Split each tile into how many sub-tiles per side? (1=no split, e.g., 2 => 2x2, 4 => 4x4): 2

Choose a basemap layer:
  1. Standard
  2. CycloSM
  3. Cycle Map (Thunderforest)
  4. Transport Map (Thunderforest)
  5. Tracestrack Topo
  6. Topo (OpenTopoMap)
  7. Humanitarian (HOT)
  8. ShortBread
  9. MapTiler OMT
Enter the number of the layer: 1

[preflight] Testing first tile URL: https://a.basemaps.cartocdn.com/light_nolabels/4/0/0.png
[preflight] HTTP 200, content-type: image/png
[preflight] Looks good. Proceeding...
```

---

## Output Structure

- Output directory: `output/zoom_{z}/`
- Files are organized using 1-based coordinates:
  - `output/zoom_{z}/{x}/{y}.png`
  - Example: `output/zoom_4/3/7.png`

Notes:
- If you chose a split factor `S`, each original tile is split into `S × S` sub-tiles and saved individually with adjusted 1-based coordinates.
- For example, with split factor 2 and tile at `(tx, ty)`, the sub-tiles are saved at:
  - `gx = tx * 2 + sx + 1`, `gy = ty * 2 + sy + 1`, where `sx, sy ∈ {0, 1}`

---

## Caching

- Cache directory: `tile_cache/z/x/y.png`
- If a tile exists in the cache, it will be loaded from disk (faster and reduces server load).
- Corrupted cache entries are automatically discarded and re-downloaded.

---

## Providers and API Keys

Some providers require an API key. The script will prompt you when needed.

- Thunderforest (Cycle Map, Transport): requires `apikey`
- Tracestrack Topo: requires `key`
- MapTiler OMT: requires `key`
- Shortbread: may not be publicly available; requests could fail
- Standard, CycloSM, OpenTopoMap, HOT: generally do not require keys for light use, but still have usage policies and rate limits

You’ll be asked to paste your key when required. The script embeds it into the URL template for requests.

---

## Rate Limiting and Politeness

- The default delay between tile downloads is 0.2 seconds.
- You can adjust `delay` in the script if needed, but do not hammer public endpoints.
- For large-scale or commercial usage:
  - Run your own tile server
  - Or obtain a commercial plan from a provider

---

## Troubleshooting

- Preflight shows HTTP 401/403:
  - Invalid/missing API key, or your key doesn’t have access to the selected map/style
- Preflight shows HTTP 404/410:
  - The tile may not exist at the chosen zoom/provider
- Many tiles fail or are blank:
  - You may be outside provider coverage, or blocked by the provider
  - Reduce request rate (increase delay), or switch to a different provider
- Cache but unexpected images:
  - Remove the `tile_cache` folder to force fresh downloads

---

## Programmatic Use (Optional)

While the script is interactive, it also exposes helper functions that you can import and use in your own Python code:

- `download_tile(session, url_template, z, x, y, cache_dir=None, retries=3, backoff=0.5, subdomains=None) -> PIL.Image`
- `split_tile_into_quarters(img) -> tuple[PIL.Image, PIL.Image, PIL.Image, PIL.Image]`
- `split_image_grid(img, cols, rows) -> list[tuple[sx, sy, subimage]]`

There is also a `create_pyramid(...)` function designed for a layer-based pyramid workflow, though the default `main()` uses an area-based interactive flow and a custom per-zoom save layout.

---

## License and Attribution

- Map data and tiles are subject to the terms of the respective providers.
- OpenStreetMap data is © OpenStreetMap contributors. Follow the OSM Tile Usage Policy and each provider’s terms.
