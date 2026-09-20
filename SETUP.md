# FE Infinity — Setup and Running

Everything needed to get from a fresh clone to a playable `.gba`, plus the
importers for adding your own maps and tilesets.

This is the operational guide. `README.md` covers what the project *is*;
this covers what you have to install, what you have to supply, and what to
type.

---

## 1. What you need installed

| Thing | Why | Install |
|---|---|---|
| **Deno** | Runs the entire generation pipeline and the Web UI | <https://deno.com/> |
| **mono** | Runs `ColorzCore.exe`, the Event Assembler that actually writes the ROM | `sudo apt install mono-complete` (Debian/Ubuntu/WSL), `brew install mono` (macOS) |
| **Python 3** | Eight of the assemble scripts are Python (definitions, tables, text, music, maps, graphics) | Usually already present; `sudo apt install python3` otherwise |
| **A way to run `tmx2ea.exe`** | Converts Tiled maps into Event Assembler format. It is a Windows binary with no Linux equivalent in this repo | **WSL:** nothing to do, Windows interop runs it. **Native Linux/macOS:** install `wine` |
| **A GBA emulator** | To play the result | mGBA, VBA-M, whatever you like — optional, the build just hands off the file |

You do **not** need wine on WSL. The build tries direct execution first and
only falls back to wine if that fails, and it detects the failure by output
text rather than exit code, because different systems report "couldn't
execute this .exe" in different ways.

You do **not** need wine for the tileset compressor either — there's a
Python implementation in `romBuilder/LinuxTools/compress` that produces
identical GBA LZ77 output.

### A clean FE8 ROM

The build patches a real Fire Emblem: The Sacred Stones ROM. It is not in
this repo and never will be.

Put yours at:

```
romBuilder/Clean.gba
```

Exact filename, exact location. `run.sh` and `AssembleAll.sh` both check
for it and stop immediately if it's missing.

The build chain is: `Clean.gba` → `makeAnims.sh` → `WithAnimations.gba` →
`MAKEHACK.sh` → **`hack.gba`** (your finished romhack). The two
intermediate files are generated; you only supply `Clean.gba`.

---

## 2. Applying the patch

The patch is a single file containing every fix. Apply it to a clean
clone of upstream.

```bash
git clone https://github.com/i-am-neon/fe-infinity.git
cd fe-infinity
git apply --binary /path/to/fe-infinity-update.patch
```

`--binary` is required. The patch carries binary files (the ported
tileset configs and the Linux tool builds), and without that flag git
will refuse or silently mangle them.

### Checking it worked

```bash
git apply --check --binary /path/to/fe-infinity-update.patch
```

Run this *before* applying. Silence means it will apply cleanly; any
output means it won't, and nothing has been changed yet.

### If you get "type 100644, expected 100755" warnings

Harmless. That's git noting a file's executable bit differs. Every shell
script in the build is invoked as `bash script.sh` rather than
`./script.sh` specifically so the executable bit doesn't matter — that
bit gets lost routinely on checkouts that have passed through Windows
tooling.

### Re-applying after a new patch

Patches are cumulative against upstream, not against each other. To take
a newer one, start from a clean clone again rather than stacking:

```bash
git clone https://github.com/i-am-neon/fe-infinity.git fe-infinity-new
cd fe-infinity-new
git apply --binary /path/to/newer.patch
cp ../fe-infinity/romBuilder/Clean.gba romBuilder/
cp ../fe-infinity/server/.env server/                        # your API key / Ollama settings
cp ../fe-infinity/server/webui/prompts/prompt-overrides.json server/webui/prompts/  # if you edited prompts
```

Those three carried-over files are the only state worth keeping — the
first is your ROM, the other two are gitignored on purpose.

---

## 3. Running it

### The Web UI (easiest)

```bash
cd server
deno task webui
```

Open <http://localhost:8787>. Three tabs:

- **Setup** — choose OpenAI or Ollama, enter your key or URL. Writes
  `server/.env`.
- **Prompts** — every system prompt in the pipeline, editable, with
  "Reset to default". Saved to
  `server/webui/prompts/prompt-overrides.json`, applied on your next run,
  no restart.
- **Generate** — describe a world, pick a chapter count, watch the log
  stream. Tick **skip ROM build** to generate story and character data
  only — useful for iterating on prompts without needing mono or a ROM.

Set a different port with `WEBUI_PORT=9000 deno task webui`.

### The command line

```bash
cd server
deno task main
```

Generates a game and builds the ROM in one pass.

To build a ROM from data that's already been generated:

```bash
cd romBuilder
bash run.sh
```

On success you get `romBuilder/hack.gba` and the script tries to open it
in whatever your system associates with `.gba`. If it can't figure that
out it just prints the path.

### Trying the build without burning any tokens

```bash
cd server
deno task write-test
cd ../romBuilder
bash run.sh
```

