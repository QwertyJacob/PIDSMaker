# Engineering Status — Local Docker Install (CPU-only)

## What's been done

- Set up the local Docker install following `docs/docs/ten-minute-install.md`, adapted for a machine with no GPU:
  - `compose-pidsmaker.yml`: removed the `deploy.resources.reservations.devices` NVIDIA GPU reservation block, since there's no `nvidia-container-toolkit`/GPU on this host.
  - `Dockerfile`: switched PyTorch/torchvision/torchaudio and PyTorch Geometric extension wheels from CUDA (`+cu117`) to CPU-only (`+cpu`) builds. Dropped `pyg_lib` (optional PyG accelerator, not directly imported anywhere in the codebase, GPU-oriented).
  - Skipped the CUDA/`nvidia-container-toolkit` install section of the docs entirely — not needed for CPU-only.
- Created `.env` from the provided `.env.local` template, and local `./data` and `./artifacts` directories.
- Built and started both containers:
  - `postgres` (from `compose-postgres.yml`) — healthy.
  - `pidsmaker-pids` (from `compose-pidsmaker.yml`) — built and running.
- Verified inside the `pids` container:
  - `torch==1.13.1+cpu`, `torch.cuda.is_available() == False`, `torch_geometric==2.5.3` import fine.
  - `psycopg2` connects successfully from the `pids` container to the `postgres` container over the internal Docker network (`host=postgres, port=5432` — the framework's own CLI defaults, not the host-mapped `DOCKER_PORT` in `.env`).
- No datasets have been downloaded or loaded yet.

## How to interact now

Both containers should already be running (check with `docker ps`). If they're stopped, bring them back up:

```sh
source .env
docker compose -p postgres -f compose-postgres.yml up -d
docker compose -f compose-pidsmaker.yml up -d
```

Get a shell in the pipeline container:

```sh
docker compose exec pids bash
```

Inside the container, the `pids` conda env is active by default (see `~/.bashrc`). From there you'd normally run e.g.:

```sh
python pidsmaker/main.py SYSTEM DATASET
```

but this requires a loaded dataset in Postgres first (see below).

## Possible next steps

- **Load a dataset.** Nothing is loaded into Postgres yet. Smallest options per the docs are `CLEARSCOPE_E3` (4.8 GB uncompressed) or `CADETS_E3` (10 GB). Requires either:
  - downloading the `.dump` file manually from the [Google Drive folder](https://drive.google.com/drive/folders/1hqfz8__zVqb3QzBuOI2SxrW4lLIdYqFr) into `./data`, or
  - using `./download_datasets.sh DATASET_NAME YOUR_ACCESS_TOKEN` (needs a Google OAuth token, see docs), then
  - `pg_restore -U postgres -h localhost -p 5432 -d DATASET_NAME /data/DATASET_NAME.dump` inside the postgres container (or `./scripts/load_dumps.sh` to load everything present in `./data`).
- **Run a first pipeline** once a dataset is loaded, e.g. `python pidsmaker/main.py kairos CLEARSCOPE_E3` from inside the `pids` container, to confirm the CPU-only setup produces a working end-to-end run (expect it to be considerably slower than GPU).
- **W&B.** Not yet configured — run `wandb login` inside the container if experiment tracking/historization is wanted.
- **Performance expectations.** Since this is CPU-only, training will be much slower than the GPU-oriented defaults in `config/*.yml`. May be worth trying smaller hyperparameters (e.g. reduced `training.node_hid_dim`, fewer epochs) for a first smoke test rather than the full tuned configs.
