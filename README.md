# 🛡️ Enterprise-Grade Mobile SOC Home Lab

**Status Proyek:** Selesai & Beroperasi
**Fokus Utama:** SIEM, Network IDS, VPN Mesh Overlay, Detection Engineering

---

## 📌 Deskripsi Proyek
Proyek ini bertujuan membangun ekosistem *Security Operations Center* (SOC) skala lab yang portabel dan tahan terhadap perubahan infrastruktur jaringan. Fokus utama proyek ini adalah optimasi *pipeline* log, *Detection Engineering* untuk berbagai vektor serangan (Web, Network, Auth, FIM), serta penerapan arsitektur *Zero-Trust Network* menggunakan VPN Mesh.

> **Catatan Portofolio:** Infrastruktur lab ini dirancang khusus agar tetap beroperasi secara *real-time* meskipun berpindah jaringan Wi-Fi publik (portabel), mensimulasikan lingkungan *Enterprise Site-to-Site*.

---

## 🏗️ Topologi & Arsitektur Jaringan
Untuk mengatasi masalah IP Dinamis (DHCP) yang sering memutus koneksi antara Agent dan SIEM, lab ini dihubungkan menggunakan protokol WireGuard melalui **Tailscale Overlay Network**.

* **Wazuh Manager (SIEM & Otak Deteksi):** Bertindak sebagai sentral pengumpulan log.
* **Wazuh Agent (Target Serangan):** Berjalan di dalam Virtual Machine (Ubuntu) menggunakan konfigurasi jaringan NAT.
* **Konektivitas VPN Mesh:** Kedua *node* diberikan IP Statis permanen (`100.x.x.x`) oleh Tailscale. Hal ini memastikan Agent selalu berstatus **Active** dan log serangan selalu terkirim dengan aman menembus NAT/Firewall jaringan lokal.

*(Tempatkan diagram arsitektur atau screenshot topologi Anda di sini)*
`![Diagram Arsitektur](link-gambar-anda.png)`

---

## ⚙️ System Tuning & Optimasi Limitasi Log
Log tingkat lanjut yang dihasilkan oleh Suricata (Network IDS) memiliki *fields* JSON yang sangat besar. Secara standar, hal ini menyebabkan *error* `Too many fields for JSON decoder` pada *engine* Wazuh dan memblokir log masuk.

**Resolusi Teknis:**
Memperbesar kapasitas *buffer* JSON secara manual di mesin Manager.
* **File Konfigurasi:** `/var/ossec/etc/local_internal_options.conf`
* **Parameter yang dioptimasi:**
  ```text
  analysisd.max_json_decode_fields=8192
  analysisd.decode_recursion_level=30
