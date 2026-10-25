# UTS Graf Sains Data: Sistem Rekomendasi RetailRocket dengan Neo4j

      Nama  : Fauzan Rafingudin  
      NIM   : 24031554111  
      Kelas : 2024B  

---
  
Repositori ini berisi pengerjaan UTS mata kuliah **Graf Sains Data** (S1 Sains Data, Universitas Negeri Surabaya, Gasal 2026/2027): membangun graf interaksi e-commerce di Neo4j, memproyeksikannya menjadi graf item-item, lalu membuat rekomendasi dengan KNN item-based dan mengevaluasinya dengan Hit@5 dan NDCG@5.

- **Kombinasi periode:** #22
  - P1 (training): 16-30 Juni 2015
  - P2 (testing): 01-15 Juli 2015
- **Dataset:** [RetailRocket ecommerce-dataset](https://www.kaggle.com/datasets/retailrocket/ecommerce-dataset) (`events.csv`)

## Isi repositori

| Berkas | Keterangan |
|---|---|
| `tutorial.ipynb` | Tugas pendahuluan 1.2: tutorial driver Neo4j (`execute_query`, simpul `Person`, relasi `KNOWS`) beserta jawaban checkpoint |
| `test.ipynb` | Tugas utama: impor data (1.3), 2.1 Wrangling dan EDA, 2.2 Analytics. Setiap nomor soal punya kode dan sel **Jawaban** sendiri |
| `events_p1.csv` | Hasil filter periode P1 (data training, dimasukkan ke graf) |
| `events_p2.csv` | Hasil filter periode P2 (data testing, ground truth evaluasi) |
| `requirements.txt` | Dependensi Python |

`events.csv` asli (sekitar 90 MB) **tidak disertakan** karena melebihi batas GitHub. Unduh dari Kaggle dan letakkan di `data/events.csv` bila ingin menjalankan ulang dari awal.

## Cara menjalankan

1. Buat environment dan pasang dependensi (Python 3.10 atau lebih baru):
   ```bash
   python -m venv myvenv
   source myvenv/bin/activate        # Windows: myvenv\Scripts\activate
   pip install -r requirements.txt
   ```
   Driver `neo4j` harus versi 5.x ke atas karena kode memakai `driver.execute_query`.
2. Jalankan Neo4j lokal (Neo4j Desktop atau Docker) di `neo4j://127.0.0.1:7687`.
3. Isi password lewat environment variable, bukan di dalam notebook:
   ```bash
   export NEO4J_PASSWORD="password_kamu"     # Windows PowerShell: $env:NEO4J_PASSWORD="password_kamu"
   ```
4. Letakkan `events.csv` di `data/`, lalu jalankan `tutorial.ipynb` dan `test.ipynb` dari sel paling atas. Bila database berisi data lama, kosongkan dulu dengan `MATCH (n) DETACH DELETE n` (hati-hati pada shared database), karena relasi dibuat dengan `CREATE` dan akan menggandakan data bila impor diulang.

## Ringkasan metode

1. **Filter periode:** data difilter pada `timestamp` (epoch milidetik) menjadi P1 dan P2, lalu disimpan sebagai CSV.
2. **Graf Neo4j:** hanya P1 yang diimpor. Simpul `Visitor` dan `Item`, relasi `VIEWED`, `ADDED_TO_CART`, dan `BOUGHT`. Constraint unik pada `visitorId` dan `itemId`.
3. **VSM:** matriks interaksi user x item (sparse) dengan bobot view = 1, addtocart = 3, transaction = 5.
4. **Proyeksi:** dipilih proyeksi **item-item** karena ukurannya sekitar 4 kali lebih kecil dan vektor item lebih padat daripada vektor user.
5. **Similarity:** cosine similarity pada 10.000 item terpopuler. Pasangan item dengan kurang dari 3 user bersama dibuang agar similarity tidak bergantung pada satu atau dua user. Lima item paling mirip dengan item acuan ditulis kembali ke graf sebagai relasi `SIMILAR_TO`.
6. **Rekomendasi:** KNN item-based (k = 5). Skor item = jumlah (bobot interaksi x cosine) dari tetangga terdekat tiap item yang pernah disentuh user, item yang sudah pernah disentuh dikeluarkan. User uji dipilih acak dengan seed tetap (`np.random.default_rng(42)`).
7. **Evaluasi:** Hit@5 dan NDCG@5 terhadap item yang disentuh user di P2 (semua jenis event, terbatas pada item kandidat), dibandingkan dengan baseline item terpopuler yang belum dilihat.

## Hasil utama

| Besaran | Nilai |
|---|---|
| Baris P1 / P2 | 309.830 / 333.778 |
| Simpul graf | 161.249 Visitor, 81.034 Item |
| Relasi graf | 299.133 VIEWED, 7.985 ADDED_TO_CART, 2.712 BOUGHT |
| VSM User | 161.249 x 81.034, density 1,83e-05, 80,9% user hanya menyentuh 1 item |
| Matriks similarity (setelah filter) | 10.000 x 10.000, 20.240 entri tidak nol, 3.250 item memiliki tetangga |
| User eligible (riwayat di P1 dan interaksi di P2) | 3.160 |
| User dengan minimal 3 rekomendasi | 1.283 (40,6%) |
| **Hit@5 KNN (seluruh user eligible)** | **0,0487** |
| **NDCG@5 KNN (seluruh user eligible)** | **0,0236** |
| Hit@5 baseline item terpopuler | 0,0041 |

KNN sekitar 12 kali lebih baik daripada baseline populer, tetapi nilai absolut rendah karena interaksi sangat sparse dan P2 hanya dua minggu.

## Asumsi dan keterbatasan

- **Bobot 1/3/5** adalah asumsi berdasarkan kekuatan sinyal minat, bukan hasil perhitungan dari data.
- **Ambang minimal 3 user bersama** dipilih agar similarity tidak rapuh. Tanpa filter ini, tetangga teratas hanya memiliki 1-2 user bersama. Nilai ambang belum dituning.
- **Cakupan:** hanya 3.250 dari 10.000 item (32,5%) yang punya tetangga setelah filter, sehingga hanya 40,6% user eligible yang mendapat minimal 3 rekomendasi. User tanpa rekomendasi dihitung 0 pada evaluasi.
- **Definisi relevan:** item yang disentuh user di P2 dengan jenis event apa pun, terbatas pada 10.000 item kandidat.
- Soal 2.2 no. 1b meminta 3 item iklan, sedangkan Hit@5 membutuhkan 5 rekomendasi. Notebook menampilkan 3 item teratas sebagai iklan dan mengevaluasi top-5 dari KNN yang sama.
- Query Cypher 7(a) memakai cosine biner (tanpa bobot) langsung dari graf sebagai cross-check, sehingga nilainya berbeda dari cosine berbobot di Python yang ditulis pada 7(b).
