import 'package:flutter/material.dart';
import 'package:nfc_manager/nfc_manager.dart';

void main() {
  runApp(const WalCheckApp());
}

class WalCheckApp extends StatelessWidget {
  const WalCheckApp({Key? key}) : super(key:Key? key);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'WALCHECK',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        primarySwatch: Colors.blue,
        scaffoldBackgroundColor: const Color(0xFFF4F6F9),
      ),
      home: const WalCheckHomePage(),
    );
  }
}

// Model data kartu
class CardItem {
  final String id;
  final String name;
  final IconData icon;
  String? nfcTagId; // UID dari NFC Tag yang ditempelkan
  bool isScanned;

  CardItem({
    required this.id,
    required this.name,
    required this.icon,
    this.nfcTagId,
    this.isScanned = false,
  });
}

class WalCheckHomePage extends StatefulWidget {
  const WalCheckHomePage({Key? key}) : super(key: key);

  @override
  State<WalCheckHomePage> createState() => _WalCheckHomePageState();
}

class _WalCheckHomePageState extends State<WalCheckHomePage> {
  bool _isNfcAvailable = false;
  bool _isScanning = false;

  // 4 Kategori Kartu sesuai spesifikasi
  final List<CardItem> _cards = [
    CardItem(id: 'ktm', name: 'KTM (Kartu Tanda Mahasiswa)', icon: Icons.school),
    CardItem(id: 'ktp', name: 'KTP (Kartu Tanda Penduduk)', icon: Icons.badge),
    CardItem(id: 'atm', name: 'Kartu ATM', icon: Icons.credit_card),
    CardItem(id: 'sim', name: 'SIM (Surat Izin Mengemudi)', icon: Icons.directions_car),
  ];

  @override
  void initState() {
    super.initState();
    _checkNfcAvailability();
  }

  // Memeriksa apakah HP mendukung dan menyalakan NFC
  Future<void> _checkNfcAvailability() async {
    bool isAvailable = await NfcManager.instance.isAvailable();
    setState(() {
      _isNfcAvailable = isAvailable;
    });
  }

