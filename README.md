# Module 9: Visualizing Software Architecture - YOMU

Dokumen ini menjelaskan arsitektur YOMU dalam kondisi saat ini (_current architecture_) menggunakan pendekatan C4 Model. Fokusnya adalah memahami bentuk sistem, aktor yang memakai sistem, container utama, relasi antarservice, dan deployment awal yang tergambar dari repository serta diagram yang tersedia.

## 1. Current Architecture of YOMU

YOMU adalah platform pembelajaran membaca berbasis web. Secara arsitektur, YOMU dibangun sebagai aplikasi frontend Next.js yang berkomunikasi dengan beberapa backend Spring Boot. Backend dipisahkan berdasarkan domain fitur sehingga setiap service memiliki tanggung jawab yang relatif jelas.

| Bagian           | Teknologi                                           | Tanggung jawab utama                                                                                 |
| ---------------- | --------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `frontend`       | Next.js, React, TypeScript                          | UI web untuk admin dan student, routing halaman, session cookie, dan sebagian proxy request backend. |
| `be-auth`        | Spring Boot, Supabase Auth, PostgreSQL/Supabase DB  | Login, register, refresh token, logout, profil user, role admin/student, dan validasi identitas.     |
| `be-bacaan`      | Spring Boot, PostgreSQL, Supabase JWT config        | Bacaan, quiz, progress membaca, submit quiz, dan integrasi event quiz selesai ke achievement.        |
| `be-forum`       | Spring Boot, PostgreSQL, Supabase JWT/JWKS          | Forum message, reply, reaction, dan validasi write operation dengan token Supabase.                  |
| `be-liga`        | Spring Boot, Supabase JWT/JWKS, Supabase PostgreSQL | Clan/liga, member, application, join/quit clan, dan validasi user berbasis token Supabase.           |
| `be-achievement` | Spring Boot, PostgreSQL, Auth service validation    | Achievement, daily mission, student progress, featured achievement, dan event `quiz-completed`.      |
| Supabase Auth    | External service                                    | Identity provider, JWT issuer, JWKS provider, dan integrasi Google SSO.                              |
| Google OAuth     | External service                                    | Provider login Google yang digunakan melalui Supabase.                                               |

Arsitektur ini dapat dipahami sebagai **modular backend / microservice sederhana**. Pemisahan domain sudah terlihat dari folder dan service: auth, bacaan, forum, liga, dan achievement. Setiap service memiliki API sendiri dan database/domain data sendiri. Keuntungan pendekatan ini adalah domain lebih mudah dipahami dan ownership fitur lebih jelas. Trade-off-nya adalah integrasi antarservice dan konfigurasi deployment menjadi lebih kompleks dibanding aplikasi monolitik.

Beberapa karakter penting dari arsitektur saat ini:

- Frontend menjadi entry point utama untuk student dan admin.
- Auth dan forum sudah memiliki pola akses yang lebih terkontrol melalui Next.js API route/proxy.
- Bacaan dan liga pada beberapa bagian frontend masih terlihat dipanggil langsung menggunakan URL service lokal seperti `localhost:808x`.
- Supabase dipakai sebagai pusat identitas melalui JWT/JWKS.
- `be-bacaan` memiliki integrasi ke `be-achievement` ketika quiz selesai, sehingga achievement dapat diperbarui berdasarkan aktivitas student.
- Setiap domain utama memiliki database PostgreSQL sendiri atau koneksi PostgreSQL yang dipisahkan secara logis.

## 2. System Context Diagram

System Context Diagram melihat YOMU sebagai satu software system utuh. Diagram ini menunjukkan siapa pengguna YOMU dan external system apa yang langsung relevan terhadap sistem.

```mermaid
flowchart LR
  Student[Student]
  Admin[Admin]
  Supabase[Supabase Auth]
  Google[Google OAuth]

  YOMU["YOMU Learning Platform"]

  Student -->|"reads, takes quizzes, uses forum, joins clan, views achievements"| YOMU
  Admin -->|"manages users, readings, quizzes, and achievements"| YOMU
  YOMU -->|"authentication, JWT issuance, JWKS validation, user identity"| Supabase
  YOMU -->|"Google sign-in flow through Supabase"| Google
  Google -.->|"OAuth identity provider for"| Supabase
```

### Penjelasan Context Diagram

