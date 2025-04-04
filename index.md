Oke, mari kita bangun situs download game ini langkah demi langkah. Saya akan menggunakan PHP, PostgreSQL, Bootstrap 5 untuk CSS, dan sedikit JavaScript vanilla untuk interaktivitas.

**Struktur Folder Proyek:**

```
game-download-site/
├── admin/
│   ├── index.php           # Dashboard Admin (login redirect/game list)
│   ├── login.php           # Halaman Login Admin
│   ├── logout.php          # Proses Logout Admin
│   ├── manage_games.php    # Halaman Tambah/Edit/Hapus Game
│   ├── process_game.php    # Skrip PHP untuk proses tambah/edit/hapus game
│   ├── view_feedback.php   # Halaman lihat semua masukan
│   └── auth_check.php      # Script untuk cek apakah admin sudah login
├── css/
│   └── style.css           # CSS Kustom
├── js/
│   └── script.js           # JavaScript Kustom
├── includes/
│   ├── db.php              # Koneksi Database
│   ├── header.php          # Bagian Header HTML (termasuk nav)
│   ├── footer.php          # Bagian Footer HTML
│   └── functions.php       # Fungsi-fungsi bantuan (misal: sanitasi)
├── uploads/                # Folder untuk menyimpan gambar cover game (pastikan writable by web server)
├── templates/              # (Opsional) bisa berisi file template HTML
├── .htaccess               # (Opsional, untuk URL rewriting & keamanan)
├── config.php.example      # Contoh file konfigurasi (rename ke config.php)
├── index.php               # Halaman Utama (List Game)
├── about.php               # Halaman Tentang Kami
├── feedback.php            # Halaman Masukan
└── game_detail.php         # (Fitur Tambahan) Halaman Detail Game
```

**Langkah 1: Database Setup (PostgreSQL)**

Buat database (misalnya `game_db`) dan jalankan SQL berikut untuk membuat tabel:

```sql
-- Tabel untuk Admin
CREATE TABLE admins (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tabel untuk Game
CREATE TABLE games (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    size VARCHAR(50),
    genre VARCHAR(100),
    description TEXT,
    image_filename VARCHAR(255), -- Nama file gambar yang diupload
    download_link TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Fungsi untuk update kolom updated_at secara otomatis (PostgreSQL specific)
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
   NEW.updated_at = NOW();
   RETURN NEW;
END;
$$ language 'plpgsql';

-- Trigger untuk memanggil fungsi di atas saat tabel games diupdate
CREATE TRIGGER update_games_updated_at
BEFORE UPDATE ON games
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();


-- Tabel untuk Masukan (Feedback)
CREATE TABLE feedback (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    message TEXT NOT NULL,
    submitted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- (Opsional) Tabel untuk komentar per game
CREATE TABLE game_comments (
    id SERIAL PRIMARY KEY,
    game_id INTEGER NOT NULL REFERENCES games(id) ON DELETE CASCADE, -- Jika game dihapus, komentar ikut terhapus
    user_name VARCHAR(100) NOT NULL,
    comment TEXT NOT NULL,
    rating SMALLINT CHECK (rating >= 1 AND rating <= 5), -- Rating 1-5
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert admin default (Ganti 'passwordkuat' dengan hash password yang aman!)
-- Cara generate hash di PHP: echo password_hash('passwordkuat', PASSWORD_DEFAULT);
INSERT INTO admins (username, password_hash) VALUES ('admin', '$2y$10$YOUR_GENERATED_PASSWORD_HASH_HERE'); -- GANTI INI!
```

**Penting:**
*   Ganti `$2y$10$YOUR_GENERATED_PASSWORD_HASH_HERE` dengan hash password yang sebenarnya. Jalankan `php -r "echo password_hash('password_admin_anda', PASSWORD_DEFAULT);"` di terminal Anda untuk membuatnya.
*   Pastikan folder `uploads/` dapat ditulis oleh web server (misalnya, `chmod 775 uploads` dan atur owner/group yang sesuai).

**Langkah 2: File Konfigurasi (`config.php`)**

Buat file `config.php` dari `config.php.example` dan isi detail koneksi database Anda:

```php
<?php
// Aktifkan display error saat development, matikan di production
ini_set('display_errors', 1); // Set 0 di production
error_reporting(E_ALL);

// Database Credentials (Ganti sesuai setting Anda)
define('DB_HOST', 'localhost'); // atau IP/hostname server PostgreSQL Anda
define('DB_PORT', '5432');      // Port default PostgreSQL
define('DB_NAME', 'game_db');   // Nama database Anda
define('DB_USER', 'your_db_user'); // Username database Anda
define('DB_PASS', 'your_db_password'); // Password database Anda

// Base URL (jika perlu, sesuaikan jika situs tidak di root)
// define('BASE_URL', 'http://localhost/game-download-site/');

// Konstanta lain jika diperlukan
define('ITEMS_PER_PAGE', 10); // Untuk pagination
define('UPLOAD_DIR', __DIR__ . '/uploads/'); // Path absolut ke folder uploads
define('MAX_FILE_SIZE', 5 * 1024 * 1024); // Batas ukuran file upload (contoh: 5MB)
define('ALLOWED_MIME_TYPES', ['image/jpeg', 'image/png', 'image/gif', 'image/webp']);

// Mulai session di semua halaman
if (session_status() === PHP_SESSION_NONE) {
    session_start();
}

// Generate CSRF token jika belum ada
if (empty($_SESSION['csrf_token'])) {
    $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
}
?>
```

**Langkah 3: Koneksi Database (`includes/db.php`)**

```php
<?php
require_once __DIR__ . '/../config.php'; // Sesuaikan path jika perlu

$dsn = "pgsql:host=" . DB_HOST . ";port=" . DB_PORT . ";dbname=" . DB_NAME;
$options = [
    PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION, // Lemparkan Exception jika ada error
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,       // Hasil query sebagai array assosiatif
    PDO::ATTR_EMULATE_PREPARES   => false,                  // Gunakan native prepared statements
];

try {
    $pdo = new PDO($dsn, DB_USER, DB_PASS, $options);
} catch (\PDOException $e) {
    // Di production, log error ini, jangan tampilkan ke user
    error_log("Database Connection Error: " . $e->getMessage());
    // Tampilkan pesan error generik ke user atau halaman error kustom
    die("Terjadi masalah saat menghubungkan ke database. Silakan coba lagi nanti.");
    // exit; // atau redirect ke halaman error
}
?>
```

**Langkah 4: Header dan Footer (`includes/header.php`, `includes/footer.php`)**

**`includes/header.php`:**

```php
<?php require_once __DIR__ . '/../config.php'; ?>
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="description" content="Situs Download Game Terbaru dan Terlengkap.">
    <!-- Tambahkan meta tag SEO lainnya jika perlu -->
    <title><?php echo isset($pageTitle) ? htmlspecialchars($pageTitle) : 'Game Downloader'; ?></title>

    <!-- Bootstrap CSS -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-T3c6CoIi6uLrA9TneNEoa7RxnatzjcDSCmG1MXxSR1GAsXEV/Dwwykc2MPK8M2HN" crossorigin="anonymous">

    <!-- Font Awesome -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.1/css/all.min.css" integrity="sha512-DTOQO9RWCH3ppGqcWaEA1BIZOC6xxalwEsw9c2QQeAIftl+Vegovlnee1c9QX4TctnWMn13TZye+giMm8e2LwA==" crossorigin="anonymous" referrerpolicy="no-referrer" />

    <!-- Custom CSS -->
    <link rel="stylesheet" href="/css/style.css?v=<?php echo time(); // Cache busting ?>">
    <!-- Ganti "/css/style.css" dengan path yang benar jika perlu -->

</head>
<body class="d-flex flex-column min-vh-100 bg-light">

<nav class="navbar navbar-expand-lg navbar-dark bg-primary shadow-sm sticky-top">
  <div class="container">
    <a class="navbar-brand fw-bold" href="/"> <i class="fas fa-gamepad"></i> GameZone </a>
    <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarNav" aria-controls="navbarNav" aria-expanded="false" aria-label="Toggle navigation">
      <span class="navbar-toggler-icon"></span>
    </button>
    <div class="collapse navbar-collapse" id="navbarNav">
      <ul class="navbar-nav ms-auto mb-2 mb-lg-0">
        <li class="nav-item">
          <a class="nav-link <?php echo basename($_SERVER['PHP_SELF']) == 'index.php' ? 'active' : ''; ?>" aria-current="page" href="/"> <i class="fas fa-home"></i> Beranda</a>
        </li>
        <li class="nav-item">
          <a class="nav-link <?php echo basename($_SERVER['PHP_SELF']) == 'about.php' ? 'active' : ''; ?>" href="/about.php"> <i class="fas fa-info-circle"></i> Tentang</a>
        </li>
        <li class="nav-item">
          <a class="nav-link <?php echo basename($_SERVER['PHP_SELF']) == 'feedback.php' ? 'active' : ''; ?>" href="/feedback.php"> <i class="fas fa-comments"></i> Masukan</a>
        </li>
         <?php if (isset($_SESSION['admin_logged_in']) && $_SESSION['admin_logged_in'] === true): ?>
          <li class="nav-item dropdown">
            <a class="nav-link dropdown-toggle" href="#" role="button" data-bs-toggle="dropdown" aria-expanded="false">
              <i class="fas fa-user-shield"></i> Admin Menu
            </a>
            <ul class="dropdown-menu dropdown-menu-end">
              <li><a class="dropdown-item" href="/admin/index.php"><i class="fas fa-tachometer-alt fa-fw me-2"></i>Dashboard</a></li>
              <li><a class="dropdown-item" href="/admin/manage_games.php"><i class="fas fa-plus-circle fa-fw me-2"></i>Tambah Game</a></li>
               <li><a class="dropdown-item" href="/admin/view_feedback.php"><i class="fas fa-envelope-open-text fa-fw me-2"></i>Lihat Masukan</a></li>
              <li><hr class="dropdown-divider"></li>
              <li><a class="dropdown-item" href="/admin/logout.php"><i class="fas fa-sign-out-alt fa-fw me-2"></i>Logout</a></li>
            </ul>
          </li>
         <?php else: ?>
           <!-- Mungkin link login admin di sini atau di footer -->
           <!-- <li class="nav-item"><a class="nav-link" href="/admin/login.php">Admin Login</a></li> -->
         <?php endif; ?>
      </ul>
       <!-- Fitur Pencarian -->
       <form class="d-flex ms-lg-3" role="search" action="/index.php" method="GET">
          <input class="form-control me-2 form-control-sm" type="search" name="search" placeholder="Cari game..." aria-label="Search" value="<?php echo isset($_GET['search']) ? htmlspecialchars($_GET['search']) : ''; ?>">
          <button class="btn btn-outline-light btn-sm" type="submit"><i class="fas fa-search"></i></button>
        </form>
    </div>
  </div>
</nav>

<main class="container mt-4 flex-grow-1">
```

**`includes/footer.php`:**

```php
</main> <!-- End main container -->

<footer class="footer mt-auto py-3 bg-dark text-white text-center">
    <div class="container">
        <span>&copy; <?php echo date('Y'); ?> GameZone - Situs Download Game Terpercaya.</span>
        <br>
        <small><a href="/admin/login.php" class="text-secondary">Admin Login</a></small>
    </div>
</footer>

<!-- Bootstrap Bundle with Popper -->
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/js/bootstrap.bundle.min.js" integrity="sha384-C6RzsynM9kWDrMNeT87bh95OGNyZPhcTNXj1NW7RuBCsyN/o0jlpcV8Qyq46cDfL" crossorigin="anonymous"></script>

<!-- Custom JS -->
<script src="/js/script.js?v=<?php echo time(); // Cache busting ?>"></script>
<!-- Ganti "/js/script.js" dengan path yang benar jika perlu -->

</body>
</html>
```

**Langkah 5: File CSS dan JS Kustom**

**`css/style.css`:**

