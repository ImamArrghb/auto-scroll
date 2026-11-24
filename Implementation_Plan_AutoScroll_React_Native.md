# Implementation Plan — Auto-scroll TikTok untuk Android

Tanggal: 24 November 2026. Status: rencana implementasi, belum ada pengujian perangkat atau bukti bahwa posisi video TikTok dapat dibaca.

## 1. Tujuan dan batas keberhasilan

Membangun aplikasi Android berbasis React Native yang membantu pengguna berpindah ke video TikTok berikutnya setelah video selesai, terutama saat tangan tidak bisa digunakan. Target distribusi awal adalah APK untuk pengguna sendiri dan kelompok penguji; publikasi Play Store belum termasuk scope.

Risiko utama adalah keterbacaan status pemutaran TikTok, bukan pembuatan UI atau gestur swipe. Aksesibilitas Android tidak menjamin bahwa TikTok mengekspos durasi, posisi, status pause, maupun identitas video. Semuanya harus dibuktikan melalui prototipe perangkat nyata.

Mode interval adalah fitur terpisah yang dipilih pengguna. Keberhasilan mode interval tidak boleh dilaporkan sebagai keberhasilan deteksi akhir video. Jika aksesibilitas tidak menyediakan sinyal yang memadai, fitur inti dinyatakan belum layak dengan pendekatan ini.

## 2. Keputusan teknologi

| Komponen | Pilihan | Alasan |
|---|---|---|
| Aplikasi | React Native + TypeScript | UI, onboarding, pengaturan, status |
| Setup | Proyek React Native dengan direktori Android yang dikelola langsung | Membutuhkan service dan modul native khusus |
| Modul Android | Java | Menghindari ketergantungan pada penulisan Kotlin untuk fitur inti |
| Integrasi JS–Android | Turbo Native Module + Codegen | Kontrak bertipe antara TypeScript dan Java |
| Mesin otomatisasi | AccessibilityService + controller native | Tetap dapat bekerja ketika Activity React Native berada di background, selama service masih tersedia |
| Penyimpanan | SharedPreferences native | Satu sumber pengaturan untuk service dan UI |
| Pengujian | Jest untuk aturan TS, JUnit untuk logika native, uji perangkat nyata untuk TikTok | Pengujian mengikuti risiko |
| Distribusi | Signed release APK | Bisa dibagikan tanpa publikasi Play Store |

Pilih versi stabil React Native saat inisialisasi, kunci dependensi dan toolchain sesuai template resmi. Minimum SDK mengikuti versi RN terpilih; jangan menjanjikan dukungan semua Android sebelum build dan uji kompatibilitas. Expo Go tidak menjadi runtime prototipe karena service khusus perlu build native. Tidak diperlukan backend atau akun aplikasi pada versi awal.

## 3. Scope versi pertama

Termasuk:

- Onboarding yang menjelaskan aksesibilitas dan membawa pengguna ke pengaturan Android.
- Dashboard dengan status layanan, Mulai, Jeda, dan Hentikan.
- Mode akhir video hanya pada kombinasi perangkat/TikTok yang lolos validasi.
- Mode interval yang dipilih secara eksplisit; preset awal 15, 30, dan 60 detik, dapat disesuaikan.
- Sesi hanya untuk feed video vertikal TikTok standar dalam portrait.
- Jeda ketika konteks tidak didukung, sinyal hilang, ada dialog/komentar, atau aplikasi target tidak aktif.
- Hentikan sesi saat layar terkunci; tidak otomatis aktif setelah reboot atau service dibuat ulang.
- Kontrol jeda/hentikan dari notifikasi sesi jika izin dan dukungan platform tersedia. Tetap sediakan kontrol di aplikasi.
- Pengaturan lokal, status kompatibilitas, dan diagnostik teknis minimal.

Di luar versi pertama: TikTok Lite, aplikasi lain, kendali suara, AI pemilihan konten, like/follow/comment otomatis, login, backend, monetisasi, perekaman layar, OCR, dan pembaruan otomatis. LIVE dan postingan foto tidak diperlakukan sebagai video berdurasi biasa; jeda saat dikenali atau saat konteks tidak dapat dipastikan.

