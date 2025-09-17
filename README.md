# PT. Mahang Diamond Perkasa

Website resmi **PT. Mahang Diamond Perkasa**  
Solusi aplikasi offline untuk manajemen rumah ibadah 

---

## 📌 Tentang Kami
PT. Mahang Diamond Perkasa adalah perusahaan yang berfokus pada pengembangan solusi teknologi, khususnya aplikasi offline yang dapat membantu pengelolaan administrasi, keuangan, dan kegiatan rumah ibadah aplikasi kami dari PT. Mahang Diamond Perkasa – sebuah solusi manajemen rumah ibadah offline.
Dengan aplikasi ini, pengelolaan keuangan, jadwal ibadah, dan kegiatan pengajian menjadi lebih mudah dan rapi, tanpa perlu koneksi internet.
Aplikasi ini juga mendukung multi-user dengan hak akses berbeda, sehingga bendahara, admin, dan anggota bisa bekerja sesuai perannya.
Semua data tersimpan dengan di dukung flaskdisk dan laporan bisa dibuat otomatis dan cepat.
Singkatnya, aplikasi kami menghadirkan kemudahan, keamanan, dan efisiensi untuk pengelolaan rumah ibadah modern.

---

## 🎯 Fitur Utama
- ✅ Aplikasi offline, tidak memerlukan koneksi internet.
- ✅ Pencatatan keuangan rumah ibadah
- ✅ Jadwal kegiatan ibadah, pengajian, dan acara khusus.
- ✅ Laporan otomatis dan rapi.
- ✅ Multi pengguna dengan hak akses berbeda.

---

## 🛠️ Teknologi yang Digunakan
- HTML, CSS, dan JavaScript untuk antarmuka pengguna.
- Database lokal untuk penyimpanan data offline.
- GitHub Pages untuk dokumentasi dan publikasi.

---

## 📥 Instalasi
1. **Download** aplikasi dari link resmi kami.
2. Ekstrak file `.zip` hasil unduhan.
3. Jalankan aplikasi dengan membuka file `index.html` di browser.
novapro/
 ├─ lib/
 │   ├─ main.dart
 │   ├─ screens/
 │   │   ├─ home_screen.dart
 │   │   ├─ finance_screen.dart
 │   │   ├─ schedule_screen.dart
 │   │   └─ staff_screen.dart
 │   ├─ models/
 │   │   ├─ transaction.dart
 │   │   └─ staff.dart
 │   └─ db/
 │       └─ database_helper.dart
 ├─ pubspec.yaml
 └─ README.md
---name: novapro
description: NOVAPRO - Manajemen Rumah Ibadah (Kas, Jadwal, Struktur, dll.)
publish_to: 'none'

environment:
  sdk: ">=2.19.0 <4.0.0"

dependencies:
  flutter:
    sdk: flutter
  sqflite: ^2.3.0
  path_provider: ^2.0.15
  provider: ^6.0.5

dev_dependencies:
  flutter_test:
    sdk: flutter

flutter:
  uses-material-design: true
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
          'amount': double.parse(_amountController.text),
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

## 📄 Lisensi
Hak cipta © 2025 PT. Mahang Diamond Perkasa.  
Dilarang memperbanyak atau menggunakan tanpa izin resmi.

---

## 📞 Kontak
- 📧 Email: **info@mahangdiamondperkasa.co.id**
- 🌐 Website: **https://pt.mahangdiamondperkasa.github.io**
- 📱 WhatsApp: **+62-8xxx-xxxx-xxx**

---

**PT. Mahang Diamond Perkasa – Solusi Aplikasi Offline untuk rumah ibadah**# pt.mahangdiamonperkasa.github.io
Website resmi PT.mahang diamond perkasa-solusi aplikasi offline untuk rumah ibadah
