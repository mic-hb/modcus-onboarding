# 01 — Pengenalan Proyek Modcus

## Apa itu Modcus?

**Modcus** adalah platform AI untuk **tanya-jawab tentang keuangan (financial Q&A)** dan **rekomendasi saham**. Untuk sekarang, dibangun atas teknologi **Retrieval-Augmented Generation (RAG)** dan **Agentic AI**. Bayangkan sebuah chatbot yang sangat pintar tentang keuangan — bisa menjawab pertanyaan tentang laporan keuangan perusahaan, memberikan analisis saham, dan merekomendasikan saham berdasarkan data yang ada.

</br>

**Apa bedanya kita dengan pakai ChatGPT atau Gemini biasa?**

1. ChatGPT/Gemini tidak spesifik dilatih dengan data finansial (misal. laporan keuangan perusahaan di Indonesia). Chatbot kita spesifik menggunakan data-data finansial perusahaan di Indonesia.
2. ChatGPT/Gemini bersifat general-purpose. Chatbot kita dibuat khusus untuk expertise di bidang keuangan/finansial dan analisis saham.
3. ChatGPT/Gemini sekarang sudah ada fitur "search the web" untuk mencari sumber informasi terbaru. Tapi, fitur search ini sekali lagi bersifat sangat broad & general, tidak spesifik mencari databaase keuangan atau database perusahaan di Indonesia.

</br>

**2 (dua) aspek utama yang kita fokuskan/unggulkan adalah:**

1. **Memiliki, memproses dan menyajikan sumber data berjumlah besar yang kredibel, akurat, dan resmi.**
   Untuk sekarang, tim kita masih kecil dan budget kita terbatas. Kita mungkin belum bisa membuat model LLM sendiri, atau bahkan sekedar melakukan fine-tuning yang efektif. Salah satu cara untuk membuat model LLM "lebih pintar" (terutama untuk bidang tertentu), adalah dengan menyiapkan data berupa dokumen yang bisa kita jadikan referensi untuk menjawab sebuah pertanyaan. Kemudian, kita buat sistem RAG supaya LLM bisa mencari dokumen yang relevan untuk membantu menjawab pertanyaan tersebut.
2. **Mendesain alur kerja dan workflow yang terstruktur untuk proses analisis dengan mereplikasi cara berpikir seorang pakar/expert di bidang keuangan.**
   Contoh, untuk analisis saham, kita menggunakan pendekatan fundamental dengan konsep value investing dan growth investing. Ini adalah cara analisis yang umum dipakai untuk mengambil keputusan investasi saham. Kita bisa mengaplikasikan pendekatan ini dengan cara membuat sistem Agentic AI (misalnya).

</br>

**2 (dua) aspek tersebut direalisasikan menjadi 2 (dua) proses juga, yaitu:**

1. **Ingestion/Indexing** - yaitu memproses ribuan source document (laporan tahunan, laporan keuangan, dll.) dan membangun sebuah knowledge base & index yang dijadikan dasar referensi untuk menjawab pertanyaan user.
2. **Query** - yaitu proses menjawab pertanyaan user berdasarkan informasi dari knowledge base yang sudah dibangun.

</br>

**Sederhananya:** Modcus adalah AI yang membaca dan memahami ribuan dokumen keuangan (laporan tahunan, laporan keuangan, berita, dll.), lalu bisa menjawab pertanyaan user berdasarkan pemahaman dari dokumen-dokumen tersebut.

---

## Visi dan Misi

### Visi

> Menyediakan platform AI yang reliable dan scalable untuk analisis keuangan berbasis bukti (evidence-based), yang dapat digunakan oleh investor dari level pemula hingga ahli.

### Misi

1. **Ingest** — Menerima dan memproses dokumen (laporan tahunan, laporan keuangan, dll.) untuk membangun knowledge base.
2. **Query** — Menjawab pertanyaan pengguna menggunakan LLM (Large Language Model) yang terhubung ke knowledge base.
3. **Recommend** — Memberikan rekomendasi saham berdasarkan analisis keuangan yang terstruktur serta berbasis data.
4. **Monitor** — Memantau semua operasi dengan logging, metrik, dan audit yang jelas.

