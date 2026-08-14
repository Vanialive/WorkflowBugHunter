# WorkflowBugHunter

# Pest Control for Bus
Alur operasional pengendalian hama khusunya kkecoak untuk armada bus, dari penjadwalan sampai dokumentasi tracking.
Referensi: form inspeksi & rekomendasi tingkat infestasi kecoa dan jadwal fogging unit divisi Malang

## Alur proses
```mermaid
flowchart TD
    A["Jadwal & reminder visitasi<br/>Tracking H-1 / H-2 / H-3"] --> B["Kedatangan unit & inspeksi"]
    B --> C["Rekap & klasifikasi<br/>Ringan / Sedang / Berat"]
    C --> D["Rekomendasi Vendor"]
    D --> E{"Keputusan Customer<br>persetujuan/pilihan customer"}
    E -->|Setuju| F["Setuju paket<br/>Level = rekomendasi"]
    E -->|Pilih program lain| G["Pilih program lain<br/>Pilihan Customer"]
    E -->|Tunda| H["Tunda keputusan<br/>Belum eksekusi"]
    H -.kembali ke jadwal.-> A
    F --> I["Level Program Treatment<br/>Program fog/gel bait/sticky trap ssesuai level & decker"]
    G --> I
    I --> |Basic| J1["H0 - Basic<br/> fogging"]

```

### Catatan alur

- **Basic** = one-time treatment, selesai langsung di H0, tidak masuk cadence lanjutan.
- **Persetujuan tier** dan **eksekusi treatment** terjadi di visit yang sama (H0) — bukan proses terpisah yang bisa menunda treatment.
- **Pilih program lain** vs **diskresi komposisi lapangan** adalah dua hal berbeda:
  - *Pilih program lain* = keputusan customer soal **tier** (basic/standard/premium), dicatat sebagai rekomendasi vendor vs pilihan customer.
  - *Diskresi lapangan* = penyesuaian **komposisi material** (fog/bait/trap) di visit tertentu, independen dari tier mana yang dipilih.
- **H+28** bukan tahap treatment — itu batas siklus. Reorder setelah H+28 dihitung sebagai H0 baru.

## Skema database (ERD)

```mermaid
erDiagram
    CUSTOMER ||--o{ BUS : owns
    BUS ||--o{ JADWAL_VISITASI : scheduled
    JADWAL_VISITASI ||--o{ PLOTTING : assigns
    PERSONEL ||--o{ PLOTTING : assigned_to
    JADWAL_VISITASI ||--o{ INSPEKSI : produces
    INSPEKSI ||--o{ INSPEKSI_AREA : scores
    BUS ||--o{ PAKET_TREATMENT : has
    PAKET_TREATMENT ||--o{ TREATMENT_VISIT : includes
    TREATMENT_VISIT ||--o{ PEMAKAIAN_BAHAN : uses
    BAHAN ||--o{ PEMAKAIAN_BAHAN : consumed_in
    BUS ||--o{ INVOICE : billed

    CUSTOMER {
        uuid id PK
        string nama
        string kontak
    }
    BUS {
        uuid id PK
        uuid customer_id FK
        string kode_bus
        string jenis_decker
        string konfigurasi_seat
        string lokasi_garasi
    }
    JADWAL_VISITASI {
        uuid id PK
        uuid bus_id FK
        date tanggal
        time jam_estimasi
        time jam_selesai_operasional
    }
    PLOTTING {
        uuid id PK
        uuid jadwal_id FK
        uuid personel_id FK
        string alat_bahan
    }
    PERSONEL {
        uuid id PK
        string nama
        string role
    }
    INSPEKSI {
        uuid id PK
        uuid jadwal_id FK
        string tipe
        int jumlah_area_diperiksa
        int jumlah_area_aktif
        bool nimfa_ditemukan
        bool ootheca_ditemukan
        bool koloni_aktif_ditemukan
        string hasil_klasifikasi
    }
    INSPEKSI_AREA {
        uuid id PK
        uuid inspeksi_id FK
        string nama_area
        int skor
    }
    PAKET_TREATMENT {
        uuid id PK
        uuid bus_id FK
        string level_rekomendasi_sistem
        string level_disepakati
        date tanggal_mulai
        string status_approval
    }
    TREATMENT_VISIT {
        uuid id PK
        uuid paket_id FK
        string hari_ke
        date tanggal
        string dokumentasi_before
        string dokumentasi_after
    }
    BAHAN {
        uuid id PK
        string nama_bahan
        string satuan
    }
    PEMAKAIAN_BAHAN {
        uuid id PK
        uuid treatment_visit_id FK
        uuid bahan_id FK
        float jumlah_standar
        float jumlah_aktual
    }
    INVOICE {
        uuid id PK
        uuid bus_id FK
        string periode
        float jumlah
        uuid checker_id FK
        string status
    }
```

### Catatan skema

- `PAKET_TREATMENT.level_rekomendasi_sistem` vs `level_disepakati` memisahkan rekomendasi otomatis dari keputusan final customer — memungkinkan tracking seberapa sering customer override rekomendasi.
- `PEMAKAIAN_BAHAN.jumlah_standar` vs `jumlah_aktual` menyimpan standar dari tabel matrix (level + jenis decker) terpisah dari realisasi lapangan, sesuai bagian "Jumlah Material Terpasang" di form inspeksi.
- `INSPEKSI_AREA` menyimpan skor 0–3 per area (16 area) — sumber data untuk menghitung `jumlah_area_aktif` di tabel `INSPEKSI`.
