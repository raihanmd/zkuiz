# LeetCode sebagai referensi workspace Zkuiz

Tanggal pemeriksaan: 2026-10-29. Tahap: reverse engineering perilaku dan riset arsitektur, bukan implementasi platform.

## Arah proyek

- LeetCode menjadi referensi utama alur latihan, editor, Run, Submit, dan panel yang dapat digeser atau diubah ukurannya.
- Identitas visual Zkuiz harus berbeda. Mengikuti pola interaksi tidak berarti menyalin tampilan atau source LeetCode.
- Stack ditetapkan: Next.js, Bun, shadcn/ui.
- Komponen harus modular dan dapat dikomposisikan, mengikuti prinsip shadcn.

Detail worker, transport hasil, database, editor, bahasa circuit pertama, dan verifier belum ditetapkan.

## Cara investigasi dan batas bukti

Target utama adalah workspace tamu pada https://leetcode.com/problems/two-sum/. Metode pemeriksaan meliputi observasi DOM dan runtime browser, pembacaan bundle klien yang dimuat halaman, serta peninjauan dokumentasi resmi.

Sebagian percobaan browser terhalang timeout dan Cloudflare challenge. Observasi interaksi dilakukan pada tab yang berhasil memuat halaman. Analisis bundle dilakukan melalui pembacaan langsung; tidak ada graph statis lengkap. Cakupan dan keterbatasan pengambilan bukti dicatat dalam manifest.

Tidak ada login, kredensial, atau submission sukses. Run dan Submit menampilkan pemberitahuan autentikasi. Karena itu, backend LeetCode, sandbox, queue vendor, sistem operasi worker, dan penegakan resource limit tidak terverifikasi.

Artefak bukti lokal:

- [LC-R01: DOM dan runtime](evidence/leetcode-2026-10-29/runtime-observation.json).
- [LC-R02: resize dan pemeriksaan setelah reload](evidence/leetcode-2026-10-29/layout-reload-check.json).
- [LC-S01: identitas bundle, hash, cakupan pencarian, dan batas analisis](evidence/leetcode-2026-10-29/manifest.json).
- [Diagram interaktif alur dan pembagian panel](leetcode-workspace-map.html), sebagai ilustrasi usulan; bukan implementasi aplikasi atau eksekusi circuit.

Catatan browser yang hanya menerima challenge tidak menjadi bukti perilaku workspace atau eksekusi kode.

## 1. Bentuk playground yang terlihat

Workspace terdiri dari deskripsi soal di kiri, editor di kanan atas, dan testcase/hasil di kanan bawah. Kelompok kiri memiliki tab Description, Editorial, Solutions, Submissions. Kelompok kanan bawah memiliki Testcase dan Test Result. [LC-R01]

Runtime menyediakan Monaco editor; model kode awal memakai bahasa C++, dan editor testcase memakai model plaintext. DOM dan resource yang dimuat mendukung identifikasi Monaco. Ini observasi pada halaman tersebut, bukan klaim semua editor LeetCode memakai konfigurasi sama. [LC-R01]

Pemisah vertikal mengubah lebar kolom. Pemisah horizontal mengubah tinggi editor dan panel bawah. Pengujian event mouse berhasil memindahkan posisi pemisah kolom dari x=716 ke x=837, serta pemisah baris dari y=846 ke y=606 pada viewport 1440×900. Layout dan tab Test Result yang dipilih bertahan setelah navigasi ulang. [LC-R02]

Layout tersimpan di localStorage dengan key `QD_LAYOUT_JSON`. Nilainya berupa pohon row, tabset, dan tab, dengan identitas komponen, selected tab, serta bobot ukuran. Bobot adalah nilai relatif; jangan menganggapnya persentase yang selalu berjumlah 100. Node ID berubah setelah reload, sedangkan ukuran dan identitas komponen tetap dapat dibandingkan. [LC-R01, LC-R02]

Nama class dan kode yang dimuat menunjukkan keluarga FlexLayout. Ini bukti penggunaan pola model layout bertingkat pada snapshot klien, bukan pembenaran untuk mengambil seluruh dependency atau implementasinya. Dokumentasi resmi juga menyebut dynamic layout yang menampilkan deskripsi, editor, dan debug window bersamaan: https://leetcode.com/discuss/post/5736503/feature-release-notes-by-leetcode-9awg/.

