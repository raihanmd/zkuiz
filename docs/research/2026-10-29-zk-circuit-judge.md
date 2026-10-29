# Riset judge untuk Zkuiz

Tanggal: 2026-10-29. Status: rekomendasi riset, belum keputusan implementasi. Tool ditinjau melalui dokumentasi dan paper; belum diinstal atau dibenchmark.

## Tujuan

Platform seperti LeetCode untuk belajar zero knowledge, terutama menulis arithmetic circuit. Pengguna membaca soal, mengedit circuit, mencoba input di playground, lalu mengirim jawaban untuk dinilai. Tantangan utamanya adalah membedakan circuit yang menghitung jawaban benar melalui witness generator dengan circuit yang benar-benar membatasi semua witness yang dapat diterima.

## Kesimpulan

Mulai dengan Circom dan soal circuit kecil. Gunakan judge berlapis: kompilasi, pemeriksaan statis, tes terhadap implementasi referensi, pencarian witness yang melanggar spesifikasi, serta verifikasi formal untuk soal yang didukung. Symbolic execution berguna sebagai teknik pendukung, tetapi analisis harus mencakup constraint pada finite field. Eksekusi simbolik witness generator saja tidak memadai.

Jangan menjadikan keberhasilan pembuatan atau verifikasi proof sebagai definisi jawaban benar. Proof membuktikan pemenuhan circuit yang dikompilasi, sedangkan judge perlu mengecek kesesuaian circuit dengan spesifikasi soal. Sifat zero knowledge dari protokol juga berbeda dari ketepatan relasi yang ditulis peserta.

## Bug yang harus ditangkap judge

Contoh fragmen Circom untuk soal perkalian:

```circom
signal input a;
signal input b;
signal output c;
c <-- a * b;
```

Witness generator menghitung hasil yang benar, tetapi `<--` tidak membuat constraint perkalian. Witness dengan output lain dapat memenuhi constraint yang tersisa. Untuk kasus sederhana ini, `c <== a * b;` menggabungkan assignment dan constraint. Ini bukan berarti semua penggunaan `<--` salah: hint sah bila ada constraint terpisah yang membatasi hasilnya.

Dokumentasi resmi menjelaskan perbedaan `<--`, `<==`, dan `===`: https://github.com/iden3/circom/blob/master/mkdocs/docs/circom-language/constraint-generation.md dan https://github.com/iden3/circom/blob/master/mkdocs/docs/circom-language/signals.md (ditinjau 2026-10-29 melalui Context7).

Kasus lain yang perlu ada dalam benchmark:

- Selector menerima nilai selain 0 dan 1 karena constraint boolean hilang.
- Range check atau comparison salah pada batas domain.
- Circuit selalu menghasilkan nol: deterministik, tetapi salah menurut soal.
- Circuit menolak sebagian input sah: terlalu membatasi atau overconstrained.
- Circuit tidak memiliki witness sama sekali: pemeriksaan implikasi bisa lolos secara vakum.
- Public input tidak terikat ke komputasi yang diminta.
- Arithmetic wraparound: operasi modulo prime field disalahartikan sebagai integer tanpa batas.
- Witness generator gagal pada input sah meskipun relasi constraint mengizinkan witness.

## Kontrak matematis per soal

Definisikan `C(x,y,w)` sebagai constraint peserta, `S(x,y)` sebagai spesifikasi hasil, dan `D(x)` sebagai domain input sah. Semua variabel yang relevan harus jelas: input publik, input privat, output, dan witness internal. Field, encoding integer, lebar bit, serta perlakuan input di luar domain harus ditentukan penulis soal.

Bedakan precondition yang dijamin harness dari properti yang wajib dibuktikan circuit. Untuk soal yang meminta circuit menegakkan `0 <= x < 256`, jangan mengasumsikan batas itu sebagai precondition solver: asumsi tersebut akan menyembunyikan range check yang hilang. Cari juga witness yang diterima dengan nilai di luar rentang. Sebaliknya, soal operasi pada input 8-bit yang dijamin oleh komponen pemanggil boleh mempunyai precondition tersebut jika kontraknya eksplisit.

Untuk soal komputasi dengan output deterministik, cari pelanggaran soundness:

```text
exists x,y,w: D(x) AND C(x,y,w) AND NOT S(x,y)
```

SAT berarti ada counterexample. UNSAT berarti tidak ada pelanggaran dalam model dan domain tersebut, jika penerjemahan constraint dan spesifikasi benar. Timeout atau UNKNOWN tidak memberikan bukti lulus maupun bukti salah. Bila witness invalid ditemukan solver, evaluasi ulang witness pada R1CS asli sebelum melaporkan kegagalan.

Periksa completeness secara terpisah:

```text
forall x: D(x) => exists y,w: C(x,y,w) AND S(x,y)
```