This writes the sample game in `server/testData/test-game-obj.ts` to the
rom builder with no AI calls at all. It's the fastest way to confirm your
mono / Python / ROM setup works before involving a model.

---

## 4. Choosing a model

Set this in the Web UI's Setup tab, or by hand in `server/.env` — copy
`server/.env.example` to start.

### OpenAI

```
AI_PROVIDER=openai
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o          # optional; defaults to gpt-4o-mini
```

`gpt-4o` is slower and costs more but writes noticeably better stories
and follows the schemas more reliably.

### Ollama

```
AI_PROVIDER=ollama
OLLAMA_MODEL=qwen3-fe
OLLAMA_BASE_URL=http://localhost:11434/v1
```

Three things matter here, and all three bite:

**Ollama 0.5 or newer.** The pipeline relies on JSON-schema-constrained
decoding, which landed in 0.5. On older versions the schema is dropped
and the model is merely *asked* for the right shape — local models handle
that badly. Check with `ollama --version`.

**The context window.** Ollama defaults to 4,096 tokens and this pipeline
routinely exceeds that (a unit-placement prompt includes the map's whole
terrain grid). Ollama does not error on an over-long prompt — it silently
truncates, and the model then returns nonsense that fails validation.
Build a variant with a real context window:

```bash
cat > Modelfile <<'EOF'
FROM qwen3:14b
PARAMETER num_ctx 32768
EOF

ollama create qwen3-fe -f Modelfile
```

Then point `OLLAMA_MODEL` at `qwen3-fe`. Adjust `num_ctx` to your VRAM —
32k is comfortable, 16k workable, 8k marginal.

**WSL can't reach Windows `localhost`.** If Ollama runs on Windows and
this runs in WSL, they're in separate network namespaces. Don't install a
second Ollama inside WSL — you'd re-download every model and have to sort
out CUDA passthrough. Instead:

1. On Windows, set `OLLAMA_HOST=0.0.0.0` and restart Ollama
   (`setx OLLAMA_HOST "0.0.0.0"`, then restart the app).
2. Allow port 11434 through Windows Firewall on private networks.
3. The project auto-detects the Windows host from WSL's default route, so
   it often just works. If not, run `ipconfig` on Windows, take the
   vEthernet (WSL) adapter's IPv4 address, and set
   `OLLAMA_BASE_URL=http://<that-ip>:11434/v1`.

Run all `ollama` commands (`pull`, `create`, `list`) in PowerShell on
Windows, not in WSL.

A run checks connectivity at startup and, if it fails, prints which
addresses it tried — and if it finds Ollama somewhere other than the
configured address, tells you the correct `OLLAMA_BASE_URL`.

---

## 5. Adding maps

Drop Tiled `.tmx` files into `server/map-processing/maps-to-import/` and:

```bash
cd server
deno task import-maps
```

Each map gets parsed for dimensions, terrain, and points of interest
(chests, villages, thrones, stairs), split into named regions the AI can
reason about, and added to the pool chapters are drawn from. Already
imported maps are skipped, so re-running is safe. Use `--force` to redo
them:

```bash
deno task import-maps -- --force
deno task import-maps -- --dir /some/other/folder
```

### Format

