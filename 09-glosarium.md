# 09 — Glosarium

Daftar istilah dan terminologi yang sering digunakan di proyek Modcus, dikelompokkan per kategori.

---

## 🤖 AI / Machine Learning

| Istilah                                  | Penjelasan                                                                                            |
| ---------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| **LLM (Large Language Model)**           | Model AI besar yang bisa memahami dan menghasilkan teks (contoh: GPT-4, Claude, Gemini)               |
| **RAG (Retrieval-Augmented Generation)** | Teknik yang menggabungkan pencarian informasi (retrieval) dengan pembuatan teks (generation) oleh LLM |
| **Embedding**                            | Representasi numerik (vector) dari teks. Teks dengan arti mirip punya embedding yang berdekatan       |
| **Vector**                               | Array angka yang merepresentasikan data. Dalam konteks NLP, merepresentasikan makna teks              |
| **Chunking**                             | Memotong dokumen panjang menjadi bagian-bagian kecil (chunks) untuk diproses                          |
| **Token**                                | Unit terkecil yang diproses oleh LLM. Kira-kira 1 token = 3/4 kata dalam bahasa Inggris               |
| **Prompt**                               | Instruksi atau pertanyaan yang dikirim ke LLM                                                         |
| **System Prompt**                        | Instruksi khusus yang menentukan perilaku dan persona LLM                                             |
| **Context Window**                       | Jumlah maksimal token yang bisa diproses LLM dalam satu kali panggilan                                |
| **Inference**                            | Proses LLM menghasilkan output berdasarkan input yang diberikan                                       |
| **Grounding**                            | Memastikan jawaban LLM berdasarkan data/fakta yang ada, bukan halusinasi                              |
| **Hallucination**                        | Ketika LLM menghasilkan informasi yang tidak benar atau tidak ada di data sumber                      |
| **Citation**                             | Referensi ke sumber data yang digunakan untuk menghasilkan jawaban                                    |
| **Knowledge Base (KB)**                  | Kumpulan dokumen dan data yang menjadi sumber pengetahuan sistem                                      |
| **Agent**                                | Komponen AI otonom yang bisa mengambil keputusan dan melakukan aksi dalam pipeline                    |
| **Guardrail**                            | Mekanisme pengaman yang membatasi perilaku LLM agar tetap dalam domain yang diinginkan                |

---

## 🏗️ Backend & Arsitektur

| Istilah                                     | Penjelasan                                                                            |
| ------------------------------------------- | ------------------------------------------------------------------------------------- |
| **API (Application Programming Interface)** | Antarmuka untuk komunikasi antar aplikasi                                             |
| **REST (Representational State Transfer)**  | Gaya arsitektur untuk membangun API menggunakan HTTP methods                          |
| **Endpoint**                                | URL tertentu di API yang melayani request tertentu (contoh: `/v1/query/chat`)         |
| **Monorepo**                                | Satu repository Git yang berisi kode untuk beberapa service/proyek                    |
| **Microservice**                            | Arsitektur di mana aplikasi dipecah menjadi service-service kecil yang independen     |
| **Service**                                 | Komponen aplikasi yang berjalan independen (contoh: Ingestion Service, Query Service) |
| **Middleware**                              | Kode yang berjalan di antara request masuk dan response keluar (contoh: auth, CORS)   |
| **Dependency Injection (DI)**               | Pola desain di mana dependency diberikan dari luar, bukan dibuat di dalam fungsi      |
| **ORM (Object-Relational Mapper)**          | Library yang memetakan objek Python ke tabel database (contoh: SQLModel)              |
| **Migration**                               | Script yang mengubah skema database (menambah tabel, kolom, dll.)                     |
| **CORS (Cross-Origin Resource Sharing)**    | Mekanisme keamanan browser yang mengontrol akses antar domain                         |
| **Health Check**                            | Endpoint untuk mengecek apakah service berjalan normal                                |
| **Idempotent**                              | Operasi yang menghasilkan hasil yang sama meskipun dijalankan berulang kali           |
| **Payload**                                 | Data yang dikirim dalam body HTTP request                                             |
| **DTO (Data Transfer Object)**              | Model data yang digunakan untuk transfer data antar layer                             |

---

## 🗄️ Database