  // Memulai Proses Pemindaian / Scanning NFC
  void _startNfcSession() async {
    if (!_isNfcAvailable) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(
          content: Text('Fitur NFC tidak tersedia atau belum diaktifkan di HP Anda!'),
          backgroundColor: Colors.red,
        ),
      );
      return;
    }

    setState(() {
      _isScanning = true;
    });

    NfcManager.instance.startSession(
      onDiscovered: (NfcTag tag) async {
        // Mendapatkan Identifier / UID unik dari NFC Tag Pasif
        String tagId = _extractTagIdentifier(tag);

        setState(() {
          // Logika pendaftaran otomatis jika belum di-pair ke kartu tertentu
          _processScannedTag(tagId);
        });
      },
    );
  }

  // Memproses Tag yang berhasil di-scan
  void _processScannedTag(String tagId) {
    // 1. Cek apakah NFC Tag sudah terdaftar di salah satu kartu
    int foundIndex = _cards.indexWhere((card) => card.nfcTagId == tagId);

    if (foundIndex != -1) {
      // Jika sudah terdaftar, tandai sebagai TERDETEKSI / SUDAH DIBAWA
      _cards[foundIndex].isScanned = true;
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(
          content: Text('${_cards[foundIndex].name} berhasil terdeteksi!'),
          backgroundColor: Colors.green,
          duration: const Duration(seconds: 1),
        ),
      );
    } else {
      // Jika Tag belum terdaftar, daftarkan ke kartu pertama yang belum memiliki Tag
      int unmappedIndex = _cards.indexWhere((card) => card.nfcTagId == null);
      if (unmappedIndex != -1) {
        _cards[unmappedIndex].nfcTagId = tagId;
        _cards[unmappedIndex].isScanned = true;
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(
            content: Text('Tag baru terikat ke ${_cards[unmappedIndex].name}!'),
            backgroundColor: Colors.blue,
            duration: const Duration(seconds: 2),
          ),
        );
      } else {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(
            content: Text('NFC Tag tidak dikenali dalam daftar WALCHECK.'),
            backgroundColor: Colors.orange,
          ),
        );
      }
    }
  }

  // Mengambil ID unik dari Payload NFC Tag
  String _extractTagIdentifier(NfcTag tag) {
    Map<String, dynamic> data = tag.data;
    if (data.containsKey('isodep')) {
      return data['isodep']['identifier'].toString();
    } else if (data.containsKey('nfca')) {
      return data['nfca']['identifier'].toString();
    } else if (data.containsKey('nfcb')) {
      return data['nfcb']['identifier'].toString();
    } else if (data.containsKey('mifare')) {
      return data['mifare']['identifier'].toString();
    }
    return tag.data.toString();
  }

  // Menghentikan Pemindaian
  void _stopNfcSession() {
    NfcManager.instance.stopSession();
    setState(() {
      _isScanning = false;
    });
    _showSummaryDialog();
  }

  // Mengubah status scanning secara toggle
  void _toggleNfcScanning() {
    if (_isScanning) {
      _stopNfcSession();
    } else {
      _startNfcSession();
    }
  }

  // Reset Hasil Scan untuk pemeriksaan ulang
  void _resetScan() {
    setState(() {
      for (var card in _cards) {
        card.isScanned = false;
      }
    });
  }

  // Menampilkan Dialog Hasil/Notifikasi jika ada kartu yang belum ter-scan
  void _showSummaryDialog() {
    List<CardItem> missingCards = _cards.where((card) => !card.isScanned).toList();

    showDialog(
      context: context,
      builder: (BuildContext context) {
        return AlertDialog(
          title: Row(
            children: [
              Icon(
                missingCards.isEmpty ? Icons.check_circle : Icons.warning_amber_rounded,
                color: missingCards.isEmpty ? Colors.green : Colors.orange,
              ),
              const SizedBox(width: 8),
              Text(missingCards.isEmpty ? 'Dompet Lengkap!' : 'Kartu Tertinggal!'),
            ],
          ),
          content: Column(
            mainAxisSize: MainAxisSize.min,
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              if (missingCards.isEmpty)
                const Text('Semua kartu penting (KTM, KTP, ATM, SIM) sudah terdeteksi di dalam dompet.')
              else ...[
                const Text('Perhatian! Kartu berikut BELUM terdeteksi:'),
                const SizedBox(height: 10),
                ...missingCards.map((card) => Padding(
                      padding: const EdgeInsets.symmetric(vertical: 4),
                      child: Row(
                        children: [
                          const Icon(Icons.close, color: Colors.red, size: 18),
                          const SizedBox(width: 8),
                          Expanded(
                            child: Text(
                              'Kartu ${card.name} belum terdeteksi',
                              style: const TextStyle(fontWeight: FontWeight.bold, color: Colors.red),
                            ),
                          ),
                        ],
                      ),
                    )),
              ]
            ],
          ),
          actions: [
            TextButton(
              onPressed: () => Navigator.of(context).pop(),
              child: const Text('OK'),
            ),
          ],
        );
      },
    );
  }

  @override
  Widget build(BuildContext context) {
    int scannedCount = _cards.where((c) => c.isScanned).length;

    return Scaffold(
      appBar: AppBar(
        title: const Text('WALCHECK', style: TextStyle(fontWeight: FontWeight.bold)),
        centerTitle: true,
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh),
            onPressed: _resetScan,
            tooltip: 'Reset Scan',
          )
        ],
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          children: [
            // Banner Status NFC HP
            Container(
              padding: const EdgeInsets.all(12),
              decoration: BoxDecoration(
                color: _isNfcAvailable ? Colors.green.shade50 : Colors.red.shade50,
                borderRadius: BorderRadius.circular(10),
                border: Border.all(color: _isNfcAvailable ? Colors.green : Colors.red),
              ),
              child: Row(
                children: [
                  Icon(
                    _isNfcAvailable ? Icons.nfc : Icons.nfc_outlined,
                    color: _isNfcAvailable ? Colors.green : Colors.red,
                  ),
                  const SizedBox(width: 10),
                  Expanded(
                    child: Text(
                      _isNfcAvailable
                          ? 'Sistem NFC Siap Digunakan'
                          : 'NFC Belum Aktif / Tidak Didukung HP Ini',
                      style: TextStyle(
                        color: _isNfcAvailable ? Colors.green.shade900 : Colors.red.shade900,
                        fontWeight: FontWeight.w600,
                      ),
                    ),
                  ),
                ],
              ),
            ),
            const SizedBox(height: 16),

            // Indikator Progress Scan
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                const Text(
                  'Daftar Kartu Dompet:',
                  style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
                ),
                Text(
                  '$scannedCount / ${_cards.length} Terdeteksi',
                  style: const TextStyle(fontWeight: FontWeight.bold, color: Colors.blue),
                ),
              ],
            ),
            const SizedBox(height: 10),

            // List Kartu (KTM, KTP, ATM, SIM)
            Expanded(
              child: ListView.builder(
                itemCount: _cards.length,
                itemBuilder: (context, index) {
                  final card = _cards[index];
                  return Card(
                    elevation: 2,
                    margin: const EdgeInsets.symmetric(vertical: 6),
                    shape: RoundedRectangleBorder(
                      borderRadius: BorderRadius.circular(12),
                    ),
                    child: ListTile(
                      leading: CircleAvatar(
                        backgroundColor: card.isScanned ? Colors.green.shade100 : Colors.grey.shade200,
                        child: Icon(
                          card.icon,
                          color: card.isScanned ? Colors.green : Colors.grey,
                        ),
                      ),
                      title: Text(
                        card.name,
                        style: const TextStyle(fontWeight: FontWeight.bold),
                      ),
                      subtitle: Text(
                        card.isScanned
                            ? 'Kartu sudah dibawa'
                            : 'Belum terdeteksi (Tempelkan NFC Tag)',
                        style: TextStyle(
                          color: card.isScanned ? Colors.green : Colors.red,
                          fontSize: 12,
                        ),
                      ),
                      trailing: card.isScanned
                          ? const Icon(Icons.check_circle, color: Colors.green, size: 28)
                          : const Icon(Icons.radio_button_unchecked, color: Colors.grey, size: 28),
                    ),
                  );
                },
              ),
            ),

            // Tombol Mulai Scan NFC
            SizedBox(
              width: double.infinity,
              height: 52,
              child: ElevatedButton.icon(
                onPressed: _toggleNfcScanning,
                style: ElevatedButton.styleFrom(
                  backgroundColor: _isScanning ? Colors.orange : Colors.blue,
                  shape: RoundedRectangleBorder(
                    borderRadius: BorderRadius.circular(12),
                  ),
                ),
                icon: Icon(_isScanning ? Icons.stop : Icons.center_focus_strong, color: Colors.white),
                label: Text(
                  _isScanning ? 'Selesai / Hentikan Periksa' : 'Mulai Pindai NFC Dompet',
                  style: const TextStyle(fontSize: 16, fontWeight: FontWeight.bold, color: Colors.white),
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
}
dependencies:
  flutter:
    sdk: flutter
  nfc_manager: ^3.3.0
  <uses-permission android:name="android.permission.NFC" />
<uses-feature android:name="android.hardware.nfc" android:required="true" />
