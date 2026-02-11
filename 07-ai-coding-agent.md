# 07 — AI Coding Agent

Di project ini, kita merekomendasikan menggunakan **AI coding agents** untuk mempercepat pekerjaan. Bagian ini menjelaskan tools yang tersedia dan cara menggunakannya secara efektif.

---

## Apa itu AI Coding Agent?

AI coding agent adalah tools yang memanfaatkan LLM untuk membantu menulis kodingan.

Kegunaan:

- **Menulis code** sesuai instruksi
- **Menjelaskan code** yang ada
- **Debugging** dan tracing bug
- **Refactoring** code
- **Menulis tests** dan dokumentasi
- **Menjawab pertanyaan** tentang codebase

> **Penting:** AI agent bersifat **membantu**, bukan pengganti. Kita tetap harus memahami kode yang ditulis dan me-review hasilnya.

---

## Tools yang Bisa Digunakan

### 1. Cursor

**Cursor** adalah IDE (berbasis VS Code) dengan fitur AI yang terintegrasi langsung di editor.

### 2. OpenCode

**OpenCode** adalah AI coding dalam bentuk program terminal (CLI).

### 3. Claude Code (Anthropic)

**Claude Code** adalah AI coding agent dari Anthropic (Claude)

---

## Best Practices Saat Pakai AI Agent

### ✅ Yang Harus Dilakukan

#### 1. Berikan Konteks yang Jelas

```
❌ "Fix the bug"
✅ "Fix the foreign key violation error in llm_logger.py. The error happens
   because LLMCallRecord is created before LLMCallFile, but LLMCallFile
   has a foreign key to LLMCallRecord.id"
```

#### 2. Minta Plan Sebelum Implementasi

```
✅ "Before making any changes, create a plan for implementing ticker
   inference. Show me the plan first and wait for my approval."
```

#### 3. Review Setiap Perubahan

Jangan langsung accept semua perubahan dari AI. Review dan pastikan:
- Code masih mengikuti konvensi proyek
- Tidak ada code yang dihapus secara tidak sengaja
- Logic-nya benar dan make sense
- Error handling dan logging ada

#### 4. Gunakan Iterasi Kecil

```
✅ "Let's implement this step by step:
   Step 1: Create the data model
   Step 2: Add the service layer
   Step 3: Create the API route
   Step 4: Write tests"
```

#### 5. Sebutkan File dan Konteks yang Relevan

```
✅ "Look at how parsing is implemented in @modcus_api_ingest/services/parsing/
   docling_parser.py and follow the same pattern for the new parser"
```

### ❌ Yang Harus Dihindari

#### 1. Jangan Blindly Accept

```
❌ AI menghasilkan 500 baris code → langsung accept semua
✅ AI menghasilkan 500 baris code → review per section, minta revisi jika perlu
```

#### 2. Jangan Biarkan AI Menghapus Code Tanpa Alasan

```
❌ AI menghapus comments, logger.info(), atau code yang dianggap tidak perlu → accept
✅ "Do not remove any comments or logging statements unless I specifically ask"
```

#### 3. Jangan Terlalu Mengandalkan pada AI untuk Arsitektur

AI coding agent bagus untuk implementasi task spesifik, tapi untuk keputusan arsitektural (misalnya, pattern apa yang dipakai, bagaimana service berkomunikasi) biasakan diskusi dengan tim.

#### 4. Jangan Lupa Commit Secara Teratur

Saat AI membuat banyak perubahan, commit secara teratur agar kamu bisa rollback jika ada yang salah.

---

## Aturan Khusus AI Agent di Proyek Ini

Proyek Modcus punya file [`AGENTS.md`](../../../AGENTS.md) yang berisi aturan khusus untuk AI agent. Biasanya, kita bisa langsung mention file tersebut ke AI dan AI akan mengikuti aturan tersebut.

Ringkasannya:

### Aturan Umum

1. **Tanya jika ragu** — lebih baik bertanya daripada membuat asumsi
2. **Default ke implementasi sederhana** — hindari over-engineering
3. **Jangan hapus code** yang berfungsi kecuali diminta
4. **Jangan hapus comments** — comments ada untuk alasan tertentu
5. **Jangan hapus logging** — jika tidak diminta
6. **Ikuti konvensi** yang sudah ada di codebase

### Konvensi Kode

| Aturan         | Detail                                                       |
| -------------- | ------------------------------------------------------------ |
| **Type hints** | Wajib di semua fungsi (parameter dan return type)            |
| **Docstrings** | Wajib dengan format Args/Returns/Raises                      |
| **Logging**    | Gunakan `import logger from loguru`, jangan `import logging` |
| **Exceptions** | Custom exceptions untuk domain-specific errors               |
| **Config**     | Gunakan Pydantic settings, jangan di-hardcode                |

### Anti-patterns yang WAJIB Dihindari

```python
# ❌ Jangan import antar service
from modcus_api_ingest.services import something

# ❌ Jangan hardcode configuration
DATABASE_URL = "postgresql://user:pass@localhost/db"

# ❌ Jangan gunakan logging standard
import logging
logger = logging.getLogger(__name__)
```

---

## Contoh Workflow dengan AI Agent

Berikut contoh workflow saat menggunakan AI agent:

### Skenario: Menambah Parser Baru

```
Developer:  "I need to add a new parser for XLSX files in the ingestion
            service. Look at the existing DoclingParser in
            modcus_api_ingest/services/parsing/docling_parser.py as
            reference for the pattern.

            Before coding, create a plan covering:
            1. Class structure
            2. How it integrates with the pipeline
            3. What tests are needed

            Wait for my approval before implementing."

AI Agent:   [Membuat plan...]

Developer:  [Review plan, berikan feedback...]
            "Looks good, but also handle the case where the XLSX has
            merged cells. Go ahead and implement."

AI Agent:   [Implementasi sesuai plan...]

Developer:  [Review kode, commit]
            git add .
            git commit -m "feat(ingest): add XLSX parser with merged cell support"
```

---

## Apa Selanjutnya?

Lanjut ke bagian berikutnya untuk memahami **konvensi dokumentasi dan changelog**:

➡️ [08 — Dokumentasi & Changelog](08-dokumentasi-changelog.md)