```css
/* Ganti warna utama Bootstrap jika perlu */
:root {
    /* --bs-primary: #your-color; */
    /* --bs-secondary: #your-color; */
}

body {
    background-color: #f8f9fa; /* Warna background default yang sedikit abu-abu */
}

.navbar {
    /* Contoh customisasi navbar */
    /* background: linear-gradient(90deg, rgba(2,0,36,1) 0%, rgba(9,9,121,1) 35%, rgba(0,212,255,1) 100%); */
}

.game-card {
    transition: transform .2s ease-in-out, box-shadow .2s ease-in-out;
    border: none; /* Hilangkan border default card */
    overflow: hidden; /* Pastikan gambar tidak keluar card */
}

.game-card:hover {
    transform: translateY(-5px);
    box-shadow: 0 8px 20px rgba(0,0,0,.15);
}

.game-card img {
    aspect-ratio: 16 / 9; /* Jaga rasio gambar cover */
    object-fit: cover; /* Pastikan gambar mengisi area tanpa distorsi */
    transition: transform .3s ease;
}

.game-card:hover img {
     transform: scale(1.05); /* Efek zoom kecil pada hover */
}


.card-body {
    position: relative; /* Untuk positioning absolut elemen di dalamnya jika perlu */
}

.description-truncate {
    overflow: hidden;
    /* max-height: 3.6em; /* Sekitar 2 baris teks */
    /* line-height: 1.8em; */
    /* position: relative; */
    display: -webkit-box;
    -webkit-line-clamp: 2; /* Batasi ke 2 baris untuk browser WebKit */
    -webkit-box-orient: vertical;
    text-overflow: ellipsis;
}

/* .description-truncate.expanded { */
    /* max-height: none; */
/*    -webkit-line-clamp: unset; */
/* } */

/* Tampilan untuk read more link yang lebih baik */
/* .read-more-link { */
    /* position: absolute; */
    /* bottom: 10px; */
    /* right: 15px; */
    /* background: white; */
/* } */

.badge-genre {
    font-size: 0.8em;
    margin-right: 5px;
}

.footer {
    font-size: 0.9em;
}

/* Style untuk halaman admin */
.admin-section {
    background-color: #fff;
    padding: 2rem;
    border-radius: 0.5rem;
    box-shadow: 0 4px 10px rgba(0,0,0,.1);
}

/* Style untuk feedback list */
.feedback-item {
    border-left: 4px solid #0d6efd; /* Garis biru di kiri */
    margin-bottom: 1rem;
    padding: 1rem;
    background-color: #f8f9fa;
    border-radius: 0.25rem;
}

.feedback-item .name {
    font-weight: bold;
    color: #0d6efd;
}

.feedback-item .date {
    font-size: 0.85em;
    color: #6c757d;
}
```

**`js/script.js`:**

```javascript
document.addEventListener('DOMContentLoaded', function() {

    // 1. Read More / Read Less functionality (Versi Sederhana)
    // Kita gunakan data attribute untuk toggle kelas
    const readMoreLinks = document.querySelectorAll('.read-more-toggle');
    readMoreLinks.forEach(link => {
        link.addEventListener('click', function(event) {
            event.preventDefault();
            const targetId = this.getAttribute('data-target');
            const descriptionElement = document.getElementById(targetId);
            const dotsElement = descriptionElement.querySelector('.dots');
            const moreTextElement = descriptionElement.querySelector('.more-text');

            if (descriptionElement.classList.contains('expanded')) {
                // Collapse
                descriptionElement.classList.remove('expanded');
                if (dotsElement) dotsElement.style.display = 'inline';
                if (moreTextElement) moreTextElement.style.display = 'none';
                this.textContent = 'Lihat Banyak';
            } else {
                // Expand
                descriptionElement.classList.add('expanded');
                 if (dotsElement) dotsElement.style.display = 'none';
                 if (moreTextElement) moreTextElement.style.display = 'inline';
                this.textContent = 'Lihat Sedikit';
            }
        });
    });

    // 2. Delete Confirmation (Untuk Admin)
    const deleteButtons = document.querySelectorAll('.delete-game-btn');
    deleteButtons.forEach(button => {
        button.addEventListener('click', function(event) {
            event.preventDefault(); // Prevent default link behavior
            const gameName = this.getAttribute('data-game-name');
            const deleteUrl = this.href; // Get the URL from the link

            // Gunakan Modal Bootstrap untuk konfirmasi yang lebih baik
            const confirmationModal = new bootstrap.Modal(document.getElementById('deleteConfirmationModal'), {
                keyboard: false
            });

            // Set nama game di modal
            const modalGameNameElement = document.getElementById('modalGameName');
            if (modalGameNameElement) {
                modalGameNameElement.textContent = gameName;
            }

            // Set action untuk tombol Hapus di modal
            const confirmDeleteButton = document.getElementById('confirmDeleteButton');
            if (confirmDeleteButton) {
                // Hapus event listener lama jika ada untuk mencegah duplikasi
                const newConfirmButton = confirmDeleteButton.cloneNode(true);
                confirmDeleteButton.parentNode.replaceChild(newConfirmButton, confirmDeleteButton);

                // Tambahkan event listener baru
                newConfirmButton.addEventListener('click', () => {
                    // Redirect atau submit form untuk menghapus
                    // Dalam contoh ini, kita redirect ke URL hapus
                     window.location.href = deleteUrl;

                    // Atau jika menggunakan form POST:
                    // const deleteForm = document.getElementById('deleteForm'); // Asumsikan ada form
                    // deleteForm.action = deleteUrl;
                    // deleteForm.submit();
                });
            }

            confirmationModal.show();

            // Konfirmasi bawaan browser (Alternatif sederhana jika tidak pakai modal)
            // if (confirm(`Apakah Anda yakin ingin menghapus game "${gameName}"?`)) {
            //     window.location.href = deleteUrl; // Redirect to delete script
            // }
        });
    });

    // 3. Preview Gambar saat Upload (Untuk Admin)
    const imageInput = document.getElementById('gameImage');
    const imagePreview = document.getElementById('imagePreview');
    const currentImage = document.getElementById('currentImage'); // Gambar saat ini (untuk edit)

    if (imageInput && imagePreview) {
        imageInput.addEventListener('change', function(event) {
            const file = event.target.files[0];
            if (file) {
                const reader = new FileReader();
                reader.onload = function(e) {
                    // Sembunyikan gambar saat ini jika ada (saat edit)
                    if (currentImage) {
                        currentImage.style.display = 'none';
                    }
                    // Tampilkan preview
                    imagePreview.innerHTML = `<img src="${e.target.result}" class="img-thumbnail" style="max-height: 150px;" alt="Preview">`;
                    imagePreview.style.display = 'block';
                }
                reader.readAsDataURL(file);
            } else {
                 // Jika tidak ada file dipilih, tampilkan lagi gambar saat ini (jika edit) atau kosongkan preview
                 if (currentImage) {
                     currentImage.style.display = 'block';
                     imagePreview.style.display = 'none';
                     imagePreview.innerHTML = '';
                 } else {
                    imagePreview.style.display = 'none';
                    imagePreview.innerHTML = '';
                 }

            }
        });
    }


    // 4. Client-side form validation example (Bootstrap handles a lot of this)
    // Example: Check if feedback message is not empty (though 'required' attribute is better)
    const feedbackForm = document.getElementById('feedbackForm');
    if (feedbackForm) {
        feedbackForm.addEventListener('submit', function(event) {
            const nameInput = document.getElementById('name');
            const messageInput = document.getElementById('message');
            let isValid = true;

            // Clear previous errors (simple example)
            nameInput.classList.remove('is-invalid');
            messageInput.classList.remove('is-invalid');

            if (nameInput.value.trim() === '') {
                nameInput.classList.add('is-invalid');
                isValid = false;
            }
            if (messageInput.value.trim() === '') {
                 messageInput.classList.add('is-invalid');
                isValid = false;
            }

            if (!isValid) {
                event.preventDefault(); // Prevent form submission if invalid
                // You could display a more prominent error message here
            }
            // Server-side validation is still crucial!
        });
    }


});
```

**Langkah 6: Fungsi Bantuan (`includes/functions.php`)**

```php
<?php
require_once __DIR__ . '/../config.php';

/**
 * Membersihkan output untuk mencegah XSS.
 * Selalu gunakan ini saat menampilkan data dari database atau input user ke HTML.
 */
function e(string|null $string): string {
    return htmlspecialchars((string)$string, ENT_QUOTES, 'UTF-8');
}

/**
 * Memverifikasi CSRF token.
 */
function verifyCsrfToken(string $submittedToken): bool {
    return isset($_SESSION['csrf_token']) && hash_equals($_SESSION['csrf_token'], $submittedToken);
}

/**
 * Memotong teks dan menambahkan elipsis (...).
 */
 function truncateText(string $text, int $maxLength = 100): string {
    if (mb_strlen($text) > $maxLength) {
        $text = mb_substr($text, 0, $maxLength) . '...';
    }
    return $text;
}


/**
 * Membuat deskripsi dengan fitur "lihat banyak/sedikit".
 * Menggunakan ID unik untuk setiap deskripsi.
 */
function renderExpandableDescription(string $text, string $uniqueId, int $truncateLength = 150): string {
    $shortText = truncateText($text, $truncateLength);
    $fullText = e($text); // Sanitasi teks lengkap

    // Cek apakah teks memang perlu dipotong
    if (mb_strlen($text) <= $truncateLength) {
        return '<p>' . e($text) . '</p>'; // Tampilkan teks lengkap jika pendek
    }

    // Versi Sederhana dengan JS Toggle Class (Lihat script.js)
    $visiblePart = e(mb_substr($text, 0, $truncateLength));
    $hiddenPart = e(mb_substr($text, $truncateLength));

    // Pastikan ID unik
    $descId = 'desc-' . $uniqueId;

    return <<<HTML
    <div class="description-container" id="{$descId}">
       <p class="mb-0">
            {$visiblePart}<span class="dots">...</span><span class="more-text" style="display:none;">{$hiddenPart}</span>
       </p>
        <a href="#" class="read-more-toggle small text-decoration-none" data-target="{$descId}">Lihat Banyak</a>
    </div>
HTML;


    /* Versi Lama dengan ID dan JS terpisah (kurang ideal)
    $id = 'desc-' . $uniqueId;
    $linkId = 'link-' . $uniqueId;

    $output = '<div id="' . $id . '" class="description-truncate">';
    $output .= e($text);
    $output .= '</div>';
    if (strlen($text) > $truncateLength) { // Hanya tampilkan link jika teks memang panjang
         $output .= '<a href="#" id="' . $linkId . '" class="read-more-link small" onclick="toggleDescription(\'' . $id . '\', this); return false;">Lihat Banyak</a>';
    }
    return $output;
    */
}


/**
 * Upload gambar game.
 * Mengembalikan nama file yang diupload atau false jika gagal.
 */
function uploadGameImage(array $fileInput): string|false {
    if (!isset($fileInput['error']) || is_array($fileInput['error'])) {
        $_SESSION['flash_message'] = ['type' => 'danger', 'message' => 'Parameter upload file tidak valid.'];
        return false;
    }

    switch ($fileInput['error']) {
        case UPLOAD_ERR_OK:
            break;
        case UPLOAD_ERR_NO_FILE:
            // Tidak ada file diupload, mungkin saat edit tidak ganti gambar
            return null; // Kembalikan null untuk indikasi tidak ada file baru
        case UPLOAD_ERR_INI_SIZE:
        case UPLOAD_ERR_FORM_SIZE:
            $_SESSION['flash_message'] = ['type' => 'danger', 'message' => 'Ukuran file melebihi batas maksimal (' . (MAX_FILE_SIZE / 1024 / 1024) . ' MB).'];
            return false;
        default:
             $_SESSION['flash_message'] = ['type' => 'danger', 'message' => 'Terjadi kesalahan saat upload file (Kode: ' . $fileInput['error'] . ').'];
            return false;
    }

    // Cek ukuran file
    if ($fileInput['size'] > MAX_FILE_SIZE) {
        $_SESSION['flash_message'] = ['type' => 'danger', 'message' => 'Ukuran file melebihi batas maksimal (' . (MAX_FILE_SIZE / 1024 / 1024) . ' MB).'];
        return false;
    }

    // Cek tipe MIME
    $finfo = new finfo(FILEINFO_MIME_TYPE);
    $mimeType = $finfo->file($fileInput['tmp_name']);
    if (false === array_search($mimeType, ALLOWED_MIME_TYPES, true)) {
         $_SESSION['flash_message'] = ['type' => 'danger', 'message' 'Format file tidak diizinkan. Hanya ' . implode(', ', ALLOWED_MIME_TYPES) . ' yang diperbolehkan.'];
        return false;
    }

    // Buat nama file unik untuk menghindari konflik
    $extension = pathinfo($fileInput['name'], PATHINFO_EXTENSION);
    $safeFilename = preg_replace('/[^a-zA-Z0-9_-]/', '_', pathinfo($fileInput['name'], PATHINFO_FILENAME)); // Sanitasi nama file
    $newFilename = $safeFilename . '_' . uniqid() . '.' . $extension;
    $destination = UPLOAD_DIR . $newFilename;

    // Pindahkan file yang diupload
    if (!move_uploaded_file($fileInput['tmp_name'], $destination)) {
        $_SESSION['flash_message'] = ['type' => 'danger', 'message' => 'Gagal memindahkan file yang diupload.'];
        error_log("Gagal memindahkan file upload ke: " . $destination); // Log error
        return false;
    }

    return $newFilename; // Kembalikan nama file baru jika sukses
}

/**
 * Menampilkan pesan flash (notifikasi sementara).
 */
function displayFlashMessage(): void {
    if (isset($_SESSION['flash_message'])) {
        $type = $_SESSION['flash_message']['type'] ?? 'info'; // Default 'info'
        $message = $_SESSION['flash_message']['message'] ?? '';
        // Pastikan type adalah salah satu dari tipe alert Bootstrap
        $validTypes = ['primary', 'secondary', 'success', 'danger', 'warning', 'info', 'light', 'dark'];
        if (!in_array($type, $validTypes)) {
            $type = 'info'; // Fallback ke info jika tipe tidak valid
        }
        echo '<div class="alert alert-' . $type . ' alert-dismissible fade show" role="alert">';
        echo e($message); // Sanitasi pesan
        echo '<button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>';
        echo '</div>';
        unset($_SESSION['flash_message']); // Hapus pesan setelah ditampilkan
    }
}

/**
 * Generate pagination links.
 */
function generatePagination(int $currentPage, int $totalPages, string $baseUrl = '?'): string {
    if ($totalPages <= 1) {
        return '';
    }

    $paginationHtml = '<nav aria-label="Page navigation"><ul class="pagination justify-content-center">';

    // Tombol Previous
    $prevDisabled = ($currentPage <= 1) ? ' disabled' : '';
    $prevPage = $currentPage - 1;
    $paginationHtml .= '<li class="page-item' . $prevDisabled . '">';
    $paginationHtml .= '<a class="page-link" href="' . $baseUrl . 'page=' . $prevPage . '" aria-label="Previous"><span aria-hidden="true">&laquo;</span></a>';
    $paginationHtml .= '</li>';

    // Tombol Halaman
    // Logic untuk menampilkan ... jika halaman terlalu banyak (opsional)
    $startPage = max(1, $currentPage - 2);
    $endPage = min($totalPages, $currentPage + 2);

    if ($startPage > 1) {
         $paginationHtml .= '<li class="page-item"><a class="page-link" href="' . $baseUrl . 'page=1">1</a></li>';
         if ($startPage > 2) {
              $paginationHtml .= '<li class="page-item disabled"><span class="page-link">...</span></li>';
         }
    }


    for ($i = $startPage; $i <= $endPage; $i++) {
        $active = ($i == $currentPage) ? ' active' : '';
        $paginationHtml .= '<li class="page-item' . $active . '"><a class="page-link" href="' . $baseUrl . 'page=' . $i . '">' . $i . '</a></li>';
    }

     if ($endPage < $totalPages) {
          if ($endPage < $totalPages - 1) {
               $paginationHtml .= '<li class="page-item disabled"><span class="page-link">...</span></li>';
          }
         $paginationHtml .= '<li class="page-item"><a class="page-link" href="' . $baseUrl . 'page=' . $totalPages . '">' . $totalPages . '</a></li>';
    }


    // Tombol Next
    $nextDisabled = ($currentPage >= $totalPages) ? ' disabled' : '';
    $nextPage = $currentPage + 1;
    $paginationHtml .= '<li class="page-item' . $nextDisabled . '">';
    $paginationHtml .= '<a class="page-link" href="' . $baseUrl . 'page=' . $nextPage . '" aria-label="Next"><span aria-hidden="true">&raquo;</span></a>';
    $paginationHtml .= '</li>';

    $paginationHtml .= '</ul></nav>';
    return $paginationHtml;
}

?>
```