**Implikasi untuk Zkuiz:** nested resizable panels cukup untuk kebutuhan awal. Satu grup horizontal memisahkan soal dari area kerja; grup vertikal di area kerja memisahkan editor dari hasil. shadcn menyediakan wrapper ResizablePanelGroup, ResizablePanel, ResizableHandle. Resize tidak sama dengan docking bebas: memindahkan tab ke grup lain memerlukan model layout tambahan jika nanti benar-benar dibutuhkan. Jangan menambahkan docking framework hanya karena referensinya memilikinya.

## 2. Run adalah job, bukan terminal yang harus terus hidup

Dokumentasi resmi membedakan Run Code untuk testcase yang dapat diedit dari Submit untuk penilaian lebih menyeluruh. Input, output, expected output, serta error tampil pada panel hasil. Sumber: https://support.leetcode.com/hc/en-us/articles/360012016874-Start-your-Coding-Practice.

Pada bundle halaman yang diperiksa, controller Run:

1. Membentuk payload berisi `lang`, `question_id`, `typed_code`, dan `data_input` dari daftar testcase.
2. Mengirim POST ke endpoint yang dinamai enqueue dalam kode klien.
3. Membaca `interpret_id`, serta ID expected-result opsional pada cabang tertentu.
4. Memeriksa endpoint hasil secara berkala. Implementasi polling memiliki batas percobaan dan delay bertambah dengan batas maksimum.
5. Memetakan state pending/judging dan hasil ke model tampilan.

Template endpoint yang teramati secara statis:

```text
POST /problems/{slug}/interpret_solution/
GET  /submissions/detail/{id}/check/
```

Path dan controller terdapat pada bundle klien yang dicatat di LC-S01. Ini bukan dokumentasi API publik atau jaminan bahwa endpoint selalu sama. Karena sesi tamu dibatasi autentikasi, request dan respons sukses tersebut tidak direplay.

Controller membedakan compile error, runtime error, timeout, batas memory/output, invalid testcase, hasil benar/salah, dan kegagalan jaringan/server. Stdout dinormalisasi sebagai field tersendiri, terpisah dari jawaban fungsi dan expected answer. Dokumentasi resmi menjelaskan manfaat stdout untuk debugging: https://support.leetcode.com/hc/en-us/articles/360011834194-Am-I-allowed-to-print-something-to-stdout.

**Kesimpulan terbatas:** klien dirancang untuk eksekusi asinkron berbasis job dan pemeriksaan hasil. Nama enqueue mendukung inferensi adanya penjadwalan pekerjaan; bundle tidak mengungkap implementasi queue atau sandbox server.

## 3. Submit menggunakan alur tersendiri

Bundle memisahkan controller Submit dari Run. Controller melakukan pemeriksaan signed-in/verified, mengambil kode dan bahasa saat klik, masuk ke state pending, lalu memanggil layanan Submit. Daftar status sibuk mencakup pending, judging, preparing, compiling, dan running tests. Saat memperoleh submission ID, controller dapat membuka detail dengan judul Judging... dan memperbarui cache daftar submission. [LC-S01]

Registry endpoint klien memuat:

```text
POST /problems/{slug}/submit/
GET  /submissions/detail/{id}/check/
GET  /submissions/detail/{id}/v2/check/
```

Registry tersebut menunjukkan alternatif path, bukan bukti semua varian dipanggil pada satu submission. Implementasi lengkap helper Submit v2 tidak diikuti lintas seluruh dependency; jangan mengasumsikan polling dan mapping internalnya identik dengan Run.

Dokumentasi resmi menyatakan submission tersimpan di history dan dievaluasi dengan pemeriksaan backend, batas waktu/ruang, serta corner cases. Riset ini tidak memverifikasi jumlah testcase atau kebijakan internalnya. Untuk Zkuiz, Submit perlu mengacu pada versi soal, toolchain, dan kebijakan pemeriksaan agar hasil dapat direproduksi.

## 4. Console, workspace, job, dan terminal adalah empat hal berbeda

**Workspace** adalah dokumen yang sedang diedit: kode, testcase, bahasa, tab aktif, serta ukuran panel. Menyimpan workspace tidak mensyaratkan proses atau container yang terus hidup.

**Run job** adalah satu evaluasi snapshot kode dan testcase pilihan pengguna. Job berakhir setelah hasil atau batas eksekusinya tercapai.