Pada level context, YOMU diposisikan sebagai sistem utama yang memberikan nilai ke dua jenis pengguna: student dan admin. Student menggunakan sistem untuk aktivitas pembelajaran, sedangkan admin menggunakan sistem untuk mengelola data dan fitur pembelajaran.

| Elemen                 | Jenis                    | Analisis                                                                                                                                                                                                                                        |
| ---------------------- | ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Student                | Person                   | Pengguna utama. Student membaca bacaan, mengerjakan quiz, melihat progress, memakai forum, bergabung ke clan, dan melihat achievement. Availability fitur student penting karena alur belajar bergantung pada beberapa backend sekaligus.       |
| Admin                  | Person                   | Pengguna pengelola sistem. Admin membutuhkan akses ke fitur administrasi seperti pengelolaan bacaan, quiz, user, achievement, dan konfigurasi konten. Security pada akses admin menjadi aspek penting karena dampaknya langsung ke data sistem. |
| YOMU Learning Platform | Software System          | Sistem yang dikembangkan oleh kelompok. Pada level context, seluruh frontend dan backend dianggap sebagai satu sistem karena semuanya bersama-sama membentuk pengalaman YOMU.                                                                   |
| Supabase Auth          | External Software System | Komponen eksternal paling penting untuk identitas. Supabase menerbitkan token, menyediakan JWKS untuk validasi token, dan menjadi dasar otorisasi bagi beberapa backend, termasuk `be-liga`.                                                    |
| Google OAuth           | External Software System | Provider login eksternal. Dalam arsitektur ini, Google OAuth bukan dipanggil sebagai service domain bisnis, tetapi sebagai bagian dari alur autentikasi melalui Supabase.                                                                       |

## 3. Container Diagram

Diagram ini memperlihatkan container utama di dalam YOMU, relasi antarcontainer, dan database yang digunakan masing-masing domain.

![YOMU Container Diagram](image/container.png)

### Penjelasan Container Diagram

Container diagram menunjukkan bahwa YOMU tidak hanya terdiri dari satu aplikasi backend. Sistem ini terbagi menjadi frontend Next.js dan beberapa backend Spring Boot yang berkomunikasi lewat REST API.

| Container               | Peran                            | Analisis                                                                                                                                                                                                                                                    |
| ----------------------- | -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Single Page Application | UI utama untuk admin dan student | Container ini adalah pintu masuk semua user. Admin dan student mengakses aplikasi melalui browser menggunakan HTTPS. Frontend bertugas menampilkan UI dan mengirim request ke backend domain yang sesuai.                                                   |
| Authentication          | Service auth dan profile user    | Service ini menangani autentikasi user, data profil, dan role. Karena service lain membutuhkan identitas user, auth menjadi komponen penting untuk security dan user management.                                                                            |
| Bacaan                  | Service bacaan dan quiz          | Service ini menangani fitur pembelajaran inti: daftar bacaan, detail bacaan, quiz, submit quiz, dan progress. Karena ini domain utama aplikasi, availability dan data integrity pada service ini penting.                                                   |
| Forum                   | Service forum diskusi            | Service ini menangani message, reply, dan reaction. Forum bergantung pada token user agar operasi tulis dapat dikaitkan dengan pemilik message/reaction.                                                                                                    |
| Liga                    | Service clan/liga                | Service ini menangani fitur clan, anggota, dan pendaftaran clan. Berdasarkan repository, `be-liga` adalah Spring Boot service yang memakai Supabase JWT/JWKS untuk validasi token dan memakai Supabase PostgreSQL sebagai database pada deployment diagram. |
| Achievements            | Service achievement              | Service ini menangani achievement, daily mission, dan progress achievement. Service ini juga menerima event dari bacaan ketika quiz selesai.                                                                                                                |
| Authentication Database | Database auth                    | Menyimpan data user/profile dan data terkait auth lokal. Pada diagram deployment, database ini ditempatkan di Supabase.                                                                                                                                     |
| Bacaan Database         | Database bacaan                  | Menyimpan data bacaan, quiz, question, dan progress membaca.                                                                                                                                                                                                |
| Forum Database          | Database forum                   | Menyimpan message, reply, reaction, dan data forum lain.                                                                                                                                                                                                    |
| Liga Database           | Database liga                    | Menyimpan clan, member, application, dan data domain liga. Pada deployment diagram, database ini ditempatkan di Supabase.                                                                                                                                   |
| Achievements Database   | Database achievement             | Menyimpan achievement, daily mission, user achievement, dan progress achievement.                                                                                                                                                                           |