**Langkah 7: Halaman Utama (`index.php`)**

```php
<?php
require_once __DIR__ . '/includes/db.php';
require_once __DIR__ . '/includes/functions.php';

$pageTitle = "Beranda - Download Game";

// Pagination Logic
$page = isset($_GET['page']) && is_numeric($_GET['page']) ? (int)$_GET['page'] : 1;
$limit = ITEMS_PER_PAGE;
$offset = ($page - 1) * $limit;

// Search and Filter Logic
$searchQuery = isset($_GET['search']) ? trim($_GET['search']) : '';
$genreFilter = isset($_GET['genre']) ? trim($_GET['genre']) : '';

$sqlBase = "FROM games WHERE 1=1"; // Start with a condition that's always true
$params = [];
$baseUrlParams = []; // For pagination links

if (!empty($searchQuery)) {
    $sqlBase .= " AND (name ILIKE :search OR description ILIKE :search)"; // ILIKE for case-insensitive search in PostgreSQL
    $params[':search'] = '%' . $searchQuery . '%';
    $baseUrlParams['search'] = $searchQuery;
}

if (!empty($genreFilter)) {
    $sqlBase .= " AND genre = :genre";
    $params[':genre'] = $genreFilter;
     $baseUrlParams['genre'] = $genreFilter;
}

// Count total items for pagination
$sqlCount = "SELECT COUNT(*) " . $sqlBase;
$stmtCount = $pdo->prepare($sqlCount);
$stmtCount->execute($params);
$totalItems = $stmtCount->fetchColumn();
$totalPages = ceil($totalItems / $limit);

// Fetch games for the current page
$sql = "SELECT id, name, size, genre, description, image_filename " . $sqlBase . " ORDER BY created_at DESC LIMIT :limit OFFSET :offset";
$stmt = $pdo->prepare($sql);

// Bind named parameters for main query
foreach ($params as $key => $val) {
    $stmt->bindValue($key, $val);
}
// Bind limit and offset separately (need to specify type for LIMIT/OFFSET in PDO)
$stmt->bindValue(':limit', $limit, PDO::PARAM_INT);
$stmt->bindValue(':offset', $offset, PDO::PARAM_INT);

$stmt->execute();
$games = $stmt->fetchAll();

// --- Fetch Genres for Filtering ---
$genreStmt = $pdo->query("SELECT DISTINCT genre FROM games WHERE genre IS NOT NULL AND genre != '' ORDER BY genre");
$genres = $genreStmt->fetchAll(PDO::FETCH_COLUMN);
// ---

// --- Build base URL for pagination ---
$paginationBaseUrl = '/index.php?' . http_build_query($baseUrlParams) . '&';
// ---

include __DIR__ . '/includes/header.php';
?>

<div class="row mb-3">
    <div class="col-md-8">
        <h1>Daftar Game Terbaru</h1>
        <?php if (!empty($searchQuery)): ?>
            <p class="lead">Hasil pencarian untuk: "<?php echo e($searchQuery); ?>"</p>
        <?php endif; ?>
         <?php if (!empty($genreFilter)): ?>
            <p class="lead">Menampilkan game dengan genre: "<?php echo e($genreFilter); ?>"</p>
        <?php endif; ?>
    </div>
     <div class="col-md-4 align-self-center">
         <!-- Filter by Genre Dropdown -->
         <?php if (!empty($genres)): ?>
         <div class="dropdown float-md-end">
           <button class="btn btn-secondary dropdown-toggle btn-sm" type="button" id="genreFilterDropdown" data-bs-toggle="dropdown" aria-expanded="false">
             <i class="fas fa-filter"></i> Filter Genre
           </button>
           <ul class="dropdown-menu" aria-labelledby="genreFilterDropdown">
             <li><a class="dropdown-item <?php echo empty($genreFilter) ? 'active' : ''; ?>" href="/index.php?<?php echo http_build_query(['search' => $searchQuery]); ?>">Semua Genre</a></li>
             <?php foreach ($genres as $genre): ?>
                <li><a class="dropdown-item <?php echo ($genreFilter == $genre) ? 'active' : ''; ?>" href="/index.php?<?php echo http_build_query(['search' => $searchQuery, 'genre' => $genre]); ?>"><?php echo e($genre); ?></a></li>
             <?php endforeach; ?>
           </ul>
         </div>
         <?php endif; ?>
     </div>
</div>

<?php displayFlashMessage(); // Tampilkan notifikasi jika ada ?>

<div class="row row-cols-1 row-cols-md-2 row-cols-lg-3 g-4">
    <?php if (count($games) > 0): ?>
        <?php foreach ($games as $game): ?>
            <div class="col">
                <div class="card h-100 game-card shadow-sm">
                    <?php
                        $imagePath = '/uploads/' . ($game['image_filename'] ?? 'default.jpg'); // Ganti default.jpg jika perlu
                        // Cek jika file gambar ada, jika tidak tampilkan placeholder
                        if (empty($game['image_filename']) || !file_exists(UPLOAD_DIR . $game['image_filename'])) {
                             $imagePath = 'https://via.placeholder.com/400x225.png?text=No+Image'; // Placeholder image
                        }
                    ?>
                    <img src="<?php echo e($imagePath); ?>" class="card-img-top" alt="<?php echo e($game['name']); ?>">
                    <div class="card-body d-flex flex-column">
                        <h5 class="card-title"><?php echo e($game['name']); ?></h5>
                        <div class="mb-2">
                            <?php if (!empty($game['genre'])): ?>
                                <span class="badge bg-primary badge-genre"><?php echo e($game['genre']); ?></span>
                            <?php endif; ?>
                            <?php if (!empty($game['size'])): ?>
                                <span class="badge bg-secondary badge-genre"><i class="fas fa-hdd"></i> <?php echo e($game['size']); ?></span>
                            <?php endif; ?>
                        </div>
                        <?php echo renderExpandableDescription($game['description'] ?? 'Tidak ada deskripsi.', $game['id']); ?>

                        <div class="mt-auto pt-2"> <!-- mt-auto pushes button to bottom -->
                             <!-- Link ke Halaman Detail (Fitur Tambahan) -->
                             <a href="/game_detail.php?id=<?php echo $game['id']; ?>" class="btn btn-sm btn-outline-info me-1"><i class="fas fa-eye"></i> Detail</a>
                            <!-- Tombol Download (Harusnya ke halaman detail atau langsung link) -->
                            <?php if (!empty($game['download_link'])): // Tampilkan hanya jika ada link ?>
                               <a href="<?php echo e($game['download_link']); // Nanti bisa dibuat halaman perantara dulu ?>" class="btn btn-sm btn-success" target="_blank" rel="noopener noreferrer"><i class="fas fa-download"></i> Download</a>
                            <?php else: ?>
                                <button class="btn btn-sm btn-secondary" disabled><i class="fas fa-download"></i> Download</button>
                            <?php endif; ?>
                        </div>
                    </div>
                </div>
            </div>
        <?php endforeach; ?>
    <?php else: ?>
        <div class="col-12">
            <div class="alert alert-warning text-center" role="alert">
                Tidak ada game yang ditemukan.
                 <?php if(!empty($searchQuery) || !empty($genreFilter)): ?>
                    <a href="/" class="alert-link">Tampilkan Semua Game</a>
                 <?php endif; ?>
            </div>
        </div>
    <?php endif; ?>
</div>

<!-- Pagination -->
<div class="mt-5">
    <?php echo generatePagination($page, $totalPages, $paginationBaseUrl); ?>
</div>


<?php include __DIR__ . '/includes/footer.php'; ?>
```

**Langkah 8: Halaman Tentang (`about.php`)**

```php
<?php
$pageTitle = "Tentang Kami";
include __DIR__ . '/includes/header.php';
?>

<h1>Tentang GameZone</h1>
<hr>

<p class="lead">Selamat datang di GameZone!</p>

<p>GameZone adalah portal terdepan Anda untuk menemukan dan mengunduh game-game terbaru dan terpopuler. Kami berdedikasi untuk menyediakan koleksi game yang beragam, mulai dari game indie hingga judul-judul AAA, untuk berbagai platform.</p>

<p>Misi kami adalah:</p>
<ul>
    <li>Menyediakan akses mudah dan cepat ke link download game yang aman dan terverifikasi.</li>
    <li>Menjadi sumber informasi terpercaya mengenai game, termasuk deskripsi, genre, dan ukuran file.</li>
    <li>Membangun komunitas gamer yang positif dan suportif.</li>
</ul>

<p>Situs ini dikelola oleh tim yang bersemangat tentang dunia game dan berkomitmen untuk memberikan pengalaman terbaik bagi para pengguna. Kami terus berusaha memperbarui koleksi game kami dan meningkatkan fitur situs.</p>

<p>Terima kasih telah mengunjungi GameZone. Selamat bermain!</p>

<?php include __DIR__ . '/includes/footer.php'; ?>
```

**Langkah 9: Halaman Masukan (`feedback.php`)**