| Istilah                             | Penjelasan                                                                                   |
| ----------------------------------- | -------------------------------------------------------------------------------------------- |
| **PostgreSQL (Postgres)**           | Database relasional open-source yang digunakan proyek ini                                    |
| **SQL (Structured Query Language)** | Bahasa standar untuk berinteraksi dengan database relasional                                 |
| **Schema**                          | Struktur/blueprint database (tabel, kolom, tipe data, relasi)                                |
| **Table**                           | Kumpulan data yang terstruktur dalam baris dan kolom                                         |
| **Primary Key (PK)**                | Kolom unik yang mengidentifikasi setiap baris dalam tabel                                    |
| **Foreign Key (FK)**                | Kolom yang merujuk ke primary key di tabel lain (relasi antar tabel)                         |
| **Index**                           | Struktur data untuk mempercepat pencarian di database                                        |
| **ACID**                            | Prinsip database: Atomicity, Consistency, Isolation, Durability                              |
| **Transaction**                     | Serangkaian operasi database yang dieksekusi secara atomik (semua berhasil atau semua gagal) |
| **Vector Database**                 | Database khusus untuk menyimpan dan mencari vector embeddings                                |
| **pgvector**                        | Extension PostgreSQL untuk menyimpan vector embeddings                                       |
| **ChromaDB**                        | Vector database open-source, ringan, cocok untuk development                                 |
| **Qdrant**                          | Vector database high-performance untuk production                                            |

---

## 🐳 DevOps & Infrastructure

| Istilah                                                  | Penjelasan                                                                |
| -------------------------------------------------------- | ------------------------------------------------------------------------- |
| **Docker**                                               | Platform untuk menjalankan aplikasi dalam container yang terisolasi       |
| **Container**                                            | Paket ringan yang berisi aplikasi dan semua dependency-nya                |
| **Image**                                                | Template/blueprint untuk membuat container                                |
| **Dockerfile**                                           | File instruksi untuk membuild Docker image                                |
| **Docker Compose**                                       | Tool untuk menjalankan beberapa container sekaligus                       |
| **Volume**                                               | Penyimpanan data yang persist meskipun container restart                  |
| **WSL (Windows Subsystem for Linux)**                    | Fitur Windows untuk menjalankan Linux di dalamnya                         |
| **CI/CD (Continuous Integration/Continuous Deployment)** | Otomatisasi build, test, dan deploy                                       |
| **Environment Variable**                                 | Variabel yang di-set di luar kode, biasanya untuk konfigurasi dan secrets |
| **Redis**                                                | In-memory data store, digunakan sebagai message broker untuk Celery       |
| **Celery**                                               | Framework Python untuk background task processing                         |
| **Worker**                                               | Proses yang mengambil dan menjalankan task dari queue                     |
| **Message Broker**                                       | Perantara pesan antara producer dan consumer (Redis di proyek ini)        |

---

## 📊 Keuangan (Domain)

| Istilah                                    | Penjelasan                                                                |
| ------------------------------------------ | ------------------------------------------------------------------------- |
| **Ticker**                                 | Kode unik saham di bursa efek (contoh: BBCA = Bank Central Asia)          |
| **Emiten**                                 | Perusahaan yang sahamnya diperdagangkan di bursa                          |
| **Annual Report (Laporan Tahunan)**        | Laporan keuangan dan operasional perusahaan selama satu tahun             |
| **Financial Statement (Laporan Keuangan)** | Laporan yang menunjukkan posisi keuangan perusahaan                       |
| **P/E Ratio (Price-to-Earnings)**          | Rasio harga saham terhadap laba per saham                                 |
| **Revenue (Pendapatan)**                   | Total pendapatan perusahaan dari operasi bisnis                           |
| **Gross Margin**                           | Persentase pendapatan yang tersisa setelah dikurangi HPP                  |
| **Fundamental Analysis**                   | Analisis saham berdasarkan data keuangan dan operasional perusahaan       |
| **Value Investing**                        | Strategi investasi yang mencari saham undervalued berdasarkan fundamental |
| **Disclaimer**                             | Pernyataan bahwa informasi bukan saran investasi dan mengandung risiko    |

---

## 🔧 Python & Development

