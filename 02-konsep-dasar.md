# 02 — Konsep Dasar

Sebelum mulai bekerja di project ini, ada beberapa konsep dasar yang perlu dipahami. Bagian ini akan menjelaskan masing-masing secara singkat.

---

## 🐧 WSL & Linux

### Apa itu Linux?

**Linux** adalah sistem operasi (OS) yang banyak digunakan di server dan dunia software development. Kebanyakan server di dunia menggunakan Linux. Project Modcus juga menggunakan environment Linux.

### Apa itu WSL?

**WSL (Windows Subsystem for Linux)** adalah fitur di Windows yang memungkinkan kita menjalankan Linux *di dalam* Windows tanpa perlu dual-boot atau virtual machine yang berat. Dengan WSL, kita bisa menggunakan terminal Linux langsung di Windows. Kita juga bisa menyambungkan VSCode (atau editor lain) langsung ke environment WSL.

**Kenapa perlu WSL?**

- Docker, Python, dan banyak tools lain lebih baik jika digunakan di Linux
- Proyek ini didesain untuk environment Linux
- Command dan beberapa script di project ini menggunakan syntax Linux

### Command Dasar

Beberapa command yang paling sering digunakan (anggap cheat-sheet kalau perlu).

```bash
# Navigasi file dan folder
ls                  # Lihat isi folder saat ini
ls -la              # Lihat isi folder dengan detail (termasuk hidden files)
cd nama-folder      # Masuk ke folder
cd ..               # Keluar satu level ke atas
cd ~                # Kembali ke home directory
pwd                 # Tampilkan path folder saat ini

# Manipulasi file dan folder
mkdir nama-folder   # Buat folder baru
touch nama-file     # Buat file kosong
cp file1 file2      # Copy file
mv file1 file2      # Pindahkan/rename file
rm nama-file        # Hapus file
rm -rf nama-folder  # Hapus folder beserta isinya (hati-hati!)

# Membaca file
cat nama-file       # Tampilkan seluruh isi file
head -n 20 file     # Tampilkan 20 baris pertama
tail -n 20 file     # Tampilkan 20 baris terakhir

# Lainnya
echo "hello"        # Print teks ke terminal
grep "kata" file    # Cari teks dalam file
```

---

## 🔀 Git (Version Control)

### Apa itu Git?

**Git** adalah sistem *version control* — alat untuk melacak perubahan/changes pada source code sebuah project.
### Konsep Penting

| Konsep                | Penjelasan                                                                 |
| --------------------- | -------------------------------------------------------------------------- |
| **Repository (Repo)** | Folder proyek yang dilacak oleh Git (folder paling luar)                   |
| **Commit**            | "Snapshot" atau checkpoint dari perubahan source code                      |
| **Branch**            | Cabang terpisah untuk mengerjakan fitur tanpa mengganggu source code utama |
| **Merge**             | Menggabungkan satu branch ke branch lain                                   |
| **Pull Request (PR)** | Request untuk merge branch kamu ke branch utama (biasanya perlu review)    |
| **Clone**             | Mendownload repo dari remote (misalnya GitHub) ke folder lokal             |
| **Push**              | Mengirim commit dari lokal ke remote                                       |
| **Pull**              | Mengambil perubahan terbaru dari remote                                    |

### Command Git Dasar

Sekali lagi, anggap saja cheat-sheet kalau sewaktu-waktu perlu.

```bash
# Setup awal
git clone <url-repo>          # Clone repository
git status                    # Lihat status perubahan
git log --oneline -10         # Lihat 10 commit terakhir

# Workflow harian
git checkout -b nama-branch   # Buat dan pindah ke branch baru
git add .                     # Stage semua perubahan
git commit -m "feat: add X"   # Commit dengan pesan
git push origin nama-branch   # Push ke remote

# Sinkronisasi
git pull origin main          # Ambil perubahan terbaru dari main
git merge main                # Merge main ke branch saat ini
```

### Conventional Commits

Di sini, kita menggunakan format khusus untuk menulis pesan commit. Istilahnya adalah **Conventional Commits**. Formatnya adalah:

```
<type>: <deskripsi singkat>
```

**Type yang digunakan:**

| Type       | Kapan Digunakan                      | Contoh                              |
| ---------- | ------------------------------------ | ----------------------------------- |
| `feat`     | Menambah fitur baru                  | `feat: add new llm provider`        |
| `fix`      | Memperbaiki bug                      | `fix: fix v1 endpoints`             |
| `docs`     | Perubahan dokumentasi saja           | `docs: update onboarding guide`     |
| `refactor` | Refactoring kode tanpa ubah behavior | `refactor: simplify query pipeline` |
| `chore`    | Task lainnya (config, deps, dll.)    | `chore: update dependencies`        |
| `test`     | Menambah atau memperbaiki test       | `test: add unit tests for parser`   |