```php
<?php
require_once __DIR__ . '/includes/db.php';
require_once __DIR__ . '/includes/functions.php';

$pageTitle = "Masukan Pengguna";
$errors = [];
$successMessage = '';
$submittedData = ['name' => '', 'message' => '']; // Untuk pre-fill form jika error

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // 1. Verify CSRF Token
    if (!isset($_POST['csrf_token']) || !verifyCsrfToken($_POST['csrf_token'])) {
        $errors[] = 'Sesi tidak valid atau telah kedaluwarsa. Silakan coba lagi.';
         // Regenerate token? Tergantung kebijakan keamanan
         // $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
    } else {
        // 2. Validate Input
        $name = trim($_POST['name'] ?? '');
        $message = trim($_POST['message'] ?? '');
        $submittedData = ['name' => $name, 'message' => $message]; // Simpan input

        if (empty($name)) {
            $errors['name'] = 'Nama tidak boleh kosong.';
        } elseif (strlen($name) > 100) {
             $errors['name'] = 'Nama terlalu panjang (maksimal 100 karakter).';
        }

        if (empty($message)) {
            $errors['message'] = 'Masukan tidak boleh kosong.';
        }

        // 3. Process if valid
        if (empty($errors)) {
            try {
                $sql = "INSERT INTO feedback (name, message) VALUES (:name, :message)";
                $stmt = $pdo->prepare($sql);
                $stmt->bindParam(':name', $name, PDO::PARAM_STR);
                $stmt->bindParam(':message', $message, PDO::PARAM_STR);
                $stmt->execute();

                $_SESSION['flash_message'] = ['type' => 'success', 'message' => 'Terima kasih atas masukan Anda!'];
                // Redirect untuk mencegah resubmission form (Post/Redirect/Get pattern)
                header("Location: feedback.php");
                exit;

            } catch (PDOException $e) {
                error_log("Error inserting feedback: " . $e->getMessage());
                 $errors['db'] = 'Terjadi kesalahan saat menyimpan masukan Anda. Silakan coba lagi.';
            }
        }
         // Jika ada error validasi atau DB, $errors tidak kosong
         if(!empty($errors)){
             $_SESSION['flash_message'] = ['type' => 'danger', 'message' => 'Gagal mengirim masukan. Periksa error di bawah.'];
         }
    }

}

include __DIR__ . '/includes/header.php';
?>

<h1>Berikan Masukan Anda</h1>
<p class="lead">Kami menghargai pendapat Anda untuk membantu kami meningkatkan situs ini.</p>
<hr>

<?php displayFlashMessage(); // Tampilkan pesan sukses/error dari session ?>

<?php if (!empty($errors) && isset($errors['db'])): ?>
    <div class="alert alert-danger"><?php echo e($errors['db']); ?></div>
<?php endif; ?>
 <?php if (!empty($errors) && isset($errors[0]) && strpos($errors[0], 'Sesi tidak valid') !== false): ?>
     <div class="alert alert-danger"><?php echo e($errors[0]); ?></div>
 <?php endif; ?>


<div class="card shadow-sm">
    <div class="card-body">
        <form action="feedback.php" method="POST" id="feedbackForm" novalidate>
            <input type="hidden" name="csrf_token" value="<?php echo e($_SESSION['csrf_token']); ?>">

            <div class="mb-3">
                <label for="name" class="form-label">Nama Anda <span class="text-danger">*</span></label>
                <input type="text" class="form-control <?php echo isset($errors['name']) ? 'is-invalid' : ''; ?>" id="name" name="name" required maxlength="100" value="<?php echo e($submittedData['name']); ?>">
                <?php if (isset($errors['name'])): ?>
                    <div class="invalid-feedback"><?php echo e($errors['name']); ?></div>
                <?php endif; ?>
            </div>

            <div class="mb-3">
                <label for="message" class="form-label">Masukan Anda <span class="text-danger">*</span></label>
                <textarea class="form-control <?php echo isset($errors['message']) ? 'is-invalid' : ''; ?>" id="message" name="message" rows="5" required><?php echo e($submittedData['message']); ?></textarea>
                 <?php if (isset($errors['message'])): ?>
                    <div class="invalid-feedback"><?php echo e($errors['message']); ?></div>
                <?php endif; ?>
            </div>

            <button type="submit" class="btn btn-primary"><i class="fas fa-paper-plane"></i> Kirim Masukan</button>
        </form>
    </div>
</div>

<?php include __DIR__ . '/includes/footer.php'; ?>
```

**Langkah 10: Area Admin (Folder `admin/`)**

Ini adalah bagian yang paling kompleks.

**`admin/auth_check.php`:** (File ini disertakan di awal setiap halaman admin)

```php
<?php
// Pastikan session dimulai (biasanya sudah di config.php)
if (session_status() === PHP_SESSION_NONE) {
    session_start();
}

// Jika admin belum login, redirect ke halaman login
if (!isset($_SESSION['admin_logged_in']) || $_SESSION['admin_logged_in'] !== true) {
    // Simpan URL yang diminta agar bisa redirect kembali setelah login (opsional)
    $_SESSION['redirect_url'] = $_SERVER['REQUEST_URI'];
    header('Location: /admin/login.php'); // Sesuaikan path jika perlu
    exit;
}

// (Opsional) Tambahkan pengecekan inactivity timeout di sini
// if (isset($_SESSION['last_activity']) && (time() - $_SESSION['last_activity'] > 1800)) { // contoh 30 menit
//     session_unset();
//     session_destroy();
//     header('Location: /admin/login.php?status=session_expired');
//     exit;
// }
// $_SESSION['last_activity'] = time(); // Update last activity time
?>
```

**`admin/login.php`:**

```php
<?php
// Jangan include auth_check.php di sini!
require_once __DIR__ . '/../includes/db.php'; // Koneksi DB
require_once __DIR__ . '/../includes/functions.php'; // Fungsi helper

// Jika sudah login, redirect ke dashboard
if (isset($_SESSION['admin_logged_in']) && $_SESSION['admin_logged_in'] === true) {
    header('Location: /admin/index.php'); // Sesuaikan path jika perlu
    exit;
}

$pageTitle = "Admin Login";
$error = '';
$username = '';

// --- Rate Limiting Sederhana (Contoh) ---
$maxLoginAttempts = 5;
$lockoutTime = 300; // 5 menit

if (!isset($_SESSION['login_attempts'])) {
    $_SESSION['login_attempts'] = 0;
    $_SESSION['login_locked_until'] = 0;
}

$isLocked = ($_SESSION['login_locked_until'] > time());

if ($_SERVER['REQUEST_METHOD'] === 'POST' && !$isLocked) {
    // Verify CSRF Token
    if (!isset($_POST['csrf_token']) || !verifyCsrfToken($_POST['csrf_token'])) {
        $error = 'Sesi tidak valid atau telah kedaluwarsa. Silakan muat ulang halaman dan coba lagi.';
         // Regenerate token?
         $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
    } else {
        $username = trim($_POST['username'] ?? '');
        $password = $_POST['password'] ?? '';

        if (empty($username) || empty($password)) {
            $error = 'Username dan password tidak boleh kosong.';
            $_SESSION['login_attempts']++;
        } else {
            try {
                $sql = "SELECT id, username, password_hash FROM admins WHERE username = :username";
                $stmt = $pdo->prepare($sql);
                $stmt->bindParam(':username', $username, PDO::PARAM_STR);
                $stmt->execute();
                $admin = $stmt->fetch();

                if ($admin && password_verify($password, $admin['password_hash'])) {
                    // Login Berhasil
                    session_regenerate_id(true); // Cegah session fixation
                    $_SESSION['admin_logged_in'] = true;
                    $_SESSION['admin_id'] = $admin['id'];
                    $_SESSION['admin_username'] = $admin['username'];
                    // $_SESSION['last_activity'] = time(); // Set initial activity time

                    // Reset rate limiting
                    unset($_SESSION['login_attempts']);
                    unset($_SESSION['login_locked_until']);

                    // Redirect ke halaman tujuan atau dashboard
                    $redirectUrl = $_SESSION['redirect_url'] ?? '/admin/index.php';
                    unset($_SESSION['redirect_url']); // Hapus dari session
                    header('Location: ' . $redirectUrl);
                    exit;
                } else {
                    // Login Gagal
                    $error = 'Username atau password salah.';
                    $_SESSION['login_attempts']++;

                     // Implementasi Lockout
                     if ($_SESSION['login_attempts'] >= $maxLoginAttempts) {
                         $_SESSION['login_locked_until'] = time() + $lockoutTime;
                         $error .= ' Anda telah mencoba login terlalu banyak. Akun terkunci selama ' . ($lockoutTime / 60) . ' menit.';
                         // Log kejadian ini
                         error_log("Login lockout triggered for username: " . $username . " from IP: " . $_SERVER['REMOTE_ADDR']);
                     }
                }
            } catch (PDOException $e) {
                 error_log("Admin login error: " . $e->getMessage());
                 $error = 'Terjadi kesalahan pada server. Silakan coba lagi nanti.';
            }
        }
    }
} elseif ($isLocked) {
     $remainingTime = ceil(($_SESSION['login_locked_until'] - time()) / 60);
     $error = "Terlalu banyak percobaan login. Silakan tunggu sekitar {$remainingTime} menit lagi.";
}

// --- Generate CSRF token jika belum ada (seharusnya sudah di config.php) ---
if (empty($_SESSION['csrf_token'])) {
    $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
}
// ---

include __DIR__ . '/../includes/header.php'; // Gunakan header standar
?>

<div class="row justify-content-center mt-5">
    <div class="col-md-6 col-lg-4">
        <div class="card shadow-lg">
             <div class="card-header bg-primary text-white text-center">
                <h3 class="mb-0"><i class="fas fa-user-shield"></i> Admin Login</h3>
            </div>
            <div class="card-body p-4">
                <?php if ($error): ?>
                    <div class="alert alert-danger" role="alert">
                        <?php echo e($error); ?>
                    </div>
                <?php endif; ?>
                 <?php if (isset($_GET['status']) && $_GET['status'] == 'session_expired'): ?>
                     <div class="alert alert-warning" role="alert">
                        Sesi Anda telah berakhir karena tidak aktif. Silakan login kembali.
                    </div>
                 <?php endif; ?>

                <form action="login.php" method="POST" novalidate>
                    <input type="hidden" name="csrf_token" value="<?php echo e($_SESSION['csrf_token']); ?>">
                    <div class="mb-3">
                        <label for="username" class="form-label">Username</label>
                        <div class="input-group">
                           <span class="input-group-text"><i class="fas fa-user"></i></span>
                           <input type="text" class="form-control <?php echo ($error && !empty($username)) ? 'is-invalid' : ''; ?>" id="username" name="username" required value="<?php echo e($username); ?>">
                        </div>
                    </div>
                    <div class="mb-3">
                        <label for="password" class="form-label">Password</label>
                         <div class="input-group">
                             <span class="input-group-text"><i class="fas fa-lock"></i></span>
                            <input type="password" class="form-control <?php echo $error ? 'is-invalid' : ''; ?>" id="password" name="password" required>
                         </div>
                    </div>
                    <div class="d-grid">
                        <button type="submit" class="btn btn-primary" <?php echo $isLocked ? 'disabled' : ''; ?>>
                           <i class="fas fa-sign-in-alt"></i> Login
                        </button>
                    </div>
                </form>
            </div>
            <div class="card-footer text-center text-muted small">
                Lupa password? Hubungi administrator.
            </div>
        </div>
         <div class="text-center mt-3">
             <a href="/" class="text-secondary small"><i class="fas fa-arrow-left"></i> Kembali ke Beranda</a>
        </div>
    </div>
</div>

<?php include __DIR__ . '/../includes/footer.php'; ?>
```

**`admin/index.php`:** (Dashboard sederhana, bisa menampilkan list game juga)