---

## Latar Belakang Project

### Sedikit Sejarah

Project ini awalnya menggunakan library RAG yang bernama **RAG-Anything** dan **LightRAG** (pada versi v0.1). Cara kerja librarynya adalah dengan membuat sebuah *knowledge graph* besar yang sangat detail dari semua dokumen yang diproses. Yang dilakukan adalah one-step ahead daripada sekadar RAG konvensional (membuat _vector index_). Kelebihannya, konteks yang dihasilkan sangat detail dan ada hubungan atau relasi antar topik, antar dokumen, bahkan antar entitas. Tapi ada masalah besar:

| Masalah                              | Dampak                                                                             |
| ------------------------------------ | ---------------------------------------------------------------------------------- |
| **Ingestion/indexing sangat lambat** | Proses 100-200 halaman PDF bisa memakan waktu 5-15 jam                             |
| **Biaya sangat mahal**               | Token LLM dan Embedding untuk ekstraksi entitas dan relasi bisa 10-50x lebih mahal |
| **Resource-intensive**               | Membutuhkan memori dan komputasi  tinggi                                           |
| **Tidak scalable**                   | Target 15.000 dokumen tidak realistis dengan pendekatan ini                        |

### Solusi: Rebuild ke v0.2

Karena masalah di atas, kita memutuskan untuk **membangun ulang (rebuild)** sistem dari nol menggunakan arsitektur baru:

- **LlamaIndex** sebagai RAG engine (menggantikan LightRAG)
- **LangGraph** sebagai query orchestrator (menggantikan pendekatan monolitik)

**Hasil yang diharapkan:**

| Metrik               | v0.1 (Lama)      | v0.2 (Baru)        | Improvement                                     |
| -------------------- | ---------------- | ------------------ | ----------------------------------------------- |
| Ingestion speed      | 5-15 jam/dokumen | < 60 detik/dokumen | 10x lebih cepat                                 |
| Biaya per dokumen    | > $1.00          | < $0.10            | 90% lebih murah                                 |
| Query accuracy       | 80-90%           | 80-90%             | Akurasi dan kualitas tetap sama atau lebih baik |
| Simple query latency | 5-10 detik       | < 2 detik          | 3-5x lebih cepat                                |

---

## Penjelasan Sistem

### Epic Brief & Plan

Untuk versi Modcus v0.2, kita punya sebuah [**Epic Brief**](../specs/2026-02-03_v0.2-project-rebuild/01-epic-brief.md), yaitu dokumentasi yang berisis rencana implementasi, latar belakang membuat v0.2, pembahasan detail, dan fitur apa saja yang akan dibuat (--> [di sini](../specs/2026-02-03_v0.2-project-rebuild/01-epic-brief.md))

Selain sebuah **Epic Brief**, kita juga punya dokumen terkait seperti:
- [**Core Flows**](../specs/2026-02-03_v0.2-project-rebuild/02-core-flows.md) - Gambaran tentang apa yang bisa dilakukan dengan Modcus v0.2
- [**Technical Plan**](../specs/2026-02-03_v0.2-project-rebuild/03-technical-plan.md) - Rancangan teknis secara detail
- [**Architecture Validation**](../specs/2026-02-03_v0.2-project-rebuild/04-architecture-validation.md) - Penjelasan tentang arsitektur sistem

---

### Komponen Utama Sistem

Sistem Modcus dibagi menjadi beberapa _service_ atau **komponen** yang memiliki fungsi masing-masing:

1. Ingestion Service API - sebuah API yang menyediakan endpoint untuk menjalankan proses ingestion
2. Celery Worker - untuk memproses background job (terhubung dengan Ingestion Service)
3. Query Service API - sebuah API yang menyediakan endpoint untuk meng-handle query dari user
4. Database (PostgreSQL) - database utama sistem ini untuk menyimpan metadata dokumen, daftar perusahaan, daftar dokumen, dll.
5. Vector Database - database untuk menyimpan vektor embedding

---

### Source Code

