import 'dart:async';
import 'dart:math';
import 'dart:convert';
import 'dart:io';
import 'package:flutter/foundation.dart' show kIsWeb;
import 'package:marquee/marquee.dart';
import 'package:pdf/pdf.dart';
import 'package:pdf/widgets.dart' as pw;
import 'package:printing/printing.dart';
import 'package:path_provider/path_provider.dart';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:provider/provider.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:intl/intl.dart';
import 'package:intl/date_symbol_data_local.dart';
import 'package:image_picker/image_picker.dart';
import 'package:video_player/video_player.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';


extension DateTimeExt on DateTime {
  int get dayOfYear {
    final start = DateTime(year, 1, 1);
    return difference(start).inDays + 1;
  }
}

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Tangkap semua Flutter error dan print ke console
  FlutterError.onError = (FlutterErrorDetails details) {
    debugPrint('FLUTTER ERROR: ${details.exception}');
    debugPrint('STACK: ${details.stack}');
    FlutterError.presentError(details);
  };

  await initializeDateFormatting('id', null);
  await initializeDateFormatting('en', null);
  await initializeDateFormatting('ar', null);
  await initializeDateFormatting();
  await AdzanNotifikasi.init(); // inisialisasi notifikasi
  runApp(
    ChangeNotifierProvider(
      create: (_) => AppProvider()..loadData(),
      child: const MyApp(),
    ),
  );
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  static ThemeData _buildTheme(bool isDark) {
    if (isDark) {
      return ThemeData.dark(useMaterial3: true).copyWith(
        scaffoldBackgroundColor: const Color(0xFF121212),
        cardColor: const Color(0xFF1E1E1E),
      );
    }
    return ThemeData(
      useMaterial3: true,
      scaffoldBackgroundColor: Colors.white,
      colorSchemeSeed: const Color(0xFF631414),
      cardTheme: CardTheme(
        elevation: 4,
        shadowColor: Colors.black.withValues(alpha: 0.08),
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.all(Radius.circular(16))),
        color: Colors.white,
      ),
      appBarTheme: const AppBarTheme(
        backgroundColor: Color(0xFF631414),
        foregroundColor: Colors.white,
        elevation: 2,
        shadowColor: Colors.black38,
        iconTheme: IconThemeData(color: Colors.white),
        titleTextStyle: TextStyle(color: Colors.white, fontSize: 18, fontWeight: FontWeight.bold),
      ),
      bottomNavigationBarTheme: const BottomNavigationBarThemeData(
        backgroundColor: Colors.white,
        selectedItemColor: Color(0xFF631414),
        unselectedItemColor: Color(0xFF616161),
        type: BottomNavigationBarType.fixed,
        elevation: 8,
      ),
      dividerTheme: const DividerThemeData(color: Color(0xFFE8F5E9), thickness: 1),
      inputDecorationTheme: InputDecorationTheme(
        filled: true,
        fillColor: const Color(0xFFF9FBF9),
        border: OutlineInputBorder(
          borderRadius: BorderRadius.circular(12),
          borderSide: const BorderSide(color: Color(0xFFE0E0E0)),
        ),
        enabledBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(12),
          borderSide: const BorderSide(color: Color(0xFFE0E0E0)),
        ),
        focusedBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(12),
          borderSide: const BorderSide(color: Color(0xFF631414), width: 1.5),
        ),
      ),
      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          backgroundColor: const Color(0xFF631414),
          foregroundColor: Colors.white,
          shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
          elevation: 2,
        ),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    // Selector bahasa - rebuild home saat bahasa berubah (Reload Key pattern)
    return Selector<AppProvider, String>(
      selector: (_, p) => p.bahasa,
      builder: (_, bahasa, __) => MaterialApp(
        title: 'NovaPro - Manajemen Masjid dan Mushollah',
        debugShowCheckedModeBanner: false,
        theme: _buildTheme(false),
        // Key berubah saat bahasa berubah - paksa rebuild semua widget
        home: PublicHome(key: ValueKey('home_$bahasa')),
      ),
    );
  }
}

// ============================================================
// ZOOM WRAPPER - Pinch to zoom semua halaman
// ============================================================
class ZoomWrapper extends StatefulWidget {
  final Widget child;
  const ZoomWrapper({required this.child, super.key});
  @override
  State<ZoomWrapper> createState() => _ZoomWrapperState();
}

class _ZoomWrapperState extends State<ZoomWrapper> {
  final TransformationController _ctrl = TransformationController();
  double _scale = 1.0;
  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  void _resetZoom() {
    // Animasi smooth kembali ke normal
    final begin = _ctrl.value;
    final end = Matrix4.identity();
    final tween = Matrix4Tween(begin: begin, end: end);
    final animCtrl = AnimationController(
      vsync: Navigator.of(context),
      duration: const Duration(milliseconds: 250),
    );
    animCtrl.addListener(() {
      _ctrl.value = tween.evaluate(animCtrl);
    });
    animCtrl.addStatusListener((status) {
      if (status == AnimationStatus.completed) {
        animCtrl.dispose();
        setState(() { _scale = 1.0; });
      }
    });
    animCtrl.forward();
  }

  @override
  Widget build(BuildContext context) {
    return Stack(
      children: [
        InteractiveViewer(
          transformationController: _ctrl,
          minScale: 0.8,
          maxScale: 5.0,
          scaleFactor: 100.0,
          // Pan hanya aktif saat sudah zoom
          panEnabled: _scale > 1.05,
          boundaryMargin: EdgeInsets.zero,
          constrained: true,
          onInteractionUpdate: (details) {
            final newScale = _ctrl.value.getMaxScaleOnAxis();
            if ((newScale - _scale).abs() > 0.01) {
              setState(() => _scale = newScale);
            }
          },
          onInteractionEnd: (_) {
            final newScale = _ctrl.value.getMaxScaleOnAxis();
            setState(() {
              _scale = newScale;
              if (newScale < 1.0) _resetZoom();
            });
          },
          child: widget.child,
        ),
        // Tombol reset - tampil saat zoom aktif
        if (_scale > 1.05)
          Positioned(
            top: 8, right: 8,
            child: GestureDetector(
              onTap: _resetZoom,
              child: Container(
                padding: const EdgeInsets.symmetric(horizontal: 10, vertical: 5),
                decoration: BoxDecoration(
                  color: kPrimary.withValues(alpha: 0.85),
                  borderRadius: BorderRadius.circular(16),
                  boxShadow: [BoxShadow(color: Colors.black38, blurRadius: 4)],
                ),
                child: const Row(mainAxisSize: MainAxisSize.min, children: [
                  Icon(Icons.zoom_out_map, color: Colors.white, size: 14),
                  SizedBox(width: 4),
                  Text('Reset', style: TextStyle(color: Colors.white, fontSize: 11, fontWeight: FontWeight.bold)),
                ]),
              ),
            ),
          ),
      ],
    );
  }
}

// ============================================================
// DATABASE KOTA & KALKULATOR WAKTU SHOLAT OFFLINE
// ============================================================
class _KotaData {
  final String nama;
  final double lat;
  final double lng;
  final double timezone;
  const _KotaData(this.nama, this.lat, this.lng, this.timezone);
}

class SholatCalculator {
  static const List<_KotaData> _kotaList = [
    _KotaData("Banda Aceh",5.5577,95.3222,7),_KotaData("Aceh",4.6951,96.7494,7),
    _KotaData("Medan",3.5952,98.6722,7),_KotaData("Padang",-0.9471,100.4172,7),
    _KotaData("Pekanbaru",0.5071,101.4478,7),_KotaData("Batam",1.0456,104.0305,7),
    _KotaData("Jambi",-1.6101,103.6131,7),_KotaData("Palembang",-2.9761,104.7754,7),
    _KotaData("Bengkulu",-3.7928,102.2608,7),_KotaData("Bandar Lampung",-5.3971,105.2668,7),
    _KotaData("Jakarta",-6.2088,106.8456,7),_KotaData("Depok",-6.4025,106.7942,7),
    _KotaData("Bogor",-6.5971,106.8060,7),_KotaData("Bekasi",-6.2383,106.9756,7),
    _KotaData("Tangerang",-6.1781,106.6297,7),_KotaData("Bandung",-6.9175,107.6191,7),
    _KotaData("Cirebon",-6.7063,108.5570,7),_KotaData("Semarang",-6.9932,110.4203,7),
    _KotaData("Solo",-7.5755,110.8243,7),_KotaData("Surakarta",-7.5755,110.8243,7),
    _KotaData("Yogyakarta",-7.7971,110.3688,7),_KotaData("Magelang",-7.4797,110.2177,7),
    _KotaData("Surabaya",-7.2575,112.7521,7),_KotaData("Malang",-7.9666,112.6326,7),
    _KotaData("Kediri",-7.8480,112.0171,7),_KotaData("Jember",-8.1724,113.7000,7),
    _KotaData("Banyuwangi",-8.2191,114.3691,7),_KotaData("Serang",-6.1201,106.1503,7),
    _KotaData("Pontianak",-0.0263,109.3425,7),_KotaData("Palangkaraya",-2.2161,113.9135,7),
    _KotaData("Banjarmasin",-3.3186,114.5944,8),_KotaData("Samarinda",-0.5022,117.1536,8),
    _KotaData("Balikpapan",-1.2379,116.8529,8),_KotaData("Tarakan",3.3000,117.6333,8),
    _KotaData("Makassar",-5.1477,119.4327,8),_KotaData("Parepare",-4.0135,119.6295,8),
    _KotaData("Kendari",-3.9985,122.5129,8),_KotaData("Palu",-0.8917,119.8707,8),
    _KotaData("Manado",1.4748,124.8421,8),_KotaData("Gorontalo",0.5435,123.0568,8),
    _KotaData("Denpasar",-8.6500,115.2167,8),_KotaData("Mataram",-8.5833,116.1167,8),
    _KotaData("Kupang",-10.1772,123.6070,8),_KotaData("Ambon",-3.6954,128.1814,9),
    _KotaData("Ternate",0.7833,127.3667,9),_KotaData("Jayapura",-2.5337,140.7181,9),
    _KotaData("Sorong",-0.8833,131.2500,9),_KotaData("Merauke",-8.4667,140.4000,9),
    _KotaData("Kuala Lumpur",3.1390,101.6869,8),_KotaData("Singapura",1.3521,103.8198,8),
    _KotaData("Singapore",1.3521,103.8198,8),_KotaData("Brunei",4.9031,114.9398,8),
    _KotaData("Riyadh",24.6877,46.7219,3),_KotaData("Mekah",21.3891,39.8579,3),
    _KotaData("Madinah",24.5247,39.5692,3),_KotaData("Dubai",25.2048,55.2708,4),
    _KotaData("Istanbul",41.0082,28.9784,3),_KotaData("Kairo",30.0444,31.2357,2),
    _KotaData("London",51.5074,-0.1278,0),_KotaData("Sydney",-33.8688,151.2093,10),
    _KotaData("Tokyo",35.6762,139.6503,9),
  ];

  // ignore: library_private_types_in_public_api
  static _KotaData? cariKota(String alamat) {
    final lower = alamat.toLowerCase();
    for (final kota in _kotaList) {
      if (lower.contains(kota.nama.toLowerCase())) return kota;
    }
    return null;
  }

  static Map<String, String> hitungWaktu(double lat, double lng, double timezone, {int koreksi = 0}) {
    final now = DateTime.now();
    // Gunakan timezone dari device user secara otomatis (bukan dari database)
    final tzDevice = now.timeZoneOffset.inMinutes / 60.0;
    return _hitung(lat, lng, tzDevice, now.year, now.month, now.day, koreksi);
  }

  static Map<String, String> _hitung(double lat, double lng, double tz, int year, int month, int day, int koreksi) {
    final jd = _julianDate(year, month, day);
    final d = jd - 2451545.0;
    final g = (357.529 + 0.98560028 * d) % 360;
    final q = (280.459 + 0.98564736 * d) % 360;
    final l = (q + 1.915 * sin(g * pi/180) + 0.020 * sin(2*g*pi/180)) % 360;
    final e = 23.439 - 0.00000036 * d;
    final ra = atan2(cos(e*pi/180)*sin(l*pi/180), cos(l*pi/180)) * 180/pi;
    final dec = asin(sin(e*pi/180)*sin(l*pi/180)) * 180/pi;
    final eqT = q - ra;
    final zawal = 12 + tz - lng/15 - eqT/60;

    String fmt(double t) {
      final total = t + koreksi/60.0;
      var h = total.floor();
      var m = ((total - h)*60).round();
      if (m >= 60) { h++; m -= 60; }
      h = h % 24;
      return '${h.toString().padLeft(2,'0')}:${m.toString().padLeft(2,'0')}';
    }

    double ha(double angle) {
      final cosH = (sin(angle*pi/180) - sin(lat*pi/180)*sin(dec*pi/180)) / (cos(lat*pi/180)*cos(dec*pi/180));
      if (cosH < -1 || cosH > 1) return 0;
      return acos(cosH) * 180/pi;
    }

    final subuh   = zawal - ha(-18)/15;
    final dzuhur  = zawal + 0.5/60;
    final zm      = atan(1/(1+tan((lat-dec).abs()*pi/180)))*180/pi;
    final cosAshar = (sin(zm*pi/180) - sin(lat*pi/180)*sin(dec*pi/180))/(cos(lat*pi/180)*cos(dec*pi/180));
    final ashar   = zawal + (cosAshar.abs() <= 1 ? acos(cosAshar)*180/pi : 0)/15;
    final maghrib = zawal + ha(-0.8333)/15;
    final isya    = zawal + ha(-17)/15;

    return Map<String, String>.from({
      'subuh': fmt(subuh), 'dzuhur': fmt(dzuhur), 'ashar': fmt(ashar),
      'maghrib': fmt(maghrib), 'isya': fmt(isya),
    });
  }

  static double _julianDate(int y, int m, int d) {
    if (m <= 2) { y--; m += 12; }
    final a = (y/100).floor();
    final b = 2 - a + (a/4).floor();
    return (365.25*(y+4716)).floor() + (30.6001*(m+1)).floor() + d + b - 1524.5;
  }
}

class AppProvider with ChangeNotifier {
  String mode = "public";
  String adminPass = "123";
  bool isFirstLogin = true; // wajib ganti password saat pertama login
  bool isDarkMode = false;

  String namaIbadah = "";
  String fotoMasjid = "";  // base64 atau kosong
  String fotoQris      = "";  // base64 gambar QRIS
  String namaBank      = "";  // misal: BRI, BCA, Dana
  String noRekening    = "";  // nomor rekening
  String namaPemilik   = "";  // nama pemilik rekening
  List<Map<String, dynamic>> pengumuman = [];
  String alamat = "";
  String runningText = "";
  String houseUniqueCode = "NP-0000";

  List<Map<String, dynamic>> galeri = <Map<String, dynamic>>[];
  List<Map<String, dynamic>> donaturList = <Map<String, dynamic>>[];
  List<Map<String, dynamic>> acaraList = <Map<String, dynamic>>[]; // data donatur terpisah

  void addDonatur(String nama, bool anonim, String kategori, double jumlah, String matauang, String ket, String tgl) {
    donaturList = List<Map<String, dynamic>>.from(donaturList)
      ..add({
        'nama': anonim ? 'Anonim' : nama,
        'anonim': anonim,
        'kategori': kategori,
        'jumlah': jumlah,
        'matauang': matauang,
        'keterangan': ket,
        'tgl': tgl,
      });
    _safeNotify();
    saveDonatur();
  }

  void deleteDonatur(int index) {
    donaturList = List<Map<String, dynamic>>.from(donaturList)..removeAt(index);
    _safeNotify();
    saveDonatur();
  }

  // ── ACARA ──
  void addAcara(String namaAcara, String tanggal, String sambutan) {
    acaraList = List<Map<String, dynamic>>.from(acaraList)
      ..add({'namaAcara': namaAcara, 'tanggal': tanggal, 'sambutan': sambutan, 'tamu': <Map<String, dynamic>>[]});
    _safeNotify();
    saveAcara();
  }

  void deleteAcara(int index) {
    acaraList = List<Map<String, dynamic>>.from(acaraList)..removeAt(index);
    _safeNotify();
    saveAcara();
  }

  void addAcaraDenganTamu(String namaAcara, String tanggal, String sambutan, List<Map<String, dynamic>> tamu) {
    acaraList = List<Map<String, dynamic>>.from(acaraList)
      ..add({'namaAcara': namaAcara, 'tanggal': tanggal, 'sambutan': sambutan, 'tamu': List<Map<String, dynamic>>.from(tamu)});
    _safeNotify();
    saveAcara();
  }

  void updateAcaraDenganTamu(int index, String namaAcara, String tanggal, String sambutan, List<Map<String, dynamic>> tamu) {
    final updated = Map<String, dynamic>.from(acaraList[index]);
    updated['namaAcara'] = namaAcara;
    updated['tanggal']   = tanggal;
    updated['sambutan']  = sambutan;
    updated['tamu']      = List<Map<String, dynamic>>.from(tamu);
    acaraList = List<Map<String, dynamic>>.from(acaraList)..[index] = updated;
    _safeNotify();
    saveAcara();
  }

  void updateAcara(int index, String namaAcara, String tanggal, String sambutan) {
    final updated = Map<String, dynamic>.from(acaraList[index]);
    updated['namaAcara'] = namaAcara;
    updated['tanggal']   = tanggal;
    updated['sambutan']  = sambutan;
    acaraList = List<Map<String, dynamic>>.from(acaraList)..[index] = updated;
    _safeNotify();
    saveAcara();
  }

  void addTamu(int acaraIdx, String nama, String jabatan, String foto) {
    final acara = Map<String, dynamic>.from(acaraList[acaraIdx]);
    final tamu  = List<Map<String, dynamic>>.from(acara['tamu'] as List);
    tamu.add({'nama': nama, 'jabatan': jabatan, 'foto': foto});
    acara['tamu'] = tamu;
    acaraList = List<Map<String, dynamic>>.from(acaraList)..[acaraIdx] = acara;
    _safeNotify();
    saveAcara();
  }

  void deleteTamu(int acaraIdx, int tamuIdx) {
    final acara = Map<String, dynamic>.from(acaraList[acaraIdx]);
    final tamu  = List<Map<String, dynamic>>.from(acara['tamu'] as List)..removeAt(tamuIdx);
    acara['tamu'] = tamu;
    acaraList = List<Map<String, dynamic>>.from(acaraList)..[acaraIdx] = acara;
    _safeNotify();
    saveAcara();
  }

  void updateFotoTamu(int acaraIdx, int tamuIdx, String foto) {
    final acara = Map<String, dynamic>.from(acaraList[acaraIdx]);
    final tamu  = List<Map<String, dynamic>>.from(acara['tamu'] as List);
    final t     = Map<String, dynamic>.from(tamu[tamuIdx]);
    t['foto']   = foto;
    tamu[tamuIdx] = t;
    acara['tamu'] = tamu;
    acaraList = List<Map<String, dynamic>>.from(acaraList)..[acaraIdx] = acara;
    _safeNotify();
    saveAcara();
  }

  // ── QRIS ──
  Future<void> saveQris() async {
    await (await SharedPreferences.getInstance()).setString('qris_foto', fotoQris.toString());
    await (await SharedPreferences.getInstance()).setString('qris_bank', namaBank.toString());
    await (await SharedPreferences.getInstance()).setString('qris_norek', noRekening.toString());
    await (await SharedPreferences.getInstance()).setString('qris_pemilik', namaPemilik.toString());
  }

  Future<void> loadQris() async {
    fotoQris    = (await SharedPreferences.getInstance()).getString('qris_foto')    ?? '';
    namaBank    = (await SharedPreferences.getInstance()).getString('qris_bank')    ?? '';
    noRekening  = (await SharedPreferences.getInstance()).getString('qris_norek')   ?? '';
    namaPemilik = (await SharedPreferences.getInstance()).getString('qris_pemilik') ?? '';
  }

  void updateQris(String foto, String bank, String noRek, String pemilik) {
    fotoQris    = foto;
    namaBank    = bank;
    noRekening  = noRek;
    namaPemilik = pemilik;
    _safeNotify();
    saveQris();
  }

  Future<void> saveAcara() async {
    String _encodeAcara() => acaraList.map((a) {
      final tamu = (a['tamu'] as List<dynamic>).map((t) =>
        '${(t['nama'] ?? '')}~~${(t['jabatan'] ?? '')}~~${(t['foto'] ?? '')}').join('||');
      return '${a['namaAcara'] ?? ''}:::${a['tanggal'] ?? ''}:::${a['sambutan'] ?? ''}:::$tamu';
    }).join(';;;');
    try {
      await (await SharedPreferences.getInstance()).setString('acara_data', _encodeAcara());
    } catch (_) {
      final prefs = await SharedPreferences.getInstance();
      await prefs.setString('acara_data', _encodeAcara());
    }
  }

  Future<void> loadAcara() async {
    try {
      final raw = (await SharedPreferences.getInstance()).getString('acara_data') ?? '';
      if (raw.isEmpty) return;
      acaraList = raw.split(';;;').map((s) {
        final parts = s.split(':::');
        final namaAcara = parts.isNotEmpty ? parts[0] : '';
        final tanggal   = parts.length > 1  ? parts[1] : '';
        final sambutan  = parts.length > 2  ? parts[2] : '';
        final tamuRaw   = parts.length > 3  ? parts[3] : '';
        final tamu = tamuRaw.isEmpty ? <Map<String, dynamic>>[] :
          tamuRaw.split('||').map((t) {
            final tp = t.split('~~');
            return Map<String, dynamic>.from({
              'nama':    tp.isNotEmpty ? tp[0] : '',
              'jabatan': tp.length > 1 ? tp[1] : '',
              'foto':    tp.length > 2 ? tp[2] : '',
            });
          }).toList();
        return Map<String, dynamic>.from({'namaAcara': namaAcara, 'tanggal': tanggal, 'sambutan': sambutan, 'tamu': tamu});
      }).toList();
    } catch (_) {
      final prefs = await SharedPreferences.getInstance();
      final raw = prefs.getString('acara_data') ?? '';
      if (raw.isEmpty) return;
      // same parsing
    }
  }

  double get totalDonatur => donaturList
      .where((d) => d['matauang'] == 'IDR')
      .fold(0, (sum, d) => sum + (d['jumlah'] as double));

  Future<void> saveDonatur() async {
    final encoded = donaturList.map((d) {
      final nama = (d['nama'] ?? '').toString();
      final anon = d['anonim'].toString();
      final kat  = (d['kategori'] ?? '').toString();
      final jml  = (d['jumlah'] ?? 0).toString();
      final mtu  = (d['matauang'] ?? 'IDR').toString();
      final ket  = (d['keterangan'] ?? '').toString();
      final tgl  = (d['tgl'] ?? '').toString();
      return '$nama|$anon|$kat|$jml|$mtu|$ket|$tgl';
    }).toList();
    try {
      await (await SharedPreferences.getInstance()).setString('donatur_data', encoded.join(';;'));
    } catch (_) {
      final prefs = await SharedPreferences.getInstance();
      await prefs.setStringList('donatur_data', encoded);
    }
  }


  Future<void> loadDonatur() async {
    try {
      final raw = (await SharedPreferences.getInstance()).getString('donatur_data') ?? '';
      if (raw.isEmpty) return;
      donaturList = raw.split(';;').map((s) {
        final p = s.split('|');
        if (p.length < 6) return <String, dynamic>{};
        return Map<String, dynamic>.from({
          'nama': p[0], 'anonim': p[1] == 'true',
          'kategori': p[2], 'jumlah': double.tryParse(p[3]) ?? 0.0,
          'matauang': p[4], 'keterangan': p[5],
          'tgl': p.length > 6 ? p[6] : '',
        });
      }).where((d) => d.isNotEmpty).toList();
      _safeNotify();
    } catch (_) {}
  }
  int durasiSlideDefault = 10;

  List<Map<String, dynamic>> transaksi = <Map<String, dynamic>>[];
  double saldoAwal = 0.0;         // Saldo awal kas - input manual
  double totalMasukSimpan = 0.0;  // Akumulasi total masuk - tidak pernah berkurang
  double totalKeluarSimpan = 0.0; // Akumulasi total keluar - tidak pernah berkurang

  List<Map<String, dynamic>> inventaris = [];

  // Jadwal Imsak Ramadhan
  bool aktifRamadhan = false; // toggle tampil slide imsak
  int menitSebelumSubuh = 10; // imsak = subuh - X menit
  int tahunRamadhan = DateTime.now().year;

  // Jadwal Imam & Khatib Jumat
  List<Map<String, dynamic>> jadwalJumat = [];
  // Format: {'tanggal': 'dd/MM/yyyy', 'imam': 'Nama', 'khatib': 'Nama', 'tema': 'Tema Khutbah'}

  List<Map<String, dynamic>> pengurus = [
    {'jabatan': 'Ketua',       'nama': '-'},
    {'jabatan': 'Bendahara',   'nama': '-'},
    {'jabatan': 'Sekretaris',  'nama': '-'},
  ];

  String get ketua      => pengurus.isNotEmpty ? pengurus[0]['nama'] ?? '-' : '-';
  String get bendahara  => pengurus.length > 1  ? pengurus[1]['nama'] ?? '-' : '-';
  String get sekretaris => pengurus.length > 2  ? pengurus[2]['nama'] ?? '-' : '-';

  String waktuSubuh = "04:45";
  String waktuDzuhur = "12:15";
  String waktuAshar = "15:30";
  String waktuMaghrib = "18:25";
  String waktuIsya = "19:35";
  bool jadwalManual = false; // true = admin sudah edit manual, jangan ditimpa auto

  Timer? liveTimer;
  int liveSeconds = 0;

  int koreksiMenit = 2;
  int menitIqomah  = 10;

  String bahasa = 'id';

  double kecepatanRunningText = 18.0;
  String warnaRunningText = 'FFFFFF';

  bool smartDisplayAktif = true;
  bool sedangAdzan       = false;

  void setMode(String m) {
    mode = m;
    _safeNotify();
  }

  // Safe notify - langsung
  void _safeNotify() {
    if (hasListeners) notifyListeners();
  }



  void setRamadhan(bool aktif, {int? menit, int? tahun}) {
    aktifRamadhan = aktif;
    if (menit != null) menitSebelumSubuh = menit;
    if (tahun != null) tahunRamadhan = tahun;
    _safeNotify();
    saveData();
  }

  // Hitung waktu imsak dari waktu subuh
  String get waktuImsak {
    try {
      final parts = waktuSubuh.split(':');
      final subuhDt = DateTime(2000, 1, 1, int.parse(parts[0]), int.parse(parts[1]));
      final imsakDt = subuhDt.subtract(Duration(minutes: menitSebelumSubuh));
      return '${imsakDt.hour.toString().padLeft(2,'0')}:${imsakDt.minute.toString().padLeft(2,'0')}';
    } catch (_) { return '04:35'; }
  }

  void tambahJadwalJumat(Map<String, dynamic> data) {
    jadwalJumat.add(data);
    jadwalJumat.sort((a, b) => a['tanggal'].compareTo(b['tanggal']));
    _safeNotify();
    saveData();
  }

  void hapusJadwalJumat(int index) {
    jadwalJumat.removeAt(index);
    _safeNotify();
    saveData();
  }

  void editJadwalJumat(int index, Map<String, dynamic> data) {
    jadwalJumat[index] = data;
    jadwalJumat.sort((a, b) => a['tanggal'].compareTo(b['tanggal']));
    _safeNotify();
    saveData();
  }

  void setSaldoAwal(double nilai) {
    saldoAwal = nilai;
    _safeNotify();
    SharedPreferences.getInstance().then((p) => p.setDouble('saldoAwal', nilai));
  }

  void setFirstLoginDone() {
    isFirstLogin = false;
    _safeNotify();
    SharedPreferences.getInstance().then((p) => p.setBool('isFirstLogin', false));
  }

  bool login(String pass) {
    if (pass == adminPass) {
      mode = "admin";
      // Tunggu frame selesai baru notify
      WidgetsBinding.instance.addPostFrameCallback((_) {
        if (hasListeners) notifyListeners();
      });
      return true;
    }
    return false;
  }

  void logout() {
    mode = "public";
    WidgetsBinding.instance.addPostFrameCallback((_) {
      if (hasListeners) notifyListeners();
    });
  }

  void toggleDarkMode() {
    isDarkMode = !isDarkMode;
    _safeNotify();
  }

  void generateUniqueCode() {
    final tahun = DateTime.now().year;
    final acak = Random().nextInt(9000) + 1000;
    houseUniqueCode = "NP-$tahun-$acak";
    _safeNotify();
    saveData();
  }

  DateTime _parseTgl(String tgl) {
    try {
      final parts = tgl.split('/');
      if (parts.length == 3) {
        return DateTime(int.parse(parts[2]), int.parse(parts[1]), int.parse(parts[0]));
      }
    } catch (_) {}
    return DateTime.now();
  }

  void addTransaksi(String tgl, String keterangan, String jenis, double jumlah, {String donatur = ''}) {
    // Update akumulasi permanen
    if (jenis == 'masuk') {
      totalMasukSimpan += jumlah;
    } else {
      totalKeluarSimpan += jumlah;
    }
    SharedPreferences.getInstance().then((p) {
      p.setDouble('totalMasukSimpan', totalMasukSimpan);
      p.setDouble('totalKeluarSimpan', totalKeluarSimpan);
    });
    final newList = List<Map<String, dynamic>>.from(transaksi)
      ..add(<String, dynamic>{
        'tgl': tgl,
        'keterangan': keterangan,
        'jenis': jenis,
        'jumlah': jumlah,
        'donatur': donatur,
      });
    // Sort berdasarkan tanggal terlama ke terbaru
    newList.sort((a, b) => _parseTgl(a['tgl']).compareTo(_parseTgl(b['tgl'])));
    transaksi = newList;
    _safeNotify();
    saveTransaksi();
  }

  void deleteTransaksi(int index) {
    // Tidak perlu update totalMasukSimpan - sudah tersimpan saat addTransaksi
    transaksi.removeAt(index);
    _safeNotify();
    saveTransaksi();
  }

  // Hapus semua riwayat tapi simpan snapshot dulu
  void hapusSemualRiwayat() {
    // totalMasukSimpan sudah benar - tidak perlu update
    transaksi.clear();
    _safeNotify();
    saveTransaksi();
  }

  // Simpan snapshot total sebelum hapus semua riwayat
  void simpanSnapshot() {
    totalMasukSimpan += totalMasuk;
    totalKeluarSimpan += totalKeluar;
    SharedPreferences.getInstance().then((p) {
      p.setDouble('totalMasukSimpan', totalMasukSimpan);
      p.setDouble('totalKeluarSimpan', totalKeluarSimpan);
    });
  }

  void resetSnapshot() {
    totalMasukSimpan = 0.0;
    totalKeluarSimpan = 0.0;
    SharedPreferences.getInstance().then((p) {
      p.setDouble('totalMasukSimpan', 0.0);
      p.setDouble('totalKeluarSimpan', 0.0);
    });
  }

  double get totalMasuk => transaksi
      .where((t) => t['jenis'] == 'masuk')
      .fold(0.0, (sum, t) => sum + (t['jumlah'] as double));

  double get totalKeluar => transaksi
      .where((t) => t['jenis'] == 'keluar')
      .fold(0.0, (sum, t) => sum + (t['jumlah'] as double));

  // Getter pakai totalMasukSimpan - sudah include semua transaksi
  double get totalMasukAll => totalMasukSimpan;
  double get totalKeluarAll => totalKeluarSimpan;
  double get saldo => saldoAwal + totalMasukAll - totalKeluarAll;
  double get saldoAkhir => saldo;

  Future<void> saveTransaksi() async {
    try {
      final encoded = transaksi
          .map((t) => '${t['tgl']}|${t['keterangan']}|${t['jenis']}|${t['jumlah']}|${t['donatur'] ?? ''}')
          .toList();
      await (await SharedPreferences.getInstance()).setString('transaksi_data', encoded.join(';;'));
      await (await SharedPreferences.getInstance()).setDouble('saldoAwal', saldoAwal);
    } catch (_) {}
    final prefs = await SharedPreferences.getInstance();
    final List<String> encoded = transaksi.map((t) =>
        "${t['tgl']}|${t['keterangan']}|${t['jenis']}|${t['jumlah']}|${t['donatur'] ?? ''}").toList();
    await prefs.setStringList('transaksi', encoded);
  }

  Future<void> loadTransaksi() async {
    // Coba load dari localStorage dulu
    try {
      final raw = (await SharedPreferences.getInstance()).getString('transaksi_data');
      if (raw != null && raw.isNotEmpty) {
        final parts = raw.split(';;');
        transaksi = parts.where((s) => s.isNotEmpty).map((s) {
          final p = s.split('|');
          return Map<String, dynamic>.from({
            'tgl':        p.isNotEmpty ? p[0] : '',
            'keterangan': p.length > 1 ? p[1] : '',
            'jenis':      p.length > 2 ? p[2] : 'masuk',
            'jumlah':     p.length > 3 ? double.tryParse(p[3]) ?? 0.0 : 0.0,
            'donatur':    p.length > 4 ? p[4] : '',
          });
        }).toList();
        transaksi.sort((a, b) => _parseTgl(a['tgl']).compareTo(_parseTgl(b['tgl'])));
        _safeNotify();
        return;
      }
    } catch (_) {}
    // Fallback ke SharedPreferences
    final prefs = await SharedPreferences.getInstance();
    final List<String> encoded = prefs.getStringList('transaksi') ?? [];
    transaksi = encoded.map((s) {
      final parts = s.split('|');
      return Map<String, dynamic>.from({
        'tgl':        parts[0],
        'keterangan': parts[1],
        'jenis':      parts[2],
        'jumlah':     double.tryParse(parts[3]) ?? 0.0,
        'donatur':    parts.length > 4 ? parts[4] : '',
      });
    }).toList();
    transaksi.sort((a, b) => _parseTgl(a['tgl']).compareTo(_parseTgl(b['tgl'])));
  }

  void addMedia(String path, String type, String judul) {
    galeri.add({
      'path': path,
      'type': type,
      'durasi': durasiSlideDefault,
      'judul': judul,
    });
    _safeNotify();
    saveGaleri();
  }

  Future<void> addMediaBase64(String sourcePath, String type, String judul) async {
    try {
      final tglSekarang = DateFormat('dd/MM/yyyy').format(DateTime.now());
      String finalPath;

      if (kIsWeb) {
        // Web: simpan sebagai base64 karena tidak ada file system
        final xfile = XFile(sourcePath);
        final bytes = await xfile.readAsBytes();
        final mime = type == 'video' ? 'video/mp4' : 'image/jpeg';
        finalPath = 'data:$mime;base64,${base64Encode(bytes)}';
      } else {
        // Android: copy ke app documents directory
        final dir = await getApplicationDocumentsDirectory();
        final ext = type == 'video' ? 'mp4' : 'jpg';
        final fileName = 'media_${DateTime.now().millisecondsSinceEpoch}.$ext';
        final destFile = File('${dir.path}/$fileName');
        final xfile = XFile(sourcePath);
        final bytes = await xfile.readAsBytes();
        await destFile.writeAsBytes(bytes);
        finalPath = destFile.path;
      }

      galeri = List<Map<String, dynamic>>.from(galeri)
        ..add(<String, dynamic>{
          'path': finalPath,
          'type': type,
          'durasi': durasiSlideDefault,
          'judul': judul,
          'tgl': tglSekarang,
        });
      await saveGaleriBase64();
      if (!hasListeners) return;
      notifyListeners();
    } catch (e) {
      debugPrint('addMediaBase64 error: $e');
    }
  }

  Future<void> saveGaleriBase64() async {
    try {
      final keys = galeri.map((g) =>
        "${g['path']}|${g['type']}|${g['durasi']}|${g['judul'] ?? ''}|${g['tgl'] ?? ''}").toList();
      await (await SharedPreferences.getInstance()).setString('galeri_index', keys.join(';;'));
    } catch (e) {
      debugPrint('saveGaleriBase64 error: $e');
    }
    // TIDAK ada notifyListeners di sini - sudah ada di addMediaBase64
  }

  Future<void> loadGaleriBase64() async {
    try {
      final raw = (await SharedPreferences.getInstance()).getString('galeri_index');
      if (raw != null && raw.isNotEmpty) {
        final keys = raw.split(';;');
        final result = <Map<String, dynamic>>[];
        for (final key in keys) {
          final parts = key.split('|');
          if (parts.length < 4) continue;
          if (parts[0] == 'b64') continue; // skip entri base64 lama
          final f = File(parts[0]);
          if (await f.exists()) {
            result.add({
              'path': parts[0],
              'type': parts[1],
              'durasi': int.tryParse(parts[2]) ?? 10,
              'judul': parts[3],
              'tgl': parts.length > 4 ? parts[4] : '',
            });
          }
        }
        if (result.isNotEmpty) {
          galeri = result;
          _safeNotify();
          return;
        }
      }
    } catch (e) {
      debugPrint('loadGaleriBase64 error: $e');
    }
    await loadGaleri();
  }

  Future<void> deleteMedia(int index) async {
    try {
      final path = galeri[index]['path'] as String? ?? '';
      if (path.isNotEmpty && !path.startsWith('base64:')) {
        final f = File(path);
        if (await f.exists()) await f.delete();
      }
    } catch (_) {}
    galeri = List<Map<String, dynamic>>.from(galeri)..removeAt(index);
    _safeNotify();
    await saveGaleriBase64();
  }

  void updateDurasiSlide(int detik) {
    durasiSlideDefault = detik;

    for (var item in galeri) {
      if (item['type'] == 'image') item['durasi'] = detik;
    }
    _safeNotify();
    saveGaleri();
    saveData();
  }

  Future<void> saveGaleri() async {
    final prefs = await SharedPreferences.getInstance();
    final encoded = galeri
        .map((g) => "${g['path']}|${g['type']}|${g['durasi']}|${g['judul']}")
        .toList();
    await prefs.setStringList('galeri', encoded);
    await prefs.setInt('durasiSlide', durasiSlideDefault);
    await prefs.setInt('koreksiMenit', koreksiMenit);
    await prefs.setInt('menitIqomah', menitIqomah);
    await prefs.setString('bahasa', bahasa);
    await prefs.setDouble('kecepatanRunningText', kecepatanRunningText);
    await prefs.setString('warnaRunningText', warnaRunningText);
    await prefs.setBool('smartDisplayAktif', smartDisplayAktif);
  }

  Future<void> loadGaleri() async {
    final prefs = await SharedPreferences.getInstance();
    durasiSlideDefault = prefs.getInt('durasiSlide') ?? 10;
    final encoded = prefs.getStringList('galeri') ?? [];
    galeri = encoded.map((s) {
      final parts = s.split('|');
      return Map<String, dynamic>.from({
        'path': parts[0],
        'type': parts[1],
        'durasi': int.tryParse(parts[2]) ?? 10,
        'judul': parts.length > 3 ? parts[3] : '',
      });
    }).toList();
  }

  Future<Map<String, dynamic>> eksporData() async {
    return {
      'versi': '1.0',
      'tglBackup': DateTime.now().toIso8601String(),
      'namaIbadah': namaIbadah,
      'alamat': alamat,
      'runningText': runningText,
      'houseUniqueCode': houseUniqueCode,
      'adminPass': adminPass,
      'jadwalJumat': jadwalJumat,
      'aktifRamadhan': aktifRamadhan,
      'menitSebelumSubuh': menitSebelumSubuh,
      'tahunRamadhan': tahunRamadhan,
      'isFirstLogin': isFirstLogin,
      'pengurus': pengurus,
      'waktuSubuh': waktuSubuh,
      'waktuDzuhur': waktuDzuhur,
      'waktuAshar': waktuAshar,
      'waktuMaghrib': waktuMaghrib,
      'waktuIsya': waktuIsya,
      'koreksiMenit': koreksiMenit,
      'menitIqomah': menitIqomah,
      'bahasa': bahasa,
      'kecepatanRunningText': kecepatanRunningText,
      'warnaRunningText': warnaRunningText,
      'smartDisplayAktif': smartDisplayAktif,
      'transaksi': transaksi,
      'galeri': galeri,
      'inventaris': inventaris,
    };
  }

  Future<String> simpanBackup() async {
    try {
      final data  = await eksporData();
      final json  = jsonEncode(data);
      final dir   = await getExternalStorageDirectory();
      final tgl   = DateFormat('yyyyMMdd_HHmm').format(DateTime.now());
      final file  = File('${dir!.path}/novapro_backup_$tgl.json');
      await file.writeAsString(json);
      return file.path;
    } catch (e) {
      return '';
    }
  }

  Future<bool> importBackup(String jsonString) async {
    try {
      final data = jsonDecode(jsonString) as Map<String, dynamic>;
      namaIbadah           = data['namaIbadah']           ?? namaIbadah;
      alamat               = data['alamat']               ?? alamat;
      runningText          = data['runningText']          ?? runningText;
      houseUniqueCode      = data['houseUniqueCode']      ?? houseUniqueCode;
      adminPass            = data['adminPass']            ?? adminPass;
      isFirstLogin           = data['isFirstLogin']           ?? true;
      if (data['jadwalJumat'] != null) {
        jadwalJumat = List<Map<String, dynamic>>.from(data['jadwalJumat']);
      }
      aktifRamadhan     = data['aktifRamadhan']     ?? false;
      menitSebelumSubuh = data['menitSebelumSubuh'] ?? 10;
      tahunRamadhan     = data['tahunRamadhan']     ?? DateTime.now().year;
      if (data['pengurus'] != null) {
        pengurus = List<Map<String, String>>.from(
          (data['pengurus'] as List).map((e) => Map<String, String>.from(e)));
      }
      waktuSubuh           = data['waktuSubuh']           ?? waktuSubuh;
      waktuDzuhur          = data['waktuDzuhur']          ?? waktuDzuhur;
      waktuAshar           = data['waktuAshar']           ?? waktuAshar;
      waktuMaghrib         = data['waktuMaghrib']         ?? waktuMaghrib;
      waktuIsya            = data['waktuIsya']            ?? waktuIsya;
      koreksiMenit         = data['koreksiMenit']         ?? koreksiMenit;
      menitIqomah          = data['menitIqomah']          ?? menitIqomah;
      bahasa               = data['bahasa']               ?? bahasa;
      kecepatanRunningText = (data['kecepatanRunningText'] as num?)?.toDouble() ?? kecepatanRunningText;
      warnaRunningText     = data['warnaRunningText']     ?? warnaRunningText;
      smartDisplayAktif    = data['smartDisplayAktif']    ?? smartDisplayAktif;
      if (data['transaksi'] != null) {
        transaksi = List<Map<String, dynamic>>.from(data['transaksi']);
      }
      if (data['galeri'] != null) {
        galeri = List<Map<String, dynamic>>.from(data['galeri']);
      }
      await saveData();
      await saveTransaksi();
      await saveGaleri();
      _safeNotify();
      return true;
    } catch (e) {
      return false;
    }
  }

  String waktuDenganKoreksi(String waktuStr) {
    try {
      final parts = waktuStr.split(':');
      final dt = DateTime(0, 1, 1,
          int.parse(parts[0]), int.parse(parts[1]));
      final koreksi = dt.add(Duration(minutes: koreksiMenit));
      return '${koreksi.hour.toString().padLeft(2,'0')}:${koreksi.minute.toString().padLeft(2,'0')}';
    } catch (_) {
      return waktuStr;
    }
  }

  Map<String, dynamic> hitungMundurAdzan() {
    final now = DateTime.now();
    final waktuList = [
      {'nama': labelSubuh,   'waktu': waktuSubuh},
      {'nama': labelDzuhur,  'waktu': waktuDzuhur},
      {'nama': labelAshar,   'waktu': waktuAshar},
      {'nama': labelMaghrib, 'waktu': waktuMaghrib},
      {'nama': labelIsya,    'waktu': waktuIsya},
    ];
    for (final w in waktuList) {
      final parts = (w['waktu'] as String).split(':');
      final target = DateTime(now.year, now.month, now.day,
          int.parse(parts[0]), int.parse(parts[1]));
      final diff = target.difference(now);
      if (diff.inSeconds > 0 && diff.inMinutes <= 60) {
        return {
          'nama': w['nama'],
          'menitLagi': diff.inMinutes,
          'detikLagi': diff.inSeconds % 60,
          'modeIqomah': diff.inMinutes <= menitIqomah,
        };
      }
    }
    return {'nama': '', 'menitLagi': -1, 'detikLagi': 0, 'modeIqomah': false};
  }

  void updateKoreksi(int menit, int iqomah) {
    koreksiMenit = menit;
    menitIqomah  = iqomah;
    _safeNotify();
    saveData();
  }

  void setBahasa(String b) {
    bahasa = b;
    _safeNotify();
    saveData();
  }

  String get labelSubuh   => bahasa=='en' ? 'FAJR'   : bahasa=='ar' ? 'الفجر'   : 'SUBUH';
  String get labelDzuhur  => bahasa=='en' ? 'DHUHR'  : bahasa=='ar' ? 'الظهر'   : 'DZUHUR';
  String get labelAshar   => bahasa=='en' ? 'ASR'    : bahasa=='ar' ? 'العصر'   : 'ASHAR';
  String get labelMaghrib => bahasa=='en' ? 'MAGHRIB': bahasa=='ar' ? 'المغرب'  : 'MAGHRIB';
  String get labelIsya    => bahasa=='en' ? 'ISHA'   : bahasa=='ar' ? 'العشاء'  : 'ISYA';

  // ── LABEL UI MULTI BAHASA ──
  String _t(String id, String en, String ar) =>
      bahasa == 'en' ? en : bahasa == 'ar' ? ar : id;

  // Terjemah jabatan default masjid
  String terjemahJabatan(String jabatan) {
    if (bahasa == 'en') {
      if (jabatan == 'Ketua') return 'Chairman';
      if (jabatan == 'Bendahara') return 'Treasurer';
      if (jabatan == 'Sekretaris') return 'Secretary';
      if (jabatan == 'Wakil Ketua') return 'Vice Chairman';
      if (jabatan == 'Ketua Umum') return 'General Chairman';
    } else if (bahasa == 'ar') {
      if (jabatan == 'Ketua') return 'الرئيس';
      if (jabatan == 'Bendahara') return 'أمين الصندوق';
      if (jabatan == 'Sekretaris') return 'السكرتير';
      if (jabatan == 'Wakil Ketua') return 'نائب الرئيس';
      if (jabatan == 'Ketua Umum') return 'الرئيس العام';
    }
    return jabatan;
  }

  // Terjemah kondisi inventaris
  String terjemahKondisi(String kondisi) {
    if (bahasa == 'en') {
      if (kondisi == 'Baik') return 'Good';
      if (kondisi == 'Rusak Ringan') return 'Minor Damage';
      if (kondisi == 'Rusak Berat') return 'Major Damage';
      return kondisi;
    } else if (bahasa == 'ar') {
      if (kondisi == 'Baik') return 'جيد';
      if (kondisi == 'Rusak Ringan') return 'تلف خفيف';
      if (kondisi == 'Rusak Berat') return 'تلف شديد';
      return kondisi;
    }
    return kondisi;
  }

  // Format tanggal sesuai bahasa - input: dd/MM/yyyy
  String formatTgl(String tgl) {
    if (tgl.isEmpty) return tgl;
    try {
      final parts = tgl.split('/');
      if (parts.length == 3) {
        final dt = DateTime(int.parse(parts[2]), int.parse(parts[1]), int.parse(parts[0]));
        final locale = bahasa == 'ar' ? 'ar' : bahasa == 'en' ? 'en' : 'id';
        try { return DateFormat('d MMM yyyy', locale).format(dt); }
        catch (_) { return DateFormat('d MMM yyyy').format(dt); }
      }
    } catch (_) {}
    return tgl;
  }

  // Navigasi
  String get lBeranda    => _t('Beranda',    'Home',       'الرئيسية');
  String get lPengurus   => _t('Pengurus',   'Committee',  'الهيئة');
  String get lGaleri     => _t('Galeri',     'Gallery',    'المعرض');
  String get lKas        => _t('Kas',        'Finance',    'الصندوق');
  String get lDonasi     => _t('Donasi',     'Donation',   'التبرع');

  // Umum
  String get lSimpan     => _t('Simpan',     'Save',       'حفظ');
  String get lBatal      => _t('Batal',      'Cancel',     'إلغاء');
  String get lHapus      => _t('Hapus',      'Delete',     'حذف');
  String get lEdit       => _t('Edit',       'Edit',       'تعديل');
  String get lTambah     => _t('Tambah',     'Add',        'إضافة');
  String get lYa         => _t('Ya',         'Yes',        'نعم');
  String get lTidak      => _t('Tidak',      'No',         'لا');
  String get lNama       => _t('Nama',       'Name',       'الاسم');
  String get lTanggal    => _t('Tanggal',    'Date',       'التاريخ');
  String get lKeterangan => _t('Keterangan', 'Note',       'ملاحظة');
  String get lJumlah     => _t('Jumlah',     'Amount',     'المبلغ');
  String get lKeluar     => _t('Keluar',     'Logout',     'خروج');
  String get lPosting    => _t('POSTING',    'POST',       'نشر');

  // Jadwal sholat
  String get lJadwalSholat  => _t('Jadwal Sholat',  'Prayer Times',   'أوقات الصلاة');
  String get lJadwalManual  => _t('Jadwal Manual',  'Manual Schedule','جدول يدوي');
  String get lKoreksiMenit  => _t('Koreksi Ikhtiyat','Time Correction','تصحيح الوقت');
  String get lIqomah        => _t('Iqomah',         'Iqamah',         'الإقامة');

  // Pengumuman
  String get lPengumuman         => _t('Pengumuman',           'Announcement',       'الإعلانات');
  String get lTulisPengumuman    => _t('Tulis Pengumuman',     'Write Announcement', 'كتابة الإعلان');
  String get lBelumAdaPengumuman => _t('Belum ada pengumuman', 'No announcements yet','لا إعلانات');
  String get lPengumumanDitambah => _t('✅ Pengumuman ditambahkan!','✅ Announcement added!','✅ تمت الإضافة!');
  String get lTambahFotoOpsional => _t('Tap untuk tambah foto (opsional)','Tap to add photo (optional)','اضغط لإضافة صورة (اختياري)');

  // Profil
  String get lEditProfil  => _t('Edit Profil',     'Edit Profile',    'تعديل الملف');
  String get lNamaMasjid  => _t('Nama Masjid/Musholla','Mosque Name', 'اسم المسجد');
  String get lAlamat      => _t('Alamat',           'Address',         'العنوان');
  String get lRunningText => _t('Running Text',     'Running Text',    'نص متحرك');

  // Pengurus
  String get lPengurus2       => _t('Pengurus',      'Committee',   'الهيئة الإدارية');
  String get lJabatan         => _t('Jabatan',       'Position',    'المنصب');
  String get lTambahPengurus  => _t('Tambah Pengurus','Add Member', 'إضافة عضو');
  String get lEditPengurus    => _t('Edit Pengurus', 'Edit Member', 'تعديل العضو');
  String get lHapusPengurus   => _t('Hapus Pengurus','Remove',      'حذف العضو');

  // Kas
  String get lArusKas     => _t('Arus Kas',     'Cash Flow',    'التدفق النقدي');
  String get lPemasukan   => _t('Pemasukan',    'Income',       'الدخل');
  String get lPengeluaran => _t('Pengeluaran',  'Expense',      'المصروف');
  String get lSaldo       => _t('Saldo',        'Balance',      'الرصيد');
  String get lMasuk       => _t('Masuk',        'In',           'وارد');
  String get lKeluar2     => _t('Keluar',       'Out',          'صادر');
  String get lTransaksi   => _t('Transaksi',    'Transaction',  'معاملة');
  String get lTambahTransaksi => _t('Tambah Transaksi','Add Transaction','إضافة معاملة');
  String get lHapusTransaksi  => _t('Hapus Transaksi?','Delete Transaction?','حذف المعاملة؟');
  String get lEditTransaksi   => _t('Edit Transaksi',  'Edit Transaction',  'تعديل المعاملة');
  String get lBelumAdaTransaksi => _t('Belum ada transaksi','No transactions yet','لا معاملات');
  String get lSaldoTidakCukup => _t('Saldo tidak cukup!','Insufficient balance!','رصيد غير كافٍ!');
  String get lKetKosong   => _t('⚠️ Keterangan tidak boleh kosong!','⚠️ Note cannot be empty!','⚠️ الملاحظة مطلوبة!');
  String get lJumlahNol   => _t('⚠️ Jumlah harus lebih dari 0!','⚠️ Amount must be > 0!','⚠️ المبلغ يجب أن يكون > 0!');
  String get lAdaDonatur  => _t('Ada Donatur',  'Has Donor',    'يوجد متبرع');
  String get lAnonim      => _t('Anonim',       'Anonymous',    'مجهول');
  String get lNamaDonatur => _t('Nama Donatur', 'Donor Name',   'اسم المتبرع');

  // Donatur
  String get lDonatur       => _t('Donatur',      'Donor',         'المتبرع');
  String get lTambahDonatur => _t('Tambah Donatur','Add Donor',     'إضافة متبرع');
  String get lHapusDonatur  => _t('Hapus Donatur?','Delete Donor?', 'حذف المتبرع؟');
  String get lYakinHapusDonasi => _t('Yakin hapus data donasi ini?','Delete this donation?','حذف هذه البيانات؟');
  String get lBelumAdaDonatur  => _t('Belum ada data donatur','No donor data yet','لا بيانات متبرعين');
  String get lTotalDonasi      => _t('Total Donasi','Total Donation','إجمالي التبرع');
  String get lKategori         => _t('Kategori',   'Category',      'الفئة');

  // Inventaris
  String get lInventaris      => _t('Inventaris',     'Inventory',    'المخزون');
  String get lNamaBarang      => _t('Nama Barang',    'Item Name',    'اسم الصنف');
  String get lSatuan          => _t('Satuan',         'Unit',         'الوحدة');
  String get lKondisi         => _t('Kondisi',        'Condition',    'الحالة');
  String get lTambahInventaris=> _t('Tambah Inventaris','Add Item',   'إضافة صنف');
  String get lEditInventaris  => _t('Edit Inventaris', 'Edit Item',   'تعديل الصنف');
  String get lBelumAdaInventaris => _t('Belum ada inventaris','No inventory yet','لا مخزون');
  String get lNamaBarangKosong=> _t('⚠️ Nama barang tidak boleh kosong!','⚠️ Item name required!','⚠️ اسم الصنف مطلوب!');

  // Acara
  String get lAcara       => _t('Acara',        'Events',       'الفعاليات');
  String get lJadwalJumat  => _t('Jadwal Jumat',  'Friday Schedule',  'جدول الجمعة');
  String get lRamadhan     => _t('Ramadhan',      'Ramadan',          'رمضان');
  String get lImsak        => _t('Imsak',         'Imsak',            'إمساك');
  String get lBerbuka      => _t('Berbuka',        'Iftar',            'إفطار');
  String get lSahur        => _t('Sahur',          'Suhoor',           'سحور');
  String get lJadwalImsak  => _t('Jadwal Imsak',   'Imsak Schedule',   'جدول الإمساك');
  String get lImam        => _t('Imam',         'Imam',            'الإمام');
  String get lKhatib      => _t('Khatib',       'Khatib',          'الخطيب');
  String get lTemaKhutbah => _t('Tema Khutbah', 'Sermon Topic',    'موضوع الخطبة');
  String get lNamaAcara   => _t('Nama Acara',   'Event Name',   'اسم الفعالية');
  String get lSambutan    => _t('Sambutan',      'Speech',       'كلمة ترحيب');
  String get lTamu        => _t('Tamu',          'Guest',        'الضيف');
  String get lTambahAcara => _t('Tambah Acara',  'Add Event',    'إضافة فعالية');
  String get lTambahTamu  => _t('Tambah Tamu',   'Add Guest',    'إضافة ضيف');

  // Galeri
  String get lGaleri2     => _t('Galeri',        'Gallery',      'المعرض');
  String get lTambahMedia => _t('Tambah Media',  'Add Media',    'إضافة وسائط');
  String get lJudul       => _t('Judul',         'Title',        'العنوان');
  String get lDurasi      => _t('Durasi (detik)','Duration (s)', 'المدة (ث)');

  // Pengaturan
  String get lPengaturan     => _t('Pengaturan',    'Settings',      'الإعدادات');
  String get lBahasa         => _t('Bahasa',        'Language',      'اللغة');
  String get lDarkMode       => _t('Mode Gelap',    'Dark Mode',     'الوضع الليلي');
  String get lGantiPassword  => _t('Ganti Password','Change Password','تغيير كلمة المرور');
  String get lPasswordLama   => _t('Password Lama', 'Old Password',  'كلمة المرور القديمة');
  String get lPasswordBaru   => _t('Password Baru', 'New Password',  'كلمة المرور الجديدة');
  String get lSmartDisplay   => _t('Smart Display', 'Smart Display', 'العرض الذكي');

  // Login admin
  String get lLoginAdmin    => _t('Login Admin',        'Admin Login',     'دخول المسؤول');
  String get lPassword      => _t('Password',           'Password',        'كلمة المرور');
  String get lLoginBtn      => _t('MASUK',              'LOGIN',           'دخول');
  String get lPasswordSalah => _t('Password salah!',    'Wrong password!', 'كلمة المرور خاطئة!');

  // Laporan & cetak
  String get lLaporan       => _t('Laporan Keuangan',   'Financial Report','التقرير المالي');
  String get lCetak         => _t('Cetak ke Printer',   'Print',           'طباعة');
  String get lSimpanPdf     => _t('Simpan PDF ke HP',   'Save PDF',        'حفظ PDF');
  String get lPratinjauPdf  => _t('Pratinjau PDF',      'Preview PDF',     'معاينة PDF');
  String get lTransaksiDiperbarui => _t('Transaksi diperbarui!','Transaction updated!','تم التحديث!');

  String get tanggalHijriah => tanggalHijriahBahasa(bahasa);

  String tanggalHijriahBahasa(String lang) {
    final now   = DateTime.now();
    final epoch = DateTime(622, 7, 16);
    final selisih = now.difference(epoch).inDays;
    final tahunH  = (selisih / 354.367).floor() + 1;
    final sisaH   = selisih % 354;
    final bulanH  = (sisaH / 29.5).floor() + 1;
    final hariH   = (sisaH % 30) + 1;
    final bln = bulanH.clamp(1, 12);

    // Nama bulan sesuai bahasa
    const namaBulanID = [
      "", "Muharram", "Safar", "Rabi'ul Awal", "Rabi'ul Akhir",
      "Jumadil Awal", "Jumadil Akhir", "Rajab", "Sya'ban",
      "Ramadhan", "Syawal", "Dzulqa'dah", "Dzulhijjah"
    ];
    const namaBulanEN = [
      "", "Muharram", "Safar", "Rabi al-Awwal", "Rabi al-Thani",
      "Jumada al-Awwal", "Jumada al-Thani", "Rajab", "Sha'ban",
      "Ramadan", "Shawwal", "Dhu al-Qi'dah", "Dhu al-Hijjah"
    ];
    const namaBulanAR = [
      "", "مُحَرَّم", "صَفَر", "رَبِيع الأَوَّل", "رَبِيع الآخِر",
      "جُمَادَى الأُولَى", "جُمَادَى الآخِرَة", "رَجَب", "شَعْبَان",
      "رَمَضَان", "شَوَّال", "ذُو القَعْدَة", "ذُو الحِجَّة"
    ];

    if (lang == 'en') {
      return "$hariH ${namaBulanEN[bln]} $tahunH H";
    } else if (lang == 'ar') {
      // Angka Arab
      final angkaArab = ['٠','١','٢','٣','٤','٥','٦','٧','٨','٩'];
      String toArab(int n) {
        return n.toString().characters.map((c) {
          final d = int.tryParse(c);
          return d != null ? angkaArab[d] : c;
        }).join();
      }
      return "${toArab(hariH)} ${namaBulanAR[bln]} ${toArab(tahunH)} هـ";
    }
    return "$hariH ${namaBulanID[bln]} $tahunH H";
  }

  void toggleSmartDisplay() {
    smartDisplayAktif = !smartDisplayAktif;
    _safeNotify();
    saveData();
  }

  int durasiAdzanMenit = 10; // default 10 menit
  Timer? _adzanTimer;

  void setSedangAdzan(bool v) {
    sedangAdzan = v;
    _adzanTimer?.cancel();
    if (v) {
      _adzanTimer = Timer(Duration(minutes: durasiAdzanMenit), () {
        sedangAdzan = false;
        _safeNotify();
      });
    }
    _safeNotify();
  }

  void setDurasiAdzan(int menit) {
    durasiAdzanMenit = menit.clamp(1, 30);
    _safeNotify();
    saveData();
  }

  void updateRunningTextSettings(double kecepatan, String warna) {
    kecepatanRunningText = kecepatan.clamp(5.0, 60.0);
    warnaRunningText     = warna;
    _safeNotify();
    saveData();
  }

  void tambahPengumuman(String teks, {String foto = ''}) {
    pengumuman.insert(0, {
      'teks': teks,
      'waktu': DateTime.now().toString(),
      'foto': foto,
    });
    _safeNotify();
    _savePengumuman();
  }

  void hapusPengumuman(int index) {
    pengumuman.removeAt(index);
    _safeNotify();
    _savePengumuman();
  }

  Future<void> _savePengumuman() async {
    final data = pengumuman.map((e) => '${e["teks"]}||${e["waktu"]}||${e["foto"] ?? ""}').toList().join(';;;');
    try {
      await (await SharedPreferences.getInstance()).setString('pengumuman', data.toString());
    } catch (_) {
      final prefs = await SharedPreferences.getInstance();
      await prefs.setString('pengumuman', data);
    }
  }

  Future<void> _loadPengumuman() async {
    String raw = '';
    try {
      raw = (await SharedPreferences.getInstance()).getString('pengumuman') ?? '';
    } catch (_) {
      final prefs = await SharedPreferences.getInstance();
      raw = prefs.getString('pengumuman') ?? '';
    }
    if (raw.isEmpty) { pengumuman = []; return; }
    pengumuman = raw.split(';;;').map<Map<String, dynamic>>((e) {
      final parts = e.split('||');
      return Map<String, dynamic>.from({
        'teks': parts.isNotEmpty ? parts[0] : '',
        'waktu': parts.length > 1 ? parts[1] : '',
        'foto': parts.length > 2 ? parts[2] : '',
      });
    }).toList();
  }

  // Deteksi kota dari alamat dan update waktu sholat otomatis
  // Hanya update jika admin belum pernah edit manual
  void _autoUpdateWaktuSholat(String alamatBaru) {
    if (jadwalManual) return; // admin sudah edit manual, jangan ditimpa
    final kota = SholatCalculator.cariKota(alamatBaru);
    if (kota != null) {
      final waktu = SholatCalculator.hitungWaktu(kota.lat, kota.lng, kota.timezone, koreksi: koreksiMenit);
      waktuSubuh   = waktu['subuh']   ?? waktuSubuh;
      waktuDzuhur  = waktu['dzuhur']  ?? waktuDzuhur;
      waktuAshar   = waktu['ashar']   ?? waktuAshar;
      waktuMaghrib = waktu['maghrib'] ?? waktuMaghrib;
      waktuIsya    = waktu['isya']    ?? waktuIsya;
    }
  }


  // Running text otomatis dari nama masjid + alamat + opsional user
  String get runningTextAktif {
    final nama = namaIbadah.trim();
    final almt = alamat.trim();
    final opsional = runningText.trim();

    // Bagian otomatis: SELAMAT DATANG DI [NAMA] - [ALAMAT]
    String auto = '';
    if (nama.isNotEmpty) {
      final welcome = bahasa == 'en' ? 'WELCOME TO' : bahasa == 'ar' ? 'أهلاً بكم في' : 'SELAMAT DATANG DI';
      auto = '$welcome $nama';
      if (almt.isNotEmpty) auto += ' - $almt';
    }

    // Gabungkan: otomatis + opsional user
    if (auto.isNotEmpty && opsional.isNotEmpty) {
      return '$auto  ✦  $opsional';
    } else if (auto.isNotEmpty) {
      return auto;
    } else if (opsional.isNotEmpty) {
      return opsional;
    }
    return '';
  }

  void updateProfil(String nama, String almt, String running) {
    namaIbadah = nama;
    alamat = almt;
    runningText = running;
    _autoUpdateWaktuSholat(almt);
    _safeNotify();
    saveData();
  }

  Future<void> saveProfilAsync(String nama, String almt, String running) async {
    namaIbadah = nama;
    alamat = almt;
    runningText = running;
    _autoUpdateWaktuSholat(almt);
    _safeNotify();
    try {
      await (await SharedPreferences.getInstance()).setString('namaIbadah', nama.toString());
      await (await SharedPreferences.getInstance()).setString('alamat', almt.toString());
      // Simpan runningText dalam chunks jika panjang
      await (await SharedPreferences.getInstance()).remove('runningText');
      final chunks = <String>[];
      const chunkSize = 500;
      for (var i = 0; i < running.length; i += chunkSize) {
        chunks.add(running.substring(i, i + chunkSize > running.length ? running.length : i + chunkSize));
      }
      await (await SharedPreferences.getInstance()).setString('runningTextChunks', chunks.length.toString());
      for (var i = 0; i < chunks.length; i++) {
        await (await SharedPreferences.getInstance()).setString('runningText_$i', chunks[i].toString());
      }
    } catch (_) {}
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString('namaIbadah', namaIbadah);
    await prefs.setString('alamat', alamat);
    await prefs.setString('runningText', runningText);
  }

  Future<void> updateFotoMasjid(String base64) async {
    fotoMasjid = base64;
    _safeNotify();
    try {
      await (await SharedPreferences.getInstance()).setString('fotoMasjid', base64.toString());
    } catch (_) {
      final prefs = await SharedPreferences.getInstance();
      await prefs.setString('fotoMasjid', base64);
    }
  }

  void tambahPengurus(String jabatan, String nama, {String foto = ''}) {
    pengurus.add({'jabatan': jabatan, 'nama': nama, 'foto': foto});
    _safeNotify();
    savePengurus();
  }

  void editPengurus(int index, String jabatan, String nama, {String? foto}) {
    pengurus[index] = {'jabatan': jabatan, 'nama': nama, 'foto': foto ?? pengurus[index]['foto'] ?? ''};
    _safeNotify();
    savePengurus();
  }

  void hapusPengurus(int index) {
    pengurus.removeAt(index);
    _safeNotify();
    // Bersihkan semua key lama dulu baru save ulang
    SharedPreferences.getInstance().then((prefs) async {
      // Hapus key yang sudah tidak perlu
      final oldCount = (prefs.getInt('pengurus_count') ?? 0);
      for (int i = pengurus.length; i < oldCount; i++) {
        await prefs.remove('pg_jab_$i');
        await prefs.remove('pg_nama_$i');
        await prefs.remove('pg_foto_$i');
      }
    });
    savePengurus();
  }

  Future<void> savePengurus() async {
    final prefs = await SharedPreferences.getInstance();
    // Simpan tiap pengurus secara terpisah agar foto base64 tidak terpotong
    await prefs.setInt('pengurus_count', pengurus.length);
    for (int i = 0; i < pengurus.length; i++) {
      await prefs.setString('pg_jab_$i', pengurus[i]['jabatan'] ?? '');
      await prefs.setString('pg_nama_$i', pengurus[i]['nama'] ?? '');
      await prefs.setString('pg_foto_$i', pengurus[i]['foto'] ?? '');
    }
  }

  Future<void> loadPengurus() async {
    final prefs = await SharedPreferences.getInstance();
    final count = prefs.getInt('pengurus_count') ?? 0;
    if (count > 0) {
      pengurus = List.generate(count, (i) => <String, dynamic>{
        'jabatan': prefs.getString('pg_jab_$i') ?? '',
        'nama':    prefs.getString('pg_nama_$i') ?? '-',
        'foto':    prefs.getString('pg_foto_$i') ?? '',
      });
      return;
    }
    // Fallback ke format lama (tanpa foto)
    final raw = prefs.getString('pengurus_data') ?? '';
    if (raw.isEmpty) return;
    pengurus = raw.split(';;;').map<Map<String, dynamic>>((s) {
      final parts = s.split('|');
      return <String, dynamic>{
        'jabatan': parts[0],
        'nama': parts.length > 1 ? parts[1] : '-',
        'foto': parts.length > 2 ? parts[2] : '',
      };
    }).toList();
  }

  void addInventaris(String nama, int jumlah, String satuan, String kondisi, String ket) {
    final tgl = DateFormat('dd/MM/yyyy').format(DateTime.now());
    inventaris.add(Map<String, dynamic>.from({
      'nama': nama,
      'jumlah': jumlah,
      'satuan': satuan,
      'kondisi': kondisi,
      'keterangan': ket,
      'tgl': tgl,
    }));
    _safeNotify();
    saveInventaris();
  }

  void editInventaris(int i, String nama, int jumlah, String satuan, String kondisi, String ket) {
    final tglLama = inventaris[i]['tgl'] ?? DateFormat('dd/MM/yyyy').format(DateTime.now());
    inventaris[i] = {
      'nama': nama,
      'jumlah': jumlah,
      'satuan': satuan,
      'kondisi': kondisi,
      'keterangan': ket,
      'tgl': tglLama, // preserve tanggal input awal
    };
    _safeNotify();
    saveInventaris();
  }

  void deleteInventaris(int i) {
    inventaris.removeAt(i);
    _safeNotify();
    saveInventaris();
  }

  Future<void> saveInventaris() async {
    final encoded = inventaris.map((inv) =>
        "${inv['nama']}|${inv['jumlah']}|${inv['satuan']}|${inv['kondisi']}|${inv['keterangan']}|${inv['tgl'] ?? ''}").toList();
    await (await SharedPreferences.getInstance()).setString('inventaris_data', encoded.join(';;;'));
  }

  Future<void> loadInventaris() async {
    final raw = (await SharedPreferences.getInstance()).getString('inventaris_data') ?? '';
    if (raw.isEmpty) { inventaris = []; return; }
    inventaris = raw.split(';;;').map((s) {
      final p = s.split('|');
      return Map<String, dynamic>.from({
        'nama':        p[0],
        'jumlah':      int.tryParse(p.length > 1 ? p[1] : '1') ?? 1,
        'satuan':      p.length > 2 ? p[2] : 'unit',
        'kondisi':     p.length > 3 ? p[3] : 'Baik',
        'keterangan':  p.length > 4 ? p[4] : '',
        'tgl':         p.length > 5 ? p[5] : '',
      });
    }).toList();
  }

  void editTransaksi(int index, String tgl, String keterangan, String jenis, double jumlah, {String donatur = ''}) {
    // Adjust akumulasi - kurangi nilai lama, tambah nilai baru
    final old = transaksi[index];
    final oldJumlah = (old['jumlah'] as double);
    if (old['jenis'] == 'masuk') totalMasukSimpan -= oldJumlah;
    else totalKeluarSimpan -= oldJumlah;
    if (jenis == 'masuk') totalMasukSimpan += jumlah;
    else totalKeluarSimpan += jumlah;
    SharedPreferences.getInstance().then((p) {
      p.setDouble('totalMasukSimpan', totalMasukSimpan);
      p.setDouble('totalKeluarSimpan', totalKeluarSimpan);
    });
    transaksi[index] = Map<String, dynamic>.from({
      'tgl': tgl,
      'keterangan': keterangan,
      'jenis': jenis,
      'jumlah': jumlah,
      'donatur': donatur,
    });
    _safeNotify();
    saveTransaksi();
  }

  void updateJadwal(String subuh, String dzuhur, String ashar, String maghrib, String isya) {
    waktuSubuh = subuh;
    waktuDzuhur = dzuhur;
    waktuAshar = ashar;
    waktuMaghrib = maghrib;
    waktuIsya = isya;
    jadwalManual = true; // tandai sudah edit manual, jangan ditimpa auto
    _safeNotify();
    saveData();
  }

  void updatePassword(String newPass) {
    adminPass = newPass;
    _safeNotify();
    saveData();
  }

  void startLive() {
    mode = "live";
    liveSeconds = 0;
    liveTimer = Timer.periodic(const Duration(seconds: 1), (_) {
      liveSeconds++; // hitung saja, tidak rebuild UI
    });
    _safeNotify();
  }

  void stopLive() {
    liveTimer?.cancel();
    liveTimer = null;
    mode = "admin";
    _safeNotify();
  }

  String get liveTime {
    final m = liveSeconds ~/ 60;
    final s = liveSeconds % 60;
    return "${m.toString().padLeft(2, '0')}:${s.toString().padLeft(2, '0')}";
  }

  Future<void> saveData() async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString('mode', mode == "live" ? "public" : mode);
    await prefs.setBool('isDarkMode', isDarkMode);
    await prefs.setString('namaIbadah', namaIbadah);
    await prefs.setString('alamat', alamat);
    await prefs.setString('runningText', runningText);
    await prefs.setString('houseUniqueCode', houseUniqueCode);
    await prefs.setString('adminPass', adminPass);
    await prefs.setBool('isFirstLogin', isFirstLogin);
    await savePengurus();
    await prefs.setString('waktuSubuh', waktuSubuh);
    await prefs.setString('waktuDzuhur', waktuDzuhur);
    await prefs.setString('waktuAshar', waktuAshar);
    await prefs.setString('waktuMaghrib', waktuMaghrib);
    await prefs.setString('waktuIsya', waktuIsya);
    await prefs.setInt('durasiSlide', durasiSlideDefault);
    await prefs.setString('bahasa', bahasa);
    await prefs.setInt('koreksiMenit', koreksiMenit);
    await prefs.setInt('menitIqomah', menitIqomah);
    await prefs.setDouble('kecepatanRunningText', kecepatanRunningText);
    await prefs.setString('warnaRunningText', warnaRunningText);
    await prefs.setBool('smartDisplayAktif', smartDisplayAktif);
    await prefs.setInt('durasiAdzanMenit', durasiAdzanMenit);
    // Simpan jadwal jumat
    try {
      final jj = jadwalJumat.map((j) =>
        '${j['tanggal']}||${j['imam']}||${j['khatib']}||${j['tema'] ?? ''}').toList();
      await prefs.setStringList('jadwalJumat', jj);
    } catch (_) {}
    // Simpan Ramadhan settings
    await prefs.setBool('aktifRamadhan', aktifRamadhan);
    await prefs.setInt('menitSebelumSubuh', menitSebelumSubuh);
    // Simpan waktu sholat ke localStorage juga
    try {
      await (await SharedPreferences.getInstance()).setString('waktuSubuh', waktuSubuh.toString());
      await (await SharedPreferences.getInstance()).setString('waktuDzuhur', waktuDzuhur.toString());
      await (await SharedPreferences.getInstance()).setString('waktuAshar', waktuAshar.toString());
      await (await SharedPreferences.getInstance()).setString('waktuMaghrib', waktuMaghrib.toString());
      await (await SharedPreferences.getInstance()).setString('waktuIsya', waktuIsya.toString());
      await (await SharedPreferences.getInstance()).setString('jadwalManual', jadwalManual.toString());
    } catch (_) {}
  }

  Future<void> loadData() async {
    final prefs = await SharedPreferences.getInstance();
    final savedMode = prefs.getString('mode') ?? "public";
    mode = (savedMode == "live") ? "public" : savedMode;
    isDarkMode = prefs.getBool('isDarkMode') ?? false;
    try {
      namaIbadah = (await SharedPreferences.getInstance()).getString('namaIbadah') ?? prefs.getString('namaIbadah') ?? "";
      alamat = (await SharedPreferences.getInstance()).getString('alamat') ?? prefs.getString('alamat') ?? "";
      // Load runningText dari chunks jika ada
      final chunkCount = int.tryParse((await SharedPreferences.getInstance()).getString('runningTextChunks') ?? '0') ?? 0;
      if (chunkCount > 0) {
        final buf = StringBuffer();
        for (var i = 0; i < chunkCount; i++) {
          buf.write((await SharedPreferences.getInstance()).getString('runningText_$i') ?? '');
        }
        runningText = buf.toString();
      } else {
        runningText = (await SharedPreferences.getInstance()).getString('runningText') ?? prefs.getString('runningText') ?? "";
      }
      fotoMasjid = (await SharedPreferences.getInstance()).getString('fotoMasjid') ?? "";
    } catch (_) {
      namaIbadah = prefs.getString('namaIbadah') ?? "";
      alamat = prefs.getString('alamat') ?? "";
      runningText = prefs.getString('runningText') ?? "";
      fotoMasjid = prefs.getString('fotoMasjid') ?? "";
    }
    houseUniqueCode = prefs.getString('houseUniqueCode') ?? "NP-0000";

    if (houseUniqueCode == "NP-0000") generateUniqueCode();
    adminPass    = prefs.getString('adminPass') ?? "123";
    isFirstLogin = prefs.getBool('isFirstLogin') ?? true;
    saldoAwal      = prefs.getDouble('saldoAwal') ?? 0.0;
    totalMasukSimpan  = prefs.getDouble('totalMasukSimpan') ?? 0.0;
    totalKeluarSimpan = prefs.getDouble('totalKeluarSimpan') ?? 0.0;
    aktifRamadhan     = prefs.getBool('aktifRamadhan') ?? false;
    menitSebelumSubuh = prefs.getInt('menitSebelumSubuh') ?? 10;
    // Load jadwal jumat dari SharedPreferences
    try {
      final jj = prefs.getStringList('jadwalJumat') ?? [];
      if (jj.isNotEmpty) {
        jadwalJumat = jj.map((s) {
          final parts = s.split('||');
          return <String, dynamic>{
            'tanggal': parts.length > 0 ? parts[0] : '',
            'imam': parts.length > 1 ? parts[1] : '',
            'khatib': parts.length > 2 ? parts[2] : '',
            'tema': parts.length > 3 ? parts[3] : '',
          };
        }).toList();
      }
    } catch (_) {}
    await loadPengurus();
    waktuSubuh   = (await SharedPreferences.getInstance()).getString('waktuSubuh')   ?? prefs.getString('waktuSubuh')   ?? "04:45";
      waktuDzuhur  = (await SharedPreferences.getInstance()).getString('waktuDzuhur')  ?? prefs.getString('waktuDzuhur')  ?? "12:15";
      waktuAshar   = (await SharedPreferences.getInstance()).getString('waktuAshar')   ?? prefs.getString('waktuAshar')   ?? "15:30";
      waktuMaghrib = (await SharedPreferences.getInstance()).getString('waktuMaghrib') ?? prefs.getString('waktuMaghrib') ?? "18:25";
      waktuIsya    = (await SharedPreferences.getInstance()).getString('waktuIsya')    ?? prefs.getString('waktuIsya')    ?? "19:35";
      jadwalManual = ((await SharedPreferences.getInstance()).getString('jadwalManual') ?? 'false') == 'true';
    koreksiMenit         = prefs.getInt('koreksiMenit')    ?? 2;
    menitIqomah          = prefs.getInt('menitIqomah')     ?? 10;
    bahasa               = prefs.getString('bahasa')        ?? 'id';
    kecepatanRunningText = prefs.getDouble('kecepatanRunningText') ?? 18.0;
    warnaRunningText     = prefs.getString('warnaRunningText')     ?? 'FFFFFF';
    smartDisplayAktif    = prefs.getBool('smartDisplayAktif')      ?? true;
    durasiAdzanMenit     = prefs.getInt('durasiAdzanMenit')        ?? 10;
    await loadTransaksi();
    await loadGaleriBase64();
    await loadDonatur();
    await loadAcara();
    await loadQris();
    await loadInventaris();
    await _loadPengumuman();
    _safeNotify();
  }
}

// ── Palet Warna NovaPro ──
const kPrimary      = Color(0xFF631414);  // Marun utama
const kPrimaryLight = Color(0xFF8B1A1A);  // Marun terang (gradient)
const kPrimaryDark  = Color(0xFF3E0A0A);  // Marun gelap (gradient)
const kBg           = Color(0xFFFFFFFF);  // Putih bersih
const kBgGreen      = Color(0xFFE8F5E9);  // Hijau samar (section bawah)
const kCard         = Colors.white;
const kCardGreen    = Color(0xFFF1F8E9);  // Card hijau samar
const kTextDark     = Color(0xFF1A1A1A);
const kTextGrey     = Color(0xFF757575);
const kDivider      = Color(0xFFE0E0E0);
const kIconGrey     = Color(0xFF616161);
const kAccent       = Color(0xFFFFD700);  // Emas
const kAccentLight  = Color(0xFFFFF176);  // Emas muda
const kGreen        = Color(0xFF2E7D32);  // Hijau islami
const kGreenLight   = Color(0xFF43A047);  // Hijau terang

// Gradient marun (atas ke bawah)
const kGradientPrimary = LinearGradient(
  begin: Alignment.topLeft,
  end: Alignment.bottomRight,
  colors: [Color(0xFF3E0A0A), Color(0xFF631414), Color(0xFF8B1A1A)],
);

// Gradient header (marun ke putih)
const kGradientHeader = LinearGradient(
  begin: Alignment.topCenter,
  end: Alignment.bottomCenter,
  colors: [Color(0xFF631414), Color(0xFF8B1A1A), Color(0xFFFFFFFF)],
  stops: [0.0, 0.6, 1.0],
);

// ── Safe Run Helper (anti bentrok frame) ──
void safeRun(VoidCallback fn) {
  Future.delayed(Duration.zero, fn);
}

// ── Notifikasi Adzan ──
class AdzanNotifikasi {
  static final _notif = FlutterLocalNotificationsPlugin();
  static bool _initialized = false;

  static Future<void> init() async {
    if (_initialized) return;
    const android = AndroidInitializationSettings('@mipmap/ic_launcher');
    const settings = InitializationSettings(android: android);
    await _notif.initialize(settings);
    _initialized = true;
  }

  static Future<void> tampilkan(String namaWaktu) async {
    await init();
    const detail = AndroidNotificationDetails(
      'adzan_channel', 'Waktu Adzan',
      channelDescription: 'Notifikasi waktu sholat',
      importance: Importance.high,
      priority: Priority.high,
      icon: '@mipmap/ic_launcher',
    );
    await _notif.show(
      0,
      '🕌 Waktu ',
      'Telah masuk waktu sholat ',
      const NotificationDetails(android: detail),
    );
  }
}

// ── Responsive Scale Helper ──
class AppScale {
  final double w;
  AppScale(this.w);

  // Breakpoints 4 layar
  bool get isPhone   => w < 600;           // HP < 600px
  bool get isTablet  => w >= 600 && w < 1024;  // Tablet 600-1024px
  bool get isLaptop  => w >= 1024 && w < 1600; // Laptop 1024-1600px
  bool get isTV      => w >= 1600;          // TV/Proyektor >= 1600px

  // Font scale: HP=1.0, Tablet=1.3, Laptop=1.6, TV=2.2
  double get fs => isPhone ? 1.0 : isTablet ? 1.3 : isLaptop ? 1.6 : 2.2;

  // Font sizes
  double get title    => (16 * fs).clamp(16.0, 40.0);
  double get subtitle => (13 * fs).clamp(13.0, 30.0);
  double get body     => (12 * fs).clamp(12.0, 26.0);
  double get small    => (10 * fs).clamp(10.0, 22.0);
  double get caption  => (9  * fs).clamp(9.0,  18.0);

  // Padding & spacing
  double get pad   => (12 * fs).clamp(12.0, 36.0);
  double get padSm => (8  * fs).clamp(8.0,  24.0);
  double get padXs => (4  * fs).clamp(4.0,  14.0);
  double get gap   => (8  * fs).clamp(8.0,  22.0);

  // Icon sizes
  double get iconSm => (18 * fs).clamp(18.0, 44.0);
  double get iconMd => (24 * fs).clamp(24.0, 56.0);
  double get iconLg => (32 * fs).clamp(32.0, 80.0);

  // Avatar radius
  double get avatarSm => (18 * fs).clamp(18.0, 44.0);
  double get avatarMd => (24 * fs).clamp(24.0, 60.0);

  // Kolom grid: HP=2, Tablet=3, Laptop=4, TV=5
  int get gridCols => isPhone ? 2 : isTablet ? 3 : isLaptop ? 4 : 5;

  // Layout helper
  bool get useSidePanel => !isPhone;
  double get contentMaxWidth => isPhone ? double.infinity : isTablet ? 800 : isLaptop ? 1100 : 1400;
  double get sideWidth => isTablet ? 280 : 340;
  int get pengurusCols => isPhone ? 3 : isTablet ? 4 : isLaptop ? 5 : 6;
}

// Helper extension untuk mudah akses dari context
extension AppScaleX on BuildContext {
  AppScale get sc => AppScale(MediaQuery.of(this).size.width);
}

class NovaProMasterRoot extends StatefulWidget {
  const NovaProMasterRoot({super.key});
  @override
  State<NovaProMasterRoot> createState() => _NovaProMasterRootState();
}

class _NovaProMasterRootState extends State<NovaProMasterRoot> {
  @override
  Widget build(BuildContext context) {
    final mode = context.select<AppProvider, String>((p) => p.mode);
    switch (mode) {
      case "admin": return const AdminShell();
      case "live":  return const PublicDisplay();
      default:      return const PublicHome();
    }
  }
}

// Widget password field dengan toggle visibility
class _PasswordField extends StatefulWidget {
  final TextEditingController controller;
  final String hint;
  const _PasswordField({required this.controller, required this.hint});
  @override
  State<_PasswordField> createState() => _PasswordFieldState();
}

class _PasswordFieldState extends State<_PasswordField> {
  bool _show = false;
  @override
  Widget build(BuildContext context) {
    return TextField(
      controller: widget.controller,
      obscureText: !_show,
      autofocus: true,
      decoration: InputDecoration(
        hintText: widget.hint,
        prefixIcon: const Icon(Icons.lock, color: kPrimary),
        filled: true, fillColor: kBg,
        border: OutlineInputBorder(borderRadius: BorderRadius.circular(8),
            borderSide: const BorderSide(color: kDivider)),
        enabledBorder: OutlineInputBorder(borderRadius: BorderRadius.circular(8),
            borderSide: const BorderSide(color: kDivider)),
        suffixIcon: IconButton(
          icon: Icon(_show ? Icons.visibility_off : Icons.visibility, color: Colors.grey, size: 20),
          onPressed: () => setState(() => _show = !_show),
        ),
      ),
    );
  }
}

class PublicHome extends StatefulWidget {
  const PublicHome({super.key});
  @override
  State<PublicHome> createState() => _PublicHomeState();
}

class _PublicHomeState extends State<PublicHome> {
  final _scrollCtrl = ScrollController();
  // Controller di dalam State - sesuai best practice Flutter
  final _passCtrl = TextEditingController();

  @override
  void initState() {
    super.initState();
  }

  @override
  void dispose() {
    _scrollCtrl.dispose();
    _passCtrl.dispose(); // Dispose di sini - aman
    super.dispose();
  }

  void _showLupaPassword(BuildContext ctx, AppProvider p) {
    // Generate 4 angka acak unik
    final random = Random();
    final angka = List.generate(9, (i) => i + 1)..shuffle(random);
    final soal = angka.take(4).toList();
    final jawaban = List<int>.from(soal)..sort();
    
    // State puzzle
    final pilihan = <int>[];
    final newPassCtrl = TextEditingController();
    bool puzzleSolved = false;

    showDialog(
      context: ctx,
      barrierDismissible: true,
      builder: (dCtx) => StatefulBuilder(
        builder: (dCtx2, setStateDialog) => AlertDialog(
          shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
          title: Row(children: [
            const Icon(Icons.lock_reset, color: Colors.orange),
            const SizedBox(width: 8),
            Expanded(child: Text(
              p._t('Lupa Password','Forgot Password','نسيت كلمة المرور'),
              style: const TextStyle(fontSize: 15),
            )),
          ]),
          content: SingleChildScrollView(child: Column(mainAxisSize: MainAxisSize.min, children: [
            if (!puzzleSolved) ...[
              // Instruksi
              Container(
                padding: const EdgeInsets.all(10),
                decoration: BoxDecoration(
                  color: Colors.blue.shade50,
                  borderRadius: BorderRadius.circular(8),
                ),
                child: Text(
                  p._t(
                    'Susun angka berikut dari TERKECIL ke TERBESAR:',
                    'Arrange the numbers from SMALLEST to LARGEST:',
                    'رتّب الأرقام من الأصغر إلى الأكبر:',
                  ),
                  style: TextStyle(fontSize: 12, color: Colors.blue.shade800),
                  textAlign: TextAlign.center,
                ),
              ),
              const SizedBox(height: 12),
              // Tampil angka soal
              Row(mainAxisAlignment: MainAxisAlignment.center, children: soal.map((n) =>
                Container(
                  margin: const EdgeInsets.symmetric(horizontal: 4),
                  width: 38, height: 38,
                  decoration: BoxDecoration(
                    color: kPrimary,
                    borderRadius: BorderRadius.circular(8),
                  ),
                  child: Center(child: Text('$n',
                    style: const TextStyle(color: Colors.white, fontSize: 17, fontWeight: FontWeight.bold))),
                )
              ).toList()),
              const SizedBox(height: 10),
              // Area jawaban user
              Text(p._t('Jawaban kamu:','Your answer:','إجابتك:'),
                style: const TextStyle(fontSize: 12, color: Colors.grey)),
              const SizedBox(height: 6),
              Row(mainAxisAlignment: MainAxisAlignment.center, children: [
                ...List.generate(4, (i) => Container(
                  margin: const EdgeInsets.symmetric(horizontal: 4),
                  width: 38, height: 38,
                  decoration: BoxDecoration(
                    color: i < pilihan.length ? Colors.green.shade100 : Colors.grey.shade100,
                    borderRadius: BorderRadius.circular(8),
                    border: Border.all(
                      color: i < pilihan.length ? Colors.green : Colors.grey.shade300,
                      width: 2,
                    ),
                  ),
                  child: Center(child: Text(
                    i < pilihan.length ? '${pilihan[i]}' : '',
                    style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold,
                      color: Colors.green.shade700),
                  )),
                )),
              ]),
              const SizedBox(height: 12),
              // Tombol angka untuk dipilih
              Wrap(spacing: 8, runSpacing: 8, alignment: WrapAlignment.center,
                children: soal.map((n) {
                  final sudahDipilih = pilihan.contains(n);
                  return GestureDetector(
                    onTap: sudahDipilih ? null : () {
                      setStateDialog(() {
                        pilihan.add(n);
                        // Cek jawaban kalau sudah 4 pilihan
                        if (pilihan.length == 4) {
                          if (pilihan.toString() == jawaban.toString()) {
                            puzzleSolved = true;
                          } else {
                            // Salah - reset
                            Future.delayed(const Duration(milliseconds: 600), () {
                              setStateDialog(() => pilihan.clear());
                            });
                          }
                        }
                      });
                    },
                    child: Container(
                      width: 44, height: 44,
                      decoration: BoxDecoration(
                        color: sudahDipilih ? Colors.grey.shade300 : kPrimary.withValues(alpha: 0.85),
                        borderRadius: BorderRadius.circular(8),
                      ),
                      child: Center(child: Text('$n',
                        style: TextStyle(
                          color: sudahDipilih ? Colors.grey : Colors.white,
                          fontSize: 18, fontWeight: FontWeight.bold))),
                    ),
                  );
                }).toList(),
              ),
              const SizedBox(height: 8),
              // Tombol reset pilihan
              if (pilihan.isNotEmpty && !puzzleSolved)
                TextButton.icon(
                  onPressed: () => setStateDialog(() => pilihan.clear()),
                  icon: const Icon(Icons.refresh, size: 14),
                  label: Text(p._t('Ulangi','Reset','إعادة'),
                    style: const TextStyle(fontSize: 12)),
                ),
            ] else ...[
              // Puzzle benar - isi password baru
              Icon(Icons.check_circle, color: Colors.green, size: 48),
              const SizedBox(height: 8),
              Text(p._t('Benar! Masukkan password baru:','Correct! Enter new password:','صحيح! أدخل كلمة مرور جديدة:'),
                style: const TextStyle(fontSize: 13), textAlign: TextAlign.center),
              const SizedBox(height: 12),
TextField(
                controller: newPassCtrl,
                obscureText: true,
                autofocus: true,
                decoration: InputDecoration(
                  labelText: p._t('Password Baru','New Password','كلمة المرور الجديدة'),
                  prefixIcon: const Icon(Icons.lock, color: kPrimary),
                  border: OutlineInputBorder(borderRadius: BorderRadius.circular(8)),
                ),
              ),
            ],
          ])),
          actions: [
            TextButton(
              onPressed: () {
                newPassCtrl.dispose();
                Navigator.pop(dCtx2);
              },
              child: Text(p.lBatal),
            ),
            if (puzzleSolved)
              ElevatedButton(
                style: ElevatedButton.styleFrom(
                  backgroundColor: kPrimary, foregroundColor: Colors.white),
                onPressed: () {
                  final np = newPassCtrl.text.trim();
                  if (np.length < 4) {
                    ScaffoldMessenger.of(ctx).showSnackBar(SnackBar(
                      content: Text(p._t('Minimal 4 karakter!','Min 4 characters!','4 أحرف على الأقل!')),
                      backgroundColor: Colors.red));
                    return;
                  }
                  p.updatePassword(np);
                  newPassCtrl.dispose();
                  Navigator.pop(dCtx2);
                  ScaffoldMessenger.of(ctx).showSnackBar(SnackBar(
                    content: Text(p._t('Password berhasil direset!','Password reset successful!','تم إعادة تعيين كلمة المرور!')),
                    backgroundColor: Colors.green));
                },
                child: Text(p._t('SIMPAN','SAVE','حفظ')),
              ),
          ],
        ),
      ),
    );
  }

  void _showWajibGantiPassword(BuildContext ctx, AppProvider p) {
    final newPassCtrl = TextEditingController();
    final konfirmCtrl = TextEditingController();
    showDialog(
      context: ctx,
      barrierDismissible: false,
      builder: (dCtx) {
        bool _showPass1 = false, _showPass2 = false;
        return StatefulBuilder(
        builder: (dCtx2, setStateDialog) => AlertDialog(
          shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
          title: Row(children: [
          const Icon(Icons.lock_reset, color: Colors.orange),
          const SizedBox(width: 8),
          Expanded(child: Text(
            p._t('Ganti Password','Change Password','تغيير كلمة المرور'),
            style: const TextStyle(fontSize: 16),
          )),
        ]),
        content: SingleChildScrollView(child: Column(mainAxisSize: MainAxisSize.min, children: [
          TextField(
            controller: newPassCtrl,
            obscureText: !_showPass1,
            decoration: InputDecoration(
              labelText: p._t('Password Baru','New Password','كلمة المرور الجديدة'),
              prefixIcon: const Icon(Icons.lock, color: kPrimary),
              border: OutlineInputBorder(borderRadius: BorderRadius.circular(8)),
              suffixIcon: IconButton(
                icon: Icon(_showPass1 ? Icons.visibility_off : Icons.visibility, color: Colors.grey, size: 20),
                onPressed: () => setStateDialog(() => _showPass1 = !_showPass1),
              ),
            ),
          ),
          const SizedBox(height: 8),
          TextField(
            controller: konfirmCtrl,
            obscureText: !_showPass2,
            decoration: InputDecoration(
              labelText: p._t('Konfirmasi','Confirm','تأكيد'),
              prefixIcon: const Icon(Icons.lock_outline, color: kPrimary),
              border: OutlineInputBorder(borderRadius: BorderRadius.circular(8)),
              suffixIcon: IconButton(
                icon: Icon(_showPass2 ? Icons.visibility_off : Icons.visibility, color: Colors.grey, size: 20),
                onPressed: () => setStateDialog(() => _showPass2 = !_showPass2),
              ),
            ),
          ),
        ])),
        actions: [
          ElevatedButton(
            style: ElevatedButton.styleFrom(
              backgroundColor: kPrimary, foregroundColor: Colors.white,
              minimumSize: const Size(double.infinity, 44),
            ),
            onPressed: () {
              final np = newPassCtrl.text.trim();
              final kp = konfirmCtrl.text.trim();
              if (np.length < 4) {
                if (!ctx.mounted) return;
                ScaffoldMessenger.of(ctx).showSnackBar(SnackBar(
                  content: Text(p._t('Password minimal 4 karakter!','Password min 4 characters!','كلمة المرور 4 أحرف على الأقل!')),
                  backgroundColor: Colors.red));
                return;
              }
              if (np != kp) {
                ScaffoldMessenger.of(ctx).showSnackBar(SnackBar(
                  content: Text(p._t('Password tidak cocok!','Passwords do not match!','كلمتا المرور غير متطابقتان!')),
                  backgroundColor: Colors.red));
                return;
              }
              p.updatePassword(np);
              p.setFirstLoginDone();
              Navigator.pop(dCtx);
              if (!ctx.mounted) return;
              // Dispose setelah navigator pop selesai
              WidgetsBinding.instance.addPostFrameCallback((_) {
                newPassCtrl.dispose();
                konfirmCtrl.dispose();
              });
              ScaffoldMessenger.of(ctx).showSnackBar(SnackBar(
                content: Text(p._t('Password berhasil diganti!','Password changed successfully!','تم تغيير كلمة المرور بنجاح!')),
                backgroundColor: Colors.green));
              Navigator.of(ctx).pushAndRemoveUntil(
                MaterialPageRoute(builder: (_) => const AdminShell()),
                (route) => false,
              );
            },
            child: Text(p._t('SIMPAN & LANJUT','SAVE & CONTINUE','حفظ ومتابعة')),
          ),
        ],
      ),
        );  // StatefulBuilder
      },
    );
  }

  void _showLogin(BuildContext outerCtx) {
    _passCtrl.clear(); // Reset password setiap buka dialog
    final ap = outerCtx.read<AppProvider>();
    
    showDialog(
      context: outerCtx,
      barrierDismissible: true,
      barrierColor: Colors.black87,
      builder: (dialogCtx) => AlertDialog(
        backgroundColor: Colors.white,
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
        title: Row(children: [
          const Icon(Icons.mosque, color: kPrimary),
          const SizedBox(width: 8),
          Expanded(child: Text(
            ap._t("Login Admin","Admin Login","تسجيل دخول المشرف"),
            overflow: TextOverflow.ellipsis,
            maxLines: 2,
          )),
        ]),
        content: _PasswordField(
          controller: _passCtrl,
          hint: ap._t("Password","Password","كلمة المرور"),
        ),
        actionsAlignment: MainAxisAlignment.spaceBetween,
        actions: [
          TextButton(
            onPressed: () {
              Navigator.pop(dialogCtx);
              Future.delayed(const Duration(milliseconds: 100), () {
                if (!outerCtx.mounted) return;
                _showLupaPassword(outerCtx, outerCtx.read<AppProvider>());
              });
            },
            child: Text(ap._t('Lupa Password?','Forgot Password?','نسيت كلمة المرور؟'),
              style: const TextStyle(color: Colors.grey, fontSize: 12)),
          ),
          Row(mainAxisSize: MainAxisSize.min, children: [
          TextButton(
            onPressed: () => Navigator.pop(dialogCtx),
            child: Text(ap.lBatal),
          ),
          ElevatedButton(
            style: ElevatedButton.styleFrom(backgroundColor: kPrimary, foregroundColor: Colors.white),
            onPressed: () {
              final pass = _passCtrl.text;
              Navigator.pop(dialogCtx);
              Future.delayed(const Duration(milliseconds: 300), () {
                if (!outerCtx.mounted) return;
                final apInner = outerCtx.read<AppProvider>();
                final loginOk = apInner.login(pass);
                if (loginOk) {
                  Future.delayed(const Duration(milliseconds: 100), () {
                    if (!outerCtx.mounted) return;
                    if (apInner.isFirstLogin) {
                      _showWajibGantiPassword(outerCtx, apInner);
                    } else {
                      Navigator.of(outerCtx).pushAndRemoveUntil(
                        MaterialPageRoute(builder: (_) => const AdminShell()),
                        (route) => false,
                      );
                    }
                  });
                } else {
                  if (!outerCtx.mounted) return;
                  ScaffoldMessenger.of(outerCtx).showSnackBar(
                    SnackBar(
                      content: Text(apInner.lPasswordSalah),
                      backgroundColor: Colors.red,
                    ),
                  );
                }
              });
            },
            child: Text(ap.lLoginBtn),
          ),
          ]),  // Row
        ],
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    final p = context.watch<AppProvider>();
    final sc = context.sc;
    return Scaffold(
      resizeToAvoidBottomInset: true,
      backgroundColor: const Color(0xFFFFFFFF),
      appBar: AppBar(
        flexibleSpace: Container(decoration: const BoxDecoration(gradient: kGradientPrimary)),
        backgroundColor: Colors.transparent,
        elevation: 2,
        shadowColor: Colors.black38,
        toolbarHeight: sc.isTV ? 80 : sc.isLaptop ? 68 : sc.isTablet ? 60 : kToolbarHeight,
        title: Column(crossAxisAlignment: CrossAxisAlignment.start, mainAxisSize: MainAxisSize.min, children: [
            Text("NOVAPRO", style: TextStyle(color: kAccent, fontSize: sc.title, fontWeight: FontWeight.w900, letterSpacing: 1)),
            Text(p._t('MANAJEMEN MASJID DAN MUSHOLLAH','MOSQUE & PRAYER HALL MANAGEMENT','إدارة المساجد والمصليات'), style: TextStyle(color: Colors.white70, fontSize: sc.caption, fontWeight: FontWeight.w600, letterSpacing: 0.5)),
            Text('Powered by Novarizal', style: TextStyle(
              color: const Color(0xFFFFD700),
              fontSize: sc.caption * 0.8,
              fontStyle: FontStyle.italic,
              fontWeight: FontWeight.w600,
            )),
          ]),
        actions: [
          _FbIconBtn(icon: Icons.settings, onTap: () => _showLogin(context), tooltip: "Admin"),
        ],
      ),
      body: ZoomWrapper(child: ListView(
          padding: EdgeInsets.fromLTRB(sc.pad, sc.padSm, sc.pad, sc.pad),
          children: [

                    // Profil Masjid - Banner gradient
            Container(
              decoration: BoxDecoration(
                gradient: const LinearGradient(
                  begin: Alignment.topLeft, end: Alignment.bottomRight,
                  colors: [Color(0xFF3E0A0A), Color(0xFF631414), Color(0xFF8B1A1A)],
                ),
                borderRadius: BorderRadius.circular(20),
                boxShadow: [
                  BoxShadow(color: kPrimary.withValues(alpha: 0.3), blurRadius: 16, offset: const Offset(0, 6)),
                ],
              ),
              child: Padding(
                padding: EdgeInsets.all(sc.pad),
                child: Row(children: [
                  // Foto masjid dengan border emas
                  Container(
                    decoration: BoxDecoration(
                      shape: BoxShape.circle,
                      border: Border.all(color: kAccent, width: 2.5),
                      boxShadow: [BoxShadow(color: Colors.black26, blurRadius: 8)],
                    ),
                    child: CircleAvatar(
                      radius: sc.avatarMd, backgroundColor: kPrimaryDark,
                      backgroundImage: p.fotoMasjid.isNotEmpty ? MemoryImage(base64Decode(p.fotoMasjid)) : null,
                      child: p.fotoMasjid.isEmpty ? Icon(Icons.mosque, color: kAccent, size: sc.avatarMd) : null,
                    ),
                  ),
                  SizedBox(width: sc.padSm),
                  Expanded(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                    Text(p.namaIbadah, overflow: TextOverflow.ellipsis,
                        style: TextStyle(fontSize: sc.subtitle, fontWeight: FontWeight.bold, color: Colors.white)),
                    SizedBox(height: 2),
                    Text(p.alamat, overflow: TextOverflow.ellipsis,
                        style: TextStyle(color: Colors.white70, fontSize: sc.body)),
                    SizedBox(height: 4),
                    Container(
                      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 3),
                      decoration: BoxDecoration(
                        color: kAccent.withValues(alpha: 0.2),
                        borderRadius: BorderRadius.circular(20),
                        border: Border.all(color: kAccent.withValues(alpha: 0.5)),
                      ),
                      child: Text("${p._t('ID','ID','الرمز')}: ${p.houseUniqueCode}",
                        style: TextStyle(color: kAccent, fontSize: sc.small, fontWeight: FontWeight.w600)),
                    ),
                  ])),
                ]),
              ),
            ),
            SizedBox(height: sc.gap),

            // Jadwal Sholat
            Container(
              decoration: BoxDecoration(
                color: kCard,
                borderRadius: BorderRadius.circular(20),
                boxShadow: [
                  BoxShadow(color: Colors.black.withValues(alpha: 0.07), blurRadius: 12, offset: const Offset(0, 4)),
                ],
              ),
              child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                // Header jadwal sholat
                Container(
                  padding: EdgeInsets.symmetric(horizontal: sc.pad, vertical: sc.padSm),
                  decoration: BoxDecoration(
                    gradient: LinearGradient(
                      colors: [kBgGreen, Colors.white],
                      begin: Alignment.centerLeft, end: Alignment.centerRight,
                    ),
                    borderRadius: const BorderRadius.only(
                      topLeft: Radius.circular(20), topRight: Radius.circular(20)),
                  ),
                  child: Row(children: [
                    Container(
                      padding: const EdgeInsets.all(6),
                      decoration: BoxDecoration(
                        color: kPrimary, borderRadius: BorderRadius.circular(8)),
                      child: Icon(Icons.access_time, color: kAccent, size: sc.iconSm),
                    ),
                    SizedBox(width: sc.padSm),
                    Flexible(child: Text(p.lJadwalSholat, overflow: TextOverflow.ellipsis,
                      style: TextStyle(color: kPrimary, fontWeight: FontWeight.bold, fontSize: sc.subtitle))),
                  ]),
                ),
                Padding(padding: EdgeInsets.all(sc.padSm), child: Column(children: [
                LayoutBuilder(builder: (ctx, bc) {
                  // Kalau layar sempit pakai Wrap agar tidak overflow
                  final boxes = [
                    _SholatBox(p.labelSubuh,   p.waktuSubuh),
                    _SholatBox(p.labelDzuhur,  p.waktuDzuhur),
                    _SholatBox(p.labelAshar,   p.waktuAshar),
                    _SholatBox(p.labelMaghrib, p.waktuMaghrib),
                    _SholatBox(p.labelIsya,    p.waktuIsya),
                  ];
                  if (bc.maxWidth < 320) {
                    // Layar sangat sempit - Wrap
                    return Wrap(
                      spacing: 4,
                      runSpacing: 4,
                      children: boxes.map((b) => SizedBox(
                        width: (bc.maxWidth - 20) / 3,
                        child: b,
                      )).toList(),
                    );
                  }
                  // Layar normal - Row dengan Expanded
                  return Row(children: boxes.map((b) => Expanded(child: b)).toList());
                }),
              ])),
            ]),
            ),
            SizedBox(height: sc.gap),

            // Pengumuman
            if (p.pengumuman.isEmpty)
              Container(
                padding: const EdgeInsets.all(24),
                decoration: BoxDecoration(color: kCard, borderRadius: BorderRadius.circular(12)),
                child: Center(child: Column(mainAxisSize: MainAxisSize.min, children: [
                  const Icon(Icons.campaign_outlined, size: 48, color: kDivider),
                  const SizedBox(height: 8),
                  Text(p.lBelumAdaPengumuman, style: const TextStyle(color: kTextGrey)),
                ])),
              )
            else
              ...p.pengumuman.map((um) => Container(
                margin: const EdgeInsets.only(bottom: 8),
                padding: const EdgeInsets.all(14),
                decoration: BoxDecoration(color: kCard, borderRadius: BorderRadius.circular(12),
                  boxShadow: [BoxShadow(color: kDivider, blurRadius: 4)]),
                child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                  Row(children: [
                    CircleAvatar(radius: 16, backgroundColor: kPrimary,
                      backgroundImage: p.fotoMasjid.isNotEmpty ? MemoryImage(base64Decode(p.fotoMasjid)) : null,
                      child: p.fotoMasjid.isEmpty ? Icon(Icons.campaign, color: Colors.white, size: 16) : null,
                    ),
                    const SizedBox(width: 8),
                    Expanded(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                      Text(p.namaIbadah, overflow: TextOverflow.ellipsis,
                          style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 13)),
                      Text(p.lPengumuman, style: TextStyle(color: kTextGrey, fontSize: 11)),
                    ])),
                  ]),
                  const SizedBox(height: 8),
                  if ((um['foto'] ?? '').isNotEmpty) ...[
                    ClipRRect(
                      borderRadius: BorderRadius.circular(8),
                      child: Image.memory(base64Decode(um['foto']!),
                        width: double.infinity, height: 180, fit: BoxFit.cover),
                    ),
                    const SizedBox(height: 8),
                  ],
                  Text(um['teks'] ?? '', style: const TextStyle(fontSize: 14)),
                  const SizedBox(height: 4),
                  Text(um['waktu'] != null && um['waktu']!.isNotEmpty
                    ? () {
                        final locale = p.bahasa == 'ar' ? 'ar' : p.bahasa == 'en' ? 'en' : 'id';
                        try { return DateFormat('dd MMM yyyy, HH:mm', locale).format(DateTime.tryParse(um['waktu']!) ?? DateTime.now()); }
                        catch (_) { return DateFormat('dd MMM yyyy, HH:mm').format(DateTime.tryParse(um['waktu']!) ?? DateTime.now()); }
                      }()
                    : '',
                    style: const TextStyle(color: kTextGrey, fontSize: 10)),
                ]),
              )).toList(),

          ],  // end children ListView
        ),
    )
    );  // end Scaffold
  }
}

class AdminShell extends StatefulWidget {
  const AdminShell({super.key});
  @override
  State<AdminShell> createState() => _AdminShellState();
}

class _AdminShellState extends State<AdminShell> {
  int _idx = 0;

  static const _icons  = [Icons.home, Icons.people, Icons.photo_library,
                           Icons.account_balance_wallet, Icons.volunteer_activism];

  Widget _buildPage(int idx) {
    switch (idx) {
      case 0: return _AdminHome();
      case 1: return PengurusPage(onBack: () => setState(() => _idx = 0));
      case 2: return GaleriPage(onBack: () => setState(() => _idx = 0));
      case 3: return ArusKasTab(onBack: () => setState(() => _idx = 0));
      case 4: return DonaturTab(onBack: () => setState(() => _idx = 0));
      case 5: return InventarisTab(onBack: () => setState(() => _idx = 0));
      case 6: return AcaraPage(onBack: () => setState(() => _idx = 0));
      case 7: return StatistikPage(onBack: () => setState(() => _idx = 0));
      case 8: return _LainnyaPage();
      default: return _AdminHome();
    }
  }

  @override
  Widget build(BuildContext context) {
    final sc = context.sc;
    // FIX UTAMA: Hapus Consumer dari sini.
    // Consumer + IndexedStack = 9 halaman aktif sekaligus, semua context.watch
    // terdaftar sebagai dependents. Saat logout/login -> notifyListeners() ->
    // widget tree dihancurkan tapi dependents belum bersih -> CRASH.
    return Scaffold(
      resizeToAvoidBottomInset: true,
      backgroundColor: kBg,
      appBar: AppBar(
        backgroundColor: kCard,
        elevation: 1,
        shadowColor: kDivider,
        toolbarHeight: sc.isTV ? 80 : sc.isLaptop ? 68 : sc.isTablet ? 60 : kToolbarHeight,
        title: Selector<AppProvider, String>(
          selector: (_, ap) => ap.bahasa,
          builder: (_, bahasa, __) {
            final ap = context.read<AppProvider>();
            return Column(crossAxisAlignment: CrossAxisAlignment.start, mainAxisSize: MainAxisSize.min, children: [
              Text("NOVAPRO", style: TextStyle(color: kPrimary, fontSize: sc.title, fontWeight: FontWeight.w900, letterSpacing: 1)),
              Text(ap._t('MANAJEMEN MASJID DAN MUSHOLLAH','MOSQUE MANAGEMENT','إدارة المساجد'), style: TextStyle(color: kPrimary.withValues(alpha: 0.7), fontSize: sc.caption, fontWeight: FontWeight.w600, letterSpacing: 0.5, overflow: TextOverflow.ellipsis)),
              Text('Powered by Novarizal', style: TextStyle(
              color: const Color(0xFFFFD700),
              fontSize: sc.caption * 0.8,
              fontStyle: FontStyle.italic,
              fontWeight: FontWeight.w600,
            )),
            ]);
          },
        ),
        actions: [
          _FbIconBtn(
            icon: Icons.live_tv, color: Colors.red,
            onTap: () {
              context.read<AppProvider>().startLive();
              if (!context.mounted) return;
              Navigator.of(context).pushAndRemoveUntil(
                MaterialPageRoute(builder: (_) => const PublicDisplay()),
                (route) => false,
              );
            }, tooltip: 'Live Mode',
          ),
          Selector<AppProvider, bool>(
            selector: (_, p) => p.isDarkMode,
            builder: (ctx, isDark, __) => _FbIconBtn(
              icon: isDark ? Icons.light_mode : Icons.dark_mode,
              onTap: () => ctx.read<AppProvider>().toggleDarkMode(), tooltip: "Tema",
            ),
          ),
          _FbIconBtn(
            icon: Icons.logout,
            onTap: () {
              context.read<AppProvider>().logout();
              if (!context.mounted) return;
              Navigator.of(context).pushAndRemoveUntil(
                MaterialPageRoute(builder: (_) => const PublicHome()),
                (route) => false,
              );
            },
            tooltip: 'Keluar',
          ),
        ],
      ),
      // FIX: Ganti IndexedStack dengan lazy builder.
      // IndexedStack build SEMUA halaman sekaligus walau tidak terlihat.
      // Lazy builder hanya build halaman yang aktif -> tidak ada leak.
      body: ZoomWrapper(child: _buildPage(_idx)),
      bottomNavigationBar: Container(
        decoration: const BoxDecoration(
          color: kCard,
          border: Border(top: BorderSide(color: kDivider, width: 0.5)),
        ),
        child: BottomNavigationBar(
          currentIndex: _idx,
          onTap: (i) => setState(() => _idx = i),
          type: BottomNavigationBarType.fixed,
          backgroundColor: Colors.white,
          elevation: 8,
          selectedItemColor: kPrimary,
          unselectedItemColor: kIconGrey,
          showSelectedLabels: true,
          showUnselectedLabels: true,
          selectedFontSize: 11,
          unselectedFontSize: 11,
          iconSize: 24,
          items: List.generate(_icons.length, (i) {
              final p = context.read<AppProvider>();
              final labels = [
                p.lBeranda,
                p.lPengurus,
                p.lGaleri,
                p.lKas,
                p.lDonasi,
              ];
              return BottomNavigationBarItem(icon: Icon(_icons[i]), label: labels[i]);
            }),
        ),
      ),
    );
  }
}

class _AdminHome extends StatefulWidget {
  const _AdminHome();
  @override
  State<_AdminHome> createState() => _AdminHomeState();
}

class _AdminHomeState extends State<_AdminHome> {
  // Controllers di State - aman dari lifecycle crash
  final _pengumumanCtrl = TextEditingController();
  final _subuhCtrl   = TextEditingController();
  final _dzuhurCtrl  = TextEditingController();
  final _asharCtrl   = TextEditingController();
  final _maghribCtrl = TextEditingController();
  final _isyaCtrl    = TextEditingController();
  final _passLamaCtrl = TextEditingController();
  final _passBaruCtrl = TextEditingController();
  final _passConfCtrl = TextEditingController();

  @override
  void dispose() {
    _pengumumanCtrl.dispose();
    _subuhCtrl.dispose();
    _dzuhurCtrl.dispose();
    _asharCtrl.dispose();
    _maghribCtrl.dispose();
    _isyaCtrl.dispose();
    _passLamaCtrl.dispose();
    _passBaruCtrl.dispose();
    _passConfCtrl.dispose();
    super.dispose();
  }

  void _tambahPengumuman(BuildContext ctx, AppProvider p) {
    _pengumumanCtrl.clear();
    // Simpan referensi controller agar bisa diakses di StatefulBuilder
    final pengCtrl = _pengumumanCtrl;
    String fotoBase64 = '';
    showDialog(
      context: ctx,
      builder: (_) => StatefulBuilder(
        builder: (ctx2, setS) => AlertDialog(
          shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
          title: Text(p.lTulisPengumuman, style: const TextStyle(fontWeight: FontWeight.bold)),
          content: SizedBox(
            width: double.maxFinite,
            child: SingleChildScrollView(child: Column(mainAxisSize: MainAxisSize.min, children: [
              // Kotak foto opsional
              GestureDetector(
                onTap: () async {
                  final picked = await ImagePicker().pickImage(source: ImageSource.gallery, imageQuality: 70);
                  if (picked == null) return;
                  final bytes = await picked.readAsBytes();
                  setS(() { fotoBase64 = base64Encode(bytes); });
                },
                child: Container(
                  width: double.infinity,
                  height: fotoBase64.isNotEmpty ? 160 : 80,
                  decoration: BoxDecoration(
                    color: kPrimary.withValues(alpha: 0.05),
                    borderRadius: BorderRadius.circular(10),
                    border: Border.all(color: kPrimary.withValues(alpha: 0.3), width: 1.5),
                  ),
                  child: fotoBase64.isNotEmpty
                    ? Stack(children: [
                        ClipRRect(
                          borderRadius: BorderRadius.circular(9),
                          child: Image.memory(base64Decode(fotoBase64),
                            width: double.infinity, height: 160, fit: BoxFit.cover)),
                        Positioned(top: 6, right: 6,
                          child: GestureDetector(
                            onTap: () => setS(() => fotoBase64 = ''),
                            child: Container(
                              padding: const EdgeInsets.all(4),
                              decoration: const BoxDecoration(color: Colors.red, shape: BoxShape.circle),
                              child: const Icon(Icons.close, color: Colors.white, size: 14)),
                          )),
                      ])
                    : Column(mainAxisAlignment: MainAxisAlignment.center, children: [
                        Icon(Icons.add_photo_alternate, color: kPrimary.withValues(alpha: 0.5), size: 28),
                        const SizedBox(height: 4),
                        Text(p.lTambahFotoOpsional,
                          style: TextStyle(color: kPrimary.withValues(alpha: 0.5), fontSize: 11)),
                      ]),
                ),
              ),
              const SizedBox(height: 12),
              TextField(
                controller: pengCtrl,
                maxLines: 4,
                autofocus: true,
                maxLength: 500,
                decoration: InputDecoration(
                  hintText: p._t("Tulis pengumuman di sini...","Write announcement here...","اكتب الإعلان هنا..."),
                  filled: true, fillColor: kBg,
                  border: OutlineInputBorder(borderRadius: BorderRadius.circular(8)),
                ),
              ),
            ])),
          ),
          actions: [
            TextButton(onPressed: () { Navigator.pop(ctx2); },
              child: Text(p.lBatal)),
            ElevatedButton(
              style: ElevatedButton.styleFrom(backgroundColor: kPrimary, foregroundColor: Colors.white),
              onPressed: () {
                final teks = pengCtrl.text.trim();
                if (teks.isEmpty) return;
                p.tambahPengumuman(teks, foto: fotoBase64);
                Navigator.pop(ctx2);
                ScaffoldMessenger.of(ctx).showSnackBar(
                  SnackBar(content: Text(p.lPengumumanDitambah)));
              },
              child: Text(p.lPosting),
            ),
          ],
        ),
      ),
    );
  }

  void _editProfil(BuildContext ctx, AppProvider p) {
    final n = TextEditingController(text: p.namaIbadah);
    final a = TextEditingController(text: p.alamat);
    final r = TextEditingController(text: p.runningText);
    showDialog(
      context: ctx,
      barrierDismissible: false,
      builder: (_) => StatefulBuilder(
        builder: (ctx, setState) => AlertDialog(
        title: Text(p.lEditProfil),
        insetPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 24),
        content: SizedBox(
          width: double.maxFinite,
          child: SingleChildScrollView(child: Column(mainAxisSize: MainAxisSize.min, children: [
          _field(n, p.lNamaMasjid, Icons.mosque),
          const SizedBox(height: 10),
          _field(a, p.lAlamat, Icons.location_on),
          const SizedBox(height: 4),
          Builder(builder: (ctx2) {
              final kota = SholatCalculator.cariKota(a.text);
              return kota != null
                ? Row(children: [
                    const Icon(Icons.check_circle, color: Colors.green, size: 14),
                    const SizedBox(width: 4),
                    Flexible(child: Text("${p._t('Terdeteksi','Detected','تم الكشف')}: ${kota.nama}",
                      overflow: TextOverflow.ellipsis,
                      style: const TextStyle(color: Colors.green, fontSize: 12))),
                  ])
                : Row(children: [
                    const Icon(Icons.info_outline, color: Colors.orange, size: 14),
                    const SizedBox(width: 4),
                    Expanded(child: Text(p._t('Ketik nama kota untuk deteksi waktu sholat','Type city name to detect prayer times','اكتب اسم المدينة لاكتشاف أوقات الصلاة'),
                      overflow: TextOverflow.ellipsis,
                      style: const TextStyle(color: Colors.orange, fontSize: 12))),
                  ]);
            }
          ),
          const SizedBox(height: 10),
          TextField(
            controller: r,
            maxLines: 6, minLines: 3, maxLength: null,
            keyboardType: TextInputType.multiline,
            decoration: InputDecoration(
              labelText: p._t('Running Text (opsional)','Running Text (optional)','نص متحرك (اختياري)'),
              hintText: p._t('Contoh: RAMAIKAN MASJID • JADWAL SHOLAT TEPAT WAKTU','e.g.: FILL THE MOSQUE • PRAYER ON TIME','مثال: أحيوا المسجد • الصلاة في وقتها'),
              alignLabelWithHint: true,
              border: OutlineInputBorder(borderRadius: BorderRadius.circular(8)),
              helperText: p._t('Ketik sepanjang apapun, tidak ada batas!','Type as long as you want, no limit!','اكتب بقدر ما تشاء، لا حد!'),
              helperStyle: const TextStyle(color: Colors.green, fontSize: 11),
            ),
          ),
        ])),
        ),
        actions: [
          TextButton(onPressed: () => Navigator.pop(ctx), child: Text(p.lBatal)),
          ElevatedButton(
            style: ElevatedButton.styleFrom(backgroundColor: Colors.blue, foregroundColor: Colors.white),
            onPressed: () async {
              final picked = await ImagePicker().pickImage(source: ImageSource.gallery, imageQuality: 70);
                if (picked == null) return;
                final bytes = await picked.readAsBytes();
                await p.updateFotoMasjid(base64Encode(bytes));
              // ignore: use_build_context_synchronously
              ScaffoldMessenger.of(ctx).showSnackBar(
                  SnackBar(content: Text(p._t('Foto berhasil diubah!','Photo updated!','تم تحديث الصورة!'))));
            },
            child: Text(p._t('GANTI FOTO','CHANGE PHOTO','تغيير الصورة')),
          ),
          ElevatedButton(
            style: ElevatedButton.styleFrom(backgroundColor: kPrimary, foregroundColor: Colors.white),
            onPressed: () async {
              final nama = n.text.trim();
              final almt = a.text.trim();
              final run  = r.text.trim();
              if (nama.isEmpty) return;
              await p.saveProfilAsync(nama, almt, run);
              // ignore: use_build_context_synchronously
              Navigator.pop(ctx);
              // ignore: use_build_context_synchronously
              ScaffoldMessenger.of(ctx).showSnackBar(
                  SnackBar(content: Text(p._t('Profil disimpan!','Profile saved!','تم حفظ الملف!'))));
            },
            child: Text(p.lSimpan),
          ),
        ],
      ),
    ),      // StatefulBuilder
    );
  }

  void _editJadwal(BuildContext ctx, AppProvider p) {
    _subuhCtrl.text   = p.waktuSubuh;
    _dzuhurCtrl.text  = p.waktuDzuhur;
    _asharCtrl.text   = p.waktuAshar;
    _maghribCtrl.text = p.waktuMaghrib;
    _isyaCtrl.text    = p.waktuIsya;
    showDialog(
      context: ctx,
      builder: (_) => AlertDialog(
        title: Text(p._t('Edit Jadwal Sholat','Edit Prayer Schedule','تعديل جدول الصلاة')),
        content: SingleChildScrollView(child: Column(mainAxisSize: MainAxisSize.min, children: [
          _field(_subuhCtrl,   p.labelSubuh,   Icons.wb_twilight),
          const SizedBox(height: 8),
          _field(_dzuhurCtrl,  p.labelDzuhur,  Icons.wb_sunny),
          const SizedBox(height: 8),
          _field(_asharCtrl,   p.labelAshar,   Icons.wb_sunny_outlined),
          const SizedBox(height: 8),
          _field(_maghribCtrl, p.labelMaghrib, Icons.wb_twilight_outlined),
          const SizedBox(height: 8),
          _field(_isyaCtrl,    p.labelIsya,    Icons.nightlight_round),
        ])),
        actions: [
          TextButton(onPressed: () => Navigator.pop(ctx), child: Text(p.lBatal)),
          ElevatedButton(
            style: ElevatedButton.styleFrom(backgroundColor: kPrimary, foregroundColor: Colors.white),
            onPressed: () {
              p.updateJadwal(_subuhCtrl.text, _dzuhurCtrl.text, _asharCtrl.text, _maghribCtrl.text, _isyaCtrl.text);
              Navigator.pop(ctx);
              ScaffoldMessenger.of(ctx).showSnackBar(
                  SnackBar(content: Text(p._t('Jadwal disimpan!','Schedule saved!','تم حفظ الجدول!'))));
            },
            child: Text(p.lSimpan),
          ),
        ],
      ),
    );
  }

  void _gantiPass(BuildContext ctx, AppProvider p) {
    _passLamaCtrl.clear();
    _passBaruCtrl.clear();
    _passConfCtrl.clear();
    showDialog(
      context: ctx,
      builder: (_) => AlertDialog(
        title: Text(p.lGantiPassword),
        content: Column(mainAxisSize: MainAxisSize.min, children: [
          _field(_passLamaCtrl, p.lPasswordLama, Icons.lock_outline, obscure: true),
          const SizedBox(height: 8),
          _field(_passBaruCtrl, p.lPasswordBaru, Icons.lock,         obscure: true),
          const SizedBox(height: 8),
          _field(_passConfCtrl, p._t('Konfirmasi','Confirm','تأكيد'), Icons.lock_reset, obscure: true),
        ]),
        actions: [
          TextButton(onPressed: () => Navigator.pop(ctx), child: Text(p.lBatal)),
          ElevatedButton(
            style: ElevatedButton.styleFrom(backgroundColor: kPrimary, foregroundColor: Colors.white),
            onPressed: () {
              if (_passLamaCtrl.text != p.adminPass) {
                ScaffoldMessenger.of(ctx).showSnackBar(
                    SnackBar(content: Text(p._t('Password lama salah!','Old password incorrect!','كلمة المرور القديمة خاطئة!'))));
                return;
              }
              if (_passBaruCtrl.text != _passConfCtrl.text) {
                ScaffoldMessenger.of(ctx).showSnackBar(
                    SnackBar(content: Text(p._t('Konfirmasi tidak cocok!','Confirmation does not match!','التأكيد غير متطابق!'))));
                return;
              }
              p.updatePassword(_passBaruCtrl.text);
              Navigator.pop(ctx);
              ScaffoldMessenger.of(ctx).showSnackBar(
                  SnackBar(content: Text(p._t('Password berhasil diubah!','Password changed!','تم تغيير كلمة المرور!'))));
            },
            child: Text(p.lSimpan),
          ),
        ],
      ),
    );
  }

  static Widget _field(TextEditingController ctrl, String label, IconData icon, {bool obscure = false}) =>
      TextField(
        controller: ctrl, obscureText: obscure,
        decoration: InputDecoration(
          labelText: label, isDense: true,
          prefixIcon: Icon(icon, color: kPrimary, size: 20),
          filled: true, fillColor: kBg,
          border: OutlineInputBorder(borderRadius: BorderRadius.circular(8),
              borderSide: const BorderSide(color: kDivider)),
          enabledBorder: OutlineInputBorder(borderRadius: BorderRadius.circular(8),
              borderSide: const BorderSide(color: kDivider)),
        ),
      );

  @override
  Widget build(BuildContext context) {
    final sc = context.sc;
    return Consumer<AppProvider>(
      builder: (context, p, _) {
    final fmt = NumberFormat('#,##0', 'id');
    return ListView(padding: EdgeInsets.all(sc.padSm), children: [

      _FbCard(child: Padding(
        padding: const EdgeInsets.fromLTRB(12, 12, 12, 6),
        child: Column(children: [
          Row(children: [
            GestureDetector(
              onTap: () async {
                final picked = await ImagePicker().pickImage(source: ImageSource.gallery, imageQuality: 70);
                if (picked == null) return;
                final bytes = await picked.readAsBytes();
                await p.updateFotoMasjid(base64Encode(bytes));
              },
              child: Stack(children: [
                CircleAvatar(
                  radius: sc.avatarSm,
                  backgroundColor: kPrimary,
                  backgroundImage: p.fotoMasjid.isNotEmpty
                      ? MemoryImage(base64Decode(p.fotoMasjid)) : null,
                  child: p.fotoMasjid.isEmpty
                      ? Icon(Icons.mosque, color: kAccent, size: 20) : null,
                ),
                Positioned(
                  bottom: 0, right: 0,
                  child: Container(
                    width: 14, height: 14,
                    decoration: BoxDecoration(
                      color: Colors.blue,
                      shape: BoxShape.circle,
                      border: Border.all(color: Colors.white, width: 1),
                    ),
                    child: Icon(Icons.camera_alt, color: Colors.white, size: 9),
                  ),
                ),
              ]),
            ),
            const SizedBox(width: 10),
            Expanded(
              child: GestureDetector(
                onTap: () => _tambahPengumuman(context, p),
                child: Container(
                  padding: const EdgeInsets.symmetric(horizontal: 14, vertical: 11),
                  decoration: BoxDecoration(
                    border: Border.all(color: kDivider),
                    borderRadius: BorderRadius.circular(22),
                  ),
                  child: Text(
                    p._t('Apa pengumuman hari ini, ${p.namaIbadah.split(" ").first}?',"What's today's announcement, ${p.namaIbadah.split(' ').first}?",'ما إعلان اليوم، ${p.namaIbadah.split(" ").first}؟'),
                    style: TextStyle(color: kTextGrey, fontSize: sc.body),
                  ),
                ),
              ),
            ),
          ]),
          const Divider(color: kDivider, height: 20),

          Row(mainAxisAlignment: MainAxisAlignment.spaceAround, children: [
            Expanded(child: _FbPostAction(icon: Icons.live_tv,   color: Colors.red,    label: p._t('Live','Live','بث مباشر'),
                onTap: () => p.startLive())),
            Expanded(child: _FbPostAction(icon: Icons.access_time, color: Colors.blue, label: p._t('Jadwal','Schedule','جدول'),
                onTap: () => _editJadwal(context, p))),
            Expanded(child: _FbPostAction(icon: Icons.edit,       color: Colors.green,  label: p._t('Profil','Profile','ملف'),
                onTap: () => _editProfil(context, p))),
          ]),
          const SizedBox(height: 4),
        ]),
      )),
      const SizedBox(height: 8),

      // ── SHORTCUT GRID - Responsive ──
      Padding(
        padding: EdgeInsets.symmetric(horizontal: sc.padXs),
        child: LayoutBuilder(builder: (ctx, bc) {
          final btns = [
            _ShortcutBtn(icon: Icons.inventory_2, label: p.lInventaris, color: Colors.teal,
              onTap: () => Navigator.push(context, MaterialPageRoute(builder: (_) => InventarisTab(onBack: () => Navigator.pop(context))))),
            _ShortcutBtn(icon: Icons.celebration, label: p.lAcara, color: Colors.orange,
              onTap: () => Navigator.push(context, MaterialPageRoute(builder: (_) => AcaraPage(onBack: () => Navigator.pop(context))))),
            _ShortcutBtn(icon: Icons.mosque, label: p.lJadwalJumat, color: kPrimary,
              onTap: () => Navigator.push(context, MaterialPageRoute(builder: (_) => JadwalJumatPage(onBack: () => Navigator.pop(context))))),
            _ShortcutBtn(icon: Icons.bar_chart, label: p._t('Statistik','Statistics','الإحصاء'), color: Colors.blue,
              onTap: () => Navigator.push(context, MaterialPageRoute(builder: (_) => StatistikPage(onBack: () => Navigator.pop(context))))),
            _ShortcutBtn(icon: Icons.menu, label: p._t('Lainnya','More','المزيد'), color: kPrimary,
              onTap: () => Navigator.push(context, MaterialPageRoute(builder: (_) => _LainnyaPage()))),
          ];
          // Layar lebar: semua dalam 1 baris, layar sempit: 2 baris 2 kolom
          if (bc.maxWidth < 280) {
            return Wrap(
              spacing: 4, runSpacing: 4,
              children: btns.map((b) => SizedBox(
                width: (bc.maxWidth - 12) / 2, child: b)).toList(),
            );
          }
          return Row(children: btns.map((b) => Expanded(child: b)).toList());
        }),
      ),
      const SizedBox(height: 8),

      // Tampilan pengumuman di Admin Home
      if (p.pengumuman.isEmpty)
        _FbCard(child: Padding(
          padding: EdgeInsets.symmetric(vertical: sc.padXs, horizontal: sc.padSm),
          child: Row(children: [
            Icon(Icons.campaign_outlined, size: sc.iconSm, color: kTextGrey),
            const SizedBox(width: 8),
            Flexible(child: Text(p.lBelumAdaPengumuman,
                overflow: TextOverflow.ellipsis,
                style: const TextStyle(color: kTextGrey, fontSize: 13))),
          ]),
        ))
      else
        ...List.generate(p.pengumuman.length, (i) {
          final um = p.pengumuman[i];
          final teks = um['teks']?.toString() ?? '';
          final waktu = um['waktu']?.toString() ?? '';
          final foto = um['foto']?.toString() ?? '';
          String waktuStr = '';
          try {
            final dt = DateTime.parse(waktu);
            final locale = p.bahasa == 'ar' ? 'ar' : p.bahasa == 'en' ? 'en' : 'id';
            try { waktuStr = DateFormat('d MMM yyyy, HH:mm', locale).format(dt); }
            catch (_) { waktuStr = DateFormat('d MMM yyyy, HH:mm').format(dt); }
          } catch (_) {}
          return _FbCard(child: Padding(
            padding: const EdgeInsets.all(12),
            child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
              Row(children: [
                CircleAvatar(radius: 18, backgroundColor: kPrimary,
                    child: const Icon(Icons.campaign, color: Colors.white, size: 18)),
                const SizedBox(width: 8),
                Expanded(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                  Text(p.namaIbadah, overflow: TextOverflow.ellipsis, style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 13)),
                  Text(waktuStr.isNotEmpty ? waktuStr : p.lPengumuman, style: TextStyle(color: kTextGrey, fontSize: 11)),
                ])),
                IconButton(
                  icon: const Icon(Icons.delete_outline, color: kTextGrey, size: 20),
                  onPressed: () => p.hapusPengumuman(i),
                ),
              ]),
              if (foto.isNotEmpty) ...[
                const SizedBox(height: 8),
                ClipRRect(
                  borderRadius: BorderRadius.circular(8),
                  child: Image.memory(base64Decode(foto),
                    width: double.infinity, height: 160, fit: BoxFit.cover),
                ),
              ],
              if (teks.isNotEmpty) ...[
                const SizedBox(height: 8),
                Text(teks, style: const TextStyle(fontSize: 14),
                  maxLines: 3, overflow: TextOverflow.ellipsis),
              ],
            ]),
          ));
        }),

      const SizedBox(height: 8),

      // Jadwal Sholat horizontal 5 kotak
      _FbCard(child: Padding(
        padding: const EdgeInsets.symmetric(vertical: 10, horizontal: 8),
        child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
          Row(children: [
            Icon(Icons.access_time, color: kPrimary, size: 16),
            const SizedBox(width: 6),
            Expanded(child: Text(p.lJadwalSholat, style: TextStyle(color: kPrimary, fontWeight: FontWeight.bold, fontSize: 13))),
            IconButton(icon: Icon(Icons.edit, size: 16, color: kIconGrey), onPressed: () => _editJadwal(context, p), padding: EdgeInsets.zero, constraints: BoxConstraints()),
          ]),
          const SizedBox(height: 8),
          Row(children: [
            _SholatBox(p.labelSubuh,   p.waktuSubuh),
            _SholatBox(p.labelDzuhur,  p.waktuDzuhur),
            _SholatBox(p.labelAshar,   p.waktuAshar),
            _SholatBox(p.labelMaghrib, p.waktuMaghrib),
            _SholatBox(p.labelIsya,    p.waktuIsya),
          ]),
        ]),
      )),
      const SizedBox(height: 4),

      _FbCard(child: Column(children: [
        _FbCardHeader(icon: Icons.account_balance_wallet, title: p._t('Ringkasan Kas','Cash Summary','ملخص الصندوق'),
            sub: p._t('Laporan Keuangan','Financial Report','التقرير المالي')),
        const Divider(color: kDivider, height: 1),
        Container(
          color: kPrimary,
          padding: const EdgeInsets.symmetric(vertical: 10, horizontal: 8),
          child: Column(children: [
            // Baris saldo awal
            // Hint untuk user baru
            if (p.saldoAwal == 0 && p.totalMasukAll == 0)
              Padding(
                padding: const EdgeInsets.only(bottom: 4),
                child: Row(children: [
                  const Icon(Icons.info_outline, color: Colors.amber, size: 12),
                  const SizedBox(width: 4),
                  Expanded(child: Text(
                    p._t('Tap untuk input saldo awal kas','Tap to set opening balance','انقر لإدخال الرصيد الأولي'),
                    style: const TextStyle(color: Colors.amber, fontSize: 10),
                  )),
                ]),
              ),
            GestureDetector(
              onTap: () {
                final ctrl = TextEditingController(text: p.saldoAwal > 0 ? p.saldoAwal.toStringAsFixed(0) : '');
                showDialog(context: context, builder: (_) => AlertDialog(
                  title: Row(children: [
                    const Icon(Icons.account_balance_wallet, color: kPrimary),
                    const SizedBox(width: 8),
                    Expanded(child: Text(p._t('Saldo Awal Kas','Opening Balance','الرصيد الأولي'))),
                  ]),
                  content: Column(mainAxisSize: MainAxisSize.min, children: [
                    Text(p._t(
                      'Masukkan saldo kas sebelum pakai aplikasi.',
                      'Enter existing cash balance before using app.',
                      'أدخل رصيد الصندوق قبل استخدام التطبيق.',
                    ), style: const TextStyle(fontSize: 12, color: Colors.grey)),
                    const SizedBox(height: 12),
                    TextField(
                      controller: ctrl,
                      keyboardType: TextInputType.number,
                      autofocus: true,
                      decoration: InputDecoration(
                        labelText: p._t('Saldo Awal (Rp)','Opening Balance (Rp)','الرصيد الأولي (Rp)'),
                        prefixIcon: const Icon(Icons.account_balance_wallet, color: kPrimary),
                        border: OutlineInputBorder(borderRadius: BorderRadius.circular(8)),
                      ),
                    ),
                  ]),
                  actions: [
                    TextButton(onPressed: () => Navigator.pop(context), child: Text(p.lBatal)),
                    ElevatedButton(
                      style: ElevatedButton.styleFrom(backgroundColor: kPrimary, foregroundColor: Colors.white),
                      onPressed: () {
                        final nilai = double.tryParse(ctrl.text.replaceAll('.','').replaceAll(',','')) ?? 0;
                        p.setSaldoAwal(nilai);
                        Navigator.pop(context);
                        ScaffoldMessenger.of(context).showSnackBar(SnackBar(
                          content: Text(p._t('Saldo awal berhasil diatur!','Opening balance set!','تم تعيين الرصيد الأولي!')),
                          backgroundColor: Colors.green));
                      },
                      child: Text(p.lSimpan),
                    ),
                  ],
                ));
              },
              child: Row(mainAxisAlignment: MainAxisAlignment.spaceBetween, children: [
                Text(p._t('Saldo Awal','Opening Balance','الرصيد الأولي'),
                  style: const TextStyle(color: Colors.white70, fontSize: 11)),
                Row(children: [
                  Text('Rp ${fmt.format(p.saldoAwal)}',
                    style: const TextStyle(color: Colors.white, fontSize: 11, fontWeight: FontWeight.bold)),
                  const SizedBox(width: 4),
                  const Icon(Icons.edit, color: Colors.white54, size: 12),
                ]),
              ]),
            ),
            const SizedBox(height: 6),
            Row(children: [
              Expanded(child: _KasStat(p.lSaldo,
                  "Rp ${fmt.format(p.saldo)}", p.saldo >= 0 ? Colors.greenAccent : Colors.redAccent)),
              Container(width: 1, height: 36, color: Colors.white24),
              Expanded(child: _KasStat(p.lMasuk, "Rp ${fmt.format(p.totalMasukAll)}", Colors.greenAccent)),
              Container(width: 1, height: 36, color: Colors.white24),
              Expanded(child: _KasStat(p.lKeluar2, "Rp ${fmt.format(p.totalKeluarAll)}", Colors.redAccent)),
            ]),
          ]),
        ),
        const Divider(color: kDivider, height: 1),
        Row(children: [
          Expanded(child: TextButton.icon(
            onPressed: () {},
            icon: Icon(Icons.thumb_up_outlined, size: 18, color: kIconGrey),
            label: Text(p._t('Suka','Like','إعجاب'), style: TextStyle(color: kIconGrey)),
          )),
          Container(width: 1, height: 36, color: kDivider),
          Expanded(child: TextButton.icon(
            onPressed: () {},
            icon: Icon(Icons.share_outlined, size: 18, color: kIconGrey),
            label: Text(p._t('Bagikan','Share','مشاركة'), style: TextStyle(color: kIconGrey)),
          )),
        ]),
      ])),

      const SizedBox(height: 8),

      _FbCard(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
        _FbCardHeader(icon: Icons.people, title: p._t('Struktur Pengurus','Committee Structure','هيكل اللجنة'),
            sub: "${p.pengurus.length} ${p._t('anggota terdaftar','members registered','أعضاء مسجلون')}"),
        const Divider(color: kDivider, height: 1),
        ...p.pengurus.take(3).map((pg) => ListTile(
          dense: true,
          leading: CircleAvatar(radius: 16, backgroundColor: kBg,
              child: Text(
                (pg['nama'] ?? '-').isNotEmpty ? (pg['nama'] ?? '-')[0].toUpperCase() : '?',
                style: const TextStyle(color: kPrimary, fontWeight: FontWeight.bold),
              )),
          title: Text(pg['nama'] ?? '-',
              style: const TextStyle(fontWeight: FontWeight.w600, fontSize: 14)),
          subtitle: Text(pg['jabatan'] ?? '-',
              style: const TextStyle(color: kPrimary, fontSize: 12)),
        )),
        if (p.pengurus.length > 3)
          Padding(
            padding: const EdgeInsets.fromLTRB(16, 0, 16, 8),
            child: Text("+ ${p.pengurus.length - 3} ${p._t('lainnya','others','آخرين')}",
                style: const TextStyle(color: kTextGrey, fontSize: 12)),
          ),
        const Divider(color: kDivider, height: 1),
        Row(children: [
          Expanded(child: TextButton.icon(
            onPressed: () {},
            icon: Icon(Icons.thumb_up_outlined, size: 18, color: kIconGrey),
            label: Text(p._t('Suka','Like','إعجاب'), style: TextStyle(color: kIconGrey)),
          )),
          Container(width: 1, height: 36, color: kDivider),
          Expanded(child: TextButton.icon(
            onPressed: () {},
            icon: Icon(Icons.share_outlined, size: 18, color: kIconGrey),
            label: Text(p._t('Bagikan','Share','مشاركة'), style: TextStyle(color: kIconGrey)),
          )),
        ]),
      ])),
      const SizedBox(height: 8),

      _FbCard(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
        _FbCardHeader(icon: Icons.security, title: p._t('Keamanan & Lisensi','Security & License','الأمان والترخيص'), sub: p._t('Status aktif','Active status','الحالة نشطة')),
        const Divider(color: kDivider, height: 1),
        ListTile(
          dense: true,
          leading: const Icon(Icons.verified_user, color: Colors.green),
          title: Text(p._t('Status Lisensi','License Status','حالة الترخيص')),
          subtitle: Text(p._t('AKTIF — Sekali Beli Full Fitur','ACTIVE — One-time Purchase Full Features','نشط — شراء مرة واحدة كامل الميزات')),
          trailing: const Icon(Icons.check_circle, color: Colors.green),
        ),
        ListTile(
          dense: true,
          leading: Icon(Icons.qr_code, color: kPrimary),
          title: Text(p._t('Kode Unik Perangkat','Device Unique Code','رمز الجهاز الفريد')),
          subtitle: Text(p.houseUniqueCode,
              style: const TextStyle(fontWeight: FontWeight.bold)),
          trailing: IconButton(
            icon: Icon(Icons.refresh, color: kIconGrey, size: 18),
            onPressed: () => showDialog(
              context: context,
              builder: (_) => AlertDialog(
                title: Text(p._t('Generate Ulang Kode?','Regenerate Code?','إعادة توليد الرمز؟')),
                content: Text(p._t('Kode lama akan terganti.','Old code will be replaced.','سيتم استبدال الرمز القديم.')),
                actions: [
                  TextButton(onPressed: () => Navigator.pop(context), child: Text(p.lBatal)),
                  ElevatedButton(
                    style: ElevatedButton.styleFrom(backgroundColor: kPrimary, foregroundColor: Colors.white),
                    onPressed: () { p.generateUniqueCode(); Navigator.pop(context); },
                    child: Text(p._t('GENERATE','GENERATE','توليد')),
                  ),
                ],
              ),
            ),
          ),
        ),
        ListTile(
          dense: true,
          leading: const Icon(Icons.lock, color: kPrimary),
          title: Text(p.lGantiPassword),
          trailing: const Icon(Icons.chevron_right, color: kIconGrey),
          onTap: () => _gantiPass(context, p),
        ),
      ])),
      const SizedBox(height: 16),
    ]);
    }, // end Consumer builder
    );
  }
}

// ══════════════════════════════════════════════
//  STATISTIK PAGE
// ══════════════════════════════════════════════
class StatistikPage extends StatefulWidget {
  final VoidCallback? onBack;
  const StatistikPage({super.key, this.onBack});
  @override
  State<StatistikPage> createState() => _StatistikPageState();
}

class _StatistikPageState extends State<StatistikPage> {
  int _tab = 0; // 0=Kas, 1=Donatur

  List<Map<String, dynamic>> _getDataBulanan(List<Map<String, dynamic>> transaksi, String locale) {
    final now = DateTime.now();
    final result = <Map<String, dynamic>>[];
    for (int i = 5; i >= 0; i--) {
      final bulan = DateTime(now.year, now.month - i, 1);
      final key = '${bulan.year}-${bulan.month.toString().padLeft(2,'0')}';
      String label;
      try { label = DateFormat('MMM', locale).format(bulan); }
      catch (_) { label = DateFormat('MMM').format(bulan); }
      double masuk = 0, keluar = 0;
      for (final t in transaksi) {
        final tglStr = t['tgl']?.toString() ?? '';
        try {
          final parts = tglStr.split('/');
          if (parts.length == 3) {
            final tglBulan = '${parts[2]}-${parts[1].padLeft(2,'0')}';
            if (tglBulan == key) {
              final jml = (t['jumlah'] is num) ? (t['jumlah'] as num).toDouble() : 0.0;
              if (t['jenis'] == 'masuk') masuk += jml;
              else keluar += jml;
            }
          }
        } catch (_) {}
      }
      result.add({'label': label, 'masuk': masuk, 'keluar': keluar});
    }
    return result;
  }

  List<Map<String, dynamic>> _getDonaturBulanan(List<Map<String, dynamic>> donaturList, String locale) {
    final now = DateTime.now();
    final result = <Map<String, dynamic>>[];
    for (int i = 5; i >= 0; i--) {
      final bulan = DateTime(now.year, now.month - i, 1);
      final key = '${bulan.month.toString().padLeft(2,'0')}/${bulan.year}';
      String label;
      try { label = DateFormat('MMM', locale).format(bulan); }
      catch (_) { label = DateFormat('MMM').format(bulan); }
      double total = 0; int jumlah = 0;
      for (final d in donaturList) {
        final tglRaw4 = d['tgl']?.toString() ?? '';
        final tgl = tglRaw4;  // keep raw for date parsing
        if (tgl.contains('/')) {
          final parts = tgl.split('/');
          if (parts.length == 3) {
            final bulanKey = '${parts[1].padLeft(2,'0')}/${parts[2]}';
            if (bulanKey == key && d['matauang'] == 'IDR') {
              total += (d['jumlah'] is num) ? (d['jumlah'] as num).toDouble() : 0.0;
              jumlah++;
            }
          }
        }
      }
      result.add({'label': label, 'total': total, 'jumlah': jumlah});
    }
    return result;
  }

  Widget _buildGrafikBatang({
    required List<Map<String, dynamic>> data,
    required String keyA, required String keyB,
    required Color colorA, required Color colorB,
    required String labelA, required String labelB,
    required NumberFormat fmt,
  }) {
    final allValues = data.expand((d) => [
      (d[keyA] as num).toDouble(), (d[keyB] as num).toDouble()
    ]).toList();
    final maxVal = allValues.isEmpty ? 1.0 : (allValues.reduce((a, b) => a > b ? a : b) * 1.2);
    final hasData = allValues.any((v) => v > 0);

    if (!hasData) {
      return SizedBox(height: 150,
        child: Center(child: Consumer<AppProvider>(
          builder: (_, p, __) => Text(p._t('Belum ada data','No data yet','لا بيانات بعد'), style: const TextStyle(color: Colors.grey)))));
    }

    return Column(children: [
      // Legend
      Row(mainAxisAlignment: MainAxisAlignment.center, children: [
        Container(width: 12, height: 12, decoration: BoxDecoration(
          color: colorA, borderRadius: BorderRadius.circular(2))),
        const SizedBox(width: 4),
        Text(labelA, style: const TextStyle(fontSize: 11)),
        const SizedBox(width: 16),
        Container(width: 12, height: 12, decoration: BoxDecoration(
          color: colorB, borderRadius: BorderRadius.circular(2))),
        const SizedBox(width: 4),
        Text(labelB, style: const TextStyle(fontSize: 11)),
      ]),
      const SizedBox(height: 12),
      SizedBox(
        height: 180,
        child: Row(crossAxisAlignment: CrossAxisAlignment.end,
          children: data.asMap().entries.map((e) {
            final d = e.value;
            final vA = (d[keyA] as num).toDouble();
            final vB = (d[keyB] as num).toDouble();
            final hA = maxVal > 0 ? (vA / maxVal * 150) : 0.0;
            final hB = maxVal > 0 ? (vB / maxVal * 150) : 0.0;
            return Expanded(child: Column(mainAxisAlignment: MainAxisAlignment.end, children: [
              Row(mainAxisAlignment: MainAxisAlignment.center,
                crossAxisAlignment: CrossAxisAlignment.end, children: [
                Container(width: 14, height: hA.clamp(2.0, 150.0),
                  decoration: BoxDecoration(color: colorA,
                    borderRadius: const BorderRadius.vertical(top: Radius.circular(3)))),
                const SizedBox(width: 3),
                if (keyB != keyA) Container(width: 14, height: hB.clamp(2.0, 150.0),
                  decoration: BoxDecoration(color: colorB,
                    borderRadius: const BorderRadius.vertical(top: Radius.circular(3)))),
              ]),
              const SizedBox(height: 4),
              Text(d['label'], style: TextStyle(fontSize: 10, color: Colors.grey[600])),
            ]));
          }).toList(),
        ),
      ),
    ]);
  }

  @override
  Widget build(BuildContext context) {
    final p = context.watch<AppProvider>();
    final fmt = NumberFormat('#,##0', 'id');
    final locale = p.bahasa == 'en' ? 'en' : p.bahasa == 'ar' ? 'ar' : 'id';
    final dataBulanan = _getDataBulanan(p.transaksi, locale);
    final dataDonatur = _getDonaturBulanan(p.donaturList, locale);
    final totalMasuk6  = dataBulanan.fold(0.0, (s, d) => s + (d['masuk']  as double));
    final totalKeluar6 = dataBulanan.fold(0.0, (s, d) => s + (d['keluar'] as double));
    final totalDonasi6 = dataDonatur.fold(0.0, (s, d) => s + (d['total']  as double));

    // Kategori pengeluaran
    final Map<String, double> kategoriKeluar = {};
    for (final t in p.transaksi.where((t) => t['jenis'] == 'keluar')) {
      final ket = t['keterangan']?.toString() ?? 'Lainnya';
      final jml = (t['jumlah'] is num) ? (t['jumlah'] as num).toDouble() : 0.0;
      kategoriKeluar[ket] = (kategoriKeluar[ket] ?? 0) + jml;
    }
    final topKeluar = kategoriKeluar.entries.toList()
      ..sort((a, b) => b.value.compareTo(a.value));

    // Top donatur
    final Map<String, double> perDonatur = {};
    for (final d in p.donaturList.where((d) => d['matauang'] == 'IDR')) {
      final nama = d['nama']?.toString() ?? 'Anonim';
      final jml  = (d['jumlah'] is num) ? (d['jumlah'] as num).toDouble() : 0.0;
      perDonatur[nama] = (perDonatur[nama] ?? 0) + jml;
    }
    final topDonatur = perDonatur.entries.toList()
      ..sort((a, b) => b.value.compareTo(a.value));

    return Scaffold(
      backgroundColor: kBg,
      body: ZoomWrapper(child: Column(children: [
        // Header
        Container(
          color: const Color(0xFF1a3a5c),
          padding: const EdgeInsets.symmetric(horizontal: 4, vertical: 10),
          child: Row(children: [
            IconButton(
              icon: const Icon(Icons.arrow_back, color: Colors.white),
              onPressed: () => widget.onBack?.call(),
            ),
            const Icon(Icons.bar_chart, color: Colors.white, size: 20),
            const SizedBox(width: 8),
            Expanded(child: Consumer<AppProvider>(builder: (_, p, __) => Text(p._t('STATISTIK','STATISTICS','الإحصاء'),
              style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold, fontSize: 16)))),
          ]),
        ),
        // Tab
        Container(
          color: Colors.white,
          child: Row(children: [
            Expanded(child: GestureDetector(
              onTap: () => setState(() => _tab = 0),
              child: Container(
                padding: const EdgeInsets.symmetric(vertical: 12),
                decoration: BoxDecoration(
                  border: Border(bottom: BorderSide(
                    color: _tab == 0 ? const Color(0xFF1a3a5c) : Colors.transparent, width: 2))),
                child: Text(p._t('Kas & Keuangan','Cash & Finance','الصندوق والمالية'), textAlign: TextAlign.center,
                  style: TextStyle(fontWeight: FontWeight.bold, fontSize: 13,
                    color: _tab == 0 ? const Color(0xFF1a3a5c) : Colors.grey)),
              ),
            )),
            Expanded(child: GestureDetector(
              onTap: () => setState(() => _tab = 1),
              child: Container(
                padding: const EdgeInsets.symmetric(vertical: 12),
                decoration: BoxDecoration(
                  border: Border(bottom: BorderSide(
                    color: _tab == 1 ? const Color(0xFF1a3a5c) : Colors.transparent, width: 2))),
                child: Text(p._t('Donatur','Donors','المتبرعون'), textAlign: TextAlign.center,
                  style: TextStyle(fontWeight: FontWeight.bold, fontSize: 13,
                    color: _tab == 1 ? const Color(0xFF1a3a5c) : Colors.grey)),
              ),
            )),
          ]),
        ),
        // Konten
        Expanded(child: SingleChildScrollView(
          padding: const EdgeInsets.all(12),
          child: _tab == 0
            ? Column(children: [
                // Ringkasan 3 kotak
                Row(children: [
                  Expanded(child: statBox(p._t('Total Masuk','Total Income','إجمالي الدخل'), 'Rp ${fmt.format(p.totalMasukAll)}', Colors.green, Icons.arrow_downward)),
                  const SizedBox(width: 8),
                  Expanded(child: statBox(p._t('Total Keluar','Total Expense','إجمالي المصروف'), 'Rp ${fmt.format(p.totalKeluarAll)}', kPrimary, Icons.arrow_upward)),
                  const SizedBox(width: 8),
                  Expanded(child: statBox(p.lSaldo, 'Rp ${fmt.format(p.saldoAkhir)}',
                    p.saldo >= 0 ? Colors.teal : Colors.red, Icons.account_balance_wallet)),
                ]),
                const SizedBox(height: 12),
                Container(
                  padding: const EdgeInsets.all(14),
                  decoration: BoxDecoration(color: Colors.white,
                    borderRadius: BorderRadius.circular(12),
                    boxShadow: [BoxShadow(color: kDivider, blurRadius: 4)]),
                  child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                    Text(p._t('Pemasukan vs Pengeluaran (6 Bulan)','Income vs Expense (6 Months)','الدخل مقابل المصروف (6 أشهر)'),
                      style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 13)),
                    const SizedBox(height: 4),
                    Text('${p.lMasuk} Rp ${fmt.format(totalMasuk6)}  •  ${p.lKeluar2} Rp ${fmt.format(totalKeluar6)}',
                      style: const TextStyle(fontSize: 11, color: Colors.grey)),
                    const SizedBox(height: 12),
                    _buildGrafikBatang(data: dataBulanan,
                      keyA: 'masuk', keyB: 'keluar',
                      colorA: Colors.green, colorB: kPrimary,
                      labelA: p.lMasuk, labelB: p.lKeluar2, fmt: fmt),
                  ]),
                ),
                const SizedBox(height: 12),
                if (topKeluar.isNotEmpty)
                Container(
                  padding: const EdgeInsets.all(14),
                  decoration: BoxDecoration(color: Colors.white,
                    borderRadius: BorderRadius.circular(12),
                    boxShadow: [BoxShadow(color: kDivider, blurRadius: 4)]),
                  child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                    Text(p._t('Top Pengeluaran','Top Expenses','أعلى المصروفات'),
                      style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 13)),
                    const SizedBox(height: 10),
                    ...topKeluar.take(5).map((e) {
                      final pct = p.totalKeluar > 0 ? e.value / p.totalKeluar : 0.0;
                      return Padding(
                        padding: const EdgeInsets.only(bottom: 8),
                        child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                          Row(children: [
                            Expanded(child: Text(e.key, style: const TextStyle(fontSize: 12),
                              overflow: TextOverflow.ellipsis)),
                            Text('Rp ${fmt.format(e.value)}',
                              style: const TextStyle(fontSize: 12, fontWeight: FontWeight.bold)),
                          ]),
                          const SizedBox(height: 4),
                          LinearProgressIndicator(
                            value: pct.clamp(0.0, 1.0),
                            backgroundColor: Colors.grey[200],
                            color: kPrimary,
                            minHeight: 6,
                            borderRadius: BorderRadius.circular(3),
                          ),
                        ]),
                      );
                    }),
                  ]),
                ),
              ])
            : Column(children: [
                // Ringkasan donatur
                Row(children: [
                  Expanded(child: statBox(p._t('Total Donatur','Total Donors','إجمالي المتبرعين'), '${p.donaturList.length} ${p._t('orang','people','شخص')}',
                    Colors.blue, Icons.people)),
                  const SizedBox(width: 8),
                  Expanded(child: statBox(p._t('Total Donasi','Total Donation','إجمالي التبرع'), 'Rp ${fmt.format(p.totalDonatur)}',
                    Colors.green, Icons.volunteer_activism)),
                  const SizedBox(width: 8),
                  Expanded(child: statBox(p._t('6 Bln Terakhir','Last 6 Months','آخر 6 أشهر'), 'Rp ${fmt.format(totalDonasi6)}',
                    Colors.teal, Icons.date_range)),
                ]),
                const SizedBox(height: 12),
                Container(
                  padding: const EdgeInsets.all(14),
                  decoration: BoxDecoration(color: Colors.white,
                    borderRadius: BorderRadius.circular(12),
                    boxShadow: [BoxShadow(color: kDivider, blurRadius: 4)]),
                  child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                    Text(p._t('Donasi per Bulan (6 Bulan)','Donation per Month (6 Months)','التبرع شهرياً (6 أشهر)'),
                      style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 13)),
                    const SizedBox(height: 12),
                    _buildGrafikBatang(data: dataDonatur,
                      keyA: 'total', keyB: 'total',
                      colorA: Colors.blue, colorB: Colors.blue,
                      labelA: p._t('Donasi (IDR)','Donation (IDR)','تبرع (IDR)'),
                      labelB: p._t('Donasi (IDR)','Donation (IDR)','تبرع (IDR)'), fmt: fmt),
                  ]),
                ),
                const SizedBox(height: 12),
                if (topDonatur.isNotEmpty)
                Container(
                  padding: const EdgeInsets.all(14),
                  decoration: BoxDecoration(color: Colors.white,
                    borderRadius: BorderRadius.circular(12),
                    boxShadow: [BoxShadow(color: kDivider, blurRadius: 4)]),
                  child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                    Text(p._t('Top Donatur','Top Donors','أفضل المتبرعين'),
                      style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 13)),
                    const SizedBox(height: 10),
                    ...topDonatur.take(5).toList().asMap().entries.map((e) {
                      final rank = e.key + 1;
                      final pct = p.totalDonatur > 0 ? e.value.value / p.totalDonatur : 0.0;
                      final medalColors = [Colors.amber, Colors.grey, Colors.brown];
                      return Padding(
                        padding: const EdgeInsets.only(bottom: 10),
                        child: Row(children: [
                          Container(
                            width: 28, height: 28,
                            decoration: BoxDecoration(
                              color: rank <= 3 ? medalColors[rank-1] : Colors.grey[300],
                              shape: BoxShape.circle),
                            child: Center(child: Text('$rank',
                              style: const TextStyle(color: Colors.white,
                                fontWeight: FontWeight.bold, fontSize: 12))),
                          ),
                          const SizedBox(width: 10),
                          Expanded(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                            Text(e.value.key, style: const TextStyle(
                              fontWeight: FontWeight.bold, fontSize: 13)),
                            const SizedBox(height: 3),
                            LinearProgressIndicator(
                              value: pct.clamp(0.0, 1.0),
                              backgroundColor: Colors.grey[200],
                              color: Colors.blue,
                              minHeight: 5,
                              borderRadius: BorderRadius.circular(3)),
                          ])),
                          const SizedBox(width: 10),
                          Text('Rp ${fmt.format(e.value.value)}',
                            style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 12)),
                        ]),
                      );
                    }),
                  ]),
                ),
              ]),
        )),
      ]),
    ));
  }
}  // _StatistikPageState

Widget statBox(String label, String value, Color color, IconData icon) {
  return Expanded(child: Container(
    padding: const EdgeInsets.symmetric(vertical: 12, horizontal: 8),
    decoration: BoxDecoration(
      color: color.withValues(alpha: 0.1),
      borderRadius: BorderRadius.circular(10),
      border: Border.all(color: color.withValues(alpha: 0.3))),
    child: Column(children: [
      Icon(icon, color: color, size: 20),
      const SizedBox(height: 4),
      FittedBox(child: Text(value,
        style: TextStyle(color: color, fontWeight: FontWeight.bold, fontSize: 13))),
      const SizedBox(height: 2),
      Text(label, style: TextStyle(color: color.withValues(alpha: 0.7), fontSize: 10),
        textAlign: TextAlign.center),
    ]),
  ));
}

class _LainnyaPage extends StatelessWidget {
  const _LainnyaPage();
  @override
  Widget build(BuildContext context) {
    final p = context.watch<AppProvider>();
    return Scaffold(
      backgroundColor: kBg,
      appBar: AppBar(
        backgroundColor: kPrimary,
        foregroundColor: Colors.white,
        title: Text(p._t('Lainnya','More','المزيد'),
          style: const TextStyle(fontWeight: FontWeight.bold)),
      ),
      body: ZoomWrapper(child: ListView(padding: const EdgeInsets.only(top: 8), children: [
        _FbMenuGroup(p._t('TOOLS','TOOLS','الأدوات'), [
          _FbMenuTile(Icons.calculate, p._t('Kalkulator','Calculator','الآلة الحاسبة'),
              () => Navigator.push(context, MaterialPageRoute(builder: (_) => const KalkulatorPage()))),
        ]),
        _FbMenuGroup(p._t('DONASI & REKENING','DONATION & ACCOUNT','التبرع والحساب'), [
          _FbMenuTile(Icons.qr_code, p._t('Upload QRIS & Rekening','Upload QRIS & Account','تحميل QRIS والحساب'),
              () => Navigator.push(context, MaterialPageRoute(builder: (_) => const QrisPage()))),
        ]),
        _FbMenuGroup(p._t('DATA & BACKUP','DATA & BACKUP','البيانات والنسخ الاحتياطي'), [
          _FbMenuTile(Icons.backup, p._t('Backup & Restore','Backup & Restore','نسخ احتياطي واستعادة'),
              () => Navigator.push(context, MaterialPageRoute(builder: (_) => const BackupPage()))),
        ]),
        _FbMenuGroup(p._t('PENGATURAN','SETTINGS','الإعدادات'), [
          _FbMenuTile(Icons.settings, p._t('Pengaturan Lanjutan','Advanced Settings','الإعدادات المتقدمة'),
              () => Navigator.push(context, MaterialPageRoute(builder: (_) => const PengaturanPage()))),
        ]),
        _FbMenuGroup(p._t('IBADAH','WORSHIP','العبادة'), [
          _FbMenuTile(Icons.mosque, p.lJadwalJumat,
              () => Navigator.push(context, MaterialPageRoute(builder: (_) => JadwalJumatPage(onBack: () => Navigator.pop(context))))),
        ]),
      ])),
    );
  }
}

class _ShortcutBtn extends StatelessWidget {
  final IconData icon;
  final String label;
  final Color color;
  final VoidCallback onTap;
  const _ShortcutBtn({required this.icon, required this.label, required this.color, required this.onTap});
  @override
  Widget build(BuildContext context) => Expanded(
    child: GestureDetector(
      onTap: onTap,
      child: Container(
        margin: const EdgeInsets.symmetric(horizontal: 3),
        padding: const EdgeInsets.symmetric(vertical: 8, horizontal: 4),
        decoration: BoxDecoration(
          color: kCard,
          borderRadius: BorderRadius.circular(16),
          boxShadow: [
            BoxShadow(color: Colors.black.withValues(alpha: 0.08), blurRadius: 10, offset: const Offset(0, 3)),
          ],
          border: Border.all(color: kBgGreen, width: 1),
        ),
        child: Column(mainAxisSize: MainAxisSize.min, children: [
          Container(
            padding: const EdgeInsets.all(8),
            decoration: BoxDecoration(
              gradient: LinearGradient(
                begin: Alignment.topLeft, end: Alignment.bottomRight,
                colors: [color.withValues(alpha: 0.15), color.withValues(alpha: 0.05)]),
              shape: BoxShape.circle,
            ),
            child: Icon(icon, color: color, size: 22),
          ),
          const SizedBox(height: 4),
          Text(label,
            style: const TextStyle(fontSize: 10, fontWeight: FontWeight.w700, color: kTextDark),
            textAlign: TextAlign.center,
            maxLines: 2,
            overflow: TextOverflow.ellipsis),
        ]),
      ),
    ),
  );
}

class _FbCard extends StatelessWidget {
  final Widget child;
  const _FbCard({required this.child});
  @override
  Widget build(BuildContext context) => Container(
    margin: const EdgeInsets.symmetric(vertical: 4),
    decoration: BoxDecoration(
      color: kCard,
      borderRadius: BorderRadius.circular(16),
      boxShadow: [
        BoxShadow(color: Colors.black.withValues(alpha: 0.07), blurRadius: 12, offset: const Offset(0, 4)),
        BoxShadow(color: Colors.black.withValues(alpha: 0.03), blurRadius: 3, offset: const Offset(0, 1)),
      ],
    ),
    child: ClipRRect(
      borderRadius: BorderRadius.circular(16),
      child: child,
    ),
  );
}

class _FbCardHeader extends StatelessWidget {
  final IconData icon;
  final String title, sub;
  const _FbCardHeader({required this.icon, required this.title, required this.sub});
  @override
  Widget build(BuildContext context) => Padding(
    padding: const EdgeInsets.fromLTRB(12, 12, 6, 8),
    child: Row(children: [
      CircleAvatar(radius: 18, backgroundColor: kPrimary,
          child: Icon(icon, color: Colors.white, size: 17)),
      const SizedBox(width: 10),
      Expanded(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
        Text(title, style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 14, color: kTextDark)),
        Text(sub,   style: const TextStyle(color: kTextGrey, fontSize: 12)),
      ])),
    ]),
  );
}

class _FbIconBtn extends StatelessWidget {
  final IconData icon;
  final Color color;
  final VoidCallback onTap;
  final String tooltip;
  const _FbIconBtn({required this.icon, required this.onTap, required this.tooltip,
      this.color = Colors.white});
  @override
  Widget build(BuildContext context) => IconButton(
    icon: Icon(icon, color: color),
    onPressed: onTap,
    tooltip: tooltip,
  );
}

class _FbPostAction extends StatelessWidget {
  final IconData icon;
  final Color color;
  final String label;
  final VoidCallback onTap;
  const _FbPostAction({required this.icon, required this.color,
      required this.label, required this.onTap});
  @override
  Widget build(BuildContext context) => TextButton.icon(
    onPressed: onTap,
    icon: Icon(icon, color: color, size: 18),
    label: Flexible(child: Text(label,
      style: const TextStyle(color: kIconGrey, fontSize: 12),
      overflow: TextOverflow.ellipsis, maxLines: 1)),
  );
}

class _SholatBox extends StatelessWidget {
  final String label, time;
  const _SholatBox(this.label, this.time);
  @override
  Widget build(BuildContext context) => Container(
      margin: const EdgeInsets.symmetric(horizontal: 2),
      padding: const EdgeInsets.symmetric(vertical: 10, horizontal: 2),
      decoration: BoxDecoration(
        gradient: LinearGradient(
          begin: Alignment.topCenter, end: Alignment.bottomCenter,
          colors: [kPrimary.withValues(alpha: 0.12), kBgGreen],
        ),
        borderRadius: BorderRadius.circular(12),
        border: Border.all(color: kPrimary.withValues(alpha: 0.15), width: 1),
      ),
      child: Column(children: [
        Text(label, style: const TextStyle(color: kPrimary, fontSize: 9, fontWeight: FontWeight.w700, letterSpacing: 0.3), textAlign: TextAlign.center),
        const SizedBox(height: 4),
        Text(time, style: const TextStyle(color: kPrimary, fontWeight: FontWeight.w900, fontSize: 13), textAlign: TextAlign.center),
      ]),
  );
}

// ── GRAFIK KEUANGAN ──
class _GrafikKeuangan extends StatefulWidget {
  final List<Map<String, dynamic>> transaksi;
  const _GrafikKeuangan({required this.transaksi});
  @override
  State<_GrafikKeuangan> createState() => _GrafikKeuanganState();
}

class _GrafikKeuanganState extends State<_GrafikKeuangan> {
  int _selectedBar = -1;

  // Ambil data 6 bulan terakhir
  List<Map<String, dynamic>> _getDataBulanan() {
    final locale = context.read<AppProvider>().bahasa == 'en' ? 'en'
        : context.read<AppProvider>().bahasa == 'ar' ? 'ar' : 'id';
    final now = DateTime.now();
    final result = <Map<String, dynamic>>[];
    for (int i = 5; i >= 0; i--) {
      final bulan = DateTime(now.year, now.month - i, 1);
      final key = '${bulan.year}-${bulan.month.toString().padLeft(2,'0')}';
      String label;
      try { label = DateFormat('MMM', locale).format(bulan); }
      catch (_) { label = DateFormat('MMM').format(bulan); }
      double masuk = 0, keluar = 0;
      for (final t in widget.transaksi) {
        final tglStr = t['tgl']?.toString() ?? '';
        try {
          final parts = tglStr.split('/');
          if (parts.length == 3) {
            final tglBulan = '${parts[2]}-${parts[1].padLeft(2,'0')}';
            if (tglBulan == key) {
              final jml = (t['jumlah'] is num) ? (t['jumlah'] as num).toDouble() : 0.0;
              if (t['jenis'] == 'masuk') masuk += jml;
              else keluar += jml;
            }
          }
        } catch (_) {}
      }
      result.add({'label': label, 'masuk': masuk, 'keluar': keluar});
    }
    return result;
  }

  @override
  Widget build(BuildContext context) {
    final data = _getDataBulanan();
    final fmt = NumberFormat('#,##0', 'id');
    final allValues = data.expand((d) => [d['masuk'] as double, d['keluar'] as double]).toList();
    final maxVal = allValues.isEmpty ? 1.0 : (allValues.reduce((a, b) => a > b ? a : b) * 1.2);
    final hasData = allValues.any((v) => v > 0);

    if (!hasData) {
      return SizedBox(
        height: 120,
        child: Center(child: Consumer<AppProvider>(
          builder: (_, p, __) => Column(mainAxisAlignment: MainAxisAlignment.center, children: [
            const Icon(Icons.bar_chart, size: 40, color: Colors.grey),
            const SizedBox(height: 8),
            Text(p._t('Belum ada data transaksi','No transaction data yet','لا بيانات معاملات بعد'), style: const TextStyle(color: Colors.grey, fontSize: 13)),
          ]),
        )),
      );
    }

    return Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
      // Legend
      Wrap(spacing: 8, children: [
        Row(mainAxisSize: MainAxisSize.min, children: [
        Container(width: 12, height: 12, decoration: BoxDecoration(
          color: Colors.green, borderRadius: BorderRadius.circular(2))),
        const SizedBox(width: 4),
        Consumer<AppProvider>(builder: (_, p, __) => Text(p._t('Pemasukan','Income','دخل'), style: const TextStyle(fontSize: 11))),
        ]),
        const SizedBox(width: 8),
        Container(width: 12, height: 12, decoration: BoxDecoration(
          color: kPrimary, borderRadius: BorderRadius.circular(2))),
        const SizedBox(width: 4),
        Consumer<AppProvider>(builder: (_, p, __) => Text(p._t('Pengeluaran','Expense','مصروف'), style: const TextStyle(fontSize: 11))),
      ]),
      const SizedBox(height: 12),
      // Grafik batang
      SizedBox(
        height: 160,
        child: Row(
          crossAxisAlignment: CrossAxisAlignment.end,
          children: data.asMap().entries.map((e) {
            final i = e.key;
            final d = e.value;
            final masuk  = d['masuk']  as double;
            final keluar = d['keluar'] as double;
            final hMasuk  = maxVal > 0 ? (masuk  / maxVal * 130) : 0.0;
            final hKeluar = maxVal > 0 ? (keluar / maxVal * 130) : 0.0;
            final isSelected = _selectedBar == i;
            return Expanded(child: GestureDetector(
              onTap: () => setState(() => _selectedBar = isSelected ? -1 : i),
              child: Column(mainAxisAlignment: MainAxisAlignment.end, children: [
                // Tooltip
                if (isSelected)
                  Container(
                    margin: const EdgeInsets.only(bottom: 4),
                    padding: const EdgeInsets.symmetric(horizontal: 6, vertical: 4),
                    decoration: BoxDecoration(
                      color: Colors.black87,
                      borderRadius: BorderRadius.circular(6)),
                    child: Column(children: [
                      Text('Rp ${fmt.format(masuk)}',
                        style: const TextStyle(color: Colors.greenAccent, fontSize: 9)),
                      Text('Rp ${fmt.format(keluar)}',
                        style: const TextStyle(color: Colors.redAccent, fontSize: 9)),
                    ]),
                  ),
                // Batang
                Row(mainAxisAlignment: MainAxisAlignment.center,
                  crossAxisAlignment: CrossAxisAlignment.end,
                  children: [
                    Container(
                      width: 10,
                      height: hMasuk.toDouble().clamp(2.0, 130.0),
                      decoration: BoxDecoration(
                        color: isSelected ? Colors.greenAccent : Colors.green,
                        borderRadius: const BorderRadius.vertical(top: Radius.circular(3))),
                    ),
                    const SizedBox(width: 2),
                    Container(
                      width: 10,
                      height: hKeluar.toDouble().clamp(2.0, 130.0),
                      decoration: BoxDecoration(
                        color: isSelected ? Colors.redAccent : kPrimary,
                        borderRadius: const BorderRadius.vertical(top: Radius.circular(3))),
                    ),
                  ],
                ),
                const SizedBox(height: 4),
                Text(d['label'], style: TextStyle(
                  fontSize: 10,
                  fontWeight: isSelected ? FontWeight.bold : FontWeight.normal,
                  color: isSelected ? kPrimary : Colors.grey[600])),
              ]),
            ));
          }).toList(),
        ),
      ),
      const SizedBox(height: 12),
      const Divider(height: 1),
      const SizedBox(height: 8),
      // Ringkasan total
      Builder(builder: (ctx) {
        final totalMasuk  = data.fold(0.0, (s, d) => s + (d['masuk']  as double));
        final totalKeluar = data.fold(0.0, (s, d) => s + (d['keluar'] as double));
        return Row(mainAxisAlignment: MainAxisAlignment.spaceEvenly, children: [
          Column(children: [
            Consumer<AppProvider>(builder: (_, p, __) => Text(p._t('Total Masuk (6 bln)','Total Income (6 mo)','إجمالي الدخل'), style: const TextStyle(fontSize: 10, color: Colors.grey))),
            Text('Rp ${fmt.format(totalMasuk)}',
              style: const TextStyle(color: Colors.green, fontWeight: FontWeight.bold, fontSize: 13)),
          ]),
          Container(width: 1, height: 30, color: kDivider),
          Column(children: [
            Consumer<AppProvider>(builder: (_, p, __) => Text(p._t('Total Keluar (6 bln)','Total Expense (6 mo)','إجمالي المصروف'), style: const TextStyle(fontSize: 10, color: Colors.grey))),
            Text('Rp ${fmt.format(totalKeluar)}',
              style: TextStyle(color: kPrimary, fontWeight: FontWeight.bold, fontSize: 13)),
          ]),
        ]);
      }),
    ]);
  }
}

class _KasStat extends StatelessWidget {
  final String label, value;
  final Color color;
  const _KasStat(this.label, this.value, this.color);
  @override
  Widget build(BuildContext context) => Padding(
    padding: const EdgeInsets.symmetric(horizontal: 2),
    child: Column(children: [
      FittedBox(fit: BoxFit.scaleDown,
        child: Text(label, style: const TextStyle(color: Colors.white70, fontSize: 10, letterSpacing: 0.5))),
      const SizedBox(height: 2),
      FittedBox(
        fit: BoxFit.scaleDown,
        child: Text(value, style: TextStyle(color: color, fontWeight: FontWeight.bold, fontSize: 13)),
      ),
    ]),
  );
}

class _FbMenuGroup extends StatelessWidget {
  final String title;
  final List<Widget> items;
  const _FbMenuGroup(this.title, this.items);
  @override
  Widget build(BuildContext context) => Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
    Padding(
      padding: const EdgeInsets.fromLTRB(16, 12, 16, 4),
      child: Text(title, style: const TextStyle(color: kTextGrey, fontWeight: FontWeight.bold, fontSize: 12)),
    ),
    Container(color: kCard, child: Column(children: items)),
    const SizedBox(height: 8),
  ]);
}

class _FbMenuTile extends StatelessWidget {
  final IconData icon;
  final String label;
  final VoidCallback onTap;
  const _FbMenuTile(this.icon, this.label, this.onTap);
  @override
  Widget build(BuildContext context) => Material(
    color: Colors.transparent,
    child: ListTile(
      leading: Container(width: 36, height: 36,
          decoration: BoxDecoration(color: kBg, borderRadius: BorderRadius.circular(8)),
          child: Icon(icon, color: kPrimary, size: 22)),
      title: Text(label, style: const TextStyle(fontWeight: FontWeight.w600, fontSize: 14)),
      trailing: Icon(Icons.chevron_right, color: kIconGrey),
      onTap: onTap,
    ),
  );
}

class PublicDisplay extends StatefulWidget {
  const PublicDisplay({super.key});
  @override
  State<PublicDisplay> createState() => _PublicDisplayState();
}

class _PublicDisplayState extends State<PublicDisplay> {

  int _slideIndex  = 0;
  Timer? _slideTimer;
  Timer? _adzanCheckTimer;
  bool _isPaused   = false;

  @override
  void initState() {
    super.initState();
    Future.microtask(() {
      if (mounted) _jadwalSlideBerikutnya();
    });
    // Cek waktu azan setiap 30 detik
    _adzanCheckTimer = Timer.periodic(const Duration(seconds: 30), (_) {
      if (!mounted) return;
      final p = context.read<AppProvider>();
      if (!p.smartDisplayAktif) return;
      if (p.sedangAdzan) return; // sudah aktif, skip
      final now = DateTime.now();
      final jamSekarang = '${now.hour.toString().padLeft(2,'0')}:${now.minute.toString().padLeft(2,'0')}';
      final waktuAzan = {
        p.waktuSubuh:   p.labelSubuh,
        p.waktuDzuhur:  p.labelDzuhur,
        p.waktuAshar:   p.labelAshar,
        p.waktuMaghrib: p.labelMaghrib,
        p.waktuIsya:    p.labelIsya,
      };
      if (waktuAzan.containsKey(jamSekarang)) {
        p.setSedangAdzan(true); // otomatis ON + timer matikan sendiri
        // Tampilkan notifikasi ke HP
        AdzanNotifikasi.tampilkan(waktuAzan[jamSekarang]!);
      }
    });
  }

  @override
  void dispose() {
    _slideTimer?.cancel();
    _adzanCheckTimer?.cancel();
    super.dispose();
  }

  List<Map<String, dynamic>> _buildSlides(AppProvider p) {
    final slides = <Map<String, dynamic>>[];
    // Slide profil masjid
    slides.add({'type': 'profil'});
    // Slide sidebar (jam, jadwal sholat, pengurus ringkas) - tersendiri
    slides.add({'type': 'sidebar'});
    // Slide jadwal sholat
    slides.add({'type': 'jadwal'});
    // Slide pengumuman
    for (final um in p.pengumuman) {
      slides.add({'type': 'pengumuman', 'data': um});
    }
    // Slide pengurus
    if (p.pengurus.isNotEmpty) {
      slides.add({'type': 'pengurus'});
    }
    // Slide arus kas
    slides.add({'type': 'kas'});
    // Slide donatur
    if (p.donaturList.isNotEmpty) slides.add({'type': 'donatur'});
    // Slide QRIS donasi
    if (p.fotoQris.isNotEmpty || p.noRekening.isNotEmpty) {
      slides.add({'type': 'qris'});
    }
    // Slide acara/spanduk
    for (final a in p.acaraList) {
      slides.add({'type': 'acara', 'data': a});
    }
    // Slide galeri
    for (final g in p.galeri) {
      slides.add({'type': 'galeri', 'data': g});
    }
    // Slide inventaris
    if (p.inventaris.isNotEmpty) slides.add({'type': 'inventaris'});
    // Slide countdown hari besar Islam
    slides.add({'type': 'countdown'});
    // Slide imsak Ramadhan - hanya tampil saat Ramadhan aktif
    if (p.aktifRamadhan) slides.add({'type': 'imsak'});
    // Slide jadwal jumat - hanya tampil pada hari Jumat
    if (p.jadwalJumat.isNotEmpty && DateTime.now().weekday == DateTime.friday) {
      slides.add({'type': 'jumat'});
    }
    // Slide ayat/hadits harian
    slides.add({'type': 'ayat'});
    return slides;
  }

  void _jadwalSlideBerikutnya() {
    _slideTimer?.cancel();
    if (!mounted) return;
    if (_isPaused) return;
    final p = context.read<AppProvider>();
    final slides = _buildSlides(p);
    if (slides.isEmpty) return;

    // Cek apakah slide sekarang adalah video
    final safeIdx = _slideIndex % slides.length;
    final currentSlide = slides[safeIdx];
    final isVideo = currentSlide['type'] == 'galeri' &&
        (currentSlide['data'] as Map<String, dynamic>)['type'] == 'video';

    // Kalau video, skip timer — VideoSlideWidget akan panggil _nextSlideAfterVideo
    if (isVideo) return;

    // Kalau bukan video, pakai durasi slide normal
    int durasi = 8;
    if (currentSlide['type'] == 'galeri') {
      final d = (currentSlide['data'] as Map<String, dynamic>)['durasi'];
      final dInt = d is int ? d : int.tryParse(d.toString()) ?? 8;
      durasi = dInt < 3 ? 8 : dInt; // minimum 3 detik, default 8
    }

    _slideTimer = Timer(Duration(seconds: durasi), () {
      if (!mounted || _isPaused) return;
      Future.microtask(() {
        if (!mounted) return;
        final pInner = context.read<AppProvider>();
        final slidesInner = _buildSlides(pInner);
        if (slidesInner.isEmpty) return;
        setState(() {
          _slideIndex = (_slideIndex + 1) % slidesInner.length;
        });
        _jadwalSlideBerikutnya();
      });
    });
  }

  // Dipanggil dari VideoSlideWidget saat video selesai
  void _nextSlideAfterVideo() {
    if (!mounted || _isPaused) return;
    Future.microtask(() {
      if (!mounted) return;
      final p = context.read<AppProvider>();
      final slides = _buildSlides(p);
      if (slides.isEmpty) return;
      setState(() {
        _slideIndex = (_slideIndex + 1) % slides.length;
      });
      _jadwalSlideBerikutnya();
    });
  }

  void _prevSlide() {
    final p = context.read<AppProvider>();
    final slides = _buildSlides(p);
    setState(() {
      _slideIndex = (_slideIndex - 1 + slides.length) % slides.length;
    });
    if (!_isPaused) _jadwalSlideBerikutnya();
  }

  void _nextSlide() {
    final p = context.read<AppProvider>();
    final slides = _buildSlides(p);
    setState(() {
      _slideIndex = (_slideIndex + 1) % slides.length;
    });
    if (!_isPaused) _jadwalSlideBerikutnya();
  }



  @override
  Widget build(BuildContext context) {
    final p = context.watch<AppProvider>();
    final slides = _buildSlides(p);
    final safeIndex = slides.isEmpty ? 0 : (_slideIndex % slides.length);
    final currentSlide = slides.isEmpty ? {'type': 'profil'} : slides[safeIndex];

    return ScrollConfiguration(
      behavior: ScrollConfiguration.of(context).copyWith(scrollbars: false),
      child: Scaffold(
        backgroundColor: Colors.black,
        floatingActionButton: FloatingActionButton.small(
          backgroundColor: Colors.black54,
          onPressed: () {
              p.stopLive();
              if (!context.mounted) return;
              Navigator.of(context).pushAndRemoveUntil(
                MaterialPageRoute(builder: (_) => const AdminShell()),
                (route) => false,
              );
            },
          tooltip: p._t('Kembali ke Menu','Back to Menu','العودة إلى القائمة'),
          child: const Icon(Icons.arrow_back, color: Colors.white),
        ),
        floatingActionButtonLocation: FloatingActionButtonLocation.startTop,
        body: SafeArea(
          child: LayoutBuilder(
            builder: (ctx, constraints) {
              // === RESPONSIVE SCALE ===
              // Gunakan min dari width/height scale agar tidak overflow di layar landscape
              final sw = constraints.maxWidth;
              final sh = constraints.maxHeight;
              // fs berdasarkan lebar DAN tinggi agar proporsional di semua layar
              // HP~400px=1.0 | Tablet~768px=1.5 | Laptop~1280px=2.0 | TV~1920px=2.8
              final fsW = (sw / 420.0).clamp(0.8, 3.5);
              final fsH = (sh / 550.0).clamp(0.8, 3.5);
              final fs = (fsW < fsH ? fsW : fsH).clamp(0.8, 3.0);

              return Column(
                children: [
                  Expanded(
                    child: Row(
                      children: [
                        // ── SLIDE: Full width, sidebar sudah jadi slide tersendiri ──
                        Expanded(
                          child: ClipRect(
                            child: Stack(
                              fit: StackFit.expand,
                              children: [
                                if (p.sedangAdzan && p.smartDisplayAktif)
                                  Container(
                                    color: Colors.black87,
                                    child: Center(
                                      child: Column(
                                        mainAxisAlignment: MainAxisAlignment.center,
                                        children: [
                                          Icon(Icons.mosque, color: const Color(0xFF631414), size: 60 * fs),
                                          SizedBox(height: 16 * fs),
                                          Text(p._t('LURUSKAN DAN RAPATKAN SHAF','STRAIGHTEN AND CLOSE THE ROWS','استووا واعتدلوا الصفوف'),
                                            textAlign: TextAlign.center,
                                            style: TextStyle(color: Colors.white, fontSize: 18 * fs, fontWeight: FontWeight.bold)),
                                          SizedBox(height: 8 * fs),
                                          Text(p._t('MATIKAN HANDPHONE','SILENCE YOUR PHONE','أوقف تشغيل الهاتف'),
                                            style: TextStyle(color: Colors.white70, fontSize: 13 * fs)),
                                          SizedBox(height: 16 * fs),
                                          Text(p.houseUniqueCode,
                                            style: TextStyle(color: Colors.white24, fontSize: 9 * fs)),
                                        ],
                                      ),
                                    ),
                                  )
                                else
                                  AnimatedSwitcher(
                                    duration: const Duration(milliseconds: 400),
                                    reverseDuration: const Duration(milliseconds: 0),
                                    switchInCurve: Curves.easeInOut,
                                    switchOutCurve: Curves.easeInOut,
                                    layoutBuilder: (currentChild, previousChildren) => Stack(
                                      fit: StackFit.expand,
                                      children: [
                                        ...previousChildren,
                                        if (currentChild != null) currentChild,
                                      ],
                                    ),
                                    transitionBuilder: (child, animation) =>
                                      FadeTransition(opacity: animation, child: RepaintBoundary(child: child)),
                                    child: KeyedSubtree(
                                      key: ValueKey(safeIndex),
                                      child: _buildSlideByType(currentSlide, p),
                                    ),
                                  ),

                                // Dot indicator
                                if (slides.length > 1)
                                  Positioned(
                                    top: 8, left: 0, right: 0,
                                    child: Row(
                                      mainAxisAlignment: MainAxisAlignment.center,
                                      children: List.generate(
                                        slides.length > 10 ? 10 : slides.length,
                                        (i) {
                                          final active = i == safeIndex % (slides.length > 10 ? 10 : slides.length);
                                          return Container(
                                            margin: const EdgeInsets.symmetric(horizontal: 2),
                                            width: active ? 12 : 5,
                                            height: 5,
                                            decoration: BoxDecoration(
                                              color: active ? const Color(0xFF631414) : Colors.white38,
                                              borderRadius: BorderRadius.circular(3),
                                            ),
                                          );
                                        },
                                      ),
                                    ),
                                  ),

                                // Tombol prev/next/pause
                                Positioned(
                                  bottom: 8, left: 0, right: 0,
                                  child: Row(
                                    mainAxisAlignment: MainAxisAlignment.center,
                                    children: [
                                      IconButton(icon: Icon(Icons.skip_previous, color: Colors.white70, size: 20 * fs.clamp(1.0, 1.6)), onPressed: _prevSlide),
                                      IconButton(
                                        icon: Icon(_isPaused ? Icons.play_arrow : Icons.pause, color: Colors.white70, size: 20 * fs.clamp(1.0, 1.6)),
                                        onPressed: () {
                                          setState(() => _isPaused = !_isPaused);
                                          if (_isPaused) {
                                            _slideTimer?.cancel(); // stop timer saat jeda
                                          } else {
                                            _jadwalSlideBerikutnya(); // lanjut saat play
                                          }
                                        },
                                      ),
                                      IconButton(icon: Icon(Icons.skip_next, color: Colors.white70, size: 20 * fs.clamp(1.0, 1.6)), onPressed: _nextSlide),
                                    ],
                                  ),
                                ),
                              ],
                            ),
                          ),
                        ),


                      ],
                    ),
                  ),

                  // Running text bawah
                  ClipRect(
                    child: Container(
                      color: const Color(0xFF631414),
                      padding: const EdgeInsets.symmetric(vertical: 6),
                      child: SizedBox(
                        width: double.infinity,
                        child: _RunningText(
                          text: p.runningTextAktif,
                          kecepatan: p.kecepatanRunningText,
                          warna: p.warnaRunningText,
                          fontSize: (15 * fs).clamp(13.0, 22.0),
                        ),
                      ),
                    ),
                  ),

                  // Powered by - tidak bisa dihapus user
                  Container(
                    color: Colors.black,
                    padding: const EdgeInsets.symmetric(vertical: 4, horizontal: 12),
                    child: Row(
                      mainAxisAlignment: MainAxisAlignment.spaceBetween,
                      mainAxisSize: MainAxisSize.max,
                      children: [
                        Flexible(child: Text(
                          'NovaPro - Manajemen Masjid & Mushollah',
                          overflow: TextOverflow.ellipsis,
                          style: TextStyle(
                            color: Colors.white30,
                            fontSize: (7 * fs).clamp(7.0, 11.0),
                            fontStyle: FontStyle.italic,
                          ),
                        )),
                        const SizedBox(width: 8),
                        Text(
                          'Powered by Novarizal',
                          style: TextStyle(
                            color: const Color(0xFFFFD700),
                            fontSize: (7 * fs).clamp(7.0, 11.0),
                            fontStyle: FontStyle.italic,
                            fontWeight: FontWeight.w700,
                            letterSpacing: 0.5,
                          ),
                        ),
                      ],
                    ),
                  ),
                ],
              );
            },
          ),
        ),
      ),
    );
  }

  // Jadwal box responsive
  Widget _jadwalBoxR(String title, String time, double fs) => Container(
    width: double.infinity,
    padding: EdgeInsets.symmetric(horizontal: (5 * fs).clamp(4.0, 10.0), vertical: (3 * fs).clamp(3.0, 7.0)),
    margin: const EdgeInsets.only(bottom: 2),
    decoration: BoxDecoration(color: Colors.white10, borderRadius: BorderRadius.circular(4)),
    child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
      Text(title, style: TextStyle(color: Colors.white54, fontSize: (7 * fs).clamp(7.0, 12.0), letterSpacing: 1)),
      Text(time,  style: TextStyle(color: Colors.yellowAccent, fontSize: (11 * fs).clamp(10.0, 20.0), fontWeight: FontWeight.bold, fontFamily: 'monospace')),
    ]),
  );

  Widget _buildSlideByType(Map<String, dynamic> slide, AppProvider p) {
    final type = slide['type'] as String;
    return LayoutBuilder(builder: (ctx, bc) {
      final fsW = (bc.maxWidth / 350.0).clamp(0.8, 3.0);
      final fsH = (bc.maxHeight / 500.0).clamp(0.8, 3.0);
      final fs = fsW < fsH ? fsW : fsH;
      // Consumer agar rebuild otomatis saat bahasa berubah
      return Consumer<AppProvider>(builder: (_, pp, __) => ClipRect(
        child: _buildSlideContent(type, slide, pp, fs),
      ));
    });
  }

  Widget _buildSlideContent(String type, Map<String, dynamic> slide, AppProvider p, double fs) {
    switch (type) {
      case 'sidebar':
        return LayoutBuilder(builder: (ctx, bc) {
          // Batasi sfs dari tinggi layar juga agar tidak overflow
          final sfsW = (bc.maxWidth / 420.0).clamp(0.8, 3.0);
          final sfsH = (bc.maxHeight / 600.0).clamp(0.8, 3.0);
          final sfs = sfsW < sfsH ? sfsW : sfsH;
          final padH = (bc.maxWidth * 0.04).clamp(8.0, 32.0);
          final avatarR = (bc.maxWidth * 0.07).clamp(24.0, 80.0);
          final info = p.hitungMundurAdzan();
          final modeIq = info['modeIqomah'] as bool? ?? false;
          return Container(
            color: const Color(0xFF0D0D0D),
            child: Row(children: [
              // Kolom kiri: jam & countdown
              Expanded(
                flex: 5,
                child: Padding(
                  padding: EdgeInsets.all(padH),
                  child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [
                    // Foto & nama masjid
                    p.fotoMasjid.isNotEmpty
                      ? CircleAvatar(radius: avatarR, backgroundImage: MemoryImage(base64Decode(p.fotoMasjid)))
                      : Icon(Icons.mosque, color: Colors.white70, size: avatarR * 1.2),
                    SizedBox(height: padH * 0.5),
                    Text(p.namaIbadah,
                      textAlign: TextAlign.center, overflow: TextOverflow.ellipsis, maxLines: 2,
                      style: TextStyle(color: Colors.white, fontSize: (14 * sfs).clamp(12.0, 32.0), fontWeight: FontWeight.bold)),
                    SizedBox(height: padH),
                    // Jam digital besar
                    _LiveClockResponsive(fontScale: sfs * 1.4),
                    SizedBox(height: padH * 0.5),
                    // Tanggal Hijriah
                    Text(p.tanggalHijriah,
                      textAlign: TextAlign.center, overflow: TextOverflow.ellipsis, maxLines: 2,
                      style: TextStyle(color: Colors.greenAccent, fontSize: (11 * sfs).clamp(9.0, 24.0))),
                    SizedBox(height: padH),
                    // Countdown adzan
                    if (info['menitLagi'] != -1)
                      Container(
                        padding: EdgeInsets.symmetric(horizontal: padH, vertical: padH * 0.5),
                        decoration: BoxDecoration(
                          color: modeIq ? Colors.orange.withValues(alpha: 0.3) : Colors.white10,
                          borderRadius: BorderRadius.circular(8),
                        ),
                        child: Column(children: [
                          Text(
                            modeIq ? p._t('MENUJU IQOMAH','TO IQAMAH','نحو الإقامة') : '${p._t('MENUJU','TO','نحو')} ${info['nama']}',
                            style: TextStyle(color: modeIq ? Colors.orange : Colors.white54, fontSize: (10 * sfs).clamp(9.0, 22.0), fontWeight: FontWeight.bold)),
                          Text(
                            '${info['menitLagi']}${p._t('m','m','د')} ${info['detikLagi']}${p._t('d','s','ث')}',
                            style: TextStyle(color: modeIq ? Colors.orangeAccent : Colors.yellowAccent, fontSize: (18 * sfs).clamp(14.0, 40.0), fontWeight: FontWeight.bold)),
                        ]),
                      ),
                    // Kode unik
                    SizedBox(height: padH),
                    Text(p.houseUniqueCode,
                      style: TextStyle(color: Colors.white24, fontSize: (8 * sfs).clamp(8.0, 16.0))),
                  ]),
                ),
              ),
              // Divider vertikal
              Container(width: 1, color: Colors.white12),
              // Kolom kanan: jadwal sholat + pengurus
              Expanded(
                flex: 4,
                child: Padding(
                  padding: EdgeInsets.all(padH),
                  child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [
                    // Jadwal sholat
                    _jadwalBoxR(p.labelSubuh,   p.waktuSubuh,   sfs),
                    SizedBox(height: padH * 0.3),
                    _jadwalBoxR(p.labelDzuhur,  p.waktuDzuhur,  sfs),
                    SizedBox(height: padH * 0.3),
                    _jadwalBoxR(p.labelAshar,   p.waktuAshar,   sfs),
                    SizedBox(height: padH * 0.3),
                    _jadwalBoxR(p.labelMaghrib, p.waktuMaghrib, sfs),
                    SizedBox(height: padH * 0.3),
                    _jadwalBoxR(p.labelIsya,    p.waktuIsya,    sfs),
                    SizedBox(height: padH),
                    // Pengurus ringkas
                    Container(
                      width: double.infinity,
                      padding: EdgeInsets.all(padH * 0.8),
                      decoration: BoxDecoration(color: Colors.white10, borderRadius: BorderRadius.circular(8)),
                      child: Column(children: [
                        Text(p._t('KETUA','CHAIRMAN','الرئيس'), style: TextStyle(color: Colors.white38, fontSize: (9 * sfs).clamp(8.0, 18.0))),
                        Text(p.ketua, overflow: TextOverflow.ellipsis,
                          style: TextStyle(color: Colors.white, fontSize: (11 * sfs).clamp(9.0, 22.0), fontWeight: FontWeight.bold)),
                        SizedBox(height: padH * 0.3),
                        Text(p._t('SEKRETARIS','SECRETARY','الأمين'), style: TextStyle(color: Colors.white38, fontSize: (9 * sfs).clamp(8.0, 18.0))),
                        Text(p.sekretaris, overflow: TextOverflow.ellipsis,
                          style: TextStyle(color: Colors.white, fontSize: (11 * sfs).clamp(9.0, 22.0), fontWeight: FontWeight.bold)),
                      ]),
                    ),
                  ]),
                ),
              ),
            ]),
          );
        });

      case 'profil':
        return Container(
          color: const Color(0xFF1A0505),
          child: Center(child: SingleChildScrollView(child: Column(mainAxisAlignment: MainAxisAlignment.center, mainAxisSize: MainAxisSize.min, children: [
            CircleAvatar(
              radius: (60 * fs).clamp(40.0, 120.0),
              backgroundColor: kPrimary,
              backgroundImage: p.fotoMasjid.isNotEmpty
                  ? MemoryImage(base64Decode(p.fotoMasjid)) : null,
              child: p.fotoMasjid.isEmpty
                  ? Icon(Icons.mosque, color: kAccent, size: (60 * fs).clamp(40.0, 120.0)) : null,
            ),
            SizedBox(height: 16 * fs),
            Padding(
              padding: EdgeInsets.symmetric(horizontal: 20 * fs),
              child: FittedBox(fit: BoxFit.scaleDown, child: Text(p.namaIbadah, textAlign: TextAlign.center,
                style: TextStyle(color: Colors.white, fontSize: (24 * fs).clamp(16.0, 60.0), fontWeight: FontWeight.bold))),
            ),
            SizedBox(height: 8 * fs),
            Padding(
              padding: EdgeInsets.symmetric(horizontal: 16 * fs),
              child: FittedBox(fit: BoxFit.scaleDown, child: Text(p.alamat, textAlign: TextAlign.center,
                style: TextStyle(color: Colors.white60, fontSize: (14 * fs).clamp(10.0, 32.0)))),
            ),
            SizedBox(height: 8 * fs),
            Text(p.houseUniqueCode, style: TextStyle(color: Colors.white30, fontSize: (9 * fs).clamp(8.0, 18.0))),
          ]))),
        );

      case 'jadwal':
        return Container(
          color: const Color(0xFF0A1628),
          child: Center(child: SingleChildScrollView(child: Column(mainAxisAlignment: MainAxisAlignment.center, mainAxisSize: MainAxisSize.min, children: [
            Icon(Icons.access_time, color: Colors.white54, size: (30 * fs).clamp(20.0, 80.0)),
            SizedBox(height: 6 * fs),
            Text(p._t('JADWAL SHOLAT','PRAYER TIMES','أوقات الصلاة'), style: TextStyle(color: Colors.white70, fontSize: (13 * fs).clamp(10.0, 32.0), letterSpacing: 2)),
            SizedBox(height: 4 * fs),
            FittedBox(fit: BoxFit.scaleDown, child: Text(p.namaIbadah, style: TextStyle(color: Colors.white38, fontSize: (10 * fs).clamp(8.0, 22.0)))),
            SizedBox(height: 10 * fs),
            Padding(
              padding: EdgeInsets.symmetric(horizontal: 12 * fs),
              child: Column(children: [
                _slideSholatItemR(p.labelSubuh,   p.waktuSubuh,   fs),
                SizedBox(height: 6 * fs),
                _slideSholatItemR(p.labelDzuhur,  p.waktuDzuhur,  fs),
                SizedBox(height: 6 * fs),
                _slideSholatItemR(p.labelAshar,   p.waktuAshar,   fs),
                SizedBox(height: 6 * fs),
                _slideSholatItemR(p.labelMaghrib, p.waktuMaghrib, fs),
                SizedBox(height: 6 * fs),
                _slideSholatItemR(p.labelIsya,    p.waktuIsya,    fs),
              ]),
            ),
          ]))),
        );

      case 'pengumuman':
        final um = slide['data'] as Map<String, dynamic>;
        final umFoto = um['foto'] ?? '';
        return ClipRect(child: Container(
          color: const Color(0xFF1A1A0A),
          child: umFoto.isNotEmpty
            ? Row(children: [
                Expanded(flex: 5, child: ClipRRect(
                  child: Image.memory(base64Decode(umFoto),
                    height: double.infinity, fit: BoxFit.cover),
                )),
                Expanded(flex: 5, child: Padding(
                  padding: EdgeInsets.all((16 * fs).clamp(8.0, 40.0)),
                  child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [
                    Icon(Icons.campaign, color: Colors.amber, size: (24 * fs).clamp(18.0, 52.0)),
                    SizedBox(height: 4 * fs),
                    FittedBox(child: Text(p._t('PENGUMUMAN','ANNOUNCEMENT','الإعلانات'),
                      style: const TextStyle(color: Colors.amber, fontSize: 14, letterSpacing: 2))),
                    SizedBox(height: 2 * fs),
                    FittedBox(child: Text(p.namaIbadah,
                      style: const TextStyle(color: Colors.white38, fontSize: 10))),
                    SizedBox(height: 10 * fs),
                    Text(um['teks'] ?? '', textAlign: TextAlign.center,
                      maxLines: 8, overflow: TextOverflow.ellipsis,
                      style: TextStyle(color: Colors.white,
                        fontSize: (12 * fs).clamp(9.0, 24.0), height: 1.4)),
                  ]),
                )),
              ])
            : Center(child: Padding(
                padding: EdgeInsets.all((20 * fs).clamp(10.0, 40.0)),
                child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [
                  Icon(Icons.campaign, color: Colors.amber, size: (32 * fs).clamp(22.0, 64.0)),
                  SizedBox(height: 8 * fs),
                  FittedBox(child: Text(p._t('PENGUMUMAN','ANNOUNCEMENT','الإعلانات'),
                    style: const TextStyle(color: Colors.amber, fontSize: 16, letterSpacing: 2))),
                  SizedBox(height: 4 * fs),
                  FittedBox(child: Text(p.namaIbadah,
                    style: const TextStyle(color: Colors.white38, fontSize: 11))),
                  SizedBox(height: 14 * fs),
                  Text(um['teks'] ?? '', textAlign: TextAlign.center,
                    maxLines: 10, overflow: TextOverflow.ellipsis,
                    style: TextStyle(color: Colors.white,
                      fontSize: (14 * fs).clamp(10.0, 28.0), height: 1.4)),
                ]),
              )),
        ));

      case 'pengurus':
        // Helper kartu pengurus
        Widget kartu(Map<String, dynamic> pg, double w, double r, {bool isKetua = false}) {
          final foto = pg['foto'] ?? '';
          return SizedBox(
            width: w,
            child: Column(mainAxisSize: MainAxisSize.min, children: [
              CircleAvatar(
                radius: r,
                backgroundColor: isKetua ? Colors.greenAccent : kPrimary,
                backgroundImage: foto.isNotEmpty ? MemoryImage(base64Decode(foto)) : null,
                child: foto.isEmpty ? Text((pg['nama'] ?? 'X')[0].toUpperCase(),
                  style: TextStyle(color: Colors.white, fontSize: r * 0.7, fontWeight: FontWeight.bold)) : null,
              ),
              SizedBox(height: 4 * fs),
              Text(pg['nama'] ?? '-',
                textAlign: TextAlign.center, maxLines: 2, overflow: TextOverflow.ellipsis,
                style: TextStyle(color: Colors.white, fontSize: (11 * fs).clamp(8.0, 22.0), fontWeight: FontWeight.bold)),
              Text(p.terjemahJabatan(pg['jabatan'] ?? ''),
                textAlign: TextAlign.center, maxLines: 2, overflow: TextOverflow.ellipsis,
                style: TextStyle(color: isKetua ? Colors.greenAccent : Colors.white60, fontSize: (9 * fs).clamp(7.0, 18.0))),
            ]),
          );
        }

        return Container(
          color: const Color(0xFF0A1A0A),
          padding: EdgeInsets.all(12 * fs),
          child: Column(children: [
            // Header
            Icon(Icons.people, color: Colors.greenAccent, size: (28 * fs).clamp(20.0, 60.0)),
            SizedBox(height: 2 * fs),
            FittedBox(child: Text(p._t('STRUKTUR PENGURUS','COMMITTEE STRUCTURE','هيكل اللجنة'),
              style: TextStyle(color: Colors.greenAccent, fontSize: (14 * fs).clamp(11.0, 32.0), letterSpacing: 2))),
            SizedBox(height: 2 * fs),
            Text(p.namaIbadah, overflow: TextOverflow.ellipsis,
              style: TextStyle(color: Colors.white38, fontSize: (9 * fs).clamp(8.0, 18.0))),
            SizedBox(height: 8 * fs),

            // Bagan
            Expanded(
              child: p.pengurus.isEmpty
                ? Center(child: Text(p._t('Belum ada pengurus','No committee yet','لا هيئة بعد'),
                    style: TextStyle(color: Colors.grey, fontSize: (13 * fs).clamp(10.0, 28.0))))
                : LayoutBuilder(builder: (ctx, cst) {
                    final ketua = p.pengurus[0];
                    final anggota = p.pengurus.length > 1 ? p.pengurus.sublist(1) : <Map<String, dynamic>>[];
                    final ketuaW = (cst.maxWidth * 0.4).clamp(100.0, 220.0);
                    final ketuaR = (ketuaW * 0.22).clamp(22.0, 55.0);
                    final colW   = (cst.maxWidth * 0.42).clamp(80.0, 200.0);
                    final cardR  = (colW * 0.18).clamp(18.0, 44.0);
                    final lineC  = Colors.white24;

                    // Bagi anggota jadi 2 kolom: kiri (genap) dan kanan (ganjil)
                    final kiri   = <Map<String, dynamic>>[];
                    final kanan  = <Map<String, dynamic>>[];
                    for (int i = 0; i < anggota.length; i++) {
                      if (i % 2 == 0) kiri.add(anggota[i]);
                      else kanan.add(anggota[i]);
                    }

                    return SingleChildScrollView(
                      child: Column(children: [
                        // Ketua di tengah atas
                        Center(child: kartu(ketua, ketuaW, ketuaR, isKetua: true)),

                        if (anggota.isNotEmpty) ...[
                          // Garis vertikal ke bawah
                          Container(width: 2, height: 16 * fs, color: lineC),
                          // Garis horizontal
                          if (anggota.length > 1)
                            Container(
                              width: cst.maxWidth * 0.7,
                              height: 2,
                              color: lineC,
                            ),
                          SizedBox(height: 4 * fs),
                          // 2 kolom anggota
                          Row(
                            crossAxisAlignment: CrossAxisAlignment.start,
                            mainAxisAlignment: MainAxisAlignment.center,
                            children: [
                              // Kolom kiri
                              if (kiri.isNotEmpty)
                                Column(children: kiri.map((pg) => Padding(
                                  padding: EdgeInsets.only(bottom: 8 * fs),
                                  child: kartu(pg, colW, cardR),
                                )).toList()),

                              SizedBox(width: 16 * fs),

                              // Kolom kanan
                              if (kanan.isNotEmpty)
                                Column(children: kanan.map((pg) => Padding(
                                  padding: EdgeInsets.only(bottom: 8 * fs),
                                  child: kartu(pg, colW, cardR),
                                )).toList()),
                            ],
                          ),
                        ],
                      ]),
                    );
                  }),
            ),
          ]),
        );

      case 'kas':
        final fmt = NumberFormat('#,##0', 'id');
        // Ambil 5 transaksi terbaru
        final recentTrx = p.transaksi.reversed.take(5).toList();
        return Container(
          color: const Color(0xFF0A0A1A),
          padding: EdgeInsets.all((12 * fs).clamp(8.0, 28.0)),
          child: Column(children: [
            // Header
            Icon(Icons.account_balance_wallet, color: Colors.blueAccent, size: (24 * fs).clamp(18.0, 56.0)),
            SizedBox(height: 4 * fs),
            FittedBox(child: Text(p._t('LAPORAN KEUANGAN','FINANCIAL REPORT','التقرير المالي'),
              style: TextStyle(color: Colors.blueAccent, fontSize: (11 * fs).clamp(9.0, 24.0), letterSpacing: 2))),
            SizedBox(height: 2 * fs),
            FittedBox(child: Text(p.namaIbadah, overflow: TextOverflow.ellipsis,
              style: TextStyle(color: Colors.white38, fontSize: (8 * fs).clamp(7.0, 18.0)))),
            SizedBox(height: 8 * fs),
            // Ringkasan saldo
            if (p.saldoAwal > 0)
              _slideKasItem(p._t('SALDO AWAL','OPENING BALANCE','الرصيد الأولي'), "Rp ${fmt.format(p.saldoAwal)}", Colors.white70, fs: fs),
            if (p.saldoAwal > 0) SizedBox(height: 4 * fs),
            _slideKasItem(p._t('PEMASUKAN','INCOME','الدخل'), "Rp ${fmt.format(p.totalMasukAll)}", Colors.lightGreenAccent, fs: fs),
            SizedBox(height: 4 * fs),
            _slideKasItem(p._t('PENGELUARAN','EXPENSE','المصروف'), "Rp ${fmt.format(p.totalKeluarAll)}", Colors.redAccent, fs: fs),
            SizedBox(height: 4 * fs),
            _slideKasItem(p._t('SALDO AKHIR','FINAL BALANCE','الرصيد النهائي'), "Rp ${fmt.format(p.saldo)}", Colors.greenAccent, fs: fs),
            // Divider
            if (recentTrx.isNotEmpty) ...[
              SizedBox(height: 8 * fs),
              Row(children: [
                Expanded(child: Divider(color: Colors.white24)),
                Padding(
                  padding: EdgeInsets.symmetric(horizontal: 8 * fs),
                  child: Text(p._t('TRANSAKSI TERBARU','RECENT TRANSACTIONS','آخر المعاملات'),
                    style: TextStyle(color: Colors.white38, fontSize: (7 * fs).clamp(7.0, 14.0), letterSpacing: 1)),
                ),
                Expanded(child: Divider(color: Colors.white24)),
              ]),
              SizedBox(height: 6 * fs),
              // List transaksi terbaru
              Expanded(
                child: ListView.builder(
                  shrinkWrap: false,
                  itemCount: recentTrx.length,
                  itemBuilder: (_, i) {
                    final t = recentTrx[i];
                    final isMasuk = t['jenis'] == 'masuk';
                    final jml = (t['jumlah'] as double);
                    return Container(
                      margin: EdgeInsets.only(bottom: (4 * fs).clamp(3.0, 10.0)),
                      padding: EdgeInsets.symmetric(
                        horizontal: (8 * fs).clamp(6.0, 18.0),
                        vertical: (4 * fs).clamp(3.0, 10.0)),
                      decoration: BoxDecoration(
                        color: isMasuk ? Colors.green.withValues(alpha: 0.1) : Colors.red.withValues(alpha: 0.1),
                        borderRadius: BorderRadius.circular(6),
                        border: Border.all(color: isMasuk ? Colors.green.withValues(alpha: 0.3) : Colors.red.withValues(alpha: 0.3)),
                      ),
                      child: Row(children: [
                        Icon(isMasuk ? Icons.arrow_downward : Icons.arrow_upward,
                          color: isMasuk ? Colors.greenAccent : Colors.redAccent,
                          size: (10 * fs).clamp(10.0, 22.0)),
                        SizedBox(width: 6 * fs),
                        Expanded(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                          Text(t['keterangan'] ?? '-',
                            style: TextStyle(color: Colors.white, fontSize: (9 * fs).clamp(8.0, 18.0), fontWeight: FontWeight.w500),
                            overflow: TextOverflow.ellipsis),
                          Text(p.formatTgl(t['tgl'] ?? ''),
                            style: TextStyle(color: Colors.white38, fontSize: (7 * fs).clamp(7.0, 14.0))),
                        ])),
                        SizedBox(width: 6 * fs),
                        Text("${isMasuk ? '+' : '-'}Rp ${fmt.format(jml)}",
                          style: TextStyle(
                            color: isMasuk ? Colors.greenAccent : Colors.redAccent,
                            fontSize: (9 * fs).clamp(8.0, 18.0),
                            fontWeight: FontWeight.bold)),
                      ]),
                    );
                  },
                ),
              ),
            ],
          ]),
        );

      case 'qris':
        return Container(
          decoration: const BoxDecoration(
            gradient: LinearGradient(
              begin: Alignment.topLeft,
              end: Alignment.bottomRight,
              colors: [Color(0xFF1b5e20), Color(0xFF2e7d32), Color(0xFF388e3c)],
            ),
          ),
          child: Consumer<AppProvider>(builder: (_, pp, __) => Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              // Header
              Row(mainAxisAlignment: MainAxisAlignment.center, children: [
                Icon(Icons.volunteer_activism, color: Colors.white70, size: (14 * fs).clamp(12.0, 32.0)),
                SizedBox(width: 6 * fs),
                Text(pp._t('DONASI & INFAQ','DONATION & CHARITY','التبرع والصدقة'), style: TextStyle(color: Colors.white70, fontSize: (11 * fs).clamp(9.0, 24.0), letterSpacing: 2)),
              ]),
              SizedBox(height: 8 * fs),
              // Nama masjid
              Text(pp.namaIbadah,
                style: TextStyle(color: Colors.white, fontSize: (13 * fs).clamp(10.0, 28.0), fontWeight: FontWeight.bold),
                textAlign: TextAlign.center, overflow: TextOverflow.ellipsis),
              SizedBox(height: 10 * fs),
              // Foto QRIS
              if (pp.fotoQris.isNotEmpty)
                Container(
                  width: (160 * fs).clamp(120.0, 400.0), height: (160 * fs).clamp(120.0, 400.0),
                  decoration: BoxDecoration(
                    color: Colors.white,
                    borderRadius: BorderRadius.circular(16),
                    boxShadow: [BoxShadow(color: Colors.black38, blurRadius: 12, spreadRadius: 2)],
                  ),
                  padding: const EdgeInsets.all(8),
                  child: ClipRRect(
                    borderRadius: BorderRadius.circular(10),
                    child: Image.memory(base64Decode(pp.fotoQris), fit: BoxFit.contain),
                  ),
                )
              else
                Container(
                  width: (160 * fs).clamp(120.0, 400.0), height: (160 * fs).clamp(120.0, 400.0),
                  decoration: BoxDecoration(
                    color: Colors.white24,
                    borderRadius: BorderRadius.circular(16),
                    border: Border.all(color: Colors.white38, width: 2),
                  ),
                  child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [
                    const Icon(Icons.qr_code, color: Colors.white54, size: 80),
                    const SizedBox(height: 8),
                    Text(pp._t('QRIS belum diupload','QRIS not uploaded yet','لم يتم رفع QRIS بعد'), style: const TextStyle(color: Colors.white54, fontSize: 12)),
                  ]),
                ),
              const SizedBox(height: 16),
              // Info rekening
              Container(
                margin: const EdgeInsets.symmetric(horizontal: 24),
                padding: const EdgeInsets.symmetric(horizontal: 20, vertical: 12),
                decoration: BoxDecoration(
                  color: Colors.black26,
                  borderRadius: BorderRadius.circular(12),
                ),
                child: Column(children: [
                  if (pp.namaBank.isNotEmpty) ...[
                    Text(pp.namaBank.toUpperCase(),
                      style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold, fontSize: 15, letterSpacing: 1)),
                    const SizedBox(height: 4),
                  ],
                  if (pp.noRekening.isNotEmpty)
                    Text(pp.noRekening,
                      style: const TextStyle(color: Colors.amber, fontWeight: FontWeight.bold, fontSize: 18, letterSpacing: 2)),
                  if (pp.namaPemilik.isNotEmpty) ...[
                    const SizedBox(height: 4),
                    Text('${pp._t('a.n.','a/n','بـ')} ${pp.namaPemilik}',
                      style: const TextStyle(color: Colors.white70, fontSize: 12)),
                  ],
                ]),
              ),
              const SizedBox(height: 12),
              Text(pp._t('Scan QR atau transfer ke rekening di atas','Scan QR or transfer to the account above','امسح QR أو حوّل إلى الحساب أعلاه'),
                style: const TextStyle(color: Colors.white54, fontSize: 11),
                textAlign: TextAlign.center),
            ],
          )),
        );

      case 'acara':
        final acara = slide['data'] as Map<String, dynamic>;
        final namaAcara = acara['namaAcara'] ?? '';
        final tanggalRaw = acara['tanggal'] ?? '';
        final tanggal = tanggalRaw.isNotEmpty ? p.formatTgl(tanggalRaw) : '';
        final sambutan  = acara['sambutan']  ?? '';
        final tamuList  = (acara['tamu'] as List<dynamic>?) ?? [];
        return Container(
          decoration: const BoxDecoration(
            gradient: LinearGradient(
              begin: Alignment.topCenter,
              end: Alignment.bottomCenter,
              colors: [Color(0xFF1a0a2e), Color(0xFF16213e), Color(0xFF0f3460)],
            ),
          ),
          child: Column(mainAxisSize: MainAxisSize.max, children: [
            const SizedBox(height: 20),
            // Header: logo masjid + nama masjid
            Consumer<AppProvider>(builder: (_, pp, __) => Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                if (pp.fotoMasjid.isNotEmpty)
                  CircleAvatar(radius: 28, backgroundImage: MemoryImage(base64Decode(pp.fotoMasjid)))
                else
                  const CircleAvatar(radius: 28, backgroundColor: kPrimary,
                    child: Icon(Icons.mosque, color: Colors.white, size: 28)),
                const SizedBox(width: 12),
                Flexible(child: Text(pp.namaIbadah,
                  style: const TextStyle(color: Colors.white70, fontSize: 14, fontWeight: FontWeight.w500),
                  overflow: TextOverflow.ellipsis)),
              ],
            )),
            SizedBox(height: 8 * fs),
            // Sambutan
            if (sambutan.isNotEmpty)
              Padding(
                padding: EdgeInsets.symmetric(horizontal: (16 * fs).clamp(8.0, 36.0)),
                child: Text(sambutan,
                  style: TextStyle(color: Colors.amber, fontSize: (11 * fs).clamp(9.0, 24.0), fontStyle: FontStyle.italic),
                  textAlign: TextAlign.center, maxLines: 2, overflow: TextOverflow.ellipsis),
              ),
            SizedBox(height: 6 * fs),
            // Nama acara
            Padding(
              padding: EdgeInsets.symmetric(horizontal: (12 * fs).clamp(8.0, 28.0)),
              child: Text(namaAcara,
                style: TextStyle(color: Colors.white, fontSize: (16 * fs).clamp(12.0, 40.0), fontWeight: FontWeight.bold, letterSpacing: 1),
                textAlign: TextAlign.center, maxLines: 2, overflow: TextOverflow.ellipsis),
            ),
            // Tanggal
            if (tanggal.isNotEmpty) ...[
              SizedBox(height: 4 * fs),
              Text(tanggal, style: TextStyle(color: Colors.white54, fontSize: (10 * fs).clamp(8.0, 22.0))),
            ],
            const SizedBox(height: 16),
            // Divider
            Padding(
              padding: const EdgeInsets.symmetric(horizontal: 24),
              child: Row(children: [
                Expanded(child: Divider(color: Colors.amber.withValues(alpha: 0.5), thickness: 1)),
                Padding(
                  padding: const EdgeInsets.symmetric(horizontal: 8),
                  child: Consumer<AppProvider>(builder: (_, pp2, __) => Text(
                    pp2._t('TAMU UNDANGAN','INVITED GUESTS','الضيوف المدعوون'),
                    style: TextStyle(color: Colors.amber.withValues(alpha: 0.8), fontSize: 10, letterSpacing: 2))),
                ),
                Expanded(child: Divider(color: Colors.amber.withValues(alpha: 0.5), thickness: 1)),
              ]),
            ),
            const SizedBox(height: 12),
            // Tamu undangan horizontal scroll
            Expanded(
              child: tamuList.isEmpty
                ? Center(child: Consumer<AppProvider>(
                    builder: (_, pp2, __) => Text(
                      pp2._t('Belum ada tamu undangan','No invited guests yet','لا ضيوف مدعوون بعد'),
                      style: const TextStyle(color: Colors.white38))))
                : ListView.builder(
                    scrollDirection: Axis.horizontal,
                    padding: const EdgeInsets.symmetric(horizontal: 8),
                    itemCount: tamuList.length,
                    itemBuilder: (_, i) {
                      final t = tamuList[i] as Map<String, dynamic>;
                      final foto = (t['foto'] ?? '') as String;
                      return Container(
                        width: 90,
                        margin: const EdgeInsets.symmetric(horizontal: 4),
                        child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [
                          CircleAvatar(
                            radius: 36,
                            backgroundColor: Colors.amber.withValues(alpha: 0.2),
                            backgroundImage: foto.isNotEmpty ? MemoryImage(base64Decode(foto)) : null,
                            child: foto.isEmpty
                              ? Text((t['nama'] ?? 'T')[0].toUpperCase(),
                                  style: const TextStyle(color: Colors.amber, fontSize: 24, fontWeight: FontWeight.bold))
                              : null,
                          ),
                          const SizedBox(height: 8),
                          Text(t['nama'] ?? '-',
                            style: const TextStyle(color: Colors.white, fontSize: 12, fontWeight: FontWeight.bold),
                            textAlign: TextAlign.center, maxLines: 2, overflow: TextOverflow.ellipsis),
                          const SizedBox(height: 2),
                          Text(t['jabatan'] ?? '',
                            style: TextStyle(color: Colors.amber.withValues(alpha: 0.8), fontSize: 10),
                            textAlign: TextAlign.center, maxLines: 2, overflow: TextOverflow.ellipsis),
                        ]),
                      );
                    },
                  ),
            ),
            const SizedBox(height: 16),
          ]),
        );

      case 'galeri':
        final g = slide['data'] as Map<String, dynamic>;
        return _buildSlide(g, isPaused: _isPaused, provider: p);

      case 'donatur':
        final fmt2 = NumberFormat('#,##0', 'id');
        final topDonatur = p.donaturList.length > 5
            ? p.donaturList.sublist(p.donaturList.length - 5)
            : p.donaturList;
        return Container(
          color: const Color(0xFF0A1628),
          padding: EdgeInsets.all((12 * fs).clamp(8.0, 28.0)),
          child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [
            Icon(Icons.volunteer_activism, color: Colors.lightBlueAccent, size: (28 * fs).clamp(20.0, 70.0)),
            SizedBox(height: 6 * fs),
            Text(p._t('DONATUR','DONORS','المتبرعون'), style: TextStyle(color: Colors.lightBlueAccent, fontSize: (13 * fs).clamp(10.0, 30.0), letterSpacing: 2)),
            SizedBox(height: 4 * fs),
            Text(p.namaIbadah, overflow: TextOverflow.ellipsis, style: TextStyle(color: Colors.white38, fontSize: (10 * fs).clamp(8.0, 22.0))),
            SizedBox(height: 12 * fs),
            Column(children: [
                ...topDonatur.reversed.map((d) {
                  final jml = (d['jumlah'] as double);
                  final mtu = d['matauang'] as String? ?? 'IDR';
                  final jmlStr = mtu == 'IDR'
                      ? 'Rp ${fmt2.format(jml)}'
                      : '$mtu ${jml.toStringAsFixed(0)}';
                  return Container(
                    margin: const EdgeInsets.symmetric(vertical: 4),
                    padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 10),
                    decoration: BoxDecoration(
                      color: Colors.white10,
                      borderRadius: BorderRadius.circular(8),
                    ),
                    child: Row(
                      mainAxisAlignment: MainAxisAlignment.spaceBetween,
                      children: [
                        Flexible(child: Row(children: [
                          Icon(Icons.favorite, color: Colors.pinkAccent, size: (11 * fs).clamp(8.0, 22.0)),
                          SizedBox(width: 6 * fs),
                          Flexible(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                            Text(d['nama'] ?? '-', overflow: TextOverflow.ellipsis,
                              style: TextStyle(color: Colors.white, fontSize: (12 * fs).clamp(9.0, 26.0), fontWeight: FontWeight.bold)),
                            Text(_terjemahKategoriStatic(d['kategori'] ?? '', p.bahasa), overflow: TextOverflow.ellipsis,
                              style: TextStyle(color: Colors.white54, fontSize: (9 * fs).clamp(7.0, 18.0))),
                            if ((d['tgl'] ?? '').isNotEmpty)
                              Text(p.formatTgl(d['tgl'] ?? ''), overflow: TextOverflow.ellipsis,
                                style: TextStyle(color: Colors.white38, fontSize: (8 * fs).clamp(7.0, 16.0))),
                          ])),
                        ])),
                        Text(jmlStr,
                          style: TextStyle(color: Colors.lightBlueAccent, fontSize: (11 * fs).clamp(9.0, 24.0), fontWeight: FontWeight.bold)),
                      ],
                    ),
                  );
                }),
                SizedBox(height: 6 * fs),
                Text('${p._t('Total','Total','الإجمالي')}: Rp ${fmt2.format(p.totalDonatur)}',
                  style: TextStyle(color: Colors.greenAccent, fontSize: (11 * fs).clamp(9.0, 24.0), fontWeight: FontWeight.bold)),
            ]),
          ]),
        );

      case 'inventaris':
        return Container(
          color: const Color(0xFF0A0A2A),
          padding: EdgeInsets.all((16 * fs).clamp(10.0, 36.0)),
          child: Column(children: [
            Icon(Icons.inventory_2, color: Colors.tealAccent, size: (28 * fs).clamp(20.0, 70.0)),
            SizedBox(height: 6 * fs),
            FittedBox(child: Text(p._t('INVENTARIS MASJID','MOSQUE INVENTORY','مخزون المسجد'),
              style: TextStyle(color: Colors.tealAccent, fontSize: (12 * fs).clamp(10.0, 28.0), letterSpacing: 2))),
            SizedBox(height: 4 * fs),
            FittedBox(child: Text(p.namaIbadah, overflow: TextOverflow.ellipsis,
              style: TextStyle(color: Colors.white38, fontSize: (9 * fs).clamp(8.0, 20.0)))),
            SizedBox(height: 10 * fs),
            Expanded(
              child: ListView.builder(
                itemCount: p.inventaris.length,
                itemBuilder: (_, i) {
                  final inv = p.inventaris[i];
                  final kondisi = inv['kondisi'] ?? '';
                  final kondisiColor = kondisi == 'Baik' ? Colors.greenAccent
                    : kondisi == 'Rusak' ? Colors.redAccent : Colors.orangeAccent;
                  return Container(
                    margin: EdgeInsets.only(bottom: (6 * fs).clamp(4.0, 14.0)),
                    padding: EdgeInsets.symmetric(
                      horizontal: (12 * fs).clamp(8.0, 28.0),
                      vertical: (8 * fs).clamp(5.0, 18.0)),
                    decoration: BoxDecoration(
                      color: Colors.white10,
                      borderRadius: BorderRadius.circular(8),
                      border: Border.all(color: kondisiColor.withValues(alpha: 0.3)),
                    ),
                    child: Row(children: [
                      Icon(Icons.check_box, color: kondisiColor, size: (14 * fs).clamp(12.0, 28.0)),
                      SizedBox(width: 8 * fs),
                      Expanded(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                        Text(inv['nama'] ?? '-',
                          style: TextStyle(color: Colors.white, fontSize: (11 * fs).clamp(9.0, 22.0), fontWeight: FontWeight.bold),
                          overflow: TextOverflow.ellipsis),
                        if ((inv['keterangan'] ?? '').isNotEmpty)
                          Text(inv['keterangan'],
                            style: TextStyle(color: Colors.white54, fontSize: (9 * fs).clamp(8.0, 18.0)),
                            overflow: TextOverflow.ellipsis),
                        if ((inv['tgl'] ?? '').isNotEmpty)
                          Text(p.formatTgl(inv['tgl'] ?? ''),
                            style: TextStyle(color: Colors.white38, fontSize: (8 * fs).clamp(7.0, 16.0))),
                      ])),
                      SizedBox(width: 8 * fs),
                      Column(crossAxisAlignment: CrossAxisAlignment.end, children: [
                        Text('${inv['jumlah']} ${inv['satuan'] ?? ''}',
                          style: TextStyle(color: Colors.tealAccent, fontSize: (11 * fs).clamp(9.0, 22.0), fontWeight: FontWeight.bold)),
                        Container(
                          padding: EdgeInsets.symmetric(horizontal: 6, vertical: 2),
                          decoration: BoxDecoration(
                            color: kondisiColor.withValues(alpha: 0.2),
                            borderRadius: BorderRadius.circular(4),
                          ),
                          child: Text(p.terjemahKondisi(kondisi),
                            style: TextStyle(color: kondisiColor, fontSize: (8 * fs).clamp(7.0, 16.0))),
                        ),
                      ]),
                    ]),
                  );
                },
              ),
            ),
          ]),
        );

      case 'countdown':
        return _buildSlideCountdown(fs, p);

      case 'imsak':
        final waktuSubuhSlide = p.waktuSubuh;
        final waktuMaghribSlide = p.waktuMaghrib;
        return ClipRect(child: Container(
          decoration: const BoxDecoration(
            gradient: LinearGradient(
              begin: Alignment.topCenter, end: Alignment.bottomCenter,
              colors: [Color(0xFF0D1B2A), Color(0xFF1B3A5C)],
            ),
          ),
          padding: EdgeInsets.all((16 * fs).clamp(10.0, 36.0)),
          child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [
            Text('☪️', style: TextStyle(fontSize: (32 * fs).clamp(24.0, 80.0))),
            SizedBox(height: 6 * fs),
            FittedBox(child: Text(
              p._t('JADWAL IMSAK RAMADHAN','RAMADAN IMSAK SCHEDULE','جدول إمساك رمضان') + ' ${p.tahunRamadhan}',
              style: TextStyle(color: Colors.amberAccent, fontSize: (11 * fs).clamp(9.0, 26.0), letterSpacing: 1),
              textAlign: TextAlign.center,
            )),
            SizedBox(height: 4 * fs),
            FittedBox(child: Text(p.namaIbadah,
              style: TextStyle(color: Colors.white38, fontSize: (9 * fs).clamp(7.0, 18.0)))),
            SizedBox(height: 16 * fs),
            _slideKasItem(p.lImsak, p.waktuImsak, Colors.amberAccent, fs: fs),
            SizedBox(height: 8 * fs),
            _slideKasItem(p._t('Subuh','Fajr','الفجر'), waktuSubuhSlide, Colors.lightBlueAccent, fs: fs),
            SizedBox(height: 8 * fs),
            _slideKasItem(p.lBerbuka, waktuMaghribSlide, Colors.orangeAccent, fs: fs),
            SizedBox(height: 16 * fs),
            Text(
              p._t('Imsak','Imsak','الإمساك') + ' ${p.menitSebelumSubuh} ' + p._t('menit sebelum Subuh','min before Fajr','دقيقة قبل الفجر'),
              style: TextStyle(color: Colors.white38, fontSize: (8 * fs).clamp(7.0, 16.0)),
              textAlign: TextAlign.center,
            ),
          ]),
        ));

      case 'jumat':
        // Ambil jadwal jumat terdekat/mendatang
        final now = DateTime.now();
        final jadwalMendatang = p.jadwalJumat.where((j) {
          try {
            final parts = (j['tanggal'] ?? '').split('/');
            if (parts.length != 3) return false;
            final tglJumat = DateTime(int.parse(parts[2]), int.parse(parts[1]), int.parse(parts[0]));
            return tglJumat.isAfter(now.subtract(const Duration(days: 1)));
          } catch (_) { return false; }
        }).toList();
        if (jadwalMendatang.isEmpty) return const SizedBox.shrink();
        final nextJumat = jadwalMendatang.first;
        return ClipRect(child: Container(
          color: const Color(0xFF0A1A0A),
          padding: EdgeInsets.all((16 * fs).clamp(10.0, 36.0)),
          child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [
            Icon(Icons.mosque, color: Colors.greenAccent, size: (32 * fs).clamp(24.0, 80.0)),
            SizedBox(height: 8 * fs),
            FittedBox(child: Text(
              p._t('JADWAL JUMAT','FRIDAY SCHEDULE','جدول الجمعة'),
              style: TextStyle(color: Colors.greenAccent, fontSize: (13 * fs).clamp(10.0, 30.0), letterSpacing: 2),
            )),
            SizedBox(height: 4 * fs),
            FittedBox(child: Text(p.formatTgl(nextJumat['tanggal'] ?? ''),
              style: TextStyle(color: Colors.white54, fontSize: (10 * fs).clamp(8.0, 22.0)))),
            SizedBox(height: 16 * fs),
            // Imam
            Container(
              width: double.infinity,
              padding: EdgeInsets.symmetric(horizontal: (16 * fs).clamp(8.0, 36.0), vertical: (10 * fs).clamp(6.0, 24.0)),
              decoration: BoxDecoration(color: Colors.white10, borderRadius: BorderRadius.circular(10)),
              child: Row(children: [
                Icon(Icons.person, color: Colors.greenAccent, size: (16 * fs).clamp(12.0, 36.0)),
                SizedBox(width: 8 * fs),
                Expanded(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                  Text(p.lImam, style: TextStyle(color: Colors.white38, fontSize: (9 * fs).clamp(8.0, 18.0))),
                  Text(nextJumat['imam'] ?? '-',
                    style: TextStyle(color: Colors.white, fontSize: (13 * fs).clamp(10.0, 28.0), fontWeight: FontWeight.bold),
                    overflow: TextOverflow.ellipsis),
                ])),
              ]),
            ),
            SizedBox(height: 8 * fs),
            // Khatib
            Container(
              width: double.infinity,
              padding: EdgeInsets.symmetric(horizontal: (16 * fs).clamp(8.0, 36.0), vertical: (10 * fs).clamp(6.0, 24.0)),
              decoration: BoxDecoration(color: Colors.white10, borderRadius: BorderRadius.circular(10)),
              child: Row(children: [
                Icon(Icons.mic, color: Colors.amber, size: (16 * fs).clamp(12.0, 36.0)),
                SizedBox(width: 8 * fs),
                Expanded(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                  Text(p.lKhatib, style: TextStyle(color: Colors.white38, fontSize: (9 * fs).clamp(8.0, 18.0))),
                  Text(nextJumat['khatib'] ?? '-',
                    style: TextStyle(color: Colors.white, fontSize: (13 * fs).clamp(10.0, 28.0), fontWeight: FontWeight.bold),
                    overflow: TextOverflow.ellipsis),
                ])),
              ]),
            ),
            if ((nextJumat['tema'] ?? '').isNotEmpty) ...[
              SizedBox(height: 8 * fs),
              Container(
                width: double.infinity,
                padding: EdgeInsets.symmetric(horizontal: (16 * fs).clamp(8.0, 36.0), vertical: (10 * fs).clamp(6.0, 24.0)),
                decoration: BoxDecoration(color: Colors.white10, borderRadius: BorderRadius.circular(10)),
                child: Row(children: [
                  Icon(Icons.book, color: Colors.lightBlueAccent, size: (16 * fs).clamp(12.0, 36.0)),
                  SizedBox(width: 8 * fs),
                  Expanded(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                    Text(p.lTemaKhutbah, style: TextStyle(color: Colors.white38, fontSize: (9 * fs).clamp(8.0, 18.0))),
                    Text(nextJumat['tema'] ?? '',
                      style: TextStyle(color: Colors.white70, fontSize: (11 * fs).clamp(9.0, 24.0)),
                      maxLines: 2, overflow: TextOverflow.ellipsis),
                  ])),
                ]),
              ),
            ],
            SizedBox(height: 8 * fs),
            Text(p.namaIbadah,
              style: TextStyle(color: Colors.white24, fontSize: (9 * fs).clamp(7.0, 18.0)),
              overflow: TextOverflow.ellipsis),
          ]),
        ));

      case 'ayat':
        return _buildSlideAyat(fs, p);

      default:
        return _buildSlideKosong(p);
    }
  }

  // ── DATA HARI BESAR ISLAM (perkiraan Masehi) ──
  static const _hariBesar = [
    {'nama': 'Tahun Baru Islam 1447 H', 'nama_en': 'Islamic New Year 1447 H',  'nama_ar': 'رأس السنة الهجرية 1447 هـ', 'emoji': '🌙', 'tanggal': '2025-06-27'},
    {'nama': 'Maulid Nabi SAW 1447 H',  'nama_en': 'Prophet Birthday 1447 H', 'nama_ar': 'المولد النبوي 1447 هـ',       'emoji': '🌟', 'tanggal': '2025-09-05'},
    {'nama': 'Isra Mi\'raj 1447 H',      'nama_en': 'Isra Miraj 1447 H',       'nama_ar': 'الإسراء والمعراج 1447 هـ',  'emoji': '🚀', 'tanggal': '2026-01-27'},
    {'nama': 'Ramadhan 1447 H',          'nama_en': 'Ramadan 1447 H',           'nama_ar': 'رمضان 1447 هـ',              'emoji': '🌙', 'tanggal': '2026-02-18'},
    {'nama': 'Nuzulul Qur\'an 1447 H',   'nama_en': 'Nuzulul Quran 1447 H',    'nama_ar': 'نزول القرآن 1447 هـ',       'emoji': '📖', 'tanggal': '2026-03-15'},
    {'nama': 'Idul Fitri 1447 H',        'nama_en': 'Eid al-Fitr 1447 H',       'nama_ar': 'عيد الفطر 1447 هـ',          'emoji': '🎉', 'tanggal': '2026-03-20'},
    {'nama': 'Idul Adha 1447 H',         'nama_en': 'Eid al-Adha 1447 H',       'nama_ar': 'عيد الأضحى 1447 هـ',         'emoji': '🐄', 'tanggal': '2026-05-27'},
    {'nama': 'Tahun Baru Islam 1448 H',  'nama_en': 'Islamic New Year 1448 H',  'nama_ar': 'رأس السنة الهجرية 1448 هـ', 'emoji': '🌙', 'tanggal': '2026-06-17'},
    {'nama': 'Maulid Nabi SAW 1448 H',   'nama_en': 'Prophet Birthday 1448 H', 'nama_ar': 'المولد النبوي 1448 هـ',       'emoji': '🌟', 'tanggal': '2026-08-25'},
    {'nama': 'Isra Mi\'raj 1448 H',      'nama_en': 'Isra Miraj 1448 H',       'nama_ar': 'الإسراء والمعراج 1448 هـ',  'emoji': '🚀', 'tanggal': '2027-01-17'},
    {'nama': 'Ramadhan 1448 H',          'nama_en': 'Ramadan 1448 H',           'nama_ar': 'رمضان 1448 هـ',              'emoji': '🌙', 'tanggal': '2027-02-08'},
    {'nama': 'Idul Fitri 1448 H',        'nama_en': 'Eid al-Fitr 1448 H',       'nama_ar': 'عيد الفطر 1448 هـ',          'emoji': '🎉', 'tanggal': '2027-03-10'},
    {'nama': 'Idul Adha 1448 H',         'nama_en': 'Eid al-Adha 1448 H',       'nama_ar': 'عيد الأضحى 1448 هـ',         'emoji': '🐄', 'tanggal': '2027-05-17'},
  ];

  Map<String, dynamic> _getHariBesarBerikutnya() {
    final now = DateTime.now();
    for (final hb in _hariBesar) {
      final tgl = DateTime.parse(hb['tanggal']!);
      if (!tgl.isBefore(now)) {
        final selisih = tgl.difference(now).inDays;
        return {'nama': hb['nama'], 'nama_en': hb['nama_en'], 'nama_ar': hb['nama_ar'], 'emoji': hb['emoji'], 'tgl': tgl, 'hari': selisih};
      }
    }
    return {'nama': 'Idul Fitri 1448 H', 'nama_en': 'Eid al-Fitr 1448 H', 'nama_ar': 'عيد الفطر 1448 هـ', 'emoji': '🎉',
      'tgl': DateTime(2027, 3, 10), 'hari': 0};
  }

  Widget _buildSlideCountdown(double fs, AppProvider p) {
    final hb = _getHariBesarBerikutnya();
    final hari = hb['hari'] as int;
    final tgl  = hb['tgl']  as DateTime;
    // Nama sesuai bahasa
    final nama = p.bahasa == 'en'
        ? (hb['nama_en'] ?? hb['nama']) as String
        : p.bahasa == 'ar'
            ? (hb['nama_ar'] ?? hb['nama']) as String
            : hb['nama'] as String;
    final emoji= hb['emoji']as String;
    String tglStr;
    final locale = p.bahasa == 'en' ? 'en' : p.bahasa == 'ar' ? 'ar' : 'id';
    try { tglStr = DateFormat('d MMMM yyyy', locale).format(tgl); }
    catch (_) { tglStr = DateFormat('d MMMM yyyy').format(tgl); }

    // Warna tema sesuai event
    Color bgTop, bgBot, accent;
    if (nama.contains('Ramadhan')) {
      bgTop = const Color(0xFF0D1B3E); bgBot = const Color(0xFF1A3A6E); accent = Colors.amberAccent;
    } else if (nama.contains('Idul Fitri')) {
      bgTop = const Color(0xFF0B3D2E); bgBot = const Color(0xFF1B6B4A); accent = Colors.greenAccent;
    } else if (nama.contains('Idul Adha')) {
      bgTop = const Color(0xFF3D1B0B); bgBot = const Color(0xFF6B3A1B); accent = Colors.orangeAccent;
    } else if (nama.contains('Maulid')) {
      bgTop = const Color(0xFF1B0B3D); bgBot = const Color(0xFF3A1B6B); accent = Colors.purpleAccent;
    } else {
      bgTop = const Color(0xFF0A1628); bgBot = const Color(0xFF1A2F50); accent = Colors.lightBlueAccent;
    }

    return Container(
      decoration: BoxDecoration(gradient: LinearGradient(
        colors: [bgTop, bgBot], begin: Alignment.topCenter, end: Alignment.bottomCenter)),
      child: LayoutBuilder(builder: (lbCtx, lbC) => Stack(children: [
        // Bintang dekoratif
        ...List.generate(12, (i) {
          final positions = [
            [0.05,0.08],[0.15,0.03],[0.25,0.12],[0.75,0.05],[0.85,0.10],[0.92,0.03],
            [0.08,0.85],[0.20,0.92],[0.78,0.88],[0.88,0.82],[0.95,0.90],[0.50,0.95],
          ];
          return Positioned(
            left: lbC.maxWidth  * positions[i][0],
            top:  lbC.maxHeight * positions[i][1],
            child: Text('✦', style: TextStyle(
              color: accent.withValues(alpha: 0.3 + (i % 3) * 0.1), fontSize: 10 + (i % 4) * 4.0)),
          );
        }),
        // Konten utama
        Center(child: SingleChildScrollView(child: Column(mainAxisAlignment: MainAxisAlignment.center, mainAxisSize: MainAxisSize.min, children: [
          Text(emoji, style: TextStyle(fontSize: (32 * fs).clamp(24.0, 90.0))),
          SizedBox(height: 6 * fs),
          Text(p._t('MENYAMBUT','WELCOMING','مرحباً بـ'), style: TextStyle(
            color: accent.withValues(alpha: 0.8), fontSize: (10 * fs).clamp(9.0, 24.0), letterSpacing: 2)),
          SizedBox(height: 4 * fs),
          Padding(
            padding: EdgeInsets.symmetric(horizontal: 16 * fs),
            child: FittedBox(fit: BoxFit.scaleDown, child: Text(nama.toUpperCase(), textAlign: TextAlign.center,
              style: TextStyle(color: Colors.white,
                fontSize: (16 * fs).clamp(12.0, 42.0), fontWeight: FontWeight.bold, letterSpacing: 1))),
          ),
          SizedBox(height: 12 * fs),
          // Kotak hari
          Container(
            padding: EdgeInsets.symmetric(horizontal: (20 * fs).clamp(12.0, 50.0), vertical: (10 * fs).clamp(6.0, 24.0)),
            decoration: BoxDecoration(
              color: Colors.white10,
              borderRadius: BorderRadius.circular(16),
              border: Border.all(color: accent.withValues(alpha: 0.5), width: 2)),
            child: Column(mainAxisSize: MainAxisSize.min, children: [
              Text(hari == 0 ? p._t('HARI INI!','TODAY!','اليوم!') : (p.bahasa == 'ar' ? hari.toString().characters.map((c) { final d = int.tryParse(c); return d != null ? ['٠','١','٢','٣','٤','٥','٦','٧','٨','٩'][d] : c; }).join() : '$hari'),
                style: TextStyle(color: accent,
                  fontSize: hari == 0 ? (32 * fs).clamp(24.0, 80.0) : (44 * fs).clamp(32.0, 110.0),
                  fontWeight: FontWeight.bold, fontFamily: 'monospace')),
              if (hari > 0)
                Text(p._t('HARI LAGI','DAYS LEFT','يوم متبقٍ'), style: TextStyle(
                  color: accent.withValues(alpha: 0.9), fontSize: (11 * fs).clamp(9.0, 26.0), letterSpacing: 2)),
            ]),
          ),
          SizedBox(height: 8 * fs),
          Text(tglStr, style: TextStyle(
            color: Colors.white60, fontSize: (10 * fs).clamp(8.0, 22.0), letterSpacing: 1)),
        ]))),
      ])),
    );
  }

  // ── DATA AYAT & HADITS HARIAN ──
  static const _ayatHadits = [
    {
      'arab': 'إِنَّمَا الْأَعْمَالُ بِالنِّيَّاتِ',
      'terjemahan': 'Sesungguhnya setiap amalan bergantung pada niatnya.',
      'terjemahan_en': 'Indeed, actions are judged by intentions.',
      'terjemahan_ar': 'إنما الأعمال بالنيات.',
      'sumber': 'HR. Bukhari & Muslim',
      'type': 'hadits',
    },
    {
      'arab': 'وَمَن يَتَّقِ اللَّهَ يَجْعَل لَّهُ مَخْرَجًا',
      'terjemahan': 'Barangsiapa bertakwa kepada Allah, niscaya Dia akan membukakan jalan keluar baginya.',
      'terjemahan_en': 'Whoever fears Allah, He will make a way out for him.',
      'terjemahan_ar': 'ومن يتق الله يجعل له مخرجاً.',
      'sumber': 'QS. At-Talaq: 2',
      'type': 'ayat',
    },
    {
      'arab': 'الْمُسْلِمُ مَنْ سَلِمَ الْمُسْلِمُونَ مِنْ لِسَانِهِ وَيَدِهِ',
      'terjemahan': 'Muslim sejati adalah orang yang kaum muslimin lainnya selamat dari lisan dan tangannya.',
      'terjemahan_en': 'A true Muslim is one from whose tongue and hand other Muslims are safe.',
      'terjemahan_ar': 'المسلم من سلم المسلمون من لسانه ويده.',
      'sumber': 'HR. Bukhari',
      'type': 'hadits',
    },
    {
      'arab': 'وَأَقِيمُوا الصَّلَاةَ وَآتُوا الزَّكَاةَ',
      'terjemahan': 'Dirikanlah shalat dan tunaikanlah zakat.',
      'terjemahan_en': 'Establish prayer and give zakah.',
      'terjemahan_ar': 'وأقيموا الصلاة وآتوا الزكاة.',
      'sumber': 'QS. Al-Baqarah: 43',
      'type': 'ayat',
    },
    {
      'arab': 'خَيْرُ النَّاسِ أَنْفَعُهُمْ لِلنَّاسِ',
      'terjemahan': 'Sebaik-baik manusia adalah yang paling bermanfaat bagi orang lain.',
      'terjemahan_en': 'The best of people are those most beneficial to others.',
      'terjemahan_ar': 'خير الناس أنفعهم للناس.',
      'sumber': 'HR. Ahmad',
      'type': 'hadits',
    },
    {
      'arab': 'إِنَّ اللَّهَ مَعَ الصَّابِرِينَ',
      'terjemahan': 'Sesungguhnya Allah bersama orang-orang yang sabar.',
      'terjemahan_en': 'Indeed, Allah is with the patient.',
      'terjemahan_ar': 'إن الله مع الصابرين.',
      'sumber': 'QS. Al-Baqarah: 153',
      'type': 'ayat',
    },
    {
      'arab': 'طَلَبُ الْعِلْمِ فَرِيضَةٌ عَلَى كُلِّ مُسْلِمٍ',
      'terjemahan': 'Menuntut ilmu adalah kewajiban bagi setiap muslim.',
      'terjemahan_en': 'Seeking knowledge is an obligation upon every Muslim.',
      'terjemahan_ar': 'طلب العلم فريضة على كل مسلم.',
      'sumber': 'HR. Ibnu Majah',
      'type': 'hadits',
    },
    {
      'arab': 'وَتَعَاوَنُوا عَلَى الْبِرِّ وَالتَّقْوَىٰ',
      'terjemahan': 'Dan tolong-menolonglah kamu dalam kebaikan dan ketakwaan.',
      'terjemahan_en': 'Cooperate in righteousness and piety.',
      'terjemahan_ar': 'وتعاونوا على البر والتقوى.',
      'sumber': 'QS. Al-Maidah: 2',
      'type': 'ayat',
    },
    {
      'arab': 'الدُّعَاءُ مُخُّ الْعِبَادَةِ',
      'terjemahan': 'Doa adalah inti dari ibadah.',
      'terjemahan_en': 'Supplication is the essence of worship.',
      'terjemahan_ar': 'الدعاء مخ العبادة.',
      'sumber': 'HR. Tirmidzi',
      'type': 'hadits',
    },
    {
      'arab': 'وَإِذَا سَأَلَكَ عِبَادِي عَنِّي فَإِنِّي قَرِيبٌ',
      'terjemahan': 'Dan apabila hamba-hamba-Ku bertanya kepadamu tentang Aku, maka sesungguhnya Aku dekat.',
      'terjemahan_en': 'When My servants ask about Me, I am near.',
      'terjemahan_ar': 'وإذا سألك عبادي عني فإني قريب.',
      'sumber': 'QS. Al-Baqarah: 186',
      'type': 'ayat',
    },
    {
      'arab': 'مَنْ صَمَتَ نَجَا',
      'terjemahan': 'Barangsiapa diam, ia selamat.',
      'terjemahan_en': 'Whoever keeps silent is saved.',
      'terjemahan_ar': 'من صمت نجا.',
      'sumber': 'HR. Tirmidzi',
      'type': 'hadits',
    },
    {
      'arab': 'وَاللَّهُ يُحِبُّ الْمُحْسِنِينَ',
      'terjemahan': 'Dan Allah mencintai orang-orang yang berbuat kebaikan.',
      'terjemahan_en': 'Allah loves those who do good.',
      'terjemahan_ar': 'والله يحب المحسنين.',
      'sumber': 'QS. Ali Imran: 134',
      'type': 'ayat',
    },
    {
      'arab': 'تَبَسُّمُكَ فِي وَجْهِ أَخِيكَ صَدَقَةٌ',
      'terjemahan': 'Senyummu di hadapan saudaramu adalah sedekah.',
      'terjemahan_en': 'Your smile at your brother is charity.',
      'terjemahan_ar': 'تبسمك في وجه أخيك صدقة.',
      'sumber': 'HR. Tirmidzi',
      'type': 'hadits',
    },
    {
      'arab': 'إِنَّ مَعَ الْعُسْرِ يُسْرًا',
      'terjemahan': 'Sesungguhnya bersama kesulitan ada kemudahan.',
      'terjemahan_en': 'Indeed, with hardship comes ease.',
      'terjemahan_ar': 'إن مع العسر يسراً.',
      'sumber': 'QS. Al-Insyirah: 6',
      'type': 'ayat',
    },
    {
      'arab': 'مَنْ أَحَبَّ لِقَاءَ اللَّهِ أَحَبَّ اللَّهُ لِقَاءَهُ',
      'terjemahan': 'Barangsiapa yang senang bertemu Allah, Allah pun senang bertemu dengannya.',
      'terjemahan_en': 'Whoever loves to meet Allah, Allah loves to meet him.',
      'terjemahan_ar': 'من أحب لقاء الله أحب الله لقاءه.',
      'sumber': 'HR. Bukhari',
      'type': 'hadits',
    },
    {
      'arab': 'وَهُوَ مَعَكُمْ أَيْنَ مَا كُنتُمْ',
      'terjemahan': 'Dan Dia bersama kamu di mana saja kamu berada.',
      'terjemahan_en': 'He is with you wherever you are.',
      'terjemahan_ar': 'وهو معكم أينما كنتم.',
      'sumber': 'QS. Al-Hadid: 4',
      'type': 'ayat',
    },
    {
      'arab': 'لَا تَحْزَنْ إِنَّ اللَّهَ مَعَنَا',
      'terjemahan': 'Janganlah engkau bersedih, sesungguhnya Allah bersama kita.',
      'terjemahan_en': 'Do not grieve, indeed Allah is with us.',
      'terjemahan_ar': 'لا تحزن إن الله معنا.',
      'sumber': 'QS. At-Taubah: 40',
      'type': 'ayat',
    },
    {
      'arab': 'حُبُّ الدُّنْيَا رَأْسُ كُلِّ خَطِيئَةٍ',
      'terjemahan': 'Cinta dunia adalah pangkal segala kesalahan.',
      'terjemahan_en': 'Love of the world is the root of all evil.',
      'terjemahan_ar': 'حب الدنيا رأس كل خطيئة.',
      'sumber': 'HR. Baihaqi',
      'type': 'hadits',
    },
    {
      'arab': 'وَقُل رَّبِّ زِدْنِي عِلْمًا',
      'terjemahan': 'Dan katakanlah: Ya Tuhanku, tambahkanlah ilmuku.',
      'terjemahan_en': 'Say: My Lord, increase me in knowledge.',
      'terjemahan_ar': 'وقل رب زدني علماً.',
      'sumber': 'QS. Thaha: 114',
      'type': 'ayat',
    },
    {
      'arab': 'الصَّبْرُ نِصْفُ الْإِيمَانِ',
      'terjemahan': 'Sabar adalah separuh dari iman.',
      'terjemahan_en': 'Patience is half of faith.',
      'terjemahan_ar': 'الصبر نصف الإيمان.',
      'sumber': 'HR. Baihaqi',
      'type': 'hadits',
    },
    {
      'arab': 'فَاذْكُرُونِي أَذْكُرْكُمْ',
      'terjemahan': 'Maka ingatlah Aku, niscaya Aku akan mengingatmu.',
      'terjemahan_en': 'Remember Me, and I will remember you.',
      'terjemahan_ar': 'فاذكروني أذكركم.',
      'sumber': 'QS. Al-Baqarah: 152',
      'type': 'ayat',
    },
    {
      'arab': 'مَنْ تَوَاضَعَ لِلَّهِ رَفَعَهُ اللَّهُ',
      'terjemahan': 'Barangsiapa merendah diri karena Allah, maka Allah akan mengangkat derajatnya.',
      'terjemahan_en': 'Whoever humbles himself for Allah, Allah will raise his rank.',
      'terjemahan_ar': 'من تواضع لله رفعه الله.',
      'sumber': 'HR. Muslim',
      'type': 'hadits',
    },
    {
      'arab': 'إِنَّ اللَّهَ لَا يُضِيعُ أَجْرَ الْمُحْسِنِينَ',
      'terjemahan': 'Sesungguhnya Allah tidak menyia-nyiakan pahala orang-orang yang berbuat baik.',
      'terjemahan_en': 'Indeed, Allah does not waste the reward of those who do good.',
      'terjemahan_ar': 'إن الله لا يضيع أجر المحسنين.',
      'sumber': 'QS. At-Taubah: 120',
      'type': 'ayat',
    },
    {
      'arab': 'اتَّقِ اللَّهَ حَيْثُمَا كُنْتَ',
      'terjemahan': 'Bertakwalah kepada Allah di mana pun kamu berada.',
      'terjemahan_en': 'Fear Allah wherever you are.',
      'terjemahan_ar': 'اتق الله حيثما كنت.',
      'sumber': 'HR. Tirmidzi',
      'type': 'hadits',
    },
    {
      'arab': 'وَاللَّهُ خَيْرُ الرَّازِقِينَ',
      'terjemahan': 'Dan Allah adalah sebaik-baik pemberi rezeki.',
      'terjemahan_en': 'Allah is the Best of providers.',
      'terjemahan_ar': 'والله خير الرازقين.',
      'sumber': 'QS. Al-Jumuah: 11',
      'type': 'ayat',
    },
    {
      'arab': 'إِنَّ مَعَ الصَّبْرِ نَصْرًا',
      'terjemahan': 'Sesungguhnya bersama kesabaran ada pertolongan.',
      'terjemahan_en': 'Indeed, with patience comes victory.',
      'terjemahan_ar': 'إن مع الصبر النصر.',
      'sumber': 'HR. Ahmad',
      'type': 'hadits',
    },
    {
      'arab': 'وَاللَّهُ يَهْدِي مَن يَشَاءُ إِلَىٰ صِرَاطٍ مُّسْتَقِيمٍ',
      'terjemahan': 'Dan Allah memberi petunjuk kepada siapa yang Dia kehendaki ke jalan yang lurus.',
      'terjemahan_en': 'Allah guides whom He wills to the straight path.',
      'terjemahan_ar': 'والله يهدي من يشاء إلى صراط مستقيم.',
      'sumber': 'QS. Al-Baqarah: 213',
      'type': 'ayat',
    },
    {
      'arab': 'مَنْ حَسُنَ إِسْلَامُهُ حَسُنَ كُلُّ شَيْءٍ مِنْهُ',
      'terjemahan': 'Barangsiapa baik islamnya, maka baik pula segala urusannya.',
      'terjemahan_en': 'Whoever\'s Islam is good, all his affairs will be good.',
      'terjemahan_ar': 'من حسن إسلامه حسن كل شيء منه.',
      'sumber': 'HR. Ahmad',
      'type': 'hadits',
    },
    {
      'arab': 'رَبَّنَا آتِنَا فِي الدُّنْيَا حَسَنَةً وَفِي الْآخِرَةِ حَسَنَةً',
      'terjemahan': 'Ya Tuhan kami, berilah kami kebaikan di dunia dan kebaikan di akhirat.',
      'terjemahan_en': 'Our Lord, grant us good in this world and good in the Hereafter.',
      'terjemahan_ar': 'ربنا آتنا في الدنيا حسنة وفي الآخرة حسنة.',
      'sumber': 'QS. Al-Baqarah: 201',
      'type': 'ayat',
    },
    {
      'arab': 'أَحَبُّ الْأَعْمَالِ إِلَى اللَّهِ أَدْوَمُهَا وَإِنْ قَلَّ',
      'terjemahan': 'Amalan yang paling dicintai Allah adalah yang paling rutin meskipun sedikit.',
      'terjemahan_en': 'The most beloved deeds to Allah are the most consistent, even if small.',
      'terjemahan_ar': 'أحب الأعمال إلى الله أدومها وإن قل.',
      'sumber': 'HR. Bukhari & Muslim',
      'type': 'hadits',
    },
    {
      'arab': 'وَمَا تَوْفِيقِي إِلَّا بِاللَّهِ',
      'terjemahan': 'Dan tidak ada taufik bagiku melainkan dari Allah.',
      'terjemahan_en': 'My success is only from Allah.',
      'terjemahan_ar': 'وما توفيقي إلا بالله.',
      'sumber': 'QS. Hud: 88',
      'type': 'ayat',
    },
    {
      'arab': 'الْيَدُ الْعُلْيَا خَيْرٌ مِنَ الْيَدِ السُّفْلَى',
      'terjemahan': 'Tangan di atas lebih baik daripada tangan di bawah.',
      'terjemahan_en': 'The upper hand is better than the lower hand.',
      'terjemahan_ar': 'اليد العليا خير من اليد السفلى.',
      'sumber': 'HR. Bukhari & Muslim',
      'type': 'hadits',
    },
    {
      'arab': 'إِنَّ اللَّهَ عَلَىٰ كُلِّ شَيْءٍ قَدِيرٌ',
      'terjemahan': 'Sesungguhnya Allah Maha Kuasa atas segala sesuatu.',
      'terjemahan_en': 'Indeed, Allah is over all things competent.',
      'terjemahan_ar': 'إن الله على كل شيء قدير.',
      'sumber': 'QS. Al-Baqarah: 20',
      'type': 'ayat',
    },
    {
      'arab': 'لَا ضَرَرَ وَلَا ضِرَارَ',
      'terjemahan': 'Tidak boleh membahayakan diri sendiri maupun orang lain.',
      'terjemahan_en': 'There shall be no harm and no reciprocation of harm.',
      'terjemahan_ar': 'لا ضرر ولا ضرار.',
      'sumber': 'HR. Ibnu Majah',
      'type': 'hadits',
    },
    {
      'arab': 'وَعَسَىٰ أَن تَكْرَهُوا شَيْئًا وَهُوَ خَيْرٌ لَّكُمْ',
      'terjemahan': 'Boleh jadi kamu membenci sesuatu, padahal ia baik bagimu.',
      'terjemahan_en': 'Perhaps you dislike something while it is good for you.',
      'terjemahan_ar': 'وعسى أن تكرهوا شيئاً وهو خير لكم.',
      'sumber': 'QS. Al-Baqarah: 216',
      'type': 'ayat',
    },
    {
      'arab': 'إِنَّ اللَّهَ جَمِيلٌ يُحِبُّ الْجَمَالَ',
      'terjemahan': 'Sesungguhnya Allah Maha Indah dan mencintai keindahan.',
      'terjemahan_en': 'Indeed, Allah is beautiful and loves beauty.',
      'terjemahan_ar': 'إن الله جميل يحب الجمال.',
      'sumber': 'HR. Muslim',
      'type': 'hadits',
    },
    {
      'arab': 'وَلَا تَيْأَسُوا مِن رَّوْحِ اللَّهِ',
      'terjemahan': 'Dan janganlah kamu berputus asa dari rahmat Allah.',
      'terjemahan_en': 'Do not despair of Allah\'s mercy.',
      'terjemahan_ar': 'ولا تيأسوا من روح الله.',
      'sumber': 'QS. Yusuf: 87',
      'type': 'ayat',
    },
    {
      'arab': 'مَنْ كَانَ يُؤْمِنُ بِاللَّهِ وَالْيَوْمِ الْآخِرِ فَلْيَقُلْ خَيْرًا أَوْ لِيَصْمُتْ',
      'terjemahan': 'Barangsiapa beriman kepada Allah dan hari akhir, hendaklah berkata baik atau diam.',
      'terjemahan_en': 'Whoever believes in Allah and the Last Day, let him speak good or be silent.',
      'terjemahan_ar': 'من كان يؤمن بالله واليوم الآخر فليقل خيراً أو ليصمت.',
      'sumber': 'HR. Bukhari & Muslim',
      'type': 'hadits',
    },
    {
      'arab': 'إِنَّ اللَّهَ غَفُورٌ رَّحِيمٌ',
      'terjemahan': 'Sesungguhnya Allah Maha Pengampun lagi Maha Penyayang.',
      'terjemahan_en': 'Indeed, Allah is Forgiving and Merciful.',
      'terjemahan_ar': 'إن الله غفور رحيم.',
      'sumber': 'QS. Al-Baqarah: 173',
      'type': 'ayat',
    },
    {
      'arab': 'أَكْمَلُ الْمُؤْمِنِينَ إِيمَانًا أَحْسَنُهُمْ خُلُقًا',
      'terjemahan': 'Mukmin yang paling sempurna imannya adalah yang paling baik akhlaknya.',
      'terjemahan_en': 'The most complete believer is the one with the best character.',
      'terjemahan_ar': 'أكمل المؤمنين إيماناً أحسنهم خلقاً.',
      'sumber': 'HR. Tirmidzi',
      'type': 'hadits',
    },
    {
      'arab': 'وَاللَّهُ سَمِيعٌ عَلِيمٌ',
      'terjemahan': 'Dan Allah Maha Mendengar lagi Maha Mengetahui.',
      'terjemahan_en': 'Allah is All-Hearing, All-Knowing.',
      'terjemahan_ar': 'والله سميع عليم.',
      'sumber': 'QS. Al-Baqarah: 227',
      'type': 'ayat',
    },
    {
      'arab': 'إِذَا مَاتَ ابْنُ آدَمَ انْقَطَعَ عَمَلُهُ إِلَّا مِنْ ثَلَاثٍ',
      'terjemahan': 'Jika anak Adam meninggal, terputuslah amalnya kecuali tiga hal: sedekah jariyah, ilmu yang bermanfaat, dan doa anak yang sholeh.',
      'terjemahan_en': 'When a person dies, his deeds end except three: ongoing charity, beneficial knowledge, or a righteous child who prays.',
      'terjemahan_ar': 'إذا مات ابن آدم انقطع عمله إلا من ثلاثة: صدقة جارية أو علم ينتفع به أو ولد صالح يدعو له.',
      'sumber': 'HR. Muslim',
      'type': 'hadits',
    },
    {
      'arab': 'حَسْبُنَا اللَّهُ وَنِعْمَ الْوَكِيلُ',
      'terjemahan': 'Cukuplah Allah menjadi penolong kami dan Allah adalah sebaik-baik pelindung.',
      'terjemahan_en': 'Sufficient for us is Allah and He is the best Protector.',
      'terjemahan_ar': 'حسبنا الله ونعم الوكيل.',
      'sumber': 'QS. Ali Imran: 173',
      'type': 'ayat',
    },
    {
      'arab': 'مَنْ سَلَكَ طَرِيقًا يَلْتَمِسُ فِيهِ عِلْمًا سَهَّلَ اللَّهُ لَهُ طَرِيقًا إِلَى الْجَنَّةِ',
      'terjemahan': 'Barangsiapa menempuh jalan untuk mencari ilmu, Allah mudahkan baginya jalan menuju surga.',
      'terjemahan_en': 'Whoever takes a path seeking knowledge, Allah eases his way to Paradise.',
      'terjemahan_ar': 'من سلك طريقاً يلتمس فيه علماً سهل الله له طريقاً إلى الجنة.',
      'sumber': 'HR. Muslim',
      'type': 'hadits',
    },
    {
      'arab': 'وَأَنَّ إِلَىٰ رَبِّكَ الْمُنتَهَىٰ',
      'terjemahan': 'Dan sesungguhnya kepada Tuhanmulah kesudahan segala sesuatu.',
      'terjemahan_en': 'And to your Lord is the final destination.',
      'terjemahan_ar': 'وأن إلى ربك المنتهى.',
      'sumber': 'QS. An-Najm: 42',
      'type': 'ayat',
    },
    {
      'arab': 'الْمُؤْمِنُ مِرْآةُ الْمُؤْمِنِ',
      'terjemahan': 'Seorang mukmin adalah cermin bagi mukmin lainnya.',
      'terjemahan_en': 'A believer is a mirror to another believer.',
      'terjemahan_ar': 'المؤمن مرآة المؤمن.',
      'sumber': 'HR. Abu Dawud',
      'type': 'hadits',
    },
    {
      'arab': 'وَاللَّهُ يَعْلَمُ وَأَنتُمْ لَا تَعْلَمُونَ',
      'terjemahan': 'Allah mengetahui sedangkan kamu tidak mengetahui.',
      'terjemahan_en': 'Allah knows while you do not know.',
      'terjemahan_ar': 'والله يعلم وأنتم لا تعلمون.',
      'sumber': 'QS. Al-Baqarah: 232',
      'type': 'ayat',
    },
    {
      'arab': 'أَفْضَلُ الصِّيَامِ بَعْدَ رَمَضَانَ شَهْرُ اللَّهِ الْمُحَرَّمُ',
      'terjemahan': 'Puasa yang paling utama setelah Ramadhan adalah puasa di bulan Muharram.',
      'terjemahan_en': 'The best fasting after Ramadan is in the month of Muharram.',
      'terjemahan_ar': 'أفضل الصيام بعد رمضان شهر الله المحرم.',
      'sumber': 'HR. Muslim',
      'type': 'hadits',
    },
    {
      'arab': 'إِنَّ اللَّهَ يُحِبُّ التَّوَّابِينَ وَيُحِبُّ الْمُتَطَهِّرِينَ',
      'terjemahan': 'Sesungguhnya Allah mencintai orang-orang yang bertaubat dan orang-orang yang menyucikan diri.',
      'terjemahan_en': 'Allah loves those who repent and those who purify themselves.',
      'terjemahan_ar': 'إن الله يحب التوابين ويحب المتطهرين.',
      'sumber': 'QS. Al-Baqarah: 222',
      'type': 'ayat',
    },
    {
      'arab': 'كُلُّ ابْنِ آدَمَ خَطَّاءٌ وَخَيْرُ الْخَطَّائِينَ التَّوَّابُونَ',
      'terjemahan': 'Setiap anak Adam pasti berbuat salah, dan sebaik-baik orang yang bersalah adalah yang bertaubat.',
      'terjemahan_en': 'Every son of Adam sins, and the best of sinners are those who repent.',
      'terjemahan_ar': 'كل ابن آدم خطاء وخير الخطائين التوابون.',
      'sumber': 'HR. Tirmidzi',
      'type': 'hadits',
    },
    {
      'arab': 'وَتُوبُوا إِلَى اللَّهِ جَمِيعًا',
      'terjemahan': 'Dan bertaubatlah kamu sekalian kepada Allah.',
      'terjemahan_en': 'And turn to Allah in repentance, all of you.',
      'terjemahan_ar': 'وتوبوا إلى الله جميعاً.',
      'sumber': 'QS. An-Nur: 31',
      'type': 'ayat',
    },
    {
      'arab': 'خَيْرُكُمْ مَنْ تَعَلَّمَ الْقُرْآنَ وَعَلَّمَهُ',
      'terjemahan': 'Sebaik-baik kalian adalah orang yang mempelajari Al-Qur\'an dan mengajarkannya.',
      'sumber': 'HR. Bukhari',
      'type': 'hadits',
    },
    {
      'arab': 'وَنَحْنُ أَقْرَبُ إِلَيْهِ مِنْ حَبْلِ الْوَرِيدِ',
      'terjemahan': 'Dan Kami lebih dekat kepadanya daripada urat lehernya sendiri.',
      'terjemahan_en': 'We are closer to him than his jugular vein.',
      'terjemahan_ar': 'ونحن أقرب إليه من حبل الوريد.',
      'sumber': 'QS. Qaf: 16',
      'type': 'ayat',
    },
    {
      'arab': 'الْبِرُّ حُسْنُ الْخُلُقِ',
      'terjemahan': 'Kebaikan adalah akhlak yang baik.',
      'terjemahan_en': 'Righteousness is good character.',
      'terjemahan_ar': 'البر حسن الخلق.',
      'sumber': 'HR. Muslim',
      'type': 'hadits',
    },
    {
      'arab': 'إِنَّ اللَّهَ لَا يُغَيِّرُ مَا بِقَوْمٍ حَتَّىٰ يُغَيِّرُوا مَا بِأَنفُسِهِمْ',
      'terjemahan': 'Sesungguhnya Allah tidak mengubah keadaan suatu kaum sebelum mereka mengubah keadaan diri mereka sendiri.',
      'terjemahan_en': 'Allah does not change a condition of a people until they change themselves.',
      'terjemahan_ar': 'إن الله لا يغير ما بقوم حتى يغيروا ما بأنفسهم.',
      'sumber': 'QS. Ar-Ra\'d: 11',
      'type': 'ayat',
    },
    {
      'arab': 'مَنْ لَا يَشْكُرِ النَّاسَ لَا يَشْكُرِ اللَّهَ',
      'terjemahan': 'Barangsiapa tidak bersyukur kepada manusia, maka ia tidak bersyukur kepada Allah.',
      'terjemahan_en': 'Whoever does not thank people does not thank Allah.',
      'terjemahan_ar': 'من لا يشكر الناس لا يشكر الله.',
      'sumber': 'HR. Abu Dawud',
      'type': 'hadits',
    },
    {
      'arab': 'وَاشْكُرُوا لِي وَلَا تَكْفُرُونِ',
      'terjemahan': 'Dan bersyukurlah kepada-Ku, dan janganlah kamu mengingkari nikmat-Ku.',
      'terjemahan_en': 'Be grateful to Me and do not deny My favor.',
      'terjemahan_ar': 'واشكروا لي ولا تكفرون.',
      'sumber': 'QS. Al-Baqarah: 152',
      'type': 'ayat',
    },
    {
      'arab': 'أَقْرَبُ مَا يَكُونُ الْعَبْدُ مِنْ رَبِّهِ وَهُوَ سَاجِدٌ',
      'terjemahan': 'Saat paling dekat seorang hamba dengan Tuhannya adalah ketika ia sedang sujud.',
      'terjemahan_en': 'The closest a servant gets to his Lord is when he is in prostration.',
      'terjemahan_ar': 'أقرب ما يكون العبد من ربه وهو ساجد.',
      'sumber': 'HR. Muslim',
      'type': 'hadits',
    },
    {
      'arab': 'وَلَذِكْرُ اللَّهِ أَكْبَرُ',
      'terjemahan': 'Dan sesungguhnya mengingat Allah adalah lebih besar (keutamaannya).',
      'terjemahan_en': 'And the remembrance of Allah is greater.',
      'terjemahan_ar': 'ولذكر الله أكبر.',
      'sumber': 'QS. Al-Ankabut: 45',
      'type': 'ayat',
    },
    {
      'arab': 'مَنْ صَلَّى عَلَيَّ وَاحِدَةً صَلَّى اللَّهُ عَلَيْهِ عَشْرًا',
      'terjemahan': 'Barangsiapa bershalawat kepadaku sekali, Allah akan bershalawat kepadanya sepuluh kali.',
      'terjemahan_en': 'Whoever sends one blessing upon me, Allah sends ten blessings upon him.',
      'terjemahan_ar': 'من صلى علي واحدة صلى الله عليه عشراً.',
      'sumber': 'HR. Muslim',
      'type': 'hadits',
    },
    {
      'arab': 'اللَّهُ نُورُ السَّمَاوَاتِ وَالْأَرْضِ',
      'terjemahan': 'Allah adalah cahaya langit dan bumi.',
      'terjemahan_en': 'Allah is the Light of the heavens and the earth.',
      'terjemahan_ar': 'الله نور السماوات والأرض.',
      'sumber': 'QS. An-Nur: 35',
      'type': 'ayat',
    },
  ];

  Widget _buildSlideAyat(double fs, AppProvider p) {
    // Pilih ayat berdasarkan hari agar ganti setiap hari
    final hariIni = DateTime.now().dayOfYear;
    final index = hariIni % _ayatHadits.length;
    final data = _ayatHadits[index];
    final isAyat = data['type'] == 'ayat';
    // Terjemahan sesuai bahasa
    final terjemahan = p.bahasa == 'en'
        ? (data['terjemahan_en'] ?? data['terjemahan']!)
        : p.bahasa == 'ar'
            ? (data['terjemahan_ar'] ?? data['terjemahan']!)
            : data['terjemahan']!;

    return Container(
      decoration: BoxDecoration(
        gradient: LinearGradient(
          colors: isAyat
            ? [const Color(0xFF0B2A1A), const Color(0xFF1A4A2E)]
            : [const Color(0xFF1A1A0B), const Color(0xFF3A3A1A)],
          begin: Alignment.topLeft,
          end: Alignment.bottomRight,
        ),
      ),
      child: Stack(children: [
        // Ornamen sudut
        Positioned(top: 16, left: 16,
          child: Text('❖', style: TextStyle(
            color: Colors.white.withValues(alpha: 0.1), fontSize: 48))),
        Positioned(bottom: 16, right: 16,
          child: Text('❖', style: TextStyle(
            color: Colors.white.withValues(alpha: 0.1), fontSize: 48))),
        // Konten
        Center(
          child: SingleChildScrollView(
            padding: EdgeInsets.symmetric(horizontal: (16 * fs).clamp(10.0, 40.0), vertical: (12 * fs).clamp(8.0, 30.0)),
            child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [
              // Badge tipe
              Container(
                padding: EdgeInsets.symmetric(horizontal: (10 * fs).clamp(6.0, 24.0), vertical: (3 * fs).clamp(2.0, 8.0)),
                decoration: BoxDecoration(
                  color: isAyat ? Colors.green.withValues(alpha: 0.3) : Colors.amber.withValues(alpha: 0.3),
                  borderRadius: BorderRadius.circular(16),
                  border: Border.all(color: isAyat ? Colors.greenAccent : Colors.amberAccent, width: 1)),
                child: Text(
                  isAyat ? p._t('📖  AYAT AL-QUR\'AN','📖  QUR\'AN VERSE','📖  آية قرآنية') : p._t('📜  HADITS NABI ﷺ','📜  HADITH ﷺ','📜  حديث النبي ﷺ'),
                  style: TextStyle(
                    color: isAyat ? Colors.greenAccent : Colors.amberAccent,
                    fontSize: (9 * fs).clamp(8.0, 20.0), letterSpacing: 1, fontWeight: FontWeight.bold)),
              ),
              SizedBox(height: (12 * fs).clamp(8.0, 30.0)),
              // Teks Arab
              Text(
                data['arab']!,
                textAlign: TextAlign.center,
                style: TextStyle(
                  color: Colors.white,
                  fontSize: (18 * fs).clamp(14.0, 52.0),
                  fontWeight: FontWeight.bold,
                  height: 1.6,
                ),
              ),
              SizedBox(height: (10 * fs).clamp(6.0, 24.0)),
              // Garis pemisah
              Row(children: [
                Expanded(child: Divider(color: Colors.white24)),
                Padding(
                  padding: EdgeInsets.symmetric(horizontal: (6 * fs).clamp(4.0, 14.0)),
                  child: Text('﷽', style: TextStyle(color: Colors.white38, fontSize: (13 * fs).clamp(10.0, 28.0))),
                ),
                Expanded(child: Divider(color: Colors.white24)),
              ]),
              SizedBox(height: (10 * fs).clamp(6.0, 24.0)),
              // Terjemahan
              Text(
                '"$terjemahan"',
                textAlign: TextAlign.center,
                style: TextStyle(
                  color: Colors.white70,
                  fontSize: (11 * fs).clamp(9.0, 26.0),
                  fontStyle: FontStyle.italic,
                  height: 1.5,
                ),
              ),
              SizedBox(height: (8 * fs).clamp(4.0, 18.0)),
              // Sumber
              Text(
                p.bahasa == 'ar'
                  ? (data['sumber']!).replaceAll('HR.', 'رواه').replaceAll('QS.', 'سورة').replaceAll('Bukhari & Muslim', 'البخاري ومسلم').replaceAll('Bukhari', 'البخاري').replaceAll('Muslim', 'مسلم').replaceAll('Ahmad', 'أحمد').replaceAll('Tirmidzi', 'الترمذي').replaceAll('Ibnu Majah', 'ابن ماجه')
                  : data['sumber']!,
                style: TextStyle(
                  color: isAyat ? Colors.greenAccent : Colors.amberAccent,
                  fontSize: (9 * fs).clamp(8.0, 20.0), letterSpacing: 1),
              ),
            ]),
          ),
        ),
      ]),
    );
  }


  // Responsive version of sholat item
  Widget _slideSholatItemR(String label, String waktu, double fs) => Container(
    width: double.infinity,
    padding: EdgeInsets.symmetric(horizontal: (8 * fs).clamp(6.0, 20.0), vertical: (6 * fs).clamp(4.0, 14.0)),
    decoration: BoxDecoration(
      color: Colors.white10,
      borderRadius: BorderRadius.circular(8),
    ),
    child: Row(
      mainAxisAlignment: MainAxisAlignment.spaceBetween,
      children: [
        Row(children: [
          Icon(Icons.circle, size: (5 * fs).clamp(4.0, 10.0), color: kAccent),
          SizedBox(width: (6 * fs).clamp(4.0, 14.0)),
          Text(label, style: TextStyle(color: Colors.white70, fontSize: (11 * fs).clamp(9.0, 26.0), letterSpacing: 1)),
        ]),
        Text(waktu, style: TextStyle(color: Colors.white, fontSize: (14 * fs).clamp(11.0, 32.0), fontWeight: FontWeight.bold, fontFamily: 'monospace')),
      ],
    ),
  );

  Widget _slideKasItem(String label, String nilai, Color color, {double fs = 1.0}) => Container(
    width: double.infinity,
    padding: EdgeInsets.symmetric(horizontal: (10 * fs).clamp(6.0, 24.0), vertical: (8 * fs).clamp(5.0, 18.0)),
    margin: const EdgeInsets.symmetric(vertical: 3),
    decoration: BoxDecoration(color: Colors.white10, borderRadius: BorderRadius.circular(8)),
    child: Row(mainAxisAlignment: MainAxisAlignment.spaceBetween, children: [
      Row(children: [
        Icon(Icons.circle, size: (6 * fs).clamp(4.0, 12.0), color: color),
        SizedBox(width: (8 * fs).clamp(4.0, 16.0)),
        Text(label, style: TextStyle(color: Colors.white70, fontSize: (11 * fs).clamp(9.0, 24.0), letterSpacing: 1)),
      ]),
      Flexible(child: FittedBox(fit: BoxFit.scaleDown,
        child: Text(nilai, style: TextStyle(color: color, fontSize: (12 * fs).clamp(10.0, 26.0), fontWeight: FontWeight.bold)),
      )),
    ]),
  );

  // Static helper terjemah kategori untuk slide
  static String _terjemahKategoriStatic(String k, String lang) {
    const en = {
      'Sedekah': 'Alms', 'Infak': 'Infaq', 'Zakat': 'Zakat',
      'Zakat Fitrah': 'Zakat Fitrah', 'Wakaf': 'Waqf',
      'Santunan Anak Yatim': 'Orphan Aid', 'Santunan Panti Asuhan': 'Orphanage Aid',
      'Pembangunan Masjid': 'Mosque Building', 'Operasional Masjid': 'Mosque Operations',
      'Lainnya': 'Others',
    };
    const ar = {
      'Sedekah': 'صدقة', 'Infak': 'إنفاق', 'Zakat': 'زكاة',
      'Zakat Fitrah': 'زكاة الفطر', 'Wakaf': 'وقف',
      'Santunan Anak Yatim': 'مساعدة الأيتام', 'Santunan Panti Asuhan': 'مساعدة دور الأيتام',
      'Pembangunan Masjid': 'بناء المسجد', 'Operasional Masjid': 'تشغيل المسجد',
      'Lainnya': 'أخرى',
    };
    if (lang == 'en') return en[k] ?? k;
    if (lang == 'ar') return ar[k] ?? k;
    return k;
  }

  // Helper load image - support web (data URI) dan Android (file path)
  Widget _loadImage(String path, {BoxFit fit = BoxFit.contain}) {
    if (path.startsWith('data:image') || path.startsWith('data:video')) {
      final b64 = path.split(',').last;
      return Image.memory(base64Decode(b64), fit: fit,
        errorBuilder: (_, __, ___) => _buildSlideError());
    }
    if (kIsWeb) {
      // Web tidak support Image.file
      return const Icon(Icons.broken_image, color: Colors.grey, size: 48);
    }
    return Image.file(File(path), fit: fit,
      errorBuilder: (_, __, ___) => _buildSlideError());
  }

  Widget _buildSlide(Map<String, dynamic> item, {bool isPaused = false, AppProvider? provider}) {
    final p2 = provider ?? context.read<AppProvider>();
    final path  = item['path'] as String;
    final type  = item['type'] as String;
    final judul = (item['judul'] ?? '').toString().trim();
    final tglRaw = (item['tgl'] ?? '').toString().trim();
    // Format tanggal sesuai bahasa
    final tgl = tglRaw.isNotEmpty ? p2.formatTgl(tglRaw) : '';

    Widget slideWidget;

    if (type == 'image') {
      slideWidget = Container(
        color: Colors.black,
        child: SizedBox.expand(
          child: _loadImage(path, fit: BoxFit.contain),
        ),
      );
    } else {
      // Video - putar langsung dari file, pindah slide setelah video selesai
      return _VideoSlideWidget(
        path: path,
        judul: judul,
        tgl: tgl,
        isPaused: isPaused,
        onVideoEnd: _nextSlideAfterVideo,
      );
    }

    // Stack dengan foto profil kanan tengah style TikTok
    return Stack(
      children: [
        slideWidget,

        // Judul + tanggal kiri bawah
        if (judul.isNotEmpty || tgl.isNotEmpty)
          Positioned(
            bottom: 16, left: 12, right: 70,
            child: Column(crossAxisAlignment: CrossAxisAlignment.start, mainAxisSize: MainAxisSize.min, children: [
              if (judul.isNotEmpty)
                Text(judul,
                  style: const TextStyle(color: Colors.white, fontSize: 16, fontWeight: FontWeight.bold,
                    shadows: [Shadow(color: Colors.black, blurRadius: 6)]),
                  maxLines: 2, overflow: TextOverflow.ellipsis),
              if (tgl.isNotEmpty)
                Text(tgl,
                  style: const TextStyle(color: Colors.white70, fontSize: 12,
                    shadows: [Shadow(color: Colors.black, blurRadius: 4)])),
            ]),
          ),

        // Foto profil masjid KANAN TENGAH - style TikTok
        Positioned(
          right: 10,
          top: 0, bottom: 0,
          child: Center(
            child: Container(
              decoration: BoxDecoration(
                shape: BoxShape.circle,
                border: Border.all(color: Colors.white, width: 2.5),
                boxShadow: [BoxShadow(color: Colors.black54, blurRadius: 8, spreadRadius: 1)],
              ),
              child: CircleAvatar(
                radius: 28,
                backgroundColor: const Color(0xFF631414),
                backgroundImage: p2.fotoMasjid.isNotEmpty
                    ? MemoryImage(base64Decode(p2.fotoMasjid)) : null,
                child: p2.fotoMasjid.isEmpty
                    ? const Icon(Icons.mosque, color: Colors.white, size: 26) : null,
              ),
            ),
          ),
        ),
      ],
    );
  }

  Widget _buildSlideKosong(AppProvider p) => Container(
        color: const Color(0xFF1A0505),
        child: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              Icon(Icons.mosque, color: Color(0xFF631414), size: 100),
              const SizedBox(height: 20),
              Text(p.namaIbadah, overflow: TextOverflow.ellipsis,
                  style: const TextStyle(
                      color: Colors.white,
                      fontSize: 28,
                      fontWeight: FontWeight.bold)),
              const SizedBox(height: 8),
              Text(p.alamat,
                  style: const TextStyle(color: Colors.white54, fontSize: 16)),
            ],
          ),
        ),
      );

  Widget _buildSlideError() => Container(
        color: Colors.black,
        child: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const Icon(Icons.broken_image, color: Colors.white38, size: 60),
              const SizedBox(height: 8),
              Consumer<AppProvider>(
                builder: (_, p, __) => Text(
                  p._t('File tidak ditemukan','File not found','الملف غير موجود'),
                  style: const TextStyle(color: Colors.white38)),
              ),
            ],
          ),
        ),
      );
}

// Widget jam mandiri - tidak rebuild slide saat detik berubah
class _LiveClock extends StatefulWidget {
  const _LiveClock();
  @override
  State<_LiveClock> createState() => _LiveClockState();
}

// Responsive version of LiveClock - scales with fontScale
class _LiveClockResponsive extends StatefulWidget {
  final double fontScale;
  const _LiveClockResponsive({required this.fontScale});
  @override
  State<_LiveClockResponsive> createState() => _LiveClockResponsiveState();
}

class _LiveClockResponsiveState extends State<_LiveClockResponsive> {
  late String _clock;
  late String _dateStr;
  Timer? _t;
  int _tzOffsetMenit = 420;

  @override
  void initState() {
    super.initState();
    try {
      final off = DateTime.now().timeZoneOffset.inMinutes;
      _tzOffsetMenit = (off != 0) ? off : 420;
    } catch (_) {}
    _clock = '';
    _dateStr = '';
    _update();
    _t = Timer.periodic(const Duration(seconds: 1), (_) { if (mounted) _update(); });
  }

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    // Rebuild saat bahasa berubah
    _update();
  }

  void _update() {
    final local = DateTime.now().toUtc().add(Duration(minutes: _tzOffsetMenit));
    if (mounted) {
      final bahasa = context.read<AppProvider>().bahasa;
      final locale = bahasa == 'ar' ? 'ar' : bahasa == 'en' ? 'en' : 'id';
      setState(() {
        _clock   = '${local.hour.toString().padLeft(2,'0')}:${local.minute.toString().padLeft(2,'0')}:${local.second.toString().padLeft(2,'0')}';
        try { _dateStr = DateFormat('EEEE, dd MMM yyyy', locale).format(local); }
        catch (_) { _dateStr = DateFormat('EEEE, dd MMM yyyy', 'id').format(local); }
      });
    }
  }

  @override
  void dispose() { _t?.cancel(); super.dispose(); }

  @override
  Widget build(BuildContext context) {
    final fs = widget.fontScale;
    // Watch bahasa - trigger _update saat bahasa berubah
    final bahasa = context.watch<AppProvider>().bahasa;
    final locale = bahasa == 'ar' ? 'ar' : bahasa == 'en' ? 'en' : 'id';
    final local = DateTime.now().toUtc().add(Duration(minutes: _tzOffsetMenit));
    String dateStr;
    try { dateStr = DateFormat('EEEE, dd MMM yyyy', locale).format(local); }
    catch (_) { dateStr = _dateStr; }

    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 4),
      child: Column(children: [
        Text(dateStr,
          textAlign: TextAlign.center,
          style: TextStyle(color: Colors.white54, fontSize: (8 * fs).clamp(8.0, 14.0))),
        FittedBox(
          fit: BoxFit.scaleDown,
          child: Text(_clock,
            style: TextStyle(
              color: Colors.yellowAccent,
              fontSize: (20 * fs).clamp(16.0, 48.0),
              fontWeight: FontWeight.bold,
              fontFamily: 'monospace')),
        ),
      ]),
    );
  }
}

class _LiveClockState extends State<_LiveClock> {
  late String _clock;
  late String _dateStr;
  Timer? _t;

  @override
  void initState() {
    super.initState();
    _initTimezone();
    _update();
    _t = Timer.periodic(const Duration(seconds: 1), (_) {
      if (mounted) _update();
    });
  }

  int _tzOffsetMenit = 420; // default WIB UTC+7

  void _initTimezone() {
    try {
      final off = DateTime.now().timeZoneOffset.inMinutes;
      _tzOffsetMenit = (off != 0) ? off : 420;
    } catch (_) {
      _tzOffsetMenit = 420;
    }
  }

  void _update() {
    final nowUtc = DateTime.now().toUtc();
    final local = nowUtc.add(Duration(minutes: _tzOffsetMenit));
    if (mounted) {
      final bahasa = context.read<AppProvider>().bahasa;
      final locale = bahasa == 'ar' ? 'ar' : bahasa == 'en' ? 'en' : 'id';
      setState(() {
        _clock   = '${local.hour.toString().padLeft(2,'0')}:${local.minute.toString().padLeft(2,'0')}:${local.second.toString().padLeft(2,'0')}';
        try { _dateStr = DateFormat('EEEE, dd MMM yyyy', locale).format(local); }
        catch (_) { _dateStr = DateFormat('EEEE, dd MMM yyyy', 'id').format(local); }
      });
    }
  }

  @override
  void dispose() { _t?.cancel(); super.dispose(); }

  @override
  Widget build(BuildContext context) => Column(children: [
    Padding(
      padding: const EdgeInsets.symmetric(horizontal: 6),
      child: Text(_dateStr,
          textAlign: TextAlign.center,
          style: const TextStyle(color: Colors.white54, fontSize: 10)),
    ),
    SizedBox(
      width: 140,
      child: FittedBox(
        fit: BoxFit.scaleDown,
        child: Text(_clock,
          style: const TextStyle(
              color: Colors.yellowAccent,
              fontSize: 24,
              fontWeight: FontWeight.bold,
              fontFamily: 'monospace')),
      ),
    ),
  ]);
}

class _RunningText extends StatelessWidget {
  final String text;
  final double kecepatan;
  final String warna;
  final double fontSize;
  const _RunningText({
    required this.text,
    this.kecepatan = 18.0,
    this.warna = 'FFFFFF',
    this.fontSize = 16.0,
  });

  Color _parseWarna(String hex) {
    try { return Color(int.parse('FF$hex', radix: 16)); }
    catch (_) { return Colors.white; }
  }

  @override
  Widget build(BuildContext context) {
    final color = _parseWarna(warna);
    if (text.isEmpty) return const SizedBox(height: 30);
    return SizedBox(
      height: 30,
      child: Marquee(
        text: text,
        style: TextStyle(
          color: color,
          fontSize: fontSize,
          fontWeight: FontWeight.w500,
          letterSpacing: 0.3,
        ),
        scrollAxis: Axis.horizontal,
        crossAxisAlignment: CrossAxisAlignment.center,
        blankSpace: 150.0,
        velocity: kecepatan * 0.8,
        pauseAfterRound: Duration.zero,
        showFadingOnlyWhenScrolling: false,
        fadingEdgeStartFraction: 0.0,
        fadingEdgeEndFraction: 0.0,
        accelerationDuration: Duration.zero,
        decelerationDuration: Duration.zero,
        startPadding: 0.0,
        textScaleFactor: 1.0,
      ),
    );
  }
}

class PengurusPage extends StatefulWidget {
  final VoidCallback? onBack;
  const PengurusPage({super.key, this.onBack});
  @override
  State<PengurusPage> createState() => _PengurusPageState();
}

class _PengurusPageState extends State<PengurusPage> {
  // Controller di State - aman dari lifecycle crash
  final _jabCtrl  = TextEditingController();
  final _namaCtrl = TextEditingController();

  @override
  void dispose() {
    _jabCtrl.dispose();
    _namaCtrl.dispose();
    super.dispose();
  }

  void _showForm(BuildContext context, AppProvider p, {int? editIndex}) {
    // Set nilai awal
    _jabCtrl.text  = editIndex != null ? p.pengurus[editIndex]['jabatan'] ?? '' : '';
    _namaCtrl.text = editIndex != null ? p.pengurus[editIndex]['nama'] ?? '' : '';

    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: Text(editIndex != null ? p.lEditPengurus : p.lTambahPengurus),
        content: SingleChildScrollView(child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            TextField(
              controller: _jabCtrl,
              textCapitalization: TextCapitalization.words,
              decoration: InputDecoration(
                labelText: p.lJabatan,
                hintText: p._t('contoh: Wakil Ketua, Sie Kebersihan...','e.g.: Vice Chairman, Cleaning...','مثال: نائب الرئيس، قسم النظافة...'),
              ),
            ),
            const SizedBox(height: 8),
            TextField(
              controller: _namaCtrl,
              textCapitalization: TextCapitalization.words,
              decoration: InputDecoration(labelText: p.lNama),
            ),
          ],
        )),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: Text(p.lBatal),
          ),
          ElevatedButton(
            style: ElevatedButton.styleFrom(
                backgroundColor: const Color(0xFF631414),
                foregroundColor: Colors.white),
            onPressed: () {
              if (_jabCtrl.text.trim().isEmpty || _namaCtrl.text.trim().isEmpty) return;
              if (editIndex != null) {
                p.editPengurus(editIndex, _jabCtrl.text.trim(), _namaCtrl.text.trim());
              } else {
                p.tambahPengurus(_jabCtrl.text.trim(), _namaCtrl.text.trim());
              }
              Navigator.pop(context);
            },
            child: Text(editIndex != null ? p.lSimpan : p.lTambah),
          ),
        ],
      ),
    );
  }


  Future<void> _uploadFoto(BuildContext context, AppProvider p, int i) async {
    final pg = p.pengurus[i];
    final picked = await ImagePicker().pickImage(source: ImageSource.gallery, imageQuality: 70);
    if (picked == null) return;
    final bytes = await picked.readAsBytes();
    p.editPengurus(i, pg['jabatan'] ?? '', pg['nama'] ?? '', foto: base64Encode(bytes));
  }

  @override
  Widget build(BuildContext context) {
    final p = context.watch<AppProvider>();
    return Scaffold(
      backgroundColor: kBg,
      body: ZoomWrapper(child: Column(
        children: [
          // Header
          Container(
            color: kPrimary,
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
            child: Row(children: [
              IconButton(
                icon: const Icon(Icons.arrow_back, color: Colors.white),
                onPressed: () => widget.onBack?.call(),
              ),
              const Icon(Icons.people, color: Colors.white, size: 18),
              const SizedBox(width: 8),
              Expanded(child: Text(
                '${p._t('PENGURUS MASJID','MOSQUE COMMITTEE','هيئة المسجد')}  •  ${p.pengurus.length} ${p._t('anggota','members','أعضاء')}',
                style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold, fontSize: 13),
              )),
              IconButton(
                icon: const Icon(Icons.person_add, color: Colors.white),
                tooltip: p.lTambahPengurus,
                onPressed: () => _showForm(context, p),
              ),
            ]),
          ),

          // Grid pengurus
          Expanded(
            child: p.pengurus.isEmpty
              ? Center(child: Column(mainAxisSize: MainAxisSize.min, children: [
                  const Icon(Icons.people_outline, size: 80, color: Colors.grey),
                  const SizedBox(height: 12),
                  Text(p._t('Belum ada pengurus','No committee yet','لا هيئة بعد'), style: const TextStyle(color: Colors.grey, fontSize: 15)),
                  const SizedBox(height: 20),
                  ElevatedButton.icon(
                    style: ElevatedButton.styleFrom(backgroundColor: kPrimary, foregroundColor: Colors.white),
                    icon: const Icon(Icons.add),
                    label: Text(p.lTambahPengurus),
                    onPressed: () => _showForm(context, p),
                  ),
                ]))
              : LayoutBuilder(builder: (ctx2, bc2) {
                  // Responsive: makin lebar layar makin besar card
                  final cardMax = bc2.maxWidth < 600 ? 130.0 : bc2.maxWidth < 1024 ? 150.0 : 180.0;
                  final ratio   = bc2.maxWidth < 600 ? 0.88 : bc2.maxWidth < 1024 ? 0.85 : 0.78;
                  return GridView.builder(
                  padding: const EdgeInsets.all(12),
                  gridDelegate: SliverGridDelegateWithMaxCrossAxisExtent(
                    maxCrossAxisExtent: cardMax,
                    mainAxisSpacing: 12,
                    crossAxisSpacing: 12,
                    childAspectRatio: ratio,
                  ),
                  itemCount: p.pengurus.length,
                  itemBuilder: (ctx, i) {
                    final pg = p.pengurus[i];
                    final foto = pg['foto'] ?? '';
                    final avatarR = bc2.maxWidth < 600 ? 20.0 : 26.0;
                    return ClipRRect(
                      borderRadius: BorderRadius.circular(12),
                      child: Container(
                        clipBehavior: Clip.hardEdge,
                        decoration: BoxDecoration(
                          color: kCard,
                          borderRadius: BorderRadius.circular(12),
                          boxShadow: [BoxShadow(color: kDivider, blurRadius: 4)],
                        ),
                        child: Column(mainAxisAlignment: MainAxisAlignment.spaceEvenly, children: [
                        // Foto
                        GestureDetector(
                          onTap: () => _uploadFoto(context, p, i),
                          child: Stack(alignment: Alignment.bottomRight, children: [
                            CircleAvatar(
                              radius: avatarR,
                              backgroundColor: kPrimary,
                              backgroundImage: foto.isNotEmpty ? MemoryImage(base64Decode(foto)) : null,
                              child: foto.isEmpty
                                ? Text('${(pg['nama'] ?? 'X')[0].toUpperCase()}',
                                    style: TextStyle(color: Colors.white, fontSize: avatarR * 0.6, fontWeight: FontWeight.bold))
                                : null,
                            ),
                            Container(
                              width: 16, height: 16,
                              decoration: BoxDecoration(
                                color: Colors.blue,
                                shape: BoxShape.circle,
                                border: Border.all(color: Colors.white, width: 1.5),
                              ),
                              child: const Icon(Icons.camera_alt, color: Colors.white, size: 9),
                            ),
                          ]),
                        ),
                        Padding(
                          padding: const EdgeInsets.symmetric(horizontal: 4),
                          child: Text(pg['nama'] ?? '-',
                            style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 10),
                            textAlign: TextAlign.center,
                            maxLines: 1,
                            overflow: TextOverflow.ellipsis,
                          ),
                        ),
                        Padding(
                          padding: const EdgeInsets.symmetric(horizontal: 4),
                          child: Text(pg['jabatan'] ?? '-',
                            style: TextStyle(color: kPrimary, fontSize: 9),
                            textAlign: TextAlign.center,
                            maxLines: 1,
                            overflow: TextOverflow.ellipsis,
                          ),
                        ),
                        // Tombol edit & hapus
                        Row(mainAxisAlignment: MainAxisAlignment.center, children: [
                          InkWell(
                            onTap: () => _showForm(context, p, editIndex: i),
                            child: const Padding(
                              padding: EdgeInsets.all(4),
                              child: Icon(Icons.edit, size: 14, color: Colors.grey),
                            ),
                          ),
                          const SizedBox(width: 8),
                          InkWell(
                            onTap: () async {
                              final ok = await showDialog<bool>(
                                context: context,
                                builder: (_) => AlertDialog(
                                  title: Text(p.lHapusPengurus),
                                  content: Text('${pg['nama']} - ${pg['jabatan']}'),
                                  actions: [
                                    TextButton(onPressed: () => Navigator.pop(context, false), child: Text(p.lBatal)),
                                    TextButton(onPressed: () => Navigator.pop(context, true),
                                      child: Text(p.lHapus, style: const TextStyle(color: Colors.red))),
                                  ],
                                ),
                              );
                              if (ok == true) p.hapusPengurus(i);
                            },
                            child: const Padding(
                              padding: EdgeInsets.all(4),
                              child: Icon(Icons.delete_outline, size: 14, color: Colors.red),
                            ),
                          ),
                        ]),
                      ]),
                    ));  // Container + ClipRRect
                  },
                );
                }) // LayoutBuilder
          ),
        ],
      ),
    ),
      floatingActionButton: FloatingActionButton.extended(
        backgroundColor: kPrimary,
        foregroundColor: Colors.white,
        icon: const Icon(Icons.person_add),
        label: Text(p._t('Tambah','Add','إضافة')),
        onPressed: () => _showForm(context, p),
      ),
    );
  }
}

// ══════════════════════════════════════════════
// HALAMAN ACARA / SPANDUK DIGITAL
// ══════════════════════════════════════════════
class AcaraPage extends StatefulWidget {
  final VoidCallback? onBack;
  const AcaraPage({super.key, this.onBack});
  @override
  State<AcaraPage> createState() => _AcaraPageState();
}

class _AcaraPageState extends State<AcaraPage> {
  final _namaAcaraCtrl    = TextEditingController();
  final _tanggalAcaraCtrl = TextEditingController();
  final _sambutanAcaraCtrl= TextEditingController();
  final _namaTamuCtrl     = TextEditingController();
  final _jabatanTamuCtrl  = TextEditingController();
  // Controller untuk form tamu inline di dalam form acara
  final _namaTamuInlineCtrl    = TextEditingController();
  final _jabatanTamuInlineCtrl = TextEditingController();

  @override
  void dispose() {
    _namaAcaraCtrl.dispose();
    _tanggalAcaraCtrl.dispose();
    _sambutanAcaraCtrl.dispose();
    _namaTamuCtrl.dispose();
    _jabatanTamuCtrl.dispose();
    _namaTamuInlineCtrl.dispose();
    _jabatanTamuInlineCtrl.dispose();
    super.dispose();
  }

  void _showFormAcara(BuildContext ctx, AppProvider p, {int? editIdx}) {
    _namaAcaraCtrl.text     = editIdx != null ? p.acaraList[editIdx]['namaAcara'] ?? '' : '';
    _tanggalAcaraCtrl.text  = editIdx != null ? p.acaraList[editIdx]['tanggal']   ?? '' : '';
    _sambutanAcaraCtrl.text = editIdx != null ? p.acaraList[editIdx]['sambutan']  ?? '' : '';
    // tamu sementara di dalam dialog sebelum disimpan
    final List<Map<String, dynamic>> tamuTemp = editIdx != null
        ? List<Map<String, dynamic>>.from(
            (p.acaraList[editIdx]['tamu'] as List<dynamic>).map((e) => Map<String, dynamic>.from(e as Map)))
        : [];
    showDialog(
      context: ctx,
      builder: (_) => StatefulBuilder(
        builder: (ctx2, setS) => AlertDialog(
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
        title: Text(editIdx != null ? p.lTambahAcara.replaceFirst('Tambah','Edit') : p.lTambahAcara,
            style: const TextStyle(color: kPrimary, fontWeight: FontWeight.bold)),
        content: SizedBox(
          width: double.maxFinite,
          child: SingleChildScrollView(child: Column(mainAxisSize: MainAxisSize.min, crossAxisAlignment: CrossAxisAlignment.start, children: [
          TextField(
            controller: _namaAcaraCtrl,
            autofocus: true,
            textCapitalization: TextCapitalization.words,
            maxLength: 100,
            decoration: InputDecoration(labelText: '${p.lNamaAcara} *', hintText: 'Maulid Nabi, Isra Miraj...',
              prefixIcon: const Icon(Icons.celebration, color: kPrimary, size: 20),
              filled: true, fillColor: kBg,
              border: OutlineInputBorder(borderRadius: BorderRadius.circular(8))),
          ),
          const SizedBox(height: 10),
          GestureDetector(
            onTap: () async {
              final picked = await showDatePicker(
                context: ctx2,
                initialDate: DateTime.tryParse(_tanggalAcaraCtrl.text.split('/').length == 3
                  ? '${_tanggalAcaraCtrl.text.split('/')[2]}-${_tanggalAcaraCtrl.text.split('/')[1]}-${_tanggalAcaraCtrl.text.split('/')[0]}'
                  : DateTime.now().toIso8601String()) ?? DateTime.now(),
                firstDate: DateTime(2000),
                lastDate: DateTime(2100),
              );
              if (picked != null) {
                _tanggalAcaraCtrl.text = DateFormat('dd/MM/yyyy').format(picked);
              }
            },
            child: AbsorbPointer(
              child: TextField(
                controller: _tanggalAcaraCtrl,
                maxLength: 50,
                keyboardType: TextInputType.datetime,
                decoration: InputDecoration(
                  labelText: p.lTanggal,
                  hintText: 'dd/MM/yyyy',
                  prefixIcon: const Icon(Icons.calendar_today, color: kPrimary, size: 20),
                  filled: true, fillColor: kBg,
                  border: OutlineInputBorder(borderRadius: BorderRadius.circular(8))),
              ),
            ),
          ),
          const SizedBox(height: 10),
          TextField(
            controller: _sambutanAcaraCtrl,
            maxLines: 3,
            maxLength: 200,
            textCapitalization: TextCapitalization.sentences,
            decoration: InputDecoration(labelText: p.lSambutan, hintText: p._t('Selamat Datang Yang Terhormat...','Welcome Distinguished Guests...','أهلاً وسهلاً بالضيوف الكرام...'),
              prefixIcon: const Icon(Icons.format_quote, color: kPrimary, size: 20),
              filled: true, fillColor: kBg,
              alignLabelWithHint: true,
              border: OutlineInputBorder(borderRadius: BorderRadius.circular(8))),
          ),
          const SizedBox(height: 14),
          Row(children: [
            const Icon(Icons.people, color: kPrimary, size: 16),
            const SizedBox(width: 6),
            Flexible(child: Text(p._t('Foto Tokoh Undangan','Guest Photos','صور الضيوف'), overflow: TextOverflow.ellipsis, style: const TextStyle(color: kPrimary, fontWeight: FontWeight.w600, fontSize: 13))),
          ]),
          const SizedBox(height: 8),
          Wrap(
            spacing: 8,
            runSpacing: 8,
            children: [
              ...List.generate(tamuTemp.length, (i) {
                final t = tamuTemp[i];
                final foto = (t['foto'] ?? '') as String;
                return SizedBox(
                  width: 85,
                  child: Column(mainAxisSize: MainAxisSize.min, children: [
                    GestureDetector(
                      onTap: () async {
                        try {
                          final picked = await ImagePicker().pickImage(source: ImageSource.gallery, imageQuality: 70);
                          if (picked == null) return;
                          final bytes = await picked.readAsBytes();
                          if (!ctx2.mounted) return;
                          setS(() { tamuTemp[i]['foto'] = base64Encode(bytes); });
                        } catch (e) {
                          debugPrint('Error foto tamu inline: $e');
                        }
                      },
                      child: Stack(alignment: Alignment.topRight, children: [
                        Container(
                          width: 85, height: 85,
                          decoration: BoxDecoration(
                            color: kPrimary.withValues(alpha: 0.07),
                            borderRadius: BorderRadius.circular(10),
                            border: Border.all(color: kPrimary.withValues(alpha: 0.35), width: 1.5),
                          ),
                          child: foto.isNotEmpty
                            ? ClipRRect(borderRadius: BorderRadius.circular(9),
                                child: Image.memory(base64Decode(foto), fit: BoxFit.cover))
                            : Column(mainAxisAlignment: MainAxisAlignment.center, children: [
                                Icon(Icons.add_a_photo, color: kPrimary.withValues(alpha: 0.45), size: 26),
                                const SizedBox(height: 3),
                                Text(p._t('Tap foto','Tap photo','اضغط الصورة'), style: TextStyle(color: kPrimary.withValues(alpha: 0.45), fontSize: 9), textAlign: TextAlign.center),
                              ]),
                        ),
                        GestureDetector(
                          onTap: () => setS(() => tamuTemp.removeAt(i)),
                          child: Container(width: 18, height: 18,
                            decoration: const BoxDecoration(color: Colors.red, shape: BoxShape.circle),
                            child: const Icon(Icons.close, color: Colors.white, size: 11)),
                        ),
                      ]),
                    ),
                    const SizedBox(height: 3),
                    Text(t['nama'] ?? '', style: const TextStyle(fontSize: 10, fontWeight: FontWeight.bold),
                      textAlign: TextAlign.center, maxLines: 1, overflow: TextOverflow.ellipsis),
                    Text(t['jabatan'] ?? '', style: const TextStyle(fontSize: 9, color: kPrimary),
                      textAlign: TextAlign.center, maxLines: 1, overflow: TextOverflow.ellipsis),
                  ]),
                );
              }),
              GestureDetector(
                onTap: () {
                  _namaTamuInlineCtrl.clear();
                  _jabatanTamuInlineCtrl.clear();
                  showDialog(
                    context: ctx2,
                    builder: (_) => AlertDialog(
                      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(10)),
                      title: Text(p.lTambahTamu, style: const TextStyle(color: kPrimary, fontSize: 15, fontWeight: FontWeight.bold)),
                      content: Column(mainAxisSize: MainAxisSize.min, children: [
                        TextField(controller: _namaTamuInlineCtrl, autofocus: true,
                          textCapitalization: TextCapitalization.words,
                          decoration: InputDecoration(labelText: '${p.lNama} *', filled: true, fillColor: kBg,
                            border: OutlineInputBorder(borderRadius: BorderRadius.circular(8)))),
                        const SizedBox(height: 10),
                        TextField(controller: _jabatanTamuInlineCtrl,
                          textCapitalization: TextCapitalization.words,
                          decoration: InputDecoration(labelText: p.lJabatan, filled: true, fillColor: kBg,
                            border: OutlineInputBorder(borderRadius: BorderRadius.circular(8)))),
                      ]),
                      actions: [
                        TextButton(onPressed: () => Navigator.pop(ctx2),
                          child: Text(p.lBatal)),
                        ElevatedButton(
                          style: ElevatedButton.styleFrom(backgroundColor: kPrimary, foregroundColor: Colors.white),
                          onPressed: () {
                            if (_namaTamuInlineCtrl.text.trim().isEmpty) return;
                            setS(() => tamuTemp.add({'nama': _namaTamuInlineCtrl.text.trim(), 'jabatan': _jabatanTamuInlineCtrl.text.trim(), 'foto': ''}));
                            Navigator.pop(ctx2);
                          },
                          child: Text(p.lTambah),
                        ),
                      ],
                    ),
                  );
                },
                child: Container(
                  width: 85, height: 85,
                  decoration: BoxDecoration(
                    color: kPrimary.withValues(alpha: 0.04),
                    borderRadius: BorderRadius.circular(10),
                    border: Border.all(color: kPrimary.withValues(alpha: 0.3), width: 1.5),
                  ),
                  child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [
                    Icon(Icons.person_add, color: kPrimary.withValues(alpha: 0.45), size: 26),
                    const SizedBox(height: 3),
                    Text('+ ${p.lTamu}', style: TextStyle(color: kPrimary.withValues(alpha: 0.45), fontSize: 9), textAlign: TextAlign.center),
                  ]),
                ),
              ),
            ],
          ),
        ]))),
        actions: [
          TextButton(onPressed: () { Navigator.pop(ctx2); },
            child: Text(p.lBatal)),
          ElevatedButton(
            style: ElevatedButton.styleFrom(backgroundColor: kPrimary, foregroundColor: Colors.white),
            onPressed: () {
              if (_namaAcaraCtrl.text.trim().isEmpty) return;
              if (editIdx != null) {
                p.updateAcaraDenganTamu(editIdx, _namaAcaraCtrl.text.trim(), _tanggalAcaraCtrl.text.trim(), _sambutanAcaraCtrl.text.trim(), tamuTemp);
              } else {
                p.addAcaraDenganTamu(_namaAcaraCtrl.text.trim(), _tanggalAcaraCtrl.text.trim(), _sambutanAcaraCtrl.text.trim(), tamuTemp);
              }
              Navigator.pop(ctx2);
            },
            child: Consumer<AppProvider>(builder: (_, p, __) => Text(editIdx != null ? p.lSimpan : p.lTambah)),
          ),
        ],
      ),
      ),
    );
  }

  void _showFormTamu(BuildContext ctx, AppProvider p, int acaraIdx) {
    _namaTamuCtrl.clear();
    _jabatanTamuCtrl.clear();
    String fotoBase64 = '';
    showDialog(
      context: ctx,
      builder: (_) => StatefulBuilder(
        builder: (ctx2, setS) => AlertDialog(
          shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
          title: Text(p.lTambahTamu, style: const TextStyle(color: kPrimary, fontWeight: FontWeight.bold)),
          content: SizedBox(
            width: double.maxFinite,
            child: SingleChildScrollView(child: Column(mainAxisSize: MainAxisSize.min, children: [
            Center(
              child: GestureDetector(
                onTap: () async {
                  try {
                    final picked = await ImagePicker().pickImage(source: ImageSource.gallery, imageQuality: 70);
                    if (picked == null) return;
                    final bytes = await picked.readAsBytes();
                    // Cek context masih valid setelah await
                    if (!ctx2.mounted) return;
                    setS(() { fotoBase64 = base64Encode(bytes); });
                  } catch (e) {
                    debugPrint('Error pick foto tamu: $e');
                  }
                },
                child: Container(
                  width: 110, height: 110,
                  decoration: BoxDecoration(
                    color: kPrimary.withValues(alpha: 0.08),
                    borderRadius: BorderRadius.circular(12),
                    border: Border.all(color: kPrimary, width: 2),
                  ),
                  child: fotoBase64.isNotEmpty
                    ? ClipRRect(borderRadius: BorderRadius.circular(10),
                        child: Image.memory(base64Decode(fotoBase64), fit: BoxFit.cover))
                    : Column(mainAxisAlignment: MainAxisAlignment.center, children: [
                        const Icon(Icons.add_a_photo, color: kPrimary, size: 36),
                        const SizedBox(height: 6),
                        Text(p._t('Tap untuk\nupload foto','Tap to\nupload photo','اضغط لرفع\nالصورة'),
                          style: const TextStyle(color: kPrimary, fontSize: 11), textAlign: TextAlign.center),
                      ]),
                ),
              ),
            ),
            const SizedBox(height: 14),
            TextField(
              controller: _namaTamuCtrl, autofocus: true,
              textCapitalization: TextCapitalization.words, maxLength: 50,
              decoration: InputDecoration(labelText: '${p.lNama} *',
                prefixIcon: const Icon(Icons.person, color: kPrimary, size: 20),
                filled: true, fillColor: kBg,
                border: OutlineInputBorder(borderRadius: BorderRadius.circular(8))),
            ),
            const SizedBox(height: 10),
            TextField(
              controller: _jabatanTamuCtrl,
              textCapitalization: TextCapitalization.words, maxLength: 50,
              decoration: InputDecoration(labelText: p.lJabatan, hintText: p._t('Camat, Lurah, Gubernur...','Mayor, Governor...','عمدة، حاكم...'),
                prefixIcon: const Icon(Icons.badge, color: kPrimary, size: 20),
                filled: true, fillColor: kBg,
                border: OutlineInputBorder(borderRadius: BorderRadius.circular(8))),
            ),
          ])),),
          actions: [
            TextButton(onPressed: () { Navigator.pop(ctx2); },
              child: Text(p.lBatal)),
            ElevatedButton(
              style: ElevatedButton.styleFrom(backgroundColor: kPrimary, foregroundColor: Colors.white),
              onPressed: () {
                // Cek nama tamu - bukan nama acara!
                if (_namaTamuCtrl.text.trim().isEmpty) return;
                p.addTamu(acaraIdx, _namaTamuCtrl.text.trim(), _jabatanTamuCtrl.text.trim(), fotoBase64);
                Navigator.pop(ctx2);
              },
              child: Text(p.lTambah),
            ),
          ],
        ),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    final p = context.watch<AppProvider>();
    return Scaffold(
      backgroundColor: kBg,
      body: ZoomWrapper(child: Column(children: [
        // Header
        Container(
          color: kPrimary,
          padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
          child: Row(children: [
            IconButton(
              icon: const Icon(Icons.arrow_back, color: Colors.white),
              tooltip: p._t('Kembali','Back','رجوع'),
              onPressed: () => widget.onBack?.call(),
            ),
            const Icon(Icons.celebration, color: Colors.white, size: 18),
            const SizedBox(width: 8),
            Expanded(child: Text('${p._t('ACARA','EVENTS','الفعاليات')}  •  ${p.acaraList.length} ${p._t('acara','events','فعاليات')}',
              style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold, fontSize: 13))),
            IconButton(
              icon: const Icon(Icons.add_circle, color: Colors.white),
              tooltip: p.lTambahAcara,
              onPressed: () => _showFormAcara(context, p),
            ),
          ]),
        ),
        // List acara
        Expanded(
          child: p.acaraList.isEmpty
            ? Center(child: Column(mainAxisSize: MainAxisSize.min, children: [
                const Icon(Icons.celebration_outlined, size: 80, color: Colors.grey),
                const SizedBox(height: 12),
                Text(p._t('Belum ada acara','No events yet','لا فعاليات بعد'), style: const TextStyle(color: Colors.grey, fontSize: 15)),
                const SizedBox(height: 20),
                ElevatedButton.icon(
                  style: ElevatedButton.styleFrom(backgroundColor: kPrimary, foregroundColor: Colors.white),
                  icon: const Icon(Icons.add),
                  label: Text(p.lTambahAcara),
                  onPressed: () => _showFormAcara(context, p),
                ),
              ]))
            : ListView.builder(
                padding: const EdgeInsets.all(12),
                itemCount: p.acaraList.length,
                itemBuilder: (ctx, aIdx) {
                  final acara  = p.acaraList[aIdx];
                  final nama   = acara['namaAcara'] ?? '';
                  final tglRaw2 = acara['tanggal'] ?? '';
                  final tgl    = tglRaw2.isNotEmpty ? p.formatTgl(tglRaw2) : '';
                  final tamu   = (acara['tamu'] as List<dynamic>?) ?? [];
                  return Container(
                    margin: const EdgeInsets.only(bottom: 12),
                    decoration: BoxDecoration(
                      color: kCard, borderRadius: BorderRadius.circular(12),
                      boxShadow: [BoxShadow(color: kDivider, blurRadius: 4)]),
                    child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                      // Header kartu acara
                      Container(
                        padding: const EdgeInsets.symmetric(horizontal: 14, vertical: 10),
                        decoration: BoxDecoration(
                          color: kPrimary.withValues(alpha: 0.08),
                          borderRadius: const BorderRadius.vertical(top: Radius.circular(12))),
                        child: Row(children: [
                          Icon(Icons.celebration, color: kPrimary, size: 18),
                          const SizedBox(width: 8),
                          Expanded(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                            Text(nama, style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 14)),
                            if (tgl.isNotEmpty)
                              Text(tgl, style: const TextStyle(color: kTextGrey, fontSize: 12)),
                          ])),
                          IconButton(icon: const Icon(Icons.edit, size: 18, color: kTextGrey),
                            onPressed: () => _showFormAcara(ctx, p, editIdx: aIdx),
                            padding: EdgeInsets.zero, constraints: const BoxConstraints()),
                          const SizedBox(width: 8),
                          IconButton(icon: const Icon(Icons.delete_outline, size: 18, color: Colors.red),
                            onPressed: () async {
                              final ok = await showDialog<bool>(context: ctx,
                                builder: (_) => AlertDialog(
                                  title: Text(p._t('Hapus Acara?','Delete Event?','حذف الفعالية؟')),
                                  content: Text(p._t('Hapus $nama beserta semua tamu?','Delete $nama and all guests?','حذف $nama وجميع الضيوف؟')),
                                  actions: [
                                    TextButton(onPressed: () => Navigator.pop(ctx, false), child: Text(p.lBatal)),
                                    TextButton(onPressed: () => Navigator.pop(ctx, true),
                                      child: Text(p.lHapus, style: const TextStyle(color: Colors.red))),
                                  ],
                                ));
                              if (ok == true) p.deleteAcara(aIdx);
                            },
                            padding: EdgeInsets.zero, constraints: const BoxConstraints()),
                        ]),
                      ),
                      // Daftar tamu
                      Padding(
                        padding: const EdgeInsets.all(10),
                        child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                          Row(children: [
                            Flexible(child: Text('${p._t('Tamu Undangan','Invited Guests','الضيوف')} (${tamu.length})',
                              overflow: TextOverflow.ellipsis,
                              style: const TextStyle(color: kPrimary, fontWeight: FontWeight.w600, fontSize: 12))),
                            const SizedBox(width: 8),
                            TextButton.icon(
                              icon: const Icon(Icons.person_add, size: 14, color: kPrimary),
                              label: Text(p._t('Tambah','Add','إضافة'), style: const TextStyle(color: kPrimary, fontSize: 12)),
                              onPressed: () => _showFormTamu(ctx, p, aIdx),
                              style: TextButton.styleFrom(padding: const EdgeInsets.symmetric(horizontal: 8)),
                            ),
                          ]),
                          const SizedBox(height: 8),
                          // Grid foto tokoh — selalu tampil, + kotak tambah di akhir
                          Wrap(
                            spacing: 8,
                            runSpacing: 8,
                            children: [
                              // Kotak per tamu
                              ...List.generate(tamu.length, (tIdx) {
                                final t = tamu[tIdx] as Map<String, dynamic>;
                                final foto = (t['foto'] ?? '') as String;
                                return SizedBox(
                                  width: 90,
                                  child: Column(mainAxisSize: MainAxisSize.min, children: [
                                    GestureDetector(
                                      onTap: () async {
                                        final picked = await ImagePicker().pickImage(source: ImageSource.gallery, imageQuality: 70);
                                        if (picked == null) return;
                                        final bytes = await picked.readAsBytes();
                                        p.updateFotoTamu(aIdx, tIdx, base64Encode(bytes));
                                      },
                                      child: Stack(alignment: Alignment.topRight, children: [
                                        Container(
                                          width: 90, height: 90,
                                          decoration: BoxDecoration(
                                            color: kPrimary.withValues(alpha: 0.08),
                                            borderRadius: BorderRadius.circular(10),
                                            border: Border.all(color: kPrimary.withValues(alpha: 0.3), width: 1.5),
                                          ),
                                          child: foto.isNotEmpty
                                            ? ClipRRect(
                                                borderRadius: BorderRadius.circular(9),
                                                child: Image.memory(base64Decode(foto), fit: BoxFit.cover))
                                            : Column(mainAxisAlignment: MainAxisAlignment.center, children: [
                                                Icon(Icons.add_a_photo, color: kPrimary.withValues(alpha: 0.5), size: 28),
                                                const SizedBox(height: 4),
                                                Consumer<AppProvider>(builder: (_, p2, __) => Text(p2._t('Tap foto','Tap photo','اضغط الصورة'), style: TextStyle(color: kPrimary.withValues(alpha: 0.5), fontSize: 9))),
                                              ]),
                                        ),
                                        // Tombol hapus
                                        GestureDetector(
                                          onTap: () => p.deleteTamu(aIdx, tIdx),
                                          child: Container(
                                            width: 18, height: 18,
                                            decoration: const BoxDecoration(color: Colors.red, shape: BoxShape.circle),
                                            child: const Icon(Icons.close, color: Colors.white, size: 12),
                                          ),
                                        ),
                                      ]),
                                    ),
                                    const SizedBox(height: 4),
                                    Text(t['nama'] ?? '-',
                                      style: const TextStyle(fontSize: 11, fontWeight: FontWeight.bold),
                                      textAlign: TextAlign.center, maxLines: 1, overflow: TextOverflow.ellipsis),
                                    Text(t['jabatan'] ?? '',
                                      style: TextStyle(fontSize: 10, color: kPrimary),
                                      textAlign: TextAlign.center, maxLines: 1, overflow: TextOverflow.ellipsis),
                                  ]),
                                );
                              }),
                              // Kotak tambah tamu baru
                              GestureDetector(
                                onTap: () => _showFormTamu(ctx, p, aIdx),
                                child: Container(
                                  width: 90, height: 90,
                                  decoration: BoxDecoration(
                                    color: kPrimary.withValues(alpha: 0.05),
                                    borderRadius: BorderRadius.circular(10),
                                    border: Border.all(color: kPrimary.withValues(alpha: 0.3), width: 1.5, style: BorderStyle.solid),
                                  ),
                                  child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [
                                    Icon(Icons.person_add, color: kPrimary.withValues(alpha: 0.5), size: 28),
                                    const SizedBox(height: 4),
                                    Text(p._t('Tambah\nTamu','Add\nGuest','إضافة\nضيف'), style: TextStyle(color: kPrimary.withValues(alpha: 0.5), fontSize: 9),
                                      textAlign: TextAlign.center),
                                  ]),
                                ),
                              ),
                            ],
                          ),
                        ]),
                      ),
                    ]),
                  );
                },
              ),
        ),
      ]),
    ),
      floatingActionButton: FloatingActionButton.extended(
        backgroundColor: kPrimary,
        foregroundColor: Colors.white,
        icon: const Icon(Icons.add),
        label: Text(p.lTambahAcara),
        onPressed: () => _showFormAcara(context, p),
      ),
    );
  }
}

// ══════════════════════════════════════════════
// HALAMAN QRIS & REKENING
// ══════════════════════════════════════════════
class QrisPage extends StatefulWidget {
  const QrisPage({super.key});
  @override
  State<QrisPage> createState() => _QrisPageState();
}

class _QrisPageState extends State<QrisPage> {
  late TextEditingController _bankCtrl;
  late TextEditingController _norekCtrl;
  late TextEditingController _pemilikCtrl;
  String _fotoQris = '';
  bool _saved = false;

  @override
  void initState() {
    super.initState();
    final p = context.read<AppProvider>();
    _bankCtrl    = TextEditingController(text: p.namaBank);
    _norekCtrl   = TextEditingController(text: p.noRekening);
    _pemilikCtrl = TextEditingController(text: p.namaPemilik);
    _fotoQris    = p.fotoQris;
  }

  @override
  void dispose() {
    _bankCtrl.dispose();
    _norekCtrl.dispose();
    _pemilikCtrl.dispose();
    super.dispose();
  }

  Future<void> _uploadQris() async {
    final picked = await ImagePicker().pickImage(source: ImageSource.gallery, imageQuality: 70);
    if (picked == null) return;
    final bytes = await picked.readAsBytes();
    if (!mounted) return;
    setState(() {
      _fotoQris = base64Encode(bytes);
      _saved = false;
    });
  }

  void _simpan() {
    final p = context.read<AppProvider>();
    p.updateQris(_fotoQris, _bankCtrl.text.trim(), _norekCtrl.text.trim(), _pemilikCtrl.text.trim());
    setState(() => _saved = true);
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text(context.read<AppProvider>()._t('✅ QRIS & Rekening tersimpan!','✅ QRIS & Account saved!','✅ تم حفظ QRIS والحساب!')), backgroundColor: Colors.green));
  }

  @override
  Widget build(BuildContext context) {
    final p = context.watch<AppProvider>();
    return Scaffold(
      backgroundColor: kBg,
      appBar: AppBar(
        title: Text(p._t('QRIS & REKENING MASJID','MOSQUE QRIS & ACCOUNT','QRIS ومعلومات الحساب')),
        backgroundColor: kPrimary,
        foregroundColor: Colors.white,
      ),
      body: ZoomWrapper(child: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [

          // Info
          Container(
            padding: const EdgeInsets.all(12),
            decoration: BoxDecoration(
              color: Colors.green.withValues(alpha: 0.1),
              borderRadius: BorderRadius.circular(10),
              border: Border.all(color: Colors.green.withValues(alpha: 0.3)),
            ),
            child: Consumer<AppProvider>(builder: (_, p2, __) => Row(children: [
              const Icon(Icons.info_outline, color: Colors.green, size: 18),
              const SizedBox(width: 8),
              Expanded(child: Text(
                p2._t(
                  'Upload foto QRIS dan isi info rekening. Akan tampil otomatis sebagai slide di Live Display.',
                  'Upload QRIS photo and fill in bank info. It will appear automatically as a slide in Live Display.',
                  'ارفع صورة QRIS وأدخل معلومات الحساب. ستظهر تلقائياً كشريحة في العرض المباشر.',
                ),
                style: const TextStyle(fontSize: 12, color: Colors.green),
              )),
            ])),
          ),
          const SizedBox(height: 20),

          // Upload foto QRIS
          Consumer<AppProvider>(builder: (_, p2, __) => Text(p2._t('FOTO QRIS','QRIS PHOTO','صورة QRIS'), style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 12, color: kTextGrey, letterSpacing: 1))),
          const SizedBox(height: 8),
          GestureDetector(
            onTap: _uploadQris,
            child: Container(
              width: double.infinity,
              height: 220,
              decoration: BoxDecoration(
                color: kCard,
                borderRadius: BorderRadius.circular(12),
                border: Border.all(color: kPrimary.withValues(alpha: 0.4), width: 2),
              ),
              child: _fotoQris.isNotEmpty
                ? Stack(children: [
                    ClipRRect(
                      borderRadius: BorderRadius.circular(10),
                      child: Image.memory(base64Decode(_fotoQris), width: double.infinity, height: 220, fit: BoxFit.contain),
                    ),
                    Positioned(bottom: 8, right: 8,
                      child: Container(
                        padding: const EdgeInsets.symmetric(horizontal: 10, vertical: 6),
                        decoration: BoxDecoration(color: kPrimary, borderRadius: BorderRadius.circular(20)),
                        child: Row(mainAxisSize: MainAxisSize.min, children: [
                          const Icon(Icons.camera_alt, color: Colors.white, size: 14),
                          const SizedBox(width: 4),
                          Text(p._t('Ganti Foto','Change Photo','تغيير الصورة'), style: const TextStyle(color: Colors.white, fontSize: 12)),
                        ]),
                      ),
                    ),
                  ])
                : Column(mainAxisAlignment: MainAxisAlignment.center, children: [
                    Icon(Icons.qr_code, color: kPrimary.withValues(alpha: 0.4), size: 64),
                    const SizedBox(height: 10),
                    Text(p._t('Tap untuk upload foto QRIS','Tap to upload QRIS photo','اضغط لرفع صورة QRIS'),
                      style: TextStyle(color: kPrimary.withValues(alpha: 0.6), fontSize: 14)),
                    const SizedBox(height: 4),
                    Text(p._t('JPG / PNG dari screenshot QRIS bank','JPG / PNG from QRIS bank screenshot','JPG / PNG من لقطة شاشة QRIS'),
                      style: TextStyle(color: kTextGrey, fontSize: 11)),
                  ]),
            ),
          ),
          const SizedBox(height: 20),

          // Info rekening
          Consumer<AppProvider>(builder: (_, p2, __) => Text(p2._t('INFO REKENING','BANK ACCOUNT','معلومات الحساب'), style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 12, color: kTextGrey, letterSpacing: 1))),
          const SizedBox(height: 8),
          TextField(
            controller: _bankCtrl,
            textCapitalization: TextCapitalization.words,
            onChanged: (_) => setState(() => _saved = false),
            decoration: InputDecoration(
              labelText: p._t('Nama Bank / Dompet Digital','Bank / E-Wallet Name','اسم البنك / المحفظة'),
              hintText: 'BRI, BCA, Dana, GoPay, OVO...',
              prefixIcon: Icon(Icons.account_balance, color: kPrimary, size: 20),
              filled: true, fillColor: kCard,
              border: OutlineInputBorder(borderRadius: BorderRadius.circular(10)),
            ),
          ),
          const SizedBox(height: 12),
          TextField(
            controller: _norekCtrl,
            keyboardType: TextInputType.number,
            onChanged: (_) => setState(() => _saved = false),
            decoration: InputDecoration(
              labelText: p._t('Nomor Rekening','Account Number','رقم الحساب'),
              hintText: '1234-5678-9012',
              prefixIcon: Icon(Icons.credit_card, color: kPrimary, size: 20),
              filled: true, fillColor: kCard,
              border: OutlineInputBorder(borderRadius: BorderRadius.circular(10)),
            ),
          ),
          const SizedBox(height: 12),
          TextField(
            controller: _pemilikCtrl,
            textCapitalization: TextCapitalization.words,
            onChanged: (_) => setState(() => _saved = false),
            decoration: InputDecoration(
              labelText: p._t('Nama Pemilik Rekening','Account Owner Name','اسم صاحب الحساب'),
              hintText: p._t('DKM Masjid Al-Ikhlas','e.g. DKM Al-Ikhlas Mosque','مثال: اسم المسجد'),
              prefixIcon: Icon(Icons.person, color: kPrimary, size: 20),
              filled: true, fillColor: kCard,
              border: OutlineInputBorder(borderRadius: BorderRadius.circular(10)),
            ),
          ),
          const SizedBox(height: 24),

          // Preview
          if (_fotoQris.isNotEmpty || _norekCtrl.text.isNotEmpty) ...[
            Consumer<AppProvider>(builder: (_, p2, __) => Text(p2._t('PREVIEW SLIDE LIVE DISPLAY','LIVE DISPLAY PREVIEW','معاينة العرض المباشر'), style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 12, color: kTextGrey, letterSpacing: 1))),
            const SizedBox(height: 8),
            Container(
              width: double.infinity,
              padding: const EdgeInsets.all(16),
              decoration: BoxDecoration(
                gradient: const LinearGradient(
                  colors: [Color(0xFF1b5e20), Color(0xFF388e3c)],
                  begin: Alignment.topLeft, end: Alignment.bottomRight,
                ),
                borderRadius: BorderRadius.circular(12),
              ),
              child: Column(children: [
                Consumer<AppProvider>(builder: (_, p, __) => Text(p._t('DONASI & INFAQ','DONATION & INFAQ','التبرع والإنفاق'), style: const TextStyle(color: Colors.white70, fontSize: 11, letterSpacing: 2))),
                const SizedBox(height: 8),
                if (_fotoQris.isNotEmpty)
                  Container(
                    width: 120, height: 120,
                    decoration: BoxDecoration(color: Colors.white, borderRadius: BorderRadius.circular(10)),
                    padding: const EdgeInsets.all(6),
                    child: Image.memory(base64Decode(_fotoQris), fit: BoxFit.contain),
                  ),
                const SizedBox(height: 10),
                if (_bankCtrl.text.isNotEmpty)
                  Text(_bankCtrl.text.toUpperCase(),
                    style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold, fontSize: 13)),
                if (_norekCtrl.text.isNotEmpty)
                  Text(_norekCtrl.text,
                    style: const TextStyle(color: Colors.amber, fontWeight: FontWeight.bold, fontSize: 16, letterSpacing: 2)),
                if (_pemilikCtrl.text.isNotEmpty)
                  Consumer<AppProvider>(builder: (_, p2, __) => Text('${p2._t('a.n.','a/n','بـ')} ${_pemilikCtrl.text}',
                    style: const TextStyle(color: Colors.white70, fontSize: 11))),
              ]),
            ),
            const SizedBox(height: 20),
          ],

          // Tombol simpan — selalu tampil
          const SizedBox(height: 8),
          SizedBox(
            width: double.infinity,
            child: ElevatedButton.icon(
              style: ElevatedButton.styleFrom(
                backgroundColor: _saved ? Colors.green : kPrimary,
                foregroundColor: Colors.white,
                padding: const EdgeInsets.symmetric(vertical: 16),
                shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(10)),
              ),
              icon: Icon(_saved ? Icons.check_circle : Icons.save, size: 22),
              label: Consumer<AppProvider>(builder: (_, p2, __) => Text(
                _saved ? p2._t('TERSIMPAN ✓','SAVED ✓','تم الحفظ ✓') : p2._t('SIMPAN','SAVE','حفظ'),
                style: const TextStyle(fontSize: 16, fontWeight: FontWeight.bold))),
              onPressed: _simpan,
            ),
          ),
          const SizedBox(height: 20),
        ]),
      ),
    )
    );
  }
}

class BackupPage extends StatefulWidget {
  const BackupPage({super.key});
  @override
  State<BackupPage> createState() => _BackupPageState();
}

class _BackupPageState extends State<BackupPage> {
  bool _loading = false;
  String _status = '';
  String _lastRestorePath = '';

  Future<void> _backup(AppProvider p) async {
    setState(() { _loading = true; _status = ''; });
    final path = await p.simpanBackup();
    if (!mounted) return;
    setState(() {
      _loading = false;
      _status  = path.isEmpty
          ? p._t('Gagal backup!','Backup failed!','فشل النسخ الاحتياطي!')
          : '${p._t('Tersimpan','Saved','محفوظ')}:\n$path';
    });
  }

  Future<void> _restore(AppProvider p) async {
    final ctrl = TextEditingController();
    final ok = await showDialog<bool>(
      context: context,
      builder: (_) => AlertDialog(
        title: Text(p._t('Restore Backup','Restore Backup','استعادة النسخ الاحتياطي')),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Text(
              p._t(
                'Masukkan path file backup .json\nContoh: /storage/emulated/0/novapro_backup_xxx.json',
                'Enter the backup file path .json\nExample: /storage/emulated/0/novapro_backup_xxx.json',
                'أدخل مسار ملف النسخ الاحتياطي .json\nمثال: /storage/emulated/0/novapro_backup_xxx.json',
              ),
              style: const TextStyle(fontSize: 12, color: Colors.grey),
            ),
            const SizedBox(height: 8),
            TextField(
              controller: ctrl,
              decoration: InputDecoration(
                labelText: p._t('Path File Backup','Backup File Path','مسار ملف النسخ الاحتياطي'),
                isDense: true,
              ),
            ),
          ],
        ),
        actions: [
          TextButton(
            onPressed: () { Navigator.pop(context, false); ctrl.dispose(); },
            child: Text(p.lBatal),
          ),
          ElevatedButton(
            style: ElevatedButton.styleFrom(
                backgroundColor: const Color(0xFF631414),
                foregroundColor: Colors.white),
            onPressed: () {
              final path = ctrl.text.trim();
              _lastRestorePath = path;
              Navigator.pop(context, path.isNotEmpty);
              ctrl.dispose();
            },
            child: Text(p._t('RESTORE','RESTORE','استعادة')),
          ),
        ],
      ),
    );

    if (ok == true) {
      setState(() { _loading = true; _status = ''; });
      try {
        final filePath = _lastRestorePath;
        final file = File(filePath.isEmpty ? '/storage/emulated/0/' : filePath);
        if (await file.exists()) {
          final json = await file.readAsString();
          final berhasil = await p.importBackup(json);
          if (!mounted) return;
          setState(() {
            _loading = false;
            _status  = berhasil
                ? p._t('Restore berhasil! Semua data telah dipulihkan.','Restore successful! All data has been recovered.','تمت الاستعادة بنجاح! تم استرداد جميع البيانات.')
                : p._t('Gagal restore! Format file tidak valid.','Restore failed! Invalid file format.','فشل الاستعادة! تنسيق الملف غير صالح.');
          });
        } else {
          setState(() { _loading = false; _status = p._t('File tidak ditemukan!','File not found!','الملف غير موجود!'); });
        }
      } catch (e) {
        setState(() { _loading = false; _status = 'Error: $e'; });
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    final p = context.watch<AppProvider>();
    return Scaffold(
      appBar: AppBar(title: Text(p._t('BACKUP & RESTORE','BACKUP & RESTORE','النسخ الاحتياطي والاستعادة'))),
      body: ZoomWrapper(child: SingleChildScrollView(
        padding: const EdgeInsets.all(20),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            Container(
              padding: const EdgeInsets.all(14),
              decoration: BoxDecoration(
                color: Colors.blue.shade50,
                borderRadius: BorderRadius.circular(10),
                border: Border.all(color: Colors.blue.shade200),
              ),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(p._t('INFO BACKUP','BACKUP INFO','معلومات النسخ الاحتياطي'),
                      style: const TextStyle(fontWeight: FontWeight.bold, color: Colors.blue)),
                  const SizedBox(height: 6),
                  Text(
                    p._t(
                      '• Backup menyimpan SEMUA data: profil, transaksi, jadwal, galeri\n'
                      '• File backup kecil (~beberapa KB)\n'
                      '• Simpan file backup ke Flashdisk atau kirim via WA\n'
                      '• Restore bisa dilakukan di HP manapun yang punya NovaPro',
                      '• Backup saves ALL data: profile, transactions, schedule, gallery\n'
                      '• Backup file is small (~few KB)\n'
                      '• Save backup file to USB or send via WhatsApp\n'
                      '• Restore can be done on any phone with NovaPro',
                      '• النسخ الاحتياطي يحفظ جميع البيانات: الملف، المعاملات، الجدول، المعرض\n'
                      '• حجم ملف النسخ الاحتياطي صغير (~بضعة KB)\n'
                      '• احفظ الملف في USB أو أرسله عبر واتساب\n'
                      '• يمكن الاستعادة على أي جهاز لديه NovaPro',
                    ),
                    style: const TextStyle(fontSize: 12),
                  ),
                ],
              ),
            ),
            const SizedBox(height: 24),

            ElevatedButton.icon(
              style: ElevatedButton.styleFrom(
                backgroundColor: const Color(0xFF631414),
                foregroundColor: Colors.white,
                padding: const EdgeInsets.symmetric(vertical: 16),
              ),
              icon: const Icon(Icons.backup),
              label: Text(p._t('BACKUP SEKARANG','BACKUP NOW','نسخ احتياطي الآن'),
                  style: const TextStyle(fontSize: 16)),
              onPressed: _loading ? null : () => _backup(p),
            ),
            const SizedBox(height: 12),

            OutlinedButton.icon(
              style: OutlinedButton.styleFrom(
                side: const BorderSide(color: Color(0xFF631414)),
                foregroundColor: const Color(0xFF631414),
                padding: const EdgeInsets.symmetric(vertical: 16),
              ),
              icon: const Icon(Icons.restore),
              label: Text(p._t('RESTORE DARI FILE','RESTORE FROM FILE','استعادة من ملف'),
                  style: const TextStyle(fontSize: 16)),
              onPressed: _loading ? null : () => _restore(p),
            ),
            const SizedBox(height: 24),

            if (_loading)
              const Center(child: CircularProgressIndicator()),
            if (_status.isNotEmpty)
              Container(
                padding: const EdgeInsets.all(12),
                decoration: BoxDecoration(
                  color: _status.contains('Gagal') || _status.contains('Error') || _status.contains('failed') || _status.contains('Failed')
                      ? Colors.red.shade50 : Colors.green.shade50,
                  borderRadius: BorderRadius.circular(8),
                  border: Border.all(
                    color: _status.contains('Gagal') || _status.contains('Error') || _status.contains('failed') || _status.contains('Failed')
                        ? Colors.red : Colors.green,
                  ),
                ),
                child: Text(
                  _status,
                  style: TextStyle(
                    color: _status.contains('Gagal') || _status.contains('Error') || _status.contains('failed') || _status.contains('Failed')
                        ? Colors.red.shade800 : Colors.green.shade800,
                    fontWeight: FontWeight.bold,
                  ),
                ),
              ),

            const SizedBox(height: 12),

            Center(
              child: Text(
                '${p._t('Kode Perangkat','Device Code','رمز الجهاز')}: ${p.houseUniqueCode}',
                style: const TextStyle(color: Colors.grey, fontSize: 12),
              ),
            ),
          ],
        ),
      ),
    )
    );
  }
}

class PengaturanPage extends StatefulWidget {
  const PengaturanPage({super.key});
  @override
  State<PengaturanPage> createState() => _PengaturanPageState();
}

class _PengaturanPageState extends State<PengaturanPage> {
  @override
  Widget build(BuildContext context) {
    final p = context.watch<AppProvider>();
    return Scaffold(
      appBar: AppBar(title: Text(p.lPengaturan)),
      body: ZoomWrapper(child: ListView(
        children: [

          _sectionHeader(p._t('RAMADHAN','RAMADAN','رمضان')),
          SwitchListTile(
            secondary: const Icon(Icons.nightlight_round, color: kPrimary),
            title: Text(p._t('Mode Ramadhan','Ramadan Mode','وضع رمضان')),
            subtitle: Text(p._t(
              'Aktifkan untuk tampilkan slide jadwal imsak di Live Display',
              'Enable to show imsak schedule slide on Live Display',
              'تفعيل لعرض شريحة جدول الإمساك في العرض المباشر',
            )),
            value: p.aktifRamadhan,
            activeColor: kPrimary,
            onChanged: (val) => p.setRamadhan(val),
          ),
          if (p.aktifRamadhan) ...[
            ListTile(
              leading: const Icon(Icons.access_time, color: kPrimary),
              title: Text(p._t('Menit Imsak Sebelum Subuh','Imsak Minutes Before Fajr','دقائق الإمساك قبل الفجر')),
              subtitle: Text(p._t(
                'Imsak ${p.menitSebelumSubuh} menit sebelum Subuh (${p.waktuImsak})',
                'Imsak ${p.menitSebelumSubuh} min before Fajr (${p.waktuImsak})',
                'الإمساك ${p.menitSebelumSubuh} دقيقة قبل الفجر (${p.waktuImsak})',
              )),
              trailing: Row(mainAxisSize: MainAxisSize.min, children: [
                IconButton(
                  icon: const Icon(Icons.remove_circle_outline),
                  onPressed: () => p.setRamadhan(true, menit: (p.menitSebelumSubuh - 1).clamp(5, 30)),
                ),
                Text('${p.menitSebelumSubuh}m', style: const TextStyle(fontWeight: FontWeight.bold)),
                IconButton(
                  icon: const Icon(Icons.add_circle_outline),
                  onPressed: () => p.setRamadhan(true, menit: (p.menitSebelumSubuh + 1).clamp(5, 30)),
                ),
              ]),
            ),
          ],
          const Divider(),

          _sectionHeader(p._t('PERAWATAN APLIKASI','APP MAINTENANCE','صيانة التطبيق')),
          ListTile(
            leading: const Icon(Icons.cleaning_services, color: Colors.orange),
            title: Text(p._t('Bersihkan Cache','Clear Cache','مسح ذاكرة التخزين')),
            subtitle: Text(p._t('Hapus cache gambar sementara','Clear temporary image cache','مسح ذاكرة الصور المؤقتة')),
            trailing: const Icon(Icons.chevron_right),
            onTap: () {
              PaintingBinding.instance.imageCache.clear();
              PaintingBinding.instance.imageCache.clearLiveImages();
              ScaffoldMessenger.of(context).showSnackBar(SnackBar(
                content: Text(p._t(
                  'Cache berhasil dibersihkan!',
                  'Cache cleared!',
                  'تم مسح ذاكرة التخزين!')),
                backgroundColor: Colors.orange.shade700,
                duration: const Duration(seconds: 2),
              ));
            },
          ),
          const Divider(),

          _sectionHeader(p._t('JADWAL & WAKTU SHOLAT','PRAYER SCHEDULE','جدول الصلاة')),
          ListTile(
            title: Text(p.lKoreksiMenit),
            subtitle: Text(p._t('Tambah ${p.koreksiMenit} menit dari waktu standar','Add ${p.koreksiMenit} min from standard time','أضف ${p.koreksiMenit} دقيقة من الوقت المعياري')),
            trailing: Row(
              mainAxisSize: MainAxisSize.min,
              children: [
                IconButton(
                  icon: const Icon(Icons.remove_circle_outline),
                  onPressed: () => p.updateKoreksi(
                      (p.koreksiMenit - 1).clamp(0, 10), p.menitIqomah),
                ),
                Text('${p.koreksiMenit}${p._t('m','m','د')}',
                    style: const TextStyle(fontWeight: FontWeight.bold)),
                IconButton(
                  icon: const Icon(Icons.add_circle_outline),
                  onPressed: () => p.updateKoreksi(
                      (p.koreksiMenit + 1).clamp(0, 10), p.menitIqomah),
                ),
              ],
            ),
          ),
          ListTile(
            title: Text(p._t('Hitung Mundur Iqomah','Iqamah Countdown','عد تنازلي للإقامة')),
            subtitle: Text(p._t('Mulai hitung ${p.menitIqomah} menit sebelum waktu','Start ${p.menitIqomah} min before prayer time','ابدأ ${p.menitIqomah} دقيقة قبل وقت الصلاة')),
            trailing: Row(
              mainAxisSize: MainAxisSize.min,
              children: [
                IconButton(
                  icon: const Icon(Icons.remove_circle_outline),
                  onPressed: () => p.updateKoreksi(
                      p.koreksiMenit, (p.menitIqomah - 1).clamp(1, 30)),
                ),
                Text('${p.menitIqomah}${p._t('m','m','د')}',
                    style: const TextStyle(fontWeight: FontWeight.bold)),
                IconButton(
                  icon: const Icon(Icons.add_circle_outline),
                  onPressed: () => p.updateKoreksi(
                      p.koreksiMenit, (p.menitIqomah + 1).clamp(1, 30)),
                ),
              ],
            ),
          ),

          const Divider(),
          _sectionHeader(p.lBahasa.toUpperCase()),
          RadioListTile<String>(
            title: const Text('Bahasa Indonesia'),
            value: 'id',
            groupValue: p.bahasa,
            onChanged: (v) {
              p.setBahasa(v!);
              // Navigate ulang agar semua halaman rebuild dengan bahasa baru
              Future.delayed(const Duration(milliseconds: 100), () {
                if (!context.mounted) return;
                Navigator.of(context).pushAndRemoveUntil(
                  MaterialPageRoute(builder: (_) => const AdminShell()),
                  (route) => false,
                );
              });
            },
          ),
          RadioListTile<String>(
            title: const Text('English'),
            value: 'en',
            groupValue: p.bahasa,
            onChanged: (v) {
              p.setBahasa(v!);
              Future.delayed(const Duration(milliseconds: 100), () {
                if (!context.mounted) return;
                Navigator.of(context).pushAndRemoveUntil(
                  MaterialPageRoute(builder: (_) => const AdminShell()),
                  (route) => false,
                );
              });
            },
          ),
          RadioListTile<String>(
            title: const Text('عربي (Arab)'),
            value: 'ar',
            groupValue: p.bahasa,
            onChanged: (v) {
              p.setBahasa(v!);
              Future.delayed(const Duration(milliseconds: 100), () {
                if (!context.mounted) return;
                Navigator.of(context).pushAndRemoveUntil(
                  MaterialPageRoute(builder: (_) => const AdminShell()),
                  (route) => false,
                );
              });
            },
          ),

          const Divider(),
          _sectionHeader(p._t('SMART DISPLAY (LAYAR TV)','SMART DISPLAY (TV SCREEN)','العرض الذكي (شاشة TV)')),
          SwitchListTile(
            title: Text(p._t('Smart Display Aktif','Smart Display Active','تفعيل العرض الذكي')),
            subtitle: Text(p._t('Layar otomatis tampil pesan sholat saat adzan','Screen auto shows prayer message at adhan time','الشاشة تعرض رسالة الصلاة عند الأذان تلقائياً')),
            value: p.smartDisplayAktif,
            onChanged: (_) => p.toggleSmartDisplay(),
          ),
          ListTile(
            title: Text(p._t('Durasi Tampilan Adzan','Adhan Display Duration','مدة عرض الأذان')),
            subtitle: Text(p._t('Layar Luruskan Shaf tayang ${p.durasiAdzanMenit} menit lalu otomatis kembali normal','Straighten rows screen shows for ${p.durasiAdzanMenit} min then auto returns','شاشة استووا تعمل ${p.durasiAdzanMenit} دقيقة ثم تعود تلقائياً')),
            trailing: Row(mainAxisSize: MainAxisSize.min, children: [
              IconButton(
                icon: const Icon(Icons.remove_circle_outline),
                onPressed: () => p.setDurasiAdzan(p.durasiAdzanMenit - 1),
              ),
              Text('${p.durasiAdzanMenit}${p._t('m','m','د')}',
                style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 16)),
              IconButton(
                icon: const Icon(Icons.add_circle_outline),
                onPressed: () => p.setDurasiAdzan(p.durasiAdzanMenit + 1),
              ),
            ]),
          ),
          SwitchListTile(
            title: Text(p._t('Simulasi Mode Adzan','Adhan Mode Simulation','محاكاة وضع الأذان')),
            subtitle: Text(p._t('Test tampilan — otomatis mati dalam ${p.durasiAdzanMenit} menit','Test display — auto off in ${p.durasiAdzanMenit} min','اختبار العرض — إيقاف تلقائي خلال ${p.durasiAdzanMenit} دقيقة')),
            value: p.sedangAdzan,
            onChanged: (v) => p.setSedangAdzan(v),
          ),
          if (p.sedangAdzan)
            Container(
              margin: const EdgeInsets.fromLTRB(16, 0, 16, 12),
              decoration: BoxDecoration(
                color: Colors.black,
                borderRadius: BorderRadius.circular(12),
                border: Border.all(color: kPrimary, width: 2),
              ),
              child: Column(children: [
                Container(
                  width: double.infinity,
                  padding: const EdgeInsets.symmetric(vertical: 6),
                  decoration: const BoxDecoration(
                    color: kPrimary,
                    borderRadius: BorderRadius.vertical(top: Radius.circular(10)),
                  ),
                  child: Row(mainAxisAlignment: MainAxisAlignment.center, children: [
                    const Icon(Icons.tv, color: Colors.white, size: 14),
                    const SizedBox(width: 6),
                    Text(p._t('PREVIEW LAYAR TV SAAT ADZAN','TV PREVIEW DURING ADHAN','معاينة شاشة TV أثناء الأذان'),
                      style: const TextStyle(color: Colors.white, fontSize: 11, fontWeight: FontWeight.bold, letterSpacing: 1)),
                  ]),
                ),
                Padding(
                  padding: const EdgeInsets.symmetric(vertical: 24, horizontal: 16),
                  child: Column(children: [
                    const Icon(Icons.mosque, color: kPrimary, size: 52),
                    const SizedBox(height: 14),
                    Text(p._t('LURUSKAN DAN RAPATKAN SHAF','STRAIGHTEN AND CLOSE THE ROWS','استووا واعتدلوا الصفوف'),
                      textAlign: TextAlign.center,
                      style: const TextStyle(color: Colors.white, fontSize: 16, fontWeight: FontWeight.bold)),
                    const SizedBox(height: 8),
                    Text(p._t('MATIKAN HANDPHONE','SILENCE YOUR PHONE','أوقف تشغيل الهاتف'),
                      style: const TextStyle(color: Colors.white70, fontSize: 13)),
                    const SizedBox(height: 16),
                    Container(
                      padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 6),
                      decoration: BoxDecoration(
                        border: Border.all(color: Colors.white24),
                        borderRadius: BorderRadius.circular(6),
                      ),
                      child: Text(p._t('Tampilan ini muncul otomatis\ndi layar TV saat waktu sholat tiba','This display appears automatically\non TV when prayer time arrives','يظهر هذا العرض تلقائياً\nعلى شاشة TV عند دخول وقت الصلاة'),
                        textAlign: TextAlign.center,
                        style: const TextStyle(color: Colors.white38, fontSize: 10)),
                    ),
                  ]),
                ),
                Padding(
                  padding: const EdgeInsets.fromLTRB(16, 0, 16, 12),
                  child: SizedBox(
                    width: double.infinity,
                    child: OutlinedButton.icon(
                      style: OutlinedButton.styleFrom(
                        foregroundColor: Colors.white54,
                        side: const BorderSide(color: Colors.white24),
                      ),
                      icon: const Icon(Icons.stop_circle_outlined, size: 16),
                      label: Text(p._t('Matikan Simulasi','Stop Simulation','إيقاف المحاكاة'), style: const TextStyle(fontSize: 12)),
                      onPressed: () => p.setSedangAdzan(false),
                    ),
                  ),
                ),
              ]),
            ),

          const Divider(),
          _sectionHeader(p._t('RUNNING TEXT','RUNNING TEXT','النص المتحرك')),
          ListTile(
            title: Text(p._t('Kecepatan','Speed','السرعة')),
            subtitle: Slider(
              value: p.kecepatanRunningText,
              min: 5,
              max: 60,
              divisions: 11,
              label: p.kecepatanRunningText <= 10
                  ? p._t('Sangat Cepat','Very Fast','سريع جداً')
                  : p.kecepatanRunningText <= 20
                      ? p._t('Cepat','Fast','سريع')
                      : p.kecepatanRunningText <= 35
                          ? p._t('Normal','Normal','عادي')
                          : p._t('Lambat','Slow','بطيء'),
              onChanged: (v) =>
                  p.updateRunningTextSettings(v, p.warnaRunningText),
            ),
          ),
          ListTile(
            title: Text(p._t('Warna Teks','Text Color','لون النص')),
            subtitle: Text(p._t('Pilih warna teks running text','Choose running text color','اختر لون النص المتحرك')),
            trailing: Row(
              mainAxisSize: MainAxisSize.min,
              children: [
                _warnaBtn(p, 'FFFFFF', p._t('Putih','White','أبيض'),  Colors.white),
                _warnaBtn(p, 'FFFF00', p._t('Kuning','Yellow','أصفر'), Colors.yellow),
                _warnaBtn(p, '00FF00', p._t('Hijau','Green','أخضر'),  Colors.green),
                _warnaBtn(p, 'FF6600', p._t('Oranye','Orange','برتقالي'), Colors.orange),
              ],
            ),
          ),

          const Divider(),
          _sectionHeader(p._t('LISENSI & KEAMANAN','LICENSE & SECURITY','الترخيص والأمان')),
          ListTile(
            title: Text(p._t('Kode Unik Perangkat','Device Unique Code','رمز الجهاز الفريد')),
            subtitle: Text(p.houseUniqueCode,
                style: const TextStyle(
                    fontWeight: FontWeight.bold, fontSize: 16)),
            trailing: IconButton(
              icon: const Icon(Icons.refresh),
              tooltip: p._t('Generate ulang kode','Regenerate code','إعادة توليد الرمز'),
              onPressed: () => showDialog(
                context: context,
                builder: (_) => AlertDialog(
                  title: Text(p._t('Generate Ulang Kode?','Regenerate Code?','إعادة توليد الرمز؟')),
                  content: Text(p._t(
                      'Kode lama akan terganti. Pastikan catat kode baru untuk verifikasi.',
                      'Old code will be replaced. Make sure to note the new code.',
                      'سيتم استبدال الرمز القديم. تأكد من تدوين الرمز الجديد.')),
                  actions: [
                    TextButton(
                        onPressed: () => Navigator.pop(context),
                        child: Text(p.lBatal)),
                    ElevatedButton(
                      onPressed: () {
                        p.generateUniqueCode();
                        Navigator.pop(context);
                      },
                      child: Text(p._t('GENERATE','GENERATE','توليد')),
                    ),
                  ],
                ),
              ),
            ),
          ),
          ListTile(
            leading: const Icon(Icons.verified_user, color: Colors.green),
            title: Text(p._t('Status Lisensi','License Status','حالة الترخيص')),
            subtitle: Text(p._t('AKTIF - Sekali Beli Full Fitur','ACTIVE - One-time Purchase Full Features','نشط - شراء مرة واحدة كامل الميزات')),
            trailing: const Icon(Icons.check_circle, color: Colors.green),
          ),

          const SizedBox(height: 20),
        ],
      ),
    )
    );
  }

  Widget _sectionHeader(String title) => Padding(
        padding: const EdgeInsets.fromLTRB(16, 12, 16, 4),
        child: Text(title,
            style: const TextStyle(
                color: Color(0xFF631414),
                fontWeight: FontWeight.bold,
                fontSize: 12)),
      );

  Widget _warnaBtn(AppProvider p, String hex, String label, Color color) =>
      GestureDetector(
        onTap: () => p.updateRunningTextSettings(p.kecepatanRunningText, hex),
        child: Container(
          margin: const EdgeInsets.symmetric(horizontal: 2),
          width: 24,
          height: 24,
          decoration: BoxDecoration(
            color: color,
            shape: BoxShape.circle,
            border: Border.all(
              color: p.warnaRunningText == hex
                  ? const Color(0xFF631414)
                  : Colors.grey,
              width: p.warnaRunningText == hex ? 2.5 : 1,
            ),
          ),
        ),
      );
}

// Widget untuk tampil gambar galeri - support web dan Android
class _GaleriImageWidget extends StatelessWidget {
  final String path;
  const _GaleriImageWidget({required this.path});

  @override
  Widget build(BuildContext context) {
    if (path.startsWith('data:image')) {
      final b64 = path.split(',').last;
      return Image.memory(base64Decode(b64), fit: BoxFit.cover,
        errorBuilder: (_, __, ___) => Container(color: Colors.grey.shade300,
          child: const Icon(Icons.broken_image, color: Colors.grey)));
    }
    if (kIsWeb) {
      return Container(color: Colors.grey.shade200,
        child: const Icon(Icons.image, color: Colors.grey, size: 32));
    }
    return Image.file(File(path), fit: BoxFit.cover,
      errorBuilder: (_, __, ___) => Container(color: Colors.grey.shade300,
        child: const Icon(Icons.broken_image, color: Colors.grey)));
  }
}

class GaleriPage extends StatefulWidget {
  final VoidCallback? onBack;
  const GaleriPage({super.key, this.onBack});
  @override
  State<GaleriPage> createState() => _GaleriPageState();
}

class _GaleriPageState extends State<GaleriPage> {
  final _judulCtrl  = TextEditingController();
  final _durasiCtrl = TextEditingController();

  @override
  void dispose() {
    _judulCtrl.dispose();
    _durasiCtrl.dispose();
    super.dispose();
  }

  void _viewMedia(Map<String, dynamic> item) {
    final isImage = item['type'] == 'image';
    final path = item['path'] as String;
    final judul = item['judul'] ?? '';
    final tglRaw3 = item['tgl'] ?? '';
    final tgl = tglRaw3.isNotEmpty ? context.read<AppProvider>().formatTgl(tglRaw3) : '';

    showDialog(
      context: context,
      barrierColor: Colors.black87,
      builder: (_) => Dialog(
        backgroundColor: Colors.black,
        insetPadding: EdgeInsets.zero,
        child: Stack(children: [
          // Konten foto atau video
          Center(
            child: isImage
                ? InteractiveViewer(
                    child: Image.file(File(path), fit: BoxFit.contain,
                      errorBuilder: (_, __, ___) => const Icon(Icons.broken_image, color: Colors.white, size: 80)),
                  )
                : _VideoPlayerFullScreen(path: path),
          ),
          // Info judul dan tanggal di bawah
          if (judul.isNotEmpty || tgl.isNotEmpty)
            Positioned(
              bottom: 0, left: 0, right: 0,
              child: Container(
                padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
                decoration: const BoxDecoration(
                  gradient: LinearGradient(
                    begin: Alignment.bottomCenter, end: Alignment.topCenter,
                    colors: [Colors.black87, Colors.transparent],
                  ),
                ),
                child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                  if (judul.isNotEmpty)
                    Text(judul, style: const TextStyle(color: Colors.white, fontSize: 15, fontWeight: FontWeight.bold)),
                  if (tgl.isNotEmpty)
                    Text(tgl, style: const TextStyle(color: Colors.white70, fontSize: 12)),
                ]),
              ),
            ),
          // Tombol tutup
          Positioned(
            top: 8, right: 8,
            child: IconButton(
              icon: const Icon(Icons.close, color: Colors.white, size: 28),
              onPressed: () => Navigator.pop(context),
            ),
          ),
        ]),
      ),
    );
  }

  void _showPickerDialog(AppProvider p) {
    _judulCtrl.clear(); // Reset judul setiap buka dialog
    String tipe = 'image';
    String base64Data = '';
    String namaFile = '';

    showDialog(
      context: context,
      builder: (_) => StatefulBuilder(
        builder: (ctx, setLocal) => AlertDialog(
          title: Text(p.lTambahMedia),
          content: SingleChildScrollView(
            child: Column(
              mainAxisSize: MainAxisSize.min,
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Wrap(spacing: 8, crossAxisAlignment: WrapCrossAlignment.center, children: [
                  Text("${p._t('Tipe','Type','النوع')}: "),
                  ChoiceChip(
                    label: Text(p._t('Foto','Photo','صورة')),
                    selected: tipe == 'image',
                    onSelected: (_) => setLocal(() { tipe = 'image'; base64Data = ''; namaFile = ''; }),
                  ),
                  ChoiceChip(
                    label: Text(p._t('Video','Video','فيديو')),
                    selected: tipe == 'video',
                    onSelected: (_) => setLocal(() { tipe = 'video'; base64Data = ''; namaFile = ''; }),
                  ),
                ]),
                const SizedBox(height: 16),
                GestureDetector(
                  onTap: () async {
                    try {
                      XFile? picked;
                      if (tipe == 'image') {
                        picked = await ImagePicker().pickImage(source: ImageSource.gallery, imageQuality: 70);
                      } else {
                        picked = await ImagePicker().pickVideo(source: ImageSource.gallery);
                      }
                      if (picked == null) return;
                      // Validasi ukuran file max 50MB
                      final fileSize = await picked.length();
                      if (fileSize > 50 * 1024 * 1024) {
                        if (ctx.mounted) ScaffoldMessenger.of(ctx).showSnackBar(
                          SnackBar(content: Text(p._t('File terlalu besar! Maksimal 50MB.','File too large! Max 50MB.','الملف كبير جداً! الحد الأقصى 50 ميغابايت.'))));
                        return;
                      }
                      setLocal(() { base64Data = picked!.path; namaFile = picked.name; });
                    } catch (e) {
                      debugPrint('Error picking file: $e');
                      if (ctx.mounted) ScaffoldMessenger.of(ctx).showSnackBar(
                        SnackBar(content: Text(p._t('Gagal memuat file:','Failed to load file:','فشل تحميل الملف:') + ' $e')));
                    }
                  },
                  child: Container(
                    width: double.infinity,
                    padding: const EdgeInsets.all(16),
                    decoration: BoxDecoration(
                      border: Border.all(color: base64Data.isEmpty ? Colors.grey : kPrimary),
                      borderRadius: BorderRadius.circular(8),
                      color: base64Data.isEmpty ? Colors.grey.shade100 : kPrimary.withAlpha(13),
                    ),
                    child: Row(children: [
                      Icon(tipe == 'image' ? Icons.photo : Icons.videocam,
                          color: base64Data.isEmpty ? Colors.grey : kPrimary),
                      const SizedBox(width: 12),
                      Expanded(child: Text(
                        base64Data.isEmpty
                            ? p._t('Klik untuk pilih ${tipe == "image" ? "foto" : "video"}','Click to select ${tipe == "image" ? "photo" : "video"}','اضغط لاختيار ${tipe == "image" ? "صورة" : "فيديو"}')
                            : namaFile,
                        style: TextStyle(color: base64Data.isEmpty ? Colors.grey : kPrimary, fontSize: 13),
                        overflow: TextOverflow.ellipsis,
                      )),
                      if (base64Data.isNotEmpty)
                        const Icon(Icons.check_circle, color: Colors.green, size: 18),
                    ]),
                  ),
                ),
                const SizedBox(height: 12),
                TextField(
                  controller: _judulCtrl,
                  maxLength: 200,
                  maxLines: 2,
                  decoration: InputDecoration(
                    labelText: p._t('Judul / Keterangan','Title / Caption','العنوان / الوصف'),
                    hintText: p._t('Tulis keterangan foto/video (maks 200 karakter)...','Write photo/video caption (max 200 chars)...','اكتب وصف الصورة/الفيديو (الحد الأقصى 200 حرف)...'),
                    isDense: true,
                  ),
                ),
                const SizedBox(height: 8),
                Text(
                  tipe == 'image'
                      ? p._t('Durasi slide: ${p.durasiSlideDefault} detik','Slide duration: ${p.durasiSlideDefault} sec','مدة الشريحة: ${p.durasiSlideDefault} ثانية')
                      : p._t('Video: akan diputar otomatis','Video: will play automatically','فيديو: سيُشغَّل تلقائياً'),
                  style: const TextStyle(color: Colors.grey, fontSize: 12),
                ),
              ],
            ),
          ),
          actions: [
            TextButton(
              onPressed: () { Navigator.pop(ctx); },
              child: Text(p.lBatal),
            ),
            ElevatedButton(
              style: ElevatedButton.styleFrom(
                  backgroundColor: const Color(0xFF631414),
                  foregroundColor: Colors.white),
              onPressed: () {
                if (base64Data.isEmpty) {
                  ScaffoldMessenger.of(ctx).showSnackBar(
                      SnackBar(content: Text(p._t('Pilih file dulu!','Select a file first!','اختر ملفاً أولاً!'))));
                  return;
                }
                final judul = _judulCtrl.text.trim().isEmpty ? namaFile : _judulCtrl.text.trim();
                final b64 = base64Data; final t = tipe;
                Navigator.pop(ctx);
                // Simpan referensi messenger sebelum async gap
                final messenger = ScaffoldMessenger.of(context);
                final prov = Provider.of<AppProvider>(context, listen: false);
                messenger.showSnackBar(
                    SnackBar(content: Text(prov._t('Menyimpan media...','Saving media...','جاري الحفظ...')), duration: const Duration(seconds: 2)));
                Future.delayed(const Duration(milliseconds: 50), () async {
                  try {
                    await prov.addMediaBase64(b64, t, judul);
                    messenger.showSnackBar(
                        SnackBar(content: Text(prov._t('Media ditambahkan!','Media added!','تمت إضافة الوسائط!')),
                          backgroundColor: Colors.green));
                  } catch (e) {
                    messenger.showSnackBar(
                        SnackBar(content: Text('Error: $e'), backgroundColor: Colors.red));
                  }
                });
              },
              child: Text(p.lTambah),
            ),
          ],
        ),
      ),
    );
  }

  void _editDurasi(AppProvider p) {
    _durasiCtrl.text = p.durasiSlideDefault.toString();
    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: Text(p._t('Durasi Slide Foto','Photo Slide Duration','مدة شريحة الصورة')),
        content: TextField(
          controller: _durasiCtrl,
          keyboardType: TextInputType.number,
          decoration: InputDecoration(
            labelText: p._t('Detik per foto','Seconds per photo','ثانية لكل صورة'),
            suffixText: p._t('detik','sec','ث'),
          ),
        ),
        actions: [
          TextButton(
              onPressed: () => Navigator.pop(context),
              child: Text(p.lBatal)),
          ElevatedButton(
            onPressed: () {
              final detik = int.tryParse(_durasiCtrl.text) ?? 10;
              p.updateDurasiSlide(detik.clamp(3, 60));
              Navigator.pop(context);
              ScaffoldMessenger.of(context).showSnackBar(
                  SnackBar(content: Text(p._t('Durasi diubah: $detik detik','Duration changed: $detik sec','تم تغيير المدة: $detik ثانية'))));
            },
            child: Text(p.lSimpan),
          ),
        ],
      ),
    );
  }

  void _konfirmasiHapus(AppProvider p, int index) {
    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: Text(p._t('Hapus Media','Delete Media','حذف الوسائط')),
        content: Text(p._t('Yakin hapus media ini dari daftar slide?','Delete this media from slide list?','حذف هذه الوسائط من قائمة الشرائح؟')),
        actions: [
          TextButton(
              onPressed: () => Navigator.pop(context),
              child: Text(p.lBatal)),
          TextButton(
            onPressed: () { p.deleteMedia(index); Navigator.pop(context); },
            child: Text(p.lHapus, style: const TextStyle(color: Colors.red)),
          ),
        ],
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    final p = context.watch<AppProvider>();
    return Column(
      children: [
        Container(
          decoration: const BoxDecoration(gradient: kGradientPrimary),
          padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 10),
          child: Row(
            children: [
              IconButton(
                icon: const Icon(Icons.arrow_back, color: Colors.white),
                onPressed: () => widget.onBack?.call(),
              ),
              const Icon(Icons.photo_library, color: Colors.white, size: 18),
              const SizedBox(width: 8),
              Expanded(
                child: Text(p._t('GALERI SLIDE','SLIDE GALLERY','معرض الشرائح'),
                    style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold, fontSize: 14)),
              ),
              IconButton(
                icon: const Icon(Icons.timer, color: Colors.white, size: 20),
                tooltip: p._t('Atur Durasi Foto','Set Photo Duration','ضبط مدة الصورة'),
                onPressed: () => _editDurasi(p),
              ),
              IconButton(
                icon: const Icon(Icons.add_photo_alternate, color: Colors.white, size: 20),
                tooltip: p.lTambahMedia,
                onPressed: () => _showPickerDialog(p),
              ),
            ],
          ),
        ),
        Expanded(
          child: p.galeri.isEmpty
          ? Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  const Icon(Icons.photo_library_outlined, size: 80, color: Colors.grey),
                  const SizedBox(height: 16),
                  Text(p._t('Belum ada slide','No slides yet','لا شرائح بعد'),
                      style: const TextStyle(color: Colors.grey, fontSize: 16)),
                  const SizedBox(height: 8),
                  Text(p._t('Tekan + untuk menambah foto atau video','Tap + to add photo or video','اضغط + لإضافة صورة أو فيديو'),
                    style: const TextStyle(color: Colors.grey, fontSize: 13)),
                  const SizedBox(height: 24),
                  ElevatedButton.icon(
                    icon: const Icon(Icons.add),
                    label: Text(p.lTambahMedia),
                    style: ElevatedButton.styleFrom(
                        backgroundColor: const Color(0xFF631414),
                        foregroundColor: Colors.white),
                    onPressed: () => _showPickerDialog(p),
                  ),
                ],
              ),
            )
          : Column(
              children: [
                Container(
                  color: const Color(0xFF631414),
                  padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 8),
                  child: Row(children: [
                    const Icon(Icons.info_outline, color: Colors.white70, size: 14),
                    const SizedBox(width: 4),
                    Expanded(child: Text(
                      "${p.galeri.length} ${p._t('media','media','وسائط')}  •  ${p._t('Foto','Photo','صورة')}: ${p.durasiSlideDefault}${p._t('dtk','s','ث')}  •  ${p._t('Video','Video','فيديو')}: auto",
                      overflow: TextOverflow.ellipsis,
                      style: const TextStyle(color: Colors.white70, fontSize: 11),
                    )),
                  ]),
                ),
                Expanded(
                  child: ReorderableListView.builder(
                    itemCount: p.galeri.length,
                    onReorder: (oldIndex, newIndex) {
                      if (newIndex > oldIndex) newIndex--;
                      final item = p.galeri.removeAt(oldIndex);
                      p.galeri.insert(newIndex, item);
                      p.saveGaleri();
                    },
                    itemBuilder: (_, i) {
                      final item = p.galeri[i];
                      final isImage = item['type'] == 'image';
                      return ListTile(
                        key: ValueKey(i),
                        onTap: () => _viewMedia(item),
                        leading: Stack(
                          children: [
                            SizedBox(
                              width: 60, height: 60,
                              child: ClipRRect(
                                borderRadius: BorderRadius.circular(6),
                                child: isImage
                                    ? _GaleriImageWidget(path: item['path'])
                                    : Container(color: Colors.black87, child: const Icon(Icons.play_circle_filled, color: Colors.white, size: 32)),
                              ),
                            ),
                            if (!isImage)
                              const Positioned(
                                bottom: 2, right: 2,
                                child: Icon(Icons.play_arrow, color: Colors.white, size: 16),
                              ),
                          ],
                        ),
                        title: Text(
                          item['judul'] == '' ? p._t('(tanpa judul)','(no title)','(بدون عنوان)') : item['judul'],
                          style: const TextStyle(fontWeight: FontWeight.bold),
                        ),
                        subtitle: Text(
                          "${isImage ? p._t('Foto','Photo','صورة') : p._t('Video','Video','فيديو')}  •  "
                          "${(item['tgl'] ?? '').isNotEmpty ? item['tgl'] + '  •  ' : ''}"
                          "${isImage ? '${item['durasi']} ${p._t('detik','sec','ث')}' : p._t('auto selesai','auto finish','ينتهي تلقائياً')}",
                          style: const TextStyle(fontSize: 12),
                        ),
                        trailing: Row(mainAxisSize: MainAxisSize.min, children: [
                          IconButton(
                            icon: Icon(isImage ? Icons.fullscreen : Icons.play_circle_outline, color: kPrimary, size: 22),
                            tooltip: isImage ? 'Lihat Foto' : 'Putar Video',
                            onPressed: () => _viewMedia(item),
                          ),
                          IconButton(
                            icon: const Icon(Icons.delete_outline, color: Colors.grey, size: 20),
                            onPressed: () => _konfirmasiHapus(p, i),
                          ),
                        ]),
                      );
                    },
                  ),
                ),
              ],
            ),
        ),  // close Expanded
      ],
    );
  }
}

// ===== VIDEO PLAYER FULL SCREEN =====
class _VideoPlayerFullScreen extends StatefulWidget {
  final String path;
  const _VideoPlayerFullScreen({required this.path});
  @override
  State<_VideoPlayerFullScreen> createState() => _VideoPlayerFullScreenState();
}

class _VideoPlayerFullScreenState extends State<_VideoPlayerFullScreen> {
  VideoPlayerController? _ctrl;
  bool _ready = false;

  @override
  void initState() {
    super.initState();
    _init();
  }

  Future<void> _init() async {
    try {
      final ctrl = VideoPlayerController.file(File(widget.path));
      await ctrl.initialize();
      ctrl.setLooping(true);
      ctrl.play();
      if (mounted) setState(() { _ctrl = ctrl; _ready = true; });
    } catch (_) {}
  }

  @override
  void dispose() {
    _ctrl?.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (!_ready || _ctrl == null) {
      return const Center(child: CircularProgressIndicator(color: Colors.white));
    }
    return GestureDetector(
      onTap: () {
        if (_ctrl!.value.isPlaying) {
          _ctrl!.pause();
        } else {
          _ctrl!.play();
        }
        setState(() {});
      },
      child: Stack(alignment: Alignment.center, children: [
        AspectRatio(aspectRatio: _ctrl!.value.aspectRatio, child: VideoPlayer(_ctrl!)),
        if (!_ctrl!.value.isPlaying)
          const Icon(Icons.play_circle_outline, color: Colors.white70, size: 64),
      ]),
    );
  }
}

// ===== WRAPPER TAB KEUANGAN =====
class ArusKasTab extends StatefulWidget {
  final VoidCallback? onBack;
  const ArusKasTab({super.key, this.onBack});
  @override
  State<ArusKasTab> createState() => _ArusKasTabState();
}
class _ArusKasTabState extends State<ArusKasTab> {
  @override
  Widget build(BuildContext context) => _KeuanganWrapper(initialTab: 0, onBack: widget.onBack);
}

class DonaturTab extends StatefulWidget {
  final VoidCallback? onBack;
  const DonaturTab({super.key, this.onBack});
  @override
  State<DonaturTab> createState() => _DonaturTabState();
}

class _DonaturTabState extends State<DonaturTab> {
  final _namaCtrl = TextEditingController();
  final _jmlCtrl  = TextEditingController();
  final _ketCtrl  = TextEditingController();
  bool _anonim    = false;
  String _kategori = 'Sedekah';
  String _matauang = 'IDR';

  // Key kategori (disimpan tetap ID agar data konsisten)
  static const List<String> _kategoriList = [
    'Sedekah', 'Infak', 'Zakat', 'Zakat Fitrah', 'Wakaf',
    'Santunan Anak Yatim', 'Santunan Panti Asuhan',
    'Pembangunan Masjid', 'Operasional Masjid', 'Lainnya',
  ];

  // Terjemahan kategori sesuai bahasa
  static String _terjemahKategori(String k, String lang) {
    const en = {
      'Sedekah': 'Alms', 'Infak': 'Infaq', 'Zakat': 'Zakat',
      'Zakat Fitrah': 'Zakat Fitrah', 'Wakaf': 'Waqf',
      'Santunan Anak Yatim': 'Orphan Aid', 'Santunan Panti Asuhan': 'Orphanage Aid',
      'Pembangunan Masjid': 'Mosque Building', 'Operasional Masjid': 'Mosque Operations',
      'Lainnya': 'Others',
    };
    const ar = {
      'Sedekah': 'صدقة', 'Infak': 'إنفاق', 'Zakat': 'زكاة',
      'Zakat Fitrah': 'زكاة الفطر', 'Wakaf': 'وقف',
      'Santunan Anak Yatim': 'مساعدة الأيتام', 'Santunan Panti Asuhan': 'مساعدة دور الأيتام',
      'Pembangunan Masjid': 'بناء المسجد', 'Operasional Masjid': 'تشغيل المسجد',
      'Lainnya': 'أخرى',
    };
    if (lang == 'en') return en[k] ?? k;
    if (lang == 'ar') return ar[k] ?? k;
    return k;
  }

  static const List<Map<String, String>> _matauangList = [
    {'kode': 'IDR', 'simbol': 'Rp',  'nama': 'Rupiah'},
    {'kode': 'USD', 'simbol': '\$',   'nama': 'US Dollar'},
    {'kode': 'MYR', 'simbol': 'RM',  'nama': 'Ringgit'},
    {'kode': 'SGD', 'simbol': 'S\$', 'nama': 'Singapore Dollar'},
    {'kode': 'SAR', 'simbol': '﷼',   'nama': 'Riyal'},
    {'kode': 'AED', 'simbol': 'د.إ', 'nama': 'Dirham'},
    {'kode': 'EUR', 'simbol': '€',   'nama': 'Euro'},
    {'kode': 'GBP', 'simbol': '£',   'nama': 'Pound'},
  ];

  @override
  void dispose() {
    _namaCtrl.dispose(); _jmlCtrl.dispose(); _ketCtrl.dispose();
    super.dispose();
  }

  String get _tglHari => DateFormat('dd/MM/yyyy').format(DateTime.now());

  String _simbol(String kode) =>
    _matauangList.firstWhere((m) => m['kode'] == kode, orElse: () => {'simbol': kode})['simbol']!;

  void _cetakPdfDonatur(BuildContext context, AppProvider p) {
    if (p.donaturList.isEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(p._t('Belum ada data donatur untuk dicetak.','No donor data to print.','لا بيانات متبرعين للطباعة.'))));
      return;
    }
    showModalBottomSheet(
      context: context,
      shape: const RoundedRectangleBorder(
        borderRadius: BorderRadius.vertical(top: Radius.circular(16))),
      builder: (_) => SafeArea(child: Column(mainAxisSize: MainAxisSize.min, children: [
        const SizedBox(height: 8),
        Container(width: 40, height: 4,
          decoration: BoxDecoration(color: Colors.grey.shade300, borderRadius: BorderRadius.circular(2))),
        const SizedBox(height: 12),
        Consumer<AppProvider>(builder: (_, p, __) => Text(p._t('Laporan Donatur','Donor Report','تقرير المتبرعين'), style: const TextStyle(fontSize: 16, fontWeight: FontWeight.bold))),
        Text('${p.donaturList.length} ${p.bahasa == 'en' ? (p.donaturList.length == 1 ? 'donor' : 'donors') : p._t('donatur','donors','متبرع')} • Rp ${NumberFormat("#,##0","id").format(p.totalDonatur)}',
          style: const TextStyle(color: Colors.grey, fontSize: 13)),
        const Divider(),
        ListTile(
          leading: const CircleAvatar(backgroundColor: Color(0xFF631414),
            child: Icon(Icons.preview, color: Colors.white)),
          title: Consumer<AppProvider>(builder: (_, p, __) => Text(p._t('Pratinjau PDF','PDF Preview','معاينة PDF'))),
          subtitle: Consumer<AppProvider>(builder: (_, p2, __) => Text(p2._t('Lihat tampilan sebelum dicetak','Preview before printing','معاينة قبل الطباعة'))),
          onTap: () async {
            Navigator.pop(context);
            final doc = await _generatePdfDonatur(p);
            await Printing.layoutPdf(onLayout: (_) async => doc.save());
          },
        ),
        ListTile(
          leading: const CircleAvatar(backgroundColor: Colors.green,
            child: Icon(Icons.print, color: Colors.white)),
          title: Consumer<AppProvider>(builder: (_, p2, __) => Text(p2._t('Cetak ke Printer','Print','طباعة'))),
          subtitle: Text(p._t('WiFi / Bluetooth / USB','WiFi / Bluetooth / USB','واي فاي / بلوتوث / USB')),
          onTap: () async {
            Navigator.pop(context);
            final doc = await _generatePdfDonatur(p);
            if (!context.mounted) return;
            final printer = await Printing.pickPrinter(context: context);
            if (printer != null) {
              await Printing.directPrintPdf(printer: printer, onLayout: (_) async => doc.save());
            }
          },
        ),
        const SizedBox(height: 8),
      ])),
    );
  }

  Future<pw.Document> _generatePdfDonatur(AppProvider p) async {
    final doc = pw.Document();
    final fmt = NumberFormat('#,##0', 'id');
    String tglCetak;
    final pdfLocale2 = p.bahasa == 'en' ? 'en' : p.bahasa == 'ar' ? 'ar' : 'id';
    try { tglCetak = DateFormat('dd MMMM yyyy HH:mm', pdfLocale2).format(DateTime.now()); }
    catch (_) { tglCetak = DateFormat('dd/MM/yyyy HH:mm').format(DateTime.now()); }
    doc.addPage(pw.MultiPage(
      pageFormat: PdfPageFormat.a4,
      margin: const pw.EdgeInsets.all(32),
      header: (_) => pw.Column(crossAxisAlignment: pw.CrossAxisAlignment.start, children: [
        pw.Row(mainAxisAlignment: pw.MainAxisAlignment.spaceBetween, children: [
          pw.Column(crossAxisAlignment: pw.CrossAxisAlignment.start, children: [
            pw.Text(p.namaIbadah, style: pw.TextStyle(fontSize: 16, fontWeight: pw.FontWeight.bold)),
            pw.Text(p.alamat, style: const pw.TextStyle(fontSize: 10)),
          ]),
          pw.Column(crossAxisAlignment: pw.CrossAxisAlignment.end, children: [
            pw.Text((pdfLocale2 == 'en' ? 'DONOR REPORT' : pdfLocale2 == 'ar' ? 'تقرير المتبرعين' : 'LAPORAN DONATUR'), style: pw.TextStyle(fontSize: 11, fontWeight: pw.FontWeight.bold)),
            pw.Text((pdfLocale2 == 'en' ? 'Printed: ' : pdfLocale2 == 'ar' ? 'طُبع: ' : 'Dicetak: ') + tglCetak, style: const pw.TextStyle(fontSize: 9)),
          ]),
        ]),
        pw.Divider(thickness: 1.5),
        pw.SizedBox(height: 4),
      ]),
      footer: (ctx) => pw.Row(mainAxisAlignment: pw.MainAxisAlignment.spaceBetween, children: [
        pw.Text('NovaPro - Manajemen Masjid dan Mushollah', style: const pw.TextStyle(fontSize: 8)),
        pw.Text((pdfLocale2 == 'en' ? 'Page ' : pdfLocale2 == 'ar' ? 'صفحة ' : 'Halaman ') + ctx.pageNumber.toString() + ' / ' + ctx.pagesCount.toString(), style: const pw.TextStyle(fontSize: 8)),
      ]),
      build: (_) => [
        pw.Container(
          padding: const pw.EdgeInsets.all(10),
          decoration: const pw.BoxDecoration(color: PdfColors.grey100,
            borderRadius: pw.BorderRadius.all(pw.Radius.circular(6))),
          child: pw.Row(mainAxisAlignment: pw.MainAxisAlignment.spaceEvenly, children: [
            pw.Column(children: [
              pw.Text((pdfLocale2 == 'en' ? 'TOTAL DONORS' : pdfLocale2 == 'ar' ? 'إجمالي المتبرعين' : 'TOTAL DONATUR'), style: const pw.TextStyle(fontSize: 9)),
              pw.Text(p.donaturList.length.toString() + ' orang',
                style: pw.TextStyle(fontSize: 14, fontWeight: pw.FontWeight.bold, color: PdfColors.blue800)),
            ]),
            pw.Column(children: [
              pw.Text((pdfLocale2 == 'en' ? 'TOTAL DONATIONS' : pdfLocale2 == 'ar' ? 'إجمالي التبرعات' : 'TOTAL DONASI'), style: const pw.TextStyle(fontSize: 9)),
              pw.Text('Rp ' + fmt.format(p.totalDonatur),
                style: pw.TextStyle(fontSize: 14, fontWeight: pw.FontWeight.bold, color: PdfColors.green800)),
            ]),
          ]),
        ),
        pw.SizedBox(height: 16),
        pw.Text((pdfLocale2 == 'en' ? 'DONOR LIST' : pdfLocale2 == 'ar' ? 'قائمة المتبرعين' : 'DAFTAR DONATUR'), style: pw.TextStyle(fontSize: 11, fontWeight: pw.FontWeight.bold)),
        pw.SizedBox(height: 6),
        pw.Table(
          border: pw.TableBorder.all(color: PdfColors.grey400, width: 0.5),
          columnWidths: {
            0: const pw.FlexColumnWidth(0.5),
            1: const pw.FlexColumnWidth(2.5),
            2: const pw.FlexColumnWidth(2),
            3: const pw.FlexColumnWidth(2),
          },
          children: [
            pw.TableRow(
              decoration: const pw.BoxDecoration(color: PdfColors.grey200),
              children: ['No','Nama','Jumlah','Tanggal'].map((h) =>
                pw.Padding(padding: const pw.EdgeInsets.all(5),
                  child: pw.Text(h, style: pw.TextStyle(fontWeight: pw.FontWeight.bold, fontSize: 9)))).toList(),
            ),
            ...p.donaturList.asMap().entries.map((e) {
              final i = e.key; final d = e.value;
              final jumlah = (d['jumlah'] is num) ? (d['jumlah'] as num).toDouble() : double.tryParse(d['jumlah'].toString()) ?? 0;
              return pw.TableRow(
                decoration: pw.BoxDecoration(color: i % 2 == 0 ? PdfColors.white : PdfColors.grey50),
                children: [
                  pw.Padding(padding: const pw.EdgeInsets.all(5), child: pw.Text((i+1).toString(), style: const pw.TextStyle(fontSize: 9))),
                  pw.Padding(padding: const pw.EdgeInsets.all(5), child: pw.Text(d['nama']?.toString() ?? '-', style: const pw.TextStyle(fontSize: 9))),
                  pw.Padding(padding: const pw.EdgeInsets.all(5), child: pw.Text('Rp ' + fmt.format(jumlah), style: const pw.TextStyle(fontSize: 9))),
                  pw.Padding(padding: const pw.EdgeInsets.all(5), child: pw.Text(d['tanggal']?.toString() ?? '-', style: const pw.TextStyle(fontSize: 9))),
                ],
              );
            }),
            pw.TableRow(
              decoration: const pw.BoxDecoration(color: PdfColors.green50),
              children: [
                pw.Padding(padding: const pw.EdgeInsets.all(5), child: pw.Text('')),
                pw.Padding(padding: const pw.EdgeInsets.all(5), child: pw.Text('TOTAL', style: pw.TextStyle(fontWeight: pw.FontWeight.bold, fontSize: 9))),
                pw.Padding(padding: const pw.EdgeInsets.all(5), child: pw.Text('Rp ' + fmt.format(p.totalDonatur),
                  style: pw.TextStyle(fontWeight: pw.FontWeight.bold, fontSize: 9, color: PdfColors.green800))),
                pw.Padding(padding: const pw.EdgeInsets.all(5), child: pw.Text('')),
              ],
            ),
          ],
        ),
      ],
    ));
    return doc;
  }

  void _simpan(AppProvider p) {
    final nama  = _namaCtrl.text.trim();
    final rawJml = _jmlCtrl.text.trim().replaceAll('.', '').replaceAll(',', '');
    final jumlah = double.tryParse(rawJml) ?? 0;
    if (!_anonim && nama.isEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(
        content: Text(p._t('⚠️ Nama donatur tidak boleh kosong!','⚠️ Donor name cannot be empty!','⚠️ اسم المتبرع مطلوب!')),
        backgroundColor: Colors.orange));
      return;
    }
    if (jumlah <= 0) {
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(
        content: Text(p._t('⚠️ Jumlah donasi harus lebih dari 0!','⚠️ Donation amount must be > 0!','⚠️ مبلغ التبرع يجب أن يكون > 0!')),
        backgroundColor: Colors.orange));
      return;
    }
    p.addDonatur(nama, _anonim, _kategori, jumlah, _matauang, _ketCtrl.text.trim(), _tglHari);
    _namaCtrl.clear(); _jmlCtrl.clear(); _ketCtrl.clear();
    setState(() { _anonim = false; _kategori = 'Sedekah'; _matauang = 'IDR'; });
    ScaffoldMessenger.of(context).showSnackBar(SnackBar(
      content: Text(p._t('✅ Donasi ${_anonim ? 'Anonim' : nama} berhasil dicatat!','✅ Donation recorded!','✅ تم تسجيل التبرع!')),
      backgroundColor: Colors.green));
  }

  @override
  @override
  Widget build(BuildContext context) {
    final p   = context.watch<AppProvider>();
    final fmt = NumberFormat('#,##0', 'id');
    return LayoutBuilder(builder: (context, constraints) {
      return Column(children: [

        // ── Header ──
        Container(
          color: const Color(0xFF1a5276),
          padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
          child: Row(children: [
            IconButton(
              padding: EdgeInsets.zero,
              constraints: const BoxConstraints(minWidth: 36, minHeight: 36),
              icon: const Icon(Icons.arrow_back, color: Colors.white, size: 20),
              onPressed: () => widget.onBack?.call(),
            ),
            const Icon(Icons.volunteer_activism, color: Colors.white, size: 18),
            const SizedBox(width: 8),
            Expanded(child: Consumer<AppProvider>(builder: (_, p2, __) => Text(p2._t('DONATUR','DONORS','المتبرعون'), style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold, fontSize: 16)))),
            Text('${p.donaturList.length} ${p._t('donatur','donors','متبرع')}',
              style: const TextStyle(color: Colors.white70, fontSize: 12)),
            IconButton(
              icon: const Icon(Icons.picture_as_pdf, color: Colors.white, size: 20),
              tooltip: p._t('Cetak Laporan Donatur','Print Donor Report','طباعة تقرير المتبرعين'),
              onPressed: () => _cetakPdfDonatur(context, p),
            ),
          ]),
        ),

        // ── Form Input ──
        ConstrainedBox(
          constraints: BoxConstraints(
            maxHeight: constraints.maxHeight * 0.45,
          ),
          child: SingleChildScrollView(child:
          Material(
          color: Colors.white,
          child: Padding(
            padding: const EdgeInsets.fromLTRB(12, 4, 12, 4),
            child: Column(crossAxisAlignment: CrossAxisAlignment.start, mainAxisSize: MainAxisSize.min, children: [

              // Nama / Anonim
              Row(children: [
                Expanded(
                  child: TextField(
                    controller: _namaCtrl,
                    enabled: !_anonim,
                    decoration: InputDecoration(
                      labelText: _anonim ? p._t('Nama (Anonim)','Name (Anonymous)','الاسم (مجهول)') : p.lNamaDonatur,
                      prefixIcon: const Icon(Icons.person),
                      isDense: true,
                      filled: _anonim,
                      fillColor: Colors.grey.shade100,
                    ),
                  ),
                ),
                const SizedBox(width: 8),
                Column(mainAxisSize: MainAxisSize.min, children: [
                  Text(p.lAnonim, style: const TextStyle(fontSize: 11, color: Colors.grey)),
                  Switch(
                    value: _anonim,
                    onChanged: (v) => setState(() { _anonim = v; if (v) _namaCtrl.clear(); }),
                    activeColor: const Color(0xFF1a5276),
                  ),
                ]),
              ]),
              const SizedBox(height: 2),

              // Kategori
              DropdownButtonFormField<String>(
                value: _kategori,
                decoration: InputDecoration(
                  labelText: p._t('Kategori Donasi','Donation Category','فئة التبرع'),
                  prefixIcon: const Icon(Icons.category),
                  isDense: true,
                ),
                items: _kategoriList.map((k) =>
                  DropdownMenuItem(value: k, child: Text(_terjemahKategori(k, p.bahasa)))).toList(),
                onChanged: (v) => setState(() => _kategori = v!),
              ),
              const SizedBox(height: 2),

              // Mata uang + Jumlah
              Row(children: [
                Flexible(
                  flex: 2,
                  child: DropdownButtonFormField<String>(
                    value: _matauang,
                    decoration: InputDecoration(labelText: p._t('Mata Uang','Currency','العملة'), isDense: true),
                    items: _matauangList.map((m) => DropdownMenuItem(
                      value: m['kode'],
                      child: Text('${m['simbol']} ${m['kode']}', style: const TextStyle(fontSize: 12)),
                    )).toList(),
                    onChanged: (v) => setState(() => _matauang = v!),
                  ),
                ),
                const SizedBox(width: 8),
                Expanded(
                  flex: 3,
                  child: TextField(
                    controller: _jmlCtrl,
                    keyboardType: TextInputType.number,
                    decoration: InputDecoration(
                      labelText: p.lJumlah,
                      prefixText: '${_simbol(_matauang)} ',
                      isDense: true,
                    ),
                  ),
                ),
              ]),
              const SizedBox(height: 2),

              // Keterangan
              TextField(
                controller: _ketCtrl,
                decoration: InputDecoration(
                  labelText: p._t('Keterangan (opsional)','Note (optional)','ملاحظة (اختياري)'),
                  prefixIcon: const Icon(Icons.notes),
                  isDense: true,
                ),
              ),
              const SizedBox(height: 4),

              // Tombol simpan
              SizedBox(
                width: double.infinity,
                child: ElevatedButton.icon(
                  icon: const Icon(Icons.save),
                  label: Text(p._t('CATAT DONASI','RECORD DONATION','تسجيل التبرع')),
                  style: ElevatedButton.styleFrom(
                    backgroundColor: const Color(0xFF1a5276),
                    foregroundColor: Colors.white,
                  ),
                  onPressed: () => _simpan(p),
                ),
              ),
            ]),
          ),
        ))), // tutup Material + SingleChildScrollView + ConstrainedBox

        // ── Total donasi IDR ──
        if (p.donaturList.isNotEmpty)
          Container(
            color: const Color(0xFF1a5276).withAlpha(20),
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
            child: Row(children: [
              Flexible(child: Text('${p._t('Total Donasi (IDR)','Total Donation (IDR)','إجمالي التبرع (IDR)')}:',
                overflow: TextOverflow.ellipsis,
                style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 13))),
              const SizedBox(width: 8),
              Text('Rp ${fmt.format(p.totalDonatur)}',
                style: const TextStyle(color: Color(0xFF1a5276), fontWeight: FontWeight.bold, fontSize: 14)),
            ]),
          ),

        // ── Daftar donatur ──
        Expanded(
          child: p.donaturList.isEmpty
            ? Center(child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [
                const Icon(Icons.volunteer_activism, size: 64, color: Colors.grey),
                const SizedBox(height: 12),
                Text(p.lBelumAdaDonatur, style: const TextStyle(color: Colors.grey, fontSize: 16)),
                const SizedBox(height: 4),
                Text(p._t('Isi form di atas untuk mencatat donasi','Fill in the form above to record donation','املأ النموذج أعلاه لتسجيل التبرع'),
                  style: const TextStyle(color: Colors.grey, fontSize: 12)),
              ]))
            : ListView.builder(
                padding: const EdgeInsets.all(8),
                itemCount: p.donaturList.length,
                itemBuilder: (_, i) {
                  final d = p.donaturList[p.donaturList.length - 1 - i];
                  final idxAsli = p.donaturList.length - 1 - i;
                  final simbol = _simbol(d['matauang'] ?? 'IDR');
                  final jml    = (d['jumlah'] as double);
                  final jmlStr = d['matauang'] == 'IDR'
                      ? 'Rp ${fmt.format(jml)}'
                      : '$simbol ${jml.toStringAsFixed(2)}';
                  return Card(
                    margin: const EdgeInsets.symmetric(vertical: 4, horizontal: 4),
                    elevation: 1,
                    child: Padding(
                      padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 8),
                      child: Row(crossAxisAlignment: CrossAxisAlignment.center, children: [
                        CircleAvatar(
                          backgroundColor: const Color(0xFF1a5276),
                          child: Text(
                            (d['nama'] as String).isNotEmpty ? (d['nama'] as String)[0].toUpperCase() : '?',
                            style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold),
                          ),
                        ),
                        const SizedBox(width: 12),
                        Expanded(
                          child: Column(crossAxisAlignment: CrossAxisAlignment.start, mainAxisSize: MainAxisSize.min, children: [
                            Row(children: [
                              Flexible(child: Text(d['nama'] ?? '-',
                                style: const TextStyle(fontWeight: FontWeight.bold),
                                overflow: TextOverflow.ellipsis)),
                              if (d['anonim'] == true) ...[
                                const SizedBox(width: 6),
                                Container(
                                  padding: const EdgeInsets.symmetric(horizontal: 6, vertical: 2),
                                  decoration: BoxDecoration(
                                    color: Colors.grey.shade200,
                                    borderRadius: BorderRadius.circular(10),
                                  ),
                                  child: Text(p.lAnonim, style: const TextStyle(fontSize: 10, color: Colors.grey)),
                                ),
                              ],
                            ]),
                            Text(d['kategori'] ?? '', style: const TextStyle(color: Color(0xFF1a5276), fontSize: 12)),
                            if ((d['keterangan'] ?? '').isNotEmpty)
                              Text(d['keterangan'], style: const TextStyle(color: Colors.grey, fontSize: 11),
                                overflow: TextOverflow.ellipsis),
                            Text(p.formatTgl(d['tgl'] ?? ''), style: const TextStyle(color: Colors.grey, fontSize: 11)),
                          ]),
                        ),
                        const SizedBox(width: 8),
                        Column(mainAxisSize: MainAxisSize.min, crossAxisAlignment: CrossAxisAlignment.end, children: [
                          ConstrainedBox(
                            constraints: const BoxConstraints(maxWidth: 120),
                            child: Text(jmlStr,
                              style: const TextStyle(color: Color(0xFF1a5276), fontWeight: FontWeight.bold, fontSize: 13),
                              overflow: TextOverflow.ellipsis, maxLines: 1),
                          ),
                          GestureDetector(
                            onTap: () => _konfirmasiHapus(p, idxAsli),
                            child: const Padding(
                              padding: EdgeInsets.only(top: 4),
                              child: Icon(Icons.delete_outline, color: Colors.red, size: 18),
                            ),
                          ),
                        ]),
                      ]),
                    ),
                  );
                },
              ),
        ),

      ]); // tutup Column
    }); // tutup LayoutBuilder
  }

  void _konfirmasiHapus(AppProvider p, int index) {
    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: Text(p.lHapusDonatur),
        content: Text(p.lYakinHapusDonasi),
        actions: [
          TextButton(onPressed: () => Navigator.pop(context), child: Text(p.lBatal)),
          TextButton(
            onPressed: () { p.deleteDonatur(index); Navigator.pop(context); },
            child: Text(p.lHapus, style: const TextStyle(color: Colors.red)),
          ),
        ],
      ),
    );
  }
}

class InventarisTab extends StatefulWidget {
  final VoidCallback? onBack;
  const InventarisTab({super.key, this.onBack});
  @override
  State<InventarisTab> createState() => _InventarisTabState();
}
class _InventarisTabState extends State<InventarisTab> {
  @override
  Widget build(BuildContext context) => _KeuanganWrapper(initialTab: 2, onBack: widget.onBack);
}

class _KeuanganWrapper extends StatelessWidget {
  final int initialTab;
  final VoidCallback? onBack;
  const _KeuanganWrapper({required this.initialTab, this.onBack});
  @override
  Widget build(BuildContext context) => KeuanganPage(initialTab: initialTab, onBack: onBack);
}

class KeuanganPage extends StatefulWidget {
  final int initialTab;
  final VoidCallback? onBack;
  const KeuanganPage({super.key, this.initialTab = 0, this.onBack});
  @override
  State<KeuanganPage> createState() => _KeuanganPageState();
}

class _KeuanganPageState extends State<KeuanganPage> {
  final _tglCtrl      = TextEditingController();
  final _ketCtrl      = TextEditingController();
  final _jmlCtrl      = TextEditingController();
  final _donaturCtrl  = TextEditingController();
  // Controllers inventaris
  final _invNamaCtrl  = TextEditingController();
  final _invJmlCtrl   = TextEditingController();
  final _invSatCtrl   = TextEditingController();
  final _invKetCtrl   = TextEditingController();
  String _jenis       = 'masuk';
  bool _isDonatur     = false;
  bool _anonim        = false;
  int _tabIndex       = 0;

  final _fmt = NumberFormat('#,##0', 'id');

  @override
  void initState() {
    super.initState();
    _tabIndex = widget.initialTab;
    _tglCtrl.text = DateFormat('dd/MM/yyyy').format(DateTime.now());
  }

  @override
  void dispose() {
    _tglCtrl.dispose();
    _ketCtrl.dispose();
    _jmlCtrl.dispose();
    _donaturCtrl.dispose();
    _invNamaCtrl.dispose();
    _invJmlCtrl.dispose();
    _invSatCtrl.dispose();
    _invKetCtrl.dispose();
    super.dispose();
  }

  void _tambah(AppProvider p) {
    final rawText = _jmlCtrl.text.trim()
        .replaceAll('.', '')
        .replaceAll(',', '')
        .replaceAll(' ', '');
    final jumlah = double.tryParse(rawText) ?? 0;
    final ket = _ketCtrl.text.trim();
    if (ket.isEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(
          content: Text(p.lKetKosong),
          backgroundColor: Colors.orange,
        ),
      );
      return;
    }
    if (jumlah <= 0) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(
          content: Text(p.lJumlahNol),
          backgroundColor: Colors.orange,
        ),
      );
      return;
    }
    // Cek saldo jika pengeluaran
    if (_jenis == 'keluar' && jumlah > p.saldo) {
      final fmt = NumberFormat('#,##0', 'id');
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(
          content: Text(
            "${p.lSaldoTidakCukup} Saldo: Rp ${fmt.format(p.saldo)}, "
            "Butuh: Rp ${fmt.format(jumlah)}",
          ),
          backgroundColor: Colors.red,
          duration: const Duration(seconds: 3),
        ),
      );
      return;
    }
    final donatur = _isDonatur
        ? (_anonim ? 'Anonim' : _donaturCtrl.text.trim())
        : '';
    p.addTransaksi(_tglCtrl.text, ket, _jenis, jumlah, donatur: donatur);
    _ketCtrl.clear();
    _jmlCtrl.clear();
    _donaturCtrl.clear();
    setState(() { _isDonatur = false; _anonim = false; });
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        content: Text(p._t(
          '✅ ${_jenis == 'masuk' ? p.lPemasukan : p.lPengeluaran} berhasil dicatat!',
          '✅ ${_jenis == 'masuk' ? p.lPemasukan : p.lPengeluaran} recorded successfully!',
          '✅ تم تسجيل ${_jenis == 'masuk' ? p.lPemasukan : p.lPengeluaran} بنجاح!',
        )),
        backgroundColor: _jenis == 'masuk' ? Colors.green : Colors.red,
      ),
    );
  }

  void _editTransaksi(BuildContext context, AppProvider p, int index) {
    final t = p.transaksi[index];
    final tglCtrl     = TextEditingController(text: t['tgl']);
    final ketCtrl     = TextEditingController(text: t['keterangan']);
    final jmlCtrl     = TextEditingController(
        text: (t['jumlah'] as double).toInt().toString());
    final donaturCtrl = TextEditingController(text: t['donatur'] ?? '');
    String jenis = t['jenis'];

    showDialog(
      context: context,
      builder: (_) => StatefulBuilder(
        builder: (ctx, setLocal) => AlertDialog(
          title: Text(p.lEditTransaksi),
          content: SingleChildScrollView(
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                TextField(
                  controller: tglCtrl,
                  decoration: InputDecoration(
                      labelText: p.lTanggal, isDense: true),
                ),
                const SizedBox(height: 8),
                TextField(
                  controller: ketCtrl,
                  decoration: InputDecoration(
                      labelText: p.lKeterangan, isDense: true),
                ),
                const SizedBox(height: 8),
                TextField(
                  controller: jmlCtrl,
                  keyboardType: TextInputType.number,
                  decoration: InputDecoration(
                      labelText: '${p.lJumlah} (Rp)', isDense: true),
                ),
                const SizedBox(height: 8),
                TextField(
                  controller: donaturCtrl,
                  decoration: InputDecoration(
                      labelText: p.lNamaDonatur,
                      isDense: true),
                ),
                const SizedBox(height: 8),
                Wrap(
                  spacing: 8,
                  crossAxisAlignment: WrapCrossAlignment.center,
                  children: [
                    Text('${p.lTransaksi}: '),
                    ChoiceChip(
                      label: Text(p.lMasuk),
                      selected: jenis == 'masuk',
                      selectedColor: Colors.green.shade100,
                      onSelected: (_) => setLocal(() => jenis = 'masuk'),
                    ),
                    const SizedBox(width: 8),
                    ChoiceChip(
                      label: Text(p.lKeluar2),
                      selected: jenis == 'keluar',
                      selectedColor: Colors.red.shade100,
                      onSelected: (_) => setLocal(() => jenis = 'keluar'),
                    ),
                  ],
                ),
              ],
            ),
          ),
          actions: [
            TextButton(
              onPressed: () {
                Navigator.pop(context);
                tglCtrl.dispose(); ketCtrl.dispose();
                jmlCtrl.dispose(); donaturCtrl.dispose();
              },
              child: Text(p.lBatal),
            ),
            ElevatedButton(
              style: ElevatedButton.styleFrom(
                  backgroundColor: const Color(0xFF631414),
                  foregroundColor: Colors.white),
              onPressed: () {
                final jumlah = double.tryParse(
                    jmlCtrl.text.replaceAll('.','').replaceAll(',','')) ?? 0;
                p.editTransaksi(index, tglCtrl.text, ketCtrl.text, jenis, jumlah,
                    donatur: donaturCtrl.text.trim());
                Navigator.pop(context);
                tglCtrl.dispose(); ketCtrl.dispose();
                jmlCtrl.dispose(); donaturCtrl.dispose();
                ScaffoldMessenger.of(context).showSnackBar(
                    SnackBar(content: Text(p.lTransaksiDiperbarui)));
              },
              child: Text(p.lSimpan),
            ),
          ],
        ),
      ),
    );
  }

  void _konfirmasiHapusSemua(BuildContext context, AppProvider p) {
    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: Row(children: [
          const Icon(Icons.warning, color: Colors.red),
          const SizedBox(width: 8),
          Expanded(child: Text(
            p._t('Hapus Semua Riwayat','Delete All History','حذف كل السجل'),
            style: const TextStyle(color: Colors.red))),
        ]),
        content: Column(mainAxisSize: MainAxisSize.min, children: [
          Text(p._t(
            '⚠️ Semua riwayat transaksi akan dihapus. Total masuk, total keluar, dan saldo akhir akan TETAP TERSIMPAN.',
            '⚠️ All transaction history will be deleted. Total income, expense, and balance will REMAIN SAVED.',
            '⚠️ سيتم حذف جميع سجلات المعاملات. سيظل الرصيد النهائي محفوظاً.',
          ), style: const TextStyle(fontSize: 13)),
        ]),
        actions: [
          TextButton(onPressed: () => Navigator.pop(context), child: Text(p.lBatal)),
          ElevatedButton(
            style: ElevatedButton.styleFrom(backgroundColor: Colors.red, foregroundColor: Colors.white),
            onPressed: () {
              p.hapusSemualRiwayat();
              Navigator.pop(context);
              ScaffoldMessenger.of(context).showSnackBar(SnackBar(
                content: Text(p._t(
                  'Riwayat dihapus. Saldo tetap tersimpan.',
                  'History deleted. Balance remains saved.',
                  'تم حذف السجل. الرصيد محفوظ.',
                )),
                backgroundColor: Colors.orange,
              ));
            },
            child: Text(p._t('Hapus Semua','Delete All','حذف الكل')),
          ),
        ],
      ),
    );
  }

  void _konfirmasiHapus(BuildContext context, AppProvider p, int index) {
    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: Text(p.lHapusTransaksi),
        content: Text(p.lYa + "?"),
        actions: [
          TextButton(onPressed: () => Navigator.pop(context), child: Text(p.lBatal)),
          TextButton(
            onPressed: () {
              p.deleteTransaksi(index);
              Navigator.pop(context);
            },
            child: Text(p.lHapus, style: TextStyle(color: Colors.red)),
          ),
        ],
      ),
    );
  }

  Future<pw.Document> _buildPdf(AppProvider p) async {
    final doc = pw.Document();
    final fmt = NumberFormat('#,##0', 'id');
    final pdfLocale = p.bahasa == 'en' ? 'en' : p.bahasa == 'ar' ? 'ar' : 'id';
    String tglCetak;
    try {
      tglCetak = DateFormat('dd MMMM yyyy HH:mm', pdfLocale).format(DateTime.now());
    } catch (_) {
      tglCetak = DateFormat('dd/MM/yyyy HH:mm').format(DateTime.now());
    }

    doc.addPage(
      pw.MultiPage(
        pageFormat: PdfPageFormat.a4,
        margin: const pw.EdgeInsets.all(32),
        header: (_) => pw.Column(
          crossAxisAlignment: pw.CrossAxisAlignment.start,
          children: [
            pw.Row(
              mainAxisAlignment: pw.MainAxisAlignment.spaceBetween,
              children: [
                pw.Column(
                  crossAxisAlignment: pw.CrossAxisAlignment.start,
                  children: [
                    pw.Text(p.namaIbadah,
                        style: pw.TextStyle(
                            fontSize: 16, fontWeight: pw.FontWeight.bold)),
                    pw.Text(p.alamat,
                        style: const pw.TextStyle(fontSize: 10)),
                    pw.Text('Kode: ${p.houseUniqueCode}',
                        style: const pw.TextStyle(fontSize: 9)),
                  ],
                ),
                pw.Column(
                  crossAxisAlignment: pw.CrossAxisAlignment.end,
                  children: [
                    pw.Text((pdfLocale == 'en' ? 'FINANCIAL REPORT' : pdfLocale == 'ar' ? 'التقرير المالي' : 'LAPORAN KEUANGAN'),
                        style: pw.TextStyle(
                            fontSize: 11, fontWeight: pw.FontWeight.bold)),
                    pw.Text('Dicetak: $tglCetak',
                        style: const pw.TextStyle(fontSize: 9)),
                  ],
                ),
              ],
            ),
            pw.Divider(thickness: 1.5),
            pw.SizedBox(height: 4),
          ],
        ),
        footer: (ctx) => pw.Row(
          mainAxisAlignment: pw.MainAxisAlignment.spaceBetween,
          children: [
            pw.Text('NovaPro - Manajemen Masjid dan Mushollah',
                style: const pw.TextStyle(fontSize: 8)),
            pw.Text('Halaman ${ctx.pageNumber} / ${ctx.pagesCount}',
                style: const pw.TextStyle(fontSize: 8)),
          ],
        ),
        build: (_) => [

          pw.Container(
            padding: const pw.EdgeInsets.all(10),
            decoration: const pw.BoxDecoration(
              color: PdfColors.grey100,
              borderRadius: pw.BorderRadius.all(pw.Radius.circular(6)),
            ),
            child: pw.Row(
              mainAxisAlignment: pw.MainAxisAlignment.spaceEvenly,
              children: [
                _pdfBox('TOTAL MASUK',
                    'Rp ${fmt.format(p.totalMasuk)}', PdfColors.green800),
                _pdfBox('TOTAL KELUAR',
                    'Rp ${fmt.format(p.totalKeluar)}', PdfColors.red800),
                _pdfBox('SALDO AKHIR',
                    'Rp ${fmt.format(p.saldo)}',
                    p.saldo >= 0 ? PdfColors.blue800 : PdfColors.red900),
              ],
            ),
          ),
          pw.SizedBox(height: 14),

          pw.Text((pdfLocale == 'en' ? 'TRANSACTION DETAILS' : pdfLocale == 'ar' ? 'تفاصيل المعاملات' : 'RINCIAN TRANSAKSI'),
              style: pw.TextStyle(
                  fontSize: 11, fontWeight: pw.FontWeight.bold)),
          pw.SizedBox(height: 6),

          pw.Table(
            border: pw.TableBorder.all(
                color: PdfColors.grey400, width: 0.5),
            columnWidths: {
              0: const pw.FixedColumnWidth(60),
              1: const pw.FlexColumnWidth(3),
              2: const pw.FixedColumnWidth(50),
              3: const pw.FlexColumnWidth(2),
            },
            children: [

              pw.TableRow(
                decoration:
                    const pw.BoxDecoration(color: PdfColors.grey800),
                children: [
                  _pdfCell('TANGGAL',    bold: true, white: true),
                  _pdfCell('KETERANGAN', bold: true, white: true),
                  _pdfCell('JENIS',      bold: true, white: true),
                  _pdfCell('JUMLAH (Rp)',bold: true, white: true, right: true),
                ],
              ),

              ...p.transaksi.asMap().entries.map((e) {
                final i = e.key;
                final t = e.value;
                final masuk = t['jenis'] == 'masuk';
                return pw.TableRow(
                  decoration: pw.BoxDecoration(
                      color: i.isEven ? PdfColors.white : PdfColors.grey50),
                  children: [
                    _pdfCell(t['tgl']),
                    _pdfCell(t['keterangan']),
                    _pdfCell(
                      masuk ? 'MASUK' : 'KELUAR',
                      bold: true,
                      color: masuk ? PdfColors.green800 : PdfColors.red800,
                    ),
                    _pdfCell(
                      fmt.format(t['jumlah']),
                      right: true,
                      color: masuk ? PdfColors.green800 : PdfColors.red800,
                    ),
                  ],
                );
              }),

              pw.TableRow(
                decoration:
                    const pw.BoxDecoration(color: PdfColors.grey200),
                children: [
                  _pdfCell(''),
                  _pdfCell('SALDO AKHIR', bold: true),
                  _pdfCell(''),
                  _pdfCell(fmt.format(p.saldo),
                      bold: true,
                      right: true,
                      color: p.saldo >= 0
                          ? PdfColors.blue900
                          : PdfColors.red900),
                ],
              ),
            ],
          ),
          pw.SizedBox(height: 30),

          // Nomor seri + tempat tanggal
          pw.Row(
            mainAxisAlignment: pw.MainAxisAlignment.spaceBetween,
            children: [
              pw.Text(
                'No. Seri: NP-KAS-${DateTime.now().year}-${DateTime.now().millisecondsSinceEpoch.toString().substring(8)}',
                style: const pw.TextStyle(fontSize: 9, color: PdfColors.grey600)),
              pw.Column(crossAxisAlignment: pw.CrossAxisAlignment.end, children: [
                pw.Text(p.alamat, style: const pw.TextStyle(fontSize: 10)),
                pw.Text(tglCetak, style: const pw.TextStyle(fontSize: 10)),
              ]),
            ],
          ),
          pw.SizedBox(height: 24),

          // Tanda tangan 3 pengurus
          pw.Row(
            mainAxisAlignment: pw.MainAxisAlignment.spaceAround,
            children: [
              _pdfTtd(p._t('Ketua','Chairman','الرئيس'), p.ketua),
              _pdfTtd(p._t('Sekretaris','Secretary','السكرتير'), p.sekretaris),
              _pdfTtd(p._t('Bendahara','Treasurer','أمين الصندوق'), p.bendahara),
            ],
          ),
        ],
      ),
    );
    return doc;
  }

  // Helper tanda tangan PDF
  pw.Widget _pdfTtd(String jabatan, String nama) => pw.Column(
    crossAxisAlignment: pw.CrossAxisAlignment.center,
    children: [
      pw.Text(jabatan, style: pw.TextStyle(fontSize: 10, fontWeight: pw.FontWeight.bold)),
      pw.SizedBox(height: 40),
      pw.Container(width: 110, height: 0.5, color: PdfColors.black),
      pw.SizedBox(height: 4),
      pw.Text(
        nama.isEmpty || nama == '-' ? '(________________)' : nama,
        style: pw.TextStyle(fontSize: 10, fontWeight: pw.FontWeight.bold)),
    ],
  );

  pw.Widget _pdfCell(String text,
      {bool bold = false,
      bool white = false,
      bool right = false,
      PdfColor? color}) =>
      pw.Padding(
        padding: const pw.EdgeInsets.symmetric(horizontal: 6, vertical: 5),
        child: pw.Text(
          text,
          textAlign: right ? pw.TextAlign.right : pw.TextAlign.left,
          style: pw.TextStyle(
            fontSize: 9,
            fontWeight: bold ? pw.FontWeight.bold : pw.FontWeight.normal,
            color: white ? PdfColors.white : (color ?? PdfColors.black),
          ),
        ),
      );

  pw.Widget _pdfBox(String label, String value, PdfColor color) =>
      pw.Column(
        crossAxisAlignment: pw.CrossAxisAlignment.center,
        children: [
          pw.Text(label, style: const pw.TextStyle(fontSize: 9)),
          pw.SizedBox(height: 4),
          pw.Text(value,
              style: pw.TextStyle(
                  fontSize: 11,
                  fontWeight: pw.FontWeight.bold,
                  color: color)),
        ],
      );

  void _cetakLaporanDonatur(BuildContext context, AppProvider p) {
    if (p.donaturList.isEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(p._t('Belum ada data donatur untuk dicetak.','No donor data to print.','لا بيانات متبرعين للطباعة.'))));
      return;
    }
    showModalBottomSheet(
      context: context,
      shape: const RoundedRectangleBorder(
          borderRadius: BorderRadius.vertical(top: Radius.circular(16))),
      builder: (_) => SafeArea(
        child: Column(mainAxisSize: MainAxisSize.min, children: [
          const SizedBox(height: 8),
          Container(width: 40, height: 4,
            decoration: BoxDecoration(color: Colors.grey.shade300, borderRadius: BorderRadius.circular(2))),
          const SizedBox(height: 12),
          Text(p._t('Laporan Donatur','Donor Report','تقرير المتبرعين'), style: const TextStyle(fontSize: 16, fontWeight: FontWeight.bold)),
          const Divider(),
          ListTile(
            leading: const CircleAvatar(backgroundColor: Color(0xFF631414),
              child: Icon(Icons.preview, color: Colors.white)),
            title: Text(p.lPratinjauPdf),
            subtitle: Text(p._t('Lihat tampilan sebelum dicetak','Preview before printing','معاينة قبل الطباعة')),
            onTap: () async {
              Navigator.pop(context);
              final doc = await _buildPdfDonatur(p);
              await Printing.layoutPdf(onLayout: (_) async => doc.save());
            },
          ),
          ListTile(
            leading: const CircleAvatar(backgroundColor: Colors.blue,
              child: Icon(Icons.save_alt, color: Colors.white)),
            title: Text(p.lSimpanPdf),
            subtitle: Text(p._t('Tersimpan di folder penyimpanan','Saved in storage folder','محفوظ في مجلد التخزين')),
            onTap: () async {
              Navigator.pop(context);
              try {
                final doc   = await _buildPdfDonatur(p);
                final bytes = await doc.save();
                final dir   = await getExternalStorageDirectory();
                final tgl   = DateFormat('yyyyMMdd_HHmm').format(DateTime.now());
                final file  = File('${dir!.path}/Donatur_$tgl.pdf');
                await file.writeAsBytes(bytes);
                if (context.mounted) {
                  ScaffoldMessenger.of(context).showSnackBar(SnackBar(
                    content: Text('${p._t('PDF disimpan','PDF saved','تم حفظ PDF')}: ${file.path}'),
                    duration: const Duration(seconds: 4)));
                }
              } catch (e) {
                if (context.mounted) {
                  ScaffoldMessenger.of(context).showSnackBar(
                    SnackBar(content: Text('${p._t('Gagal simpan','Save failed','فشل الحفظ')}: $e')));
                }
              }
            },
          ),
          ListTile(
            leading: const CircleAvatar(backgroundColor: Colors.green,
              child: Icon(Icons.print, color: Colors.white)),
            title: Text(p.lCetak),
            subtitle: Text(p._t('WiFi / Bluetooth / USB printer','WiFi / Bluetooth / USB printer','طابعة واي فاي / بلوتوث / USB')),
            onTap: () async {
              Navigator.pop(context);
              final doc = await _buildPdfDonatur(p);
              if (!context.mounted) return;
              final printer = await Printing.pickPrinter(context: context);
              if (printer != null) {
                await Printing.directPrintPdf(
                  printer: printer, onLayout: (_) async => doc.save());
              }
            },
          ),
          const SizedBox(height: 8),
        ]),
      ),
    );
  }

  Future<pw.Document> _buildPdfDonatur(AppProvider p) async {
    final doc = pw.Document();
    final fmt = NumberFormat('#,##0', 'id');
    final pdfLocale = p.bahasa == 'en' ? 'en' : p.bahasa == 'ar' ? 'ar' : 'id';
    String tglCetak;
    try {
      tglCetak = DateFormat('dd MMMM yyyy HH:mm', pdfLocale).format(DateTime.now());
    } catch (_) {
      tglCetak = DateFormat('dd/MM/yyyy HH:mm').format(DateTime.now());
    }

    doc.addPage(
      pw.MultiPage(
        pageFormat: PdfPageFormat.a4,
        margin: const pw.EdgeInsets.all(32),
        header: (_) => pw.Column(crossAxisAlignment: pw.CrossAxisAlignment.start, children: [
          pw.Row(mainAxisAlignment: pw.MainAxisAlignment.spaceBetween, children: [
            pw.Column(crossAxisAlignment: pw.CrossAxisAlignment.start, children: [
              pw.Text(p.namaIbadah, style: pw.TextStyle(fontSize: 16, fontWeight: pw.FontWeight.bold)),
              pw.Text(p.alamat, style: const pw.TextStyle(fontSize: 10)),
            ]),
            pw.Column(crossAxisAlignment: pw.CrossAxisAlignment.end, children: [
              pw.Text((pdfLocale == 'en' ? 'DONOR REPORT' : pdfLocale == 'ar' ? 'تقرير المتبرعين' : 'LAPORAN DONATUR'), style: pw.TextStyle(fontSize: 11, fontWeight: pw.FontWeight.bold)),
              pw.Text((pdfLocale == 'en' ? 'Printed: ' : pdfLocale == 'ar' ? 'طُبع: ' : 'Dicetak: ') + tglCetak, style: const pw.TextStyle(fontSize: 9)),
            ]),
          ]),
          pw.Divider(thickness: 1.5),
          pw.SizedBox(height: 4),
        ]),
        footer: (ctx) => pw.Row(mainAxisAlignment: pw.MainAxisAlignment.spaceBetween, children: [
          pw.Text('NovaPro - Manajemen Masjid dan Mushollah', style: const pw.TextStyle(fontSize: 8)),
          pw.Text((pdfLocale == 'en' ? 'Page ' : pdfLocale == 'ar' ? 'صفحة ' : 'Halaman ') + ctx.pageNumber.toString() + ' / ' + ctx.pagesCount.toString(), style: const pw.TextStyle(fontSize: 8)),
        ]),
        build: (_) => [
          // Ringkasan
          pw.Container(
            padding: const pw.EdgeInsets.all(10),
            decoration: const pw.BoxDecoration(
              color: PdfColors.grey100,
              borderRadius: pw.BorderRadius.all(pw.Radius.circular(6))),
            child: pw.Row(mainAxisAlignment: pw.MainAxisAlignment.spaceEvenly, children: [
              pw.Column(children: [
                pw.Text((pdfLocale == 'en' ? 'TOTAL DONORS' : pdfLocale == 'ar' ? 'إجمالي المتبرعين' : 'TOTAL DONATUR'), style: const pw.TextStyle(fontSize: 9)),
                pw.Text(p.donaturList.length.toString() + ' orang',
                  style: pw.TextStyle(fontSize: 14, fontWeight: pw.FontWeight.bold, color: PdfColors.blue800)),
              ]),
              pw.Column(children: [
                pw.Text((pdfLocale == 'en' ? 'TOTAL DONATIONS' : pdfLocale == 'ar' ? 'إجمالي التبرعات' : 'TOTAL DONASI'), style: const pw.TextStyle(fontSize: 9)),
                pw.Text('Rp ' + fmt.format(p.totalDonatur),
                  style: pw.TextStyle(fontSize: 14, fontWeight: pw.FontWeight.bold, color: PdfColors.green800)),
              ]),
            ]),
          ),
          pw.SizedBox(height: 16),
          // Tabel donatur
          pw.Text((pdfLocale == 'en' ? 'DONOR LIST' : pdfLocale == 'ar' ? 'قائمة المتبرعين' : 'DAFTAR DONATUR'), style: pw.TextStyle(fontSize: 11, fontWeight: pw.FontWeight.bold)),
          pw.SizedBox(height: 6),
          pw.Table(
            border: pw.TableBorder.all(color: PdfColors.grey400, width: 0.5),
            columnWidths: {
              0: const pw.FlexColumnWidth(0.5),
              1: const pw.FlexColumnWidth(2.5),
              2: const pw.FlexColumnWidth(2),
              3: const pw.FlexColumnWidth(2),
            },
            children: [
              // Header tabel
              pw.TableRow(
                decoration: const pw.BoxDecoration(color: PdfColors.grey200),
                children: [
                  pw.Padding(padding: const pw.EdgeInsets.all(5),
                    child: pw.Text('No', style: pw.TextStyle(fontWeight: pw.FontWeight.bold, fontSize: 9))),
                  pw.Padding(padding: const pw.EdgeInsets.all(5),
                    child: pw.Text((pdfLocale == 'en' ? 'Name' : pdfLocale == 'ar' ? 'الاسم' : 'Nama'), style: pw.TextStyle(fontWeight: pw.FontWeight.bold, fontSize: 9))),
                  pw.Padding(padding: const pw.EdgeInsets.all(5),
                    child: pw.Text((pdfLocale == 'en' ? 'Amount' : pdfLocale == 'ar' ? 'المبلغ' : 'Jumlah'), style: pw.TextStyle(fontWeight: pw.FontWeight.bold, fontSize: 9))),
                  pw.Padding(padding: const pw.EdgeInsets.all(5),
                    child: pw.Text((pdfLocale == 'en' ? 'Date' : pdfLocale == 'ar' ? 'التاريخ' : 'Tanggal'), style: pw.TextStyle(fontWeight: pw.FontWeight.bold, fontSize: 9))),
                ],
              ),
              // Baris data
              ...p.donaturList.asMap().entries.map((e) {
                final i = e.key;
                final d = e.value;
                final jumlah = (d['jumlah'] is num) ? (d['jumlah'] as num).toDouble() : double.tryParse(d['jumlah'].toString()) ?? 0;
                return pw.TableRow(
                  decoration: pw.BoxDecoration(color: i % 2 == 0 ? PdfColors.white : PdfColors.grey50),
                  children: [
                    pw.Padding(padding: const pw.EdgeInsets.all(5),
                      child: pw.Text((i + 1).toString(), style: const pw.TextStyle(fontSize: 9))),
                    pw.Padding(padding: const pw.EdgeInsets.all(5),
                      child: pw.Text(d['nama']?.toString() ?? '-', style: const pw.TextStyle(fontSize: 9))),
                    pw.Padding(padding: const pw.EdgeInsets.all(5),
                      child: pw.Text('Rp ' + fmt.format(jumlah), style: const pw.TextStyle(fontSize: 9))),
                    pw.Padding(padding: const pw.EdgeInsets.all(5),
                      child: pw.Text(d['tanggal']?.toString() ?? '-', style: const pw.TextStyle(fontSize: 9))),
                  ],
                );
              }),
              // Total
              pw.TableRow(
                decoration: const pw.BoxDecoration(color: PdfColors.green50),
                children: [
                  pw.Padding(padding: const pw.EdgeInsets.all(5), child: pw.Text('')),
                  pw.Padding(padding: const pw.EdgeInsets.all(5),
                    child: pw.Text('TOTAL', style: pw.TextStyle(fontWeight: pw.FontWeight.bold, fontSize: 9))),
                  pw.Padding(padding: const pw.EdgeInsets.all(5),
                    child: pw.Text('Rp ' + fmt.format(p.totalDonatur),
                      style: pw.TextStyle(fontWeight: pw.FontWeight.bold, fontSize: 9, color: PdfColors.green800))),
                  pw.Padding(padding: const pw.EdgeInsets.all(5), child: pw.Text('')),
                ],
              ),
            ],
          ),
        ],
      ),
    );

    // Tambah halaman tanda tangan di akhir
    doc.addPage(
      pw.Page(
        pageFormat: PdfPageFormat.a4,
        margin: const pw.EdgeInsets.all(32),
        build: (_) => pw.Column(
          crossAxisAlignment: pw.CrossAxisAlignment.start,
          children: [
            pw.Text(p.namaIbadah, style: pw.TextStyle(fontSize: 14, fontWeight: pw.FontWeight.bold)),
            pw.Text(p.alamat, style: const pw.TextStyle(fontSize: 10)),
            pw.Divider(thickness: 1),
            pw.SizedBox(height: 16),
            pw.Text((pdfLocale == 'en' ? 'DONOR REPORT' : pdfLocale == 'ar' ? 'تقرير المتبرعين' : 'LAPORAN DONATUR'), style: pw.TextStyle(fontSize: 12, fontWeight: pw.FontWeight.bold)),
            pw.SizedBox(height: 8),
            pw.Text('Total Donatur: ${p.donaturList.length} orang', style: const pw.TextStyle(fontSize: 10)),
            pw.Text('Total Donasi: Rp ${NumberFormat('#,##0', 'id').format(p.totalDonatur)}', style: const pw.TextStyle(fontSize: 10)),
            pw.SizedBox(height: 40),

            // Nomor seri
            pw.Text(
              'No. Seri: NP-DON-${DateTime.now().year}-${DateTime.now().millisecondsSinceEpoch.toString().substring(8)}',
              style: const pw.TextStyle(fontSize: 9, color: PdfColors.grey600)),
            pw.SizedBox(height: 8),

            // Tempat tanggal
            pw.Align(
              alignment: pw.Alignment.centerRight,
              child: pw.Column(crossAxisAlignment: pw.CrossAxisAlignment.end, children: [
                pw.Text(p.alamat, style: const pw.TextStyle(fontSize: 10)),
                pw.Text(DateFormat('dd MMMM yyyy').format(DateTime.now()), style: const pw.TextStyle(fontSize: 10)),
              ]),
            ),
            pw.SizedBox(height: 24),

            // Tanda tangan 3 pengurus
            pw.Row(
              mainAxisAlignment: pw.MainAxisAlignment.spaceAround,
              children: [
                _pdfTtd(p._t('Ketua','Chairman','الرئيس'), p.ketua),
                _pdfTtd(p._t('Sekretaris','Secretary','السكرتير'), p.sekretaris),
                _pdfTtd(p._t('Bendahara','Treasurer','أمين الصندوق'), p.bendahara),
              ],
            ),
          ],
        ),
      ),
    );

    return doc;
  }

  void _cetakLaporan(BuildContext context, AppProvider p) {
    if (p.transaksi.isEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(
          content: Text(p._t('Belum ada transaksi untuk dicetak.','No transactions to print.','لا معاملات للطباعة.'))));
      return;
    }

    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      shape: const RoundedRectangleBorder(
          borderRadius: BorderRadius.vertical(top: Radius.circular(16))),
      builder: (_) => SingleChildScrollView(
        child: SafeArea(
          child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            const SizedBox(height: 8),
            Container(
              width: 40, height: 4,
              decoration: BoxDecoration(
                  color: Colors.grey.shade300,
                  borderRadius: BorderRadius.circular(2)),
            ),
            const SizedBox(height: 12),
            Text(p.lLaporan,
                style: const TextStyle(fontSize: 16, fontWeight: FontWeight.bold)),
            Text('${p.transaksi.length} ${p._t('transaksi','transactions','معاملات')}',
                style: const TextStyle(color: Colors.grey, fontSize: 13)),
            const Divider(),

            ListTile(
              leading: const CircleAvatar(
                  backgroundColor: Color(0xFF631414),
                  child: Icon(Icons.preview, color: Colors.white)),
              title: Text(p.lPratinjauPdf),
              subtitle: Text(p._t('Lihat tampilan sebelum dicetak','Preview before printing','معاينة قبل الطباعة')),
              onTap: () async {
                Navigator.pop(context);
                final doc = await _buildPdf(p);
                await Printing.layoutPdf(onLayout: (_) async => doc.save());
              },
            ),

            ListTile(
              leading: const CircleAvatar(
                  backgroundColor: Colors.blue,
                  child: Icon(Icons.save_alt, color: Colors.white)),
              title: Text(p.lSimpanPdf),
              subtitle: Text(p._t('Tersimpan di folder penyimpanan','Saved in storage folder','محفوظ في مجلد التخزين')),
              onTap: () async {
                Navigator.pop(context);
                try {
                  final doc   = await _buildPdf(p);
                  final bytes = await doc.save();
                  final dir   = await getExternalStorageDirectory();
                  final tgl   = DateFormat('yyyyMMdd_HHmm').format(DateTime.now());
                  final file  = File('${dir!.path}/Laporan_$tgl.pdf');
                  await file.writeAsBytes(bytes);
                  if (context.mounted) {
                    ScaffoldMessenger.of(context).showSnackBar(SnackBar(
                      content: Text('${p._t('PDF disimpan','PDF saved','تم حفظ PDF')}: ${file.path}'),
                      duration: const Duration(seconds: 4),
                    ));
                  }
                } catch (e) {
                  if (context.mounted) {
                    ScaffoldMessenger.of(context).showSnackBar(
                        SnackBar(content: Text('${p._t('Gagal simpan','Save failed','فشل الحفظ')}: $e')));
                  }
                }
              },
            ),

            ListTile(
              leading: const CircleAvatar(
                  backgroundColor: Colors.green,
                  child: Icon(Icons.print, color: Colors.white)),
              title: Text(p.lCetak),
              subtitle: Text(p._t('WiFi / Bluetooth / USB printer','WiFi / Bluetooth / USB printer','طابعة واي فاي / بلوتوث / USB')),
              onTap: () async {
                Navigator.pop(context);
                final doc = await _buildPdf(p);
                if (!context.mounted) return;
                final printer = await Printing.pickPrinter(context: context);
                if (printer != null) {
                  await Printing.directPrintPdf(
                    printer: printer,
                    onLayout: (_) async => doc.save(),
                  );
                }
              },
            ),

            const SizedBox(height: 8),
          ],
        ),
        ), // SafeArea
      ), // SingleChildScrollView
    );
  }

  @override
  Widget build(BuildContext context) {
    final p = context.watch<AppProvider>();
    final titles = [
      p._t('ARUS KAS','CASH FLOW','التدفق النقدي'),
      p._t('DONATUR','DONORS','المتبرعون'),
      p._t('INVENTARIS','INVENTORY','المخزون'),
    ];
    final icons  = [Icons.swap_vert, Icons.volunteer_activism, Icons.inventory_2];
    // Clamp textScaler agar font tidak ikut setting aksesibilitas sistem
    return MediaQuery(
      data: MediaQuery.of(context).copyWith(textScaler: TextScaler.noScaling),
      child: Scaffold(
        resizeToAvoidBottomInset: true,
        backgroundColor: kBg,
        body: SafeArea(child: Column(
        children: [
          Container(
            color: kPrimary,
            padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 8),
            child: Row(
              children: [
                IconButton(
                  padding: EdgeInsets.zero,
                  constraints: const BoxConstraints(minWidth: 36, minHeight: 36),
                  icon: const Icon(Icons.arrow_back, color: Colors.white, size: 20),
                  onPressed: () => widget.onBack?.call(),
                ),
                Icon(icons[_tabIndex], color: Colors.white, size: 16),
                const SizedBox(width: 6),
                Expanded(
                  child: Text(titles[_tabIndex],
                      overflow: TextOverflow.ellipsis,
                      style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold, fontSize: 13)),
                ),
                if (_tabIndex == 0)
                IconButton(
                  padding: EdgeInsets.zero,
                  constraints: const BoxConstraints(minWidth: 36, minHeight: 36),
                  icon: const Icon(Icons.print, color: Colors.white, size: 18),
                  tooltip: p._t('Cetak Laporan Kas','Print Cash Report','طباعة تقرير الصندوق'),
                  onPressed: () => _cetakLaporan(context, p),
                ),
                if (_tabIndex == 1)
                IconButton(
                  padding: EdgeInsets.zero,
                  constraints: const BoxConstraints(minWidth: 36, minHeight: 36),
                  icon: const Icon(Icons.print, color: Colors.white, size: 18),
                  tooltip: p._t('Cetak Laporan Donatur','Print Donor Report','طباعة تقرير المتبرعين'),
                  onPressed: () => _cetakLaporanDonatur(context, p),
                ),
              ],
            ),
          ),
          Expanded(
            child: _tabIndex == 2
          ? _buildInventaris(p)
          : Column(
        children: [

          Container(
            color: const Color(0xFF631414),
            padding: const EdgeInsets.all(12),
            child: Row(
              children: [
                Expanded(child: _summaryBox(p.lMasuk.toUpperCase(),  p.totalMasukAll,  Colors.greenAccent)),
                const SizedBox(width: 4),
                Expanded(child: _summaryBox(p.lKeluar2.toUpperCase(), p.totalKeluarAll, Colors.redAccent)),
                const SizedBox(width: 4),
                Expanded(child: _summaryBox(p.lSaldo.toUpperCase(),  p.saldo,
                    p.saldo >= 0 ? Colors.yellowAccent : Colors.red)),
              ],
            ),
          ),

          Flexible(
            fit: FlexFit.loose,
            child: SingleChildScrollView(
            child: Container(
            padding: const EdgeInsets.symmetric(horizontal: 10, vertical: 6),
            color: Colors.grey.shade100,
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                Row(
                  children: [
                    Expanded(
                      child: GestureDetector(
                        onTap: () async {
                          final parts = _tglCtrl.text.split('/');
                          final init = parts.length == 3
                            ? DateTime.tryParse('${parts[2]}-${parts[1]}-${parts[0]}') ?? DateTime.now()
                            : DateTime.now();
                          final picked = await showDatePicker(
                            context: context,
                            initialDate: init,
                            firstDate: DateTime(2000),
                            lastDate: DateTime(2100),
                          );
                          if (picked != null) setState(() => _tglCtrl.text = DateFormat('dd/MM/yyyy').format(picked));
                        },
                        child: AbsorbPointer(
                          child: TextField(
                            controller: _tglCtrl,
                            decoration: InputDecoration(
                              labelText: p.lTanggal, isDense: true,
                              hintText: 'dd/MM/yyyy'),
                          ),
                        ),
                      ),
                    ),
                    const SizedBox(width: 8),
                    DropdownButton<String>(
                      value: _jenis,
                      items: [
                        DropdownMenuItem(value: 'masuk',  child: Text(p.lMasuk)),
                        DropdownMenuItem(value: 'keluar', child: Text(p.lKeluar2)),
                      ],
                      onChanged: (v) => setState(() => _jenis = v!),
                    ),
                  ],
                ),
                const SizedBox(height: 4),
                TextField(
                  controller: _ketCtrl,
                  decoration: InputDecoration(
                      labelText: p.lKeterangan, isDense: true),
                ),
                const SizedBox(height: 4),
                Row(
                  children: [
                    Expanded(
                      child: TextField(
                        controller: _jmlCtrl,
                        keyboardType: const TextInputType.numberWithOptions(decimal: false, signed: false),
                        inputFormatters: [FilteringTextInputFormatter.digitsOnly],
                        decoration: InputDecoration(
                            labelText: "${p.lJumlah} (Rp)", isDense: true,
                            hintText: '0',
                            prefixText: 'Rp '),
                      ),
                    ),
                    const SizedBox(width: 8),
                    ElevatedButton(
                      style: ElevatedButton.styleFrom(
                          backgroundColor: const Color(0xFF631414),
                          foregroundColor: Colors.white),
                      onPressed: () => _tambah(p),
                      child: Text(p.lTambah),
                    ),
                  ],
                ),

                if (_jenis == 'masuk') ...[
                  const SizedBox(height: 6),
                  Wrap(
                    crossAxisAlignment: WrapCrossAlignment.center,
                    children: [
                      Row(mainAxisSize: MainAxisSize.min, children: [
                        Checkbox(
                          value: _isDonatur,
                          onChanged: (v) => setState(() => _isDonatur = v!),
                        ),
                        Text(p.lAdaDonatur),
                      ]),
                      if (_isDonatur) ...[
                        Row(mainAxisSize: MainAxisSize.min, children: [
                          Checkbox(
                            value: _anonim,
                            onChanged: (v) => setState(() => _anonim = v!),
                          ),
                          Text(p.lAnonim),
                        ]),
                        if (!_anonim)
                          SizedBox(
                            width: 150,
                            child: TextField(
                              controller: _donaturCtrl,
                              decoration: InputDecoration(
                                  labelText: p.lNamaDonatur, isDense: true),
                            ),
                          ),
                      ],
                    ],
                  ),
                ],
              ],
            ),
          ))), // tutup Container form + SingleChildScrollView + Flexible

          // Tombol hapus semua riwayat
          if (p.transaksi.isNotEmpty)
            Padding(
              padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 4),
              child: Row(children: [
                Expanded(child: Text(
                  p._t('Riwayat','History','السجل') + ' (${p.transaksi.length})',
                  style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 12))),
                TextButton.icon(
                  icon: const Icon(Icons.delete_sweep, color: Colors.red, size: 16),
                  label: Text(p._t('Hapus Semua','Delete All','حذف الكل'),
                    style: const TextStyle(color: Colors.red, fontSize: 12)),
                  onPressed: () => _konfirmasiHapusSemua(context, p),
                ),
              ]),
            ),

          Expanded(
            child: p.transaksi.isEmpty
                ? Center(
                    child: Text(p.lBelumAdaTransaksi,
                        style: const TextStyle(color: Colors.grey)))
                : Builder(builder: (bCtx) {
                    final pp = bCtx.watch<AppProvider>();
                    final list = _tabIndex == 1
                        ? pp.transaksi.where((t) =>
                            (t['donatur'] ?? '').isNotEmpty).toList()
                        : pp.transaksi;
                    if (list.isEmpty) {
                      return Center(
                        child: Text(
                          _tabIndex == 1 ? pp.lBelumAdaDonatur : pp.lBelumAdaTransaksi,
                          style: const TextStyle(color: Colors.grey)));
                    }
                    return ListView.builder(
                      itemCount: list.length,
                      itemBuilder: (_, i) {
                      final t = list[i];
                      final realIdx = pp.transaksi.indexOf(t);
                      final isMasuk = t['jenis'] == 'masuk';
                      final trxKey = '${t['tgl']}_${t['keterangan']}_${t['jumlah']}_$i';
                      return Dismissible(
                        key: ValueKey(trxKey),
                        direction: DismissDirection.endToStart,
                        background: Container(
                          alignment: Alignment.centerRight,
                          padding: const EdgeInsets.only(right: 20),
                          color: Colors.red,
                          child: const Icon(Icons.delete, color: Colors.white),
                        ),
                        confirmDismiss: (_) async {
                          if (!context.mounted) return false;
                          final ok = await showDialog<bool>(
                            context: context,
                            builder: (dialogCtx) => AlertDialog(
                              title: Text(pp.lHapusTransaksi),
                              content: Text(t['keterangan']),
                              actions: [
                                TextButton(
                                    onPressed: () => Navigator.pop(dialogCtx, false),
                                    child: Text(pp.lBatal)),
                                TextButton(
                                  onPressed: () => Navigator.pop(dialogCtx, true),
                                  child: Text(pp.lHapus,
                                      style: const TextStyle(color: Colors.red)),
                                ),
                              ],
                            ),
                          );
                          if (ok == true) {
                            pp.deleteTransaksi(_tabIndex == 1 ? realIdx : i);
                          }
                          return false; // Jangan dismiss otomatis - biarkan Provider rebuild
                        },
                        child: Material(
                          color: Colors.white,
                          child: ListTile(
                          dense: true,
                          leading: CircleAvatar(
                            radius: 16,
                            backgroundColor:
                                isMasuk ? Colors.green.shade100 : Colors.red.shade100,
                            child: Icon(
                              isMasuk ? Icons.arrow_downward : Icons.arrow_upward,
                              size: 16,
                              color: isMasuk ? Colors.green : Colors.red,
                            ),
                          ),
                          title: Text(t['keterangan'],
                              style: const TextStyle(fontSize: 14)),
                            subtitle: Column(
                              crossAxisAlignment: CrossAxisAlignment.start,
                              children: [
                              Text(p.formatTgl(t['tgl'] ?? ''),
                                  style: const TextStyle(fontSize: 12)),
                              if ((t['donatur'] ?? '').isNotEmpty)
                                Row(children: [
                                  const Icon(Icons.volunteer_activism,
                                      size: 11, color: Colors.amber),
                                  const SizedBox(width: 2),
                                  Flexible(child: Text(t['donatur'],
                                      overflow: TextOverflow.ellipsis,
                                      style: const TextStyle(
                                          fontSize: 11, color: Colors.amber))),
                                ]),
                            ],
                          ),
                          trailing: SizedBox(
                            width: 110,
                            child: Row(
                              mainAxisSize: MainAxisSize.min,
                              children: [
                                Flexible(child: Text(
                                  "${isMasuk ? '+' : '-'}Rp${_fmt.format(t['jumlah'])}",
                                  overflow: TextOverflow.ellipsis,
                                  style: TextStyle(
                                    color: isMasuk ? Colors.green : Colors.red,
                                    fontWeight: FontWeight.bold,
                                    fontSize: 11,
                                  ),
                                )),
                                IconButton(
                                  padding: EdgeInsets.zero,
                                  constraints: const BoxConstraints(),
                                  icon: const Icon(Icons.edit, size: 14, color: Colors.grey),
                                  onPressed: () => _editTransaksi(context, pp, _tabIndex == 1 ? realIdx : i),
                                ),
                                IconButton(
                                  padding: EdgeInsets.zero,
                                  constraints: const BoxConstraints(),
                                  icon: const Icon(Icons.delete_outline, size: 14, color: Colors.grey),
                                  onPressed: () => _konfirmasiHapus(context, pp, _tabIndex == 1 ? realIdx : i),
                                ),
                              ],
                            ),
                          ),
                          ), // ListTile
                        ), // Material
                      );
                    },
                  );
                }),
          ),
        ],
        ),
          ),  // close Expanded
        ],
      ), // close Column
      ), // close Scaffold body
    ), // close Scaffold
    ); // close MediaQuery
  }

  Widget _buildInventaris(AppProvider p) {
    return Column(
      children: [

        Container(
          color: const Color(0xFF631414),
          padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 6),
          child: Row(
            children: [
              Icon(Icons.inventory_2, color: Colors.white70, size: 14),
              const SizedBox(width: 6),
              Expanded(child: Text('${p.inventaris.length} ${p._t('barang','item','صنف')}  •  ${p._t("Geser kiri hapus","Swipe left to delete","اسحب لليسار للحذف")}',
                  overflow: TextOverflow.ellipsis,
                  style: const TextStyle(color: Colors.white70, fontSize: 11))),
              const SizedBox(width: 4),
              ElevatedButton(
                style: ElevatedButton.styleFrom(
                    backgroundColor: Colors.white,
                    foregroundColor: const Color(0xFF631414),
                    padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
                    minimumSize: const Size(0, 0),
                    tapTargetSize: MaterialTapTargetSize.shrinkWrap),
                onPressed: () => _showFormInventaris(context, p),
                child: Text('+ ${p.lTambah}', style: const TextStyle(fontSize: 11)),
              ),
            ],
          ),
        ),

        Expanded(
          child: p.inventaris.isEmpty
              ? Center(
                  child: Column(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: [
                      const Icon(Icons.inventory_2_outlined,
                          size: 64, color: Colors.grey),
                      const SizedBox(height: 12),
                      Text(p.lBelumAdaInventaris,
                          style: const TextStyle(color: Colors.grey, fontSize: 15)),
                      const SizedBox(height: 16),
                      ElevatedButton.icon(
                        style: ElevatedButton.styleFrom(
                            backgroundColor: const Color(0xFF631414),
                            foregroundColor: Colors.white),
                        icon: const Icon(Icons.add, size: 18),
                        label: Text(p.lTambahInventaris,
                            style: const TextStyle(fontSize: 14)),
                        onPressed: () => _showFormInventaris(context, p),
                      ),
                    ],
                  ),
                )
              : ListView.builder(
                  itemCount: p.inventaris.length,
                  itemBuilder: (_, i) {
                    final inv = p.inventaris[i];
                    final kondisi = inv['kondisi'] as String;
                    final kondisiColor = kondisi == 'Baik'
                        ? Colors.green
                        : kondisi == 'Rusak Ringan'
                            ? Colors.orange
                            : Colors.red;
                    return Dismissible(
                      key: ValueKey('inv-$i'),
                      direction: DismissDirection.endToStart,
                      background: Container(
                        alignment: Alignment.centerRight,
                        padding: const EdgeInsets.only(right: 20),
                        color: Colors.red,
                        child: Icon(Icons.delete, color: Colors.white),
                      ),
                      onDismissed: (_) => p.deleteInventaris(i),
                      child: Material(
                        color: Colors.white,
                        child: ListTile(
                          leading: CircleAvatar(
                            backgroundColor: kondisiColor.withValues(alpha: 0.15),
                            child: Icon(Icons.inventory_2,
                                color: kondisiColor, size: 20),
                          ),
                          title: Text(inv['nama'],
                              style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 14)),
                          subtitle: Column(
                            crossAxisAlignment: CrossAxisAlignment.start,
                            children: [
                              Text(
                                '${inv['jumlah']} ${inv['satuan']}  •  ${inv['keterangan']}',
                                style: const TextStyle(fontSize: 12),
                              ),
                              if ((inv['tgl'] ?? '').isNotEmpty)
                                Text(
                                  p.formatTgl(inv['tgl'] ?? ''),
                                  style: const TextStyle(fontSize: 11, color: Colors.grey),
                                ),
                            ],
                          ),
                          trailing: SizedBox(
                            width: 90,
                            child: Row(
                              mainAxisSize: MainAxisSize.min,
                              children: [
                                Flexible(
                                  child: Container(
                                    padding: const EdgeInsets.symmetric(horizontal: 4, vertical: 2),
                                    decoration: BoxDecoration(
                                      color: kondisiColor.withValues(alpha: 0.15),
                                      borderRadius: BorderRadius.circular(4),
                                    ),
                                    child: Text(p.terjemahKondisi(kondisi),
                                      overflow: TextOverflow.ellipsis,
                                      style: TextStyle(fontSize: 9, color: kondisiColor, fontWeight: FontWeight.bold)),
                                  ),
                                ),
                                IconButton(
                                  padding: EdgeInsets.zero,
                                  constraints: const BoxConstraints(),
                                  icon: const Icon(Icons.edit, size: 16, color: Colors.grey),
                                  onPressed: () => _showFormInventaris(context, p, editIndex: i),
                                ),
                              ],
                            ),
                          ),
                        ),
                      ),
                    );
                  },
                ),
        ),
      ],
    );
  }

  void _showFormInventaris(BuildContext context, AppProvider p, {int? editIndex}) {
    final inv = editIndex != null ? p.inventaris[editIndex] : null;
    _invNamaCtrl.text = inv?['nama'] ?? '';
    _invJmlCtrl.text  = inv?['jumlah']?.toString() ?? '1';
    _invSatCtrl.text  = inv?['satuan'] ?? 'unit';
    _invKetCtrl.text  = inv?['keterangan'] ?? '';
    String kondisi    = inv?['kondisi'] ?? 'Baik';

    showDialog(
      context: context,
      builder: (_) => StatefulBuilder(
        builder: (ctx, setLocal) => AlertDialog(
          title: Text(editIndex != null ? p.lEditInventaris : p.lTambahInventaris),
          content: SingleChildScrollView(
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                TextField(controller: _invNamaCtrl,
                    decoration: InputDecoration(labelText: p.lNamaBarang, isDense: true)),
                const SizedBox(height: 8),
                Row(children: [
                  Expanded(child: TextField(controller: _invJmlCtrl,
                      keyboardType: TextInputType.number,
                      decoration: InputDecoration(labelText: p.lJumlah, isDense: true))),
                  const SizedBox(width: 8),
                  Expanded(child: TextField(controller: _invSatCtrl,
                      decoration: InputDecoration(labelText: p.lSatuan, isDense: true))),
                ]),
                const SizedBox(height: 8),
                DropdownButtonFormField<String>(
                  value: kondisi,
                  decoration: InputDecoration(labelText: p.lKondisi, isDense: true),
                  items: [
                    DropdownMenuItem(value: 'Baik',         child: Text(p._t('Baik','Good','جيد'))),
                    DropdownMenuItem(value: 'Rusak Ringan', child: Text(p._t('Rusak Ringan','Minor Damage','تلف خفيف'))),
                    DropdownMenuItem(value: 'Rusak Berat',  child: Text(p._t('Rusak Berat','Major Damage','تلف شديد'))),
                  ],
                  onChanged: (v) => setLocal(() => kondisi = v!),
                ),
                const SizedBox(height: 8),
                TextField(controller: _invKetCtrl,
                    decoration: InputDecoration(labelText: p.lKeterangan, isDense: true)),
              ],
            ),
          ),
          actions: [
            TextButton(
              onPressed: () => Navigator.pop(ctx),
              child: Text(p.lBatal),
            ),
            ElevatedButton(
              style: ElevatedButton.styleFrom(
                  backgroundColor: const Color(0xFF1a5276),
                  foregroundColor: Colors.white),
              onPressed: () {
                if (_invNamaCtrl.text.trim().isEmpty) {
                  ScaffoldMessenger.of(ctx).showSnackBar(SnackBar(
                    content: Text(p.lNamaBarangKosong),
                    backgroundColor: Colors.orange,
                  ));
                  return;
                }
                final jml = int.tryParse(_invJmlCtrl.text.trim()) ?? 1;
                if (editIndex != null) {
                  p.editInventaris(editIndex, _invNamaCtrl.text.trim(),
                      jml, _invSatCtrl.text.trim(), kondisi, _invKetCtrl.text.trim());
                } else {
                  p.addInventaris(_invNamaCtrl.text.trim(),
                      jml, _invSatCtrl.text.trim(), kondisi, _invKetCtrl.text.trim());
                }
                Navigator.pop(ctx);
              },
              child: Text(editIndex != null ? p.lSimpan : p.lTambah),
            ),
          ],
        ),
      ),
    );
  }

  Widget _summaryBox(String label, double value, Color color) => Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          FittedBox(fit: BoxFit.scaleDown, child: Text(label,
              style: const TextStyle(color: Colors.white70, fontSize: 12))),
          const SizedBox(height: 4),
          FittedBox(fit: BoxFit.scaleDown, child: Text(
            "Rp ${NumberFormat('#,##0', 'id').format(value)}",
            style: TextStyle(
                color: color, fontWeight: FontWeight.bold, fontSize: 14),
          )),
        ],
      );
  }


// ── Widget Video Player untuk Live Slide ──
class _VideoSlideWidget extends StatefulWidget {
  final String path;
  final String judul;
  final String tgl;
  final bool isPaused;
  final VoidCallback? onVideoEnd;
  const _VideoSlideWidget({required this.path, required this.judul, this.tgl = '', this.isPaused = false, this.onVideoEnd});
  @override
  State<_VideoSlideWidget> createState() => _VideoSlideWidgetState();
}

class _VideoSlideWidgetState extends State<_VideoSlideWidget> {
  VideoPlayerController? _ctrl;
  bool _ready = false;
  bool _error = false;

  @override
  void initState() {
    super.initState();
    _initVideo();
  }

  Future<void> _initVideo() async {
    try {
      // Coba file dulu (Android), kalau gagal set error
      VideoPlayerController ctrl;
      try {
        ctrl = VideoPlayerController.file(File(widget.path));
        await ctrl.initialize();
      } catch (e) {
        // Web tidak support file:// langsung
        if (mounted) {
          setState(() => _error = true);
          // Auto skip ke slide berikutnya setelah 3 detik
          Future.delayed(const Duration(seconds: 3), () {
            if (mounted) widget.onVideoEnd?.call();
          });
        }
        return;
      }
      ctrl.setLooping(false);
      ctrl.addListener(() {
        if (_ctrl != null &&
            _ctrl!.value.position >= _ctrl!.value.duration &&
            _ctrl!.value.duration > Duration.zero &&
            !_ctrl!.value.isPlaying) {
          widget.onVideoEnd?.call();
        }
      });
      ctrl.play();
      if (mounted) setState(() { _ctrl = ctrl; _ready = true; });
    } catch (_) {
      if (mounted) {
        setState(() => _error = true);
        Future.delayed(const Duration(seconds: 3), () {
          if (mounted) widget.onVideoEnd?.call();
        });
      }
    }
  }

  @override
  void didUpdateWidget(_VideoSlideWidget old) {
    super.didUpdateWidget(old);
    if (old.isPaused != widget.isPaused) {
      if (widget.isPaused) {
        _ctrl?.pause();
      } else {
        _ctrl?.play();
      }
    }
  }

  @override
  void dispose() {
    _ctrl?.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (_error) {
      return Container(color: Colors.black,
        child: const Center(child: Icon(Icons.videocam_off, color: Colors.white38, size: 60)));
    }
    if (!_ready || _ctrl == null) {
      return Container(color: Colors.black,
        child: const Center(child: CircularProgressIndicator(color: Colors.white38)));
    }
    return Container(
      color: Colors.black,
      child: Stack(fit: StackFit.expand, children: [
        Center(
          child: AspectRatio(
            aspectRatio: _ctrl!.value.aspectRatio,
            child: VideoPlayer(_ctrl!),
          ),
        ),
        // Judul + tanggal kiri bawah
        if (widget.judul.isNotEmpty || widget.tgl.isNotEmpty)
          Positioned(
            bottom: 16, left: 12, right: 70,
            child: Column(crossAxisAlignment: CrossAxisAlignment.start, mainAxisSize: MainAxisSize.min, children: [
              if (widget.judul.isNotEmpty)
                Text(widget.judul,
                  style: const TextStyle(color: Colors.white, fontSize: 16, fontWeight: FontWeight.bold,
                    shadows: [Shadow(color: Colors.black, blurRadius: 6)]),
                  maxLines: 2, overflow: TextOverflow.ellipsis),
              if (widget.tgl.isNotEmpty)
                Text(widget.tgl,
                  style: const TextStyle(color: Colors.white70, fontSize: 12,
                    shadows: [Shadow(color: Colors.black, blurRadius: 4)])),
            ]),
          ),

        // Foto profil masjid KANAN TENGAH - style TikTok
        Positioned(
          right: 10,
          top: 0, bottom: 0,
          child: Center(
            child: Consumer<AppProvider>(builder: (_, p, __) => Container(
              decoration: BoxDecoration(
                shape: BoxShape.circle,
                border: Border.all(color: Colors.white, width: 2.5),
                boxShadow: [BoxShadow(color: Colors.black54, blurRadius: 8, spreadRadius: 1)],
              ),
              child: CircleAvatar(
                radius: 28,
                backgroundColor: const Color(0xFF631414),
                backgroundImage: p.fotoMasjid.isNotEmpty
                    ? MemoryImage(base64Decode(p.fotoMasjid)) : null,
                child: p.fotoMasjid.isEmpty
                    ? const Icon(Icons.mosque, color: Colors.white, size: 26) : null,
              ),
            )),
          ),
        ),
      ]),
    );
  }
}

// ══════════════════════════════════════════════
// HALAMAN JADWAL IMAM & KHATIB JUMAT
// ══════════════════════════════════════════════
class JadwalJumatPage extends StatefulWidget {
  final VoidCallback? onBack;
  const JadwalJumatPage({super.key, this.onBack});
  @override
  State<JadwalJumatPage> createState() => _JadwalJumatPageState();
}

class _JadwalJumatPageState extends State<JadwalJumatPage> {
  final _tglCtrl    = TextEditingController();
  final _imamCtrl   = TextEditingController();
  final _khatibCtrl = TextEditingController();
  final _temaCtrl   = TextEditingController();

  @override
  void dispose() {
    _tglCtrl.dispose();
    _imamCtrl.dispose();
    _khatibCtrl.dispose();
    _temaCtrl.dispose();
    super.dispose();
  }

  void _showForm(BuildContext context, AppProvider p, {int? editIndex}) {
    if (editIndex != null) {
      final d = p.jadwalJumat[editIndex];
      _tglCtrl.text    = d['tanggal'] ?? '';
      _imamCtrl.text   = d['imam'] ?? '';
      _khatibCtrl.text = d['khatib'] ?? '';
      _temaCtrl.text   = d['tema'] ?? '';
    } else {
      _tglCtrl.clear();
      _imamCtrl.clear();
      _khatibCtrl.clear();
      _temaCtrl.clear();
    }
    showDialog(
      context: context,
      builder: (ctx) => StatefulBuilder(
        builder: (ctx2, setS) => Dialog(
        insetPadding: EdgeInsets.fromLTRB(
          16, 16, 16, MediaQuery.of(ctx2).viewInsets.bottom + 16),
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
        child: SingleChildScrollView(
          child: Padding(
            padding: const EdgeInsets.all(16),
            child: Column(mainAxisSize: MainAxisSize.min, children: [
              Row(children: [
                const Icon(Icons.mosque, color: kPrimary),
                const SizedBox(width: 8),
                Expanded(child: Text(
                  editIndex != null
                    ? p._t('Edit Jadwal','Edit Schedule','تعديل الجدول')
                    : p._t('Tambah Jadwal','Add Schedule','إضافة جدول'),
                  style: const TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
                )),
              ]),
              const SizedBox(height: 12),
              TextField(
                controller: _tglCtrl,
                readOnly: true,
                decoration: InputDecoration(
                  labelText: p._t('Tanggal','Date','التاريخ'),
                  prefixIcon: const Icon(Icons.calendar_today, color: kPrimary),
                  border: OutlineInputBorder(borderRadius: BorderRadius.circular(8)),
                ),
                onTap: () async {
                  final picked = await showDatePicker(
                    context: context,
                    initialDate: DateTime.now(),
                    firstDate: DateTime(2024),
                    lastDate: DateTime(2030),
                  );
                  if (picked != null) {
                    _tglCtrl.text =
                      '${picked.day.toString().padLeft(2,'0')}/'
                      '${picked.month.toString().padLeft(2,'0')}/'
                      '${picked.year}';
                  }
                },
              ),
              const SizedBox(height: 8),
              TextField(
                controller: _imamCtrl,
                textCapitalization: TextCapitalization.words,
                decoration: InputDecoration(
                  labelText: p.lImam,
                  prefixIcon: const Icon(Icons.person, color: kPrimary),
                  border: OutlineInputBorder(borderRadius: BorderRadius.circular(8)),
                ),
              ),
              const SizedBox(height: 8),
              TextField(
                controller: _khatibCtrl,
                textCapitalization: TextCapitalization.words,
                decoration: InputDecoration(
                  labelText: p.lKhatib,
                  prefixIcon: const Icon(Icons.mic, color: kPrimary),
                  border: OutlineInputBorder(borderRadius: BorderRadius.circular(8)),
                ),
              ),
              const SizedBox(height: 8),
              TextField(
                controller: _temaCtrl,
                textCapitalization: TextCapitalization.sentences,
                maxLines: 1,
                decoration: InputDecoration(
                  labelText: p.lTemaKhutbah,
                  prefixIcon: const Icon(Icons.book, color: kPrimary),
                  border: OutlineInputBorder(borderRadius: BorderRadius.circular(8)),
                ),
              ),
              const SizedBox(height: 12),
              Row(mainAxisAlignment: MainAxisAlignment.end, children: [
                TextButton(
                  onPressed: () => Navigator.pop(context),
                  child: Text(p.lBatal),
                ),
                const SizedBox(width: 8),
                ElevatedButton(
                  style: ElevatedButton.styleFrom(
                    backgroundColor: kPrimary,
                    foregroundColor: Colors.white,
                  ),
                  onPressed: () {
                    if (_tglCtrl.text.isEmpty ||
                        _imamCtrl.text.isEmpty ||
                        _khatibCtrl.text.isEmpty) return;
                    final data = {
                      "tanggal": _tglCtrl.text.trim(),
                      "imam": _imamCtrl.text.trim(),
                      "khatib": _khatibCtrl.text.trim(),
                      "tema": _temaCtrl.text.trim(),
                    };
                    if (editIndex != null) {
                      p.editJadwalJumat(editIndex, data);
                    } else {
                      p.tambahJadwalJumat(data);
                    }
                    Navigator.pop(context);
                  },
                  child: Text(editIndex != null ? p.lSimpan : p.lTambah),
                ),
              ]),
            ]),
          ),
        ),
      ),  // Dialog
      ),  // StatefulBuilder
    );
  }

  @override
  Widget build(BuildContext context) {
    final p = context.watch<AppProvider>();
    return Scaffold(
      backgroundColor: kBg,
      appBar: AppBar(
        backgroundColor: kPrimary,
        foregroundColor: Colors.white,
        leading: IconButton(
          icon: const Icon(Icons.arrow_back),
          onPressed: () => widget.onBack != null
            ? widget.onBack!()
            : Navigator.pop(context),
        ),
        title: Text(p.lJadwalJumat,
          style: const TextStyle(fontWeight: FontWeight.bold)),
        actions: [
          IconButton(
            icon: const Icon(Icons.add),
            tooltip: p._t('Tambah','Add','إضافة'),
            onPressed: () => _showForm(context, p),
          ),
        ],
      ),
      body: ZoomWrapper(
        child: p.jadwalJumat.isEmpty
          ? Center(child: Column(mainAxisSize: MainAxisSize.min, children: [
              const Icon(Icons.mosque, size: 80, color: Colors.grey),
              const SizedBox(height: 12),
              Text(
                p._t('Belum ada jadwal Jumat',
                     'No Friday schedule yet',
                     'لا جدول جمعة بعد'),
                style: const TextStyle(color: Colors.grey, fontSize: 15),
              ),
              const SizedBox(height: 20),
              ElevatedButton.icon(
                style: ElevatedButton.styleFrom(
                  backgroundColor: kPrimary,
                  foregroundColor: Colors.white,
                ),
                icon: const Icon(Icons.add),
                label: Text(p._t('Tambah Jadwal','Add Schedule','إضافة جدول')),
                onPressed: () => _showForm(context, p),
              ),
            ]))
          : ListView.builder(
              padding: const EdgeInsets.all(12),
              itemCount: p.jadwalJumat.length,
              itemBuilder: (ctx, i) {
                final d = p.jadwalJumat[i];
                final tgl = p.formatTgl(d['tanggal'] ?? '');
                return Card(
                  margin: const EdgeInsets.only(bottom: 10),
                  shape: RoundedRectangleBorder(
                    borderRadius: BorderRadius.circular(12)),
                  child: ListTile(
                    contentPadding: const EdgeInsets.symmetric(
                      horizontal: 16, vertical: 8),
                    leading: Container(
                      width: 48, height: 48,
                      decoration: BoxDecoration(
                        color: kPrimary.withValues(alpha: 0.1),
                        borderRadius: BorderRadius.circular(8),
                      ),
                      child: const Icon(Icons.mosque, color: kPrimary),
                    ),
                    title: Text(tgl,
                      style: const TextStyle(
                        fontWeight: FontWeight.bold, fontSize: 13)),
                    subtitle: Column(
                      crossAxisAlignment: CrossAxisAlignment.start,
                      children: [
                        const SizedBox(height: 4),
                        Row(children: [
                          const Icon(Icons.person, size: 13, color: Colors.grey),
                          const SizedBox(width: 4),
                          Expanded(child: Text('${p.lImam}: ${d['imam']}',
                            overflow: TextOverflow.ellipsis,
                            style: const TextStyle(fontSize: 12))),
                        ]),
                        Row(children: [
                          const Icon(Icons.mic, size: 13, color: Colors.grey),
                          const SizedBox(width: 4),
                          Expanded(child: Text('${p.lKhatib}: ${d['khatib']}',
                            overflow: TextOverflow.ellipsis,
                            style: const TextStyle(fontSize: 12))),
                        ]),
                        if ((d['tema'] ?? '').isNotEmpty)
                          Row(children: [
                            const Icon(Icons.book, size: 13, color: Colors.grey),
                            const SizedBox(width: 4),
                            Expanded(child: Text(
                              '${p.lTemaKhutbah}: ${d['tema']}',
                              style: const TextStyle(fontSize: 12),
                              overflow: TextOverflow.ellipsis,
                            )),
                          ]),
                      ],
                    ),
                    trailing: Row(mainAxisSize: MainAxisSize.min, children: [
                      IconButton(
                        icon: const Icon(Icons.edit, size: 18, color: Colors.grey),
                        onPressed: () => _showForm(context, p, editIndex: i),
                        padding: EdgeInsets.zero,
                        constraints: const BoxConstraints(),
                      ),
                      const SizedBox(width: 8),
                      IconButton(
                        icon: const Icon(Icons.delete_outline,
                          size: 18, color: Colors.red),
                        onPressed: () async {
                          final ok = await showDialog<bool>(
                            context: context,
                            builder: (_) => AlertDialog(
                              title: Text(p._t(
                                'Hapus Jadwal','Delete Schedule','حذف الجدول')),
                              content: Text(
                                '${d['imam']} - ${d['khatib']}'),
                              actions: [
                                TextButton(
                                  onPressed: () => Navigator.pop(context, false),
                                  child: Text(p.lBatal),
                                ),
                                TextButton(
                                  onPressed: () => Navigator.pop(context, true),
                                  child: Text(p.lHapus,
                                    style: const TextStyle(color: Colors.red)),
                                ),
                              ],
                            ),
                          );
                          if (ok == true) p.hapusJadwalJumat(i);
                        },
                        padding: EdgeInsets.zero,
                        constraints: const BoxConstraints(),
                      ),
                    ]),
                  ),
                );
              },
            ),
      ),
      floatingActionButton: FloatingActionButton.extended(
        backgroundColor: kPrimary,
        foregroundColor: Colors.white,
        icon: const Icon(Icons.add),
        label: Text(p._t('Tambah','Add','إضافة')),
        onPressed: () => _showForm(context, p),
      ),
    );
  }
}
class KalkulatorPage extends StatefulWidget {
  const KalkulatorPage({super.key});
  @override
  State<KalkulatorPage> createState() => _KalkulatorPageState();
}

class _KalkulatorPageState extends State<KalkulatorPage> {
  String _display = "0";
  String _ekspresi = "";
  double _val1 = 0;
  String _op = "";
  bool _newInput = true;

  // Konversi angka ke Arab jika bahasa Arab
  static const _arabDigits = ['٠','١','٢','٣','٤','٥','٦','٧','٨','٩'];
  String _toDisplay(String s, String bahasa) {
    if (bahasa != 'ar') return s;
    return s.characters.map((c) {
      final d = int.tryParse(c);
      return d != null ? _arabDigits[d] : c;
    }).join();
  }
  String _opLabel(String op) {
    if (op == "x") return "×";
    if (op == "/") return "÷";
    return op;
  }

  void _onDigit(String d) {
    setState(() {
      if (_newInput) {
        _display = d;
        _newInput = false;
      } else {
        _display = _display == "0" ? d : _display + d;
      }
    });
  }

  void _onDot() {
    setState(() {
      if (_newInput) {
        _display = "0.";
        _newInput = false;
      } else if (!_display.contains('.')) {
        _display += '.';
      }
    });
  }

  void _onOp(String op) {
    setState(() {
      _val1 = double.tryParse(_display) ?? 0;
      _ekspresi = "$_display ${_opLabel(op)}";
      _op = op;
      _newInput = true;
    });
  }

  void _onEqual() {
    setState(() {
      final val2 = double.tryParse(_display) ?? 0;
      double result = 0;
      switch (_op) {
        case "+": result = _val1 + val2; break;
        case "-": result = _val1 - val2; break;
        case "x": result = _val1 * val2; break;
        case "/": result = val2 != 0 ? _val1 / val2 : 0; break;
      }
      final resultStr = result % 1 == 0
          ? result.toInt().toString()
          : result.toStringAsFixed(4);
      _ekspresi = "$_ekspresi $_display = $resultStr";
      _display = resultStr;
      _op = "";
      _newInput = true;
    });
  }

  void _onClear() {
    setState(() {
      _display = "0";
      _ekspresi = "";
      _val1 = 0;
      _op = "";
      _newInput = true;
    });
  }

  Widget _btn(String label, {Color? color, VoidCallback? onTap}) {
    return Expanded(
      child: Padding(
        padding: const EdgeInsets.all(4),
        child: ElevatedButton(
          style: ElevatedButton.styleFrom(
            backgroundColor: color ?? const Color(0xFF631414),
            foregroundColor: Colors.white,
            padding: const EdgeInsets.symmetric(vertical: 18),
            shape: RoundedRectangleBorder(
                borderRadius: BorderRadius.circular(8)),
          ),
          onPressed: onTap,
          child: Text(label, style: const TextStyle(fontSize: 20)),
        ),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    final p = context.watch<AppProvider>();
    final bahasa = p.bahasa;
    return Scaffold(
      appBar: AppBar(title: Text(p._t("KALKULATOR","CALCULATOR","الآلة الحاسبة"))),
      body: ZoomWrapper(child: Column(
        children: [
          Container(
            width: double.infinity,
            padding: const EdgeInsets.fromLTRB(24, 20, 24, 20),
            color: Colors.black,
            child: Column(crossAxisAlignment: CrossAxisAlignment.end, children: [
              // Ekspresi: 5 × 5 = 25
              Text(
                _ekspresi.isEmpty ? "" : _toDisplay(_ekspresi, bahasa),
                textAlign: TextAlign.right,
                style: const TextStyle(color: Colors.white54, fontSize: 18),
                maxLines: 2,
                overflow: TextOverflow.ellipsis,
              ),
              const SizedBox(height: 4),
              // Angka aktif
              FittedBox(
                fit: BoxFit.scaleDown,
                alignment: Alignment.centerRight,
                child: Text(
                  _toDisplay(_display, bahasa),
                  textAlign: TextAlign.right,
                  style: TextStyle(
                    color: _op.isNotEmpty && _newInput ? Colors.white54 : Colors.white,
                    fontSize: 56,
                    fontWeight: FontWeight.bold,
                  ),
                ),
              ),
              // Operator aktif
              if (_op.isNotEmpty && _newInput)
                Align(
                  alignment: Alignment.centerRight,
                  child: Container(
                    padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 2),
                    decoration: BoxDecoration(
                      color: kPrimary.withValues(alpha: 0.7),
                      borderRadius: BorderRadius.circular(4),
                    ),
                    child: Text(_opLabel(_op),
                      style: const TextStyle(color: Colors.white, fontSize: 16, fontWeight: FontWeight.bold)),
                  ),
                ),
            ]),
          ),
          const Divider(height: 1),
          Expanded(
            child: Column(
              children: [
                Expanded(
                  child: Row(children: [
                    _btn("C", color: Colors.grey.shade700, onTap: _onClear),
                    _btn("+/-",
                        color: Colors.grey.shade700,
                        onTap: () => setState(() => _display =
                            (-(double.tryParse(_display) ?? 0)).toString())),
                    _btn("%",
                        color: Colors.grey.shade700,
                        onTap: () => setState(() => _display =
                            ((double.tryParse(_display) ?? 0) / 100)
                                .toString())),
                    _btn("/",
                        color: Colors.orange, onTap: () => _onOp("/")),
                  ]),
                ),
                Expanded(
                  child: Row(children: [
                    _btn(bahasa == 'ar' ? '٧' : "7", onTap: () => _onDigit("7")),
                    _btn(bahasa == 'ar' ? '٨' : "8", onTap: () => _onDigit("8")),
                    _btn(bahasa == 'ar' ? '٩' : "9", onTap: () => _onDigit("9")),
                    _btn("x",
                        color: Colors.orange, onTap: () => _onOp("x")),
                  ]),
                ),
                Expanded(
                  child: Row(children: [
                    _btn(bahasa == 'ar' ? '٤' : "4", onTap: () => _onDigit("4")),
                    _btn(bahasa == 'ar' ? '٥' : "5", onTap: () => _onDigit("5")),
                    _btn(bahasa == 'ar' ? '٦' : "6", onTap: () => _onDigit("6")),
                    _btn("-",
                        color: Colors.orange, onTap: () => _onOp("-")),
                  ]),
                ),
                Expanded(
                  child: Row(children: [
                    _btn(bahasa == 'ar' ? '١' : "1", onTap: () => _onDigit("1")),
                    _btn(bahasa == 'ar' ? '٢' : "2", onTap: () => _onDigit("2")),
                    _btn(bahasa == 'ar' ? '٣' : "3", onTap: () => _onDigit("3")),
                    _btn("+",
                        color: Colors.orange, onTap: () => _onOp("+")),
                  ]),
                ),
                Expanded(
                  child: Row(children: [
                    _btn(bahasa == 'ar' ? '٠' : "0", onTap: () => _onDigit("0")),
                    _btn(".", onTap: _onDot),
                    _btn("DEL",
                        color: Colors.grey.shade700,
                        onTap: () {
                          setState(() {
                            if (_display.length > 1) {
                              _display = _display.substring(0, _display.length - 1);
                            } else {
                              _display = "0";
                            }
                          });
                        }),
                    _btn("=", color: Colors.green, onTap: _onEqual),
                  ]),
                ),
              ],
            ),
          ),
        ],
      ),
    ));
  }
}

