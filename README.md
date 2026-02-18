import 'dart:async';
import 'dart:io';

import 'package:flutter/gestures.dart';
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'package:image_picker/image_picker.dart';
import 'package:path_provider/path_provider.dart';
import 'package:shared_preferences/shared_preferences.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  runApp(
    ChangeNotifierProvider(
      create: (_) => AppProvider()..loadData(),
      child: const MyApp(),
    ),
  );
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'NOVAPRO ALL-IN-ONE',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        primaryColor: const Color(0xFF631414),
        scaffoldBackgroundColor: const Color(0xFFF5F5F5),
        colorScheme:
            ColorScheme.fromSwatch().copyWith(secondary: const Color(0xFFFFD700)),
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
  String alamat = "SOLUSI OFFLINE - NOVARIZAL DEVELOPER";
  String runningText =
      "SELAMAT DATANG DI NOVAPRO • SISTEM MANAJEMEN MODERN • BY: NOVARIZAL DEVELOPER";

  String houseUniqueCode =
      "NP-${DateTime.now().millisecondsSinceEpoch.toString().substring(7)}";

  List<Map<String, dynamic>> struktur = [
    {"jabatan": "Ketua", "anggota": []},
    {"jabatan": "Bendahara", "anggota": []},
    {"jabatan": "Sekretaris", "anggota": []},
  ];

  List<File> galeri = [];

  Timer? liveTimer;
  int liveSeconds = 0;

  /* ===================== SAVE / LOAD ===================== */

  Future<void> saveData() async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString('namaIbadah', namaIbadah);
    await prefs.setString('alamat', alamat);
    await prefs.setString('runningText', runningText);

    List<String> paths = galeri.map((e) => e.path).toList();
    await prefs.setStringList('galeriPaths', paths);

    notifyListeners();
  }

  Future<void> loadData() async {
    final prefs = await SharedPreferences.getInstance();
    namaIbadah = prefs.getString('namaIbadah') ?? namaIbadah;
    alamat = prefs.getString('alamat') ?? alamat;
    runningText = prefs.getString('runningText') ?? runningText;

    final paths = prefs.getStringList('galeriPaths');
    if (paths != null) {
      galeri = paths.map((e) => File(e)).toList();
    }

    notifyListeners();
  }

  /* ===================== LOGIN ===================== */

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

  /* ===================== LIVE ===================== */

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
    mode = "admin";
    notifyListeners();
  }

  String get liveTime {
    final m = liveSeconds ~/ 60;
    final s = liveSeconds % 60;
    return "${m.toString().padLeft(2, '0')}:${s.toString().padLeft(2, '0')}";
  }

  /* ===================== DATA ===================== */

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
    saveData();
  }

  Future<void> cleanCache() async {
    final dir = await getTemporaryDirectory();
    if (dir.existsSync()) {
      dir.listSync().forEach((f) {
        try {
          f.deleteSync(recursive: true);
        } catch (_) {}
      });
    }
    notifyListeners();
  }
}

/* ============================================================
   ROOT
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
      final max = _scrollController.position.maxScrollExtent;
      final cur = _scrollController.position.pixels;
      if (cur >= max) {
        _scrollController.jumpTo(0);
      } else {
        _scrollController.jumpTo(cur + 2);
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    final p = Provider.of<AppProvider>(context);
    return Scaffold(
      body: Container(
        decoration: const BoxDecoration(
            gradient:
                LinearGradient(colors: [Color(0xFF631414), Color(0xFF2E0909)])),
        child: Stack(
          children: [
            Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  const Icon(Icons.mosque,
                      size: 120, color: Color(0xFFFFD700)),
                  Text(p.namaIbadah,
                      style: const TextStyle(
                          color: Colors.white,
                          fontSize: 42,
                          fontWeight: FontWeight.bold)),
                  Text(p.alamat,
                      style: const TextStyle(
                          color: Color(0xFFFFD700), fontSize: 18)),
                  const SizedBox(height: 10),
                  Text("ID: ${p.houseUniqueCode}",
                      style:
                          const TextStyle(color: Colors.white24, fontSize: 12)),
                ],
              ),
            ),
            Positioned(
                top: 30,
                right: 20,
                child: IconButton(
                    icon: const Icon(Icons.settings, color: Colors.white24),
                    onPressed: () => _showLogin(context, p))),
            Positioned(
              bottom: 0,
              child: Container(
                height: 70,
                width: MediaQuery.of(context).size.width,
                color: const Color(0xFF631414).withOpacity(0.9),
                child: ListView(
                  controller: _scrollController,
                  scrollDirection: Axis.horizontal,
                  children: [
                    Padding(
                        padding: const EdgeInsets.all(18),
                        child: Text(p.runningText,
                            style: const TextStyle(
                                color: Color(0xFFFFD700),
                                fontSize: 22,
                                fontWeight: FontWeight.bold))),
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
              title: const Text("Akses Admin"),
              content: TextField(
                controller: c,
                obscureText: true,
                decoration: const InputDecoration(labelText: "Password"),
              ),
              actions: [
                ElevatedButton(
                    onPressed: () {
                      if (p.login(c.text)) Navigator.pop(context);
                    },
                    child: const Text("Masuk"))
              ],
            ));
  }
}

/* ============================================================
   DASHBOARD
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
        crossAxisCount: 2,
        padding: const EdgeInsets.all(20),
        children: [
          _card(Icons.group, "STRUKTUR", () => p.mode = "struktur"),
          _card(Icons.photo, "GALERI", () => p.mode = "gallery"),
          _card(Icons.tv, "LIVE", () => p.startLive()),
          _card(Icons.calculate, "KALKULATOR", () => p.mode = "calc"),
        ],
      ),
      bottomNavigationBar: TextButton(
          onPressed: () => p.logout(), child: const Text("KELUAR")),
    );
  }

  Widget _card(IconData i, String t, VoidCallback tap) => InkWell(
        onTap: tap,
        child: Card(
            child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [Icon(i, size: 40), Text(t)],
        )),
      );
}

/* ============================================================
   STRUKTUR PENGURUS
============================================================ */