Completeness memiliki pergantian kuantor dan umumnya lebih sulit daripada query counterexample soundness. MVP bisa memakai exhaustive testing untuk domain kecil dan pengujian berbasis properti untuk domain besar, dengan label jaminan yang jujur. Pengujian witness generator adalah pemeriksaan tambahan: adanya witness secara matematis belum menjamin generator peserta berhasil menghitungnya.

Untuk soal relasional, misalnya pembuktian pengetahuan preimage hash atau Merkle membership, gunakan relasi `R(public,private)`. Jangan mewajibkan seluruh witness atau private input unik: banyak witness dapat sah. Tentukan apakah input privat tetap diberikan harness atau termasuk witness yang boleh dipilih peserta. Spesifikasi harus membahas pengikatan public input dan public output.

Membandingkan jumlah constraint atau bentuk R1CS dengan solusi referensi bukan pemeriksaan kesetaraan semantik. Implementasi berbeda bisa benar. Menolak semua witness internal yang tidak unik juga bisa menolak implementasi sah, contohnya auxiliary inverse yang bebas pada cabang tertentu.

## Peran symbolic execution

Circuit umumnya dielaborasi menjadi constraint statis; banyak loop dan percabangan struktural telah ditentukan saat kompilasi. Karena itu, pemodelan langsung constraint sering lebih sesuai daripada menjelajahi semua jalur witness generator.

Teknik simbolik dapat membantu membentuk query solver, mencari input batas, dan menghasilkan witness curang. Solver harus mempertahankan semantik finite field. Encoding integer atau bit-vector tanpa reduksi modulo dan batas yang benar dapat menghasilkan kesimpulan salah.

cvc5 menyediakan theory finite field berorde prime. Ini menjadikannya kandidat backend untuk proof of concept, bukan bukti bahwa seluruh circuit dapat diselesaikan cepat. Range comparison dan konversi integer memerlukan encoding eksplisit. Dukungan build dan dependensi solver perlu divalidasi saat benchmark. Dokumentasi: https://cvc5.github.io/docs/latest/theories/finite_field.html (ditinjau 2026-10-29).

## Tool yang relevan

### Circom dan testing harness

Circom cocok sebagai kandidat awal karena constraint dan witness generation terlihat jelas, dan tersedia beberapa analyzer. Kompilasi menghasilkan artefak untuk evaluasi witness dan constraint. circom_tester atau Circomkit dapat dievaluasi sebagai harness; belum dipilih atau diuji dalam riset ini. Gunakan compiler, field, flag optimisasi, dan library yang dipatok versinya. Pertimbangkan artefak dengan simbol untuk analisis serta artefak final untuk pengukuran biaya; pastikan keduanya berasal dari sumber yang sama.

### Circomspect

Static analyzer dan linter dari Trail of Bits. Cocok untuk feedback cepat mengenai pola berisiko. Warning bukan bukti circuit salah; ketiadaan warning bukan bukti benar. Repository mendokumentasikan dukungan beberapa field/curve dan output SARIF. Sumber: https://github.com/trailofbits/circomspect dan https://github.com/trailofbits/circomspect/blob/main/doc/analysis_passes.md (ditinjau 2026-10-29).

### Picus

Tool untuk memverifikasi uniqueness dan mendeteksi underconstrained signals. Repository mendokumentasikan target Circom, R1CS, dan gnark. Berguna sebagai pemeriksaan tambahan untuk soal fungsi deterministik, tetapi uniqueness tidak membuktikan fungsi yang dihitung sesuai soal. Perlu menetapkan signal mana yang memang wajib unik. Sumber: https://github.com/Veridise/Picus (ditinjau 2026-10-29).

### CIVER

Kandidat terdekat untuk verifikasi berbasis spesifikasi: mendukung weak safety, tag, dan precondition/postcondition pada Circom. Repository menyebut backend Z3 dan verifikasi modular mengikuti struktur template. Dukungan constraint format lain disebut sebagai pengembangan mendatang pada README yang ditinjau; jangan menganggap R1CS/PLONK/ACIR sudah tersedia. Perlu menguji ekspresivitas spesifikasi, perilaku counterexample, asumsi domain, dan completeness secara terpisah. Sumber: https://github.com/costa-group/circom_civer (ditinjau 2026-10-29).

### Ecne

Menganalisis apakah constraint R1CS menentukan output dari input. Menjadi pembanding untuk pemeriksaan uniqueness. README membedakan kemampuan ini dari tujuan masa depan untuk mencocokkan fungsi dengan spesifikasi formal. Jangan menganggapnya sebagai judge semantik lengkap. Sumber: https://github.com/franklynwang/EcneProject (ditinjau 2026-10-29).

### zkFuzz

Paper memperkenalkan Trace-Constraint Consistency Test dan mutasi komputasi untuk mengekspos ketidaksesuaian komputasi dengan constraint. Relevan untuk memperkuat testing adversarial; bukan bukti universal kesesuaian terhadap spesifikasi soal yang independen. Sumber: https://arxiv.org/abs/2504.11961 dan https://github.com/Koukyosyumei/zkFuzz (ditinjau 2026-10-29).