### Relasi Penting Antarcontainer

Frontend melakukan REST API call ke service domain sesuai fitur yang sedang dipakai user. Untuk auth, request diarahkan ke Authentication. Untuk bacaan, request diarahkan ke Bacaan. Untuk forum, request diarahkan ke Forum. Untuk clan/liga, request diarahkan ke Liga. Untuk achievement, request diarahkan ke Achievements.

Relasi yang paling penting secara arsitektural:

- `Single Page Application -> Authentication`: dipakai untuk login, register, session, profile, dan role.
- `Single Page Application -> Bacaan`: dipakai untuk fitur membaca dan quiz.
- `Single Page Application -> Forum`: dipakai untuk forum diskusi dan reaction.
- `Single Page Application -> Liga`: dipakai untuk fitur clan/liga.
- `Single Page Application -> Achievements`: dipakai untuk melihat dan mengelola achievement.
- `Bacaan -> Achievements`: dipakai untuk mengirim event ketika quiz selesai agar achievement student dapat diperbarui.
- `Forum` dan `Liga -> Supabase Auth/JWKS`: dipakai untuk validasi JWT dari Supabase. Khusus `be-liga`, dependency ini penting karena service memakai `AUTH_SUPABASE_URL` sebagai issuer JWT.
- `Bacaan -> Supabase JWT/JWKS config`: service bacaan memiliki konfigurasi Supabase JWT/JWKS, walaupun enforcement security perlu dilihat dari implementasi filter dan endpoint.
- `Achievements -> Authentication`: `be-achievement` memvalidasi token dengan memanggil `be-auth` melalui `AUTH_SERVICE_URL`, sehingga availability auth service ikut memengaruhi endpoint achievement yang protected.

### Analisis Arsitektural dari Container Diagram

Pemisahan container sudah cukup baik karena mengikuti domain bisnis. Bacaan, forum, liga, dan achievement tidak dicampur dalam satu backend besar. Ini membuat perubahan pada satu fitur lebih terisolasi. Misalnya perubahan pada forum tidak harus mengubah service bacaan.

Namun, container diagram juga memperlihatkan titik integrasi yang perlu dijaga:

- Authentication menjadi pusat identitas. Jika konfigurasi JWT/JWKS salah, beberapa service dapat gagal mengenali user.
- Frontend menjadi penghubung ke banyak backend. Jika pola pemanggilan backend tidak konsisten, deployment dan CORS menjadi lebih sulit.
- Bacaan dan Achievement memiliki coupling melalui event quiz selesai. Ini masuk akal secara bisnis, tetapi perlu dijaga agar kegagalan achievement tidak menggagalkan proses submit quiz.
- Setiap service memiliki database sendiri, yang bagus untuk isolasi domain. Konsekuensinya, query lintas domain tidak bisa sembarangan join database dan harus melalui API/service boundary.

## 4. Deployment Diagram

![YOMU Deployment Diagram](image/deployement.png)

### Penjelasan Deployment Diagram

Deployment diagram menunjukkan bahwa frontend, backend, dan database tidak semuanya berada di satu tempat. Frontend ditempatkan sebagai aplikasi Next.js di Vercel. Backend service berjalan sebagai aplikasi Spring Boot pada AWS EC2 Ubuntu Server. Database dipisahkan berdasarkan service: `be-auth` dan `be-liga` memakai PostgreSQL yang diakses melalui Supabase pooler, sedangkan `be-forum` dan `be-achievement` memiliki konfigurasi docker-compose dengan PostgreSQL container sendiri. `be-bacaan` memakai PostgreSQL biasa dengan konfigurasi default `jdbc:postgresql://localhost:5432/yomu_db`, yang pada deployment diagram direpresentasikan sebagai database terpisah untuk service bacaan.