**Submit job** adalah evaluasi snapshot kiriman terhadap kebijakan judge. Hasilnya terkait history dan versi spesifikasi soal; input pengujian tersembunyi tidak boleh dikirim ke browser.

**Console/log panel** menampilkan stdout, stderr, diagnostik, dan hasil terstruktur. Ini dapat bekerja tanpa PTY atau shell.

**Terminal interaktif**, jika kelak diperlukan, membutuhkan runtime yang menerima stdin, mempertahankan proses, serta mengirim output dua arah. xterm.js hanya emulator tampilan terminal; proses shell/PTY dan isolasi tetap harus disediakan backend. Sumber: https://github.com/xtermjs/xterm.js#what-xtermjs-is-not.

Pada workspace tamu yang diperiksa, Test Result menampilkan empty state You must run your code first, dan tidak ditemukan widget `.xterm`. Ini mendukung desain console hasil pada halaman tersebut; tidak membuktikan tidak adanya terminal di bagian produk LeetCode lainnya. [LC-R01]

**Rekomendasi MVP:** gunakan editor, testcase, hasil terstruktur, serta log. Setiap klik Run/Submit membuat job; membuka halaman atau mengganti tab tidak perlu membuat sandbox. Terminal interaktif dapat menjadi fitur lanjutan bila pengguna perlu menjalankan perintah manual atau REPL. Jika ditambahkan, session terminal harus terpisah dari job judge, memiliki ownership, idle TTL, batas proses/resource, resize, dan cleanup.

## 5. Pemetaan ke stack yang dipilih

Catatan dokumentasi terpisah: [stack dan kontrak komponen](2026-10-29-stack-and-component-contracts.md).

**Next.js** menangani halaman soal, pemuatan data publik, otorisasi, penerimaan job, pembacaan status, dan history. Editor, tab, panel resize, serta subscriptions hasil berada pada client boundary yang terukur. Compiler/prover/solver tidak perlu dijalankan di lifecycle render atau request web yang harus tetap cepat.

**Bun** menjadi package manager dan kandidat runtime aplikasi/orkestrator. Toolchain circuit native tetap executable terpisah. Bun Worker bukan boundary sandbox; resourceLimits Worker tidak boleh diasumsikan ditegakkan. Pembatasan proses dan sandbox harus diuji pada deployment yang dipilih. Sumber: https://bun.com/guides/ecosystem/nextjs dan https://bun.com/docs/runtime/workers.

**shadcn/ui** menyediakan primitive lokal yang dapat dikomposisikan. Dokumentasi resizable saat ditinjau menggunakan `orientation`; versi primitive dan wrapper harus dipatok bersama saat implementasi. Sumber: https://ui.shadcn.com/docs dan https://ui.shadcn.com/docs/components/base/resizable.

**Judge worker terpisah** mengambil job, menjalankan kompilasi, evaluasi witness, testing adversarial, dan checker yang dipilih. Worker memakai lingkungan sementara dengan batas CPU, RAM, waktu, proses, output, filesystem, serta jaringan. Nama produk queue/cloud/container belum dipilih. Pemisahan worker adalah usulan arsitektur, bukan temuan teknologi backend LeetCode.

**Transport hasil:** polling merupakan opsi paling kecil yang cukup untuk status akhir dan telah terlihat pada controller Run LeetCode. SSE dapat ditambahkan untuk event satu arah dan progres/log; WebSocket dibutuhkan bila ada kebutuhan dua arah seperti terminal. Pilihan final menunggu kebutuhan produk dan deployment, bukan sekadar meniru teknologi yang diduga dipakai LeetCode.

## 6. Prinsip komponen modular

Pisahkan tiga tanggung jawab: primitive UI, komponen domain, dan orkestrasi pekerjaan. Ini bukan memecah setiap elemen menjadi file atau membuat komponen universal dengan puluhan flag.

Struktur awal yang diusulkan:

```text
apps/web/
  app/                         halaman dan route handlers
  components/ui/               primitive shadcn milik proyek
  features/workspace/
    components/                problem, editor, testcase, hasil, log
    hooks/                     state draft/layout dan lifecycle job
    model/                     tipe dan reducer domain
    api/                       adapter request/status
apps/judge/                    worker dan adapter toolchain
packages/judge-contracts/      kontrak request, event, hasil
```