**`.tmx` only.** The build runs maps through `tmx2ea`, which reads Tiled
files exclusively. FEMapCreator can export `.map` and `.mar` as well —
those aren't readable here. Export `.tmx`, or open the file in
[Tiled](https://www.mapeditor.org/) and save as `.tmx`.

All of Tiled's layer encodings work: base64 with zlib or gzip, plain
base64, CSV, and raw XML `<tile gid>` elements. The layer can be named
`Main` or Tiled's default `Tile Layer 1`, and a single-layer map is
accepted whatever its layer is called.

### Tilesets

The tileset named inside the `.tmx` has to be one of two things:

1. **A stock FE8 tileset**, named with its 8 hex IDs — `3C00CE3E` means
   ObjectType `3C`, PaletteID `CE`, TileConfig `3E`. A matching
   `romBuilder/Maps/Tilesets/3C00CE3E.png` must exist. Descriptive names
   work as long as the hex is in there: `FE7 - Fort - 16007718` and
   `16 00 77 18` both resolve.

2. **A ported tileset**, named after one of these:

   | Name | From | What it is |
   |---|---|---|
   | `DesertTemple` | FE6 | Desert temple / bastion |
   | `OutdoorSnowy` | FE6 | Snowy outdoor |
   | `IndoorSnowy` | FE6 | Snowy indoor |
   | `WesternIsles` | FE7 | Western Isles, extended |
   | `ImprovedCastle` | FE8 | Castle with tile animations |

   Matching is loose: it needs every word of the name, in any order. So
   `OutdoorSnowy`, `Outdoor Snowy`, `FE6 Snowy Outdoor` and
   `FE6 - Snowy - Outdoor` all find the same tileset. It won't
   over-match — `FE8 - Castle` does *not* resolve to `ImprovedCastle`,
   since "improved" is missing.

A map whose tileset fails both checks is skipped with an explanation
rather than imported. That's deliberate — a missing tileset produces a
chapter with corrupted graphics, which is far harder to diagnose later
than a message now.

### Area names

Areas come straight from the tile data — no AI call, so this works on a
local Ollama setup and costs nothing. You get names like "Northwest
Throne Room" with coordinates that are correct by construction.

`--use-ai` instead has an OpenAI vision model describe them (more
evocative, needs `OPENAI_API_KEY` and a `.png` beside the `.tmx`).

---

## 6. Tileset and terrain data

Two importers, both rarely needed.

### Refreshing terrain from FEMapCreator

FEMapCreator ships `Tileset_Data.xml` and `Terrain_Data.xml`. These
regenerate the terrain lookup tables — which is how the AI reads what's
on a map.

Put the XMLs anywhere and pass their paths:

```bash
cd server
deno task import-tileset-data -- \
  --tileset-data map-processing/Tileset_Data.xml \
  --terrain-data map-processing/Terrain_Data.xml
```

**Worth running once.** Without `--terrain-data`, 26 of the 41 stock
tilesets have terrain IDs with no name (39, 40, 41, 24, 47 and others),
and those tiles reach the unit-placement AI as `Unknown Terrain 39`. It
still works, it's just reasoning about opaque labels.

Regenerates `server/map-processing/lookup-tables/tileset-id-to-terrain.ts`
and `server/map-processing/lookup-tables/terrain-id-to-name.ts`. Existing
hand-curated terrain names are merged rather than replaced.

### Re-deriving ported tileset terrain

```bash
cd server
deno task import-ported-tileset-terrain
```

Reads the terrain out of each ported tileset's own `.mapchip_config` —
the last 0x400 bytes are its 1024 terrain tags. Only needed if you change
one of those configs or add a new ported tileset.

### Rebuilding ported tileset binaries

```bash
bash romBuilder/Maps/Tilesets/NewTilesets/AssembleNewTilesets.sh
```

Compresses each tileset's mapchip config into the `.dmp` files that
`romBuilder/Maps/Tilesets/NewTilesets/TilesetInstaller.event` includes.
Only needed if you edit a config or add a tileset. Re-run
`deno task import-ported-tileset-terrain` afterwards so the terrain
lookup matches the new data.

It refuses to overwrite a good `.dmp` with a failed build, which is worth
knowing because that's exactly how these files ended up as 0 bytes in the
first place: the original used `wine compress.exe x >x.dmp`, and `>`
truncates the output file *before* the command runs.

---

## 7. Command reference

All `deno task` commands run from `server/`.

| Command | What it does |
|---|---|
| `deno task webui` | Web UI on :8787 |
| `deno task main` | Generate a game and build the ROM |
| `deno task write-test` | Write sample data to the rom builder, no AI |
| `deno task test` | Run the test suite |
| `deno task import-maps` | Import `.tmx` maps from `maps-to-import/` |
| `deno task import-tileset-data` | Regenerate terrain tables from FEMapCreator XMLs |
| `deno task import-ported-tileset-terrain` | Re-derive ported tileset terrain from their configs |
| `bash romBuilder/run.sh` | Build the ROM from already-generated data |

---

## 8. When things break

**`Clean.gba not found`** — you haven't put an FE8 ROM at
`romBuilder/Clean.gba`. See section 1.

**`mono: command not found`**, or a message telling you to install it —
`sudo apt install mono-complete`.

**`Exec format error`** or **`cannot execute: required file not found`**
on a `.sh` — a line-ending or executable-bit problem from Windows
tooling. Every script is invoked as `bash x.sh` to avoid this; if you hit
it anyway, you're calling a script directly. Use `bash` explicitly.

**`run-detectors: unable to find an interpreter for ./tmx2ea.exe`** — WSL
couldn't run the Windows binary directly. It should fall back to wine
automatically; if wine isn't installed, install it or run the build from
Windows.

**`Undefined identifier: <something>`** during assembly — a generated
`.event` file references a ROM symbol that doesn't exist. Usually a class
name or a map property. The name in the message is the thing to search
for.

**`Errors occurred; no changes written.`** — Event Assembler refused the
whole build. Scroll up; the real error is above this line, and there may
be several.

**Ollama returns `{}` or echoes the schema back** — almost always one of
the three Ollama issues in section 4: version below 0.5, context window
too small, or the model isn't the enlarged-context variant you built.

**A run dies during generation with a schema validation error** — the
model returned something that didn't fit. Each call retries with
corrective feedback, so a single failure self-corrects; exhausting all
retries usually means the model is too small or its context is truncated.

---

## Credits

Buildfiles adapted from
[Legends of Avenir](https://github.com/Snakey11/Legends-of-Avenir).
Portraits by
[Kanna](https://github.com/Klokinator/FE-Repo/tree/main/Portrait%20Repository/Spriting%20Community%20OC's%20(Grouped%20by%20Artist)/Kanna).