## 4. Arsitektur dan pembagian tanggung jawab

React Native menyediakan layar dan mengirim perintah konfigurasi/sesi. Turbo Native Module menjadi penghubung. AccessibilityService memiliki akses ke node UI dan mengeksekusi gesture; SessionController native menjadi sumber kebenaran status sesi. UI membaca snapshot saat kembali aktif dan menerima event saat runtime JS tersedia.

| Bagian | Tanggung jawab |
|---|---|
| HomeScreen | Status, mode aktif, kontrol sesi |
| PermissionScreen | Penjelasan akses dan tautan pengaturan; periksa ulang setelah kembali |
| SettingsScreen | Interval dan perilaku sesi |
| NativeAutoScroll spec | Perintah dan event bertipe |
| AutoScrollModule.java | Implementasi kontrak, validasi input, koneksi ke controller |
| AutoScrollAccessibilityService.java | Lifecycle Android, event window, akses node, dispatch gesture |
| SessionController.java | State machine, penjadwalan native, cancel pending work |
| TikTokUiAdapter.java | Pengenalan feed, indikator posisi, konten tidak didukung |
| PlaybackTracker.java | Menilai progres dan kemungkinan akhir video dari sampel berurutan |
| GestureExecutor.java | Satu swipe, callback selesai/batal, cooldown |
| SettingsStore.java | Menyimpan konfigurasi; status RUNNING tidak dipulihkan otomatis |
| SessionNotification.java | Kontrol native melalui notifikasi dan receiver |

Timer swipe tidak memakai setInterval JavaScript. Jangan menganggap service tidak dapat dihentikan sistem. Ketika service hilang, sesi berhenti dan UI menampilkan status sebenarnya. Foreground service tambahan tidak dibuat kecuali ada kebutuhan yang terbukti dan jenis/aturan Android yang sesuai.

## 5. Tahap 0 — bukti kelayakan teknis

Ini gerbang sebelum membangun aplikasi lengkap.

1. Inisialisasi proyek RN, build ke HP Android nyata, lalu buat AccessibilityService minimal.
2. Deklarasikan kemampuan membaca window dan melakukan gesture. Pengguna mengaktifkan layanan melalui pengaturan Android.
3. Dalam build development, inspeksi node TikTok untuk mencari resource ID, class, contentDescription, teks waktu, dan RangeInfo jika tersedia. Tidak mengasumsikan nama node atau struktur sebelum observasi.
4. Ambil sampel berurutan ketika video berjalan, dijeda, buffering, berpindah, seek manual, dan loop. Pastikan nilai yang ditemukan benar-benar posisi pemutaran, bukan kontrol lain.
5. Cari cara mengenali konteks feed aktif serta perubahan video. Pisahkan adapter agar perubahan UI TikTok tidak menyebar ke seluruh aplikasi.
6. Uji satu swipe dan konfirmasi perpindahan konten. Callback gesture selesai hanya membuktikan gestur selesai, bukan video sudah berubah.
7. Catat perangkat, versi Android, versi TikTok, bahasa UI, sinyal yang tersedia, dan batasannya.

Keputusan hasil:

- GO: progres dan konteks dapat dikenali secara cukup konsisten untuk melanjutkan prototipe akhir video pada konfigurasi teruji.
- LIMITED: swipe berfungsi tetapi progres tidak terbaca andal. Mode interval dapat dikembangkan, tetapi tidak dianggap memenuhi tujuan inti.
- NO-GO: konteks atau gesture tidak dapat dikendalikan cukup aman. Hentikan pengembangan fitur otomatis sampai pendekatan direvisi.

Jika hasil LIMITED/NO-GO, buat laporan singkat bukti dan opsi. OCR/progress-bar visual memerlukan eksperimen baru dan tidak ditambahkan diam-diam ke scope.

Output: debug APK, matriks kelayakan, dan keputusan teknis. Pengujian ini membutuhkan HP dan TikTok nyata; emulator saja tidak mencukupi.

