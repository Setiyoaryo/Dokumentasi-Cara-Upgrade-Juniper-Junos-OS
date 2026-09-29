# Juniper Junos OS Upgrade Guide — MX204

> **Practical upgrade runbook** untuk MX204 VM Host, berdasarkan dokumentasi Juniper dan hasil upgrade yang benar-benar dijalankan pada lab/production workflow.
>
> Target contoh pada dokumentasi ini:
>
> - **Device:** MX204
> - **Hostname:** `Router`
> - **Current Junos:** `20.4R2.7`
> - **Target Junos:** `22.2R3-S7.4`
> - **Package:** `/var/tmp/junos-vmhost-install-mx-x86-64-22.2R3-S7.4.tgz`

---

## Table of Contents

- [1. Overview](#1-overview)
- [2. Important: VM Host vs System Software](#2-important-vm-host-vs-system-software)
- [3. Prerequisites](#3-prerequisites)
- [4. Step 1 — Clean Storage](#4-step-1--clean-storage)
- [5. Step 2 — Transfer the Junos Package](#5-step-2--transfer-the-junos-package)
- [6. Step 3 — Verify SSH Access](#6-step-3--verify-ssh-access)
- [7. Step 4 — Check Current System State](#7-step-4--check-current-system-state)
- [8. Step 5 — Verify the Package in /var/tmp](#8-step-5--verify-the-package-in-vartmp)
- [9. Step 6 — Snapshot Before Upgrade](#9-step-6--snapshot-before-upgrade)
- [10. Step 7 — First Installation Attempt](#10-step-7--first-installation-attempt)
- [11. Step 8 — Correct VM Host Installation](#11-step-8--correct-vm-host-installation)
- [12. Step 9 — Handle Validation Failure](#12-step-9--handle-validation-failure)
- [13. Step 10 — Successful Installation](#13-step-10--successful-installation)
- [14. Step 11 — Reboot](#14-step-11--reboot)
- [15. Step 12 — Post-Upgrade Verification](#15-step-12--post-upgrade-verification)
- [16. Troubleshooting](#16-troubleshooting)
- [17. Reference](#17-reference)

---

# 1. Overview

MX204 menggunakan **VM Host architecture**. Karena itu, upgrade package yang digunakan untuk host software + Junos OS adalah:

`junos-vmhost-install-<platform>-x86-64-<release>.tgz`

Untuk package VM Host, Juniper mendokumentasikan penggunaan:

```text
request vmhost software add <package>
```

Juniper menjelaskan bahwa `junos-vmhost-install-x.tgz` meng-upgrade **host software dan Junos OS**, dan penggunaan VM Host package direkomendasikan untuk workflow tersebut.

Reference: [Juniper — request vmhost software add](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-vmhost-software-add.html)

---

# 2. Important: VM Host vs System Software

## Jangan tertukar

Untuk MX204 dengan VM Host:

| Package | Command | Kegunaan |
|---|---|---|
| `junos-install-*.tgz` | `request system software add` | Junos OS package biasa |
| `junos-vmhost-install-*.tgz` | `request vmhost software add` | **Junos OS + VM Host software** |

Contoh package pada dokumentasi ini:

```text
junos-vmhost-install-mx-x86-64-22.2R3-S7.4.tgz
```

Maka command yang benar:

```bash
request vmhost software add /var/tmp/junos-vmhost-install-mx-x86-64-22.2R3-S7.4.tgz
```

### Pengalaman troubleshooting

Percobaan pertama menggunakan:

```bash
request system software add /var/tmp/junos-vmhost-install-mx-x86-64-22.2R3-S7.4.tgz
```

menghasilkan:

```text
warning: Package /var/tmp/junos-vmhost-install-mx-x86-64-22.2R3-S7.4.tgz not supported, please try vmhost command
```

Artinya package tersebut memang harus diproses dengan **VM Host command**.

Juniper mendokumentasikan bahwa package `junos-vmhost-install-x.tgz` dipasang menggunakan `request vmhost software add`.

---

# 3. Prerequisites

Sebelum upgrade:

- Pastikan akses **console/OOB** tersedia.
- Pastikan SSH management dapat digunakan.
- Pastikan konfigurasi sudah tersimpan.
- Pastikan ada ruang penyimpanan yang cukup.
- Pastikan package Junos sesuai dengan **platform MX204**.
- Pastikan hostname/device yang akan di-upgrade sudah benar.
- Hindari upgrade saat ada pekerjaan maintenance lain yang berjalan.

Contoh device:

```text
Hostname : Router
Model    : mx204
Current  : 20.4R2.7
Target   : 22.2R3-S7.4
```

---

# 4. Step 1 — Clean Storage

Langkah pertama yang digunakan pada workflow ini adalah membersihkan file-file log/archive yang tidak diperlukan.

Jalankan:

```bash
request system storage cleanup
```

Command ini membantu membersihkan file yang dapat dihapus oleh Junos sehingga ruang pada filesystem menjadi lebih lega.

Contoh hasil:

```text
List of files to delete:

48.7K Sep 23 20:40 /var/log/apsd.0.gz
48.8K Sep 23 14:10 /var/log/apsd.1.gz
...
62.0K Aug 20 15:45 /var/log/jddosd.0.gz
...
```

Setelah cleanup, cek kembali storage:

```bash
show system storage
```

Contoh kondisi setelah cleanup:

```text
Filesystem          Size   Used   Avail  Capacity  Mounted on
/dev/gpt/junos      10.0G  2.5G   6.7G   27%       /..mount
/dev/gpt/config     952M   108K   876M   0%        /..mount/config
/dev/gpt/var        30G    197M   27G    1%        /..mount/var
tmpfs               10.0G  168K   10.0G  0%        /..mount/tmp
tmpfs                2.7G  896K    2.7G  0%        /..mount/mfs
```

> **Catatan:** Jangan menghapus file secara manual dari filesystem sistem kecuali memang diperlukan dan sudah dipahami. Gunakan command Junos terlebih dahulu.

---

# 5. Step 2 — Transfer the Junos Package

Package dapat dikirim ke MX204 menggunakan SCP.

Target directory:

```text
/var/tmp
```

Contoh dari workstation/Linux:

```bash
scp junos-vmhost-install-mx-x86-64-22.2R3-S7.4.tgz user@<MGMT-IP>:/var/tmp/
```

Contoh:

```bash
scp junos-vmhost-install-mx-x86-64-22.2R3-S7.4.tgz aryo@192.0.2.10:/var/tmp/
```

Juniper juga merekomendasikan software image ditempatkan di `/var/tmp`.

> **Penting:** Pastikan SSH dari workstation ke management IP MX204 benar-benar dapat diakses sebelum transfer.

---

# 6. Step 3 — Verify SSH Access

Sebelum mengirim package, pastikan port SSH terbuka.

Dari Linux:

```bash
nc -vz <MGMT-IP> 22
```

atau:

```bash
ssh -v aryo@<MGMT-IP>
```

Yang perlu dipastikan:

```text
TCP/22 reachable
SSH authentication works
User memiliki permission yang diperlukan
```

Pada platform VM Host seperti MX204, Juniper juga mendokumentasikan kebutuhan **root SSH access untuk proses internal tertentu selama upgrade/reboot**. Untuk konfigurasi SSH, Juniper menyebut penggunaan `root-login deny-password` selama proses ini agar trusted internal communication tetap dapat berlangsung, kemudian konfigurasi dapat dikembalikan sesuai kebijakan setelah upgrade.

Reference:
[Juniper — Installing, Upgrading, Backing Up, and Recovery of VM Host](https://www.juniper.net/documentation/us/en/software/junos/junos-install-upgrade/topics/concept/installation-upgrade-vmhost.html)

---

# 7. Step 4 — Check Current System State

Sebelum melakukan upgrade, catat kondisi perangkat.

## Check Junos Version

```bash
show version
```

Contoh:

```text
Hostname: Router
Model: mx204
Junos: 20.4R2.7
JUNOS OS Kernel 64-bit [20210220.a5d6a89_builder_stable_11]
...
```

## Check License

```bash
show license
```

Simpan output tersebut sebagai baseline sebelum upgrade.

## Check Hardware

```bash
show chassis hardware
```

## Optional: Check Alarms

```bash
show chassis alarms
show system alarms
```

---

# 8. Step 5 — Verify the Package in /var/tmp

Pastikan package benar-benar tersedia.

```bash
file list /var/tmp/
```

atau:

```bash
file list /var/tmp/junos-vmhost-install-mx-x86-64-22.2R3-S7.4.tgz
```

Pastikan nama file **sama persis** dengan package yang ingin digunakan:

```text
/var/tmp/junos-vmhost-install-mx-x86-64-22.2R3-S7.4.tgz
```

Bila tersedia checksum dari Juniper Software Download Center, verifikasi checksum sebelum instalasi.

---

# 9. Step 6 — Snapshot Before Upgrade

Jika device sudah berada pada kondisi stabil, Juniper menyarankan membuat VM Host snapshot sebelum upgrade:

```bash
request vmhost snapshot
```

Snapshot berfungsi sebagai backup/recovery point dari software environment VM Host.

> **Penting:** Setelah upgrade selesai dan versi baru sudah dinyatakan stabil, snapshot dapat dibuat kembali untuk menyimpan kondisi versi baru. Snapshot baru akan menggantikan snapshot sebelumnya.

Reference:
[Juniper — request vmhost software add](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-vmhost-software-add.html)

---

# 10. Step 7 — First Installation Attempt

## Percobaan 1: menggunakan system software command

Percobaan:

```bash
request system software add /var/tmp/junos-vmhost-install-mx-x86-64-22.2R3-S7.4.tgz
```

Hasil:

```text
warning: Package /var/tmp/junos-vmhost-install-mx-x86-64-22.2R3-S7.4.tgz
not supported, please try vmhost command
```

### Kesimpulan

Package tersebut adalah **VM Host package**, sehingga command yang digunakan harus:

```bash
request vmhost software add
```

---

# 11. Step 8 — Correct VM Host Installation

Jalankan:

```bash
request vmhost software add /var/tmp/junos-vmhost-install-mx-x86-64-22.2R3-S7.4.tgz
```

Pada package dengan release berbeda, Junos secara default menjalankan validation terhadap konfigurasi.

Secara konseptual:

```text
VM Host package
      │
      ▼
request vmhost software add
      │
      ▼
package verification
      │
      ▼
validate configuration
      │
      ▼
install
```

Juniper mendokumentasikan bahwa `no-validate` digunakan untuk men-suppress default validation ketika loading package dengan release berbeda.

Reference:
[Juniper — request vmhost software add](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-vmhost-software-add.html)

---

# 12. Step 9 — Handle Validation Failure

Pada percobaan pertama dengan validation default, proses berhenti pada tahap validation.

Output:

```text
Junos Validation begin. Procedure will take few minutes.

Mounting dsa-x86-64-22.2R3-S7.4

chroot: pwd_mkdb: No such file or directory

Hardware Database regeneration succeeded

Validating against /config/juniper.conf.gz

Abort trap (core dumped)

Validation failed

ERROR: Failed to add
/var/tmp/junos-vmhost-install-mx-x86-64-22.2R3-S7.4.tgz

warning: Host software installation has failed.
```

## Analisis

Ada beberapa hal penting dari output:

```text
Package signature verification  → BERHASIL
Package mounting               → BERHASIL
Hardware DB regeneration      → BERHASIL
Configuration validation      → GAGAL
```

Kegagalan terjadi ketika package divalidasi terhadap:

```text
/config/juniper.conf.gz
```

dan proses validation berakhir:

```text
Abort trap (core dumped)
```

---

# 13. Step 10 — Successful Installation

Karena validation default gagal pada workflow ini, instalasi dilanjutkan dengan opsi:

```bash
request vmhost software add /var/tmp/junos-vmhost-install-mx-x86-64-22.2R3-S7.4.tgz no-validate
```

> **Catatan penting:** `no-validate` berarti proses **configuration validation prerequisite** dilewati. Ini bukan berarti signature verification atau seluruh integrity checking package dimatikan.

Output keberhasilan pada workflow ini berakhir dengan:

```text
0 ... upgrade complete.

A REBOOT IS REQUIRED TO LOAD THIS SOFTWARE CORRECTLY.

Use the 'request vmhost reboot' command to reboot the system.
```

Dengan kata lain:

```text
Package verification  → OK
Package mounting      → OK
Hardware DB           → OK
Validation            → SKIPPED
Installation          → SUCCESS
Reboot                → REQUIRED
```

---

# 14. Step 11 — Reboot

Setelah software add berhasil:

```bash
request vmhost reboot
```

Juniper secara eksplisit menggunakan `request vmhost reboot` pada workflow VM Host.

Reference:
[Juniper — VM Host Installation and Upgrade](https://www.juniper.net/documentation/us/en/software/junos/junos-install-upgrade/topics/concept/installation-upgrade-vmhost.html)

### Production reminder

Reboot akan memutus traffic/control-plane. Pastikan:

- console/OOB tersedia
- maintenance window tersedia
- BGP/IGP/MPLS convergence dipahami
- upstream/peer notification sudah dilakukan bila diperlukan

---

# 15. Step 12 — Post-Upgrade Verification

Setelah router kembali online, lakukan verification berurutan.

## 15.1 Verify Version

```bash
show version
```

Target:

```text
Junos: 22.2R3-S7.4
```

## 15.2 Verify Hardware

```bash
show chassis hardware
```

## 15.3 Verify Alarms

```bash
show chassis alarms
show system alarms
```

## 15.4 Verify Routing

```bash
show route summary
```

## 15.5 Verify BGP

```bash
show bgp summary
```

Pastikan session utama kembali ke state:

```text
Established
```

## 15.6 Verify OSPF

```bash
show ospf neighbor
```

## 15.7 Verify MPLS / LDP

Jika device menggunakan MPLS:

```bash
show ldp neighbor
show mpls interface
```

## 15.8 Verify Interfaces

```bash
show interfaces terse
```

Lakukan pengecekan untuk memastikan interface penting kembali `up/up`.

---

# 16. Troubleshooting

## Error: package not supported, please try vmhost command

### Error

```text
warning: Package ... not supported, please try vmhost command
```

### Penyebab

Package yang digunakan merupakan:

```text
junos-vmhost-install-*.tgz
```

### Solusi

Gunakan:

```bash
request vmhost software add /var/tmp/<package>.tgz
```

Bukan:

```bash
request system software add /var/tmp/<package>.tgz
```

---

## Error: Validation failed / Abort trap

### Error

```text
Validating against /config/juniper.conf.gz

Abort trap (core dumped)

Validation failed
```

### Solusi pada workflow ini

Gunakan:

```bash
request vmhost software add /var/tmp/<package>.tgz no-validate
```

Namun, gunakan `no-validate` **hanya ketika sesuai dengan release/platform requirements dan dokumentasi Juniper**.

---

## Error: SSH / root access issue

Untuk MX204 VM Host, Juniper mencatat bahwa root SSH access diperlukan selama bagian tertentu dari upgrade/reboot untuk komunikasi internal antara Junos VM dan Linux host.

Pastikan konfigurasi tidak menghalangi kebutuhan tersebut. Juniper mendokumentasikan penggunaan:

```text
root-login deny-password
```

untuk kebutuhan ini selama upgrade, lalu lakukan restore konfigurasi SSH setelah proses selesai.

Reference:
[Juniper — VM Host Installation](https://www.juniper.net/documentation/us/en/software/junos/junos-install-upgrade/topics/concept/installation-upgrade-vmhost.html)

---

# 17. Reference

## Official Juniper Documentation

### VM Host Software Add

[Juniper — request vmhost software add](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-vmhost-software-add.html)

Dokumentasi command ini menjelaskan:

- `request vmhost software add`
- package `junos-vmhost-install-x.tgz`
- `no-validate`
- snapshot sebelum upgrade
- reboot setelah software add

### VM Host Installation / Upgrade

[Juniper — Installing, Upgrading, Backing Up, and Recovery of VM Host](https://www.juniper.net/documentation/us/en/software/junos/junos-install-upgrade/topics/concept/installation-upgrade-vmhost.html)

Dokumentasi ini menjelaskan workflow install/upgrade VM Host dan behavior platform-specific termasuk MX204.

### System Software Add

[Juniper — request system software add](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-software-add.html)

Digunakan untuk package Junos biasa. Untuk VM Host package pada MX204, gunakan workflow `request vmhost software add`.

---

# Upgrade Flow Summary

```text
                    ┌───────────────────────┐
                    │  MX204 / BR-SBY...    │
                    │     20.4R2.7          │
                    └───────────┬───────────┘
                                │
                                ▼
                 request system storage cleanup
                                │
                                ▼
                       Check system storage
                                │
                                ▼
                     Transfer package via SCP
                                │
                                ▼
                            /var/tmp
                                │
                                ▼
                      show version / license
                                │
                                ▼
                       Verify package exists
                                │
                                ▼
                     request vmhost snapshot
                                │
                                ▼
             request system software add
                                │
                                ▼
                   ❌ NOT SUPPORTED
                                │
                                ▼
              request vmhost software add
                                │
                                ▼
                     ❌ VALIDATION FAILED
                                │
                         no-validate
                                │
                                ▼
              request vmhost software add
                       ... no-validate
                                │
                                ▼
                       ✅ upgrade complete
                                │
                                ▼
                    request vmhost reboot
                                │
                                ▼
                        Router reboot
                                │
                                ▼
                   Post-upgrade checks
                                │
                                ├─ show version
                                ├─ show chassis alarms
                                ├─ show route summary
                                ├─ show bgp summary
                                ├─ show ospf neighbor
                                └─ show ldp neighbor
```

---

## Example Command Sequence

```bash
# 1. Cleanup
request system storage cleanup

# 2. Check storage
show system storage

# 3. Check current version/license
show version
show license

# 4. Verify package
file list /var/tmp/

# 5. Snapshot
request vmhost snapshot

# 6. WRONG for VM Host package
request system software add /var/tmp/junos-vmhost-install-mx-x86-64-22.2R3-S7.4.tgz

# 7. Correct command
request vmhost software add /var/tmp/junos-vmhost-install-mx-x86-64-22.2R3-S7.4.tgz

# 8. If validation fails and no-validate is appropriate
request vmhost software add /var/tmp/junos-vmhost-install-mx-x86-64-22.2R3-S7.4.tgz no-validate

# 9. Reboot
request vmhost reboot

# 10. Post-check
show version
show chassis hardware
show chassis alarms
show system alarms
show route summary
show bgp summary
show ospf neighbor
show ldp neighbor
```

---

> **Operational note:** Jangan menjadikan `no-validate` sebagai langkah otomatis untuk semua MX/Junos upgrades. Selalu cek release notes dan platform-specific upgrade requirements untuk source release → target release terlebih dahulu.
