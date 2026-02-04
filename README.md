import 'dart:async';
import 'dart:io';
import 'package:flutter/gestures.dart';
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'package:image_picker/image_picker.dart';
import 'package:path_provider/path_provider.dart';

void main() {
  runApp(
    ChangeNotifierProvider(
      create: (_) => AppProvider(),
      child: const MyApp(),
    ),
  );
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'NOVAPRO PREMIUM',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        primaryColor: const Color(0xFF631414),
        scaffoldBackgroundColor: const Color(0xFFF5F5F5),
        colorScheme: ColorScheme.fromSwatch()
            .copyWith(secondary: const Color(0xFFFFD700)),
        useMaterial3: true,
      ),
      home: const NovaProMasterRoot(),
    );
  }
}

/* ============================================================
   PROVIDER
   ============================================================ */

class AppProvider with ChangeNotifier {
  bool isLogin = false;
  String mode = "public";

  String adminPass = "123";

  String namaIbadah = "MASJID AN-NUUR";
  String alamat = "SOLUSI OFFLINE - PT. MAHANG DIAMOND PERKASA";

  String runningText =
      "SELAMAT DATANG DI NOVAPRO PREMIUM • SISTEM MANAJEMEN RUMAH IBADAH MODERN • OFFLINE & AMAN • PT. MAHANG DIAMOND PERKASA";

  Timer? liveTimer;
  int liveSeconds = 0;

  List<Map<String, dynamic>> struktur = [
    {
      "jabatan": "Ketua",
      "anggota": [
        {"nama": "Kariyono Hadi, BA", "foto": null}
      ]
    },
    {
      "jabatan": "Bendahara",
      "anggota": [
        {"nama": "Suherdi", "foto": null}
      ]
    },
    {
      "jabatan": "Sekretaris",
      "anggota": [
        {"nama": "Anang Ariyanto", "foto": null}
      ]
    },
  ];

  List<File> galeri = [];

  bool login(String pass) {
    if (pass == adminPass) {
      isLogin = true;
      mode = "admin";
      notifyListeners();
      return true;
    }
    return false;
  }

  void logout() {
    isLogin = false;
    mode = "public";
    stopLive();
    notifyListeners();
  }

  void startLive() {
    mode = "live";
    liveSeconds = 0;
    liveTimer?.cancel();
    liveTimer = Timer.periodic(const Duration(seconds: 1), (_) {
      liveSeconds++;
      notifyListeners();
    });
  }

  void stopLive() {
    liveTimer?.cancel();
    mode = "public";
    notifyListeners();
  }

  String get liveTime {
    final m = liveSeconds ~/ 60;
    final s = liveSeconds % 60;
    return "${m.toString().padLeft(2, '0')}:${s.toString().padLeft(2, '0')}";
  }

  void tambahAnggota(List list, String nama, File? foto) {
    list.add({"nama": nama, "foto": foto});
    notifyListeners();
  }

  void hapusAnggota(List list, int index) {
    list.removeAt(index);
    notifyListeners();
  }

  void tambahFotoGaleri(File f) {
    galeri.add(f);
    notifyListeners();
  }

  Future<void> cleanCache() async {
    try {
      final dir = await getTemporaryDirectory();
      if (dir.existsSync()) {
        for (var f in dir.listSync()) {
          try {
            f.deleteSync();
          } catch (_) {}
        }
      }
    } catch (_) {}
    notifyListeners();
  }
}

/* ============================================================
   ROOT + ANDROID BOX SUPPORT
   ============================================================ */

class NovaProMasterRoot extends StatelessWidget {
  const NovaProMasterRoot({super.key});

  @override
  Widget build(BuildContext context) {
    final p = Provider.of<AppProvider>(context);

    Widget body;

    switch (p.mode) {
      case "admin":
        body = const AdminDashboard();
        break;
      case "struktur":
        body = const StrukturManager();
        break;
      case "gallery":
        body = const GalleryManager();
        break;
      case "live":
        body = const LiveDisplay();
        break;
      case "calc":
        body = const CalculatorPage();
        break;
      default:
        body = const PublicDisplay();
    }

    return WillPopScope(
      onWillPop: () async {
        if (p.mode != "public") {
          p.mode = "admin";
          p.notifyListeners();
          return false;
        }
        return true;
      },
      child: Listener(
        onPointerDown: (event) {
          if (event.kind == PointerDeviceKind.mouse &&
              (event.buttons & kSecondaryMouseButton) != 0) {
            if (p.mode == "public") {
              p.mode = "admin";
              p.notifyListeners();
            }
          }
        },
        child: body,
      ),
    );
  }
}