## 6. Tahap 1 — mesin sesi native

State: IDLE, ARMED, OBSERVING, SWIPING, COOLDOWN, PAUSED, UNSUPPORTED, ERROR.

- Mulai hanya diterima jika service terhubung dan konfigurasi valid.
- ARMED menunggu feed TikTok yang dikenali. Kembali dari aplikasi lain memerlukan kelanjutan sesi yang jelas; default setelah jeda adalah pengguna menekan Lanjutkan.
- OBSERVING mengumpulkan sampel terkini dan memverifikasi konteks.
- SWIPING hanya mengizinkan satu gesture yang sedang berjalan.
- COOLDOWN menunggu konfirmasi perubahan video; jika tidak ada, jeda dan laporkan kegagalan, tanpa retry berulang.
- PAUSED/UNSUPPORTED/ERROR membatalkan penjadwalan dan tidak melakukan swipe.
- Hentikan selalu membatalkan sesi; ketika service disconnect atau proses dibuat ulang, kembali IDLE.

Sebelum setiap swipe, periksa ulang paket foreground/root window, konteks feed, layar menyala/tidak terkunci, konfigurasi, dan identitas generasi sesi. Gunakan token generasi untuk mengabaikan callback/timer lama setelah Stop atau pergantian video.

Pantau perubahan paket/window secukupnya untuk mengetahui pengguna meninggalkan TikTok; jangan membaca node aplikasi lain. Memfilter seluruh event hanya ke TikTok dapat membuat perpindahan ke aplikasi lain tidak terdeteksi, sehingga pengecekan window terbaru sebelum gesture tetap wajib.

Keterbatasan: gesture yang sudah dikirim dapat selesai sebelum pembatalan diproses. Hindari klaim bahwa Stop bisa menarik kembali semua gesture yang sudah dimulai.

## 7. Tahap 2 — deteksi akhir video

Urutan sumber sinyal yang diuji: node progress/range, teks waktu berjalan dan total, lalu indikator semantik lain yang benar-benar ditemukan. Tidak mengandalkan satu resource ID atau lama video berada di layar.

Aturan awal:

1. Validasi sampel milik feed dan video yang sama; timestamp memakai jam monotonik.
2. Butuhkan beberapa sampel konsisten untuk membuktikan progres berjalan.
3. Jangan menyamakan progres diam dengan video selesai; bisa berarti pause atau buffering.
4. Akhir hanya diputuskan berdasarkan bukti teruji bahwa progres mencapai akhir atau terjadi loop sesudah progres mendekati akhir.
5. Penurunan posisi besar tidak otomatis berarti loop; bisa seek manual. Jika ambigu, jeda.
6. Setelah swipe, reset sampel dan tunggu konten baru; jangan memakai progres video sebelumnya.
7. Jika sinyal hilang, kedaluwarsa, atau konteks tidak pasti, jeda dan tampilkan alasan. Tidak beralih otomatis ke interval.

Rentang normalisasi belum tentu satuan detik. Selisih waktu dan toleransi hanya ditetapkan setelah observasi. Jangan default ke swipe pada 95% durasi karena itu berpotensi memotong akhir video. Deteksi loop dapat menambah jeda singkat setelah akhir; ukur dan jelaskan pada pengguna jika itu satu-satunya sinyal andal.

Pemantauan berbasis event, ditambah polling native terbatas hanya selama sesi/target aktif jika diperlukan. Interval sampling, batas traversal node, dan timeout ditentukan dari pengukuran agar tidak membaca seluruh pohon UI terus-menerus.

Mode interval memakai jam monotonik dan menghitung ulang untuk video baru. Jika video baru/perubahan konteks tidak bisa diverifikasi, jeda daripada menebak. Mode ini boleh berpindah sebelum video selesai dan harus menjelaskan perilaku tersebut di UI.

## 8. Tahap 3 — UI React Native dan kontrak modul