**Contoh commit message yang baik:**

```
feat: add vertex ai llm provider support
fix: resolve foreign key violation in llm logger
docs: add onboarding documentation for new developers
```

---

## 🐍 Python

### Apa itu Python?

**Python** adalah bahasa pemrograman yang digunakan di project ini. Python dikenal karena syntax-nya yang mudah dibaca dan ekosistem library-nya yang sangat luas, terutama untuk AI dan data science.

Jadi, alasan utama kita menggunakan Python untuk sistem Modcus adalah karena banyak framework/library di domain AI yang tersedia.

**Versi yang digunakan:** Python 3.11+

### Konsep Python yang Relevan

Berikut beberapa konsep Python yang banyak digunakan di proyek ini.

#### 1. Type Hints

Semua variabel, parameter, function **wajib** disertai dengan type hint atau tipe data. Contoh:
- ❌ `name = "modcus"`
- ✅ `name: str = "modcus"`

#### 2. Docstring

Biasakan untuk memberikan Docstring atau dokumentasi singkat untuk semua function, class, atau komponen lain.

```python
def calculate_pe_ratio(price: float, earnings: float) -> float:
    """Menghitung P/E ratio. <--- ini adalah contoh Docstring

    Args:
        price: Harga saham saat ini
        earnings: Earnings per share

    Returns:
        Nilai P/E ratio
    """
    return price / earnings
```

#### 3. Async/Await

Untuk beberapa operasi, bisa menggunakan async supaya non-blocking (tidak mmem-block proses utama).

```python
async def fetch_data(url: str) -> dict:
    response = await http_client.get(url)
    return response.json()
```

#### 4. Class atau Pydantic Model

Usahakan untuk selalu menggunakan Class atau mendefinisikan struktur data menggunakan Pydantic Model.

```python
from pydantic import BaseModel

class JobResponse(BaseModel):
    job_id: str
    status: str
    progress: float
```

#### 5. Virtual Environment

Biasakan menggunakan **Virtual environment (venv)** atau lingkungan Python yang terisolasi. Jangan gunakan instalasi Python global.

Setiap proyek punya `venv` sendiri agar dependency-nya tidak saling bertabrakan. Bahkan, masing-masing module di project ini punya `venv` masing-masing.

Di project ini, kita spesifik menggunakan sebuah tools bernama **UV** untuk me-manage `venv` dan meng-install dependency.

```bash
# Membuat virtual environment (menggunakan uv)
uv venv

# Mengaktifkan virtual environment
source .venv/bin/activate     # Linux/Mac
.venv\Scripts\activate        # Windows (tanpa WSL)

# Menginstall dependency atau libary baru
uv add nama-library

# Menginstall ulang semua dependency
uv sync
```