/* ============================================================
   PUBLIC DISPLAY
   ============================================================ */

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
    _scrollController = ScrollController();
    WidgetsBinding.instance.addPostFrameCallback((_) => _startScrolling());
  }

  void _startScrolling() async {
    while (mounted && _scrollController.hasClients) {
      await Future.delayed(const Duration(milliseconds: 50));
      if (!_scrollController.hasClients) return;

      double maxScroll = _scrollController.position.maxScrollExtent;
      double currentScroll = _scrollController.position.pixels;

      if (currentScroll >= maxScroll) {
        _scrollController.jumpTo(0);
      } else {
        _scrollController.jumpTo(currentScroll + 2);
      }
    }
  }

  @override
  void dispose() {
    _scrollController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final p = Provider.of<AppProvider>(context);

    return Scaffold(
      body: Container(
        width: double.infinity,
        decoration: const BoxDecoration(
          gradient: LinearGradient(
            begin: Alignment.topLeft,
            end: Alignment.bottomRight,
            colors: [Color(0xFF631414), Color(0xFF2E0909)],
          ),
        ),
        child: Stack(
          children: [
            Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  const Icon(Icons.mosque,
                      size: 120, color: Color(0xFFFFD700)),
                  const SizedBox(height: 25),
                  Text(
                    p.namaIbadah,
                    style: const TextStyle(
                        color: Colors.white,
                        fontSize: 42,
                        fontWeight: FontWeight.bold,
                        letterSpacing: 2),
                    textAlign: TextAlign.center,
                  ),
                  const SizedBox(height: 10),
                  Text(
                    p.alamat,
                    style: const TextStyle(
                        color: Color(0xFFFFD700), fontSize: 18),
                  ),
                ],
              ),
            ),
            Positioned(
              top: 30,
              right: 20,
              child: IconButton(
                icon: const Icon(Icons.settings, color: Colors.white24),
                onPressed: () => _showLogin(context, p),
              ),
            ),
            Positioned(
              bottom: 0,
              child: Container(
                width: MediaQuery.of(context).size.width,
                height: 70,
                color: const Color(0xFF631414).withOpacity(0.95),
                child: ListView(
                  controller: _scrollController,
                  scrollDirection: Axis.horizontal,
                  children: [
                    Padding(
                      padding: const EdgeInsets.symmetric(
                          vertical: 18, horizontal: 25),
                      child: Text(
                        p.runningText,
                        style: const TextStyle(
                            color: Color(0xFFFFD700),
                            fontSize: 22,
                            fontWeight: FontWeight.bold),
                      ),
                    ),
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

  void _showLogin(BuildContext context, AppProvider p) {
    final c = TextEditingController();

    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: const Text("Admin Access"),
        content: TextField(
          controller: c,
          obscureText: true,
          decoration: const InputDecoration(labelText: "Password"),
        ),
        actions: [
          ElevatedButton(
            style: ElevatedButton.styleFrom(
                backgroundColor: const Color(0xFF631414)),
            onPressed: () {
              if (p.login(c.text)) Navigator.pop(context);
            },
            child:
                const Text("Masuk", style: TextStyle(color: Colors.white)),
          )
        ],
      ),
    );
  }
}

/* ============================================================
   ADMIN DASHBOARD
   ============================================================ */

class AdminDashboard extends StatelessWidget {
  const AdminDashboard({super.key});

  @override
  Widget build(BuildContext context) {
    final p = Provider.of<AppProvider>(context);

    return Scaffold(
      appBar: AppBar(
        title: const Text("NOVAPRO DASHBOARD"),
        backgroundColor: const Color(0xFF631414),
        foregroundColor: Colors.white,
      ),
      body: GridView.count(
        padding: const EdgeInsets.all(25),
        crossAxisCount: 2,
        crossAxisSpacing: 20,
        mainAxisSpacing: 20,
        children: [
          _card(Icons.group_work, "STRUKTUR", "Kelola Pengurus",
              () => p.mode = "struktur"),
          _card(Icons.photo_album, "GALERI", "Foto Kegiatan",
              () => p.mode = "gallery"),
          _card(Icons.sensors, "LIVE DISPLAY", "Mode Tayangan",
              () => p.startLive()),
          _card(Icons.calculate, "KALKULATOR", "Hitung cepat kas",
              () => p.mode = "calc"),
          _card(Icons.auto_fix_high, "OPTIMASI", "Bersihkan Cache",
              () async {
            await p.cleanCache();
            ScaffoldMessenger.of(context).showSnackBar(
              const SnackBar(content: Text("Cache dibersihkan")),
            );
          }),
        ],
      ),
      bottomNavigationBar: BottomAppBar(
        child: TextButton(
          onPressed: () => p.logout(),
          child: const Text(
            "KELUAR SISTEM",
            style: TextStyle(
                color: Color(0xFF631414), fontWeight: FontWeight.bold),
          ),
        ),
      ),
    );
  }

  Widget _card(IconData icon, String title, String sub, VoidCallback tap) {
    return InkWell(
      onTap: tap,
      child: Card(
        elevation: 6,
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(15)),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(icon, size: 48, color: const Color(0xFF631414)),
            const SizedBox(height: 10),
            Text(title,
                style: const TextStyle(
                    fontWeight: FontWeight.bold,
                    fontSize: 16,
                    color: Color(0xFF631414))),
            Text(sub,
                style:
                    const TextStyle(color: Colors.grey, fontSize: 12)),
          ],
        ),
      ),
    );
  }
}