```php
<?php
require_once __DIR__ . '/auth_check.php'; // Cek login dulu
require_once __DIR__ . '/../includes/db.php';
require_once __DIR__ . '/../includes/functions.php';

$pageTitle = "Admin Dashboard";

// --- Ambil data ringkasan (Contoh) ---
try {
    $totalGames = $pdo->query("SELECT COUNT(*) FROM games")->fetchColumn();
    $totalFeedback = $pdo->query("SELECT COUNT(*) FROM feedback")->fetchColumn();
} catch (PDOException $e) {
     error_log("Dashboard count error: " . $e->getMessage());
     $totalGames = 'N/A';
     $totalFeedback = 'N/A';
     $_SESSION['flash_message'] = ['type' => 'warning', 'message' => 'Gagal mengambil data ringkasan.'];
}
// ---

// --- Ambil beberapa game terbaru (Sama seperti index.php tapi versi admin) ---
$limit = 5; // Tampilkan 5 game terbaru di dashboard
try {
    $stmt = $pdo->query("SELECT id, name, genre, created_at, image_filename FROM games ORDER BY created_at DESC LIMIT $limit");
    $recentGames = $stmt->fetchAll();
} catch (PDOException $e) {
     error_log("Dashboard recent games error: " . $e->getMessage());
     $recentGames = [];
     $_SESSION['flash_message'] = ['type' => 'warning', 'message' => 'Gagal mengambil daftar game terbaru.'];
}
// ---

include __DIR__ . '/../includes/header.php';
?>

<h1><i class="fas fa-tachometer-alt"></i> Admin Dashboard</h1>
<p class="lead">Selamat datang, <?php echo e($_SESSION['admin_username'] ?? 'Admin'); ?>!</p>
<hr>

<?php displayFlashMessage(); ?>

<!-- Ringkasan Statistik -->
<div class="row mb-4">
    <div class="col-md-6 col-xl-4 mb-3">
        <div class="card text-white bg-primary h-100">
            <div class="card-body">
                <div class="d-flex justify-content-between align-items-center">
                    <div>
                        <h5 class="card-title mb-0">Total Game</h5>
                        <p class="fs-3 fw-bold mb-0"><?php echo e($totalGames); ?></p>
                    </div>
                    <i class="fas fa-gamepad fa-3x opacity-75"></i>
                </div>
            </div>
             <a href="/admin/manage_games.php" class="card-footer text-white text-decoration-none">
                Kelola Game <i class="fas fa-arrow-circle-right"></i>
            </a>
        </div>
    </div>
     <div class="col-md-6 col-xl-4 mb-3">
        <div class="card text-white bg-success h-100">
            <div class="card-body">
                 <div class="d-flex justify-content-between align-items-center">
                    <div>
                        <h5 class="card-title mb-0">Total Masukan</h5>
                        <p class="fs-3 fw-bold mb-0"><?php echo e($totalFeedback); ?></p>
                    </div>
                     <i class="fas fa-comments fa-3x opacity-75"></i>
                </div>
            </div>
             <a href="/admin/view_feedback.php" class="card-footer text-white text-decoration-none">
                Lihat Masukan <i class="fas fa-arrow-circle-right"></i>
            </a>
        </div>
    </div>
     <!-- Tambahkan card statistik lain jika perlu -->
</div>

<!-- Game Terbaru -->
<div class="card admin-section">
    <div class="card-header">
        <h5 class="mb-0"><i class="fas fa-history"></i> Game yang Baru Ditambahkan</h5>
    </div>
    <div class="card-body p-0">
        <?php if (!empty($recentGames)): ?>
        <div class="table-responsive">
            <table class="table table-striped table-hover mb-0">
                <thead>
                    <tr>
                        <th>Cover</th>
                        <th>Nama Game</th>
                        <th>Genre</th>
                        <th>Ditambahkan</th>
                        <th>Aksi</th>
                    </tr>
                </thead>
                <tbody>
                    <?php foreach ($recentGames as $game): ?>
                    <tr>
                         <td>
                             <?php
                                $imgPath = '/uploads/' . ($game['image_filename'] ?? 'default.jpg');
                                if (empty($game['image_filename']) || !file_exists(UPLOAD_DIR . $game['image_filename'])) {
                                     $imgPath = 'https://via.placeholder.com/60x34.png?text=N/A'; // Placeholder kecil
                                }
                             ?>
                             <img src="<?php echo e($imgPath); ?>" alt="Cover" style="width: 60px; height: auto; object-fit: cover; border-radius: 3px;">
                         </td>
                        <td><?php echo e($game['name']); ?></td>
                        <td><?php echo e($game['genre'] ?? '-'); ?></td>
                        <td><?php echo date('d M Y, H:i', strtotime($game['created_at'])); ?></td>
                        <td>
                           <a href="/admin/manage_games.php?action=edit&id=<?php echo $game['id']; ?>" class="btn btn-sm btn-outline-warning" title="Edit"><i class="fas fa-edit"></i></a>
                            <!-- Tombol Hapus dengan Konfirmasi JS -->
                           <a href="/admin/process_game.php?action=delete&id=<?php echo $game['id']; ?>&csrf_token=<?php echo e($_SESSION['csrf_token']); ?>"
                              class="btn btn-sm btn-outline-danger delete-game-btn"
                              data-game-name="<?php echo e($game['name']); ?>"
                              title="Hapus">
                               <i class="fas fa-trash-alt"></i>
                           </a>
                        </td>
                    </tr>
                    <?php endforeach; ?>
                </tbody>
            </table>
        </div>
         <div class="card-footer text-center">
             <a href="/admin/manage_games.php">Lihat Semua Game &raquo;</a>
         </div>
        <?php else: ?>
        <div class="alert alert-info mb-0 text-center">Belum ada game yang ditambahkan.</div>
        <?php endif; ?>
    </div>
</div>

<!-- Modal Konfirmasi Hapus (Dibutuhkan oleh script.js) -->
<div class="modal fade" id="deleteConfirmationModal" tabindex="-1" aria-labelledby="deleteConfirmationModalLabel" aria-hidden="true">
  <div class="modal-dialog">
    <div class="modal-content">
      <div class="modal-header bg-danger text-white">
        <h5 class="modal-title" id="deleteConfirmationModalLabel"><i class="fas fa-exclamation-triangle"></i> Konfirmasi Hapus</h5>
        <button type="button" class="btn-close btn-close-white" data-bs-dismiss="modal" aria-label="Close"></button>
      </div>
      <div class="modal-body">
        Apakah Anda yakin ingin menghapus game: <strong id="modalGameName"></strong>? Tindakan ini tidak dapat dibatalkan.
      </div>
      <div class="modal-footer">
        <button type="button" class="btn btn-secondary" data-bs-dismiss="modal"><i class="fas fa-times"></i> Batal</button>
        <button type="button" class="btn btn-danger" id="confirmDeleteButton"><i class="fas fa-trash-alt"></i> Ya, Hapus</button>
      </div>
    </div>
  </div>
</div>


<?php include __DIR__ . '/../includes/footer.php'; ?>
```

**`admin/manage_games.php`:** (Halaman untuk Tambah/Edit dan List Semua Game)

```php
<?php
require_once __DIR__ . '/auth_check.php';
require_once __DIR__ . '/../includes/db.php';
require_once __DIR__ . '/../includes/functions.php';

$action = $_GET['action'] ?? 'list'; // Aksi: list, add, edit
$pageTitle = "Kelola Game";
$gameData = [ // Data default untuk form tambah
    'id' => null,
    'name' => '',
    'size' => '',
    'genre' => '',
    'description' => '',
    'image_filename' => null,
    'download_link' => ''
];
$formActionUrl = '/admin/process_game.php?action=add';
$formTitle = "Tambah Game Baru";
$submitButtonText = '<i class="fas fa-plus-circle"></i> Tambah Game';

// Jika aksi adalah edit, ambil data game
if ($action === 'edit' && isset($_GET['id'])) {
    $gameId = filter_input(INPUT_GET, 'id', FILTER_VALIDATE_INT);
    if ($gameId) {
        try {
            $stmt = $pdo->prepare("SELECT * FROM games WHERE id = :id");
            $stmt->bindParam(':id', $gameId, PDO::PARAM_INT);
            $stmt->execute();
            $gameData = $stmt->fetch();

            if ($gameData) {
                $pageTitle = "Edit Game: " . e($gameData['name']);
                $formActionUrl = '/admin/process_game.php?action=edit&id=' . $gameId;
                $formTitle = "Edit Game: " . e($gameData['name']);
                $submitButtonText = '<i class="fas fa-save"></i> Simpan Perubahan';
            } else {
                 $_SESSION['flash_message'] = ['type' => 'warning', 'message' => 'Game tidak ditemukan.'];
                 header('Location: /admin/manage_games.php'); // Kembali ke list
                 exit;
            }
        } catch (PDOException $e) {
            error_log("Error fetching game for edit: " . $e->getMessage());
            $_SESSION['flash_message'] = ['type' => 'danger', 'message' => 'Gagal mengambil data game.'];
            header('Location: /admin/manage_games.php');
            exit;
        }
    } else {
         $_SESSION['flash_message'] = ['type' => 'warning', 'message' => 'ID game tidak valid.'];
         header('Location: /admin/manage_games.php');
         exit;
    }
} elseif ($action === 'add') {
    $pageTitle = "Tambah Game Baru";
    // Jika ada data form sebelumnya karena error validasi (bisa disimpan di session)
    if (isset($_SESSION['form_data'])) {
        $gameData = array_merge($gameData, $_SESSION['form_data']);
        unset($_SESSION['form_data']);
    }
     if (isset($_SESSION['form_errors'])) {
        $errors = $_SESSION['form_errors']; // Ambil error dari session
        unset($_SESSION['form_errors']);
    }
}

// --- Logic untuk List Game (jika action=list atau default) ---
$games = [];
$totalPages = 1;
$currentPage = 1;
$paginationHtml = '';

if ($action === 'list') {
    $pageTitle = "Daftar Semua Game";
    $currentPage = isset($_GET['page']) && is_numeric($_GET['page']) ? (int)$_GET['page'] : 1;
    $limit = ITEMS_PER_PAGE;
    $offset = ($currentPage - 1) * $limit;

     // Search Logic (opsional di halaman ini)
    $searchQuery = isset($_GET['search']) ? trim($_GET['search']) : '';
    $sqlBase = "FROM games WHERE 1=1";
    $params = [];
    $baseUrlParams = [];

    if (!empty($searchQuery)) {
        $sqlBase .= " AND name ILIKE :search";
        $params[':search'] = '%' . $searchQuery . '%';
         $baseUrlParams['search'] = $searchQuery;
    }

    try {
        // Count total items
        $sqlCount = "SELECT COUNT(*) " . $sqlBase;
        $stmtCount = $pdo->prepare($sqlCount);
        $stmtCount->execute($params);
        $totalItems = $stmtCount->fetchColumn();
        $totalPages = ceil($totalItems / $limit);

        // Fetch games for current page
        $sql = "SELECT id, name, genre, size, created_at, image_filename " . $sqlBase . " ORDER BY created_at DESC LIMIT :limit OFFSET :offset";
        $stmt = $pdo->prepare($sql);

         // Bind search params if any
        foreach ($params as $key => $val) {
            $stmt->bindValue($key, $val);
        }
        $stmt->bindValue(':limit', $limit, PDO::PARAM_INT);
        $stmt->bindValue(':offset', $offset, PDO::PARAM_INT);

        $stmt->execute();
        $games = $stmt->fetchAll();

        // Generate pagination links
        $paginationBaseUrl = '/admin/manage_games.php?' . http_build_query($baseUrlParams) . '&';
        $paginationHtml = generatePagination($currentPage, $totalPages, $paginationBaseUrl);

    } catch (PDOException $e) {
        error_log("Error fetching game list for admin: " . $e->getMessage());
        $_SESSION['flash_message'] = ['type' => 'danger', 'message' => 'Gagal mengambil daftar game.'];
        $games = []; // Kosongkan list jika error
    }
}
// --- End List Game Logic ---

include __DIR__ . '/../includes/header.php';
?>

<h1><?php echo e($pageTitle); ?></h1>
<hr>

<?php displayFlashMessage(); ?>

<?php if ($action === 'add' || $action === 'edit'): ?>
    <!-- Tampilkan Form Tambah/Edit -->
    <div class="card admin-section mb-4">
         <div class="card-header">
             <h5 class="mb-0"><?php echo $formTitle; ?></h5>
         </div>
        <div class="card-body">
            <form action="<?php echo $formActionUrl; ?>" method="POST" enctype="multipart/form-data" novalidate>
                <input type="hidden" name="csrf_token" value="<?php echo e($_SESSION['csrf_token']); ?>">

                <div class="row">
                    <div class="col-md-6 mb-3">
                        <label for="gameName" class="form-label">Nama Game <span class="text-danger">*</span></label>
                        <input type="text" class="form-control <?php echo isset($errors['name']) ? 'is-invalid' : ''; ?>" id="gameName" name="name" required value="<?php echo e($gameData['name'] ?? ''); ?>">
                         <?php if (isset($errors['name'])): ?><div class="invalid-feedback"><?php echo e($errors['name']); ?></div><?php endif; ?>
                    </div>
                    <div class="col-md-3 mb-3">
                        <label for="gameSize" class="form-label">Ukuran Game</label>
                        <input type="text" class="form-control <?php echo isset($errors['size']) ? 'is-invalid' : ''; ?>" id="gameSize" name="size" placeholder="Contoh: 1.5 GB / 500 MB" value="<?php echo e($gameData['size'] ?? ''); ?>">
                         <?php if (isset($errors['size'])): ?><div class="invalid-feedback"><?php echo e($errors['size']); ?></div><?php endif; ?>
                    </div>
                    <div class="col-md-3 mb-3">
                        <label for="gameGenre" class="form-label">Genre</label>
                        <input type="text" class="form-control <?php echo isset($errors['genre']) ? 'is-invalid' : ''; ?>" id="gameGenre" name="genre" placeholder="Contoh: Action, RPG" value="<?php echo e($gameData['genre'] ?? ''); ?>">
                         <?php if (isset($errors['genre'])): ?><div class="invalid-feedback"><?php echo e($errors['genre']); ?></div><?php endif; ?>
                    </div>
                </div>

                <div class="mb-3">
                    <label for="gameDescription" class="form-label">Deskripsi Game</label>
                    <textarea class="form-control <?php echo isset($errors['description']) ? 'is-invalid' : ''; ?>" id="gameDescription" name="description" rows="4"><?php echo e($gameData['description'] ?? ''); ?></textarea>
                     <?php if (isset($errors['description'])): ?><div class="invalid-feedback"><?php echo e($errors['description']); ?></div><?php endif; ?>
                </div>

                <div class="row">
                     <div class="col-md-6 mb-3">
                         <label for="gameImage" class="form-label">Cover Gambar <?php echo ($action === 'edit') ? '(Kosongkan jika tidak ingin ganti)' : '<span class="text-danger">*</span>'; ?></label>
                         <input class="form-control <?php echo isset($errors['image']) ? 'is-invalid' : ''; ?>" type="file" id="gameImage" name="image" accept="<?php echo implode(',', ALLOWED_MIME_TYPES); // Ambil dari config ?>">
                          <?php if (isset($errors['image'])): ?><div class="invalid-feedback"><?php echo e($errors['image']); ?></div><?php endif; ?>
                         <!-- Image Preview Area -->
                         <div id="imagePreview" class="mt-2" style="display: none;"></div>
                         <!-- Current Image (for edit) -->
                         <?php if ($action === 'edit' && !empty($gameData['image_filename']) && file_exists(UPLOAD_DIR . $gameData['image_filename'])): ?>
                             <div id="currentImage" class="mt-2">
                                 <p class="small mb-1">Gambar saat ini:</p>
                                 <img src="/uploads/<?php echo e($gameData['image_filename']); ?>" alt="Current Cover" class="img-thumbnail" style="max-height: 100px;">
                                  <input type="hidden" name="current_image" value="<?php echo e($gameData['image_filename']); ?>">
                             </div>
                         <?php endif; ?>
                     </div>
                     <div class="col-md-6 mb-3">
                        <label for="gameDownloadLink" class="form-label">Link Download <span class="text-danger">*</span></label>
                        <input type="url" class="form-control <?php echo isset($errors['download_link']) ? 'is-invalid' : ''; ?>" id="gameDownloadLink" name="download_link" required placeholder="https://example.com/download" value="<?php echo e($gameData['download_link'] ?? ''); ?>">
                         <?php if (isset($errors['download_link'])): ?><div class="invalid-feedback"><?php echo e($errors['download_link']); ?></div><?php endif; ?>
                    </div>
                </div>

                <hr>
                <button type="submit" class="btn btn-primary"><?php echo $submitButtonText; ?></button>
                <a href="/admin/manage_games.php" class="btn btn-secondary">Batal</a>
            </form>
        </div>
    </div>
<?php endif; ?>


<?php if ($action === 'list'): ?>
    <!-- Tampilkan List Game -->
     <div class="d-flex justify-content-between align-items-center mb-3">
        <a href="/admin/manage_games.php?action=add" class="btn btn-success"><i class="fas fa-plus"></i> Tambah Game Baru</a>
        <!-- Form Pencarian (Opsional) -->
        <form action="/admin/manage_games.php" method="GET" class="d-flex">
             <input type="hidden" name="action" value="list">
             <input class="form-control form-control-sm me-2" type="search" name="search" placeholder="Cari nama game..." aria-label="Search" value="<?php echo e($searchQuery); ?>">
             <button class="btn btn-outline-secondary btn-sm" type="submit"><i class="fas fa-search"></i></button>
              <?php if (!empty($searchQuery)): ?>
                 <a href="/admin/manage_games.php?action=list" class="btn btn-outline-danger btn-sm ms-2" title="Reset Pencarian"><i class="fas fa-times"></i></a>
             <?php endif; ?>
        </form>
    </div>

    <div class="card admin-section">
         <div class="card-header">
             <h5 class="mb-0"><i class="fas fa-list"></i> Daftar Game (<?php echo $totalItems ?? 0; ?>)</h5>
         </div>
        <div class="card-body p-0">
            <?php if (count($games) > 0): ?>
            <div class="table-responsive">
                <table class="table table-striped table-hover mb-0 align-middle">
                    <thead>
                        <tr>
                            <th style="width: 80px;">Cover</th>
                            <th>Nama Game</th>
                            <th>Genre</th>
                            <th>Ukuran</th>
                            <th>Ditambahkan</th>
                            <th style="width: 120px;">Aksi</th>
                        </tr>
                    </thead>
                    <tbody>
                        <?php foreach ($games as $game): ?>
                        <tr>
                             <td>
                                 <?php
                                    $imgPathList = '/uploads/' . ($game['image_filename'] ?? 'default.jpg');
                                    if (empty($game['image_filename']) || !file_exists(UPLOAD_DIR . $game['image_filename'])) {
                                        $imgPathList = 'https://via.placeholder.com/60x34.png?text=N/A';
                                    }
                                 ?>
                                 <img src="<?php echo e($imgPathList); ?>" alt="Cover" style="width: 60px; height: auto; object-fit: cover; border-radius: 3px;">
                             </td>
                            <td><?php echo e($game['name']); ?></td>
                            <td><?php echo e($game['genre'] ?? '-'); ?></td>
                            <td><?php echo e($game['size'] ?? '-'); ?></td>
                            <td><?php echo date('d M Y', strtotime($game['created_at'])); ?></td>
                            <td>
                               <a href="/admin/manage_games.php?action=edit&id=<?php echo $game['id']; ?>" class="btn btn-sm btn-warning me-1" title="Edit"><i class="fas fa-edit"></i></a>
                               <!-- Tombol Hapus dengan Konfirmasi JS -->
                               <a href="/admin/process_game.php?action=delete&id=<?php echo $game['id']; ?>&csrf_token=<?php echo e($_SESSION['csrf_token']); ?>"
                                  class="btn btn-sm btn-danger delete-game-btn"
                                  data-game-name="<?php echo e($game['name']); ?>"
                                  title="Hapus">
                                   <i class="fas fa-trash-alt"></i>
                               </a>
                            </td>
                        </tr>
                        <?php endforeach; ?>
                    </tbody>
                </table>
            </div>
             <?php if($totalPages > 1): ?>
             <div class="card-footer">
                 <?php echo $paginationHtml; ?>
             </div>
             <?php endif; ?>
            <?php else: ?>
            <div class="alert alert-info mb-0 text-center">
                Tidak ada game yang ditemukan.
                <?php if (!empty($searchQuery)): ?>
                    <a href="/admin/manage_games.php?action=list" class="alert-link">Reset Pencarian</a>
                <?php else: ?>
                    <a href="/admin/manage_games.php?action=add" class="alert-link">Tambahkan Game Baru</a>
                <?php endif; ?>
            </div>
            <?php endif; ?>
        </div>
    </div>

    <!-- Modal Konfirmasi Hapus (Wajib ada jika tombol hapus digunakan di halaman ini) -->
    <div class="modal fade" id="deleteConfirmationModal" tabindex="-1" aria-labelledby="deleteConfirmationModalLabel" aria-hidden="true">
      <div class="modal-dialog">
        <div class="modal-content">
          <div class="modal-header bg-danger text-white">
            <h5 class="modal-title" id="deleteConfirmationModalLabel"><i class="fas fa-exclamation-triangle"></i> Konfirmasi Hapus</h5>
            <button type="button" class="btn-close btn-close-white" data-bs-dismiss="modal" aria-label="Close"></button>
          </div>
          <div class="modal-body">
            Apakah Anda yakin ingin menghapus game: <strong id="modalGameName"></strong>? Tindakan ini tidak dapat dibatalkan dan akan menghapus gambar terkait.
          </div>
          <div class="modal-footer">
            <button type="button" class="btn btn-secondary" data-bs-dismiss="modal"><i class="fas fa-times"></i> Batal</button>
            <button type="button" class="btn btn-danger" id="confirmDeleteButton"><i class="fas fa-trash-alt"></i> Ya, Hapus</button>
          </div>
        </div>
      </div>
    </div>
<?php endif; ?>


<?php include __DIR__ . '/../includes/footer.php'; ?>
```

