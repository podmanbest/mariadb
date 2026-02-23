# Podman Rootless: Zero to Hero — Membangun Image Container

Panduan langkah demi langkah membangun image container dengan **Podman rootless** tanpa perlu root/sudo.

---

## 1. Apa itu Podman Rootless?

- **Podman** = daemonless container engine (seperti Docker, tapi tanpa daemon).
- **Rootless** = menjalankan container sebagai user biasa, bukan root.
- **Keuntungan**: lebih aman, tidak perlu sudo, cocok untuk dev dan CI.

---

## 2. Instalasi Podman

### Windows (WSL2 atau Native)

```powershell
# Pakai winget (Windows)
winget install RedHat.Podman

# Atau unduh installer dari: https://podman-desktop.io/ atau https://podman.io/
```

Setelah instalasi, pastikan `podman` ada di PATH:

```powershell
podman --version
```

### Linux

```bash
# Fedora
sudo dnf install podman

# Ubuntu/Debian
sudo apt update && sudo apt install podman

# Arch
sudo pacman -S podman
```

### macOS

```bash
brew install podman
podman machine init
podman machine start
```

---

## 3. Cek Mode Rootless

Podman secara default berjalan rootless di Linux. Cek:

```bash
podman info | grep -i rootless
# rootless: true
```

Di Windows/macOS, Podman berjalan di VM (rootless by design untuk user space).

---

## 4. Registri & Login (Opsional)

Untuk push image ke registri (Docker Hub, quay, dll.):

```bash
# Login Docker Hub (rootless, simpan kredensial di home user)
podman login docker.io

# Login Quay
podman login quay.io
```

Kredensial disimpan di `~/.config/containers/auth.json` (user-level).

---

## 5. Build Image — Dasar

### 5.1 Dockerfile Sederhana

Buat file `Dockerfile` di direktori proyek:

```dockerfile
# Contoh: image Node.js
FROM docker.io/library/node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

### 5.2 Build dengan Podman (rootless)

```bash
# Build image (tanpa sudo)
podman build -t myapp:latest .

# Build dengan nama lengkap untuk push ke registri
podman build -t docker.io/username/myapp:v1.0 .
```

- Image disimpan di storage rootless (`~/.local/share/containers/storage`).
- Tidak perlu akses root.

---

## 6. Build Lanjutan

### 6.1 Multi-stage build

```dockerfile
# Stage 1: build
FROM docker.io/library/golang:1.21-alpine AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app/server .

# Stage 2: runtime
FROM docker.io/library/alpine:latest
RUN apk add --no-cache ca-certificates
COPY --from=builder /app/server /server
EXPOSE 8080
CMD ["/server"]
```

```bash
podman build -t myapp:latest .
```

### 6.2 Build args & target

```bash
# Dengan build-arg
podman build --build-arg NODE_ENV=production -t myapp:prod .

# Build stage tertentu (multi-stage)
podman build --target builder -t myapp-builder .
```

### 6.3 Build tanpa cache

```bash
podman build --no-cache -t myapp:latest .
```

### 6.4 Dockerfile di path lain

```bash
podman build -f path/to/Dockerfile -t myapp:latest path/to/context
```

---

## 7. Menjalankan Container dari Image

```bash
# Run di foreground
podman run --rm -p 8080:8080 myapp:latest

# Run di background (detached)
podman run -d --name myapp -p 8080:8080 myapp:latest

# Lihat log
podman logs -f myapp

# Stop & hapus
podman stop myapp && podman rm myapp
```

---

## 8. Push Image ke Registri (Rootless)

```bash
# Tag untuk registri
podman tag myapp:latest docker.io/username/myapp:v1.0

# Push (pakai kredensial dari podman login)
podman push docker.io/username/myapp:v1.0
```

---

## 9. Podman vs Docker (CLI)

| Docker (root)     | Podman (rootless) |
|-------------------|--------------------|
| `docker build`    | `podman build`     |
| `docker run`      | `podman run`       |
| `docker push`     | `podman push`      |
| `docker images`   | `podman images`    |
| `docker ps`       | `podman ps`        |

Alias agar bisa pakai perintah `docker`:

```bash
alias docker=podman
# Atau symlink (Linux): sudo ln -s /usr/bin/podman /usr/bin/docker
```

---

## 10. Tips Rootless

1. **Port & binding**: Port &lt; 1024 di host kadang butuh konfigurasi (mis. `sysctl net.ipv4.ip_unprivileged_port_start=80`).
2. **Storage**: Semua data (images, containers) di `~/.local/share/containers/storage` — backup folder ini untuk backup “semua image/container” user.
3. **Rootful vs rootless**: Untuk development dan kebanyakan workload, rootless cukup. Jika butuh fitur khusus (mis. beberapa network mode), cek dokumentasi Podman.

---

## 11. Ringkasan Perintah Penting

```bash
podman build -t NAMA:TAG .          # Build image
podman images                       # Daftar image
podman run -d -p HOST:CONTAINER NAMA # Jalankan container
podman push REGISTRY/NAMA:TAG       # Push ke registri
podman login REGISTRY                # Login registri (rootless, user-level)
```

Dengan ini Anda bisa **membangun image container dengan Podman rootless** dari nol sampai siap dipakai dan di-push ke registri. Jika mau, langkah berikutnya bisa: Dockerfile untuk MariaDB, compose dengan Podman, atau integrasi CI (GitHub Actions dengan Podman).