Ini usulan pembagian tanggung jawab, bukan folder yang sudah diimplementasikan. Hindari shared UI package sebelum ada pemakai kedua yang nyata. `packages/judge-contracts` hanya untuk schema lintas web/worker; jangan memasukkan secret atau hidden tests ke paket yang dapat diimpor klien.

Komponen domain utama:

- `WorkspaceLayout` menerima slot problem/editor/results; mengurus geometri dan focus, tidak mengetahui endpoint judge.
- `CircuitEditor` menerima files/value dan callback perubahan; adapter Monaco tidak mengirim job sendiri.
- `ExecutionToolbar` menerima status serta callback Run/Submit; tidak membuat proses atau memuat hidden tests.
- `TestcaseEditor` mengedit input Run yang dikontrol caller.
- `ExecutionResults` menerima model hasil; menampilkan verdict, witness/counterexample yang boleh dibuka, dan diagnostik.
- `ExecutionLogs` menampilkan output terbatas sebagai teks; tidak mengubahnya menjadi HTML mentah.
- `SubmissionHistory` menampilkan record submission; tidak berbagi mutable draft dengan snapshot submission lama.

Hooks atau controller seperti `useWorkspaceDraft`, `useWorkspaceLayout`, dan `useExecutionJob` menyatukan state yang perlu dibagikan. Detail penamaan masih usulan. Satu controller tingkat fitur menghubungkan intent toolbar dengan adapter API; komponen presentasi tetap dapat dipakai dan diuji sendiri.

## 7. Kontrak eksekusi yang harus jelas sebelum coding

Usulan kontrak minimum untuk Zkuiz:

- Request menyebut mode Run/Submit, versi problem, versi toolchain, snapshot files, dan testcase publik untuk Run.
- Server memvalidasi ownership dan policy, menyimpan snapshot, lalu mengembalikan jobId setelah pekerjaan diterima. Klien tidak menentukan resource limits, hidden tests, atau verdict resmi.
- Job memiliki lifecycle queued/running/completed/failed/cancelled. Pisahkan phase kompilasi/testing/analisis dari verdict jawaban dan error infrastruktur.
- Kaitkan event dan hasil dengan jobId serta identitas snapshot. Pengeditan draft setelah Run tidak mengubah job yang sedang berjalan.
- Jika pengguna mengedit kode sebelum hasil kembali, tandai hasil berasal dari versi sebelumnya. Respons lama tidak boleh menimpa hasil job baru.
- Atur deduplikasi klik, batas concurrent job, retry yang aman, pembatalan, dan expiry. Putusnya koneksi browser tidak otomatis membatalkan submission yang sudah diterima.
- Log dibatasi ukuran dan aksesnya. Kirim hanya diagnostik yang boleh dilihat; hidden tests dan rahasia worker tidak ikut event klien.
- Label formally verified hanya dipakai sesuai cakupan pemeriksaan. UNKNOWN/timeout solver tetap inconclusive, mengikuti [riset judge circuit sebelumnya](2026-10-29-zk-circuit-judge.md).

## 8. Hasil checklist dan eksperimen berikutnya

**Terjawab:** susunan panel; Monaco pada halaman target; dua arah resize; persistensi ukuran/tab; testcase dan result panel terpisah; gate autentikasi; payload dan polling Run secara statis; controller Submit dan integrasi history secara statis; pemisahan console dari kebutuhan terminal.

**Sebagian terjawab:** alur Submit end-to-end, autosave source lintas sesi, docking tab bebas, perilaku mobile, dan seluruh variasi error. Ada affordance atau kode terkait, tetapi tidak semuanya dijalankan.

**Belum terjawab:** eksekusi Run/Submit terautentikasi, infrastruktur sandbox/worker LeetCode, enforcement resource limit, antrean dan penyimpanan internal, serta performa judge ZK.

Langkah berikut yang direkomendasikan adalah vertical slice kecil setelah riset ini: satu soal, draft editor, dua splitter, Run satu testcase melalui worker terisolasi, dan hasil yang terikat snapshot. Submit menambahkan policy judge dan history. Terminal interaktif serta docking bebas menunggu kebutuhan yang teruji. UI final dirancang khusus Zkuiz dengan primitive modular, bukan salinan LeetCode.
