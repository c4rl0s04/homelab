# tools

Small everyday tools with no shared state, so they can be stopped or redeployed without touching anything else.

| Service | What it does | Name | Login |
| --- | --- | --- | --- |
| [PairDrop](https://github.com/schlagmichdoch/PairDrop) | Send files between devices on the home network, like AirDrop | `drop.casa.home` → `pairdrop:3000` | None: anyone on the home network can use it |
| [Stirling-PDF](https://github.com/Stirling-Tools/Stirling-PDF) | Merge, split, compress, OCR and sign PDFs locally | `pdf.casa.home` → `stirling-pdf:8080` | Tinyauth (its own login is off) |

## Deploy with Dockhand

Git stack `tools`: repository `homelab`, branch `main`, compose file `stacks/tools/compose.yaml`. Stack variables (no secrets):

| Variable | Windows PC value |
| --- | --- |
| `DATA_ROOT` | `E:/homelab/data` |
| `TZ` | `Europe/Madrid` |

Optional: `PAIRDROP_PORT` (3005), `STIRLING_PDF_PORT` (3006), both on `127.0.0.1` only.

npm proxy hosts:

- `drop.casa.home` → `http://pairdrop:3000`, websockets on, no Tinyauth. PairDrop groups devices by the client address npm passes in `X-Forwarded-For`; private addresses all count as one network, so every device at home sees the others.
- `pdf.casa.home` → `http://stirling-pdf:8080`, with the Tinyauth block from `stacks/infra/README.md`.

## Notes

- PairDrop has no data. Files go device to device over WebRTC; the server only introduces them. Over plain HTTP a few browser extras are unavailable (copy button, installing it as an app, notifications); sending files works.
- Stirling-PDF keeps settings in `${DATA_ROOT}/stirling-pdf/configs` and saved files in `${DATA_ROOT}/stirling-pdf/storage`, backed up with the rest of `DATA_ROOT`. It is a Java app: the first start takes about a minute, and it is capped at 4 GB of memory.
- OCR languages live in `${DATA_ROOT}/stirling-pdf/tessdata`, which replaces the image's folder. The image has no Spanish, so on a new host create it before deploying:

  ```bash
  docker cp stirling-pdf:/usr/share/tesseract-ocr/5/tessdata/. "$DATA_ROOT/stirling-pdf/tessdata/"   # from a first deploy without the mount
  curl -fL -o "$DATA_ROOT/stirling-pdf/tessdata/spa.traineddata" https://github.com/tesseract-ocr/tessdata_fast/raw/main/spa.traineddata
  ```

  It is part of `DATA_ROOT`, so a restored backup already has it.

## Checklist

- [x] Deployed as a Git stack (2026-11-15); `drop.casa.home` answers (websocket upgrade 101) and `pdf.casa.home` redirects to Tinyauth; ports 3005/3006 are not reachable from the LAN.
- [ ] PairDrop: a phone and the PC see each other and a file goes through.
- [ ] Stirling-PDF: opens without its own login behind Tinyauth; merge two PDFs; OCR a scanned page in Spanish.
- [x] Portal cards and Uptime Kuma monitors added (2026-11-15, both green).
- [x] Spanish OCR: `tesseract --list-langs` in the container lists `spa` (PR #25).