class StrukturManager extends StatelessWidget {
  const StrukturManager({super.key});

  @override
  Widget build(BuildContext context) {
    final p = Provider.of<AppProvider>(context);

    return Scaffold(
      appBar: AppBar(
        title: const Text("STRUKTUR PENGURUS"),
        leading: IconButton(
            icon: const Icon(Icons.arrow_back),
            onPressed: () => p.mode = "admin"),
      ),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: p.struktur
            .map((s) => Card(
                  child: ExpansionTile(
                    title: Text(s['jabatan']),
                    children: [
                      ...s['anggota']
                          .asMap()
                          .entries
                          .map((e) => ListTile(
                                leading: CircleAvatar(
                                  backgroundImage: e.value['foto'] != null
                                      ? FileImage(e.value['foto'])
                                      : null,
                                  child: e.value['foto'] == null
                                      ? const Icon(Icons.person)
                                      : null,
                                ),
                                title: Text(e.value['nama']),
                                trailing: IconButton(
                                    icon: const Icon(Icons.delete,
                                        color: Colors.red),
                                    onPressed: () => p.hapusAnggota(
                                        s['anggota'], e.key)),
                              )),
                      TextButton.icon(
                          onPressed: () =>
                              _add(context, p, s['anggota']),
                          icon: const Icon(Icons.add),
                          label: const Text("Tambah"))
                    ],
                  ),
                ))
            .toList(),
      ),
    );
  }

  void _add(BuildContext context, AppProvider p, List list) async {
    final c = TextEditingController();
    File? foto;

    showDialog(
        context: context,
        builder: (_) => AlertDialog(
              title: const Text("Tambah Pengurus"),
              content: Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  TextField(controller: c, decoration: const InputDecoration(labelText: "Nama")),
                  const SizedBox(height: 10),
                  ElevatedButton.icon(
                      onPressed: () async {
                        final img = await ImagePicker()
                            .pickImage(source: ImageSource.gallery);
                        if (img != null) foto = File(img.path);
                      },
                      icon: const Icon(Icons.image),
                      label: const Text("Pilih Foto"))
                ],
              ),
              actions: [
                ElevatedButton(
                    onPressed: () {
                      if (c.text.isNotEmpty) {
                        p.tambahAnggota(list, c.text, foto);
                        Navigator.pop(context);
                      }
                    },
                    child: const Text("Simpan"))
              ],
            ));
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
        title: const Text("GALERI"),
        leading: IconButton(
            icon: const Icon(Icons.arrow_back),
            onPressed: () => p.mode = "admin"),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () async {
          final img =
              await ImagePicker().pickImage(source: ImageSource.gallery);
          if (img != null) p.tambahFotoGaleri(File(img.path));
        },
        child: const Icon(Icons.add_a_photo),
      ),
      body: GridView.builder(
        gridDelegate:
            const SliverGridDelegateWithFixedCrossAxisCount(crossAxisCount: 3),
        itemCount: p.galeri.length,
        itemBuilder: (_, i) => Image.file(p.galeri[i], fit: BoxFit.cover),
      ),
    );
  }
}

/* ============================================================
   LIVE
============================================================ */

class LiveDisplay extends StatelessWidget {
  const LiveDisplay({super.key});

  @override
  Widget build(BuildContext context) {
    final p = Provider.of<AppProvider>(context);

    return Scaffold(
      backgroundColor: Colors.black,
      body: Center(
        child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [
          const Icon(Icons.circle, color: Colors.red, size: 40),
          const SizedBox(height: 10),
          Text("LIVE ${p.liveTime}",
              style: const TextStyle(color: Colors.white, fontSize: 30)),
          const SizedBox(height: 30),
          ElevatedButton(
              onPressed: () => p.stopLive(),
              child: const Text("HENTIKAN"))
        ]),
      ),
    );
  }
}

/* ============================================================
   KALKULATOR
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

  void hitung(double Function(double, double) op) {
    final x = double.tryParse(a.text) ?? 0;
    final y = double.tryParse(b.text) ?? 0;
    setState(() => hasil = op(x, y));
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text("KALKULATOR KAS")),
      body: Padding(
        padding: const EdgeInsets.all(20),
        child: Column(children: [
          TextField(
              controller: a,
              keyboardType: TextInputType.number,
              decoration: const InputDecoration(labelText: "Angka 1")),
          TextField(
              controller: b,
              keyboardType: TextInputType.number,
              decoration: const InputDecoration(labelText: "Angka 2")),
          const SizedBox(height: 20),
          Wrap(spacing: 10, children: [
            ElevatedButton(onPressed: () => hitung((x, y) => x + y), child: const Text("+")),
            ElevatedButton(onPressed: () => hitung((x, y) => x - y), child: const Text("-")),
            ElevatedButton(onPressed: () => hitung((x, y) => x * y), child: const Text("×")),
            ElevatedButton(onPressed: () => hitung((x, y) => y == 0 ? 0 : x / y), child: const Text("÷")),
          ]),
          const SizedBox(height: 30),
          Text("HASIL : $hasil",
              style:
                  const TextStyle(fontSize: 28, fontWeight: FontWeight.bold))
        ]),
      ),
    );
  }
}