from flask import Flask, request, jsonify, render_template_string
import json
import os

app = Flask(__name__)

class Barang:
    def __init__(self, kode, nama, stok, kategori="Umum", harga=0):
        self.kode = kode
        self.nama = nama
        self.stok = stok
        self.kategori = kategori
        self.harga = harga

    def to_dict(self):
        return {
            "kode": self.kode,
            "nama": self.nama,
            "stok": self.stok,
            "kategori": self.kategori,
            "harga": self.harga,
            "nilai_inventori": self.stok * self.harga
        }
    
    def update(self, nama=None, stok=None, kategori=None, harga=None):
        if nama: self.nama = nama
        if stok is not None: self.stok = stok
        if kategori: self.kategori = kategori
        if harga is not None: self.harga = harga


class Inventori:
    def __init__(self, filename="inventori.json"):
        self.filename = filename
        self.data = {}
        self.load_data()
    
    def save_data(self):
        """Menyimpan data ke file JSON"""
        data_to_save = {}
        for kode, barang in self.data.items():
            data_to_save[kode] = barang.to_dict()
        
        with open(self.filename, 'w') as f:
            json.dump(data_to_save, f, indent=2)
    
    def load_data(self):
        """Memuat data dari file JSON"""
        if os.path.exists(self.filename):
            try:
                with open(self.filename, 'r') as f:
                    data = json.load(f)
                    for kode, item in data.items():
                        self.data[kode] = Barang(
                            kode=item['kode'],
                            nama=item['nama'],
                            stok=item['stok'],
                            kategori=item.get('kategori', 'Umum'),
                            harga=item.get('harga', 0)
                        )
            except Exception as e:
                print(f"Error loading data: {e}")
                self.data = {}

    def tambah_barang(self, kode, nama, stok, kategori="Umum", harga=0):
        if kode in self.data:
            return False, "Gagal: kode barang sudah ada"
        
        if not kode or not nama:
            return False, "Kode dan nama barang harus diisi"
        
        if stok < 0 or harga < 0:
            return False, "Stok dan harga tidak boleh negatif"
        
        self.data[kode] = Barang(kode, nama, stok, kategori, harga)
        self.save_data()
        return True, "Barang berhasil ditambahkan"

    def cari_barang(self, kode):
        return self.data.get(kode, None)

    def cari_by_nama(self, nama):
        """Mencari barang berdasarkan nama (partial match)"""
        results = []
        for barang in self.data.values():
            if nama.lower() in barang.nama.lower():
                results.append(barang.to_dict())
        return results

    def hapus_barang(self, kode):
        if kode in self.data:
            del self.data[kode]
            self.save_data()
            return True, "Barang berhasil dihapus"
        return False, "Barang tidak ditemukan"

    def tambah_stok(self, kode, jumlah):
        if kode in self.data:
            if jumlah <= 0:
                return False, "Jumlah harus positif"
            self.data[kode].stok += jumlah
            self.save_data()
            return True, f"Stok berhasil ditambahkan sebesar {jumlah}"
        return False, "Barang tidak ditemukan"

    def kurangi_stok(self, kode, jumlah):
        if kode in self.data:
            if jumlah <= 0:
                return False, "Jumlah harus positif"
            if self.data[kode].stok >= jumlah:
                self.data[kode].stok -= jumlah
                self.save_data()
                return True, f"Stok berhasil dikurangi sebesar {jumlah}"
            return False, "Stok tidak mencukupi"
        return False, "Barang tidak ditemukan"

    def update_barang(self, kode, **kwargs):
        if kode in self.data:
            self.data[kode].update(**kwargs)
            self.save_data()
            return True, "Barang berhasil diupdate"
        return False, "Barang tidak ditemukan"

    def semua_barang(self):
        return [b.to_dict() for b in self.data.values()]

    def total_inventori(self):
        """Menghitung total nilai inventori"""
        total = 0
        for barang in self.data.values():
            total += barang.stok * barang.harga
        return total
    
    def barang_kritis(self, batas=5):
        """Mendapatkan barang dengan stok di bawah batas"""
        return [b.to_dict() for b in self.data.values() if b.stok <= batas]

    def statistik(self):
        """Statistik inventori"""
        total_barang = len(self.data)
        total_stok = sum(b.stok for b in self.data.values())
        total_nilai = self.total_inventori()
        barang_kritis = len(self.barang_kritis())
        
        return {
            "total_barang": total_barang,
            "total_stok": total_stok,
            "total_nilai": total_nilai,
            "barang_kritis": barang_kritis
        }


inventori = Inventori()