Layar awal: onboarding, izin, dashboard, pengaturan, dan bantuan/kompatibilitas. Dashboard menampilkan mode serta status yang berbeda: izin belum aktif, siap, menunggu TikTok, mengamati, dijeda, tidak didukung, atau gagal.

Kontrak konseptual; bentuk spec final mengikuti batasan Codegen versi RN yang dipilih:

| Method/event | Isi |
|---|---|
| getServiceStatus() | Status koneksi service dan dukungan konfigurasi |
| openAccessibilitySettings() | Membuka pengaturan, bukan mengaktifkan izin otomatis |
| getSettings()/updateSettings(config) | Membaca dan memvalidasi mode/interval |
| startSession()/pauseSession()/resumeSession()/stopSession() | Perintah idempoten ke controller native |
| getSessionSnapshot() | Status, alasan, mode, dan revisi state terbaru |
| onSessionChanged | Event bertipe dengan state, reasonCode, revision |

Jangan mengirim instance AccessibilityNodeInfo atau seluruh teks layar ke JavaScript. UI menerjemahkan reasonCode menjadi pesan singkat. Listener dibersihkan saat unmount; getSessionSnapshot mengatasi event yang terlewat ketika UI berada di background.

## 9. Tahap 4 — izin dan distribusi APK

- Jelaskan fungsi aksesibilitas sebelum pengguna membuka pengaturan, termasuk bahwa aplikasi membaca elemen UI TikTok dan melakukan swipe.
- Aksesibilitas diaktifkan manual oleh pengguna; jangan menjanjikan tombol yang memberikan izin langsung.
- Rancang pembacaan UI agar lokal saja. Production build tidak menyimpan dump layar atau mengirim data konten.
- Notifikasi sesi memerlukan penanganan izin sesuai versi Android. Jika kontrol notifikasi tidak tersedia, jelaskan dan pertahankan Stop melalui aplikasi.
- Untuk APK yang dipasang langsung, sebagian perangkat/versi Android dapat membatasi pengaktifan aksesibilitas. Panduan bantuan mengikuti mekanisme pengaturan resmi yang tersedia; jangan mengakali pembatasan.
- Buat release APK yang ditandatangani, versionCode/versionName, checksum, catatan versi, panduan instalasi, daftar kompatibilitas, dan batasan diketahui.
- Simpan keystore/password di luar source control dan buat backup aman. Pembaruan APK memerlukan identitas paket dan kunci penandatanganan yang sama.
- Uji instalasi baru dan pembaruan di atas versi lama. Rilis hanya untuk kombinasi yang sudah diuji; publikasi Play Store dapat direncanakan terpisah.

## 10. Verifikasi dan kriteria penerimaan

| Skenario | Hasil yang diharapkan |
|---|---|
| Video pendek/panjang | Satu swipe sesudah akhir terverifikasi, tanpa memakai durasi tebakan |
| Pause dan buffering | Tidak menganggap posisi diam sebagai akhir |
| Seek manual | Tidak salah memutuskan loop/akhir |
| Swipe manual | Reset tracking; tidak terjadi swipe susulan dari timer lama |
| Komentar, share sheet, profil, pencarian | Jeda; tidak swipe pada konteks lain |
| LIVE/foto atau konteks tidak dikenali | UNSUPPORTED/PAUSED, tanpa gesture |
| Keluar TikTok/layar terkunci | Tidak mengirim gesture baru; sesi dijeda/dihentikan sesuai aturan |
| Stop saat timer/callback tertunda | Callback lama diabaikan; tidak ada gesture baru |
| RN Activity di background | Mesin native tetap bekerja selama service tersedia |
| Service mati/izin dicabut/reboot | Sesi berhenti; tidak aktif kembali diam-diam |
| Gesture selesai tanpa ganti video | Jeda dengan alasan; tidak melakukan retry loop |
| Notifikasi ditolak | UI tetap bisa menghentikan sesi dan menjelaskan kontrol yang tersedia |
| Instalasi/pembaruan APK | Pengaturan konsisten dan sesi tetap nonaktif setelah update |

