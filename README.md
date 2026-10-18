# Assignment Semantic Search & LLM Application

Prototype aplikasi **semantic search** yang terintegrasi dengan **LLM** untuk menjawab pertanyaan pelanggan e-commerce secara otomatis, berdasarkan knowledge base FAQ.

Alur kerjanya: setiap entri FAQ diubah menjadi embedding, disimpan di vector database (FAISS), lalu ketika ada pertanyaan masuk, sistem mencari FAQ paling relevan secara semantik dan meminta LLM menyusun jawaban dari FAQ tersebut.

Ini adalah tugas individu sesi 13 **Advanced Prompt Engineering & Context Management** di ReworkAcademy.

---

## Problem Statement

### Objective

Tim customer service sebuah e-commerce menerima ratusan pertanyaan repetitif setiap hari: soal pemesanan, pengiriman, pembayaran, hingga pengembalian barang. Sebagian besar pertanyaan itu sebenarnya sudah terjawab di knowledge base FAQ, tapi pelanggan sering bertanya dengan kata-kata yang tidak persis sama dengan isi FAQ.

Tujuan tugas ini adalah membangun prototype asisten AI yang:

1. Mencari entri FAQ paling relevan terhadap pertanyaan pelanggan, **meskipun kata-katanya berbeda** (semantic search, bukan sekadar pencocokan kata kunci).
2. Menyusun jawaban yang natural menggunakan LLM, **berdasarkan FAQ yang ditemukan** (bukan dari pengetahuan umum LLM).
3. Menyadari keterbatasannya: jika pertanyaan tidak ada jawabannya di FAQ, asisten mengatakannya dan menyarankan menghubungi customer service.

### Knowledge Base

- **Sumber:** `ecommerce_faq_knowledge_base.csv` (diberikan oleh mentor, tidak diubah).
- **Struktur:** 59 entri FAQ dengan 4 kolom:
  - `faq_id`: ID unik entri
  - `category`: kategori topik (8 kategori: Pemesanan, Pengiriman, Pembayaran, Pengembalian, Akun, Produk, Promo, Customer Service)
  - `question`: pertanyaan yang sering diajukan
  - `answer`: jawaban resmi
- **Unit retrievable:** setiap entri digabung menjadi satu teks `Q: <question> A: <answer>` (pertanyaan dan jawaban di-retrieve bersama supaya konteksnya utuh).

### Batasan dan Asumsi

- Sistem **hanya menjawab berdasarkan FAQ yang tersedia** di knowledge base. Informasi di luar FAQ dianggap tidak tersedia.
- Knowledge base tidak diubah dan tidak diganti dengan sumber lain.
- Embedding dan LLM memakai model Gemini (`gemini-embedding-001` dan `gemini-3.6-flash`).
- Teknik chunking dokumen panjang tidak diterapkan (belum menjadi learning objective sesi ini; tiap entri FAQ sudah pendek).
- Satu jalur API dipakai konsisten dari awal sampai akhir, sehingga vektor embedding tidak tercampur antar model.

---