# HTML Template untuk UI yang lebih baik
HTML_TEMPLATE = '''
<!DOCTYPE html>
<html>
<head>
    <title>Sistem Inventori Barang</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; background: #f5f5f5; }
        .container { max-width: 1200px; margin: 0 auto; background: white; padding: 20px; border-radius: 10px; box-shadow: 0 0 10px rgba(0,0,0,0.1); }
        h1 { color: #333; text-align: center; }
        .section { margin: 20px 0; padding: 20px; border: 1px solid #ddd; border-radius: 5px; }
        .form-group { margin: 10px 0; }
        label { display: inline-block; width: 150px; font-weight: bold; }
        input, select { padding: 5px; width: 300px; }
        button { padding: 10px 20px; background: #4CAF50; color: white; border: none; border-radius: 5px; cursor: pointer; }
        button:hover { background: #45a049; }
        .result { background: #f9f9f9; padding: 15px; border-left: 4px solid #4CAF50; margin: 10px 0; }
        .error { border-left-color: #f44336; }
        .success { border-left-color: #4CAF50; }
        table { width: 100%; border-collapse: collapse; margin: 10px 0; }
        th, td { padding: 10px; border: 1px solid #ddd; text-align: left; }
        th { background: #f2f2f2; }
        .stats { display: flex; justify-content: space-around; background: #e8f5e9; padding: 15px; border-radius: 5px; }
        .stat-item { text-align: center; }
        .stat-value { font-size: 24px; font-weight: bold; color: #2e7d32; }
        .nav { display: flex; gap: 10px; margin-bottom: 20px; }
        .nav button { background: #2196F3; }
        .nav button:hover { background: #1976D2; }
    </style>
</head>
<body>
    <div class="container">
        <h1>📦 Sistem Inventori Barang</h1>
        
        <div class="nav">
            <button onclick="showSection('home')">🏠 Beranda</button>
            <button onclick="showSection('tambah')">➕ Tambah Barang</button>
            <button onclick="showSection('cari')">🔍 Cari Barang</button>
            <button onclick="showSection('kelola')">🔄 Kelola Stok</button>
            <button onclick="showSection('lihat')">📋 Lihat Semua</button>
            <button onclick="showSection('statistik')">📊 Statistik</button>
        </div>
        
        <div id="home" class="section">
            <h2>Selamat Datang di Sistem Inventori</h2>
            <p>Sistem ini membantu Anda mengelola inventori barang dengan mudah.</p>
            <div class="stats">
                <div class="stat-item">
                    <div class="stat-value">{{ stats.total_barang }}</div>
                    <div>Jumlah Barang</div>
                </div>
                <div class="stat-item">
                    <div class="stat-value">{{ stats.total_stok }}</div>
                    <div>Total Stok</div>
                </div>
                <div class="stat-item">
                    <div class="stat-value">Rp {{ stats.total_nilai }}</div>
                    <div>Nilai Inventori</div>
                </div>
                <div class="stat-item">
                    <div class="stat-value" style="color: {{ 'red' if stats.barang_kritis > 0 else 'green' }}">{{ stats.barang_kritis }}</div>
                    <div>Barang Kritis</div>
                </div>
            </div>
        </div>
        
        <div id="tambah" class="section" style="display:none;">
            <h2>Tambah Barang Baru</h2>
            <form onsubmit="tambahBarang(event)">
                <div class="form-group">
                    <label>Kode Barang:</label>
                    <input type="text" id="kode" required>
                </div>
                <div class="form-group">
                    <label>Nama Barang:</label>
                    <input type="text" id="nama" required>
                </div>
                <div class="form-group">
                    <label>Stok Awal:</label>
                    <input type="number" id="stok" value="0" min="0" required>
                </div>
                <div class="form-group">
                    <label>Kategori:</label>
                    <select id="kategori">
                        <option value="Elektronik">Elektronik</option>
                        <option value="Pakaian">Pakaian</option>
                        <option value="Makanan">Makanan</option>
                        <option value="Minuman">Minuman</option>
                        <option value="Alat Tulis">Alat Tulis</option>
                        <option value="Lainnya" selected>Lainnya</option>
                    </select>
                </div>
                <div class="form-group">
                    <label>Harga (Rp):</label>
                    <input type="number" id="harga" value="0" min="0" required>
                </div>
                <button type="submit">Tambah Barang</button>
            </form>
            <div id="tambah-result"></div>
        </div>
        
        <div id="cari" class="section" style="display:none;">
            <h2>Cari Barang</h2>
            <div class="form-group">
                <label>Cari berdasarkan:</label>
                <select id="search-type">
                    <option value="kode">Kode Barang</option>
                    <option value="nama">Nama Barang</option>
                </select>
                <input type="text" id="search-query" placeholder="Masukkan kode atau nama...">
                <button onclick="cariBarang()">Cari</button>
            </div>
            <div id="cari-result"></div>
        </div>
        
        <div id="kelola" class="section" style="display:none;">
            <h2>Kelola Stok Barang</h2>
            <div class="form-group">
                <label>Kode Barang:</label>
                <input type="text" id="kelola-kode" required>
            </div>
            <div class="form-group">
                <label>Aksi:</label>
                <select id="kelola-aksi">
                    <option value="tambah">Tambah Stok</option>
                    <option value="kurangi">Kurangi Stok</option>
                </select>
            </div>
            <div class="form-group">
                <label>Jumlah:</label>
                <input type="number" id="kelola-jumlah" value="1" min="1" required>
            </div>
            <button onclick="kelolaStok()">Proses</button>
            <div id="kelola-result"></div>
        </div>
        
        <div id="lihat" class="section" style="display:none;">
            <h2>Daftar Semua Barang</h2>
            <button onclick="muatSemuaBarang()">Muat Ulang</button>
            <div id="lihat-result"></div>
        </div>
        
        <div id="statistik" class="section" style="display:none;">
            <h2>Statistik Inventori</h2>
            <div id="statistik-result"></div>
            <h3>Barang Stok Kritis (≤ 5)</h3>
            <div id="barang-kritis"></div>
        </div>
    </div>
    
    <script>
        let currentSection = 'home';
        
        function showSection(sectionId) {
            document.getElementById(currentSection).style.display = 'none';
            document.getElementById(sectionId).style.display = 'block';
            currentSection = sectionId;
            
            if (sectionId === 'statistik') {
                loadStatistik();
            } else if (sectionId === 'lihat') {
                muatSemuaBarang();
            }
        }
        
        async function tambahBarang(event) {
            event.preventDefault();
            const kode = document.getElementById('kode').value;
            const nama = document.getElementById('nama').value;
            const stok = document.getElementById('stok').value;
            const kategori = document.getElementById('kategori').value;
            const harga = document.getElementById('harga').value;
            
            const response = await fetch(`/api/add?kode=${kode}&nama=${nama}&stok=${stok}&kategori=${kategori}&harga=${harga}`);
            const result = await response.json();
            
            const resultDiv = document.getElementById('tambah-result');
            if (result.success) {
                resultDiv.innerHTML = `<div class="result success">${result.message}</div>`;
                document.getElementById('kode').value = '';
                document.getElementById('nama').value = '';
                document.getElementById('stok').value = '0';
                document.getElementById('harga').value = '0';
            } else {
                resultDiv.innerHTML = `<div class="result error">${result.message}</div>`;
            }
        }
        
        async function cariBarang() {
            const type = document.getElementById('search-type').value;
            const query = document.getElementById('search-query').value;
            const resultDiv = document.getElementById('cari-result');
            
            if (!query) {
                resultDiv.innerHTML = '<div class="result error">Masukkan kata kunci pencarian</div>';
                return;
            }
            
            if (type === 'kode') {
                const response = await fetch(`/api/cari?kode=${query}`);
                const result = await response.json();
                
                if (result.success) {
                    const barang = result.data;
                    resultDiv.innerHTML = `
                        <div class="result success">
                            <h3>Hasil Pencarian:</h3>
                            <table>
                                <tr><th>Kode</th><td>${barang.kode}</td></tr>
                                <tr><th>Nama</th><td>${barang.nama}</td></tr>
                                <tr><th>Stok</th><td>${barang.stok}</td></tr>
                                <tr><th>Kategori</th><td>${barang.kategori}</td></tr>
                                <tr><th>Harga</th><td>Rp ${barang.harga}</td></tr>
                                <tr><th>Nilai Inventori</th><td>Rp ${barang.nilai_inventori}</td></tr>
                            </table>
                        </div>
                    `;
                } else {
                    resultDiv.innerHTML = `<div class="result error">${result.message}</div>`;
                }
            } else {
                const response = await fetch(`/api/cari_nama?nama=${query}`);
                const result = await response.json();
                
                if (result.success && result.data.length > 0) {
                    let tableHTML = '<table><tr><th>Kode</th><th>Nama</th><th>Stok</th><th>Harga</th></tr>';
                    result.data.forEach(barang => {
                        tableHTML += `<tr>
                            <td>${barang.kode}</td>
                            <td>${barang.nama}</td>
                            <td>${barang.stok}</td>
                            <td>Rp ${barang.harga}</td>
                        </tr>`;
                    });
                    tableHTML += '</table>';
                    resultDiv.innerHTML = `<div class="result success"><h3>Hasil Pencarian (${result.data.length} item):</h3>${tableHTML}</div>`;
                } else {
                    resultDiv.innerHTML = `<div class="result error">${result.message}</div>`;
                }
            }
        }
        
        async function kelolaStok() {
            const kode = document.getElementById('kelola-kode').value;
            const aksi = document.getElementById('kelola-aksi').value;
            const jumlah = document.getElementById('kelola-jumlah').value;
            const resultDiv = document.getElementById('kelola-result');
            
            if (!kode || !jumlah) {
                resultDiv.innerHTML = '<div class="result error">Isi semua field</div>';
                return;
            }
            
            const endpoint = aksi === 'tambah' ? '/api/tambah_stok' : '/api/kurangi_stok';
            const response = await fetch(`${endpoint}?kode=${kode}&jumlah=${jumlah}`);
            const result = await response.json();
            
            if (result.success) {
                resultDiv.innerHTML = `<div class="result success">${result.message}</div>`;
                document.getElementById('kelola-kode').value = '';
                document.getElementById('kelola-jumlah').value = '1';
            } else {
                resultDiv.innerHTML = `<div class="result error">${result.message}</div>`;
            }
        }
        
        async function muatSemuaBarang() {
            const response = await fetch('/api/semua');
            const result = await response.json();
            const resultDiv = document.getElementById('lihat-result');
            
            if (result.success && result.data.length > 0) {
                let tableHTML = '<table><tr><th>Kode</th><th>Nama</th><th>Kategori</th><th>Stok</th><th>Harga</th><th>Nilai</th><th>Aksi</th></tr>';
                result.data.forEach(barang => {
                    tableHTML += `<tr>
                        <td>${barang.kode}</td>
                        <td>${barang.nama}</td>
                        <td>${barang.kategori}</td>
                        <td style="${barang.stok <= 5 ? 'color: red; font-weight: bold;' : ''}">${barang.stok}</td>
                        <td>Rp ${barang.harga}</td>
                        <td>Rp ${barang.nilai_inventori}</td>
                        <td><button onclick="hapusBarang('${barang.kode}')">Hapus</button></td>
                    </tr>`;
                });
                tableHTML += '</table>';
                resultDiv.innerHTML = `<div class="result success">${tableHTML}</div>`;
            } else {
                resultDiv.innerHTML = '<div class="result">Tidak ada barang dalam inventori</div>';
            }
        }
        
        async function hapusBarang(kode) {
            if (confirm(`Yakin ingin menghapus barang ${kode}?`)) {
                const response = await fetch(`/api/hapus?kode=${kode}`);
                const result = await response.json();
                if (result.success) {
                    alert(result.message);
                    muatSemuaBarang();
                } else {
                    alert(result.message);
                }
            }
        }
        
        async function loadStatistik() {
            const response = await fetch('/api/statistik');
            const result = await response.json();
            const statistikDiv = document.getElementById('statistik-result');
            
            if (result.success) {
                const stats = result.data;
                statistikDiv.innerHTML = `
                    <div class="stats">
                        <div class="stat-item">
                            <div class="stat-value">${stats.total_barang}</div>
                            <div>Jumlah Jenis Barang</div>
                        </div>
                        <div class="stat-item">
                            <div class="stat-value">${stats.total_stok}</div>
                            <div>Total Stok</div>
                        </div>
                        <div class="stat-item">
                            <div class="stat-value">Rp ${stats.total_nilai}</div>
                            <div>Nilai Inventori</div>
                        </div>
                        <div class="stat-item">
                            <div class="stat-value" style="color: ${stats.barang_kritis > 0 ? 'red' : 'green'}">${stats.barang_kritis}</div>
                            <div>Barang Kritis</div>
                        </div>
                    </div>
                `;
            }
            
            // Load barang kritis
            const responseKritis = await fetch('/api/barang_kritis');
            const resultKritis = await responseKritis.json();
            const kritisDiv = document.getElementById('barang-kritis');
            
            if (resultKritis.success && resultKritis.data.length > 0) {
                let tableHTML = '<table><tr><th>Kode</th><th>Nama</th><th>Stok</th><th>Kategori</th><th>Aksi</th></tr>';
                resultKritis.data.forEach(barang => {
                    tableHTML += `<tr>
                        <td>${barang.kode}</td>
                        <td>${barang.nama}</td>
                        <td style="color: red; font-weight: bold;">${barang.stok}</td>
                        <td>${barang.kategori}</td>
                        <td><button onclick="tambahStokKritis('${barang.kode}')">+ Stok</button></td>
                    </tr>`;
                });
                tableHTML += '</table>';
                kritisDiv.innerHTML = tableHTML;
            } else {
                kritisDiv.innerHTML = '<p>Tidak ada barang dengan stok kritis</p>';
            }
        }
        
        function tambahStokKritis(kode) {
            document.getElementById('kelola-kode').value = kode;
            document.getElementById('kelola-aksi').value = 'tambah';
            document.getElementById('kelola-jumlah').value = '10';
            showSection('kelola');
        }
        
        // Muat statistik awal
        loadStatistik();
    </script>
</body>
</html>
'''