| Deployment Node          | Container                                  | Analisis                                                                                                                                                                                                                                                            |
| ------------------------ | ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Vercel                   | Single Page Application / Next.js frontend | Vercel menjadi tempat deployment frontend. User mengakses frontend melalui browser, kemudian frontend melakukan request API ke backend service.                                                                                                                     |
| AWS EC2 Ubuntu Server    | Authentication service (`be-auth`)         | `be-auth` berjalan sebagai Spring Boot service. Service ini terhubung ke Supabase untuk auth/JWT dan menggunakan datasource PostgreSQL dari Supabase Session Pooler melalui `JDBC_DATABASE_URL` atau komponen `DB_HOST`, `DB_USER`, dan `DB_PASSWORD`.              |
| AWS EC2 Ubuntu Server    | Bacaan service (`be-bacaan`)               | `be-bacaan` berjalan sebagai Spring Boot service untuk bacaan dan quiz. Konfigurasi datasource default mengarah ke PostgreSQL `yomu_db`; service ini juga memiliki konfigurasi Supabase JWT/JWKS dan memanggil `be-achievement` ketika quiz selesai.                |
| AWS EC2 Ubuntu Server    | Forum service (`be-forum`)                 | `be-forum` berjalan sebagai Spring Boot service untuk forum. Database forum berasal dari `DB_URL`; pada docker-compose service ini menjalankan PostgreSQL container `db` sendiri, sementara Supabase dipakai untuk validasi JWT/JWKS, bukan sebagai database forum. |
| AWS EC2 Ubuntu Server    | Liga service (`be-liga`)                   | `be-liga` berjalan sebagai Spring Boot service untuk fitur clan/liga. `.env.example` menunjukkan `DB_URL` memakai Supabase pooler dan `AUTH_SUPABASE_URL` dipakai sebagai issuer JWT, sehingga liga memakai Supabase untuk database sekaligus autentikasi/JWKS.     |
| AWS EC2 Ubuntu Server    | Achievements service (`be-achievement`)    | `be-achievement` berjalan sebagai Spring Boot service untuk achievement dan daily mission. Docker-compose menyediakan PostgreSQL container sendiri, dan service ini memakai `AUTH_SERVICE_URL` untuk validasi token melalui `be-auth`.                              |
| Supabase                 | Authentication Database                    | Database auth ditempatkan di Supabase PostgreSQL. Ini sesuai dengan konfigurasi `be-auth` yang memprioritaskan `JDBC_DATABASE_URL` dari Supabase Session Pooler.                                                                                                    |
| Supabase                 | Liga Database                              | Database liga ditempatkan di Supabase PostgreSQL. Ini sesuai dengan `be-liga/.env.example` yang memakai host pooler Supabase pada `DB_URL`.                                                                                                                         |
| AWS EC2 / Docker network | Bacaan Database                            | Database bacaan adalah PostgreSQL domain bacaan. Dari konfigurasi service, default-nya adalah `jdbc:postgresql://localhost:5432/yomu_db`; pada deployment, database ini diperlakukan sebagai database terpisah dari Supabase auth/liga.                             |
| AWS EC2 / Docker network | Forum Database                             | Database forum adalah PostgreSQL container `db` dari `be-forum/docker-compose.yml`. Ia menyimpan data message, reply, dan reaction.                                                                                                                                 |
| AWS EC2 / Docker network | Achievements Database                      | Database achievement adalah PostgreSQL container dari `be-achievement/docker-compose.yml` atau datasource yang diberikan lewat `SPRING_DATASOURCE_URL`. Ia menyimpan achievement, daily mission, dan progress achievement.                                          |

## 5. The Future Architecture of the Group

Jika YOMU berhasil dan jumlah pengguna meningkat, risiko terbesar dari arsitektur saat ini bukan pada pemisahan domainnya, tetapi pada konsistensi integrasi antarservice. Risk storming diterapkan untuk melihat risiko dari beberapa sudut pandang sekaligus: availability, security, data integrity, scalability, dan deployment. Dengan teknik ini, tim dapat menilai area berisiko tinggi sebelum sistem benar-benar menerima traffic besar.

Hasil risk storming awal menunjukkan beberapa risiko utama:

| Area | Risiko saat YOMU tumbuh | Dampak |
|---|---|---|
| Availability | Frontend bergantung pada banyak backend service | Jika satu service down, fitur terkait langsung gagal untuk student/admin. |
| Data integrity | Event quiz selesai dari `be-bacaan` ke `be-achievement` masih synchronous HTTP | Progress quiz bisa tersimpan, tetapi achievement bisa gagal diperbarui. |
| Security | Validasi JWT dan role tersebar di beberapa service | Salah konfigurasi issuer, JWKS, atau role dapat membuka endpoint sensitif. |
| Scalability | Request antarservice masih request-response langsung | Latency dan cascading failure meningkat saat traffic naik. |
| Deployment | Konfigurasi URL, database, port, dan environment berbeda antarservice | Deployment production menjadi mudah salah konfigurasi. |

