# 08 — Dokumentasi & Changelog

Di project ini, **dokumentasi adalah bagian wajib dari setiap task**. Setiap perubahan harus didokumentasikan dengan baik agar kita bisa melacak apa yang sudah dikerjakan, mengapa, dan bagaimana.

---

## Prinsip Dokumentasi

1. **Tulis untuk orang lain (dan diri sendiri)** — kalau dalam 6 bulan ke depan kita baca lagi dokumentasinya, pastikan kita masih bisa paham apa yang sudah dikerjakan
2. **Jelaskan "mengapa", bukan hanya "apa"** — code sudah menjelaskan "apa" yang dibuat, dokumentasi harus menjelaskan alasan di balik keputusan yang diambil
3. **Tulis segera setelah selesai** — jangan menunda, karena konteks akan hilang
4. **Gunakan contoh konkret** — contoh code lebih mudah dipahami daripada penjelasan abstrak

---

## Struktur Dokumentasi Proyek

```
docs/
├── docs/
│   ├── index.md                    # Landing page dokumentasi
│   ├── onboarding/                 # ← Kamu di sini!
│   ├── architecture/               # Arsitektur sistem
│   ├── api/                        # API reference
│   ├── deployment/                 # Panduan deployment
│   ├── development/                # Panduan development
│   ├── guides/                     # Tutorial dan how-to
│   ├── specs/                      # Spesifikasi dan changelog
│   │   ├── 2026-02-03_v0.2.../     # Per-task spec folders
│   │   └── 2026-02-08.../
│   └── reference/                  # Referensi teknis
└── mkdocs.yml                      # Konfigurasi MkDocs
```

---

## Format Dokumentasi: Changelog

### Apa itu Changelog?

**Changelog** adalah catatan perubahan yang dibuat **setiap kali kita menyelesaikan sebuah task/ticket**. Untuk sekarang, format dokumentasi yang digunakan adalah seperti membuat changelog.

### Format Changelog

Setiap changelog disimpan di folder yang sama dengan spec/plan yang dibuat:

```
docs/docs/specs/[YYYY-MM-DD]_[hh:mm]_[deskripsi-singkat]/changelog.md

# atau

agent-os/specs/[YYYY-MM-DD]-[hh:mm]-[deskripsi-singkat]/changelog.md
```

**Contoh:**

```
docs/docs/specs/2026-02-08_0228_DoclingParser-format-support/changelog.md
```

### Contoh Template Changelog

```markdown
# Changelog: [Judul Task]

**Tanggal:** YYYY-MM-DD
**Author:** [Nama]
**Status:** Completed

## Ringkasan

Penjelasan singkat tentang apa yang dikerjakan dan mengapa.

## Perubahan

### File yang Diubah

| File                  | Tipe     | Deskripsi           |
| --------------------- | -------- | ------------------- |
| `path/to/file.py`     | Modified | Deskripsi perubahan |
| `path/to/new_file.py` | New      | Deskripsi file baru |
| `path/to/deleted.py`  | Deleted  | Alasan penghapusan  |

### Detail Implementasi

Jelaskan detail teknis yang penting:
- Pendekatan yang dipilih dan alasannya
- Trade-off yang diambil
- Edge cases yang ditangani

### Contoh Penggunaan

```python
# Contoh kode jika relevan
result = new_function(param1, param2)
```

## Backward Compatibility

Apakah ada breaking changes? Jika ya, apa mitigasinya?

## Testing

- [ ] Unit tests
- [ ] Integration tests
- [ ] Manual testing

## Notes

Catatan tambahan, hal-hal yang perlu di-follow-up, dll.
```
```

### Contoh

```markdown
# Changelog: DoclingParser Format Support

**Tanggal:** 2026-02-08
**Author:** Developer
**Status:** Completed

## Ringkasan

Memperbarui DoclingParser untuk mendukung dua format JSON:
1. **Modern format** — DoclingDocument versi baru dengan nested structure
2. **Legacy format** — Format lama yang flat

## Perubahan

### File yang Diubah

| File                                                   | Tipe     | Deskripsi                             |
| ------------------------------------------------------ | -------- | ------------------------------------- |
| `modcus_api_ingest/services/parsing/docling_parser.py` | Modified | Tambah deteksi format dan dual parser |
| `tests/test_docling_parser.py`                         | Modified | Tambah test untuk kedua format        |

### Detail Implementasi

#### Deteksi Format Otomatis

```python
def _detect_format(self, data: dict) -> str:
    """Deteksi apakah JSON menggunakan format modern atau legacy."""
    if "texts" in data and isinstance(data["texts"], list):
        return "modern"
    return "legacy"
```

Pendekatan:
- Cek keberadaan key `texts` sebagai list → modern format
- Fallback ke legacy jika tidak ditemukan

## Backward Compatibility

✅ Fully backward compatible. Format lama tetap didukung.
```
```

---

## Menulis Docstrings

### Format Wajib

Setiap function dan class **wajib** memiliki docstring dengan format Google-style:

```python
def parse_document(
    file_path: str,
    format_type: str = "auto",
    max_pages: Optional[int] = None,
) -> ParseResult:
    """Parse dokumen dan ekstrak teks beserta metadata.

    Mendukung format PDF, Docling JSON, dan LlamaParse JSON.
    Jika format_type="auto", format akan dideteksi dari ekstensi file.

    Args:
        file_path: Path absolut ke file yang akan di-parse
        format_type: Tipe format ("auto", "pdf", "docling", "llamaparse")
        max_pages: Jumlah halaman maksimal (None = semua halaman)

    Returns:
        ParseResult berisi teks, metadata, dan list gambar yang diekstrak

    Raises:
        FileNotFoundError: Jika file_path tidak ditemukan
        ParsingError: Jika format tidak didukung atau parsing gagal
        ValueError: Jika max_pages bernilai negatif

    Example:
        >>> result = parse_document("/data/annual_report.pdf")
        >>> print(result.text[:100])
        'PT Bank Central Asia Tbk...'
    """
    ...
```

### Docstring untuk Class

```python
class DoclingParser:
    """Parser untuk dokumen format Docling JSON.

    Mendukung dua sub-format:
    - Modern: DoclingDocument v2 dengan nested structure
    - Legacy: Format flat (v1)

    Format dideteksi secara otomatis berdasarkan struktur JSON.

    Attributes:
        supported_formats: List format yang didukung
        max_chunk_size: Ukuran maksimal chunk dalam karakter

    Example:
        >>> parser = DoclingParser()
        >>> result = parser.parse("document.json")
    """
    ...
```

---

## Checklist Sebelum Submit PR

Sebelum membuat Pull Request, pastikan:

- [ ] Semua function baru punya **docstring** dengan Args/Returns/Raises
- [ ] **Changelog** sudah ditulis di folder spec yang sesuai
- [ ] Commit messages mengikuti **Conventional Commits** format
- [ ] Kode sudah di-**test** (minimal manual test)
- [ ] Tidak ada **hardcoded values** atau secrets
- [ ] **Logging** sudah ditambahkan untuk operasi penting
- [ ] **Type hints** ada di semua function dan parameter

---

## Apa Selanjutnya?

Bagian terakhir — **glosarium** berisi daftar istilah yang sering digunakan di proyek ini:

➡️ [09 — Glosarium](09-glosarium.md)
