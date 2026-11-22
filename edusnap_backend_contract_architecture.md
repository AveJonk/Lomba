# ⚙️ EduSnap — BackEnd Technical Contract & Specification

Dokumen ini adalah panduan teknis khusus untuk implementasi **BackEnd, Database (Firebase), Logika Penilaian, dan Vision/OCR Engine** pada proyek **EduSnap**.

---

## 1. Arsitektur Layanan & Konfigurasi (`src/config/`)

### A. Firebase (`src/config/firebase.js`)
Menginisialisasi tiga layanan utama Firebase (Free Tier / Spark Plan):
* **Firebase Authentication:** Email/Password provider untuk autentikasi 4 peran (`admin`, `teacher`, `student`, `parent`).
* **Cloud Firestore:** Penyimpanan data relasional ringan berbasis dokumen.
* **Firebase Storage:** Penyimpanan file foto lembar jawaban fisik (`ljk_uploads/{taskId}/{studentId}_{timestamp}.jpg`).

### B. Vision / OCR Engine (`src/config/visionApi.js`)
* Menggunakan **Gemini 2.5 Flash API** (via REST API atau `@google/genai` SDK) dengan mode *Structured Output* (`responseSchema`) untuk membaca jawaban pilihan ganda maupun tulisan tangan kata/kalimat pada kertas LJK siswa secara presisi.

---

## 2. Skema Database Firestore (Collections & Documents)

### Collection: `users`
* **Document ID:** `{uid}` (mengikuti UID dari Firebase Auth)
* **Fields:**
  ```json
  {
    "uid": "string",
    "name": "string",
    "email": "string",
    "role": "admin | teacher | student | parent",
    "nisn": "string | null",
    "isActive": "boolean",
    "createdAt": "Timestamp"
  }
  ```
* **Indexing / Query:** Index pada `role` dan `nisn` (untuk mencocokkan akun `parent` dengan data `student`).

### Collection: `courses`
* **Document ID:** `{courseId}` (auto-generated)
* **Fields:**
  ```json
  {
    "courseId": "string",
    "name": "string",
    "teacherId": "string",
    "teacherName": "string",
    "semester": "string"
  }
  ```

### Collection: `tasks`
* **Document ID:** `{taskId}` (auto-generated)
* **Fields:**
  ```json
  {
    "taskId": "string",
    "courseId": "string",
    "courseName": "string",
    "teacherId": "string",
    "title": "string",
    "videoUrl": "string",
    "weight": {
      "pg": "number",
      "essay": "number"
    },
    "hasGame": "boolean",
    "gameConfig": {
      "theme": "kodok_teratai | tarik_tambang",
      "timeLimitSeconds": "number",
      "questions": [
        {
          "qId": "string",
          "question": "string",
          "options": ["string", "string", "string", "string"],
          "correctIndex": "number"
        }
      ]
    },
    "questions": [
      {
        "number": "number",
        "type": "pg | essay",
        "questionText": "string",
        "answerKey": "string",
        "maxPoints": "number"
      }
    ],
    "createdAt": "Timestamp"
  }
  ```

### Collection: `submissions`
* **Document ID:** `{taskId}_{studentId}` (Composite ID untuk mencegah duplikasi pengiriman oleh siswa yang sama pada tugas yang sama)
* **Fields:**
  ```json
  {
    "submissionId": "string",
    "taskId": "string",
    "courseName": "string",
    "studentId": "string",
    "studentName": "string",
    "nisn": "string",
    "status": "in_progress | waiting_essay | needs_reupload | completed",
    "gameScore": "number",
    "ljkImageUrl": "string",
    "answers": [
      {
        "number": "number",
        "type": "pg | essay",
        "detectedText": "string",
        "isCorrect": "boolean | null",
        "score": "number",
        "maxPoints": "number"
      }
    ],
    "pgTotalScore": "number",
    "essayTotalScore": "number",
    "finalScore": "number",
    "submittedAt": "Timestamp",
    "gradedAt": "Timestamp | null"
  }
  ```

