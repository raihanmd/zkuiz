# Zkuiz: batas stack dan kontrak komponen

Tanggal pemeriksaan: **29 Oktober 2026**. Catatan ini membahas dokumentasi resmi dan sumber primer. Rekomendasi belum diuji melalui implementasi atau benchmark; versi paket proyek belum dipatok.

## Keputusan terkonfirmasi dan usulan

**Arah proyek:** Zkuiz menggunakan **Next.js, Bun, dan shadcn/ui** untuk platform latihan arithmetic circuit ZK. Komponen memiliki tanggung jawab yang jelas dan dapat dikomposisikan. Implementasi aplikasi belum dimulai pada saat penelitian ini dilakukan.

**Usulan arsitektur:** Next.js menangani UI dan API aplikasi; pekerjaan judge berjalan asinkron di worker terpisah dengan lingkungan eksekusi terisolasi. Monaco adalah kandidat editor; xterm.js hanya diperlukan bila ada kebutuhan terminal interaktif. Bahasa circuit, compiler, mekanisme sandbox, penyimpanan, antrean, dan penyedia deployment belum dipilih.

## 1. Next.js: UI dan API aplikasi

Dokumentasi menyebut Route Handlers sebagai endpoint HTTP publik untuk pola backend-for-frontend, bukan pengganti semua kebutuhan backend. Pada deployment berbasis lambda, handler dapat dihentikan oleh timeout; state antarpemanggilan dan filesystem persisten tidak boleh diasumsikan. Ini batas deployment tersebut, bukan larangan universal menjalankan proses lama pada server Next.js yang dikelola sendiri. [N1]

**Usulan batas tanggung jawab:**

- **Server Components:** mengambil metadata soal dan data awal yang boleh dilihat pengguna. Pages/layouts menggunakan Server Components secara default. Jangan mengirim hidden tests atau rahasia judge sebagai props.
- **Client Components:** editor, resize panel, tab, input kasus uji, dan status interaktif. Props yang melintasi batas server/client harus dapat diserialisasi React. Callback UI dibuat dalam subtree client. [N2]
- **Route Handlers:** autentikasi, otorisasi, validasi ukuran/format, pembatasan permintaan, menerima snapshot jawaban, lalu mengembalikan identitas job. Endpoint status hanya menampilkan job yang berhak diakses peminta.
- **Judge worker:** mengambil job, menjalankan toolchain dan pemeriksaan, menyimpan hasil, lalu membersihkan lingkungan eksekusi. Worker dapat hidup lebih lama daripada request web.

Pemisahan ini diusulkan karena eksekusi kiriman pengguna membutuhkan pengendalian sumber daya dan isolasi. Bukan karena Next.js sendiri menyediakan sandbox. Untuk MVP, polling status cukup sebagai usulan awal; streaming/WebSocket memerlukan evaluasi deployment, bukan prasyarat.

## 2. Bun: dukungan dan batas nyata

Panduan resmi Bun mendukung Next.js. Untuk menjalankan CLI Next.js dengan runtime Bun secara eksplisit, panduan memakai `bun --bun next dev`, `bun --bun next build`, dan `bun --bun next start`. Memakai Bun sebagai package manager saja tidak membuktikan proses Next.js memakai runtime Bun. Konfigurasi Next.js tidak memiliki nilai runtime `"bun"`; jangan mengarang `export const runtime = "bun"`. [B1][N3]

Kompatibilitas Node.js masih memiliki perbedaan API. Dokumentasi saat diperiksa menyatakan `node:worker_threads.Worker` mengabaikan `resourceLimits` dan `trackUnmanagedFds`. API Web Worker Bun masih eksperimental, terutama terminasi. **Worker thread tidak boleh dijadikan batas isolasi kiriman pengguna.** [B2][B3]

`Bun.spawn()` mendukung `timeout` dan `killSignal`; sinyal timeout default adalah `SIGTERM`. Dokumentasi juga menyediakan opsi `cgroup` pada Linux untuk memasukkan subprocess ke cgroup yang sudah dikonfigurasi sebelum eksekusi; limit memory, PID, dan CPU berlaku pada turunannya. Ini kemampuan pembatasan proses, bukan sandbox lengkap untuk filesystem, jaringan, dan rahasia. [B4]

