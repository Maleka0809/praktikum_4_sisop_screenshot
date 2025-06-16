# FUSecure

Yuadi adalah seorang developer brilian yang sedang mengerjakan proyek top-secret. Sayangnya, temannya Irwandi yang terlalu penasaran memiliki kebiasaan buruk menyalin jawaban praktikum sistem operasi Yuadi tanpa izin setiap kali deadline mendekat. Meskipun sudah diperingatkan berkali-kali dan bahkan ditegur oleh asisten dosen karena sering ketahuan plagiat, Irwandi tetap tidak bisa menahan diri untuk mengintip dan menyalin jawaban praktikum Yuadi. Merasa kesal karena prestasinya terus-menerus dicuri dan integritasnya dipertaruhkan, Yuadi yang merupakan ahli keamanan data memutuskan untuk mengambil tindakan sendiri dengan membuat FUSecure, sebuah file system berbasis FUSE yang menerapkan akses read-only dan membatasi visibilitas file berdasarkan permission user.

## Deskripsi Tugas

Setelah frustrasi dengan kebiasaan plagiat Irwandi yang merugikan prestasi akademiknya, Yuadi memutuskan untuk merancang sistem keamanan yang sophisticated. Dia akan membuat sistem file khusus yang memungkinkan mereka berdua berbagi materi umum, namun tetap menjaga kerahasiaan jawaban praktikum masing-masing.

### a. Setup Direktori dan Pembuatan User

Langkah pertama dalam rencana Yuadi adalah mempersiapkan infrastruktur dasar sistem keamanannya.

1. Buat sebuah "source directory" di sistem Anda (contoh: `/home/shared_files`). Ini akan menjadi tempat penyimpanan utama semua file.

2. Di dalam source directory ini, buat 3 subdirektori: `public`, `private_yuadi`, `private_irwandi`. Buat 2 Linux users: `yuadi` dan `irwandi`. Anda dapat memilih password mereka.
   | User | Private Folder |
   | ------- | --------------- |
   | yuadi | private_yuadi |
   | irwandi | private_irwandi |

Yuadi dengan bijak merancang struktur ini: folder `public` untuk berbagi materi kuliah dan referensi yang boleh diakses siapa saja, sementara setiap orang memiliki folder private untuk menyimpan jawaban praktikum mereka masing-masing.

#### kode :
```
sudo mkdir -p /home/shared_files/private_irwandi
sudo mkdir -p /home/shared_files/private_yuadi
sudo mkdir -p /home/shared_files/public

```

#### penjelasan :
kode tersebut merupakan proses membuat source directory. `sudo` digunakan untuk perintah super user, `mkdir` untuk perintah membuat direktori, `-p`untuk parent agar bisa membuat direktori dalam direktori langsung.