## Alur Sistem

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌───────────┐    ┌─────────┐    ┌─────────┐
│ Ingest  │ →  │ Embed   │ →  │ Store   │ →  │ Retrieve  │ →  │ Prompt  │ →  │ Generate│
│ (CSV)   │    │ (API)   │    │ (FAISS) │    │ (top-k)   │    │ (RAG)   │    │ (LLM)   │
└─────────┘    └─────────┘    └─────────┘    └───────────┘    └─────────┘    └─────────┘
```

1. **Ingest**: membaca `ecommerce_faq_knowledge_base.csv` dengan Pandas; menggabungkan `question` + `answer` menjadi satu unit teks per entri.
2. **Embed**: mengubah setiap unit menjadi vektor embedding 768 dimensi (`gemini-embedding-001`) via Gemini Embeddings API.
3. **Store**: menyimpan seluruh vektor ke FAISS index (`IndexFlatIP`); vektor dinormalisasi L2 sehingga inner product = cosine similarity.
4. **Retrieve**: query pengguna di-embed dengan cara yang sama, lalu FAISS mencari top-K vektor terdekat (= entri FAQ paling relevan beserta skor kemiripan).
5. **Prompt**: hasil retrieval disusun menjadi prompt terstruktur (Persona, Task, Context, Format) yang mengarahkan LLM menjawab hanya dari context.
6. **Generate**: LLM (`gemini-3.6-flash`) menghasilkan jawaban akhir; jika context tidak relevan, LLM diminta mengakui ketidaktahuannya.

### Dukungan Dual-Provider

Notebook otomatis memilih jalur API:

| Jalur | Embedding | LLM | Kapan dipakai |
|---|---|---|---|
| A. Gemini API langsung | `gemini-embedding-001` (768d, dengan `task_type`) | `gemini-3.6-flash` | `GEMINI_API_KEY` tersedia dan region didukung (mis. laptop) |
| B. OpenRouter (fallback) | `google/gemini-embedding-001` (3072d) | `google/gemini-3.6-flash` | Gemini langsung gagal (mis. region server diblokir) |

Satu jalur dipakai konsisten untuk seluruh proses (embed dokumen, embed query, generate jawaban) supaya ruang vektor seragam. Deteksi jalur dilakukan otomatis di awal notebook.

---

## Hasil Pengujian

Sistem diuji dengan **6 query**: 2 kata persis, 2 parafrase, 2 di luar knowledge base.

| # | Jenis | Query | Top-1 similarity | Retrieval | Jawaban LLM |
|---|---|---|---|---|---|
| 1 | Kata persis | Bagaimana cara melakukan pemesanan di platform ini? | 0.858 | ✅ Akurat | Sesuai FAQ |
| 2 | Kata persis | Berapa lama estimasi waktu pengiriman barang? | 0.903 | ✅ Akurat | Sesuai FAQ |
| 3 | Parafrase | Gimana caranya saya order barang di toko kalian? | 0.839 | ✅ Akurat | Sesuai FAQ |
| 4 | Parafrase | Paketku kok belum datang ya, kapan nyampe? | 0.810 | ✅ Akurat | Sesuai FAQ |
| 5 | Di luar KB | Bagaimana cara klaim asuransi kesehatan? | 0.647 | ⚠️ Rendah | Mengakui tidak tahu |
| 6 | Di luar KB | Apakah kalian menjual tiket pesawat? | 0.650 | ⚠️ Rendah | Mengakui tidak tahu |

**Temuan utama:**

- **Semantic search menangkap makna, bukan kata kunci.** Query *"Gimana caranya saya order barang di toko kalian?"* (tanpa satu pun kata yang sama dengan FAQ) berhasil menemukan *"Bagaimana cara melakukan pemesanan di platform ini?"* dengan similarity 0.839.
- **Skor similarity berguna sebagai sinyal kepercayaan.** Query relevan mendapat skor ≥ 0.81; query di luar KB jatuh di bawah 0.65. Ambang ~0.70 layak dipertimbangkan untuk memutuskan "informasi tersedia atau tidak".
- **Prompt "akui ketidaktahuan" bekerja.** Untuk topik di luar KB, LLM menjawab jujur: belum ada informasinya, sarankan hubungi customer service. Tidak ada jawaban mengarang.

Detail lengkap tiap query (termasuk top-3 retrieval dan jawaban LLM) ada di [`documentation/retrieval_test_results.md`](documentation/retrieval_test_results.md).

---

## Keterbatasan

1. **Pencarian FAISS exact (`IndexFlatIP`)**: cepat untuk 59 entri, tapi tidak skalabel untuk jutaan dokumen. Untuk skala besar perlu `IndexIVFFlat` atau `HNSW`.
2. **Kualitas retrieval bergantung pada model embedding**: vektor dari model berbeda tidak kompatibel dan tidak boleh dicampur dalam satu index.
3. **LLM tidak 100% deterministik**: jawaban bisa sedikit bervariasi antar pemanggilan (suhu 0.3).
4. **`max_tokens` perlu cukup besar untuk Gemini 3.x**: model ini memakai sebagian jatah token untuk reasoning internal sebelum menjawab; `max_tokens` terlalu kecil membuat jawaban terpotong.
5. **Region/lokasi API**: Gemini API tidak tersedia dari semua region (mis. IP server cloud diblokir). Notebook memakai fallback OpenRouter untuk kasus ini.
6. **Tidak ada mekanisme umpan balik**: sistem belum mencatat apakah jawaban yang diberikan benar-benar membantu pelanggan.

---

## Cara Menjalankan

### 1. Setup environment

```bash
python -m venv .venv
source .venv/bin/activate      # Linux/Mac
# .venv\Scripts\activate       # Windows

pip install -r requirements.txt
```

### 2. Set API key

Buat file `.env` di root repo:

```bash
# .env: jangan di-commit (sudah di-exclude .gitignore)
GEMINI_API_KEY=your_gemini_api_key
OPENROUTER_API_KEY=your_openrouter_api_key   # opsional, untuk fallback
```

Cukup `GEMINI_API_KEY` saja jika dijalankan dari laptop/region yang mendukung Gemini. Tambahkan `OPENROUTER_API_KEY` jika ingin fallback otomatis.

### 3. Jalankan notebook

```bash
jupyter notebook notebook/starter_notebook_semantic_search.ipynb
```

Jalankan semua cell dari atas ke bawah (Kernel → Restart & Run All). Notebook akan:
1. Load knowledge base & menyiapkan unit retrievable
2. Membuat embedding (dual-provider otomatis)
3. Menyimpan ke FAISS index
4. Menjalankan semantic search + LLM untuk 6 query uji
5. Menampilkan analisis relevansi

> Catatan: notebook memanggil API embedding dan LLM (butuh koneksi internet dan API key aktif). Biaya untuk 59 dokumen + 6 query sangat kecil (free tier biasanya cukup).

---

## Struktur Project

```
semantic-search-assignment/
├── data/
│   └── ecommerce_faq_knowledge_base.csv   # knowledge base asli (tidak diubah)
├── notebook/
│   └── starter_notebook_semantic_search.ipynb  # notebook utama
├── documentation/
│   └── retrieval_test_results.md          # hasil uji retrieval & analisis
├── README.md
└── requirements.txt
```