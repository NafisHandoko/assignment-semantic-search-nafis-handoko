# Hasil Uji Retrieval & Response AI

Dokumen ini mencatat hasil pengujian sistem semantic search + LLM pada knowledge base FAQ e-commerce (59 entri, 8 kategori).

**Metodologi:** setiap query diuji dengan alur `query → embedding → semantic search (FAISS, top-3) → prompt ber-context → LLM`. Similarity score adalah cosine similarity (vektor ternormalisasi L2, `IndexFlatIP`), dengan rentang 0–1 (makin dekat 1 makin relevan).

**Jalur yang dipakai pada pengujian ini:** fallback OpenRouter (`google/gemini-embedding-001`, 3072 dimensi, dan `google/gemini-3.6-flash`) karena region server tidak mendukung akses langsung ke Gemini API.

---

## Ringkasan Hasil

| # | Kategori Uji | Query | Top-1 (similarity) | Retrieval Akurat? | Jawaban LLM |
|---|---|---|---|---|---|
| 1 | Langsung | Bagaimana cara melakukan pemesanan di platform ini? | Pemesanan (0.858) | ✅ Ya | Menjelaskan langkah pemesanan |
| 2 | Langsung | Berapa lama estimasi waktu pengiriman barang? | Pengiriman (0.903) | ✅ Ya | Menyebut estimasi 2-5 hari kerja |
| 3 | Parafrase | Gimana caranya saya order barang di toko kalian? | Pemesanan (0.839) | ✅ Ya | Menjelaskan langkah pemesanan |
| 4 | Parafrase | Paketku kok belum datang ya, kapan nyampe? | Pengiriman (0.810) | ✅ Ya | Menyebut estimasi & saran lacak |
| 5 | Tidak ada di KB | Bagaimana cara klaim asuransi kesehatan? | Pengiriman (0.647) | ⚠️ Skor rendah | Mengakui tidak tahu, sarankan CS |
| 6 | Tidak ada di KB | Apakah kalian menjual tiket pesawat? | Pembayaran (0.650) | ⚠️ Skor rendah | Mengakui tidak tahu, sarankan CS |

---

## Detail Per Query

### Query 1 — Langsung (kata persis)
**Query:** "Bagaimana cara melakukan pemesanan di platform ini?"

| Rank | Kategori | Similarity | FAQ |
|---|---|---|---|
| 1 | Pemesanan | 0.858 | Bagaimana cara melakukan pemesanan di platform ini? |
| 2 | Pemesanan | 0.748 | Apakah saya bisa memesan tanpa membuat akun? |
| 3 | Pemesanan | 0.715 | Bagaimana cara mengetahui status pesanan saya saat ini? |

**Jawaban LLM:** menjelaskan langkah-langkah pemesanan (pilih produk, tambah ke keranjang, checkout) sesuai FAQ.

**Analisis:** retrieval tepat — query identik dengan FAQ, similarity top-1 tinggi (0.858). LLM menjawab akurat berdasarkan context.

---

### Query 2 — Langsung (kata persis)
**Query:** "Berapa lama estimasi waktu pengiriman barang?"

| Rank | Kategori | Similarity | FAQ |
|---|---|---|---|
| 1 | Pengiriman | 0.903 | Berapa lama estimasi waktu pengiriman barang? |
| 2 | Pengiriman | 0.783 | Bagaimana jika paket saya belum sampai melebihi estimasi? |
| 3 | Pemesanan | 0.756 | Berapa lama waktu yang dibutuhkan sebelum pesanan diproses penjual? |

**Jawaban LLM:** "Estimasi pengiriman barang umumnya membutuhkan waktu **2-5 hari kerja** untuk area..." (sesuai FAQ).

**Analisis:** retrieval sangat akurat (similarity tertinggi di seluruh pengujian: 0.903). Jawaban LLM konsisten dengan isi FAQ.

---

### Query 3 — Parafrase (kata berbeda)
**Query:** "Gimana caranya saya order barang di toko kalian?"

| Rank | Kategori | Similarity | FAQ |
|---|---|---|---|
| 1 | Pemesanan | 0.839 | Bagaimana cara melakukan pemesanan di platform ini? |
| 2 | Produk | 0.713 | Bagaimana cara memberikan ulasan untuk produk yang sudah dibeli? |
| 3 | Pemesanan | 0.712 | Apakah saya bisa memesan tanpa membuat akun? |

**Jawaban LLM:** menjelaskan langkah pemesanan ("Halo! Untuk melakukan pemesanan di platform kami...").