Semua dependency akan dicatat dalam sebuah file bernama `pyproject.toml` seperti ini --> [`modcus_api_ingest/pyproject.toml](../../../modcus_api_ingest/pyproject.toml).

---

## 📦 UV (Package Manager)

### Apa itu UV?

Seperti dijelaskan di atas, **UV** adalah package manager atau tool yang menggantikan `pip`. UV jauh **lebih cepat** dari pip dan lebih reliable dalam mengelola dependency.

**Kenapa UV, bukan pip?**

| Aspek                  | pip                         | UV                   |
| ---------------------- | --------------------------- | -------------------- |
| Kecepatan install      | Lambat                      | Lebih cepat          |
| Dependency resolution  | Kadang konflik              | Lebih reliable       |
| Lock file              | Tidak built-in              | Built-in (`uv.lock`) |
| Virtual env management | Terpisah (`python -m venv`) | Built-in (`uv venv`) |

### Perintah UV yang Sering Digunakan

```bash
# Membuat virtual environment
uv venv

# Install dependencies dari pyproject.toml
uv sync

# Install package tertentu
uv add requests

# Melihat packages yang terinstall
uv pip list
```

---

## 🐳 Docker & Docker Compose

### Apa itu Docker?

**Docker** adalah teknologi *containerization* — cara untuk "membungkus" aplikasi beserta semua dependency-nya ke dalam sebuah paket yang disebut **container**. Container ini bisa berjalan di mana saja (laptop kita, server, cloud). Kalau code ini bisa di-run di 1 device, kita bisa expect code yang sama juga bisa di-run di device lain.

### Konsep Docker

| Konsep         | Penjelasan                                                                             |
| -------------- | -------------------------------------------------------------------------------------- |
| **Image**      | "Blueprint" atau template untuk membuat container.                                     |
| **Container**  | Sebuah instance yang dibuat dengan menjalankan sebuah image                            |
| **Dockerfile** | Instruksi untuk membuat sebuah image                                                   |
| **Volume**     | Penyimpanan data yang persistent (tidak hilang saat container di-stop atau di-restart) |

### Apa itu Docker Compose?

**Docker Compose** adalah tool untuk menjalankan **banyak container sekaligus** dengan satu command. Di project ini, kita butuh beberapa container yang berjalan bersamaan (API services, database, Redis, vector database), dan Docker Compose mengatur semuanya.

### Perintah Docker Compose yang Sering Digunakan

```bash
# Menjalankan semua services di background
docker compose up -d

# Sama, tapi dengan melakukan build ulang (biasanya kalau ada kodingan baru)
docker compose up -d --build

# Melihat status container
docker compose ps

# Melihat log container
docker compose logs -f           # Semua container
docker compose logs -f api       # Container tertentu

# Menghentikan semua services
docker compose down

# Menghentikan dan hapus volumes (reset data) <-- hati-hati menggunakan ini
docker compose down -v
```

### Docker di Modcus

Proyek Modcus memiliki beberapa file Docker Compose:

| File                          | Kegunaan                                          |
| ----------------------------- | ------------------------------------------------- |
| `docker-compose.yml`          | All-in-one (semua services jalan sekaligus)       |
| `docker-compose.deps.yml`     | Beberapa container saja (DB, Redis, vector store) |
| `docker-compose.dev.yml`      | Development mode dengan hot-reload                |
| `docker-compose.ingest.yml`   | Ingestion service saja                            |
| `docker-compose.query.yml`    | Query service saja                                |
| `docker-compose.postgres.yml` | PostgreSQL saja                                   |
| `docker-compose.redis.yml`    | Redis saja                                        |
| `docker-compose.vector.yml`   | Vector database saja (ChromaDB, Qdrant)           |

```bash
# Menjalankan file Docker Compose tertentu
docker compose -f docker-compose.dev.yml up -d
```

---

## 🌐 API & REST

### Apa itu API?

**API (Application Programming Interface)** adalah cara aplikasi berkomunikasi satu sama lain menggunakan HTTP. Masing-masing service di project ini berkomunikasi satu sama lain dengan memanfaatkan API.

### Apa itu REST?

**REST (Representational State Transfer)** adalah gaya arsitektur untuk membangun sebuah API. REST menggunakan HTTP methods yang sudah dikenal:

| Method   | Kegunaan          | Contoh                      |
| -------- | ----------------- | --------------------------- |
| `GET`    | Mengambil data    | `GET /v1/health`            |
| `POST`   | Membuat data baru | `POST /v1/ingestion/upload` |
| `PUT`    | Mengupdate data   | `PUT /v1/jobs/123`          |
| `DELETE` | Menghapus data    | `DELETE /v1/admin/kb/BBCA`  |

### Contoh Request & Response

```bash
# Request
curl -X POST http://localhost:8080/v1/query/chat \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "query": "Apa pendapatan BBCA tahun 2024?",
    "mode": "company-analysis",
    "level": "novice"
  }'

# Response (JSON)
{
  "answer": "Pendapatan BBCA pada tahun 2024 mencapai Rp 100 triliun [BBCA-AR-2024]",
  "sources": [
    {"doc_id": "BBCA_AR_2024", "ticker": "BBCA", "chunk_id": "17"}
  ],
  "metadata": {
    "mode": "company-analysis",
    "level": "novice",
    "tickers": ["BBCA"]
  }
}
```

### HTTP Status Codes

| Code  | Arti                           | Kapan Terjadi             |
| ----- | ------------------------------ | ------------------------- |
| `200` | OK — berhasil                  | Request sukses            |
| `201` | Created — berhasil dibuat      | Data baru berhasil dibuat |
| `400` | Bad Request — input salah      | Parameter tidak valid     |
| `401` | Unauthorized — tidak ada akses | API key tidak ada/salah   |
| `404` | Not Found — tidak ditemukan    | Resource tidak ada        |
| `500` | Internal Server Error          | Ada error di server       |

---

## Apa Selanjutnya?

Sekarang kamu sudah paham konsep-konsep dasar yang diperlukan. Lanjut ke bagian berikutnya untuk memahami **arsitektur sistem Modcus**:

➡️ [03 — Arsitektur Sistem](03-arsitektur-sistem.md)
