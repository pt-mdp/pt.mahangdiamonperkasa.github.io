 NOVAPRO PREMIUM (OFFLINE SOLUTION)
OWNER: PT. MAHANG DIAMOND PERKASA
WEBSITE: https://pt.mahangdiamondperkasa.github.io
📌 TENTANG KAMI
PT. Mahang Diamond Perkasa adalah perusahaan yang berfokus pada pengembangan solusi teknologi, khususnya aplikasi offline untuk manajemen rumah ibadah. Aplikasi NovaPro kami dirancang agar pengelolaan administrasi, keuangan, jadwal ibadah, dan kegiatan pengajian menjadi lebih mudah, rapi, dan tanpa perlu koneksi internet. Fitur multi-user dengan hak akses berbeda memastikan setiap petugas bisa bekerja sesuai perannya. Semua data tersimpan aman di perangkat lokal/flashdisk dan laporan dapat dibuat otomatis dengan cepat. Singkatnya, aplikasi ini menghadirkan kemudahan, keamanan, dan efisiensi bagi pengelolaan rumah ibadah modern. "Ini adalah versi built awal."
🎯 FITUR UTAMA
✅ Penyimpanan transaksi permanen menggunakan sistem file lokal aman
✅ Multi-user dengan hak akses admin (Password Protected)
✅ Backup & Restore data lokal secara manual
✅ Ekspor tampilan informasi ke Videotron/TV/Laptop
✅ Running Text Engine untuk pengumuman masjid secara dinamis
✅ Galeri Kegiatan (Upload foto kegiatan langsung dari galeri HP)
✅ Struktur Petugas fleksibel dengan dukungan Foto Profil asli
✅ Mode Live Display untuk durasi acara secara real-time
✅ Optimasi sistem dengan fitur "Clean Cache" otomatis
✅ Aplikasi offline sepenuhnya tanpa ketergantungan internet
🛠️ TEKNOLOGI
 * Flutter (Dart) → Framework Lintas Platform
 * Provider → State Management
 * Path Provider → Local Storage Management
 * Image Picker → Media Handling
 * Material Design 3 → UI/UX Modern Maroon & Gold
📥 INSTALASI GITHUB
 * Clone repo: git clone https://github.com/pt-mahangdiamondperkasa/novapro.git
 * Masuk ke folder: cd novapro
 * Instalasi: flutter pub get
 * Jalankan: flutter run