**`admin/process_game.php`:** (Skrip untuk memproses form Tambah/Edit/Hapus)

```php
<?php
require_once __DIR__ . '/auth_check.php'; // Autentikasi dulu
require_once __DIR__ . '/../includes/db.php';
require_once __DIR__ . '/../includes/functions.php';

// Cek Method & CSRF Token untuk semua aksi modifikasi data
if ($_SERVER['REQUEST_METHOD'] !== 'POST' && ($_GET['action'] ?? '') !== 'delete') { // Hapus bisa via GET+CSRF atau POST
     $_SESSION['flash_message'] = ['type' => 'danger', 'message' => 'Metode request tidak valid.'];
    header('Location: /admin/manage_games.php');
    exit;
}

// Verifikasi CSRF Token
$csrfToken = $_POST['csrf_token'] ?? $_GET['csrf_token'] ?? ''; // Ambil dari POST atau GET (untuk delete link)
if (!verifyCsrfToken($csrfToken)) {
    $_SESSION['flash_message'] = ['type' => 'danger', 'message' => 'Token CSRF tidak valid atau sesi telah berakhir.'];
    // Redirect ke halaman sebelumnya atau halaman manage
     $referer = $_SERVER['HTTP_REFERER'] ?? '/admin/manage_games.php';
     header('Location: ' . $referer);
    exit;
}

$action = $_GET['action'] ?? null;
$gameId = isset($_GET['id']) ? filter_input(INPUT_GET, 'id', FILTER_VALIDATE_INT) : null;

// --- Aksi Tambah Game ---
if ($action === 'add' && $_SERVER['REQUEST_METHOD'] === 'POST') {
    $errors = [];
    $name = trim($_POST['name'] ?? '');
    $size = trim($_POST['size'] ?? '');
    $genre = trim($_POST['genre'] ?? '');
    $description = trim($_POST['description'] ?? '');
    $downloadLink = trim($_POST['download_link'] ?? '');
    $imageFile = $_FILES['image'] ?? null;

    // Validasi Input
    if (empty($name)) $errors['name'] = 'Nama game tidak boleh kosong.';
    if (empty($downloadLink)) {
         $errors['download_link'] = 'Link download tidak boleh kosong.';
    } elseif (!filter_var($downloadLink, FILTER_VALIDATE_URL)) {
         $errors['download_link'] = 'Format link download tidak valid.';
    }
    if (strlen($name) > 255) $errors['name'] = 'Nama game terlalu panjang (max 255).';
    if (strlen($size) > 50) $errors['size'] = 'Ukuran game terlalu panjang (max 50).';
    if (strlen($genre) > 100) $errors['genre'] = 'Genre terlalu panjang (max 100).';

    // Validasi Upload Gambar (wajib saat add)
    $uploadedFilename = null;
    if (!isset($imageFile['error']) || $imageFile['error'] === UPLOAD_ERR_NO_FILE) {
        $errors['image'] = 'Cover gambar wajib diupload.';
    } else {
        $uploadResult = uploadGameImage($imageFile);
        if ($uploadResult === false) {
            // Pesan error sudah di set di fungsi uploadGameImage (via flash message)
            $errors['image'] = $_SESSION['flash_message']['message'] ?? 'Gagal mengupload gambar.'; // Ambil pesan errornya
        } elseif ($uploadResult !== null) {
            $uploadedFilename = $uploadResult;
        }
    }


    if (empty($errors)) {
        // Proses Insert ke Database
        try {
            $sql = "INSERT INTO games (name, size, genre, description, image_filename, download_link)
                    VALUES (:name, :size, :genre, :description, :image_filename, :download_link)";
            $stmt = $pdo->prepare($sql);
            $stmt->bindParam(':name', $name);
            $stmt->bindParam(':size', $size);
            $stmt->bindParam(':genre', $genre);
            $stmt->bindParam(':description', $description);
            $stmt->bindParam(':image_filename', $uploadedFilename);
            $stmt->bindParam(':download_link', $downloadLink);
            $stmt->execute();

            $_SESSION['flash_message'] = ['type' => 'success', 'message' => 'Game "' . e($name) . '" berhasil ditambahkan.'];
            header('Location: /admin/manage_games.php'); // Redirect ke list
            exit;

        } catch (PDOException $e) {
             error_log("Error adding game: " . $e->getMessage());
             $_SESSION['flash_message'] = ['type' => 'danger', 'message' => 'Gagal menambahkan game ke database.'];
             // Hapus file yang terlanjur diupload jika insert gagal
             if ($uploadedFilename && file_exists(UPLOAD_DIR . $uploadedFilename)) {
                 unlink(UPLOAD_DIR . $uploadedFilename);
             }
             // Simpan data form dan error untuk ditampilkan kembali
             $_SESSION['form_data'] = $_POST;
             $_SESSION['form_errors'] = ['db' => 'Gagal menyimpan ke database. Error: ' . $e->getCode()]; // Kode error bisa membantu debug
             header('Location: /admin/manage_games.php?action=add'); // Kembali ke form tambah
             exit;
        }
    } else {
        // Ada Error Validasi
        $_SESSION['flash_message'] = ['type' => 'danger', 'message' => 'Gagal menambahkan game. Periksa error pada form.'];
        // Hapus file yang mungkin terupload jika validasi gagal di bagian lain
         if ($uploadedFilename && file_exists(UPLOAD_DIR . $uploadedFilename)) {
             unlink(UPLOAD_DIR . $uploadedFilename);
         }
        $_SESSION['form_data'] = $_POST;
        $_SESSION['form_errors'] = $errors;
        header('Location: /admin/manage_games.php?action=add'); // Kembali ke form tambah
        exit;
    }
}

// --- Aksi Edit Game ---
elseif ($action === 'edit' && $_SERVER['REQUEST_METHOD'] === 'POST' && $gameId) {
    $errors = [];
    $name = trim($_POST['name'] ?? '');
    $size = trim($_POST['size'] ?? '');
    $genre = trim($_POST['genre'] ?? '');
    $description = trim($_POST['description'] ?? '');
    $downloadLink = trim($_POST['download_link'] ?? '');
    $currentImage = $_POST['current_image'] ?? null; // Ambil nama gambar lama
    $imageFile = $_FILES['image'] ?? null;

     // Validasi Input (Sama seperti add)
    if (empty($name)) $errors['name'] = 'Nama game tidak boleh kosong.';
    if (empty($downloadLink)) {
         $errors['download_link'] = 'Link download tidak boleh kosong.';
    } elseif (!filter_var($downloadLink, FILTER_VALIDATE_URL)) {
         $errors['download_link'] = 'Format link download tidak valid.';
    }
    // Tambahkan validasi panjang karakter jika perlu

    // Proses Upload Gambar (jika ada file baru)
    $newFilename = null;
    $deleteOldImage = false;
    if (isset($imageFile['error']) && $imageFile['error'] !== UPLOAD_ERR_NO_FILE) {
        $uploadResult = uploadGameImage($imageFile);
        if ($uploadResult === false) {
             $errors['image'] = $_SESSION['flash_message']['message'] ?? 'Gagal mengupload gambar baru.';
        } elseif ($uploadResult !== null) {
            $newFilename = $uploadResult;
            $deleteOldImage = true; // Tandai untuk hapus gambar lama jika upload baru sukses
        }
    }


    if (empty($errors)) {
         // Tentukan nama file gambar yang akan disimpan
         $imageToSave = $newFilename ?? $currentImage; // Gunakan baru jika ada, jika tidak pakai yg lama

        // Proses Update ke Database
        try {
            $sql = "UPDATE games SET
                        name = :name,
                        size = :size,
                        genre = :genre,
                        description = :description,
                        image_filename = :image_filename,
                        download_link = :download_link
                        -- updated_at akan diupdate otomatis oleh trigger
                    WHERE id = :id";
            $stmt = $pdo->prepare($sql);
            $stmt->bindParam(':name', $name);
            $stmt->bindParam(':size', $size);
            $stmt->bindParam(':genre', $genre);
            $stmt->bindParam(':description', $description);
            $stmt->bindParam(':image_filename', $imageToSave); // Simpan nama file yg relevan
            $stmt->bindParam(':download_link', $downloadLink);
            $stmt->bindParam(':id', $gameId, PDO::PARAM_INT);
            $stmt->execute();

            // Hapus gambar lama jika gambar baru berhasil diupload dan disimpan ke DB
            if ($deleteOldImage && $currentImage && $newFilename !== $currentImage) {
                 $oldImagePath = UPLOAD_DIR . $currentImage;
                 if (file_exists($oldImagePath)) {
                     @unlink($oldImagePath); // Gunakan @ untuk menekan warning jika file tidak ada/tidak bisa dihapus
                 }
            }


            $_SESSION['flash_message'] = ['type' => 'success', 'message' => 'Game "' . e($name) . '" berhasil diperbarui.'];
            header('Location: /admin/manage_games.php'); // Redirect ke list
            exit;

        } catch (PDOException $e) {
            error_log("Error updating game ID $gameId: " . $e->getMessage());
            $_SESSION['flash_message'] = ['type' => 'danger', 'message' => 'Gagal memperbarui game di database.'];
             // Hapus file baru yang terlanjur diupload jika update gagal
             if ($newFilename && file_exists(UPLOAD_DIR . $newFilename)) {
                 unlink(UPLOAD_DIR . $newFilename);
             }
             // Simpan data form dan error untuk ditampilkan kembali
             $_SESSION['form_data'] = $_POST;
             $_SESSION['form_errors'] = ['db' => 'Gagal menyimpan perubahan ke database.'];
             header('Location: /admin/manage_games.php?action=edit&id=' . $gameId); // Kembali ke form edit
             exit;
        }
    } else {
        // Ada Error Validasi
        $_SESSION['flash_message'] = ['type' => 'danger', 'message' => 'Gagal memperbarui game. Periksa error pada form.'];
         // Hapus file baru yg mungkin terupload jika validasi gagal
         if ($newFilename && file_exists(UPLOAD_DIR . $newFilename)) {
             unlink(UPLOAD_DIR . $newFilename);
         }
        $_SESSION['form_data'] = $_POST;
        $_SESSION['form_errors'] = $errors;
        header('Location: /admin/manage_games.php?action=edit&id=' . $gameId); // Kembali ke form edit
        exit;
    }
}

// --- Aksi Hapus Game ---
// Bisa via GET (dari link dengan konfirmasi JS) atau POST (dari form khusus)
elseif ($action === 'delete' && $gameId) {

     // Ambil nama file gambar sebelum menghapus record DB
     try {
         $stmt = $pdo->prepare("SELECT name, image_filename FROM games WHERE id = :id");
         $stmt->bindParam(':id', $gameId, PDO::PARAM_INT);
         $stmt->execute();
         $game = $stmt->fetch();

         if ($game) {
             $gameNameToDelete = $game['name'];
             $imageToDelete = $game['image_filename'];

             // Hapus record dari database
             $sql = "DELETE FROM games WHERE id = :id";
             $stmt = $pdo->prepare($sql);
             $stmt->bindParam(':id', $gameId, PDO::PARAM_INT);
             $stmt->execute();

             if ($stmt->rowCount() > 0) {
                 // Hapus file gambar terkait jika ada
                 if ($imageToDelete) {
                     $imagePath = UPLOAD_DIR . $imageToDelete;
                     if (file_exists($imagePath)) {
                         @unlink($imagePath);
                     }
                 }
                 $_SESSION['flash_message'] = ['type' => 'success', 'message' => 'Game "' . e($gameNameToDelete) . '" berhasil dihapus.'];
             } else {
                  $_SESSION['flash_message'] = ['type' => 'warning', 'message' => 'Game tidak ditemukan atau sudah dihapus.'];
             }

         } else {
              $_SESSION['flash_message'] = ['type' => 'warning', 'message' => 'Game dengan ID ' . $gameId . ' tidak ditemukan.'];
         }

     } catch (PDOException $e) {
         error_log("Error deleting game ID $gameId: " . $e->getMessage());
          $_SESSION['flash_message'] = ['type' => 'danger', 'message' => 'Gagal menghapus game. Terjadi kesalahan database.'];
          // Jika ada foreign key constraint (misal dari tabel komentar), penghapusan bisa gagal
          if (strpos($e->getMessage(), 'violates foreign key constraint') !== false) {
               $_SESSION['flash_message']['message'] .= ' Mungkin karena masih ada data terkait (seperti komentar).';
          }
     }

    header('Location: /admin/manage_games.php'); // Redirect kembali ke list game
    exit;
}

// --- Aksi Tidak Dikenal ---
else {
    $_SESSION['flash_message'] = ['type' => 'warning', 'message' => 'Aksi tidak valid atau parameter tidak lengkap.'];
    header('Location: /admin/manage_games.php');
    exit;
}
?>
```

