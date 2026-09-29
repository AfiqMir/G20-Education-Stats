# Laporan Prapemrosesan Data: World Bank EdStats

**Mata Kuliah:** Rekayasa Data (Data Engineering)  
**Topik Proyek:** Alur Kerja Data Preprocessing pada Dataset Pendidikan Global (World Bank EdStats): Integrasi 5 Tabel Lengkap dan Benchmarking Angka Partisipasi Murni (*Net Enrolment Rate*) Pendidikan Dasar Negara Anggota G20 (2000–2017)  
**Sumber Dataset Terbuka:** [Kaggle - World Bank Education Statistics](https://www.kaggle.com/datasets/theworldbank/education-statistics) | [World Bank EdStats Official Data Catalog](https://datacatalog.worldbank.org/search/dataset/0038480/Education-Statistics)  

---

## 1. Langkah 1: Impor Pustaka, Pemuatan 5 Berkas Data, dan Inspeksi Awal

### Justifikasi & Alasan Ilmiah
Langkah pertama dalam rekayasa data adalah memuat pustaka analisis utama (`pandas`, `numpy`, `scipy`, `sklearn`, `matplotlib`, `seaborn`) dan memuat seluruh lima berkas dataset mentah untuk memahami karakteristik dasarnya (*exploratory data inspection*).

Pada dataset World Bank EdStats:
- **Ukuran dan Dimensi Data**: Dataset utama (`EdStatsData.csv`) memiliki ukuran file yang relatif besar (~326.4 MB) dengan struktur tabel melebar (*wide format*), di mana setiap tahun dari 1970 hingga 2100 direpresentasikan sebagai kolom terpisah.
- **Deteksi Kualitas Awal**: Inspeksi awal diperlukan untuk mengetahui dimensi awal baris dan kolom, tipe data, jumlah nilai kosong (*missing values*), serta potensi kolom redundan (seperti kolom kosong tanpa nama di akhir file `Unnamed: 69` yang sering muncul akibat format ekspor CSV Bank Dunia).
- **Pemanfaatan Ekosistem 5 Berkas**: Kelima berkas dimuat secara menyeluruh untuk membangun skema bintang (*Star Schema*) yang kaya antara tabel fakta kuantitatif, tabel dimensi wilayah, taksonomi indikator, serta log metadata catatan kaki (*footnotes*).

### Profil Teknis 5 Berkas Dataset Mentah (*Initial Dataset Profiling*)

| Nama Berkas (*Entity/Relation*) | Peran Skema (*Schema Role*) | Kardinalitas (*Tuples/Rows*) | Aritas/Derajat (*Attributes/Columns*) | Ukuran File (Disk) | Alokasi RAM (*In-Memory*) | Kunci Penghubung (*Foreign Key*) |
| :--- | :--- | :---: | :---: | :---: | :---: | :--- |
| **`EdStatsData.csv`** | **Fact Table (Utama)** | **886,930** | **70** | **~326.4 MB** | **~706.3 MB** | `Country Code`, `Indicator Code` |
| **`EdStatsCountry.csv`** | **Dimension Table (Negara)** | **241** | **32** | **~139.5 KB** | **~0.41 MB** | `Country Code` |
| **`EdStatsSeries.csv`** | **Dimension Table (Indikator)**| **3,665** | **21** | **~3.7 MB** | **~6.15 MB** | `Series Code` |
| **`EdStatsFootNote.csv`** | **Metadata Log / Provenance** | **643,638** | **5** | **~39.7 MB** | **~154.8 MB** | `CountryCode`, `SeriesCode`, `Year` |
| **`EdStatsCountry-Series.csv`** | **Cross-Reference Dimension** | **613** | **4** | **~48.9 KB** | **~0.13 MB** | `CountryCode`, `SeriesCode` |

---

### Struktur Atribut dan Skema Data Mentah

1. **`EdStatsData.csv` (Tabel Fakta Utama)**:
   - **Atribut Identifikasi Entitas (4 atribut)**: `Country Name` (*string*), `Country Code` (*string / ISO-3*), `Indicator Name` (*string*), `Indicator Code` (*string*).
   - **Atribut Temporal / Metrik Waktu (65 atribut)**: Kolom `1970` hingga `2100` (*numeric/float64*), yang merepresentasikan nilai observasi historis dan proyeksi indikator pendidikan per tahun (*wide format*).
   - **Atribut Anomali / Artefak Ekspor (1 atribut)**: `Unnamed: 69` (*null*), kolom kosong tak berlabel akibat *trailing comma* pada berkas CSV mentah.

2. **`EdStatsCountry.csv` (Tabel Dimensi Negara)**:
   - **Atribut Kunci**: `Country Code` (Primary Key).
   - **Atribut Klasifikasi & Sosioekonomi Utama**: `Region` (kawasan regional), `Income Group` (klasifikasi pendapatan ekonomi World Bank), `2-alpha code`, `Currency Unit`, `National accounts base year`, dll. (Total 32 atribut).

3. **`EdStatsSeries.csv` (Tabel Dimensi Indikator)**:
   - **Atribut Kunci**: `Series Code` (Primary Key, berelasi dengan `Indicator Code` pada tabel fakta).
   - **Atribut Deskriptif & Taksonomi**: `Topic` (kategori jenjang/bidang pendidikan), `Indicator Name`, `Short definition`, `Long definition`, `Unit of measure`, `Periodicity`, dll. (Total 21 atribut).

4. **`EdStatsFootNote.csv` (Tabel Log Metadata Survei)**:
   - **Atribut Kunci Komposit**: `CountryCode`, `SeriesCode`, `Year` (memiliki format string `YR2000`).
   - **Atribut Deskripsi Metodologi**: `DESCRIPTION` (merekam apakah data berupa hasil survei resmi nasional atau estimasi model UIS UNESCO).

5. **`EdStatsCountry-Series.csv` (Tabel Asosiasi Negara-Indikator)**:
   - **Atribut Kunci Komposit**: `CountryCode`, `SeriesCode`.
   - **Atribut Catatan Khusus**: `DESCRIPTION` (merekam spesifikasi metodologi survei rumah tangga lokal).

---

## 2. Langkah 2: Strategi Reduksi Data (*Data Reduction*)

### Justifikasi & Rasional Metodologis
Data mentah `EdStatsData.csv` memiliki volume yang masif (886.930 baris x 70 kolom, alokasi memori ~740 MB) dengan rasio kekosongan data (*sparsity*) yang sangat tinggi (>80%) jika diolah secara keseluruhan. Memproses keseluruhan dataset mentah secara langsung tidak efisien dan mengandung *noise* dari data proyeksi teoritis masa depan (>2017) maupun data historis lampau (<2000) yang memiliki keterbatasan pelaporan.

Oleh karena itu, diterapkan strategi reduksi data multidimensional:
1. **Reduksi Dimensi Temporal (*Temporal Slicing & Dimensionality Reduction*)**:
   - **Tindakan**: Mengeliminasi kolom tahun 1970–1999, kolom proyeksi 5-tahunan (2020–2100), serta atribut artifak `Unnamed: 69`. Rentang observasi difokuskan pada periode kontinu **2000 s.d. 2017** (18 atribut tahun).
   - **Alasan**: Periode 2000–2017 merupakan era target *Millennium Development Goals (MDGs)* dan awal *Sustainable Development Goals (SDGs)* yang memiliki rekam pencatatan data pendidikan paling konsisten, terstandarisasi, dan relevan dengan dinamika kontemporer.
2. **Seleksi Fitur Indikator Berbasis Domain (*Domain-Specific Feature Selection: Net Enrolment Focus*)**:
   - **Tindakan**: Memilih 6 indikator inti pada jenjang Pendidikan Dasar (*Primary Education*) dengan basis kohort usia yang sama (7–12 tahun):
     1. `SE.PRM.NENR`: *Net enrolment rate, primary, both sexes (%)* (Angka Partisipasi Murni Total).
     2. `SE.PRM.NENR.FE`: *Net enrolment rate, primary, female (%)* (Partisipasi Murni Siswa Perempuan).
     3. `SE.PRM.NENR.MA`: *Net enrolment rate, primary, male (%)* (Partisipasi Murni Siswa Laki-laki).
     4. `SE.PRM.CMPT.ZS`: *Primary completion rate, both sexes (%)* (Tingkat Kelulusan Pendidikan Dasar).
     5. `SE.PRM.ENRL.TC.ZS`: *Pupil-teacher ratio in primary education* (Rasio Murid-Guru).
     6. `SE.PRM.DROP.ZS`: *Cumulative drop-out rate to last grade of primary (%)* (Tingkat Putus Sekolah).
   - **Alasan**: Pemilihan *Net Enrolment Rate (NER)* memastikan pembilang dan penyebut menggunakan basis usia yang sama persis (7–12 tahun), sehingga seluruh metrik partisipasi berada pada rentang persentase alami ($0\% - 100\%$) sesuai standar target PBB (SDG 4.1).
3. **Penyaringan Entitas Geografis (*Geographical Scope: 19 Sovereign Countries of G20*)**:
   - **Tindakan**: Membatasi observasi pada 19 negara berdaulat anggota G20: Indonesia (`IDN`), Amerika Serikat (`USA`), China (`CHN`), Jepang (`JPN`), Jerman (`DEU`), Inggris (`GBR`), India (`IND`), Brazil (`BRA`), Perancis (`FRA`), Italia (`ITA`), Kanada (`CAN`), Korea Selatan (`KOR`), Meksiko (`MEX`), Rusia (`RUS`), Arab Saudi (`SAU`), Afrika Selatan (`ZAF`), Turki (`TUR`), Australia (`AUS`), dan Argentina (`ARG`).
   - **Alasan**: Memberikan tolok ukur (*benchmarking*) strategis terhadap kapasitas dan performa pendidikan dasar Indonesia di tengah kekuatan ekonomi dunia dengan representasi negara maju (OECD) dan negara berkembang besar (*emerging economies*).

---

## 3. Langkah 3: Integrasi Data Multi-Sumber (*5-Table Integration & Schema Matching*)

### Justifikasi & Rasional Metodologis
Data kuantitatif fakta pada `EdStatsData.csv` tidak memiliki label sosioekonomi, kategori topik indikator, maupun catatan metodologi survei. Oleh karena itu, seluruh 5 tabel diintegrasikan dengan menyelesaikan konflik skema dan representasi nilai:

1. **Resolusi Ketidakcocokan Kunci (*Key Mismatch Resolution*)**:
   - Menghubungkan relasi antara `Indicator Code` (pada tabel fakta) dengan `Series Code` (pada tabel dimensi indikator `EdStatsSeries.csv`).
2. **Resolusi Konflik Nilai Representasi (*Data Value Representation Conflict Resolution*)**:
   - Kolom `Year` pada `EdStatsFootNote.csv` menggunakan format string `YR2000`, `YR2001`, dst. Dilakukan pembersihan ekspresi reguler (*regex parsing*) untuk mengekstrak angka integer `2000`–`2017` agar dapat digabungkan dengan kunci komposit `(CountryCode, SeriesCode, Year)`.
3. **Ekstraksi Fitur Provenance Data (*Data Provenance & Lineage*)**:
   - Dari kolom teks `DESCRIPTION` pada `EdStatsFootNote`, diekstraksi fitur biner baru: **`Is_Estimated`** (True = data hasil estimasi/model UNESCO UIS, False = data hasil sensus langsung).
4. **Redundansi Atribut & Uji Chi-Square ($\chi^2$)**:
   - Sesuai materi kuliah (Han & Kamber Bab 3: *Handling Redundancy in Data Integration*), uji Chi-Square antara `Income Group` dan `Region` menghasilkan $\chi^2 = 557.33$ ($df = 18, p = 8.80 \times 10^{-107} < 0.05$), membuktikan adanya dependensi regional-sosioekonomi yang sangat signifikan.
5. **Validasi Integritas Relasional (*Referential Integrity Validation*)**:
   - Penggabungan 5 tabel dieksekusi dengan operasi *Left Join* bertingkat. Hasil audit menunjukkan integritas referensial 100% tanpa adanya ledakan kartesian (*cartesian explosion*).

---

## 4. Langkah 4: Pembersihan Data (*Data Cleaning*) dan Transformasi Struktur

### Justifikasi & Rasional Metodologis
Data hasil reduksi dan integrasi masih memiliki struktur horizontal (*wide format*) yang menyulitkan pemodelan regresi, korelasi, dan analisis *time-series*. Selain itu, terdapat *missing values* sebesar 46,54% pada observasi panel mentah sebelum imputasi. Oleh karena itu, diterapkan serangkaian pembersihan data:

1. **Restrukturisasi Tidy Data (*Wide-to-Long via Unpivot/Melt*) & Format Panel (*Long-to-Feature Pivot*)**:
   - **Tindakan**: Melakukan transformasi unpivot (`pd.melt`) untuk menyatukan atribut tahun (`2000`–`2017`) ke dalam satu atribut temporal `Year` (*integer*), lalu memutarbalikkan (*pivot*) indikator menjadi kolom fitur analitis.
   - **Alasan**: Format panel (*Tidy Data Principle*) menghasilkan tepat **342 observasi unik (19 Negara x 18 Tahun)** di mana setiap kolom merepresentasikan variabel fitur kuantitatif independen.
2. **Strategi Imputasi Cerdas Bertingkat (*Hierarchical Domain-Specific Imputation*)**:
   - Menghindari pengisian sembarangan (*naive mean/zero imputation*) yang dapat merusak autokorelasi serial atau mengaburkan perbedaan sosioekonomi antarnegara:
     - **Tingkat 1 (Interpolasi Linier Temporal per Negara)**: Mengisi kekosongan data historis di antara dua titik waktu yang ada untuk negara yang sama (`group.interpolate(method='linear')`), menjaga kontinuitas tren waktu.
     - **Tingkat 2 (Imputasi Median Berbasis Income Group)**: Untuk negara yang tidak memiliki observasi pada rentang awal/akhir, nilai diestimasi menggunakan nilai median negara-negara sebaya dalam kelompok pendapatan ekonomi yang sama (*peer income group*).
     - **Tingkat 3 (Median Global G20)**: *Fallback* pengisian terakhir menggunakan nilai median kelompok G20 untuk menjamin 0% missing value tanpa mendistorsi skala data.
3. **Deteksi Outlier & Verifikasi Konsistensi Batas Logis (*Data Consistency Checks*)**:
   - Melakukan evaluasi statistik deskriptif dan visualisasi boxplot terhadap domain indikator:
     - Angka Partisipasi Murni (`SE.PRM.NENR`): Berada pada rentang logis 79,23% s.d. 100,00% (rata-rata 95,21%), membuktikan cakupan akses pendidikan dasar yang sangat kuat di G20.
     - Rasio Murid-Guru (`SE.PRM.ENRL.TC.ZS`): Berkisar antara 10,33 hingga 41,33 (mencerminkan variasi kapasitas ruang kelas nyata di G20).
     - Tingkat Putus Sekolah (`SE.PRM.DROP.ZS`): Rata-rata 6,93% dengan nilai terendah 0,02% (Jepang/Jerman) dan tertinggi 40,99% (India awal dekade 2000-an).

---

## 5. Langkah 5: Reduksi Dimensi Lanjutan dengan PCA (*Principal Component Analysis*)

### Justifikasi & Rasional Metodologis
Untuk mengatasi *curse of dimensionality* dan menyederhanakan 6 indikator kuantitatif ke dalam representasi ruang ortogonal yang lebih ringkas:
1. **Standarisasi Z-Score ($Z = \frac{X - \mu}{\sigma}$)**: Seluruh indikator diskalakan agar memiliki rata-rata 0 dan variansi 1 untuk mencegah indikator berskala besar mendominasi proses ekstraksi variansi.
2. **Eigendecomposition & Scree Plot PCA**:
   - Komponen Utama 1 (**PC1**): Menjelaskan **70,71%** dari total variansi data (Sumbu Akses Partisipasi & Kelulusan).
   - Komponen Utama 2 (**PC2**): Menjelaskan **13,76%** dari total variansi data (Sumbu Rasio Beban Pengajaran Guru).
   - **Total Informasi Tersimpan**: Dengan mereduksi 6 dimensi indikator menjadi hanya **2 Komponen Utama (2D)**, kita berhasil mempertahankan **84,47%** dari total variansi dan informasi data asli (terbukti melampaui ambang batas standar 80% pada *Scree Plot*).

---

## 6. Langkah 6: Diskretisasi Data & Pembentukan Konsep Hierarki (*Concept Hierarchy*)

### Justifikasi & Rasional Metodologis
Berdasarkan materi kuliah mengenai *Data Discretization (Interval & Concept Hierarchy Binning)*, teknik diskretisasi diterapkan pada dua indikator kunci untuk membentuk konsep hierarki kategorikal yang seimbang di negara-negara G20:
1. **Kelulusan Pendidikan Dasar (*Primary Completion Rate*)**:
   - **Kelulusan Rendah ($< 96\%$)**: Menghadapi tantangan efisiensi penuntasan pendidikan dasar (contoh: China $92,47\%$).
   - **Kelulusan Sedang ($96\% - 99\%$)**: Penuntasan stabil di tingkat menengah G20 (contoh: India $97,55\%$).
   - **Kelulusan Tinggi ($> 99\%$)**: Pencapaian kelulusan universal penuh dan penuntasan siswa lintas usia (contoh: Jerman $102,58\%$, Jepang $102,06\%$, Indonesia $102,00\%$, AS $100,62\%$, UK $100,62\%$, Brasil $100,68\%$).
2. **Rasio Murid per Guru (*Pupil-Teacher Ratio*)**:
   - **Rasio Ideal ($< 15$ murid/guru)**: Kapasitas pengajaran sangat optimal (contoh: Jerman $12,22$, AS $14,54$).
   - **Rasio Moderat ($15 - 20$ murid/guru)**: Kapasitas pengajaran standar (contoh: Indonesia $16,56$, Jepang $16,45$, China $16,29$, UK $17,39$).
   - **Rasio Padat ($> 20$ murid/guru)**: Beban pengajaran tinggi (contoh: India $31,49$, Brasil $20,92$).

> *Catatan Metodologis Metrik*: Sesuai standar resmi UNESCO UIS (*ISCED Level 1 - Primary Education* / Pendidikan Dasar), indikator kelulusan (`SE.PRM.CMPT.ZS`) dihitung menggunakan metode *Gross Intake Ratio to Last Grade*. Angka di atas 100% pada beberapa negara mencerminkan akumulasi siswa lintas usia (*over-age graduates*) yang lulus bersama siswa usia standar, dan bukan berarti ketiadaan siswa putus sekolah.

---

## 7. Temuan Analitis Kunci & Visualisasi Wawasan (*Analysis & Key Insights*)

1. **Posisi Benchmarking Indonesia di G20**:
   - Angka Partisipasi Murni (NER) Indonesia stabil di kisaran **96%–98%**, sejajar dengan negara-negara berkembang maju dan mendekati negara-negara OECD (~99%).
   - Rasio Murid-Guru Indonesia berada pada angka rata-rata ~16-17 murid/guru (2015), menempatkan Indonesia pada kategori **Rasio Moderat** di G20 (jauh lebih baik dibanding India yang >31 murid/guru).
2. **Paritas Gender di G20 (*Gender Parity Scatter*)**:
   - Plot paritas gender menunjukkan seluruh observasi negara G20 berkumpul sangat rapat pada garis diagonal $y=x$, membuktikan bahwa disparitas akses pendidikan antara anak laki-laki dan perempuan telah berhasil dihilangkan di seluruh negara anggota G20.
3. **Peta Panas Evolusi Pendidikan (*Macro Temporal Heatmap*)**:
   - Visualisasi heatmap 19 negara sepanjang 2000–2017 memperlihatkan konvergensi global di mana negara-negara berkembang G20 (seperti India dan Turki) mengalami peningkatan partisipasi yang sangat pesat dari dekade awal 2000-an menuju penuntasan universal di 2017.
4. **Studi Kasus Efektivitas Imputasi Deret Waktu (*Before vs. After: Indonesia & India*)**:
   - **Kasus Indonesia (Gap di Ujung)**: Menyambungkan ketiadaan data sensus di tahun 2000–2001 dan 2016–2017 secara mulus mengikuti batas tren riil nasional.
   - **Kasus India (Gap Tepat di Tengah)**: Menunjukkan keandalan interpolasi linier dalam merekonstruksi data yang terputus di tahun 2004, 2005, dan 2006 (dari $84.1\%$ di 2003 menuju $91.3\%$ di 2007) sehingga kurva deret waktu tersambung utuh tanpa mengubah arah akselerasi program pendidikannya.

---

## 8. Ringkasan Evaluasi Kuantitatif: Sebelum vs. Sesudah Prapemrosesan

| Parameter / Indikator Kualitas | Data Mentah Global (*EdStatsData*) | Data Panel G20 Mentah (Filter 3D) | Data Panel G20 Bersih (*Cleaned*) | Keterangan Perubahan Metodologis |
| :--- | :---: | :---: | :---: | :--- |
| **Bentuk Struktur Tabel** | *Wide Format* (70 Kolom) | *Tidy Panel Format* (342 Baris) | *Tidy Panel ML Ready* | Terstruktur rapi (*Country-Year*) |
| **Jumlah Tabel Sumber** | Terpisah di 5 berkas | Terintegrasi 5 Berkas | Terintegrasi Utuh (*Star Schema*) | Resolusi konflik `YR2000` $\to$ `2000` |
| **Kardinalitas Baris** | 886.930 baris mentah | 342 observasi panel | 342 observasi panel lengkap | Fokus pada 19 negara G20 (2000–2017) |
| **Jumlah Fitur/Atribut** | 70 kolom mentah | 6 indikator inti | 13 atribut (+ 2 PC & 2 Kategori) | Retensi variansi PCA sebesar 84,47% |
| **Rentang Partisipasi** | Nilai >100% pada GER | 79,23% – 100,00% (NER) | 79,23% – 100,00% (NER) | Basis pembilang & penyebut sama |
| **Persentase Nilai Kosong** | **86,10%** (*Extremely Sparse*) | **46,54%** (*Raw Panel Gap*) | **0,00%** (*Fully Clean*) | Imputasi berjenjang 3-Tier sukses |
| **Ukuran Berkas di Disk** | ~326,4 MB | ~55,4 KB | ~45,8 KB | Efisiensi penyimpanan >99,9% |
| **Ukuran Memori RAM** | ~740,56 MB | ~0,05 MB | ~0,04 MB | Efisiensi komputasi optimal |

---

## 9. Kesimpulan (*Conclusion*)

Alur kerja prapemrosesan data pada World Bank Global Education Statistics telah berhasil mengeksekusi integrasi **5 Tabel Lengkap**, **Data Reduction**, **Data Value Representation Conflict Resolution**, **Data Cleaning**, **Data Transformation**, dan **Dimensionality Reduction (PCA)** secara terintegrasi. Dengan mengadopsi indikator **Angka Partisipasi Murni (*Net Enrolment Rate / NER*)**, seluruh metrik partisipasi pendidikan berada pada skala persentase yang logis dan natural ($0\% - 100\%$) dengan basis usia pembilang dan penyebut yang konsisten, mempertahankan lebih dari 84% variansi informasi esensial untuk pemodelan data mining.

---