**Usulan:** Bun boleh menjadi runtime orkestrator judge. Compiler/prover tetap executable sesuai toolchain pilihan; bukan harus JavaScript/Bun. Lingkungan job perlu batas CPU, memory, waktu, jumlah proses, ukuran output, filesystem sementara, kebijakan jaringan, dan penghentian seluruh proses job. Mekanismenya belum dipilih. Verifikasi versi Bun, Next.js, native dependency, dan toolchain melalui smoke test nyata setelah prototipe diizinkan; dokumentasi dukungan belum membuktikan kombinasi proyek ini berjalan.

## 3. shadcn: komposisi dan API resizable

Prinsip resmi shadcn mencakup **Open Code** dan **Composition**: kode komponen dimiliki proyek dan antarmuka dibuat konsisten serta dapat dikomposisikan. Implikasinya untuk Zkuiz adalah primitive kecil dengan props/children yang jelas, bukan satu komponen workspace berisi fetching, editor, eksekusi, dan tampilan hasil sekaligus. [S1]

Dokumentasi resizable saat diperiksa menggunakan wrapper lokal `ResizablePanelGroup`, `ResizablePanel`, `ResizableHandle`. API kini memakai `orientation="horizontal"` atau `"vertical"` dan persentase eksplisit seperti `defaultSize="50%"`. Pada primitive `react-resizable-panels` v4, `PanelGroup` menjadi `Group`, `PanelResizeHandle` menjadi `Separator`, dan `direction` menjadi `orientation`; nama wrapper shadcn tetap sama. Jangan mencampur contoh v3 dengan wrapper v4. [S2]

**Kontrak komponen yang diusulkan, belum kode implementasi:**

| Komponen | Masukan/keluaran minimal | Tanggung jawab |
| --- | --- | --- |
| `ProblemStatement` | data soal publik | Menampilkan deskripsi; tidak mengambil hidden tests. |
| `WorkspaceLayout` | slot `problem`, `editor`, `results` | Menyusun panel dan resize; tidak mengetahui API judge. |
| `CircuitEditor` | `files`, `activePath`, `languageId`, `readOnly`; callback perubahan file/path | Adapter editor; tidak mengirim job langsung. |
| `RunControls` | status, `canRun`, `canSubmit`; `onRun`, `onSubmit` | Menyampaikan intent; mencegah aksi duplikat saat belum siap. |
| `TestInput` | input publik terkontrol; `onChange` | Mengedit input Run, bukan input tersembunyi Submit. |
| `JudgeResults` | status, hasil terstruktur, diagnostik | Menampilkan pending, hasil, error infrastruktur, dan timeout secara berbeda. |

Orkestrasi request/status berada pada controller atau hook workspace. Primitive `components/ui/*` tidak mengimpor domain soal/judge. State draft dimiliki controller editor; komponen saudara menerima data dan callback, sehingga Monaco dapat diganti tanpa mengubah kontrak judge. Ini kontrak domain yang diusulkan, bukan API bawaan shadcn.

## 4. Monaco dan xterm.js memiliki fungsi berbeda

**Monaco:** editor kode browser dengan model dokumen dan layanan bahasa. Mendukung integrasi web worker serta provider bahasa, termasuk completion dan tokenisasi. Dukungan bahasa circuit perlu dievaluasi tersendiri; syntax highlighting bukan compiler, validator constraint, atau bukti correctness. Ekstensi VS Code tidak otomatis kompatibel. README resmi juga menyatakan mobile browser belum didukung. [M1][M2]

**Usulan integrasi:** muat adapter editor di sisi client, konfigurasi worker sesuai bundler Next.js yang benar-benar dipakai, pertahankan model per file, dan dispose model/listener saat selesai. Panduan integrasi Vite bukan konfigurasi Next.js siap pakai. Aksesibilitas dan kebutuhan mobile harus diuji sebelum menetapkan Monaco.