**Analisis:** ini bukti utama bahwa search menangkap **makna, bukan kata kunci**. Query memakai kata "order", "toko", "kalian" yang tidak muncul di FAQ, tapi embedding berhasil memetakannya ke "melakukan pemesanan di platform ini" (similarity 0.839). Top-1 tepat.

---

### Query 4 — Parafrase (bahasa kasual)
**Query:** "Paketku kok belum datang ya, kapan nyampe?"

| Rank | Kategori | Similarity | FAQ |
|---|---|---|---|
| 1 | Pengiriman | 0.810 | Bagaimana jika paket saya belum sampai melebihi estimasi? |
| 2 | Pengiriman | 0.792 | Berapa lama estimasi waktu pengiriman barang? |
| 3 | Pengiriman | 0.768 | Bagaimana cara melacak paket yang sedang dikirim? |

**Jawaban LLM:** "Halo Kak! Untuk estimasi waktu pengiriman barang, umumnya membutuhkan waktu..." + saran lacak.

**Analisis:** query sangat kasual ("kok", "nyampe"), tapi ketiga hasil retrieval semuanya kategori Pengiriman yang relevan. Semantic embedding memahami maksud "paket belum datang" ≈ "paket belum sampai". Top-1 tepat.

---

### Query 5 — Tidak ada di KB (uji ketidaktahuan)
**Query:** "Bagaimana cara klaim asuransi kesehatan?"

| Rank | Kategori | Similarity | FAQ |
|---|---|---|---|
| 1 | Pengiriman | 0.647 | Apakah ongkos kirim sudah termasuk asuransi pengiriman? |
| 2 | Pengiriman | 0.635 | Apa yang harus dilakukan jika paket diterima dalam kondisi rusak? |
| 3 | Pemesanan | 0.631 | Bagaimana cara melakukan pemesanan di platform ini? |

**Jawaban LLM:** "Halo! Mohon maaf, saat ini saya belum memiliki informasi mengenai cara klaim asuransi kesehatan. Untuk bantuan lebih lanjut terkait masalah tersebut, silakan hubungi tim customer service kami. Terima kasih!"

**Analisis:** retrieval tidak menemukan FAQ yang benar-benar relevan (semua skor di bawah 0.65, top-1 hanya kemiripan kata "asuransi"). Yang penting: **LLM merespons dengan jujur** — mengakui tidak punya informasi dan menyarankan hubungi CS, sesuai instruksi prompt. Ini perilaku yang diharapkan.

---

### Query 6 — Tidak ada di KB (uji ketidaktahuan)
**Query:** "Apakah kalian menjual tiket pesawat?"

| Rank | Kategori | Similarity | FAQ |
|---|---|---|---|
| 1 | Pembayaran | 0.650 | Metode pembayaran apa saja yang tersedia? |
| 2 | Pemesanan | 0.636 | Apakah saya bisa memesan tanpa membuat akun? |
| 3 | Pemesanan | 0.625 | Apakah bisa memesan dalam jumlah besar untuk kebutuhan reseller? |

**Jawaban LLM:** "Halo! Mohon maaf, saat ini saya belum memiliki informasi mengenai penjualan tiket pesawat. Silakan hubungi tim customer service kami secara langsung untuk mendapatkan bantuan dan informasi lebih lanjut ya. Terima kasih!"

**Analisis:** retrieval tidak menemukan FAQ relevan (skor rendah, hasil hanya kemiripan topik "pemesanan/pembayaran"). LLM kembali merespons dengan jujur — mengakui tidak tahu. Perilaku konsisten dengan Query 5.

---

## Kesimpulan

1. **Semantic search bekerja dengan baik** untuk query yang jawabannya ada di KB — baik kata persis (sim ≥ 0.86) maupun parafrase/sinonim (sim ≥ 0.81). Kemampuan menangkap makna, bukan sekadar kata kunci, terbukti pada Query 3 dan 4.
2. **Skor similarity adalah sinyal kepercayaan yang berguna.** Query di luar KB menghasilkan skor < 0.65, jauh di bawah query relevan (≥ 0.81). Ambang ~0.70 bisa dipertimbangkan untuk memutuskan "informasi tersedia atau tidak".
3. **Prompt "jawab hanya dari context + akui ketidaktahuan" bekerja** — LLM tidak mengarang jawaban untuk topik di luar KB.
4. **Keterbatasan:** FAISS `IndexFlatIP` melakukan pencarian exact (brute force). Untuk 59 entri ini sangat cepat, tapi untuk skala jutaan dokumen perlu indeks aproksimasi (`IndexIVFFlat` / `HNSW`). Selain itu, kualitas retrieval bergantung pada model embedding — vektor dari model berbeda tidak bisa dicampur dalam satu index.