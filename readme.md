# CareerBytes Backend — Docker, Nginx Reverse Proxy & Load Balancing

Repository ini berisi implementasi containerization backend **CareerBytes** (Express, TypeScript, Drizzle ORM, PostgreSQL) menggunakan Docker Compose serta arsitektur **Nginx Reverse Proxy**, **Load Balancing (3 Replika)**, **Rate Limiting**, **Security Headers**, dan **Gzip Compression**.

## Arsitektur Sistem
- **Nginx (`nginx:alpine`)**: Satu-satunya pintu masuk (*single entry point*) di port `80` yang meneruskan trafik `/api/` ke cluster backend.
- **Backend (`3 replika`)**: Aplikasi Node.js/Express berjalan di jaringan internal Docker pada port `3000` tanpa mengekspos port ke host.
- **Database (`postgres:16-alpine`)**: Menyimpan data secara persisten menggunakan named volume `pgdata` tanpa mengekspos port ke host.

## Prasyarat
- Docker Engine & Docker Compose v2+
- `curl` atau PowerShell (untuk pengujian endpoint)

## Setup Environment & Cara Menjalankan

1. **Clone repository dan salin konfigurasi environment:**
   ```bash
   git clone [https://github.com/Rafi-agastya/study-club-backend-docker.git](https://github.com/Rafi-agastya/study-club-backend-docker.git)
   cd study-club-backend-docker
   cp .env.example .env

   docker compose up -d --build

   docker compose exec backend npx drizzle-kit push
   docker compose exec backend node dist/db/seedTrendingSkills.js

   1..6 | ForEach-Object { curl.exe -sI -H "Authorization: Bearer <TOKEN>" http://localhost/api/trending-skills | Select-String "X-Upstream"; Start-Sleep -Milliseconds 400 }

   Hasil :
    X-Upstream: 172.21.0.4:3000
    X-Upstream: 172.21.0.5:3000
    X-Upstream: 172.21.0.3:3000
    X-Upstream: 172.21.0.4:3000
    X-Upstream: 172.21.0.5:3000
    X-Upstream: 172.21.0.3:3000

    1..15 | ForEach-Object { $code = curl.exe -s -o NUL -w "%{http_code}" http://localhost/api/trending-skills; "Request $_`: HTTP $code" }

    hasil : 
    Request 1: HTTP 401
    Request 2: HTTP 401
    Request 3: HTTP 401
    Request 4: HTTP 401
    Request 5: HTTP 401
    Request 6: HTTP 401
    Request 7: HTTP 429
    Request 8: HTTP 429
    Request 9: HTTP 429
    Request 10: HTTP 429
    Request 11: HTTP 429
    Request 12: HTTP 401
    Request 13: HTTP 429
    Request 14: HTTP 429
    Request 15: HTTP 429 
    
    curl.exe -si -H "Accept-Encoding: gzip" -H "Authorization: Bearer <TOKEN>" http://localhost/api/trending-skills | Select-Object -First 13

    hasil : 
    HTTP/1.1 200 OK
    Server: nginx/1.31.8
    Date: Thu, 12 Nov 2026 13:07:06 GMT
    Content-Type: application/json; charset=utf-8
    Transfer-Encoding: chunked
    Connection: keep-alive
    Vary: Accept-Encoding
    X-Powered-By: Express
    Access-Control-Allow-Origin: *
    ETag: W/"3f8-4ocBlAhMuvw+xDosFoiEL5xsEm4"
    X-Upstream: 172.21.0.5:3000
    X-Content-Type-Options: nosniff
    Content-Encoding: gzip

    curl.exe -v http://localhost:3000/api/trending-skills
    Hasil :
    - Host localhost:3000 was resolved.
    - IPv6: ::1
    - IPv4: 127.0.0.1
    *   Trying [::1]:3000...
    *   Trying 127.0.0.1:3000...
    - connect to ::1 port 3000 from :: port 60640 failed: Connection refused
    - connect to 127.0.0.1 port 3000 from 0.0.0.0 port 60641 failed: Connection refused
    - Failed to connect to localhost port 3000 after 2253 ms: Could not connect to server
    - closing connection #0
    curl: (7) Failed to connect to localhost port 3000 after 2253 ms: Could not connect to server

Jawaban Pertanyaan Analisis :
(a) Kenapa upstream perlu zone? Apa yang terjadi kalau dihapus?
Directive zone mengalokasikan area shared memory agar seluruh worker process Nginx dapat berbagi state dan informasi status server upstream secara real-time, seperti jumlah koneksi aktif dan status kesehatan server. Selain itu, zone wajib ada agar Nginx bisa melakukan resolusi DNS dinamis untuk container Docker yang memiliki beberapa replika. Jika zone dihapus, setiap worker process akan menyimpan state upstream secara terpisah di memori lokalnya masing-masing sehingga pembagian beban menjadi tidak sinkron dan fitur manajemen state upstream dinamis tidak dapat berfungsi.

(b) Kenapa limit_req_status diganti ke 429? Apa bedanya dengan bawaan Nginx?
Status 429 (Too Many Requests) merupakan standar resmi HTTP (RFC 6585) yang secara eksplisit memberi tahu klien atau frontend bahwa permintaan ditolak karena melebihi batas frekuensi akses (rate limit). Secara bawaan (default), Nginx menggunakan status 503 (Service Temporarily Unavailable) untuk menolak permintaan yang terkena limit_req. Penggunaan 503 sering disalahartikan oleh sistem monitoring maupun pengguna sebagai gangguan atau crash pada server backend, sedangkan 429 memperjelas bahwa server sehat namun klien perlu memperlambat laju permintaannya.

(c) Kenapa backend dan database tidak punya ports, dan Nginx menjangkau backend lewat apa?
Konfigurasi ports pada backend dan database dihapus untuk menerapkan prinsip defense-in-depth dan single entry point, sehingga klien eksternal tidak dapat membypass lapisan keamanan, rate limiting, maupun load balancing di Nginx untuk mengakses backend atau database secara langsung dari host. Nginx tetap dapat menjangkau container backend melalui jaringan internal bawaan Docker Compose (bridge network) menggunakan mekanisme service discovery berbasis DNS internal Docker (http://backend:3000), di mana nama service backend otomatis di-resolve ke alamat IP seluruh replika container yang aktif.

