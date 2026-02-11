# 05 — Setup Lokal

Panduan ini menjelaskan langkah-langkah menyiapkan environment untuk development.

---

## Prerequisites

Sebelum mulai, pastikan kamu punya:

- 💻 Laptop/PC dengan minimal **8GB RAM** (disarankan 16GB)
- 🌐 Koneksi internet yang stabil
- 🔑 Akses ke repository Git (minta ke tim lead)
- 🔑 API key untuk LLM provider (minta ke tim lead)

---

## 1. Setup OS

### Windows (Menggunakan WSL)

> **Penting:** Untuk Windows, usahakan juga menggunakan WSL. Sebisa mungkin jangan langsung kerja di Windows.

#### Langkah 1: Aktifkan WSL (jika belum)

Buka PowerShell sebagai **Administrator** dan jalankan:

```powershell
# Install WSL dengan Ubuntu (default)
wsl --install

# Restart komputer setelah instalasi selesai
```

Setelah restart, buka **Ubuntu** dari Start Menu. Kamu akan diminta membuat username dan password untuk Linux.

#### Langkah 2: Update Ubuntu

```bash
sudo apt update && sudo apt upgrade -y
```

#### Langkah 3: Install beberapa tools

```bash
# Install tools yang diperlukan
sudo apt install -y build-essential curl wget git unzip
```

> **Tips:** Setelah setup WSL, lanjutkan ke bagian "Install Python" di bawah. Semua command selanjutnya dijalankan di terminal WSL.

### macOS

macOS bisa langsung dipakai. Cukup install **Homebrew** (package manager untuk Mac):

```bash
# Install Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Verifikasi
brew --version
```

### Linux (Ubuntu/Debian)

Linux bisa langsung dipakai. Pastikan sistem up-to-date:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential curl wget git unzip
```

---

## 2. Install Python 3.11+

### Ubuntu / WSL / Debian

```bash
# Tambahkan deadsnakes PPA (untuk versi Python terbaru)
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update

# Install Python 3.11
sudo apt install -y python3.11 python3.11-venv python3.11-dev

# Verifikasi
python3 --version
# Output: Python 3.11.x
```

### macOS

```bash
# Install via Homebrew
brew install python@3.11

# Verifikasi
python3.12 --version
```

---

## 3. Install UV (Package Manager)

```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Reload shell
source ~/.bashrc   # atau ~/.zshrc jika pakai zsh

# Verifikasi
uv --version
# Output: uv 0.x.x
```

---

## 4. Install Docker & Docker Compose

### Ubuntu / WSL

```bash
# Install Docker
curl -fsSL https://get.docker.com | sh

# Tambahkan user ke docker group (supaya tidak perlu sudo)
sudo usermod -aG docker $USER

# Logout dan login kembali supaya setting langsung di-apply
# Atau jalankan: newgrp docker

# Verifikasi
docker --version
docker compose version
```

> **Catatan WSL:** Jika mengalami masalah dengan Docker di WSL, kamu juga bisa install **Docker Desktop for Windows** yang otomatis terintegrasi dengan WSL.

### macOS

Download dan install **Docker Desktop for Mac** dari [docker.com](https://www.docker.com/products/docker-desktop/).

Setelah install, verifikasi:

```bash
docker --version
docker compose version
```

---

## 5. Install Git

### Ubuntu / WSL

```bash
# Biasanya sudah terinstall
git --version

# Kalau belum
sudo apt install -y git
```

### macOS

```bash
# Biasanya sudah terinstall via Xcode Command Line Tools
git --version

# Kalau belum
brew install git
```

### Konfigurasi Git

```bash
# Set nama dan email (gunakan nama dan email kamu)
git config --global user.name "Nama Kamu"
git config --global user.email "email@kamu.com"

# Set default branch name
git config --global init.defaultBranch main

# Verifikasi
git config --list
```

---

## 6. Clone Repository

```bash
# Buat direktori untuk menyimpan berbagai project (opsional)
mkdir -p ~/dev/projects && cd ~/dev/projects

# Clone repository (ganti URL dengan URL yang sebenarnya)
git clone <url-repository> modcus-core
cd modcus-core

# Lihat branch yang tersedia
git branch -a
```

---

## 7. Setup Environment Variables

### Copy file [`.env.example`](../../../.env.example)

```bash
# Copy file .env template
cp .env.example .env
```

### Edit File [`.env`](../../../.env)

Buka file [`.env`](../../../.env) dengan editor seperti VScode dan ganti sesuai yang diperlukan:

```bash
# Buka dengan editor (pilih salah satu)
nano .env           # Editor terminal
code .env           # VS Code (jika terinstall)
```

**Variabel yang perlu diisi:**

```env
DATABASE_URL=postgresql://modcus:modcus@localhost:5432/modcus

# ===== LLM API Key =====
# Minimal isi salah satu
GEMINI_API_KEY=your-gemini-api-key
OPENAI_API_KEY=your-openai-api-key
ANTHROPIC_API_KEY=your-anthropic-api-key

# ===== Redis =====
REDIS_URL=redis://redis:6379/0

# ===== API Security =====
API_KEY=modcus-123456789      # minimal 16 karakter
```

> **Penting:** Jangan commit `.env` ke Git. File ini sudah ada di [`.gitignore`](../../../.gitignore).

### Edit file [`config.yaml`](../../../config.yaml)

Buka file [`config.yaml`](../../../config.yaml) dengan editor seperti VScode dan ganti sesuai yang diperlukan.

Beberapa yang perlu disesuaikan:

```yaml
llm:
   # Sesuaikan dengan API key yang ada di .env
   provider: gemini        # Opsi: vertex, gemini, openai, anthropic
   model: gemini-2.5-flash
   temperature: 0.1        # Range: 0.0 to 2.0

