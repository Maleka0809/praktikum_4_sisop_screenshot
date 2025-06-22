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
setelah itu mengatur password:
#### kode :
```
sudo adduser irwandi
sudo adduser yuadi

sudo chown yuadi:yuadi /home/shared_files/private_yuadi
sudo chown irwandi:irwandi /home/shared_files/private_irwandi
sudo chmod 700 /home/shared_files/private_yuadi
sudo chmod 700 /home/shared_files/private_irwandi
sudo chmod 755 /home/shared_files/public

sudo passwd yuadi
sudo passwd irwandi
```
saya buat password nya Yuadi_private dan Irwandi_private

#### penjelasan :
kode tersebut merupakan proses membuat source directory. `sudo` digunakan untuk perintah super user, `mkdir` untuk perintah membuat direktori, `-p`untuk parent agar bisa membuat direktori dalam direktori langsung.

#### output :
![alt](https://github.com/Maleka0809/praktikum_4_sisop_screenshot/blob/baa25663ca3709002eaac6f5b8c331882f0a1ec3/Screenshot%202025-06-16%20184743.png)


![alt](https://github.com/Maleka0809/praktikum_4_sisop_screenshot/blob/2be9bedf8501128b5b391d3f446640039b62963a/Screenshot%202025-06-16%20191652.png)


![alt](https://github.com/Maleka0809/praktikum_4_sisop_screenshot/blob/2be9bedf8501128b5b391d3f446640039b62963a/Screenshot%202025-06-18%20063428.png)


![alt](https://github.com/Maleka0809/praktikum_4_sisop_screenshot/blob/2be9bedf8501128b5b391d3f446640039b62963a/Screenshot%202025-06-18%20071134.png)









### b. Akses Mount Point

Selanjutnya, Yuadi ingin memastikan sistem filenya mudah diakses namun tetap terkontrol.

FUSE mount point Anda (contoh: `/mnt/secure_fs`) harus menampilkan konten dari `source directory` secara langsung. Jadi, jika Anda menjalankan `ls /mnt/secure_fs`, Anda akan melihat `public/`, `private_yuadi/`, dan `private_irwandi/`.

#### kode :
```
mkdir -p /mnt/secure_fs
nano filefuse.c
gcc filefuse.c -o filefuse `pkg-config fuse --cflags --libs`
sudo ./filefuse /mnt/secure_fs
```

```
#define FUSE_USE_VERSION 28
#include <fuse.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <dirent.h>
#include <errno.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <pwd.h>

static const char *source_path = "/home/shared_files";

// Membuat path absolut dari relative path
void fullpath(char fpath[1024], const char *path) {
    snprintf(fpath, 1024, "%s%s", source_path, path);
}

// Ambil nama user aktif
const char* get_username() {
    uid_t uid = fuse_get_context()->uid;
    struct passwd *pw = getpwuid(uid);
    return pw ? pw->pw_name : "";
}

// Cek path private
int is_private_yuadi(const char *path) {
    return strncmp(path, "/private_yuadi", 14) == 0;
}
int is_private_irwandi(const char *path) {
    return strncmp(path, "/private_irwandi", 16) == 0;
}

// Cek izin akses berdasarkan user
int is_authorized(const char *path) {
    const char *username = get_username();
    if (is_private_yuadi(path) && strcmp(username, "yuadi") != 0)
        return 0;
    if (is_private_irwandi(path) && strcmp(username, "irwandi") != 0)
        return 0;
    return 1;
}

static int xmp_getattr(const char *path, struct stat *stbuf) {
    // Perbolehkan ls tanpa error walau akses ditolak
    if ((is_private_yuadi(path) || is_private_irwandi(path)) && !is_authorized(path)) {
        if (strcmp(path, "/private_yuadi") == 0 || strcmp(path, "/private_irwandi") == 0) {
            stbuf->st_mode = S_IFDIR | 0555;
            stbuf->st_nlink = 2;
            return 0;
        }
        return -EACCES;
    }

    char fpath[1024];
    fullpath(fpath, path);
    int res = lstat(fpath, stbuf);
    return (res == -1) ? -errno : 0;
}

static int xmp_readdir(const char *path, void *buf, fuse_fill_dir_t filler,
                       off_t offset, struct fuse_file_info *fi) {
    if (!is_authorized(path)) return -EACCES;

    char fpath[1024];
    fullpath(fpath, path);
    DIR *dp = opendir(fpath);
    if (dp == NULL) return -errno;

    struct dirent *de;
    while ((de = readdir(dp)) != NULL) {
        struct stat st;
        memset(&st, 0, sizeof(st));
        st.st_ino = de->d_ino;
        st.st_mode = de->d_type << 12;
        if (filler(buf, de->d_name, &st, 0)) break;
    }
    closedir(dp);
    return 0;
}

static int xmp_read(const char *path, char *buf, size_t size,
                    off_t offset, struct fuse_file_info *fi) {
    if (!is_authorized(path)) return -EACCES;

    char fpath[1024];
    fullpath(fpath, path);

    int fd = open(fpath, O_RDONLY);
    if (fd == -1) return -errno;

    int res = pread(fd, buf, size, offset);
    close(fd);
    return (res == -1) ? -errno : res;
}

static int xmp_write(const char *path, const char *buf, size_t size,
                     off_t offset, struct fuse_file_info *fi) {
    char fpath[1024];
    fullpath(fpath, path);

    int fd = open(fpath, O_WRONLY);
    if (fd == -1) return -errno;

    int res = pwrite(fd, buf, size, offset);
    close(fd);
    return (res == -1) ? -errno : res;
}

static int xmp_mkdir(const char *path, mode_t mode) {
    char fpath[1024];
    fullpath(fpath, path);
    return mkdir(fpath, mode) == -1 ? -errno : 0;
}

static int xmp_unlink(const char *path) {
    char fpath[1024];
    fullpath(fpath, path);
    return unlink(fpath) == -1 ? -errno : 0;
}

static int xmp_create(const char *path, mode_t mode, struct fuse_file_info *fi) {
    char fpath[1024];
    fullpath(fpath, path);
    int fd = creat(fpath, mode);
    if (fd == -1) return -errno;
    close(fd);
    return 0;
}

static int xmp_rename(const char *from, const char *to) {
    char ffrom[1024], fto[1024];
    fullpath(ffrom, from);
    fullpath(fto, to);
    return rename(ffrom, fto) == -1 ? -errno : 0;
}

static struct fuse_operations xmp_oper = {
    .getattr = xmp_getattr,
    .readdir = xmp_readdir,
    .read    = xmp_read,
    .write   = xmp_write,
    .mkdir   = xmp_mkdir,
    .unlink  = xmp_unlink,
    .create  = xmp_create,
    .rename  = xmp_rename,
};

int main(int argc, char *argv[]) {
    umask(0);
    return fuse_main(argc, argv, &xmp_oper, NULL);
}
```
#### penjelasan :

1. `fullpath(char fpath[1024], const char *path)`
- Tujuan: Menggabungkan `source_path` (`/home/shared_files`) dengan `path` dari FUSE.
- Output: Path absolut dari file asli di sistem.
- Contoh:  
  `"/private_yuadi/file.txt"` → `"/home/shared_files/private_yuadi/file.txt"`

2. `get_username()`
- Tujuan: Mengambil username user yang sedang mengakses FUSE.
- Cara kerja: 
  - Ambil `uid` dari konteks FUSE (`fuse_get_context()->uid`)
  - Gunakan `getpwuid` untuk mendapatkan nama user dari UID tersebut

3. `is_private_yuadi(const char *path)`  
4. `is_private_irwandi(const char *path)`
- Tujuan: Mengecek apakah path mengarah ke folder privat milik `yuadi` atau `irwandi`.
- Logika:  
  - `strncmp(path, "/private_yuadi", 14) == 0` → return true  
  - `strncmp(path, "/private_irwandi", 16) == 0` → return true

5. `is_authorized(const char *path)`
- Tujuan: Mengecek apakah user saat ini memiliki izin untuk mengakses path privat.
- Logika:
  - Jika path ke `private_yuadi` → hanya user "yuadi" yang boleh.
  - Jika path ke `private_irwandi` → hanya user "irwandi" yang boleh.
  - Jika bukan path privat → siapa pun boleh akses.


6. `xmp_getattr(const char *path, struct stat *stbuf)`
- Tujuan: Mengambil atribut file atau direktori.
- Khusus path privat:
  - Jika user tidak berhak, maka folder tetap bisa ditampilkan (`ls`), tapi file di dalam tidak bisa diakses (`-EACCES`).

7. `xmp_readdir(const char *path, void *buf, fuse_fill_dir_t filler, off_t offset, struct fuse_file_info *fi)`
- Tujuan: Menampilkan isi direktori (digunakan saat `ls`).
- Langkah:
  - Cek otorisasi user.
  - Buka direktori dan isi dengan `readdir`.
  - Gunakan `filler()` untuk mengisi buffer direktori di FUSE.

8. `xmp_read(const char *path, char *buf, size_t size, off_t offset, struct fuse_file_info *fi)`
- Tujuan: Membaca isi file.
- Langkah:
  - Cek otorisasi.
  - Buka file secara read-only.
  - Baca dari offset tertentu menggunakan `pread`.

9. `xmp_write(const char *path, const char *buf, size_t size, off_t offset, struct fuse_file_info *fi)`
- Tujuan: Menulis data ke file.
- Langkah:
  - Buka file secara write-only.
  - Gunakan `pwrite()` untuk menulis dari offset tertentu.
- Catatan: Perlu ditambahkan pengecekan `is_authorized` untuk keamanan.

10. `xmp_mkdir(const char *path, mode_t mode)`
- Tujuan: Membuat direktori baru.
- Langkah:
  - Gabungkan path dengan `source_path`.
  - Gunakan `mkdir()`.

11. `xmp_unlink(const char *path)`
- Tujuan: Menghapus file.
- Langkah: Gunakan `unlink()` pada path absolut.

12. `xmp_create(const char *path, mode_t mode, struct fuse_file_info *fi)`
- Tujuan: Membuat file kosong baru.
- Langkah: Gunakan `creat()` dengan mode yang diberikan.

13. `xmp_rename(const char *from, const char *to)`
- Tujuan: Mengganti nama atau memindahkan file.
- Langkah: Gunakan `rename()` dengan path absolut.



14. `struct fuse_operations xmp_oper`
- Tujuan: Mendefinisikan fungsi-fungsi yang digunakan oleh FUSE saat operasi filesystem dijalankan.
- Fungsi-fungsi yang di-assign: `getattr`, `readdir`, `read`, `write`, `mkdir`, `unlink`, `create`, `rename`.


15. `int main(int argc, char *argv[])`
- Tujuan: Entry point program.
- Langkah:
  - `umask(0);` untuk menghindari pembatasan permission file oleh shell.
  - Memanggil `fuse_main()` dengan `xmp_oper`.





#### screenshot output :
![alt](https://github.com/Maleka0809/praktikum_4_sisop_screenshot/blob/4422fee60b3673ddcff1ebc73fe0282822992026/Screenshot%202025-06-18%20115431.png)


![alt](https://github.com/Maleka0809/praktikum_4_sisop_screenshot/blob/4a3bfb2a8758311959b4e6de50843ca0c488482a/Screenshot%202025-06-18%20172057.png)





### c. Read-Only untuk Semua User

Yuadi sangat kesal dengan kebiasaan Irwandi yang suka mengubah atau bahkan menghapus file jawaban setelah menyalinnya untuk menghilangkan jejak plagiat. Untuk mencegah hal ini, dia memutuskan untuk membuat seluruh sistem menjadi read-only.

1. Jadikan seluruh FUSE mount point **read-only untuk semua user**.
2. Ini berarti tidak ada user (termasuk `root`) yang dapat membuat, memodifikasi, atau menghapus file atau folder apapun di dalam `/mnt/secure_fs`. Command seperti `mkdir`, `rmdir`, `touch`, `rm`, `cp`, `mv` harus gagal semua.

"Sekarang Irwandi tidak bisa lagi menghapus jejak plagiatnya atau mengubah file jawabanku," pikir Yuadi puas.

#### kode :

```
#define FUSE_USE_VERSION 28
#include <fuse.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <dirent.h>
#include <errno.h>
#include <sys/time.h>
#include <stdlib.h>

static const char *source_path = "/home/shared_files";


void fullpath(char fpath[1024], const char *path) {
    snprintf(fpath, 1024, "%s%s", source_path, path);
}

static int xmp_getattr(const char *path, struct stat *stbuf) {
    int res;
    char fpath[1024];
    fullpath(fpath, path);
    res = lstat(fpath, stbuf);
    if (res == -1) return -errno;
    return 0;
}

static int xmp_readdir(const char *path, void *buf, fuse_fill_dir_t filler,
                       off_t offset, struct fuse_file_info *fi) {
    DIR *dp;
    struct dirent *de;
    char fpath[1024];
    fullpath(fpath, path);

    (void) offset;
    (void) fi;

    dp = opendir(fpath);
    if (dp == NULL) return -errno;

    while ((de = readdir(dp)) != NULL) {
        struct stat st;
        memset(&st, 0, sizeof(st));
        st.st_ino = de->d_ino;
        st.st_mode = de->d_type << 12;
        if (filler(buf, de->d_name, &st, 0)) break;
    }
    closedir(dp);
    return 0;
}

static int xmp_open(const char *path, struct fuse_file_info *fi) {
    char fpath[1024];
    fullpath(fpath, path);

    int fd = open(fpath, O_RDONLY);
    if (fd == -1) return -errno;
    close(fd);
    return 0;
}

static int xmp_read(const char *path, char *buf, size_t size,
                    off_t offset, struct fuse_file_info *fi) {
    char fpath[1024];
    fullpath(fpath, path);

    int fd = open(fpath, O_RDONLY);
    if (fd == -1) return -errno;

    int res = pread(fd, buf, size, offset);
    if (res == -1) res = -errno;
    close(fd);
    return res;
}

// Semua operasi tulis akan ditolak
static int deny_write_ops() {
    uid_t uid = fuse_get_context()->uid;
    fprintf(stderr, "Write operation blocked for UID %d\n", uid);
    return -EROFS;
}

static int xmp_mkdir(const char *path, mode_t mode)       { return deny_write_ops(); }
static int xmp_rmdir(const char *path)                    { return deny_write_ops(); }
static int xmp_rename(const char *from, const char *to)   { return deny_write_ops(); }
static int xmp_write(const char *path, const char *buf, size_t size, off_t offset, struct fuse_file_info *fi) {
    return deny_write_ops();
}
static int xmp_unlink(const char *path)                   { return deny_write_ops(); }
static int xmp_create(const char *path, mode_t mode, struct fuse_file_info *fi) {
    return deny_write_ops();
}
static int xmp_truncate(const char *path, off_t size)     { return deny_write_ops(); }

static struct fuse_operations xmp_oper = {
    .getattr = xmp_getattr,
    .readdir = xmp_readdir,
    .open    = xmp_open,
    .read    = xmp_read,
    .mkdir   = xmp_mkdir,
    .rmdir   = xmp_rmdir,
    .rename  = xmp_rename,
    .write   = xmp_write,
    .unlink  = xmp_unlink,
    .create  = xmp_create,
    .truncate = xmp_truncate,
};

int main(int argc, char *argv[]) {
    umask(0);
    return fuse_main(argc, argv, &xmp_oper, NULL);
}
```
#### penjelasan :
## Penjelasan Per Fungsi

1. `#define FUSE_USE_VERSION 28`  
   Menentukan bahwa program menggunakan API FUSE versi 2.8. Ini harus ditulis sebelum mengimpor `fuse.h`.

2. `static const char *source_path = "/home/shared_files";`  
   Variabel global yang menentukan direktori sumber di mana semua file dan folder fisik sebenarnya berada. Semua path dari FUSE akan digabungkan dengan direktori ini.

3. `void fullpath(char fpath[1024], const char *path)`  
   Fungsi bantu untuk membuat path absolut dari path FUSE.  
   Contoh: jika path adalah `/a.txt`, maka fungsi ini akan menghasilkan `/home/shared_files/a.txt`.

4. `int xmp_getattr(const char *path, struct stat *stbuf)`  
   Dipanggil saat sistem atau user meminta metadata file, seperti ukuran, waktu modifikasi, mode, dan sebagainya.  
   Fungsi ini akan:
   - Membuat path lengkap menggunakan `fullpath`
   - Memanggil `lstat()` pada file tersebut untuk mengisi struktur `stbuf`
   - Jika gagal, mengembalikan nilai negatif dari `errno`

5. `int xmp_readdir(const char *path, void *buf, fuse_fill_dir_t filler, off_t offset, struct fuse_file_info *fi)`  
   Dipanggil ketika isi direktori diminta, seperti saat menggunakan `ls`.  
   Fungsi ini akan:
   - Membuat path lengkap ke direktori
   - Membuka direktori menggunakan `opendir`
   - Iterasi setiap entri direktori dengan `readdir`
   - Mengisi buffer `buf` menggunakan `filler`

6. `int xmp_open(const char *path, struct fuse_file_info *fi)`  
   Dipanggil saat user mencoba membuka file.  
   Fungsi ini hanya membuka file dalam mode read-only (`O_RDONLY`).  
   File akan dibuka untuk dicek, lalu ditutup kembali karena FUSE mengelola file descriptor-nya.

7. `int xmp_read(const char *path, char *buf, size_t size, off_t offset, struct fuse_file_info *fi)`  
   Dipanggil saat user membaca isi file.  
   Fungsi ini akan membuka file dengan `O_RDONLY`, membaca `size` byte dari posisi `offset` menggunakan `pread`, lalu menutup file.  
   Hasil baca dikembalikan atau error jika gagal.

8. `int deny_write_ops()`  
   Fungsi umum yang dipanggil oleh semua operasi tulis seperti `write`, `mkdir`, `unlink`, `rename`, dan lainnya.  
   Fungsi ini akan:
   - Mengambil UID user menggunakan `fuse_get_context()->uid`
   - Mencetak log UID ke `stderr`
   - Mengembalikan error `-EROFS` (read-only filesystem)

9. Fungsi tulis berikut akan langsung gagal karena memanggil `deny_write_ops()`:
   - `xmp_mkdir`: untuk membuat direktori
   - `xmp_rmdir`: untuk menghapus direktori
   - `xmp_rename`: untuk mengganti nama file atau folder
   - `xmp_write`: untuk menulis data ke file
   - `xmp_unlink`: untuk menghapus file
   - `xmp_create`: untuk membuat file baru
   - `xmp_truncate`: untuk mengubah ukuran file

10. `static struct fuse_operations xmp_oper`  
    Struktur ini berisi daftar fungsi yang digunakan FUSE untuk menangani setiap operasi.  
    Fungsi baca seperti `getattr`, `readdir`, `open`, dan `read` diaktifkan.  
    Fungsi tulis diarahkan ke `deny_write_ops()` sehingga ditolak.

11. `int main(int argc, char *argv[])`  
    Fungsi utama program.  
    - `umask(0)` agar file permission tidak dibatasi
    - `fuse_main()` akan menjalankan filesystem virtual berdasarkan fungsi yang didefinisikan


#### screenshot output :
![alt](https://github.com/Maleka0809/praktikum_4_sisop_screenshot/blob/243eaec579833f8245e75ab98f5129c57c729e5f/Screenshot%202025-06-18%20173242.png)

### d. Akses Public Folder

Meski ingin melindungi jawaban praktikumnya, Yuadi tetap ingin berbagi materi kuliah dan referensi dengan Irwandi dan teman-teman lainnya.

Setiap user (termasuk `yuadi`, `irwandi`, atau lainnya) harus dapat **membaca** konten dari file apapun di dalam folder `public`. Misalnya, `cat /mnt/secure_fs/public/materi_kuliah.txt` harus berfungsi untuk `yuadi` dan `irwandi`.

#### penjelasan :
sama seperti kode yang c, karena untuk soal yang c command cat tetap diperbolehkan

#### screenshot :
![alt](https://github.com/Maleka0809/praktikum_4_sisop_screenshot/blob/ea3e11c1e167356084f0bf1a2179941f2b7f3b3e/Screenshot%202025-06-18%20174409.png)

### e. Akses Private Folder yang Terbatas

Inilah bagian paling penting dari rencana Yuadi - memastikan jawaban praktikum benar-benar terlindungi dari plagiat.

1. File di dalam `private_yuadi` **hanya dapat dibaca oleh user `yuadi`**. Jika `irwandi` mencoba membaca file jawaban praktikum di `private_yuadi`, harus gagal (contoh: permission denied).
2. Demikian pula, file di dalam `private_irwandi` **hanya dapat dibaca oleh user `irwandi`**. Jika `yuadi` mencoba membaca file di `private_irwandi`, harus gagal.

"Akhirnya," senyum Yuadi, "Irwandi tidak bisa lagi menyalin jawaban praktikumku, tapi dia tetap bisa mengakses materi kuliah yang memang kubuat untuk dibagi."


#### kode :
```
#define FUSE_USE_VERSION 28
#include <fuse.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <dirent.h>
#include <errno.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <stdlib.h>
#include <ctype.h>

static const char *source_path = "/home/shared_files";

// Konversi string ke lowercase
void tolower_str(char *dest, const char *src) {
    for (int i = 0; src[i]; ++i)
        dest[i] = tolower((unsigned char)src[i]);
    dest[strlen(src)] = '\0';
}

void fullpath(char fpath[1024], const char *path) {
    snprintf(fpath, 1024, "%s%s", source_path, path);
}

// Cek apakah user mencoba mengakses direktori private orang lain
int is_access_denied(const char *path) {
    uid_t uid = fuse_get_context()->uid;

    if (uid == 1001 && strncmp(path, "/private_irwandi/", 17) == 0)
        return 1;
    if (uid == 1002 && strncmp(path, "/private_yuadi/", 15) == 0)
        return 1;

    return 0;
}

// Cek apakah penulisan ditolak berdasarkan UID dan path
int is_write_denied(const char *path) {
    uid_t uid = fuse_get_context()->uid;

    if ((uid == 1001 && strncmp(path, "/private_yuadi/", 15) == 0) ||
        (uid == 1002 && strncmp(path, "/private_irwandi/", 17) == 0)) {
        return 0; // Diizinkan menulis
    }
    return 1; // Ditolak
}

static int xmp_getattr(const char *path, struct stat *stbuf) {
    if (is_access_denied(path)) return -EACCES;

    char fpath[1024];
    fullpath(fpath, path);
    int res = lstat(fpath, stbuf);
    return (res == -1) ? -errno : 0;
}

static int xmp_readdir(const char *path, void *buf, fuse_fill_dir_t filler,
                       off_t offset, struct fuse_file_info *fi) {
    if (is_access_denied(path)) return -EACCES;

    DIR *dp;
    struct dirent *de;
    char fpath[1024];
    fullpath(fpath, path);

    (void) offset;
    (void) fi;

    dp = opendir(fpath);
    if (dp == NULL) return -errno;

    while ((de = readdir(dp)) != NULL) {
        struct stat st;
        memset(&st, 0, sizeof(st));
        st.st_ino = de->d_ino;
        st.st_mode = de->d_type << 12;
        if (filler(buf, de->d_name, &st, 0)) break;
    }
    closedir(dp);
    return 0;
}

static int xmp_open(const char *path, struct fuse_file_info *fi) {
    if (is_access_denied(path)) return -EACCES;

    char fpath[1024];
    fullpath(fpath, path);
    int fd = open(fpath, O_RDONLY);
    return (fd == -1) ? -errno : (close(fd), 0);
}

static int xmp_read(const char *path, char *buf, size_t size,
                    off_t offset, struct fuse_file_info *fi) {
    if (is_access_denied(path)) return -EACCES;

    char fpath[1024];
    fullpath(fpath, path);
    int fd = open(fpath, O_RDONLY);
    if (fd == -1) return -errno;

    int res = pread(fd, buf, size, offset);
    close(fd);
    return (res == -1) ? -errno : res;
}

static int xmp_write(const char *path, const char *buf, size_t size,
                     off_t offset, struct fuse_file_info *fi) {
    return is_write_denied(path) ? -EROFS : -EROFS;
}

static int xmp_unlink(const char *path) {
    return is_write_denied(path) ? -EROFS : -EROFS;
}

static int xmp_create(const char *path, mode_t mode, struct fuse_file_info *fi) {
    if (is_write_denied(path)) return -EROFS;

    uid_t uid = fuse_get_context()->uid;
    if ((uid == 1001 && strncmp(path, "/private_yuadi/", 15) != 0) ||
        (uid == 1002 && strncmp(path, "/private_irwandi/", 17) != 0)) {
        return -EACCES;
    }

    char fpath[1024];
    fullpath(fpath, path);
    int fd = creat(fpath, mode);
    if (fd == -1) return -errno;
    close(fd);
    return 0;
}

static int xmp_truncate(const char *path, off_t size) {
    return is_write_denied(path) ? -EROFS : -EROFS;
}

static int xmp_rename(const char *from, const char *to) {
    return -EROFS;
}

static int xmp_mkdir(const char *path, mode_t mode) {
    uid_t uid = fuse_get_context()->uid;

    if ((uid == 1001 && strncmp(path, "/private_yuadi/", 15) != 0) ||
        (uid == 1002 && strncmp(path, "/private_irwandi/", 17) != 0)) {
        return -EACCES;
    }

    char fpath[1024];
    fullpath(fpath, path);
    return mkdir(fpath, mode) == -1 ? -errno : 0;
}

static int xmp_rmdir(const char *path) {
    return is_write_denied(path) ? -EROFS : -EROFS;
}

static struct fuse_operations xmp_oper = {
    .getattr = xmp_getattr,
    .readdir = xmp_readdir,
    .open    = xmp_open,
    .read    = xmp_read,
    .write   = xmp_write,
    .unlink  = xmp_unlink,
    .create  = xmp_create,
    .truncate = xmp_truncate,
    .rename  = xmp_rename,
    .mkdir   = xmp_mkdir,
    .rmdir   = xmp_rmdir,
};

int main(int argc, char *argv[]) {
    umask(0);
    return fuse_main(argc, argv, &xmp_oper, NULL);
}

```

#### penjelasan :
Fungsi Utama

tolower_str(char *dest, const char *src)
Konversi string src menjadi huruf kecil, disalin ke dest.

fullpath(char fpath[1024], const char *path)
Membentuk path absolut ke folder sumber (/home/shared_files) dari path relatif FUSE.

Contoh:
- path = "/test.txt" → fpath = "/home/shared_files/test.txt"

is_access_denied(const char *path)
Memblokir user:
- UID 1001 (yuadi) dari mengakses /private_irwandi/
- UID 1002 (irwandi) dari mengakses /private_yuadi/

Mengembalikan:
- 1 jika akses ditolak
- 0 jika diizinkan

is_write_denied(const char *path)
Mencegah semua user menulis, kecuali:
- UID 1001 boleh menulis ke /private_yuadi/
- UID 1002 boleh menulis ke /private_irwandi/

Implementasi FUSE

xmp_getattr
Mengambil atribut file (mirip stat/ls -l). Menolak jika akses ke direktori terlarang.

xmp_readdir
Menampilkan isi direktori (ls). Ditolak jika direktori privat milik user lain.

xmp_open
Membuka file dalam mode read-only. Ditolak jika akses ke direktori privat orang lain.

xmp_read
Membaca isi file. Ditolak jika file berada di direktori privat orang lain.

xmp_write
Selalu ditolak (-EROFS) kecuali jika user punya izin menulis berdasarkan is_write_denied.

xmp_unlink
Menghapus file. Ditolak kecuali oleh user yang sesuai di direktori privatnya.

xmp_create
Membuat file baru. Hanya diizinkan untuk:
- yuadi → /private_yuadi/
- irwandi → /private_irwandi/

xmp_truncate
Memotong isi file. Diizinkan hanya jika user punya hak menulis di folder privatnya.

xmp_rename
Selalu ditolak. Sistem bersifat read-only terkait operasi rename.

xmp_mkdir
Membuat folder baru. Hanya diizinkan di folder privat masing-masing user.

xmp_rmdir
Menghapus folder. Hanya diizinkan jika punya hak tulis.

main
Menjalankan FUSE dengan:
- umask(0) agar tidak mengubah mode file asli
- fuse_main(...) dengan operasi yang sudah didefinisikan.



#### screenshot :
![alt](https://github.com/Maleka0809/praktikum_4_sisop_screenshot/blob/486519382c363e27fdac6bb9741263ff1c2300f3/Screenshot%202025-06-18%20193424.png)

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