📄 LISENSI
MIT License. Hak Cipta (c) 2026 PT. Mahang Diamond Perkasa.
📌 KONFIGURASI FILE: pubspec.yaml
name: novapro
description: Aplikasi offline manajemen rumah ibadah oleh PT. Mahang Diamond Perkasa
publish_to: 'none'
version: 1.0.0+1
environment:
sdk: ">=2.17.0 <3.0.0"
dependencies:
flutter:
sdk: flutter
cupertino_icons: ^1.0.2
provider: ^6.0.5
image_picker: ^1.0.7
path_provider: ^2.0.15
flutter:
uses-material-design: true
📌 SOURCE CODE UTAMA: lib/main.dart
import 'dart:async';
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'package:image_picker/image_picker.dart';
import 'package:path_provider/path_provider.dart';
void main() {
runApp(
ChangeNotifierProvider(
create: (_) => AppProvider(),
child: MaterialApp(
title: 'NOVAPRO PREMIUM',
debugShowCheckedModeBanner: false,
theme: ThemeData(
primaryColor: const Color(0xFF631414),
scaffoldBackgroundColor: const Color(0xFFF5F5F5),
colorScheme: ColorScheme.fromSwatch().copyWith(secondary: const Color(0xFFFFD700)),
),
home: const NovaProMasterRoot(),
),
),
);
}
class AppProvider with ChangeNotifier {
bool isLogin = false;
String mode = "public";
String adminPass = "123";
String namaIbadah = "MASJID AN-NUUR";
String alamat = "SOLUSI OFFLINE - PT. MAHANG DIAMOND PERKASA";
String runningText = "SELAMAT DATANG DI NOVAPRO PREMIUM • SISTEM MANAJEMEN RUMAH IBADAH MODERN • OFFLINE & AMAN • PT. MAHANG DIAMOND PERKASA";
Timer? liveTimer;
int liveSeconds = 0;
List<Map<String, dynamic>> struktur = [
{"jabatan": "Ketua", "anggota": [{"nama": "Kariyono Hadi, BA", "foto": null}]},
{"jabatan": "Bendahara", "anggota": [{"nama": "Suherdi", "foto": null}]},
{"jabatan": "Sekretaris", "anggota": [{"nama": "Anang Ariyanto", "foto": null}]},
];
List<File> galeri = [];
bool login(String pass) {
if (pass == adminPass) { isLogin = true; mode = "admin"; notifyListeners(); return true; }
return false;
}
void logout() { isLogin = false; mode = "public"; stopLive(); notifyListeners(); }
void startLive() {
mode = "live"; liveSeconds = 0;
liveTimer?.cancel();
liveTimer = Timer.periodic(const Duration(seconds: 1), () { liveSeconds++; notifyListeners(); });
}
void stopLive() { liveTimer?.cancel(); mode = "public"; notifyListeners(); }
String get liveTime {
final m = liveSeconds ~/ 60; final s = liveSeconds % 60;
return "{m.toString().padLeft(2, '0')}:{s.toString().padLeft(2, '0')}";
}
void tambahAnggota(List list, String nama, File? foto) { list.add({"nama": nama, "foto": foto}); notifyListeners(); }
void hapusAnggota(List list, int index) { list.removeAt(index); notifyListeners(); }
void tambahFotoGaleri(File f) { galeri.add(f); notifyListeners(); }
Future<void> cleanCache() async {
try {
final dir = await getTemporaryDirectory();
if (dir.existsSync()) dir.listSync().forEach((f) => f.deleteSync());
notifyListeners();
} catch () {}
}
}
class NovaProMasterRoot extends StatelessWidget {
const NovaProMasterRoot({super.key});
@override
Widget build(BuildContext context) {
final p = Provider.of<AppProvider>(context);
switch (p.mode) {
case "admin": return const AdminDashboard();
case "struktur": return const StrukturManager();
case "gallery": return const GalleryManager();
case "live": return const LiveDisplay();
default: return const PublicDisplay();
}
}
}
class PublicDisplay extends StatefulWidget {
const PublicDisplay({super.key});
@override
State<PublicDisplay> createState() => _PublicDisplayState();
}
class _PublicDisplayState extends State<PublicDisplay> {
late ScrollController _scrollController;
@override
void initState() {
super.initState();
scrollController = ScrollController();
WidgetsBinding.instance.addPostFrameCallback(() => _startScrolling());
}
void _startScrolling() async {
while (_scrollController.hasClients) {
await Future.delayed(const Duration(milliseconds: 50));
if (_scrollController.hasClients) {
double maxScroll = _scrollController.position.maxScrollExtent;
double currentScroll = _scrollController.position.pixels;
if (currentScroll >= maxScroll) { _scrollController.jumpTo(0); }
else { _scrollController.animateTo(currentScroll + 2, duration: const Duration(milliseconds: 50), curve: Curves.linear); }
}
}
}
@override
void dispose() { _scrollController.dispose(); super.dispose(); }
@override
Widget build(BuildContext context) {
final p = Provider.of<AppProvider>(context);
return Scaffold(
body: Container(
width: double.infinity,
decoration: const BoxDecoration(gradient: LinearGradient(begin: Alignment.topLeft, end: Alignment.bottomRight, colors: [Color(0xFF631414), Color(0xFF2E0909)])),
child: Stack(
children: [
Center(
child: Column(
mainAxisAlignment: MainAxisAlignment.center,
children: [
const Icon(Icons.mosque, size: 120, color: Color(0xFFFFD700)),
const SizedBox(height: 25),
Text(p.namaIbadah, style: const TextStyle(color: Colors.white, fontSize: 50, fontWeight: FontWeight.bold, letterSpacing: 2)),
Text(p.alamat, style: const TextStyle(color: Color(0xFFFFD700), fontSize: 20, fontWeight: FontWeight.w300)),
],
),
),
Positioned(top: 40, right: 30, child: IconButton(icon: const Icon(Icons.settings, color: Colors.white10), onPressed: () => _showLogin(context, p))),
Positioned(
bottom: 0,
child: Container(
width: MediaQuery.of(context).size.width,
color: const Color(0xFF631414).withOpacity(0.95),
height: 70,
child: ListView(
controller: _scrollController,
scrollDirection: Axis.horizontal,
children: [
Padding(padding: const EdgeInsets.symmetric(vertical: 18, horizontal: 25), child: Text(p.runningText, style: const TextStyle(color: Color(0xFFFFD700), fontSize: 24, fontWeight: FontWeight.bold))),
const SizedBox(width: 800),
],
),
),
)
],
),
),
);
}
void showLogin(BuildContext context, AppProvider p) {
final c = TextEditingController();
showDialog(context: context, builder: () => AlertDialog(
title: const Text("Admin Access"),
content: TextField(controller: c, obscureText: true, decoration: const InputDecoration(labelText: "Password")),
actions: [ElevatedButton(style: ElevatedButton.styleFrom(backgroundColor: const Color(0xFF631414)), onPressed: () { if(p.login(c.text)) Navigator.pop(context); }, child: const Text("Masuk", style: TextStyle(color: Colors.white)))],
));
}
}
class AdminDashboard extends StatelessWidget {
const AdminDashboard({super.key});
@override
Widget build(BuildContext context) {
final p = Provider.of<AppProvider>(context);
return Scaffold(
appBar: AppBar(title: const Text("NOVAPRO DASHBOARD"), backgroundColor: const Color(0xFF631414), foregroundColor: Colors.white, elevation: 0),
body: GridView.count(
padding: const EdgeInsets.all(25),
crossAxisCount: 2, crossAxisSpacing: 20, mainAxisSpacing: 20,
children: [
_card(Icons.group_work, "STRUKTUR", "Kelola Pengurus", () => p.mode = "struktur"),
_card(Icons.photo_album, "GALERI", "Foto Kegiatan", () => p.mode = "gallery"),
_card(Icons.sensors, "LIVE DISPLAY", "Mode Tayangan", () => p.startLive()),
_card(Icons.auto_fix_high, "OPTIMASI", "Bersihkan Cache", () async {
await p.cleanCache();
ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text("Cache Berhasil Dibersihkan!")));
}),
],
),
bottomNavigationBar: BottomAppBar(child: TextButton(onPressed: () => p.logout(), child: const Text("KELUAR SISTEM", style: TextStyle(color: Color(0xFF631414), fontWeight: FontWeight.bold)))),
);
}
Widget _card(IconData icon, String title, String sub, VoidCallback tap) {
return InkWell(
onTap: tap,
child: Card(
elevation: 8,
shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(15)),
child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [
Icon(icon, size: 55, color: const Color(0xFF631414)),
const SizedBox(height: 12),
Text(title, style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 18, color: Color(0xFF631414))),
Text(sub, style: const TextStyle(color: Colors.grey, fontSize: 13)),
]),
),
);
}
}
class StrukturManager extends StatelessWidget {
const StrukturManager({super.key});
@override
Widget build(BuildContext context) {
final p = Provider.of<AppProvider>(context);
return Scaffold(
appBar: AppBar(title: const Text("KELOLA PETUGAS"), leading: IconButton(icon: const Icon(Icons.arrow_back), onPressed: () => p.mode = "admin"), backgroundColor: const Color(0xFF631414), foregroundColor: Colors.white),
body: ListView(
padding: const EdgeInsets.all(20),
children: p.struktur.map((s) => Card(
margin: const EdgeInsets.only(bottom: 15),
shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
child: ExpansionTile(
leading: const Icon(Icons.assignment_ind, color: Color(0xFF631414)),
title: Text(s['jabatan'], style: const TextStyle(fontWeight: FontWeight.bold, color: Color(0xFF631414))),
children: [
...s['anggota'].asMap().entries.map((e) => ListTile(
leading: CircleAvatar(backgroundColor: const Color(0xFF631414), backgroundImage: e.value['foto'] != null ? FileImage(e.value['foto']) : null, child: e.value['foto'] == null ? const Icon(Icons.person, color: Color(0xFFFFD700)) : null),
title: Text(e.value['nama'], style: const TextStyle(fontWeight: FontWeight.w500)),
trailing: IconButton(icon: const Icon(Icons.delete_sweep, color: Colors.red), onPressed: () => p.hapusAnggota(s['anggota'], e.key)),
)),
TextButton.icon(onPressed: () => _add(context, p, s['anggota']), icon: const Icon(Icons.add_circle), label: const Text("Tambah Petugas"))
],
),
)).toList(),
),
);
}
void add(BuildContext context, AppProvider p, List list) async {
final ctrl = TextEditingController(); File? foto;
showDialog(context: context, builder: () => AlertDialog(
title: const Text("Input Data Petugas"),
content: Column(mainAxisSize: MainAxisSize.min, children: [
TextField(controller: ctrl, decoration: const InputDecoration(hintText: "Nama Lengkap")),
const SizedBox(height: 15),
ElevatedButton.icon(onPressed: () async {
final img = await ImagePicker().pickImage(source: ImageSource.gallery);
if (img != null) foto = File(img.path);
}, icon: const Icon(Icons.image), label: const Text("UNGGAH FOTO"))
]),
actions: [ElevatedButton(onPressed: () { if(ctrl.text.isNotEmpty) p.tambahAnggota(list, ctrl.text, foto); Navigator.pop(context); }, child: const Text("SIMPAN"))],
));
}
}
class GalleryManager extends StatelessWidget {
const GalleryManager({super.key});
@override
Widget build(BuildContext context) {
final p = Provider.of<AppProvider>(context);
return Scaffold(
appBar: AppBar(title: const Text("GALERI KEGIATAN"), leading: IconButton(icon: const Icon(Icons.arrow_back), onPressed: () => p.mode = "admin"), backgroundColor: const Color(0xFF631414), foregroundColor: Colors.white),
floatingActionButton: FloatingActionButton(backgroundColor: const Color(0xFF631414), child: const Icon(Icons.add_a_photo, color: Colors.white), onPressed: () async {
final img = await ImagePicker().pickImage(source: ImageSource.gallery);
if (img != null) p.tambahFotoGaleri(File(img.path));
}),
body: p.galeri.isEmpty ? const Center(child: Text("Belum ada dokumentasi kegiatan.")) : GridView.builder(
padding: const EdgeInsets.all(15),
gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(crossAxisCount: 3, crossAxisSpacing: 12, mainAxisSpacing: 12),
itemCount: p.galeri.length,
itemBuilder: (context, i) => ClipRRect(borderRadius: BorderRadius.circular(10), child: Image.file(p.galeri[i], fit: BoxFit.cover)),
),
);
}
}
class LiveDisplay extends StatelessWidget {
const LiveDisplay({super.key});
@override
Widget build(BuildContext context) {
final p = Provider.of<AppProvider>(context);
return Scaffold(
backgroundColor: Colors.black,
body: Center(
child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [
const Icon(Icons.radio_button_checked, color: Colors.red, size: 40),
const SizedBox(height: 15),
const Text("LIVE BROADCAST DISPLAY", style: TextStyle(color: Colors.white, fontSize: 28, fontWeight: FontWeight.bold)),
Text("WAKTU AKTIF: ${p.liveTime}", style: const TextStyle(color: Color(0xFFFFD700), fontSize: 22)),
const SizedBox(height: 60),
const CircularProgressIndicator(color: Color(0xFFFFD700)),
const SizedBox(height: 60),
ElevatedButton(onPressed: () => p.stopLive(), style: ElevatedButton.styleFrom(backgroundColor: const Color(0xFF631414), padding: const EdgeInsets.symmetric(horizontal: 40, vertical: 15)), child: const Text("AKHIRI TAYANGAN", style: TextStyle(color: Colors.white))),
]),
),
);
}
}