Arsitektur masa depan yang diharapkan tidak mengubah seluruh sistem secara ekstrem. Perubahannya fokus pada mitigasi yang realistis: semua request frontend masuk melalui API Gateway/BFF, validasi auth dibuat konsisten, komunikasi event quiz ke achievement dipindah ke message queue, dan setiap service tetap mempertahankan database domain masing-masing.

### Future Context Diagram

```mermaid
flowchart LR
  Student[Student]
  Admin[Admin]
  Supabase[Supabase Auth]
  Google[Google OAuth]
  Observability[Monitoring and Logging Platform]

  YOMU["YOMU Learning Platform\nFuture Architecture"]

  Student -->|"learns, quizzes, joins forum/clan, tracks achievements"| YOMU
  Admin -->|"manages content, users, quizzes, and achievements"| YOMU
  YOMU -->|"authentication, JWT, JWKS, identity"| Supabase
  YOMU -->|"Google sign-in via Supabase"| Google
  YOMU -->|"health metrics, logs, alerts"| Observability
  Google -.->|"OAuth provider"| Supabase
```

Pada context level, aktor utama tetap sama: student dan admin. Perubahan pentingnya adalah sistem masa depan menambahkan monitoring/logging sebagai external supporting system. Saat YOMU semakin besar, observability dibutuhkan agar tim dapat melihat service yang lambat, gagal, atau mengalami error sebelum berdampak luas ke pengguna.

### Future Container Diagram

```mermaid
flowchart TB
  Student[Student]
  Admin[Admin]

  subgraph YOMU["YOMU Learning Platform"]
    FE["Single Page Application\nNext.js"]
    BFF["API Gateway / BFF\nCentral backend entry point"]
    Auth["Authentication Service\nSpring Boot + Supabase Auth"]
    Bacaan["Bacaan Service\nReading and Quiz"]
    Forum["Forum Service\nMessage, Reply, Reaction"]
    Liga["Liga Service\nClan and Members"]
    Achievement["Achievement Service\nAchievement and Daily Mission"]
    Queue["Event Queue\nQuizCompleted events"]

    AuthDb[("Auth DB\nSupabase PostgreSQL")]
    BacaanDb[("Bacaan DB\nPostgreSQL")]
    ForumDb[("Forum DB\nPostgreSQL")]
    LigaDb[("Liga DB\nSupabase PostgreSQL")]
    AchievementDb[("Achievement DB\nPostgreSQL")]
  end

  Supabase["Supabase Auth / JWKS"]
  Monitor["Monitoring and Logging"]

  Student --> FE
  Admin --> FE
  FE --> BFF

  BFF --> Auth
  BFF --> Bacaan
  BFF --> Forum
  BFF --> Liga
  BFF --> Achievement

  Bacaan -->|"publish QuizCompleted"| Queue
  Queue -->|"consume event"| Achievement

  Auth --> AuthDb
  Bacaan --> BacaanDb
  Forum --> ForumDb
  Liga --> LigaDb
  Achievement --> AchievementDb

  Auth --> Supabase
  Bacaan --> Supabase
  Forum --> Supabase
  Liga --> Supabase
  Achievement --> Auth

  BFF --> Monitor
  Auth --> Monitor
  Bacaan --> Monitor
  Forum --> Monitor
  Liga --> Monitor
  Achievement --> Monitor
```

Future container diagram mempertahankan pemisahan domain yang sudah ada, tetapi menambahkan dua elemen penting. Pertama, API Gateway/BFF menjadi entry point backend yang konsisten sehingga frontend tidak perlu memanggil banyak backend secara langsung. Kedua, Event Queue memisahkan `be-bacaan` dan `be-achievement`, sehingga submit quiz tidak bergantung langsung pada availability achievement service.

Dengan perubahan ini, risiko utama dapat dikurangi tanpa menghilangkan struktur microservice sederhana yang sudah dibangun. Availability meningkat karena kegagalan achievement tidak langsung menggagalkan quiz. Security lebih mudah dikontrol karena akses backend masuk melalui satu lapisan yang konsisten. Scalability juga lebih baik karena event berulang seperti quiz completion dapat diproses secara asynchronous.


## Alvin Christian Halim - Individual
Code diagram
![](./image/code-diagram.png)

Component Diagram

![](./image/component.png)


