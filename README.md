Nama: Nisriina Wakhdah Haris<br>
NPM: 2406360445<br>
kelas: A<br>

# Individual Work - Bacaan Module


## Component Diagram
![Gambar component diagram be-bacaan](/image/Adpro-Component%20digram%20-%20Bacaan.drawio.png)

Berikut adalah penjelasan tiap lapisan (layer) dalam diagram tersebut:

### 1. Security Layer

Lapisan ini adalah pintu masuk utama untuk memvalidasi keamanan setiap permintaan (request).

* **SupabaseJWTFilter**: Berfungsi sebagai filter keamanan yang memverifikasi token Bearer JWT menggunakan Supabase JWKS untuk memastikan hanya pengguna terautentikasi yang dapat mengakses API.

### 2. Controller Layer

Berfungsi sebagai *entry point* API yang menangani request HTTP dan mengarahkannya ke service yang sesuai.

* **AdminReadingController**: Menangani operasi CRUD untuk konten bacaan (readings).
* **AdminQuizController**: Mengelola pembuatan dan pengaturan soal kuis.
* **StudentReadingController**: Menyediakan akses bagi siswa untuk melihat daftar bacaan, statistik, dan menandai bacaan yang selesai.
* **StudentQuizController**: Menangani pengambilan soal kuis dan pengiriman (submit) jawaban oleh siswa.
* **GlobalExceptionHandler**: Komponen `@RestControllerAdvice` yang menangani error secara terpusat di seluruh controller agar respon error tetap konsisten.

### 3. Service Layer

Lapisan ini berisi logika bisnis utama aplikasi.

* **AdminReadingService & StudentReadingService**: Memisahkan logika pengelolaan bacaan antara kebutuhan administratif dan konsumsi siswa.
* **AdminQuizService & StudentQuizService**: Mengelola alur kuis, mulai dari manajemen soal hingga proses penilaian (grading).
* **QuizService**: Komponen inti yang menangani penyelesaian kuis (`completeQuiz`) dan memperbarui kemajuan pengguna (`save UserProgress`).

### 4. Validator Layer (Strategy Pattern)

Sistem ini menggunakan **Strategy Pattern** melalui `QuizValidatorFactory` untuk memvalidasi berbagai jenis soal secara dinamis:

* **MultipleChoiceValidator**: Memvalidasi jawaban pilihan ganda.
* **TrueFalseValidator**: Memvalidasi jawaban benar/salah.
* **EssayValidator**: Memvalidasi jawaban esai.

### 5. Repository Layer & Database

Lapisan ini menangani interaksi langsung dengan basis data menggunakan Spring Data JPA.

* **ReadingRepository**: Mengakses data tabel bacaan.
* **QuizRepository**: Mengakses data tabel pertanyaan kuis.
* **UserProgressRepository**: Menyimpan status dan kemajuan belajar siswa.
* **Database be-bacaan**: Database pusat tempat semua data disimpan secara persisten.

## Code Diagram
![Gambar component diagram be-bacaan](/image/code%20diagram%20be-bacaan.png)

### 1. Layered Architecture

Sistem ini menggunakan pola desain Spring Boot standar untuk memastikan memisahkan tanggung jawab (*Separation of Concerns*):

* **Controller Layer**: Berfungsi sebagai antarmuka API. Terdapat pemisahan antara `AdminReadingController` (untuk manajemen konten admin) dan `StudentReadingController` (untuk konsumsi konten siswa), memungkinkan penerapan kebijakan keamanan yang berbeda pada level endpoint.
* **Service Layer**: Menangani logika bisnis yang kompleks. Komponen seperti `StudentQuizService` dan `AdminQuizService` bertindak sebagai perantara yang mengoordinasikan data antara controller dan repository.
* **Repository Layer**: Menggunakan abstraksi Spring Data JPA melalui interface seperti `ReadingRepository` dan `QuizRepository` untuk berinteraksi dengan database tanpa perlu menulis query SQL manual.

### 2. Implementasi Desain Pattern: Strategy & Factory

* **QuizValidator (Interface)**: Menetapkan kontrak standar untuk metode `validate()`, memungkinkan sistem untuk memperlakukan semua jenis validasi dengan cara yang sama (polimorfisme).
* **Concrete Validators**: Terdapat kelas spesifik seperti `MultipleChoiceValidator`, `TrueFalseValidator`, dan `EssayValidator`. Setiap kelas hanya bertanggung jawab atas satu jenis logika validasi, memenuhi *Single Responsibility Principle*.
* **QuizValidatorFactory**: Berperan sebagai pusat kontrol yang menentukan validator mana yang harus digunakan berdasarkan tipe soal yang diterima saat runtime, memungkinkan sistem bersifat *open-ended* dan memenuhi prinsip *open-closed principle*

### 3. Model Data dan Relasi (Domain Model)

* **Reading & Question**: Terdapat relasi di mana satu `Reading` (bacaan) dapat memiliki banyak `Question` (pertanyaan kuis).
* **UserProgress**: Kelas ini sangat penting karena mencatat status belajar siswa, termasuk apakah suatu bacaan telah selesai dan skor kuis yang diperoleh.
* **DTO (Data Transfer Objects)**: Terdapat berbagai kelas pendukung (seperti `ReadingResponse` atau `QuizRequest`) yang berfungsi untuk membungkus data saat dikirim melalui network, memastikan data internal database tidak terekspos seluruhnya ke frontend.

### 4. Aspek Keamanan dan Penanganan Error

* **SupabaseJWTFilter**: Terintegrasi di level awal untuk melakukan inspeksi terhadap setiap request, memastikan pengguna memiliki izin yang valid sebelum mencapai logika bisnis.
* **GlobalExceptionHandler**: Menggunakan anotasi `@RestControllerAdvice` untuk menangkap semua pengecualian (exceptions) yang terjadi di aplikasi dan mengubahnya menjadi pesan error yang rapi dan mudah dimengerti oleh sisi klien.




