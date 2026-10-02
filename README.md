# Hotel Booking Cancellation Prediction

Final project untuk memprediksi risiko pembatalan reservasi hotel sejak awal proses booking, sehingga manajemen dapat menentukan tindakan mitigasi (kebijakan deposit, komunikasi retensi, atau *controlled overbooking*) yang lebih tepat sasaran dan mengurangi kamar kosong.

## Penyusun

| No. | Nama |
|---|---|
| 1 | Maulana Imam Rifai |
| 2 | Tahlirska Elfadhila Sari Wrastamy |

## Dashboard dan Aplikasi

- 📊 **[Dashboard Tableau](https://public.tableau.com/views/HotelCancellationDashboard_17908686515740/Dashboard1?:language=en-US&publish=yes&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link)**
- 🚀 **[Aplikasi Streamlit](https://hotel-cancellation-pre.streamlit.app/)**

## Latar Belakang dan Tujuan

Pembatalan reservasi menimbulkan kerugian finansial dan mengganggu perencanaan operasional hotel. Pembatalan mendadak membuat hotel kehilangan peluang menjual kembali kamar, terutama pada periode permintaan tinggi. Kemudahan pemesanan lewat Online Travel Agency (OTA) membuat pembatalan semakin mudah dilakukan.

Proyek ini memiliki dua tujuan:

1. Membangun model klasifikasi yang memprediksi probabilitas pembatalan sejak awal proses booking.
2. Mengidentifikasi faktor yang memengaruhi pembatalan agar perencanaan operasional dapat difokuskan pada booking berisiko.

Model dievaluasi dengan **PR-AUC** (metrik utama untuk data tidak seimbang) serta **recall kelas batal**. Alasannya, dalam simulasi bisnis proyek ini, kegagalan mendeteksi booking yang akhirnya batal (False Negative) dianggap jauh lebih mahal daripada memberi tindak lanjut pada tamu yang sebenarnya datang (False Positive).

## Data dan Definisi Target

Dataset: `hotel_bookings_dataset.csv` (119.390 baris, 32 kolom) berisi reservasi City Hotel dan Resort Hotel. Target adalah `is_canceled` (1 = dibatalkan, 0 = tidak dibatalkan).

Alur pembersihan data:

| Tahap | Perlakuan | Jumlah baris |
|---|---|---:|
| Data mentah | – | 119.390 |
| Hapus duplikat | 31.994 baris duplikat (26,8%) dibuang | 87.396 |
| Missing value | `company` (94,0% hilang) dan `agent` (14,0% hilang) dibuang; 452 baris `country` kosong dibuang; `children` kosong diisi 0 | 86.944 |
| Anomali | Booking tanpa tamu (161), `adr` negatif (1), `adr` ≥ 5.000 (1), kategori `Undefined` pada `market_segment` (2) dan `distribution_channel` (5) dibuang | **86.776** |

Distribusi target:

| Kondisi | Tidak batal (0) | Batal (1) | Proporsi batal |
|---|---:|---:|---:|
| Data mentah | 75.166 | 44.224 | 37,04% |
| Setelah cleaning | 62.801 | 23.975 | **27,63%** |

Penurunan proporsi batal dari ±37% ke ±28% terutama disebabkan oleh pembuangan duplikat. Selisih ini perlu divalidasi dengan data internal perusahaan.

### Pencegahan *Data Leakage*

Karena model harus bekerja sejak awal proses booking, kolom yang nilainya baru diketahui setelah kejadian dibuang:

| Kolom | Alasan |
|---|---|
| `reservation_status` | Pada dasarnya adalah target dalam bentuk lain |
| `reservation_status_date` | Untuk booking batal, ini adalah tanggal pembatalan |
| `assigned_room_type` | Kamar yang dialokasikan baru diketahui mendekati check-in |
| `arrival_date_year`, `arrival_date_week_number`, `arrival_date_day_of_month` | Dibuang agar model tidak menghafal periode spesifik; informasi musiman tetap lewat `arrival_date_month` |

Hasil akhirnya adalah 23 fitur prediktor (9 kategorikal, 14 numerik).

## Alur Analisis

1. **Business Understanding**: perumusan masalah dan tujuan.
2. **Data Understanding dan Cleaning**: duplikat, missing value, anomali, dan pembuangan fitur *leakage*.
3. **EDA**: univariate, bivariate, multivariate (korelasi dan VIF), serta 5 pertanyaan bisnis.
4. **Preprocessing**: pengelompokan `country` (15 negara terbanyak + `Other`), One-Hot Encoding untuk fitur kategorikal, `StandardScaler` untuk fitur numerik, dan pembagian data train–test 80:20 dengan stratifikasi target (`random_state=42`).
5. **Penanganan imbalance**: `class_weight='balanced'` dan `scale_pos_weight` (rasio kelas 0:1 = 2,619), tanpa SMOTE.
6. **Benchmarking** 8 model, lalu **cross-validation 5-fold** pada 3 model teratas.
7. **Hyperparameter tuning** dengan `RandomizedSearchCV` (3-fold, scoring `average_precision`), lalu pemilihan model final dengan cross-validation 5-fold.
8. **Evaluasi test set**, optimasi threshold berbasis biaya bisnis, feature importance (native dan SHAP), serta perbandingan biaya dengan dan tanpa model.
9. **Penyimpanan model** dan fungsi deteksi booking berisiko.

## Temuan EDA

Tidak ada fitur numerik yang berkorelasi linear kuat dengan pembatalan (semua |r| < 0,2) dan seluruh VIF < 2, sehingga tidak ada masalah multikolinearitas. Pola pembatalan lebih banyak muncul dari interaksi antar fitur.

| Faktor | Temuan |
|---|---|
| **Lead time** | Cancellation rate naik hampir monoton: 8,5% (0–7 hari), 25,4% (8–30), 32,1% (31–90), 35,1% (91–180), 39,7% (181–365), 40,8% (>365) |
| **Tipe hotel** | City Hotel 30,1% vs Resort Hotel 23,7%. Pola ini terbalik pada `customer_type = Group` (Resort 10,9% vs City 9,0%) |
| **Musim** | Lebih tinggi pada April–Agustus (29,3%–32,2%) dibanding Januari–Maret dan September–November (21,3%–24,6%) |
| **Segmen pasar** | Online TA memiliki rate tertinggi (35,4%) sekaligus volume terbesar (51.478 booking); Groups di urutan kedua (27,1%) |
| **Tipe pemesan** | Transient 30,3%, Contract 16,3%, Transient-Party 15,2%, Group 9,9% |
| **Special request dan parkir** | Berkorelasi negatif dengan pembatalan (r = −0,122 dan −0,184); tamu yang meminta fasilitas lebih berkomitmen |
| **Deposit** | `Non Refund` memiliki rate **94,7%** (1.036 booking), jauh di atas `No Deposit` (26,8%) dan `Refundable` (24,3%) |

**Anomali `Non Refund`.** Temuan ini berlawanan dengan asumsi umum bahwa deposit non-refund menekan pembatalan. Dari 1.036 booking `Non Refund`, 982 berasal dari Portugal dan 893 melalui channel TA/TO, dengan segmen terbesar Groups (659) dan Offline TA/TO (285). Pola ini mengarah pada perilaku channel atau agen tertentu, bukan perilaku tamu individu, dan perlu diaudit lebih lanjut.

## Fitur Model

Model memakai 23 fitur setelah cleaning dan pembuangan kolom *leakage*.

| Kelompok | Fitur |
|---|---|
| Kategorikal (9) | `hotel`, `arrival_date_month`, `meal`, `market_segment`, `distribution_channel`, `reserved_room_type`, `deposit_type`, `customer_type`, `country_grp` |
| Numerik (14) | `lead_time`, `stays_in_weekend_nights`, `stays_in_week_nights`, `adults`, `children`, `babies`, `is_repeated_guest`, `previous_cancellations`, `previous_bookings_not_canceled`, `booking_changes`, `days_in_waiting_list`, `adr`, `required_car_parking_spaces`, `total_of_special_requests` |

`country_grp` adalah hasil pengelompokan `country` menjadi 15 negara terbanyak ditambah `Other`.

## Model dan Hasil Evaluasi

### Benchmarking (test set, parameter near-default)

| Model | Recall | Precision | F1 | ROC-AUC | PR-AUC |
|---|---:|---:|---:|---:|---:|
| **CatBoost** | 0,838 | 0,605 | 0,703 | 0,898 | **0,779** |
| XGBoost | 0,826 | 0,601 | 0,696 | 0,895 | 0,771 |
| LightGBM | 0,832 | 0,591 | 0,691 | 0,891 | 0,763 |
| Random Forest | 0,838 | 0,541 | 0,658 | 0,878 | 0,737 |
| Gradient Boosting | 0,542 | 0,732 | 0,623 | 0,874 | 0,734 |
| Decision Tree | 0,785 | 0,570 | 0,660 | 0,860 | 0,684 |
| KNN | 0,540 | 0,697 | 0,608 | 0,845 | 0,679 |
| Logistic Regression | 0,776 | 0,528 | 0,628 | 0,837 | 0,665 |

Tiga model boosting teratas berdasarkan PR-AUC (CatBoost, XGBoost, LightGBM) dibawa ke tahap tuning.

### Tuning dan Cross-Validation 5-Fold (train set)

| Model (tuned) | Recall | Precision | F1 | ROC-AUC | PR-AUC |
|---|---:|---:|---:|---:|---:|
| **LightGBM** | 0,820 | 0,604 | 0,696 | 0,894 | **0,768** |
| XGBoost | 0,813 | 0,605 | 0,694 | 0,893 | 0,766 |
| CatBoost | 0,824 | 0,581 | 0,682 | 0,885 | 0,750 |

Selisih PR-AUC antar model kecil (< 0,02), sehingga pemilihan algoritma boosting bukan faktor pembeda utama. **LightGBM** dipilih sebagai model final karena skor rata-rata cross-validation tertinggi.

Hyperparameter terbaik LightGBM: `subsample=0.85`, `num_leaves=63`, `n_estimators=350`, `learning_rate=0.05`, `colsample_bytree=0.7`.

### Evaluasi pada Test Set (17.356 booking, 4.795 batal)

| Metrik | Threshold 0,5 | Threshold optimal 0,2 |
|---|---:|---:|
| Recall kelas batal | 83,5% | **97,4%** |
| Precision kelas batal | 61,0% | 44,5% |
| F1 kelas batal | 70,5% | 61,1% |
| Accuracy | 80,7% | 65,7% |
| ROC-AUC | 0,9003 | 0,9003 |
| PR-AUC | 0,7806 | 0,7806 |

Confusion matrix pada threshold 0,2, dengan **kelas positif = batal**:

| Aktual / Prediksi | Tidak batal (0) | Batal (1) |
|---|---:|---:|
| Tidak batal (0) | TN = 6.727 | FP = 5.834 |
| Batal (1) | FN = 123 | TP = 4.672 |

Model menangkap 4.672 dari 4.795 booking yang batal. Sebagai konsekuensinya, 10.506 dari 17.356 booking ditandai berisiko, termasuk 5.834 tamu yang sebenarnya datang. Tindak lanjut yang dipakai sebaiknya berbiaya rendah.

### Feature Importance

Urutan berdasarkan importance bawaan LightGBM (jumlah split), dilengkapi SHAP summary plot di notebook:

| Peringkat | Fitur | Arah pengaruh |
|---:|---|---|
| 1 | `adr` | Harga lebih tinggi → risiko batal lebih tinggi |
| 2 | `lead_time` | Lead time lebih panjang → risiko batal lebih tinggi |
| 3 | `stays_in_week_nights` | – |
| 4 | `total_of_special_requests` | Lebih banyak permintaan → lebih jarang batal |
| 5 | `stays_in_weekend_nights` | – |
| 6 | `booking_changes` | – |
| 7 | `country_grp_PRT` | Pola pembatalan khas Portugal |
| 8 | `hotel_City Hotel` | – |

`market_segment_Online TA` juga masuk 15 fitur teratas. Temuan model konsisten dengan EDA, sehingga model belajar pola yang masuk akal secara bisnis.

## Simulasi Biaya

Simulasi menggunakan asumsi **biaya retensi 1 unit per FP** dan **biaya kamar kosong 10 unit per FN**:

- Kerugian = `(1 × FP) + (10 × FN)`
- Keuntungan = `(1 × TN) + (10 × TP)`
- Bersih = Keuntungan − Kerugian

Threshold dipilih dengan menelusuri 0,10 sampai 0,625 (kelipatan 0,025) dan mengambil nilai dengan keuntungan bersih tertinggi, yaitu **0,2**.

| Strategi | FP | FN | Total kerugian | Keuntungan bersih |
|---|---:|---:|---:|---:|
| Anggap semua tidak batal | 0 | 4.795 | 47.950 | −35.389 |
| Anggap semua batal (*full overbooking*) | 12.561 | 0 | 12.561 | 35.389 |
| **Dengan model (LightGBM, threshold 0,2)** | 5.834 | 123 | **7.064** | **46.383** |

Dibanding strategi "anggap semua tidak batal", kerugian turun 85,27%. Dibanding *full overbooking*, kerugian turun 43,76%. Angka ini merupakan simulasi berdasarkan asumsi biaya proyek, bukan biaya operasional aktual hotel.

## Cara Menjalankan

Instal dependensi (pengembangan memakai Python dengan scikit-learn 1.7.2, pandas 2.3.3, dan numpy 2.3.5):

```bash
python -m pip install pandas numpy scikit-learn matplotlib seaborn missingno statsmodels xgboost lightgbm catboost shap
```

Letakkan `hotel_bookings_dataset.csv` di folder yang sama dengan notebook, lalu jalankan `Finpro_Hotel_Booking_Cancellation.ipynb` dari atas ke bawah.

### Memakai Model yang Sudah Disimpan

File `final_model_hotel_cancellation.pkl` berisi model, threshold, daftar kolom fitur, dan daftar negara teratas.

```python
import pickle as pkl
import numpy as np
import pandas as pd

with open("final_model_hotel_cancellation.pkl", "rb") as f:
    artifact = pkl.load(f)

def prediksi_risiko_cancel(df_booking_baru, artifact=artifact):
    data = df_booking_baru.copy()

    # Grouping country yang sama seperti saat training
    if "country" in data.columns:
        top = artifact["top_countries"]
        data["country_grp"] = data["country"].apply(lambda x: x if x in top else "Other")
        data = data.drop(columns=["country"])

    missing = set(artifact["feature_columns"]) - set(data.columns)
    if missing:
        raise ValueError(f"Kolom berikut belum ada di data baru: {missing}")
    data = data[artifact["feature_columns"]]

    prob = artifact["model"].predict_proba(data)[:, 1]
    label = (prob >= artifact["threshold"]).astype(int)

    hasil = df_booking_baru.copy()
    hasil["prob_cancel"] = prob
    hasil["prediksi_risiko"] = np.where(label == 1, "Berisiko Batal", "Kemungkinan Datang")
    return hasil
```

Data baru harus memiliki kolom yang sama dengan data training, termasuk kolom `country` mentah, dan tanpa `is_canceled` serta kolom yang dibuang saat cleaning (`reservation_status`, `reservation_status_date`, `assigned_room_type`, `agent`, `company`, `arrival_date_year`, `arrival_date_week_number`, `arrival_date_day_of_month`).

## Berkas Utama

| Berkas | Fungsi |
|---|---|
| `Finpro_Hotel_Booking_Cancellation.ipynb` | Notebook cleaning, EDA, pemodelan, evaluasi, dan penyimpanan model |
| `hotel_bookings_dataset.csv` | Dataset reservasi hotel |
| `final_model_hotel_cancellation.pkl` | Model final, threshold, dan metadata fitur |
| `README.md` | Dokumentasi proyek |

## Kesimpulan dan Rekomendasi

1. **Terapkan konfirmasi ulang berbasis lead time.** Booking dengan lead time panjang (terutama > 90 hari) memiliki risiko batal jauh lebih tinggi. Konfirmasi ulang otomatis (WhatsApp, telepon, atau email) pada H-30 dan H-7 layak dipertimbangkan.
2. **Prioritaskan mitigasi pada Online TA dan Groups.** Online TA unggul pada rate dan volume, sedangkan Groups menyusul di urutan kedua.
3. **Audit skema deposit `Non Refund` di Portugal.** Cancellation rate 94,7% yang terkonsentrasi pada satu negara dan channel sangat tidak wajar.
4. **Gunakan *controlled overbooking* musiman.** Probabilitas pembatalan (`prob_cancel`) dapat dijumlahkan untuk memperkirakan *expected cancellations* per periode, terutama saat peak season April–Agustus, sebagai dasar kuota overbooking.
5. **Jalankan `prediksi_risiko_cancel()` di awal proses booking.** Booking yang ditandai berisiko diarahkan ke alur retensi (follow-up, insentif kecil, atau konfirmasi kehadiran) sebelum kamar hilang dari inventori yang dapat dijual ulang.

## Keterbatasan

- **Threshold dipilih pada test set.** Threshold 0,2 dioptimalkan menggunakan test set yang sama dengan evaluasi akhir, sehingga performa pada threshold tersebut cenderung optimistis. Untuk penggunaan operasional, threshold sebaiknya dipilih pada data validasi terpisah.
- **Asumsi biaya perlu dikalibrasi.** Rasio biaya 1 : 10 adalah asumsi proyek dan sebaiknya diganti dengan angka riil hotel.
- **Tuning terbatas.** `RandomizedSearchCV` memakai 8 iterasi (CatBoost) dan 12 iterasi (XGBoost, LightGBM) dengan 3-fold, sehingga ruang pencarian belum dijelajahi penuh.
- **Pembagian data acak.** Train–test dibagi secara acak dan berstratifikasi, bukan berdasarkan waktu. Untuk menguji kemampuan prediksi pada periode mendatang, lakukan validasi temporal.
- **Selisih proporsi batal (±37% vs ±28%)** akibat pembuangan duplikat perlu dikonfirmasi dengan data internal hotel.
- **Anomali `Non Refund`** masih berupa dugaan perilaku channel atau agen tertentu dan belum menjadi kesimpulan final.

## Teknologi

Python, pandas, NumPy, seaborn, Matplotlib, missingno, statsmodels, scikit-learn, XGBoost, LightGBM, CatBoost, SHAP, Tableau, dan Streamlit.