### zkREPL dan bahan pembanding produk

zkREPL adalah playground circuit daring dari 0xPARC. Cocok sebagai referensi interaksi editor, kompilasi, dan witness. ZK Hack menyediakan puzzle pembelajaran dan keamanan ZK. Keduanya membantu riset pengalaman belajar; jangan mengasumsikan menyediakan judge semantik seperti yang dibutuhkan Zkuiz. Sumber: https://github.com/0xPARC/zkrepl dan https://zkhack.dev/events/ (ditinjau 2026-10-29).

README circom-ecdsa secara eksplisit menyatakan unit test witness generation tidak memastikan circuit properly constrained. Ini contoh langsung mengapa test biasa perlu dilengkapi analisis constraint: https://github.com/0xPARC/circom-ecdsa/blob/master/README.md (ditinjau 2026-10-29).

## Rancangan judge yang direkomendasikan

1. Penulis soal menetapkan ABI, field, domain, spesifikasi, contoh, reference implementation, dan tingkat jaminan yang dapat diberikan.
2. Playground menjalankan kompilasi dan tes contoh untuk feedback cepat. Hasil dari browser bukan dasar penerimaan resmi.
3. Worker server yang terisolasi mengompilasi ulang submission dengan toolchain yang dipatok. Batasi CPU, RAM, waktu, proses, ukuran artefak, include path, serta akses jaringan/filesystem.
4. Jalankan static checks, hidden tests, input batas, dan pengujian berbasis properti terhadap referensi.
5. Cari witness adversarial pada constraint, tanpa bergantung hanya pada witness generator asli.
6. Untuk soal yang didukung, verifikasi implikasi constraint terhadap spesifikasi. Cegah kelulusan vakum melalui pemeriksaan input sah dan completeness sesuai cakupan soal.
7. Tampilkan hasil terpisah: compilation error, test failure, counterexample terkonfirmasi, verified dalam cakupan tertentu, atau inconclusive. UNKNOWN dan timeout tidak boleh diberi label formally verified.
8. Setelah correctness, ukur jumlah constraint atau biaya lain dengan compiler dan flag yang tetap. Pisahkan kebenaran dari optimisasi.

Proof generation sebaiknya menjadi demonstrasi atau kategori soal tersendiri. Untuk soal Groth16 dengan circuit submission yang berubah, setup khusus circuit dapat menambah biaya. Jangan memasukkan proving ke setiap siklus edit tanpa kebutuhan pembelajaran yang jelas.

## Eksperimen berikutnya

Bangun benchmark judge sebelum membangun platform penuh. Mulai dari 5–10 soal: perkalian, boolean selector, IsZero, Num2Bits, range check, comparison dengan domain eksplisit, dan penjumlahan integer dengan overflow yang ditentukan. Tunda hash atau signature besar untuk evaluasi berikutnya.

Untuk setiap soal, siapkan beberapa solusi benar dengan bentuk berbeda dan mutan salah: constraint hilang, `<==` diganti `<--`, boolean check hilang, batas off-by-one, output konstan, public input tidak terikat, constraint kontradiktif, serta asumsi integer yang tidak sah pada field.

Ukur deteksi mutan, penolakan keliru terhadap solusi benar, persentase UNKNOWN, waktu median/p95, peak RAM, dan kualitas counterexample. Tambahkan satu contoh auxiliary witness nonunik yang sah. Jangan memilih tool utama hanya berdasarkan klaim README.

Kriteria go/no-go yang diusulkan: semua mutan dasar pada benchmark kecil terdeteksi oleh pipeline; solusi benar yang beragam tetap lolos; counterexample dapat direproduksi; timeout dilaporkan jujur; latency dan RAM sesuai budget produk yang akan ditetapkan. Hasil ini hanya menjamin benchmark, bukan semua circuit.

## Pertanyaan terbuka

- Target awal: belajar menulis circuit, belajar audit circuit, atau keduanya sebagai track terpisah?
- Apakah Circom diterima sebagai bahasa pertama, dengan Noir/gnark/Halo2 menyusul bila ada kebutuhan?
- Berapa budget latency dan resource per submission?
- Apakah penerimaan berbasis tes cukup untuk soal tertentu, atau semua soal harus mempunyai bukti formal?
- Apakah CIVER atau backend cvc5 memberikan coverage dan counterexample yang cukup untuk benchmark ini?
- Apa kebijakan lisensi dan distribusi untuk tool yang diintegrasikan? Beberapa repository memakai GPL; tinjau sesuai model deployment sebelum menyalin atau mendistribusikan kode.

Tidak ada implementasi platform, instalasi tool, atau angka performa yang dihasilkan dalam riset ini.