### Collection: `attendances`
* **Document ID:** `{date_YYYYMMDD}_{studentId}`
* **Fields:**
  ```json
  {
    "attendanceId": "string",
    "date": "string (YYYY-MM-DD)",
    "studentId": "string",
    "studentName": "string",
    "nisn": "string",
    "status": "hadir | izin | sakit | alpa",
    "teacherNote": "string",
    "recordedBy": "string (teacherId)"
  }
  ```

---

## 3. Spesifikasi Kontrak Service (`src/services/`)

Setiap fungsi wajib menangani *error* dengan `try/catch` dan melempar pesan error yang ramah pengguna (`throw new Error("Pesan jelas")`).

### 1. `authService.js`
* **`login(email, password, expectedRole)`**
  * *Proses:* Memanggil `signInWithEmailAndPassword`, mengambil dokumen `users/{uid}`, memvalidasi `user.isActive === true` dan `user.role === expectedRole`. Jika tidak cocok, paksa `signOut()` dan lempar error.
  * *Return:* `Promise<User>`
* **`registerUserByAdmin(payload)`**
  * *Catatan Teknis:* Karena Firebase Client SDK akan otomatis me-login-kan user yang baru dibuat dengan `createUserWithEmailAndPassword`, gunakan **Secondary Firebase App Instance** (`initializeApp(config, "Secondary")`) di dalam fungsi ini agar sesi login Admin utama tidak terputus!
  * *Return:* `Promise<User>`

### 2. `taskService.js`
* **`createTask(taskPayload)`**
  * *Proses:* Validasi total bobot (`weight.pg + weight.essay === 100`), simpan ke koleksi `tasks`.
  * *Return:* `Promise<string>` (`taskId`)
* **`getTasksForStudent(studentId)`**
  * *Proses:* Mengambil seluruh `tasks` aktif dan menggabungkannya (*join* di sisi klien) dengan status pengerjaan siswa dari koleksi `submissions` berdasarkan `studentId`.
  * *Return:* `Promise<Array<TaskWithStatus>>`

### 3. `ocrService.js` (Kamera Pembaca Teks Kertas)
* **`scanPaperImage(rawImageFile, taskQuestions)`**
  * *Langkah 1 (Kompresi):* Panggil `imageCompressor.js` untuk mengubah resolusi gambar maksimal lebar `1280px` (kualitas JPEG `0.8`) agar proses unggah dan analisis cepat (< 3 detik).
  * *Langkah 2 (Vision API Call):* Konversi gambar ke Base64 dan kirim ke Gemini Vision API dengan *system prompt* ekstraksi terstruktur.
  * *Langkah 3 (Parsing & Pencocokan PG):* Cocokkan hasil `detectedText` pada soal bertipe `"pg"` secara otomatis dengan `answerKey` (`case-insensitive`, *trimmed*).
  * *Return:* `Promise<Array<Answer>>`

### 4. `submissionService.js`
* **`submitWork({ taskId, studentId, gameScore, imageFile, confirmedAnswers, taskData })`**
  * *Proses:*
    1. Unggah `imageFile` ke Firebase Storage (`ljk_uploads/{taskId}/{studentId}.jpg`) dan ambil `downloadURL`.
    2. Hitung skor PG (`pgTotalScore`).
    3. Tentukan `status`: jika `taskData` tidak memiliki soal `"essay"`, langsung set `status: "completed"` dan hitung `finalScore`; jika ada esai, set `status: "waiting_essay"`.
    4. Simpan dokumen ke `submissions/{taskId}_{studentId}` menggunakan `setDoc`.
  * *Return:* `Promise<Submission>`
* **`gradeEssay(submissionId, gradedAnswers, taskWeight)`**
  * *Proses:* Memperbarui skor esai (skala 0–5 per nomor) dari menu *Split-Screen* guru, memanggil `calculateFinalScore()`, mengubah `status` menjadi `"completed"`, dan mengisi `gradedAt`.
  * *Return:* `Promise<void>`
* **`requestReupload(submissionId, reason)`**
  * *Proses:* Mengubah `status` menjadi `"needs_reupload"` tanpa menghapus `gameScore` siswa, sehingga siswa hanya perlu mengulang Tahap 3 (Upload Foto LJK).
  * *Return:* `Promise<void>`