embedding:
   provider: vertex              # Opsi: vertex, openai
   model: gemini-embedding-001
   dimension: 3072               # Range: 128 to 4096, must match model
```

---

## 8. Jalankan dengan Docker Compose

Cara paling mudah untuk menjalankan semua services adalah menggunakan Docker Compose:

### Option A: Jalankan Semua (Recommended untuk Pertama Kali)

```bash
# Build dan jalankan semua services
docker compose up -d --build

# Cek status
docker compose ps

# Cek logs
docker compose logs -f
```

**Services yang akan berjalan:**

| Service       | Port | URL                   |
| ------------- | ---- | --------------------- |
| Ingestion API | 8081 | http://localhost:8081 |
| Query API     | 8080 | http://localhost:8080 |
| PostgreSQL    | 5432 | -                     |
| Redis         | 6379 | -                     |
| Vector DB     | 8000 | -                     |

### Option B: Jalankan Dependencies Dulu, Lalu Service Secara Lokal

Ini lebih cocok untuk development karena kamu bisa langsung edit kode tanpa rebuild Docker:

```bash
# Langkah 1: Jalankan dependencies saja (DB, Redis, Vector Store)
docker compose -f docker-compose.deps.yml up -d

# Langkah 2: Setup virtual environment
cd modcus_api_ingest      # atau modcus_api_query
uv venv
source .venv/bin/activate
uv pip install -e .

# Langkah 3: Jalankan service secara lokal
uvicorn main:app --reload --port 8081
```

---

## 9. Verifikasi Instalasi

Setelah semua services berjalan, verifikasi dengan langkah-langkah berikut:

### Cek Health Endpoints

```bash
# Cek Ingestion Service
curl http://localhost:8081/v1/health
# Expected: {"status": "healthy", ...}

# Cek Query Service
curl http://localhost:8080/v1/health
# Expected: {"status": "healthy", ...}
```

### Cek Swagger Documentation

Buka browser dan akses:

- **Ingestion API Docs:** http://localhost:8081/docs
- **Query API Docs:** http://localhost:8080/docs

Kamu akan melihat halaman Swagger UI yang menampilkan semua endpoint yang tersedia.

### Cek Database Connection

```bash
# Masuk ke container PostgreSQL
docker compose exec db psql -U modcus -d modcus

# Lihat tables
\dt

# Keluar
\q
```

---

## 10. Setup IDE

### VS Code (Recommended)

1. Install VS Code dari [code.visualstudio.com](https://code.visualstudio.com/)
2. Install extensions:
   - **Python** (Microsoft)
   - **Pylance** (Microsoft)
   - **Docker** (Microsoft)
   - **Remote - WSL** (jika pakai WSL)
   - **GitLens** (cukup membantu)

3. Buka project folder:

```bash
# Dari terminal WSL/Linux/Mac
code ~/dev/projects/modcus-core
```

### AI-powered IDE (Optional)

Bisa juga pakai beberapa editor yang sudah ada AI untuk coding seperti:
- Cursor - Install dari [cursor.com](https://cursor.com/download)
- Antigravity - Install dari [antigravity.google](https://antigravity.google/)

---

## Troubleshooting

### Docker Permission Denied

```bash
# Error: permission denied while trying to connect to the Docker daemon
sudo usermod -aG docker $USER
newgrp docker
```

### Port Already in Use

```bash
# Cek proses yang menggunakan port
lsof -i :8080

# Kill proses
kill -9 <PID>
```

### WSL: Docker Daemon Not Running

```bash
# Start Docker daemon di WSL
sudo service docker start

# Atau gunakan Docker Desktop for Windows
```

### Database Connection Error

```bash
# Pastikan container PostgreSQL sudah running
docker compose ps

# Cek logs PostgreSQL
docker compose logs db
```

### Python Version Mismatch

```bash
# Pastikan menggunakan Python 3.11+
python3 --version

# Jika uv menggunakan Python yang salah
uv venv --python python3.11
```

---

## Quick Reference

Berikut command-command yang paling sering digunakan sehari-hari:

```bash
# === Docker ===
docker compose up -d --build           # Start semua services
docker compose down                    # Stop semua services
docker compose logs -f                 # Lihat logs semua services
docker compose logs -f <nama-service>  # Lihat logs service tertentu
docker compose ps                      # Status semua containers

# === Development ===
cd modcus_api_ingest
# atau
cd modcus_api_query

source .venv/bin/activate        # Aktifkan virtual environment
uvicorn main:app --reload        # Jalankan service lokal dengan hot-reload
uv pip install -e .              # Install dependencies

# === Git ===
git status                        # Cek status
git add . && git commit -m "..."  # Commit
git push origin my-branch         # Push

# === Testing ===
pytest                           # Jalankan semua tests
pytest tests/test_parser.py      # Jalankan test tertentu
pytest -v                        # Verbose output
```

---

## Apa Selanjutnya?

Environment kamu sudah siap! Lanjut ke bagian berikutnya untuk mempelajari **workflow pengembangan** sehari-hari:

➡️ [06 — Workflow Pengembangan](06-workflow-pengembangan.md)
