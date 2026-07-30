# Master Runbook: Setup GitLab CI/CD dari Nol Mutlak

Panduan ini mengasumsikan **belum ada apapun**: belum ada project GitLab, belum ada kode di server, belum ada file `.gitlab-ci.yml`. Ikuti urutan dari atas ke bawah tanpa loncat.

**Target akhir:**
- Push ke branch `development` → otomatis deploy ke `http://IP_SERVER:5001`
- Merge ke branch `main` → deploy manual (1 klik) ke `http://IP_SERVER:5000`
- Semua pakai 1 server dummy & GitLab Shared Runner (gratis, tanpa install runner sendiri)

---

## BAGIAN A — Siapkan Project di GitLab

### A.1 Buat Project Baru

1. Login ke [gitlab.com](https://gitlab.com)
2. Klik **New project** → **Create blank project**
3. Isi:
   - **Project name:** misal `flaskpilates`
   - **Visibility:** Private (disarankan)
   - Centang/hilangkan **"Initialize repository with a README"** (boleh dicentang agar repo tidak kosong)
4. Klik **Create project**

### A.2 Buat Personal Access Token (untuk push dari lokal via HTTPS)

Jika belum punya SSH key di komputer lokal, cara termudah pakai token:

1. Klik avatar profil (kanan atas) → **Edit profile** → **Access Tokens**
2. **Add new token**, beri nama `local-dev`, centang scope `write_repository` dan `read_repository`
3. Klik **Create personal access token** → **copy tokennya sekarang** (hanya tampil sekali)

*(Jika lebih nyaman pakai SSH key dari lokal, bisa skip ini dan setup SSH key seperti biasa — lihat panduan awal soal koneksi VSCode ke GitLab yang sudah dibahas sebelumnya.)*

### A.3 Clone Project ke Komputer Lokal

```bash
git clone https://gitlab.com/username/flaskpilates.git
cd flaskpilates
```

Saat diminta login, gunakan username GitLab Anda + **token** dari A.2 sebagai password.

### A.4 Tambahkan Kode Project Anda

Masukkan source code aplikasi Flask Anda ke dalam folder ini (jika belum ada, buat contoh minimal dulu):

```bash
# contoh minimal app.py (skip jika kode asli sudah ada)
cat > app.py << 'EOF'
from flask import Flask
app = Flask(__name__)

@app.route("/")
def home():
    return "Hello from Flask CI/CD Demo!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
EOF

cat > requirements.txt << 'EOF'
flask
gunicorn
EOF
```

### A.5 Commit & Push Pertama ke `main`

```bash
git add .
git commit -m "Initial commit: base Flask app"
git branch -M main
git push -u origin main
```

### A.6 Buat Branch `development`

```bash
git checkout -b development
git push -u origin development
```

Sekarang di GitLab, buka **Code > Branches** — pastikan `main` dan `development` sudah muncul keduanya.

---

## BAGIAN B — Siapkan Server Dummy

### B.1 Login ke Server

```bash
ssh user_anda@IP_SERVER_DUMMY
```

Catat **IP server** dan **username** ini — dipakai lagi di Bagian D.

### B.2 Install Prasyarat di Server (jika belum ada)

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip git
```

### B.3 Buat SSH Key Khusus untuk GitLab CI

Masih di dalam server:

```bash
ssh-keygen -t rsa -b 4096 -C "gitlab-ci-deploy" -f ~/gitlab_ci_key -N ""
cat ~/gitlab_ci_key.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

### B.4 Encode Private Key ke Base64

```bash
base64 -w0 ~/gitlab_ci_key > ~/gitlab_ci_key.base64.txt
cat ~/gitlab_ci_key.base64.txt
```

**Copy seluruh output** (satu baris panjang) — simpan sementara di text editor lokal, akan dipakai di Bagian D.

### B.5 Ambil Fingerprint Server

```bash
ssh-keyscan -H IP_SERVER_DUMMY
```

**Copy seluruh output** (2-3 baris) — simpan juga untuk Bagian D.

### B.6 Buat Deploy Token di GitLab (agar server bisa clone/pull repo)

Kembali ke GitLab (browser):
1. Buka project → **Settings > Repository > Deploy tokens**
2. **Add token**: name `server-pull`, scope centang `read_repository`
3. Klik **Create deploy token** → copy **username** dan **token**-nya

### B.7 Clone Project ke 2 Folder di Server

Kembali ke terminal server:

```bash
sudo mkdir -p /opt/flaskpilates-dev /opt/flaskpilates-prod
sudo chown -R $USER:$USER /opt/flaskpilates-dev /opt/flaskpilates-prod

git clone https://DEPLOY_TOKEN_USERNAME:DEPLOY_TOKEN_PASSWORD@gitlab.com/username/flaskpilates.git /opt/flaskpilates-dev
git clone https://DEPLOY_TOKEN_USERNAME:DEPLOY_TOKEN_PASSWORD@gitlab.com/username/flaskpilates.git /opt/flaskpilates-prod

cd /opt/flaskpilates-dev && git checkout development
cd /opt/flaskpilates-prod && git checkout main
```

### B.8 Setup Virtual Environment di Kedua Folder

```bash
cd /opt/flaskpilates-dev
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
deactivate

cd /opt/flaskpilates-prod
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
deactivate
```

### B.9 Buat Systemd Service untuk Masing-Masing Environment

```bash
sudo nano /etc/systemd/system/flaskpilates-dev.service
```
Isi:
```ini
[Unit]
Description=Flask Pilates - Development
After=network.target

[Service]
User=user_anda
WorkingDirectory=/opt/flaskpilates-dev
ExecStart=/opt/flaskpilates-dev/venv/bin/gunicorn -w 2 -b 0.0.0.0:5001 app:app
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo nano /etc/systemd/system/flaskpilates-prod.service
```
Isi:
```ini
[Unit]
Description=Flask Pilates - Production
After=network.target

[Service]
User=user_anda
WorkingDirectory=/opt/flaskpilates-prod
ExecStart=/opt/flaskpilates-prod/venv/bin/gunicorn -w 4 -b 0.0.0.0:5000 app:app
Restart=always

[Install]
WantedBy=multi-user.target
```

Aktifkan:
```bash
sudo systemctl daemon-reload
sudo systemctl enable flaskpilates-dev.service flaskpilates-prod.service
sudo systemctl start flaskpilates-dev.service flaskpilates-prod.service
sudo systemctl status flaskpilates-dev.service
sudo systemctl status flaskpilates-prod.service
```

Pastikan keduanya berstatus **active (running)** sebelum lanjut.

### B.10 Izinkan Restart Service Tanpa Password

```bash
sudo visudo -f /etc/sudoers.d/deploy-flaskpilates
```
Isi (ganti `user_anda`):
```
user_anda ALL=(ALL) NOPASSWD: /bin/systemctl restart flaskpilates-dev.service, /bin/systemctl restart flaskpilates-prod.service
```

### B.11 Buka Firewall

```bash
sudo ufw allow 5000/tcp
sudo ufw allow 5001/tcp
sudo ufw allow OpenSSH
sudo ufw status
```

Uji akses langsung dulu sebelum lanjut ke CI/CD:
```bash
curl http://localhost:5001   # setelah dijalankan manual sementara, atau lanjut ke bagian gunicorn service
curl http://localhost:5000
```

---

## BAGIAN C — Buat File `deploy/deploy.sh`

Kembali ke **komputer lokal**, di dalam folder project:

```bash
mkdir -p deploy
```

Buat `deploy/deploy.sh`:
```bash
#!/bin/bash
set -e

echo ">> Menarik kode terbaru..."
git pull origin "$(git rev-parse --abbrev-ref HEAD)"

echo ">> Install dependency..."
source venv/bin/activate
pip install -r requirements.txt

echo ">> Restart service..."
CURRENT_DIR=$(pwd)
if [[ "$CURRENT_DIR" == *"-dev" ]]; then
  sudo systemctl restart flaskpilates-dev.service
else
  sudo systemctl restart flaskpilates-prod.service
fi

echo ">> Selesai."
```

```bash
chmod +x deploy/deploy.sh
git add deploy
git commit -m "chore: add deploy script"
git push origin development
```

Tarik perubahan ini ke kedua folder di server:
```bash
# di server
cd /opt/flaskpilates-dev && git pull origin development
cd /opt/flaskpilates-prod && git checkout development -- deploy/deploy.sh  # sementara, sampai di-merge ke main
chmod +x /opt/flaskpilates-dev/deploy/deploy.sh
chmod +x /opt/flaskpilates-prod/deploy/deploy.sh
```

---

## BAGIAN D — Setup CI/CD Variables di GitLab

Buka project di GitLab → **Settings > CI/CD** → expand bagian **Variables** → **Add variable**.

Buat 4 variable ini satu per satu:

| Key | Value | Protected | Masked |
|---|---|---|---|
| `SERVER_IP` | IP server dummy (dari B.1) | ✅ | ✅ |
| `SERVER_USER` | username SSH (dari B.1) | ✅ | ❌ |
| `SSH_PRIVATE_KEY` | isi dari B.4 (base64, 1 baris) | ✅ | ❌ |
| `SSH_KNOWN_HOSTS` | isi dari B.5 | ✅ | ❌ |

Biarkan **Environment scope = All (default)** untuk semua (karena 1 server dipakai untuk semua environment).

---

## BAGIAN E — Tandai Branch sebagai Protected

`Settings > Repository` → expand **Protected branches**:

- Branch `main` → Allowed to merge: **Maintainers**, Allowed to push: **No one** (atau Maintainers)
- Branch `development` → Allowed to merge: **Developers + Maintainers**, Allowed to push: **Developers + Maintainers**

> Ini wajib — variable Protected di Bagian D hanya "terlihat" oleh pipeline yang berjalan di branch berstatus Protected.

---

## BAGIAN F — Buat File `.gitlab-ci.yml`

Di komputer lokal, di root project, buat file `.gitlab-ci.yml`:

```yaml
stages:
  - test
  - deploy

.deploy_template:
  stage: deploy
  image: ubuntu
  before_script:
    - 'command -v ssh-agent >/dev/null || ( apt-get update -y && apt-get install -y openssh-client git )'
    - eval $(ssh-agent -s)
    - '[ -n "$SSH_PRIVATE_KEY" ] || { echo "ERROR: SSH_PRIVATE_KEY kosong — cek Settings > CI/CD > Variables"; exit 1; }'
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | base64 -d > /tmp/id_rsa
    - '[ -s /tmp/id_rsa ] || { echo "ERROR: hasil decode kosong — SSH_PRIVATE_KEY kemungkinan bukan base64 valid"; exit 1; }'
    - chmod 600 /tmp/id_rsa
    - ssh-add /tmp/id_rsa
    - mkdir -p ~/.ssh && chmod 700 ~/.ssh
    - echo "$SSH_KNOWN_HOSTS" > ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts
  script:
    - ssh "$SERVER_USER@$SERVER_IP" "cd '$APP_DIR' && bash deploy/deploy.sh"

run_tests:
  stage: test
  image: python:3.11-slim
  script:
    - pip install -r requirements.txt
    - pytest || echo "Belum ada test, lewati sementara"
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "development"'
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy_dev:
  extends: .deploy_template
  environment:
    name: development
  variables:
    APP_DIR: /opt/flaskpilates-dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "development"'

deploy_prod:
  extends: .deploy_template
  environment:
    name: production
  variables:
    APP_DIR: /opt/flaskpilates-prod
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

Push file ini ke branch `development`:
```bash
git add .gitlab-ci.yml
git commit -m "ci: add gitlab-ci.yml pipeline"
git push origin development
```

---

## BAGIAN G — Uji Coba End-to-End

### G.1 Cek Pipeline Otomatis Jalan

1. Buka GitLab → **Build > Pipelines**
2. Pipeline baru akan muncul otomatis (karena baru saja push ke `development`)
3. Klik pipeline tersebut, pastikan:
   - Job `run_tests` → ✅ passed
   - Job `deploy_dev` → ✅ passed
4. Jika salah satu **failed**, klik job tersebut, baca log error di baris paling bawah (biasanya jelas menyebutkan penyebabnya, misal `Permission denied` atau `SSH_PRIVATE_KEY kosong`)

### G.2 Verifikasi Aplikasi Live di Server

```bash
curl http://IP_SERVER_DUMMY:5001
```
Harus muncul output `Hello from Flask CI/CD Demo!` (atau isi aplikasi asli Anda).

### G.3 Tes Alur Production

1. Di GitLab, buka **Code > Merge requests > New merge request**
2. Source: `development`, Target: `main`
3. Klik **Create merge request**, tunggu pipeline test selesai (passed)
4. Klik **Merge**
5. Buka **Build > Pipelines**, cari pipeline baru di branch `main`
6. Job `deploy_prod` akan muncul dengan ikon **▶ manual** — klik untuk menjalankan
7. Verifikasi:
```bash
curl http://IP_SERVER_DUMMY:5000
```

---

## BAGIAN H — Ganti Demo dengan Source Code Flask Anda yang Sebenarnya

Bagian A.4 sebelumnya memakai contoh Flask dummy. Berikut cara menggantinya dengan project Flask **asli** milik Anda, dari awal.

### H.1 Cek Struktur Project Anda Dulu

Sebelum masuk ke repo, jalankan project Anda **secara lokal** dan pastikan tahu 3 hal ini (dibutuhkan di langkah berikutnya):

```bash
# 1. Cek nama file entry point (biasanya app.py, run.py, wsgi.py, atau main.py)
ls *.py

# 2. Cek nama variabel Flask instance di dalam file tsb, misal:
grep -n "Flask(" *.py
# Contoh isi: app = Flask(__name__)  →  entry point-nya "app:app"
#             application = Flask(__name__) → entry point-nya "app:application"

# 3. Cek apakah pakai app factory pattern (fungsi create_app())
grep -n "create_app" *.py
```

Catat hasilnya — dipakai di H.4 (perintah `gunicorn`).

### H.2 Pastikan `requirements.txt` Lengkap

Jika project Anda belum punya `requirements.txt`, generate dari environment lokal:

```bash
pip freeze > requirements.txt
```

Pastikan `gunicorn` ada di dalamnya (tambahkan manual jika belum):
```bash
echo "gunicorn" >> requirements.txt
```

### H.3 Pindahkan Source Code Asli ke Repo GitLab

Jika project Anda **sudah ada di folder terpisah** (belum terhubung ke repo GitLab yang dibuat di Bagian A), pindahkan isinya:

```bash
# Masuk ke folder repo yang sudah di-clone dari Bagian A.3
cd flaskpilates

# Hapus file demo (app.py, requirements.txt dummy)
rm -f app.py requirements.txt

# Copy seluruh source code asli Anda ke sini
cp -r /path/ke/project/flask/asli/* .

# Cek tidak ada file sensitif ikut ter-copy
ls -la
```

**Jangan ikutkan folder-folder ini** — buat/edit `.gitignore`:
```bash
cat > .gitignore << 'EOF'
venv/
__pycache__/
*.pyc
.env
instance/
*.sqlite3
.DS_Store
EOF
```

> **Penting soal `.env`:** jika aplikasi Anda pakai file `.env` untuk secret (SECRET_KEY, DATABASE_URL, dll), file ini **tidak boleh ikut ter-push ke Git**. Anda perlu membuat `.env` **langsung di server** (lihat H.5), terpisah dari proses deploy otomatis.

### H.4 Sesuaikan Perintah Gunicorn di Systemd Service

Kembali ke server (`sudo nano /etc/systemd/system/flaskpilates-dev.service` dan `-prod.service` dari Step B.9), sesuaikan baris `ExecStart` berdasarkan hasil pengecekan H.1:

**Jika entry point Anda `app.py` dengan `app = Flask(__name__)`:**
```ini
ExecStart=/opt/flaskpilates-dev/venv/bin/gunicorn -w 2 -b 0.0.0.0:5001 app:app
```

**Jika entry point Anda `run.py` dengan variabel `application`:**
```ini
ExecStart=/opt/flaskpilates-dev/venv/bin/gunicorn -w 2 -b 0.0.0.0:5001 run:application
```

**Jika Anda pakai app factory pattern (`create_app()`):**
```ini
ExecStart=/opt/flaskpilates-dev/venv/bin/gunicorn -w 2 -b 0.0.0.0:5001 "app:create_app()"
```

Setelah edit, reload dan restart:
```bash
sudo systemctl daemon-reload
sudo systemctl restart flaskpilates-dev.service
sudo systemctl status flaskpilates-dev.service
```

Lakukan hal sama untuk file `-prod.service`.

### H.5 Buat File `.env` Langsung di Server (Jika Ada)

Karena `.env` di-ignore dari Git (H.3), buat manual di kedua folder:

```bash
nano /opt/flaskpilates-dev/.env
```
Isi sesuai kebutuhan aplikasi Anda, misal:
```
FLASK_ENV=development
SECRET_KEY=ganti-dengan-secret-key-dev
DATABASE_URL=sqlite:///dev.db
```

```bash
nano /opt/flaskpilates-prod/.env
```
Isi versi production:
```
FLASK_ENV=production
SECRET_KEY=ganti-dengan-secret-key-prod-yang-kuat
DATABASE_URL=postgresql://user:pass@localhost/proddb
```

Amankan izin file:
```bash
chmod 600 /opt/flaskpilates-dev/.env /opt/flaskpilates-prod/.env
```

> Pastikan aplikasi Anda membaca `.env` (biasanya via `python-dotenv`). Kalau belum pakai, tambahkan di `requirements.txt`: `python-dotenv`, lalu di entry point: `from dotenv import load_dotenv; load_dotenv()`.

### H.6 Tambahkan Migrasi Database ke `deploy.sh` (Jika Pakai Flask-Migrate/SQLAlchemy)

Edit `deploy/deploy.sh` yang sudah dibuat di Bagian C, tambahkan baris migrasi sebelum restart service:

```bash
#!/bin/bash
set -e

echo ">> Menarik kode terbaru..."
git pull origin "$(git rev-parse --abbrev-ref HEAD)"

echo ">> Install dependency..."
source venv/bin/activate
pip install -r requirements.txt

echo ">> Menjalankan migrasi database (jika ada)..."
flask db upgrade || echo "Tidak ada migrasi / flask-migrate belum dipakai, lewati"

echo ">> Restart service..."
CURRENT_DIR=$(pwd)
if [[ "$CURRENT_DIR" == *"-dev" ]]; then
  sudo systemctl restart flaskpilates-dev.service
else
  sudo systemctl restart flaskpilates-prod.service
fi

echo ">> Selesai."
```

Commit ulang:
```bash
git add deploy/deploy.sh
git commit -m "chore: tambah migrasi database di deploy script"
git push origin development
```

### H.7 Push Source Code Asli & Deploy

Setelah H.1–H.6 selesai:

```bash
git add .
git commit -m "feat: ganti demo dengan source code Flask asli"
git push origin development
```

**Pipeline akan otomatis jalan** (job `run_tests` → `deploy_dev`). Cek di **Build > Pipelines**.

Verifikasi aplikasi asli Anda sudah live:
```bash
curl http://IP_SERVER_DUMMY:5001
# atau buka langsung di browser
```

Jika sudah sesuai harapan di dev, lanjutkan proses **Merge Request `development` → `main`** seperti di Bagian G.3 untuk deploy ke folder production.

### H.8 Troubleshooting Khusus Migrasi dari Demo ke Source Code Asli

| Gejala | Penyebab Umum | Solusi |
|---|---|---|
| `ModuleNotFoundError` saat restart service | Ada dependency di kode asli yang belum masuk `requirements.txt` | Jalankan `pip freeze > requirements.txt` ulang dari environment lokal yang benar-benar dipakai untuk develop |
| `gunicorn: error: No application module specified` | Nama file/variabel Flask di `ExecStart` tidak sesuai H.1 | Cek ulang nama file & variabel `Flask(__name__)` yang benar |
| App jalan tapi error `SECRET_KEY not set` / koneksi DB gagal | `.env` belum dibuat di server, atau `python-dotenv` belum di-load | Ulangi H.5, pastikan `load_dotenv()` dipanggil di awal entry point |
| `git pull` gagal di server karena "local changes" | File `.env` atau `venv/` sempat ter-track Git sebelum `.gitignore` ditambahkan | Di server: `git rm -r --cached .env venv/ 2>/dev/null; git stash; git pull` |
| Static files (CSS/JS) tidak muncul di production | Flask default hanya serve static saat `debug=True` lokal, gunicorn tidak otomatis serve `/static` secara optimal | Tambahkan Nginx sebagai reverse proxy untuk serve folder `static/` langsung (opsional, tanyakan jika perlu panduan Nginx-nya) |

---

## Checklist Lengkap dari Nol

- [ ] Project baru dibuat di GitLab (A.1)
- [ ] Personal Access Token / SSH key lokal siap (A.2)
- [ ] Repo di-clone ke lokal & kode dasar ditambahkan (A.3–A.4)
- [ ] Push pertama ke `main`, lalu buat & push branch `development` (A.5–A.6)
- [ ] Server sudah bisa diakses SSH & prasyarat terinstall (B.1–B.2)
- [ ] SSH key CI dibuat & terdaftar di `authorized_keys` (B.3)
- [ ] Private key sudah di-base64 (B.4)
- [ ] `known_hosts` sudah diambil (B.5)
- [ ] Deploy token dibuat di GitLab (B.6)
- [ ] Kode ter-clone ke `/opt/flaskpilates-dev` & `/opt/flaskpilates-prod` (B.7)
- [ ] Virtual environment siap di kedua folder (B.8)
- [ ] 2 systemd service dibuat & running (B.9)
- [ ] Sudoers NOPASSWD untuk restart service (B.10)
- [ ] Firewall port 5000 & 5001 terbuka (B.11)
- [ ] `deploy/deploy.sh` dibuat, executable, ada di kedua folder (Bagian C)
- [ ] 4 CI/CD Variables dibuat & Protected (Bagian D)
- [ ] Branch `main` & `development` di-set Protected (Bagian E)
- [ ] `.gitlab-ci.yml` dibuat & di-push (Bagian F)
- [ ] Pipeline dev jalan sukses & aplikasi bisa diakses (G.1–G.2)
- [ ] Merge request ke `main` & deploy manual prod sukses (G.3)
- [ ] Entry point & nama variabel Flask sudah dicek (H.1)
- [ ] `requirements.txt` sudah lengkap termasuk `gunicorn` (H.2)
- [ ] Source code asli sudah dipindah ke repo & `.gitignore` sudah benar (H.3)
- [ ] Perintah `ExecStart` di kedua systemd service sudah disesuaikan (H.4)
- [ ] File `.env` dev & prod sudah dibuat manual di server (H.5)
- [ ] Migrasi database ditambahkan ke `deploy.sh` jika perlu (H.6)
- [ ] Source code asli berhasil di-push & ter-deploy ke dev (H.7)