Secara source code atau kodingan, repo ini dibagi menjadi 4 bagian atau 4 modul.

```
Modcus Core v0.2
├── modcus_api_ingest (Ingestion module)  → Source code untuk Ingestion API dan Celery Worker
├── modcus_api_query (Query module)       → Source code untuk Query API
├── modcus_common (Common module)         → Code yang di-sharing (database model, settings, dll.)
└── modcus_migration                      → Code untuk mengelola skema dan migrasi database
```

---

### Mode Query

Fitur query (menjawab pertanyaan) dibagi menjadi 3 mode, yaitu:
1. Mode 1 - Company Analysis
2. Mode 2 - Stock Recommendation
3. Mode 3 - Document Analysis

---

#### Mode 1: Company Analysis (Analisis Perusahaan)

User bertanya tentang perusahaan tertentu, atau topik tertentu, seperti chatbot pada umumnya.

Mode ini dibagi lagi menjadi 3 level kompleksitas (_levels of complexity_):

| Level            | Deskripsi                                                                                         | Contoh                                                              |
| ---------------- | ------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| **1. Simple**    | Jawaban singkat dan langsung dengan mengambil informasi dari knowledge base                       | "Kapan PT ABC didirikan?"                                           |
| **2. Reasoning** | Analisis lebih mendalam, dengan penalaran, mengambil implikasi, memperhitungkan kemungkinan, dsb. | "Apa dampak penurunan gross margin PT XYZ terhadap profitabilitas?" |
| **3. Deep-dive** | Analisis terstruktur untuk valuasi saham dengan pendekatan value investing & growth investing     | "Berikan analisis fundamental lengkap BBCA"                         |

**Catatan:**
- Reasoning di sini BEDA dengan fitur "reasoning" atau "thinking" di LLM. Reasoning di sini = ada step tambahan untuk analisis.
- Level complexity dideteksi secara otomatis oleh sistem, bukan dipilih oleh user.

---

#### Mode 2: Stock Recommendations (Rekomendasi Saham)

Pengguna meminta rekomendasi saham. Sistem akan:

1. Memilih kandidat saham dari knowledge base
2. Menganalisis setiap saham secara paralel
3. Membandingkan dan me-ranking hasilnya

**Contoh:** "Rekomendasikan 5 saham perbankan terbaik"

---

#### Mode 3: Document Analysis (Analisis Dokumen)

Pengguna mengunggah dokumen sendiri dan bertanya berdasarkan dokumen tersebut. Jawabannya **hanya** berdasarkan dokumen yang diunggah, bukan dari knowledge base utama.

**Contoh:** Upload laporan Q1 2026 PT ABC, lalu tanya "Apa poin-poin penting dari laporan ini?"

---

### 3 Level Bahasa

Selain 3 **mode query** dan 3 _levels of complexity_ (khusus Mode 1), fitur query juga bisa disesuaikan menjadi 3 **level bahasa/diksi:**

1. Newbie - AI akan menjawab dengan bahasa sederhana, tidak banyak istilah rumit, agar mudah dimengerti orang awam.
2. Novice - AI akan mulai menggunakan istilah-istilah atau konsep-konsep dalam dunia Finance.
3. Expert - AI akan seperti berbicara dengan seorang expert di bidang Finance, misalnya dengan istilah-istilah yang sangat teknis.

---

### Safe-guards

Sistem ini punya 3 lapis safety measures agar tidak menjawab pertanyaan yang tidak sesuai dengan ketentuan atau di luar dugaan:

1. **Layer 1 (Validation Agent)** — AI agent yang mengklasifikasi pertanyaan user untuk menentukan maksud atau intensi, mem-filter kata-kata yang dilarang, menolak pertanyaan off-topic (di luar domain finance), dsb.
2. **Layer 2 (LLM Guardrails)** — Instruksi di system prompt untuk menolak pertanyaan off-topic (jika masih belum ter-detect Layer 1)
3. **Layer 3 (Post-Retrieval)** — If-condition jika tidak ada konteks relevan dari knowledge base, maka tolak pertanyaan user dengan bahasa yang halus dan ramah

**Contoh:**