| Istilah                        | Penjelasan                                                                   |
| ------------------------------ | ---------------------------------------------------------------------------- |
| **Type Hint**                  | Anotasi tipe data pada parameter dan return value fungsi Python              |
| **Docstring**                  | String dokumentasi di awal fungsi/class yang menjelaskan kegunaannya         |
| **Virtual Environment (venv)** | Lingkungan Python terisolasi untuk mengelola dependency per proyek           |
| **uv**                         | Package manager Python modern yang menggantikan pip                          |
| **pip**                        | Package manager Python standar (di proyek ini diganti uv)                    |
| **Pydantic**                   | Library validasi data menggunakan type hints                                 |
| **SQLModel**                   | ORM yang menggabungkan SQLAlchemy dan Pydantic                               |
| **FastAPI**                    | Web framework Python modern untuk membangun API                              |
| **Loguru**                     | Library logging Python yang mudah digunakan                                  |
| **Async/Await**                | Sintaks Python untuk operasi asynchronous (non-blocking)                     |
| **Decorator**                  | Fungsi yang membungkus fungsi lain untuk menambah perilaku (@app.get, @task) |
| **pytest**                     | Framework testing Python                                                     |
| **Fixture**                    | Setup dan teardown otomatis untuk test (pytest fixture)                      |

---

## 📋 Workflow & Process

| Istilah                  | Penjelasan                                                                         |
| ------------------------ | ---------------------------------------------------------------------------------- |
| **Spec (Spesifikasi)**   | Dokumen yang menjelaskan apa yang akan dibangun sebelum coding                     |
| **Changelog**            | Catatan perubahan yang dibuat setelah menyelesaikan task                           |
| **Conventional Commits** | Format standar pesan commit (`type: description`)                                  |
| **Pull Request (PR)**    | Permintaan untuk merge branch ke branch utama                                      |
| **Code Review**          | Proses review kode oleh anggota tim lain                                           |
| **Sprint**               | Periode waktu tertentu (biasanya 1-2 minggu) untuk menyelesaikan serangkaian task  |
| **Epic**                 | Kumpulan besar task/fitur yang saling terkait                                      |
| **User Story**           | Deskripsi fitur dari sudut pandang pengguna                                        |
| **Acceptance Criteria**  | Kriteria yang harus dipenuhi agar task dianggap selesai                            |
| **Pipeline**             | Serangkaian proses yang dijalankan secara berurutan (CI/CD pipeline, RAG pipeline) |
| **Scope**                | Batas apa yang termasuk dan tidak termasuk dalam sebuah task                       |

---

## 🔑 Proyek Modcus - Spesifik

| Istilah                           | Penjelasan                                                            |
| --------------------------------- | --------------------------------------------------------------------- |
| **Modcus**                        | Nama proyek ini — platform AI financial Q&A                           |
| **Ingestion**                     | Proses menerima, memproses, dan menyimpan dokumen ke knowledge base   |
| **Query**                         | Proses menjawab pertanyaan pengguna berdasarkan knowledge base        |
| **modcus_common**                 | Shared library yang berisi kode bersama (models, settings, utils)     |
| **Mode 1 (Company Analysis)**     | Mode query untuk menganalisis perusahaan tertentu                     |
| **Mode 2 (Stock Recommendation)** | Mode query untuk rekomendasi saham                                    |
| **Mode 3 (Document Analysis)**    | Mode query untuk menganalisis dokumen yang diupload user              |
| **Level (newbie/novice/expert)**  | Level gaya presentasi jawaban (tidak mempengaruhi rigor analisis)     |
| **Ticker Inference**              | Proses otomatis mendeteksi kode saham dari teks pertanyaan            |
| **Stage**                         | Tahap dalam pipeline processing (parse, chunk, embed, index)          |
| **Job**                           | Unit pekerjaan background yang bisa dilacak statusnya                 |
| **LLM Rotator**                   | Mekanisme untuk berganti API key secara otomatis saat key habis kuota |
| **Per-Job Logging**               | Logging yang terisolasi per job untuk memudahkan debugging            |
| **X-API-Key**                     | Header HTTP yang digunakan untuk autentikasi API                      |

---

## 📚 Referensi Tambahan

Jika ada istilah yang belum ada di glosarium ini, coba cari di:

- **[PROJECT_CONSTITUTION.md](../../../PROJECT_CONSTITUTION.md)** — Standar dan aturan teknis proyek
- **[AGENTS.md](../../../AGENTS.md)** — Guidelines khusus untuk AI agents
- **[README.md](../../../README.md)** — Overview proyek

---

🎉 **Selamat!** Kamu sudah menyelesaikan seluruh panduan onboarding. Sekarang kamu siap untuk mulai berkontribusi di proyek Modcus!

Kembali ke [Index Onboarding](index.md) jika perlu mereview bagian tertentu.
