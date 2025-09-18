# PT. Mahang Diamond Perkasa

Website resmi **PT. Mahang Diamond Perkasa**  
Solusi aplikasi offline untuk manajemen rumah ibadah 

---

## 📌 Tentang Kami
PT. Mahang Diamond Perkasa adalah perusahaan yang berfokus pada pengembangan solusi teknologi, khususnya aplikasi offline untuk manajemen rumah ibadah.  

Aplikasi **NovaPro** kami dirancang agar pengelolaan administrasi, keuangan, jadwal ibadah, dan kegiatan pengajian menjadi lebih mudah, rapi, dan **tanpa perlu koneksi internet**.  

Fitur multi-user dengan hak akses berbeda (bendahara, admin, ketua) memastikan setiap petugas bisa bekerja sesuai perannya. Semua data tersimpan aman di perangkat lokal/flashdisk dan laporan dapat dibuat otomatis dengan cepat.  

Singkatnya, aplikasi ini menghadirkan **kemudahan, keamanan, dan efisiensi** bagi pengelolaan rumah ibadah modern.
---


<h2>🎯 Fitur Utama</h2>
<pre id="fiturUtama">
✅ Penyimpanan transaksi permanen menggunakan SQLite
✅ Multi-user dengan hak akses berbeda (bendahara, admin, ketua)
✅ Backup & Restore data lokal
✅ Ekspor laporan otomatis (PDF/CSV)
✅ ScheduleScreen & StaffScreen lengkap
✅ Versi Hackathon: FinanceScreen & HomeScreen siap demo, Schedule & Staff masih placeholder
✅ Aplikasi offline sepenuhnya
✅ Pencatatan keuangan rumah ibadah
✅ Jadwal kegiatan ibadah, pengajian, dan acara khusus
✅ Laporan otomatis yang rapi dan bisa diekspor
✅ Struktur petugas fleksibel sesuai kebutuhan rumah ibadah
✅ Tampilan sederhana & mudah digunakan
</pre>
<button onclick="copyFitur()">Salin Fitur Utama</button>

<script>
function copyFitur() {
  const fitur = document.getElementById("fiturUtama").innerText;
  navigator.clipboard.writeText(fitur).then(() => {
    alert("Fitur Utama berhasil disalin!");
  });
}
</script>

---

## 🛠️ Teknologi yang Digunakan

### 📥 Instalasi
1. Clone repo: `git clone https://github.com/pt-mahangdiamondperkasa/novapro.git`
2. Masuk ke folder proyek: `cd novapro`
3. Jalankan: `flutter pub get`
4. Run aplikasi: `flutter run`

- **Flutter (Dart)** → untuk membangun aplikasi lintas platform (Android, iOS, Desktop).  
- **SQLite (sqflite package)** → database lokal untuk pencatatan kas & jadwal secara offline.  
- **Provider** → state management agar aplikasi lebih stabil & mudah dikembangkan.  
- **Path Provider** → untuk menyimpan dan mengakses file lokal (backup data).  
- **Material Design** → tampilan antarmuka yang sederhana & modern.  
- **GitHub & GitHub Pages** → untuk dokumentasi, publikasi, dan kolaborasi tim.

---

## 📥 Instalasi

## Instalasi

Berikut beberapa tampilan aplikasi (UI/UX):

![Screenshot 1](assets/screenshots/screen1.png)
![Screenshot 2](assets/screenshots/screen2.png)
![Screenshot 3](assets/screenshots/screen3.png)
![Screenshot 4](assets/screenshots/screen4.png)
![Screenshot 5](assets/screenshots/screen5.png)
![Screenshot 6](assets/screenshots/screen6.png)

# 📌 pubspec.yaml
name: novapro
description: Aplikasi offline manajemen rumah ibadah (NOVAPRO)
publish_to: 'none'
version: 1.0.0+1

environment:
  sdk: ">=2.17.0 <3.0.0"

dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.2

dev_dependencies:
  flutter_test:
    sdk: flutter

flutter:
  uses-material-design: true

// ================== main.dart ==================
import 'package:flutter/material.dart';
import 'screens/home_screen.dart';

void main() {
  runApp(const NovaProApp());
}

class NovaProApp extends StatelessWidget {
  const NovaProApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'NOVAPRO',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        primarySwatch: Colors.indigo,
        visualDensity: VisualDensity.adaptivePlatformDensity,
      ),
      home: const HomeScreen(),
    );
  }
}

// ================== screens/home_screen.dart ==================
import 'package:flutter/material.dart';
import 'finance_screen.dart';
import 'schedule_screen.dart';
import 'staff_screen.dart';

class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('NOVAPRO - Rumah Ibadah')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            ElevatedButton(
              child: const Text('💰 Manajemen Kas'),
              onPressed: () {
                Navigator.push(context,
                    MaterialPageRoute(builder: (_) => const FinanceScreen()));
              },
            ),
            ElevatedButton(
              child: const Text('📅 Jadwal Kegiatan'),
              onPressed: () {
                Navigator.push(context,
                    MaterialPageRoute(builder: (_) => const ScheduleScreen()));
              },
            ),
            ElevatedButton(
              child: const Text('👤 Struktur Petugas'),
              onPressed: () {
                Navigator.push(context,
                    MaterialPageRoute(builder: (_) => const StaffScreen()));
              },
            ),
          ],
        ),
      ),
    );
  }
}

