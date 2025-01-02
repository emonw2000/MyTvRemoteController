# TvRemoteController
> 一Sebuah aplikasi remote control HP yang meniru fungsi remote Wukong. HP menginstal aplikasi klien, sedangkan TV menginstal aplikasi server. Dalam jaringan lokal yang sama, HP dapat digunakan untuk mengontrol TV.

1. app: Aplikasi klien (diinstal di HP).
2. server: Aplikasi server (diinstal di TV).


## 1. Antarmuka Remote yang Dapat Disesuaikan
Meniru antarmuka operasi remote Wukong, dengan tombol yang dapat disesuaikan seperti atas, bawah, kiri, kanan, dan tombol OK. Tampilan dibuat dengan area berbentuk kipas untuk klik presisi dan mendukung penekanan lama.

## 2. Prinsip Kerja
Menggunakan DatagramSocket untuk membuat komunikasi antara klien dan server.

Server:
Server memulai layanan latar depan, termasuk dua thread untuk menerima dan mengirim data.
Dalam thread penerima (Runnable), DatagramSocket dibuat untuk menerima data pada port tertentu (misalnya: 5555).
Data yang diterima diproses untuk memverifikasi apakah berasal dari klien, dan jika ya, server akan mengirimkan alamat IP-nya ke klien.
Kode contoh penerima di server:

kotlin
Copy code
class ReceiverRunnable : Runnable {

    companion object {
        const val BROADCAST_PORT = 5555
    }

    @Volatile
    var isFlag = true

    override fun run() {
        val receiverBuffer = ByteArray(DATA_PACKET_SIZE)
        val datagramPacket = DatagramPacket(receiverBuffer, receiverBuffer.size)
        try {
            if (mReceiverSocket == null || mReceiverSocket?.isClosed == true) {
                mReceiverSocket = DatagramSocket(null).apply {
                    reuseAddress = true
                    bind(InetSocketAddress(BROADCAST_PORT))
                }
            }
            while (isFlag) {
                Log.i(TAG, "----------menunggu data----------")
                mReceiverSocket?.receive(datagramPacket)
                Log.e(TAG, "data diterima dari ------> ${datagramPacket.address.hostAddress}")
                mPool.submit(ParseRunnable(receiverBuffer, datagramPacket))
            }
        } catch (e: Exception) {
            e.printStackTrace()
        } finally {
            mReceiverSocket?.takeIf { !it.isClosed }?.close()
        }
    }
}
Aplikasi:
Aplikasi memasuki antarmuka pencarian, mengirim data secara broadcast ke semua IP di jaringan lokal melalui DatagramSocket pada port tertentu (misalnya: 5555).
Data yang diterima dari server diproses untuk menampilkan daftar perangkat IP yang dapat dihubungkan. Setelah pengguna memilih, koneksi dibuat dengan server menggunakan IP dan port tertentu.
Kode contoh pengiriman broadcast di aplikasi:

java
Copy code
String broadcastIp = "255.255.255.255";
if (initClientSocket == null || initClientSocket.isClosed()) {
    initClientSocket = new DatagramSocket(null);
    initClientSocket.setReuseAddress(true);
    initClientSocket.bind(new InetSocketAddress(BROADCAST_PORT));
}

byte[] buffer = getByteBuffer(NetConst.STTP_LOAD_TYPE_BROADCAST, 0, 0);
// Mengisi data broadcast
DatagramPacket datagramPacket = new DatagramPacket(buffer, packetLength,
        InetAddress.getByName(broadcastIp), BROADCAST_PORT);
while (isFlag) {
    initClientSocket.send(datagramPacket);
    Thread.sleep(2500);
    Log.i(TAG, "Mengirim broadcast ke IP");
}
Koneksi:
Setelah server dan aplikasi saling mengenali, aplikasi mengirim data tombol ke server untuk diproses.

## 3. Injeksi Tombol
Menggunakan Instrumentation untuk mengirim peristiwa tombol.

Kode contoh injeksi tombol:

kotlin
Copy code
private fun sendKeyDownUpSync(keyCode: Int) = runBlocking {
    launch {
        mInstrumentation.sendKeyDownUpSync(keyCode)
    }
}
Izin yang Diperlukan:
xml
Copy code
<uses-permission android:name="android.permission.INJECT_EVENTS" />
Catatan:
Penekanan lama diimplementasikan dengan mengirimkan ACTION_DOWN dari aplikasi ke server. Server kemudian memulai thread untuk terus mengirim peristiwa penekanan hingga menerima ACTION_UP.
Masalah bisa terjadi jika aplikasi gagal mengirimkan ACTION_UP (misalnya karena crash), sehingga server tidak menghentikan thread. Mekanisme timeout dapat digunakan untuk mengatasi masalah ini.

## 4. Metode Menjaga Proses Server Tetap Hidup
Layanan latar depan.
Broadcast untuk menghidupkan kembali.
JobService.
Sinkronisasi akun.


## 5. Tampilan
Antarmuka tombol navigasi dan pencarian perangkat
<img src="/snapshots/dpad.jpg" style="zoom: 30%"/>
<img src="/snapshots/search.jpg" style="zoom: 30%"/>