- ✅ "Apa itu P/E ratio?" → Dijawab (masih sesuai bidang keuangan)
- ✅ "Analisis profitabilitas BBCA" → Dijawab (sesuai)
- ❌ "Bagaimana cuaca hari ini?" → Ditolak (off-topic)
- ❌ "Buatkan puisi" → Ditolak (off-topic)

---

## Requirements

### Functional Requirements (FR)

Berikut beberapa requirements utama yang harus dipenuhi:

| ID     | Requirement                                                                                             |
| ------ | ------------------------------------------------------------------------------------------------------- |
| FR-001 | Menyediakan 3 mode query: Company Analysis, Stock Recommendations, Document Analysis                    |
| FR-002 | Setiap query API request harus menyertakan `mode` dan `level` (bahasa)                                  |
| FR-003 | Sistem harus bisa infer/deteksi ticker perusahaan dari pertanyaan                                       |
| FR-006 | Semua jawaban harus menyertakan ticker perusahaan, reasoning (proses berpikir), dan sitasi/sumber       |
| FR-008 | Jika evidence/data tidak cukup atau tidak ada, sistem harus transparan dan menjawab dengan terus terang |
| FR-013 | Semua query dan LLM call harus dimonitor dan di-log untuk audit                                         |
| FR-017 | Rekomendasi saham harus menyertakan disclaimer                                                          |

### Non-Functional Requirements (NFR)

| ID      | Requirement                                                                                                      |
| ------- | ---------------------------------------------------------------------------------------------------------------- |
| NFR-001 | API dilindungi dengan `X-API-Key` (karena API hanya berkomunikasi dengan sesama API, bukan langsung dengan user) |
| NFR-004 | Logging terstruktur (mode, level, tickers, timing)                                                               |
| NFR-005 | Metrik Prometheus dengan label `mode` dan `level` (kompleksitas dan bahasa)                                      |
| NFR-012 | Error responses terstandarisasi (misal `400`/`413`/`415`/`500`)                                                  |

### Success Criteria

| ID     | Kriteria                                                 |
| ------ | -------------------------------------------------------- |
| SC-001 | 90% jawaban sederhana < 2 detik                          |
| SC-003 | 95% jawaban Mode 3 hanya cite dari dokumen yang diupload |
| SC-004 | 95% akurasi deteksi ticker                               |
| SC-005 | 100% jawaban menyertakan minimal 2 citation              |
| SC-009 | 100% rekomendasi menyertakan disclaimer                  |

---

## Urutan Prioritas

Saat mengembangkan Modcus, urutan prioritas yang harus diikuti:

1. **Quality** — Kualitas adalah yang utama
2. **Efficiency** — Efisiensi dalam desain dan implementasi
3. **Speed** — Kecepatan eksekusi dan performa
4. **Cost** — Efisiensi biaya (terutama LLM token)

---

## Prinsip Development

Beberapa prinsip utama dalam segi teknis (berbicara tentang kodingan):

1. **Code Quality** — Usahakan code rapi, terstruktur, penamaan variabel jelas & tidak random, indentasi jelas (4 spaces).
2. **Modularity** — Usahakan code juga modular, tidak repetitif, dan tidak redundan. Bisa menerapkan design pattern dalam software engineering, misalnya Object-oriented (membuat Class) atau Factory.
3. **Reliability** — Pastikan sistem reliabel, perhatikan error handling, fail gracefully (misal. 1x error untuk 1 user saat melakukan 1 proses TIDAK boleh sampai mempengaruhi user atau proses lain).
4. **Observability** — Biasakan menyertakan debug logs agar bisa memonitor alur program saat mode development.
5. **Separation of Concerns** — 1 komponen hanya punya 1 tujuan dan 1 kewajiban, usahakan 1 komponen tidak menjadi Superman (bisa melakukan semua hal).

---

## Apa Selanjutnya?

Setelah memahami apa itu Modcus dan apa yang sedang dibangun, lanjut ke bagian berikutnya untuk mempelajari **konsep-konsep dasar** yang perlu dikuasai:

➡️ [02 — Konsep Dasar](02-konsep-dasar.md)
