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
    F --> I["Level Program Treatment<br/>Program fog/gel bait/sticky trap <br>sesuai level & decker"]
    G --> I
    I --> |Basic| J1["H0 - Basic<br/> Fogging"]
    I --> |Standard| J2["H0 — Standard<br/>Fog + Bait + Trap"]
    I --> |Premium| J3["H0 - Premium<br>Fog + Bait + Trap,<br>takaran lebih besar"]
    J1 --> M["Dokumentasi & update<br>tracking"]
    J2 --> K1["H+14<br/>Ulang treatment yg sama —<br>2x/bulan"]
    J3 --> K2["H+7, H+14, H+21<br/>Ulang treatment yg sama —<br>4x/bulan"]
    K1 --> M
    K2 --> M

```

### Catatan alur
- Treatment terjadi sejak H0
- **Basic** = one-time treatment (fogging saja), selesai langsung di H0, tidak masuk cadence lanjut
- **Standard** → fogging + gel bait + sticky trap sejak H0, diulang di H+14 (2x/bulan)
- **Premium** → fogging + gel bait + sticky trap (porsi lebih banyak) sejak H0, diulang di H+7, H+14, H+21 (4x/bulan)
- **Persetujuan Program Treatment** dan **Eksekusi Treatment** terjadi di visit yang sama (H0) — bukan proses terpisah
- **Pilih program lain** vs **Diskresi komposisi lapangan** adalah dua hal berbeda:
  - *Pilih program lain* = keputusan customer soal **level paket program** (basic/standard/premium), dicatat sebagai Pilihan Customer
  - *Diskresi lapangan* = penyesuaian **komposisi material** (fog/bait/trap) di visit tertentu, independen dari level mana yang dipilih, berdasarkan koordinasi TR&D dan PIC Lapangan
- **H+28** bukan tahap treatment, jadi jika H+28 akan reorder dihitung sebagai H0 baru


## Skema database (ERD)
 
```mermaid
erDiagram
    CUSTOMER ||--o{ CUSTOMER_CABANG : has
    CUSTOMER_CABANG ||--o{ BUS : owns
    GARASI ||--o{ JADWAL_VISITASI : hosts
    BUS ||--o{ JADWAL_VISITASI : scheduled
    JADWAL_VISITASI ||--o{ PLOTTING : assigns
    PERSONEL ||--o{ PLOTTING : assigned_to
    JADWAL_VISITASI ||--o{ INSPEKSI : produces
    INSPEKSI ||--o{ INSPEKSI_AREA : scores
    AREA ||--o{ INSPEKSI_AREA : scored_in
    BUS ||--o{ PAKET_TREATMENT : has
    PAKET_TREATMENT ||--o{ TREATMENT_VISIT : includes
    TREATMENT_VISIT ||--o{ PEMAKAIAN_BAHAN : uses
    BAHAN ||--o{ PEMAKAIAN_BAHAN : consumed_in
    BUS ||--o{ INVOICE : billed
 
    CUSTOMER {
        uuid id PK
        string nama
    }
    CUSTOMER_CABANG {
        uuid id PK
        uuid customer_id FK
        string nama_cabang
        string wilayah
        string pic_approval
        string info_pembayaran
    }
    BUS {
        uuid id PK
        string kode_unik
        uuid cabang_id FK
        string nama_bus
        string no_polisi
        string model_bis
        string type_unit
        string seri
        string keterangan
        bool punya_pantry
        bool punya_toilet
        bool punya_dispenser
        bool punya_sleeper_pod
        bool punya_rak_bagasi_atas
        bool punya_lantai_atas
    }
    GARASI {
        uuid id PK
        string nama_garasi
        string kota
        string wilayah
    }
    JADWAL_VISITASI {
        uuid id PK
        uuid bus_id FK
        uuid garasi_id FK
        date tanggal
        time jam_estimasi
        time jam_kedatangan_aktual
        time jam_mulai_fogging_aktual
        time jam_berangkat_aktual
        string situasi_fogging
        string keterangan
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
        int jumlah_lokasi_nimfa
        int jumlah_lokasi_ootheca
        int jumlah_lokasi_koloni_aktif
        string hasil_klasifikasi
    }
    AREA {
        uuid id PK
        string nama_area
        int urutan
    }
    INSPEKSI_AREA {
        uuid id PK
        uuid inspeksi_id FK
        uuid area_id FK
        int skor
    }
    PAKET_TREATMENT {
        uuid id PK
        uuid bus_id FK
        string level_rekomendasi_sistem
        string level_disepakati
        string status_approval
        string catatan_keputusan
        bool konfirmasi_risiko_diterima
        string pic_vendor
        string pic_customer
        date tanggal_mulai
    }
    TREATMENT_VISIT {
        uuid id PK
        uuid paket_id FK
        string hari_ke
        date tanggal
        bool inspeksi_done
        bool fogging_done
        bool dokumentasi_lengkap
        string catatan_lapangan
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

- `BUS` murni data konfigurasi (kode, plat, jenis armada, fasilitas) — lokasi/garasi bukan atribut bus, tapi atribut per-visit di `JADWAL_VISITASI.garasi_id`.
- `GARASI` dan `AREA` adalah master data — mencegah duplikasi/typo teks bebas yang berulang di tiap baris inspeksi/jadwal.
- `PAKET_TREATMENT.level_rekomendasi_sistem` vs `level_disepakati` memisahkan rekomendasi otomatis dari keputusan final customer; selisih keduanya jadi indikator "pilih program lain".
- `PEMAKAIAN_BAHAN.jumlah_standar` vs `jumlah_aktual` menyimpan standar dari tabel matrix (level + jenis decker) terpisah dari realisasi lapangan (diskresi TR&D + PIC lapangan).
- `TREATMENT_VISIT.hari_ke` menyimpan label visit (H0, H+7, H+14, H+21) per paket (siklus) — Basic cukup satu baris `H0`, Standard/Premium beberapa baris sesuai cadence.

## Belum dimodelkan (open items)

Bagian ini sengaja belum jadi tabel — masih nunggu keputusan bisnis:

- Hak akses per role (management / lapangan / customer) — termasuk apakah visitasi ke-2/ke-3 kelihatan ke customer atau cuma hasil akhir.
- Detail invoicing (checker vs printer vs penanggung jawab).
- Link supply chain ke keuangan (selisih pemakaian bahan vs pengadaan).
- Perhitungan lembur/insentif HR dari data `PLOTTING`.

### Catatan skema

- `PAKET_TREATMENT.level_rekomendasi_sistem` vs `level_disepakati` memisahkan rekomendasi otomatis dari keputusan final customer — memungkinkan tracking seberapa sering customer override rekomendasi.
- `PEMAKAIAN_BAHAN.jumlah_standar` vs `jumlah_aktual` menyimpan standar dari tabel matrix (level + jenis decker) terpisah dari realisasi lapangan, sesuai bagian "Jumlah Material Terpasang" di form inspeksi.
- `INSPEKSI_AREA` menyimpan skor 0–3 per area (16 area) — sumber data untuk menghitung `jumlah_area_aktif` di tabel `INSPEKSI`.

- `PAKET_TREATMENT.level_rekomendasi_sistem` vs `level_disepakati` memisahkan rekomendasi otomatis dari keputusan final customer.
- `PEMAKAIAN_BAHAN.jumlah_standar` vs `jumlah_aktual` menyimpan standar dari tabel matrix (level + jenis decker) terpisah dari realisasi lapangan.
- `INSPEKSI_AREA` menyimpan skor 0–3 per area (16 area) — sumber data untuk menghitung `jumlah_area_aktif` di tabel `INSPEKSI`.
- `TREATMENT_VISIT.hari_ke` menyimpan label visit (H0, H+7, H+14, H+21) — komposisi per visit tetap merujuk ke `PEMAKAIAN_BAHAN`, jadi Basic (hanya H0, fogging) dan Standard/Premium (H0 + lanjutan, fog+bait+trap) sama-sama tercatat lewat struktur yang sama.