/* ============================================================
   STRUKTUR
   ============================================================ */

class StrukturManager extends StatelessWidget {
  const StrukturManager({super.key});

  @override
  Widget build(BuildContext context) {
    final p = Provider.of<AppProvider>(context);

    return Scaffold(
      appBar: AppBar(
        title: const Text("KELOLA PETUGAS"),
        leading: IconButton(
            icon: const Icon(Icons.arrow_back),
            onPressed: () => p.mode = "admin"),
        backgroundColor: const Color(0xFF631414),
        foregroundColor: Colors.white,
      ),
      body: ListView(
        padding: const EdgeInsets.all(20),
        children: p.struktur
            .map(
              (s) => Card(
                margin: const EdgeInsets.only(bottom: 15),
                child: ExpansionTile(
                  leading: const Icon(Icons.assignment_ind,
                      color: Color(0xFF631414)),
                  title: Text(
                    s['jabatan'],
                    style: const TextStyle(
                        fontWeight: FontWeight.bold,
                        color: Color(0xFF631414)),
                  ),
                  children: [
                    ...s['anggota']
                        .asMap()
                        .entries
                        .map(
                          (e) => ListTile(
                            leading: CircleAvatar(
                              backgroundColor:
                                  const Color(0xFF631414),
                              backgroundImage: e.value['foto'] !=
                                      null
                                  ? FileImage(e.value['foto'])
                                  : null,
                              child: e.value['foto'] == null
                                  ? const Icon(Icons.person,
                                      color: Color(0xFFFFD700))
                                  : null,
                            ),
                            title: Text(e.value['nama']),
                            trailing: IconButton(
                              icon: const Icon(Icons.delete,
                                  color: Colors.red),
                              onPressed: () => p.hapusAnggota(
                                  s['anggota'], e.key),
                            ),
                          ),
                        ),
                    TextButton.icon(
                      onPressed: () =>
                          _add(context, p, s['anggota']),
                      icon: const Icon(Icons.add),
                      label: const Text("Tambah Petugas"),
                    )
                  ],
                ),
              ),
            )
            .toList(),
      ),
    );
  }

  void _add(BuildContext context, AppProvider p, List list) async {
    final ctrl = TextEditingController();
    File? foto;

    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: const Text("Input Data Petugas"),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            TextField(
                controller: ctrl,
                decoration:
                    const InputDecoration(hintText: "Nama Lengkap")),
            const SizedBox(height: 15),
            ElevatedButton.icon(
              onPressed: () async {
                final img = await ImagePicker()
                    .pickImage(source: ImageSource.gallery);
                if (img != null) foto = File(img.path);
              },
              icon: const Icon(Icons.image),
              label: const Text("Unggah Foto"),
            )
          ],
        ),
        actions: [
          ElevatedButton(
            onPressed: () {
              if (ctrl.text.isNotEmpty) {
                p.tambahAnggota(list, ctrl.text, foto);
              }
              Navigator.pop(context);
            },
            child: const Text("Simpan"),
          )
        ],
      ),
    );
  }
}

/* ============================================================
   GALERI
   ============================================================ */

class GalleryManager extends StatelessWidget {
  const GalleryManager({super.key});