@app.route('/')
def home():
    stats = inventori.statistik()
    return render_template_string(HTML_TEMPLATE, stats=stats)


# API Endpoints dengan response JSON yang konsisten
@app.route('/api/add')
def api_add():
    kode = request.args.get("kode")
    nama = request.args.get("nama")
    stok = int(request.args.get("stok", 0))
    kategori = request.args.get("kategori", "Umum")
    harga = int(request.args.get("harga", 0))
    
    success, message = inventori.tambah_barang(kode, nama, stok, kategori, harga)
    return jsonify({"success": success, "message": message})


@app.route('/api/cari')
def api_cari():
    kode = request.args.get("kode")
    barang = inventori.cari_barang(kode)
    if barang:
        return jsonify({"success": True, "data": barang.to_dict()})
    return jsonify({"success": False, "message": "Barang tidak ditemukan"})


@app.route('/api/cari_nama')
def api_cari_nama():
    nama = request.args.get("nama")
    if not nama:
        return jsonify({"success": False, "message": "Masukkan nama barang"})
    
    results = inventori.cari_by_nama(nama)
    return jsonify({"success": True, "data": results, "count": len(results)})


@app.route('/api/hapus')
def api_hapus():
    kode = request.args.get("kode")
    success, message = inventori.hapus_barang(kode)
    return jsonify({"success": success, "message": message})