---

## 4. Logika Utilitas Inti (`src/utils/`)

### A. Rumus Perhitungan Nilai Dinamis (`scoreCalculator.js`)
Menangani kasus tugas yang hanya berisi PG saja, Esai saja, atau campuran keduanya:

```javascript
export function calculateFinalScore(answers, weight = { pg: 70, essay: 30 }) {
  const pgItems = answers.filter((a) => a.type === "pg");
  const essayItems = answers.filter((a) => a.type === "essay");

  let pgPercentage = 0;
  if (pgItems.length > 0) {
    const pgEarned = pgItems.reduce((sum, item) => sum + (item.score || 0), 0);
    const pgMax = pgItems.reduce((sum, item) => sum + (item.maxPoints || 1), 0);
    pgPercentage = (pgEarned / pgMax) * 100;
  }

  let essayPercentage = 0;
  if (essayItems.length > 0) {
    const essayEarned = essayItems.reduce((sum, item) => sum + (item.teacherScore || 0), 0);
    const essayMax = essayItems.reduce((sum, item) => sum + (item.maxPoints || 5), 0);
    essayPercentage = (essayEarned / essayMax) * 100;
  }

  // Jika hanya ada PG (tidak ada soal esai)
  if (pgItems.length > 0 && essayItems.length === 0) {
    return Math.round(pgPercentage);
  }

  // Jika hanya ada Esai (tidak ada soal PG)
  if (essayItems.length > 0 && pgItems.length === 0) {
    return Math.round(essayPercentage);
  }

  // Campuran PG dan Esai sesuai bobot tugas
  const final = (pgPercentage * (weight.pg / 100)) + (essayPercentage * (weight.essay / 100));
  return Math.round(final);
}
```

### B. Standar Prompt Vision/OCR (`ocrService.js`)
Gunakan *prompt* dan skema JSON ini saat memanggil API Vision agar kamera akurat membaca tulisan per kata di kertas siswa:

```javascript
export const OCR_SYSTEM_PROMPT = `
Kamu adalah sistem pembaca Lembar Jawaban Siswa (EduSnap OMR & OCR).
Tugasmu adalah membaca gambar kertas jawaban yang ditulis tangan atau disilang oleh siswa.
Aturan ekstraksi:
1. Cocokkan nomor jawaban pada kertas dengan daftar nomor soal yang diberikan.
2. Untuk soal bertipe "pg" (Pilihan Ganda), ekstrak hanya SATU huruf kapital jawaban yang dipilih siswa ("A", "B", "C", atau "D"). Jika kosong/tidak terbaca, isi "".
3. Untuk soal bertipe "essay", transkripsikan seluruh kata dan kalimat yang ditulis siswa pada nomor tersebut secara akurat sesuai ejaan aslinya. Jika kosong, isi "".
4. Kembalikan output HANYA dalam format JSON array yang valid sesuai skema.
`;
```

---

## 5. Firestore Security Rules (Standar Produksi & Demo)

Terapkan aturan berikut pada tab **Firestore Database > Rules** di Firebase Console agar data aman namun tetap lancar saat demonstrasi:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    
    function isSignedIn() {
      return request.auth != null;
    }
    
    function getUserRole() {
      return get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role;
    }

    match /users/{userId} {
      allow read: if isSignedIn();
      allow write: if isSignedIn() && (request.auth.uid == userId || getUserRole() == 'admin');
    }

    match /courses/{courseId} {
      allow read: if isSignedIn();
      allow write: if isSignedIn() && (getUserRole() == 'teacher' || getUserRole() == 'admin');
    }

    match /tasks/{taskId} {
      allow read: if isSignedIn();
      allow write: if isSignedIn() && getUserRole() == 'teacher';
    }

    match /submissions/{submissionId} {
      allow read: if isSignedIn();
      allow create: if isSignedIn() && getUserRole() == 'student';
      allow update: if isSignedIn() && (getUserRole() == 'student' || getUserRole() == 'teacher');
    }

    match /attendances/{attendanceId} {
      allow read: if isSignedIn();
      allow write: if isSignedIn() && getUserRole() == 'teacher';
    }
  }
}
```