  @override
  Widget build(BuildContext context) {
    final p = Provider.of<AppProvider>(context);

    return Scaffold(
      appBar: AppBar(
        title: const Text("GALERI KEGIATAN"),
        leading: IconButton(
            icon: const Icon(Icons.arrow_back),
            onPressed: () => p.mode = "admin"),
        backgroundColor: const Color(0xFF631414),
        foregroundColor: Colors.white,
      ),
      floatingActionButton: FloatingActionButton(
        backgroundColor: const Color(0xFF631414),
        child: const Icon(Icons.add_a_photo, color: Colors.white),
        onPressed: () async {
          final img = await ImagePicker()
              .pickImage(source: ImageSource.gallery);
          if (img != null) p.tambahFotoGaleri(File(img.path));
        },
      ),
      body: p.galeri.isEmpty
          ? const Center(child: Text("Belum ada dokumentasi."))
          : GridView.builder(
              padding: const EdgeInsets.all(15),
              gridDelegate:
                  const SliverGridDelegateWithFixedCrossAxisCount(
                      crossAxisCount: 3,
                      crossAxisSpacing: 12,
                      mainAxisSpacing: 12),
              itemCount: p.galeri.length,
              itemBuilder: (_, i) => ClipRRect(
                borderRadius: BorderRadius.circular(10),
                child: Image.file(p.galeri[i], fit: BoxFit.cover),
              ),
            ),
    );
  }
}

/* ============================================================
   LIVE DISPLAY
   ============================================================ */

class LiveDisplay extends StatelessWidget {
  const LiveDisplay({super.key});

  @override
  Widget build(BuildContext context) {
    final p = Provider.of<AppProvider>(context);

    return Scaffold(
      backgroundColor: Colors.black,
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            const Icon(Icons.radio_button_checked,
                color: Colors.red, size: 40),
            const SizedBox(height: 15),
            const Text(
              "LIVE BROADCAST DISPLAY",
              style: TextStyle(
                  color: Colors.white,
                  fontSize: 26,
                  fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 10),
            Text(
              "WAKTU AKTIF: ${p.liveTime}",
              style: const TextStyle(
                  color: Color(0xFFFFD700), fontSize: 22),
            ),
            const SizedBox(height: 40),
            const CircularProgressIndicator(
                color: Color(0xFFFFD700)),
            const SizedBox(height: 40),
            ElevatedButton(
              onPressed: () => p.stopLive(),
              style: ElevatedButton.styleFrom(
                  backgroundColor: const Color(0xFF631414)),
              child: const Text("AKHIRI TAYANGAN",
                  style: TextStyle(color: Colors.white)),
            ),
          ],
        ),
      ),
    );
  }
}

/* ============================================================
   KALKULATOR (BARU)
   ============================================================ */

class CalculatorPage extends StatefulWidget {
  const CalculatorPage({super.key});

  @override
  State<CalculatorPage> createState() => _CalculatorPageState();
}

class _CalculatorPageState extends State<CalculatorPage> {
  final a = TextEditingController();
  final b = TextEditingController();
  double hasil = 0;

  @override
  Widget build(BuildContext context) {
    final p = Provider.of<AppProvider>(context, listen: false);

    return Scaffold(
      appBar: AppBar(
        title: const Text("KALKULATOR KAS"),
        leading: IconButton(
          icon: const Icon(Icons.arrow_back),
          onPressed: () => p.mode = "admin",
        ),
        backgroundColor: const Color(0xFF631414),
        foregroundColor: Colors.white,
      ),
      body: Padding(
        padding: const EdgeInsets.all(20),
        child: Column(
          children: [
            TextField(
              controller: a,
              keyboardType: TextInputType.number,
              decoration:
                  const InputDecoration(labelText: "Angka 1"),
            ),
            TextField(
              controller: b,
              keyboardType: TextInputType.number,
              decoration:
                  const InputDecoration(labelText: "Angka 2"),
            ),
            const SizedBox(height: 20),
            Wrap(
              spacing: 10,
              children: [
                _btn("+", () => hitung((x, y) => x + y)),
                _btn("-", () => hitung((x, y) => x - y)),
                _btn("×", () => hitung((x, y) => x * y)),
                _btn("÷", () => hitung((x, y) => x / y)),
              ],
            ),
            const SizedBox(height: 30),
            Text(
              "Hasil : $hasil",
              style: const TextStyle(
                  fontSize: 24, fontWeight: FontWeight.bold),
            )
          ],
        ),
      ),
    );
  }

  Widget _btn(String t, VoidCallback onTap) {
    return ElevatedButton(
      onPressed: onTap,
      style: ElevatedButton.styleFrom(
          backgroundColor: const Color(0xFF631414)),
      child: Text(t, style: const TextStyle(color: Colors.white)),
    );
  }

  void hitung(double Function(double, double) op) {
    final x = double.tryParse(a.text) ?? 0;
    final y = double.tryParse(b.text) ?? 0;
    setState(() => hasil = op(x, y));
  }
}
git add lib/main.dart
git commit -m "Unified NOVAPRO Premium + Android Box + Calculator"
git push