**`admin/view_feedback.php`:**

```php
<?php
require_once __DIR__ . '/auth_check.php';
require_once __DIR__ . '/../includes/db.php';
require_once __DIR__ . '/../includes/functions.php';

$pageTitle = "Lihat Semua Masukan";

// Pagination Logic
$page = isset($_GET['page']) && is_numeric($_GET['page']) ? (int)$_GET['page'] : 1;
$limit = ITEMS_PER_PAGE; // Ambil dari config
$offset = ($page - 1) * $limit;

try {
    // Count total feedback
    $totalItems = $pdo->query("SELECT COUNT(*) FROM feedback")->fetchColumn();
    $totalPages = ceil($totalItems / $limit);

    // Fetch feedback for the current page
    $sql = "SELECT id, name, message, submitted_at FROM feedback ORDER BY submitted_at DESC LIMIT :limit OFFSET :offset";
    $stmt = $pdo->prepare($sql);
    $stmt->bindValue(':limit', $limit, PDO::PARAM_INT);
    $stmt->bindValue(':offset', $offset, PDO::PARAM_INT);
    $stmt->execute();
    $feedbacks = $stmt->fetchAll();

} catch (PDOException $e) {
    error_log("Error fetching feedback: " . $e->getMessage());
    $_SESSION['flash_message'] = ['type' => 'danger', 'message' => 'Gagal mengambil data masukan.'];
    $feedbacks = [];
    $totalPages = 1;
}

$paginationHtml = generatePagination($page, $totalPages, '/admin/view_feedback.php?');

include __DIR__ . '/../includes/header.php';
?>

<h1><i class="fas fa-envelope-open-text"></i> Semua Masukan Pengguna</h1>
<p class="lead">Berikut adalah daftar masukan yang dikirim oleh pengguna.</p>
<hr>

<?php displayFlashMessage(); ?>

<div class="admin-section">
    <?php if (count($feedbacks) > 0): ?>
        <?php foreach ($feedbacks as $feedback): ?>
            <div class="feedback-item mb-3">
                <div class="d-flex justify-content-between align-items-center mb-1">
                    <span class="name"><i class="fas fa-user"></i> <?php echo e($feedback['name']); ?></span>
                    <span class="date"><i class="far fa-clock"></i> <?php echo date('d M Y, H:i', strtotime($feedback['submitted_at'])); ?></span>
                </div>
                <div class="message bg-white p-2 border rounded">
                    <?php echo nl2br(e($feedback['message'])); // nl2br untuk menjaga format baris baru ?>
                </div>
                 <!-- Tombol Hapus Feedback (Opsional) -->
                 <!-- <div class="text-end mt-1">
                      <a href="/admin/process_feedback.php?action=delete&id=<?php echo $feedback['id']; ?>&csrf_token=<?php echo e($_SESSION['csrf_token']); ?>" class="btn btn-sm btn-outline-danger delete-feedback-btn" data-feedback-id="<?php echo $feedback['id']; ?>" title="Hapus Masukan"><i class="fas fa-trash-alt"></i> Hapus</a>
                 </div> -->
            </div>
        <?php endforeach; ?>

        <!-- Pagination -->
        <div class="mt-4">
            <?php echo $paginationHtml; ?>
        </div>

    <?php else: ?>
        <div class="alert alert-info text-center">Belum ada masukan yang diterima.</div>
    <?php endif; ?>
</div>


<!-- Tambahkan Modal Konfirmasi jika implementasi tombol hapus feedback -->
<!-- <div class="modal fade" id="deleteFeedbackModal" ... > ... </div> -->


<?php include __DIR__ . '/../includes/footer.php'; ?>
```

**`admin/logout.php`:**

```php
<?php
session_start(); // Mulai session untuk mengaksesnya

// Hapus semua variabel session
$_SESSION = array();

// Hancurkan session cookie jika ada
if (ini_get("session.use_cookies")) {
    $params = session_get_cookie_params();
    setcookie(session_name(), '', time() - 42000,
        $params["path"], $params["domain"],
        $params["secure"], $params["httponly"]
    );
}

// Hancurkan session
session_destroy();

// Redirect ke halaman login atau halaman utama
header("Location: /admin/login.php?status=logged_out"); // Atau ke '/'
exit;
?>
```

**Langkah 11: Fitur Tambahan (Contoh: Halaman Detail Game)**

**`game_detail.php`:**