**xterm.js:** emulator terminal frontend; shell/proses tetap disediakan backend dan, bila perlu, PTY. Ia tidak menggantikan editor atau menjalankan compiler dengan sendirinya. Terminal yang terhubung proses memperluas kemampuan pengguna; dokumentasi keamanan meminta kehati-hatian terhadap data terminal dan koneksi backend. [X1][X2]

**Usulan MVP:** gunakan hasil terstruktur dan log teks terbatas. Tambahkan xterm hanya jika interaksi terminal/ANSI dibutuhkan; tampilkan log teks sebagai teks, jangan HTML mentah.

## 5. Kontrak judge minimum yang diusulkan

- `POST /api/judge-jobs`: menerima `problemId`, `problemVersion`, `mode: "run" | "submit"`, snapshot `files` berisi path relatif dan konten, serta input publik untuk Run bila diizinkan. Server memvalidasi path, kuota ukuran, kepemilikan, versi soal, dan mode. Pemilihan toolchain dan limit ditetapkan server; klien tidak mengirim command shell atau lokasi hidden tests.
- Setelah job diterima untuk diproses, respons `202` berisi `jobId` dan `status`. Respons penerimaan belum berarti jawaban benar. Snapshot kiriman tidak berubah ketika draft editor berubah.
- `GET /api/judge-jobs/{jobId}`: mengembalikan `status: "queued" | "running" | "completed" | "failed"`; hasil domain hanya jika tersedia. `failed` menandai kegagalan layanan judge. Compile error, runtime error, timeout kiriman, dan hasil pemeriksaan adalah outcome evaluasi, bukan otomatis kegagalan layanan.
- Hasil memisahkan `outcome`, diagnostik dengan file/range opsional, metrik dengan unit, dan log yang dibatasi serta ditandai jika terpotong. Hidden tests, witness privat, dan keluaran tool sensitif tidak otomatis dipublikasikan. Outcome `accepted` hanya berarti lolos kebijakan pemeriksaan versi soal tersebut; jangan melabelinya bukti soundness/completeness tanpa metode yang mendukung klaim itu.

Transport antrean dan penyimpanan sengaja belum ditentukan. Kontrak job perlu mencatat versi problem, toolchain, dan kebijakan pemeriksaan untuk reproduksi; detail persistence/retry menunggu kebutuhan durasi, volume, serta batas eksekusi yang nyata.

## Sumber primer

- **[N1]** Next.js, Backend for Frontend: https://nextjs.org/docs/app/guides/backend-for-frontend
- **[N2]** Next.js, Server and Client Components: https://nextjs.org/docs/app/getting-started/server-and-client-components
- **[N3]** Next.js, Route Segment Config: https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config
- **[B1]** Bun, Next.js guide: https://bun.com/guides/ecosystem/nextjs
- **[B2]** Bun, Node.js compatibility: https://bun.com/docs/runtime/nodejs-compat
- **[B3]** Bun, Workers: https://bun.com/docs/runtime/workers
- **[B4]** Bun, Spawn: https://bun.com/docs/runtime/child-process
- **[S1]** shadcn/ui, Introduction: https://ui.shadcn.com/docs
- **[S2]** shadcn/ui, Resizable dan catatan migrasi v4: https://ui.shadcn.com/docs/components/base/resizable
- **[M1]** Monaco, README/FAQ resmi: https://github.com/microsoft/monaco-editor/blob/main/README.md
- **[M2]** Monaco, integrasi ESM dan contoh provider resmi: https://github.com/microsoft/monaco-editor/blob/main/docs/integrate-esm.md dan https://github.com/microsoft/monaco-editor/blob/main/website/src/website/data/playground-samples/extending-language-services/completion-provider-example/sample.js
- **[X1]** xterm.js, README resmi: https://github.com/xtermjs/xterm.js#what-xtermjs-is-not
- **[X2]** xterm.js, Security: https://xtermjs.org/docs/guides/security/

Context7 digunakan untuk menelusuri dokumentasi Next.js, Bun, shadcn/ui, Monaco, dan xterm.js. URL primer di atas menjadi rujukan yang dapat diperiksa ulang; belum ada klaim kompatibilitas hasil pengujian lokal.