#### output :
![alt](https://github.com/Maleka0809/praktikum_4_sisop_screenshot/blob/baa25663ca3709002eaac6f5b8c331882f0a1ec3/Screenshot%202025-06-16%20184743.png)










### b. Akses Mount Point

Selanjutnya, Yuadi ingin memastikan sistem filenya mudah diakses namun tetap terkontrol.

FUSE mount point Anda (contoh: `/mnt/secure_fs`) harus menampilkan konten dari `source directory` secara langsung. Jadi, jika Anda menjalankan `ls /mnt/secure_fs`, Anda akan melihat `public/`, `private_yuadi/`, dan `private_irwandi/`.

### c. Read-Only untuk Semua User

Yuadi sangat kesal dengan kebiasaan Irwandi yang suka mengubah atau bahkan menghapus file jawaban setelah menyalinnya untuk menghilangkan jejak plagiat. Untuk mencegah hal ini, dia memutuskan untuk membuat seluruh sistem menjadi read-only.

1. Jadikan seluruh FUSE mount point **read-only untuk semua user**.
2. Ini berarti tidak ada user (termasuk `root`) yang dapat membuat, memodifikasi, atau menghapus file atau folder apapun di dalam `/mnt/secure_fs`. Command seperti `mkdir`, `rmdir`, `touch`, `rm`, `cp`, `mv` harus gagal semua.

"Sekarang Irwandi tidak bisa lagi menghapus jejak plagiatnya atau mengubah file jawabanku," pikir Yuadi puas.

### d. Akses Public Folder

Meski ingin melindungi jawaban praktikumnya, Yuadi tetap ingin berbagi materi kuliah dan referensi dengan Irwandi dan teman-teman lainnya.

Setiap user (termasuk `yuadi`, `irwandi`, atau lainnya) harus dapat **membaca** konten dari file apapun di dalam folder `public`. Misalnya, `cat /mnt/secure_fs/public/materi_kuliah.txt` harus berfungsi untuk `yuadi` dan `irwandi`.

### e. Akses Private Folder yang Terbatas

Inilah bagian paling penting dari rencana Yuadi - memastikan jawaban praktikum benar-benar terlindungi dari plagiat.

1. File di dalam `private_yuadi` **hanya dapat dibaca oleh user `yuadi`**. Jika `irwandi` mencoba membaca file jawaban praktikum di `private_yuadi`, harus gagal (contoh: permission denied).
2. Demikian pula, file di dalam `private_irwandi` **hanya dapat dibaca oleh user `irwandi`**. Jika `yuadi` mencoba membaca file di `private_irwandi`, harus gagal.

"Akhirnya," senyum Yuadi, "Irwandi tidak bisa lagi menyalin jawaban praktikumku, tapi dia tetap bisa mengakses materi kuliah yang memang kubuat untuk dibagi."

## Contoh Skenario

Setelah sistem selesai, beginilah cara kerja FUSecure dalam kehidupan akademik sehari-hari:

- **User `yuadi` login:**

  - `cat /mnt/secure_fs/public/materi_algoritma.txt` (Berhasil - materi kuliah bisa dibaca)
  - `cat /mnt/secure_fs/private_yuadi/jawaban_praktikum1.c` (Berhasil - jawaban praktikumnya sendiri)
  - `cat /mnt/secure_fs/private_irwandi/tugas_sisop.c` (Gagal dengan "Permission denied" - tidak bisa baca jawaban praktikum Irwandi)
  - `touch /mnt/secure_fs/public/new_file.txt` (Gagal dengan "Permission denied" - sistem read-only)

- **User `irwandi` login:**
  - `cat /mnt/secure_fs/public/materi_algoritma.txt` (Berhasil - materi kuliah bisa dibaca)
  - `cat /mnt/secure_fs/private_irwandi/tugas_sisop.c` (Berhasil - jawaban praktikumnya sendiri)
  - `cat /mnt/secure_fs/private_yuadi/jawaban_praktikum1.c` (Gagal dengan "Permission denied" - tidak bisa baca jawaban praktikum Yuadi)
  - `mkdir /mnt/secure_fs/new_folder` (Gagal dengan "Permission denied" - sistem read-only)

Dengan sistem ini, kedua mahasiswa akhirnya bisa belajar dengan tenang. Yuadi bisa menyimpan jawaban praktikumnya tanpa khawatir diplagiat Irwandi, sementara Irwandi terpaksa harus mengerjakan tugasnya sendiri namun masih bisa mengakses materi kuliah yang dibagikan Yuadi di folder public.

## Catatan

- Anda dapat memilih nama apapun untuk FUSE mount point Anda.
- Ingat untuk menggunakan argument `-o allow_other` saat mounting FUSE file system Anda agar user lain dapat mengaksesnya.
- Fokus pada implementasi operasi FUSE yang berkaitan dengan **membaca** dan **menolak** operasi write/modify. Anda perlu memeriksa **UID** dari user yang mengakses di dalam operasi FUSE Anda untuk menerapkan pembatasan private folder.

---

# FUSecure

Yuadi is a brilliant developer working on a top-secret project. Unfortunately, his overly curious friend Irwandi has a bad habit of copying Yuadi's operating system practicum answers without permission whenever deadlines approach. Despite multiple warnings and even being reprimanded by teaching assistants for frequent plagiarism, Irwandi just can't resist peeking at and copying Yuadi's practicum solutions. Feeling frustrated that his achievements were constantly being stolen and his integrity was at stake, Yuadi, an expert in data security, decided to take matters into his own hands by creating FUSecure, a FUSE-based file system that enforces read-only access and restricts file visibility based on user permissions.