Unit test difokuskan ke state machine, data progres, seek vs loop, sampel stale, token sesi, dan pencegahan swipe ganda. Integration test memverifikasi kontrak TS–Java dan lifecycle. Uji gesture/keterbacaan TikTok wajib dilakukan pada HP nyata.

Target penerimaan beta yang diusulkan, bukan hasil saat ini: minimal 100 akhir video yang didukung, tersebar pada setidaknya dua perangkat fisik dan dua versi Android; >=95% menghasilkan tepat satu swipe, nol swipe teramati sebelum akhir, serta nol gesture di luar konteks target dalam skenario uji. Ukur median/p95 keterlambatan setelah akhir; tetapkan batas berdasarkan hasil fase kelayakan sebelum pengujian beta. Laporkan denominator, kegagalan, cakupan, dan keterbatasan; nol kejadian dalam sampel bukan jaminan untuk semua kondisi.

Catat konsumsi CPU/baterai dan stabilitas sesi 30 menit. Ambang performa ditentukan terhadap baseline pada perangkat yang sama, bukan angka asumsi.

## 11. Backlog dan urutan pengerjaan

| Prioritas | Pekerjaan | Dependency | Bukti selesai |
|---|---|---|---|
| P0 | Setup RN dan build fisik | Toolchain | APK debug terbuka di HP |
| P0 | Service, inspeksi sinyal, swipe tunggal | Setup | Catatan kelayakan GO/LIMITED/NO-GO |
| P0 | SessionController dan pembatalan | Kelayakan | Uji state/lifecycle lulus |
| P0 | TikTokUiAdapter dan PlaybackTracker | Sinyal terverifikasi | Uji akhir/pause/seek/buffering |
| P1 | Turbo Native Module dan UI | Kontrak/controller | Status UI sesuai native |
| P1 | Mode interval eksplisit | Controller | Uji timer dan perubahan konteks |
| P1 | Kontrol notifikasi dan onboarding | Mesin sesi | Uji izin/Stop/background |
| P1 | Uji lintas perangkat dan perbaikan | Fitur terintegrasi | Matriks hasil dan batasan |
| P1 | Signed release APK dan panduan | Beta lulus | APK terpasang/terbarui pada HP penguji |

Estimasi awal untuk satu pengembang: 1–3 hari kelayakan, 3–5 hari mesin native, 2–4 hari integrasi/UI, dan 3–5 hari pengujian/paket beta; sekitar 9–17 hari kerja jika sinyal progres tersedia. Ini perkiraan perencanaan, bukan janji. Belajar modul native atau perubahan pendekatan deteksi dapat memperpanjangnya secara material.

## 12. Struktur kode yang disarankan

| Lokasi | Isi |
|---|---|
| src/screens/ | Home, Permissions, Settings, Help |
| src/components/ | Kontrol sesi dan indikator status |
| src/specs/NativeAutoScroll.ts | Spec Codegen |
| src/services/autoScroll.ts | Wrapper modul dan pemetaan error |
| src/hooks/useSession.ts | Snapshot dan subscription |
| src/types/ | Tipe UI dan konfigurasi |
| android/app/src/main/java/<package>/autoscroll/ | Module, service, controller, tracker, adapter, gesture, settings, receiver |
| android/app/src/main/res/xml/ | Konfigurasi accessibility service |
| android/app/src/test/ | Uji logika Java |
| __tests__/ | Uji TS yang relevan |
| docs/ | Kelayakan, matriks pengujian, distribusi |

## 13. Referensi teknis resmi

- React Native, Native Modules: https://reactnative.dev/docs/turbo-native-modules-introduction
- Android, AccessibilityService: https://developer.android.com/reference/android/accessibilityservice/AccessibilityService
- Android, AccessibilityNodeInfo: https://developer.android.com/reference/android/view/accessibility/AccessibilityNodeInfo
- Android Help, restricted settings: https://support.google.com/android/answer/12623953

Referensi menjelaskan kemampuan platform, bukan bukti bahwa elemen TikTok tertentu tersedia. Dukungan aktual hanya dapat ditetapkan dari pengujian.