@app.route('/api/tambah_stok')
def api_tambah_stok():
    kode = request.args.get("kode")
    jumlah = int(request.args.get("jumlah", 0))
    success, message = inventori.tambah_stok(kode, jumlah)
    return jsonify({"success": success, "message": message})


@app.route('/api/kurangi_stok')
def api_kurangi_stok():
    kode = request.args.get("kode")
    jumlah = int(request.args.get("jumlah", 0))
    success, message = inventori.kurangi_stok(kode, jumlah)
    return jsonify({"success": success, "message": message})


@app.route('/api/semua')
def api_semua():
    barang = inventori.semua_barang()
    return jsonify({"success": True, "data": barang, "count": len(barang)})


@app.route('/api/statistik')
def api_statistik():
    stats = inventori.statistik()
    return jsonify({"success": True, "data": stats})


@app.route('/api/barang_kritis')
def api_barang_kritis():
    batas = int(request.args.get("batas", 5))
    barang = inventori.barang_kritis(batas)
    return jsonify({"success": True, "data": barang, "count": len(barang)})


@app.route('/api/update')
def api_update():
    kode = request.args.get("kode")
    nama = request.args.get("nama")
    stok = request.args.get("stok")
    kategori = request.args.get("kategori")
    harga = request.args.get("harga")
    
    kwargs = {}
    if nama: kwargs['nama'] = nama
    if stok is not None: kwargs['stok'] = int(stok)
    if kategori: kwargs['kategori'] = kategori
    if harga is not None: kwargs['harga'] = int(harga)
    
    success, message = inventori.update_barang(kode, **kwargs)
    return jsonify({"success": success, "message": message})


# Tambahkan import os di bagian atas file
# (Tambahkan ini di baris paling atas, setelah from flask import ...)
import os

# ... kode Anda yang lain ...

# Di bagian paling bawah, ganti menjadi:
if __name__ == '__main__':
    port = int(os.environ.get('PORT', 5000))
    print(f"🚀 Server berjalan di port {port}")
    print("📂 Data disimpan di inventori.json")
    app.run(host='0.0.0.0', port=port, debug=False)
[event APD.py](https://github.com/user-attachments/files/24205061/event.APD.py)
web: python event A[requirements.txt](https://github.com/user-attachments/files/24205074/requirements.txt)
PD.py
Flask==2.3.3


[README.md](https://github.com/user-attachments/files/24205083/README.md)
# sistem-inventori2