```php
<?php
require_once __DIR__ . '/includes/db.php';
require_once __DIR__ . '/includes/functions.php';

$gameId = filter_input(INPUT_GET, 'id', FILTER_VALIDATE_INT);

if (!$gameId) {
    // Redirect ke halaman utama atau tampilkan 404
    header('Location: /');
    exit;
}

try {
    $stmt = $pdo->prepare("SELECT * FROM games WHERE id = :id");
    $stmt->bindParam(':id', $gameId, PDO::PARAM_INT);
    $stmt->execute();
    $game = $stmt->fetch();

    if (!$game) {
        // Game tidak ditemukan, tampilkan halaman 404 sederhana
         http_response_code(404);
         $pageTitle = "404 Game Tidak Ditemukan";
         include __DIR__ . '/includes/header.php';
         echo '<div class="alert alert-danger text-center"><h1>404</h1><p>Maaf, game yang Anda cari tidak ditemukan.</p><a href="/" class="btn btn-primary">Kembali ke Beranda</a></div>';
         include __DIR__ . '/includes/footer.php';
         exit;
    }

    $pageTitle = e($game['name']) . " - Detail Game";

     // (Opsional) Fetch Komentar untuk game ini
     $commentStmt = $pdo->prepare("SELECT user_name, comment, rating, created_at FROM game_comments WHERE game_id = :game_id ORDER BY created_at DESC");
     $commentStmt->bindParam(':game_id', $gameId, PDO::PARAM_INT);
     $commentStmt->execute();
     $comments = $commentStmt->fetchAll();

     // (Opsional) Hitung rata-rata rating
     $avgRatingStmt = $pdo->prepare("SELECT AVG(rating) as avg_rating, COUNT(rating) as rating_count FROM game_comments WHERE game_id = :game_id AND rating IS NOT NULL");
      $avgRatingStmt->bindParam(':game_id', $gameId, PDO::PARAM_INT);
      $avgRatingStmt->execute();
      $ratingInfo = $avgRatingStmt->fetch();
      $avgRating = $ratingInfo['avg_rating'] ? round($ratingInfo['avg_rating'], 1) : null;
      $ratingCount = $ratingInfo['rating_count'] ?? 0;


} catch (PDOException $e) {
     error_log("Error fetching game detail ID $gameId: " . $e->getMessage());
     // Tampilkan halaman error 500 sederhana
     http_response_code(500);
     $pageTitle = "Error Server";
     include __DIR__ . '/includes/header.php';
     echo '<div class="alert alert-danger text-center"><h1>500</h1><p>Terjadi kesalahan internal pada server. Silakan coba lagi nanti.</p></div>';
     include __DIR__ . '/includes/footer.php';
     exit;
}


include __DIR__ . '/includes/header.php';
?>

<div class="row">
    <!-- Kolom Detail Game -->
    <div class="col-lg-8">
        <div class="card shadow-sm mb-4">
             <?php
                $imagePathDetail = '/uploads/' . ($game['image_filename'] ?? 'default-large.jpg');
                if (empty($game['image_filename']) || !file_exists(UPLOAD_DIR . $game['image_filename'])) {
                     $imagePathDetail = 'https://via.placeholder.com/800x450.png?text=No+Image+Available'; // Placeholder lebih besar
                }
            ?>
            <img src="<?php echo e($imagePathDetail); ?>" class="card-img-top" alt="Cover <?php echo e($game['name']); ?>" style="aspect-ratio: 16/9; object-fit: cover;">
            <div class="card-body">
                <h1 class="card-title"><?php echo e($game['name']); ?></h1>

                 <!-- Info Singkat & Rating -->
                 <div class="d-flex flex-wrap align-items-center mb-3 text-muted small">
                     <?php if (!empty($game['genre'])): ?>
                        <span class="me-3"><i class="fas fa-tag"></i> <?php echo e($game['genre']); ?></span>
                     <?php endif; ?>
                      <?php if (!empty($game['size'])): ?>
                        <span class="me-3"><i class="fas fa-hdd"></i> <?php echo e($game['size']); ?></span>
                     <?php endif; ?>
                     <span class="me-3"><i class="far fa-calendar-alt"></i> Dirilis: <?php echo date('d M Y', strtotime($game['created_at'])); ?></span>
                     <?php if ($avgRating !== null): ?>
                       <span class="text-warning">
                           <i class="fas fa-star"></i> <?php echo e($avgRating); ?>/5
                           <span class="text-muted">(<?php echo e($ratingCount); ?> ulasan)</span>
                       </span>
                     <?php endif; ?>
                 </div>

                <h5 class="mt-4">Deskripsi</h5>
                <hr class="my-2">
                <p><?php echo nl2br(e($game['description'] ?? 'Deskripsi tidak tersedia.')); ?></p>


                 <!-- Tombol Download -->
                 <div class="mt-4 text-center">
                    <?php if (!empty($game['download_link'])): ?>
                        <a href="<?php echo e($game['download_link']); ?>" class="btn btn-lg btn-success" target="_blank" rel="noopener noreferrer">
                           <i class="fas fa-download"></i> Download Sekarang
                        </a>
                        <p class="small text-muted mt-2">Link akan terbuka di tab baru.</p>
                    <?php else: ?>
                        <button class="btn btn-lg btn-secondary" disabled><i class="fas fa-download"></i> Link Download Tidak Tersedia</button>
                    <?php endif; ?>
                 </div>
            </div>
        </div>
    </div>

     <!-- Kolom Sidebar (Info Tambahan/Komentar) -->
    <div class="col-lg-4">
        <!-- Widget Informasi Tambahan (jika ada) -->
        <!-- <div class="card shadow-sm mb-4">
            <div class="card-header">Informasi Lain</div>
            <div class="card-body"> ... </div>
        </div> -->

        <!-- Widget Komentar & Rating -->
        <div class="card shadow-sm">
            <div class="card-header">
                <h5 class="mb-0"><i class="fas fa-comments"></i> Komentar & Rating</h5>
            </div>
            <div class="card-body">
                <!-- Form Tambah Komentar -->
                 <form action="/process_comment.php" method="POST" class="mb-4 needs-validation" novalidate> <!-- Buat file process_comment.php -->
                     <input type="hidden" name="game_id" value="<?php echo $game['id']; ?>">
                     <input type="hidden" name="csrf_token" value="<?php echo e($_SESSION['csrf_token']); ?>">
                     <div class="mb-3">
                         <label for="commenterName" class="form-label">Nama Anda</label>
                         <input type="text" class="form-control form-control-sm" id="commenterName" name="user_name" required maxlength="100">
                         <div class="invalid-feedback">Nama tidak boleh kosong.</div>
                     </div>
                     <div class="mb-3">
                         <label for="commentRating" class="form-label">Rating (1-5)</label>
                         <select class="form-select form-select-sm" id="commentRating" name="rating">
                           <option value="">-- Beri Rating (Opsional) --</option>
                           <option value="5">★★★★★ (Luar Biasa)</option>
                           <option value="4">★★★★☆ (Bagus)</option>
                           <option value="3">★★★☆☆ (Cukup)</option>
                           <option value="2">★★☆☆☆ (Kurang)</option>
                           <option value="1">★☆☆☆☆ (Buruk)</option>
                         </select>
                     </div>
                     <div class="mb-3">
                         <label for="commentText" class="form-label">Komentar Anda</label>
                         <textarea class="form-control form-control-sm" id="commentText" name="comment" rows="3" required></textarea>
                         <div class="invalid-feedback">Komentar tidak boleh kosong.</div>
                     </div>
                     <button type="submit" class="btn btn-primary btn-sm">Kirim Komentar</button>
                 </form>
                 <hr>

                 <!-- Daftar Komentar -->
                 <h6><?php echo count($comments); ?> Komentar</h6>
                 <?php if (count($comments) > 0): ?>
                    <div class="list-group list-group-flush">
                        <?php foreach ($comments as $comment): ?>
                            <div class="list-group-item px-0">
                                <div class="d-flex w-100 justify-content-between">
                                    <h6 class="mb-1"><?php echo e($comment['user_name']); ?></h6>
                                    <small class="text-muted"><?php echo date('d M Y', strtotime($comment['created_at'])); ?></small>
                                </div>
                                <?php if ($comment['rating']): ?>
                                    <p class="mb-1 small text-warning">
                                        <?php for ($i = 0; $i < 5; $i++): ?>
                                            <i class="<?php echo ($i < $comment['rating']) ? 'fas' : 'far'; ?> fa-star"></i>
                                        <?php endfor; ?>
                                    </p>
                                <?php endif; ?>
                                <p class="mb-1 small"><?php echo nl2br(e($comment['comment'])); ?></p>
                            </div>
                        <?php endforeach; ?>
                    </div>
                     <!-- Tambahkan Pagination Komentar jika perlu -->
                 <?php else: ?>
                     <p class="text-muted">Belum ada komentar untuk game ini. Jadilah yang pertama!</p>
                 <?php endif; ?>
            </div>
        </div>
    </div>
</div>


<?php include __DIR__ . '/includes/footer.php'; ?>
```

**Perlu Membuat `process_comment.php`:** Skrip ini mirip dengan `feedback.php` atau `process_game.php`, bertugas memvalidasi input komentar (nama, komentar, rating, game_id, csrf_token) dan menyimpannya ke tabel `game_comments`. Jangan lupa redirect kembali ke `game_detail.php?id=...` setelah berhasil atau gagal.

**Langkah 12: Keamanan Tambahan & Perbaikan**

1.  **`.htaccess` (Untuk Apache):** Buat file `.htaccess` di root folder proyek.
    ```apacheconfig
    # Mengaktifkan Mesin Rewrite
    RewriteEngine On

    # Mencegah akses langsung ke folder includes, admin (kecuali file tertentu), dll.
    RewriteRule ^includes/ - [F,L]
    RewriteRule ^admin/(config|db|functions)\.php - [F,L] # Contoh
    RewriteRule ^uploads/ - [F,L] # Mencegah listing direktori, tapi file tetap bisa diakses via URL langsung
    RewriteRule ^config\.php - [F,L]
    RewriteRule ^\.env - [F,L] # Jika pakai file .env

    # (Opsional) Mengarahkan semua request ke index.php (untuk routing yang lebih canggih)
    # RewriteCond %{REQUEST_FILENAME} !-f
    # RewriteCond %{REQUEST_FILENAME} !-d
    # RewriteRule ^(.*)$ index.php?url=$1 [QSA,L]

    # (Opsional) Hapus ekstensi .php dari URL
    # RewriteCond %{REQUEST_FILENAME} !-d
    # RewriteCond %{REQUEST_FILENAME}\.php -f
    # RewriteRule ^([^/]+)/?$ $1.php [L]

    # (Opsional) Redirect HTTP ke HTTPS
    # RewriteCond %{HTTPS} off
    # RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]

    # Header Keamanan Tambahan
    <IfModule mod_headers.c>
        Header set X-Content-Type-Options "nosniff"
        Header set X-Frame-Options "SAMEORIGIN"
        Header set Referrer-Policy "strict-origin-when-cross-origin"
        # Header set Content-Security-Policy "default-src 'self'; script-src 'self' https://cdn.jsdelivr.net; style-src 'self' https://cdn.jsdelivr.net https://cdnjs.cloudflare.com; img-src 'self' data: https://via.placeholder.com; font-src 'self' https://cdnjs.cloudflare.com;"
        # CSP perlu diatur dengan hati-hati agar tidak memblokir resource yang sah
    </IfModule>

    # Opsi untuk mencegah directory listing
    Options -Indexes

    # Proteksi file .htaccess itu sendiri
    <Files .htaccess>
        Require all denied
    </Files>
    <Files config.php>
         Require all denied
    </Files>

    # Set default character set
    AddDefaultCharset UTF-8
    ```
    *   **Penting:** Konfigurasi `.htaccess` bisa kompleks. Uji dengan hati-hati. Kebijakan `Content-Security-Policy` (CSP) sangat efektif tetapi perlu disesuaikan dengan resource eksternal (CDN) yang Anda gunakan.

2.  **Validasi Input:** Pastikan *semua* input dari `$_GET`, `$_POST`, `$_FILES` divalidasi dengan ketat di sisi server (tipe data, panjang, format, nilai yang diizinkan) sebelum digunakan dalam query atau logika lainnya.
3.  **Sanitasi Output:** Gunakan `htmlspecialchars()` (atau fungsi `e()` kita) *setiap kali* menampilkan data yang berasal dari user atau database ke HTML.
4.  **Prepared Statements:** Sudah digunakan via PDO, ini kunci mencegah SQL Injection.
5.  **CSRF Protection:** Sudah diimplementasikan di form-form penting. Pastikan token dicek di *semua* request yang mengubah state (POST, DELETE, PUT).
6.  **Rate Limiting:** Contoh sederhana ada di `admin/login.php`. Bisa diperluas ke fitur lain yang rentan (misal: submit feedback/komentar). Implementasi yang lebih robust mungkin memerlukan penyimpanan di database atau cache (Redis/Memcached).
7.  **Upload File:** Validasi tipe MIME *dan* ekstensi. Gunakan nama file unik. Jangan percaya nama file asli dari user. Simpan file di luar *document root* jika memungkinkan, atau setidaknya batasi akses langsung via `.htaccess`.
8.  **Error Handling:** Matikan `display_errors` di production (`config.php`). Gunakan `error_log()` untuk mencatat error. Buat halaman error kustom (misal, via `.htaccess` `ErrorDocument 404 /404.php`) untuk pengalaman pengguna yang lebih baik.
9.  **Password:** Gunakan `password_hash()` dan `password_verify()`. Jangan simpan password plain text.
10. **Session Security:** Gunakan `session_regenerate_id(true)` setelah login berhasil. Pertimbangkan setting cookie session (HttpOnly, Secure).
11. **HTTPS:** Selalu gunakan HTTPS di production untuk mengenkripsi komunikasi.

**Langkah 13: SEO & Sitemap**

1.  **Meta Tags:** Header sudah menyertakan `<meta name="description">`. Tambahkan meta tag relevan lainnya (keywords, author, dll.) jika perlu. Pastikan `<title>` unik dan deskriptif untuk setiap halaman.
2.  **Sitemap (`sitemap.xml`):** Buat script PHP terpisah (misal `generate_sitemap.php`) yang mengambil semua URL penting (halaman statis, semua game detail) dari database dan menghasilkan file XML sesuai format sitemap. Jalankan script ini secara berkala (misal via cron job) atau saat ada konten baru.
    ```xml
    <!-- Contoh sitemap.xml -->
    <?xml version="1.0" encoding="UTF-8"?>
    <urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
       <url>
          <loc>https://www.yourdomain.com/</loc>
          <lastmod>2023-10-27</lastmod>
          <changefreq>daily</changefreq>
          <priority>1.0</priority>
       </url>
       <url>
          <loc>https://www.yourdomain.com/about.php</loc>
          <lastmod>2023-10-26</lastmod>
          <changefreq>monthly</changefreq>
          <priority>0.8</priority>
       </url>
       <url>
          <loc>https://www.yourdomain.com/feedback.php</loc>
          <lastmod>2023-10-26</lastmod>
          <changefreq>monthly</changefreq>
          <priority>0.7</priority>
       </url>
       <!-- Loop untuk setiap game -->
       <url>
          <loc>https://www.yourdomain.com/game_detail.php?id=123</loc>
          <lastmod>2023-10-25</lastmod> <!-- Ambil dari kolom updated_at game -->
          <changefreq>weekly</changefreq>
          <priority>0.9</priority>
       </url>
       <!-- ... game lainnya ... -->
    </urlset>
    ```
3.  **`robots.txt`:** Buat file `robots.txt` di root untuk memberi tahu crawler direktori mana yang boleh dan tidak boleh diindeks.
    ```
    User-agent: *
    Allow: /
    Disallow: /admin/
    Disallow: /includes/
    Disallow: /process_*.php
    Disallow: /uploads/ # Mungkin ingin mengizinkan gambar jika penting untuk SEO gambar
    Disallow: /login.php
    Disallow: /logout.php
    # Tambahkan URL lain yang tidak ingin diindeks

    Sitemap: https://www.yourdomain.com/sitemap.xml
    ```

**Selanjutnya:**

*   **Testing:** Uji semua fitur secara menyeluruh, termasuk skenario error dan input yang tidak valid.
*   **Deployment:** Pindahkan file ke server hosting Anda, konfigurasi database, pastikan permission folder `uploads/` benar, dan atur `display_errors` ke `0` di `config.php`.
*   **Maintenance:** Backup database secara teratur, update library (PHP, Bootstrap, dll.), dan pantau log error.

Ini adalah kerangka kerja yang cukup lengkap. Anda bisa mengembangkannya lebih lanjut dengan fitur-fitur lain sesuai kebutuhan. Selamat mencoba!