// ================== screens/finance_screen.dart ==================
import 'package:flutter/material.dart';

class FinanceScreen extends StatefulWidget {
  const FinanceScreen({super.key});

  @override
  State<FinanceScreen> createState() => _FinanceScreenState();
}

class _FinanceScreenState extends State<FinanceScreen> {
  final List<Map<String, dynamic>> _transactions = [];
  final _formKey = GlobalKey<FormState>();
  final _descController = TextEditingController();
  final _amountController = TextEditingController();

  void _addTransaction() {
    if (_formKey.currentState!.validate()) {
      setState(() {
        _transactions.add({
          'desc': _descController.text,
          'amount': double.tryParse(_amountController.text) ?? 0,
        });
        _descController.clear();
        _amountController.clear();
      });
    }
  }

  double get _balance =>
      _transactions.fold(0, (sum, item) => sum + item['amount']);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text("Manajemen Kas")),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          children: [
            Text("Saldo: Rp $_balance",
                style: const TextStyle(
                    fontWeight: FontWeight.bold, fontSize: 20)),
            Form(
              key: _formKey,
              child: Column(
                children: [
                  TextFormField(
                    controller: _descController,
                    decoration: const InputDecoration(labelText: "Deskripsi"),
                    validator: (v) =>
                        v == null || v.isEmpty ? "Wajib diisi" : null,
                  ),
                  TextFormField(
                    controller: _amountController,
                    decoration:
                        const InputDecoration(labelText: "Jumlah (+/-)"),
                    keyboardType: TextInputType.number,
                    validator: (v) =>
                        v == null || v.isEmpty ? "Wajib diisi" : null,
                  ),
                  const SizedBox(height: 10),
                  ElevatedButton(
                    onPressed: _addTransaction,
                    child: const Text("Tambah Transaksi"),
                  )
                ],
              ),
            ),
            const SizedBox(height: 20),
            Expanded(
              child: ListView.builder(
                itemCount: _transactions.length,
                itemBuilder: (ctx, i) => ListTile(
                  title: Text(_transactions[i]['desc']),
                  trailing: Text(_transactions[i]['amount'].toString()),
                ),
              ),
            )
          ],
        ),
      ),
    );
  }
}

// ================== screens/schedule_screen.dart ==================
import 'package:flutter/material.dart';

class ScheduleScreen extends StatelessWidget {
  const ScheduleScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text("Jadwal Kegiatan")),
      body: const Center(
        child: Text("📅 Fitur Jadwal akan ditambahkan di sini"),
      ),
    );
  }
}

// ================== screens/staff_screen.dart ==================
import 'package:flutter/material.dart';

class StaffScreen extends StatelessWidget {
  const StaffScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text("Struktur Petugas")),
      body: const Center(
        child: Text("👤 Fitur Struktur Petugas akan ditambahkan di sini"),
      ),
    );
  }
}
            

## 📄 Lisensi
MIT License  

Hak Cipta (c) 2025 PT. Mahang Diamond Perkasa  

Dengan ini diizinkan, tanpa biaya, kepada siapa pun yang memperoleh salinan perangkat lunak ini dan file dokumentasi terkait ("Perangkat Lunak"), untuk menggunakan Perangkat Lunak tanpa batasan, termasuk tanpa batasan hak untuk menggunakan, menyalin, mengubah, menggabungkan, menerbitkan, mendistribusikan, mensublisensikan, dan/atau menjual salinan Perangkat Lunak, serta mengizinkan orang yang diberikan Perangkat Lunak untuk melakukannya, dengan syarat berikut:  

Pernyataan hak cipta di atas dan pernyataan izin ini harus disertakan dalam semua salinan atau bagian substansial dari Perangkat Lunak.  

PERANGKAT LUNAK INI DIBERIKAN "SEBAGAIMANA ADANYA", TANPA JAMINAN APA PUN, BAIK TERSURAT MAUPUN TERSIRAT, TERMASUK NAMUN TIDAK TERBATAS PADA JAMINAN DIPERDAGANGKAN, KESESUAIAN UNTUK TUJUAN TERTENTU, DAN BEBAS DARI PELANGGARAN. DALAM KEADAAN APA PUN PENULIS ATAU PEMEGANG HAK CIPTA TIDAK BERTANGGUNG JAWAB ATAS KLAIM, KERUSAKAN, ATAU KEWAJIBAN LAIN, BAIK DALAM TINDAKAN KONTRAK, PERBUATAN MELAWAN HUKUM, ATAU LAINNYA, YANG TIMBUL DARI, KELUAR DARI, ATAU BERHUBUNGAN DENGAN PERANGKAT LUNAK ATAU PENGGUNAAN ATAU HAL-HAL LAIN DALAM PERANGKAT LUNAK.
---


## 📞 Kontak
- 📧 Email: **info@mahangdiamondperkasa.co.id**  
- 🌐 Website: **https://pt.mahangdiamondperkasa.github.io**  
- 📱 WhatsApp: **+62-812-7020-9570**


**PT. Mahang Diamond Perkasa – Solusi Aplikasi Offline untuk Rumah Ibadah**  
🌐 Website resmi: [pt.mahangdiamondperkasa.github.io](https://pt.mahangdiamondperkasa.github.io)
