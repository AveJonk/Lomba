# How to push
Alur kerjanya adalah mengerjakan perubahan di branch fitur milikmu sendiri, lalu mem-push branch tersebut ke GitHub, dan membuka Pull Request (PR) dari branch fiturmu menuju branch Development (bukan main/master).

1. Pastikan Branch Lokal Sinkron dengan Development
Sebelum membuat branch baru atau mulai ngoding, ambil update terbaru dari remote supaya tidak bentrok (conflict) nanti.

Bash
git checkout Development
git pull origin Development

2. Buat & Pindah ke Branch Fitur Baru
Jangan commit langsung di Development. Buat branch baru yang deskriptif untuk perubahanmu.

Bash
git checkout -b nama-branch-kamu
(Contoh: git checkout -b fitur/login-page atau git checkout -b fix/navbar-bug)

3. Stage, Commit, dan Push ke GitHub
Setelah selesai mengubah kode, simpan perubahan dan kirim branch baru tersebut ke repository GitHub.

Bash
git add .
git commit -m "pesan commit yang jelas"
git push -u origin nama-branch-kamu
Catatan: Flag -u (--set-upstream) cukup dipakai saat pertama kali push branch baru.
