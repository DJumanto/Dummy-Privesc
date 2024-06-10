# PRIVILEGE ESCALATION

## Topik Pembelajaran
- [What is Privilege Escalation](#what-is-privilege-escalation)
- [Enumeration](#enumeration)
- [Tools](#tools)
- [Sudo Privesc](#sudo-privesc)
- [SUID Privesc](#suid-privilege-escalation)
- [Library Hijack](#library-hijack)


## Mohon Dibaca Sebelum Lanjut
**&#9888;** **Peringatan:** Segala bentuk serangan _SQL Injection_ tanpa izin pada pemilik layanan/sistem dapat dikenakan **sanksi pidana yang serius**. Segala informasi yang dipaparkan di sini **hanya untuk tujuan pembelajaran** dan **tidak boleh** digunakan untuk aktivitas ilegal.

## What is Privilege Escalation
![privesc](https://tse4.mm.bing.net/th?id=OIP.glvQMCBjCOaEe7oD5UsOvAHaEK&pid=Api&P=0&h=180)

**_Privilege Escalation_** merupakan teknik untuk mendapatkan hak akses yang lebih tinggi supaya mendapatkan akses untuk melakukan tindakan yang tidak seharusnya bisa dilakukan oleh user yang memiliki hak akses yang lebih rendah, opo maksude? (apa maksudnya?). Anggap di suatu perusahaan terdapat empat buah role seperti berikut:

| **Role** | **Hak Akses** |
|----------|---------------------------------------------------------|
| HRD      | Menambah karyawan, memecat karyawan, menaikkan jabatan karyawan |
| Sekretaris | Membuat jadwal, melihat dokumen kegiatan, melihat dokumen classified perusahaan |
| Finance Officer | Melihat data keuangan, membuat keputusan pengeluaran, menolak pengeluaran
| Karyawan Biasa | Kerja sesuai permintaan supervisor |

Anggap posisi kalian adalah karyawan biasa kamu mempunyai rekan kerja yang menjengkelkan anggap saja namanya Pak Djumanto. Suatu hari kalian tidak sengaja melihat komputer HRD yang belum terkunci, lalu kalian memutuskan untuk membuka komputer tadi dan membuka aplikasi ERP (Enterprise Resource Planning) milik perusahaan. Ternyata disana sebagai HRD, kalian memiliki akses untuk melakukan PHK terhadap karyawan yang terdaftar sehingga ia menjadi pengangguran. Karena kalian ingat kalau kalian punya rekan yang menjengkelkan, akhirnya kalian memutuskan untuk menghapus Pak Djumanto dari perusahaan sehingga ia tidak memiliki pekerjaan di perusahaan tersebut.

![DJum](https://g.foolcdn.com/editorial/images/460806/confused-man_gettyimages-519833420.jpg)
*Djumanto yang bingung tiba-tiba ia kena PHK

Sederhana untuk dipahami bukan? dalam dunia penetration testing, teknik ini disebut dengan _Privilege Escalation_ atau peningkatan hak akses. Dalam linux ada sangat banyak teknik untuk melakukan serangan ini, namun akan kita rangkap menjadi 3 buah teknik, yaitu:
1. Sudo Privilege Escalation
2. SUID PRivilege Escalation
3. Library Hijack
4. Insecure Path Reference

Akan kita pelajari satu-satu dibawah, tetapi akan kita bahas terlebih dahulu terkait teknik enumerasi dan tools yang bisa digunakan:

## Enumeration
Untuk mengtahui teknik apa yang bisa dipakai untuk melakukan privilege escalation, kita perlu melakukan enumerasi selayaknya kita melakuakn recon sebelum mengeksploitasi. Berikut adalah beberapa cara yang bisa kalian lakukan untuk mendapatkan informasi yang sekiranya berguna dan bisa menjadi titik terang tentang cara apa yang bisa dilakukan untuk mendapatkan hak akses yang lebih tinggi:

| **Technique**                                | **Explanation**                                                                                                                                               |
|----------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Kernel Exploits**                          | Banyak teknik privilege escalation dengan memanfaatkan CVE dari kernel yang outdated untuk mengetahuinya kalian bisa menjalankan ``cat /proc/version`` atau ``uname -r``     |
| **SUID/GUID Executables**                    | Beberapa binary yang memiliki hak akses SUID, bisa dieksploitasi untuk mendapatkan hak ases yang lebih tinggi. Command: `find / -perm -u=s -type f 2>/dev/null`|
| **Misconfigured Cron Jobs**                  | Memanfaatkan miskonfigurasi cronjob, penaikan hak akses bisa dari cronjob yang writable, atau cronjob mengeksekusi command / binary milik user dengan hak akses rendah, menggunakan hak akses yang lebih tinggi. Command: `cat /etc/crontab`, `ls -la /etc/cron*`.                                                         |
| **Writable Files and Directories**           | Mencari informasi terkait file dan direktori yang writable dengan hak akses rendah. Command: `find / -writable -type d 2>/dev/null`, `find / -perm -o w -type f 2>/dev/null`.     |
| **Weak Permissions on Sensitive Files**      | Miskonfigurasi hak akses pada file sensitif seperti `/etc/shadow`, (/etc/shadow berisi user dan hash password). Command: `ls -l /etc/passwd`, `ls -l /etc/shadow`.                                                      |
| **PATH Variable Manipulation**               | Memanipulasi PATH environment variable untuk mengeksploitasi binary yang dieksekusi dengan hak akses tinggi tanpa menuliskan path spesifiknya. Command `echo $PATH`.|
| **SSH Keys**                                 | Mencari key SSH user lain untuk login pada path ``/home/{user}/.ssh/{nama_key} #biasanya id_rsa`` . Command: `find / -name id_rsa 2>/dev/null`.|
| **Sudo Misconfigurations**                   | Mengeksploitasi miskonfigurasi hak akses eksekusi menggunakan Sudo untuk menjalankan suatu binary / command dengan user lain. Command: `sudo -l`.|
| **Group Memberships**                        | Mengkesploitasi suatu binary untuk mendapatkan akses sebagai user lain yang satu grup apabila pada binary tersebut terdapat akses untuk menjalankan sebagai satu grup. Command: `id`, `groups`.|
| **Exploiting Installed Software**            | Mengkeploitasi software yang terinstall menggunakan CVE yang pernah ditemukan. Tools: `searchsploit`, `exploitdb`.|
| **Environment Variables**                    | Mengeksploitasi environment variable dengan melakukan poisining pada path dynamic library seperti `LD_PRELOAD` dan `LD_LIBRARY_PATH`. Example: `LD_PRELOAD=/path/to/malicious.so /bin/ls`.|
| **Log Files**                                | Mencari informasi dari log. Command: `cat /var/log/auth.log`, `dmesg`.                                                                      |

### Custom Scripts and Commands

```sh
# Check for SUID/SGID files
find / -perm /6000 -type f -exec ls -ld {} \; 2>/dev/null

# Check for writable directories and files
find / -writable -type d 2>/dev/null
find / -perm -o w -type f 2>/dev/null

# Check for cron jobs
cat /etc/crontab
ls -la /etc/cron*

# Check for sudo privileges
sudo -l

# Check for PATH manipulation
echo $PATH

# Check for environment variables exploitation
env | grep -i ld_

```

## Tools

Karena agak meng-capek apabila kita melakukan enumerasi secara manual, kita bisa menggunakan automated tools untuk mendapatkan informasi terkait kesuluruhan sistem yang memungkinkan untuk dieksploitasi supaya mendapatkan hak akses. Beirikut adalah tools yang bisa kalian gunakan:

```sh
# Automated enumeration
wget https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh
chmod +x LinEnum.sh
./LinEnum.sh

# Another automated tool
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

## Sudo Privesc
![sudo](https://i.redd.it/0ixosy6khwj61.jpg)

Sudo atau ``Super User Do``, merupakan command yang digunakan oleh user linux untuk menjalankan perintah / binary dengan user yang memiliki hak akses untuk menjalankan binary tersebut. Gimana maksudnya? Berikut merupakan struktur hak akses pada linux:

![perm](https://tse3.mm.bing.net/th?id=OIP.fcVcIgrf5F4uK8Fa_FQgdwAAAA&pid=Api&P=0&h=180)

Terdapat 4 segment pada linux, yakni filtype, user, group, dan others. Kita abaikan bagian pertama, bagian user group dan others menentukan siapa yang memiliki hak untuk melakukan apa pada file / direectory tersebut. bagian yang bertuliskan pada gambar (root root), menunjukkn bahwa pemiliknya adalah user root dan grup root. Apabila kamu ingin mengeksekusi, atau mengubah file tersebut, kamu perlu memiliki hak akses as super user, atau pada gambar tersebut sebagai root untuk melakukannya.

Lalu, bagaimana caranya mengetahui bahwa suatu user bisa mengeksekusi sesuatu sebagai sudo? ketikkan ``sudo -l`` untuk mendapatkan list binary yang bisa dieksekusi dengan sudo.
```sh
┌──(kali㉿LAPTOP-FPIIP2HU)-[~]
└─$ sudo -l
Matching Defaults entries for kali on LAPTOP-FPIIP2HU:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User kali may run the following commands on LAPTOP-FPIIP2HU:
    (ALL : ALL) ALL
```

Pada kasus ini, kita bisa melakukan command sudo terhadap apapun yang ada menggunakan prompt password untuk membaca / mengeksekusi / memodifikasi file-file atau direktori dengan hak akses yang berbeda dari user kita.

<img width="839" alt="image" src="https://github.com/DJumanto/JanjiTemu-App/assets/100863813/e8c0a7c7-569f-47a1-9cb9-cab39f3d58d5">

Pada kasus itu kita bisa melihat dengan user pico, kita bisa menjalankan sudo dengan password untuk menjalankan ``vi``. Gimana eksploitasinya?, kita jalankan terlebih dahulu vi nya sebagai user:
```sh
sudo vi
# Masukkan password pada prompt
```

<img width="865" alt="image" src="https://github.com/DJumanto/JanjiTemu-App/assets/100863813/48df7725-be86-4b98-83e4-fd754b97ff61">

Lalu ketikkan ``:`` dan ketikkan ``!/bin/bash`` untuk menjalankan shell bash.

<img width="250" alt="image" src="https://github.com/DJumanto/JanjiTemu-App/assets/100863813/082f5c07-0b42-4007-b175-3b9cdad3f8ae">

Loh kok bisa jadi root??. Ketika kalian menjalankan vi, kalian bisa melakukan eksekusi shell command dengan mengenekan escape, lalu menjalankan ``:`` untuk memasukkan prompt, dan mengetikkan ``!/bin/bash`` untuk mengeksekusi command ``/bin/bash``. karena tadi kita menjalankan command sebagai root, maka kita juga akan menjalankan shell nya sebagai root.

Teknik lain adalah menggunakan known CVE pada Sudo itu sendiri. Kalian bisa mencarinya di exploit-db untuk melihat berbagai macam varian CVE pada sudo yang pernah ada sebelumnya.

Informasi terkait teknik eksploitasi Sudo untuk mendapatkan hak akses, bisa kalian baca di [GTFOBINS](https://gtfobins.github.io/#)

## SUID Privilege Escalation

Cara lain adalah melakukan eksploitasi terhadap binary yang memiliki hak akses SUID untuk mendapatkan hak akses yang lebih tinggi. Apalagi itu SUID, oke mari kita belajar kembali terkait struktur dari linux permission.

<img width="527" alt="image" src="https://github.com/DJumanto/JanjiTemu-App/assets/100863813/ce08864f-c432-4772-83f5-76f67f79c0e0">

Jadi selain permission ``rwx``, linux memiliki karakter spesial yang bertindak sebagai akses dengan sifat khusus bagi yang memiliki tanda tersebut, yakni karakter ``s``. Karakter ``s`` pada permission bagain user / pemilik, memberikan informasi bahwa siapapun yang melakukan tertentu pada file tersebut, maka akan mengeksekusinya sebagai pemiliknnya. Sebagai contoh pada binary ``/usr/bin/passwd``, dengan permission ``-rwsr-x-r-x``, ketika kita mengeksekusinya, maka kita akan menjalankannya sebagai root.

Untuk mengetahui informasi terkait apa saja binary yang memiliki akses SUID, kita bisa menjalankan command berikut:

```sh
find / -perm -u=s -type f 2>/dev/null
```

Untuk informasi lebih lengkap terkait eksploitasi binary menggunakan suid, kalian bisa melihat referensinya di [GTFOBINS](https://gtfobins.github.io/#)


## Library Hijack
Selanjutnya kita masuk ke library hijack, library hijack merupakan teknik dimana kita bisa melakukan hijack pada library yang digunakan pada sebuah binary atau script bisa dieksekusi dengan hak akses yang lebih tinggi. Berikut adalah contoh library hijacking pada python, dengan kasus script python bisa dieksekusi dengan hak akses sebagai root:

<img width="548" alt="image" src="https://github.com/DJumanto/JanjiTemu-App/assets/100863813/2ac34c88-2a8e-4338-8348-770dd81f1796">

Terlihat bahwa kita bisa melakukan eksekusi script python yang terletak pada home kita sebagai root. Kita bisa saja memodifikasi filenya lalu memanggil shell seperti ``os.system('/bin/bash)`` untuk mendapatkan shell, akan tetapi:
<img width="427" alt="image" src="https://github.com/DJumanto/JanjiTemu-App/assets/100863813/78051df3-e90f-4abd-ba23-965272a59a2d">

Kita tidak bisa melakukannya karena tidak mendapat akses write ke filenya? Lalu bagaimana caranya? kita bisa melakukan library hijacking bila kita memiliki akses untuk menulis ke library yang ada. Pertama kita perlu mengetahui letak dimana library nya berada dan mengecek hak akses nya.

```sh
# List semua letak library yang ada untuk python3
python3 -m site

# Cari directory yang berisi librarynya
```

<img width="379" alt="image" src="https://github.com/DJumanto/JanjiTemu-App/assets/100863813/e8abe5a0-89bb-42b1-b368-07fa44ab358b">

Pada kasus ini, library ``base64.py`` yang terletak di ``/usr/lib/python3.8/base64.py`` bisa kita modifikasi sebagai root, karena kita memiliki passwordnya, kita bisa menambahkan kode untuk mengimport ``os`` dan mengeksekusi ``os.system("/bin/bash")`` untuk mendapatkan akses root. berikut adalah contohnya:

<img width="533" alt="image" src="https://github.com/DJumanto/JanjiTemu-App/assets/100863813/fa9999b6-b7a3-49bb-9725-9a3f96ef0d1e">

* Menambahkan library os pada library base64

<img width="575" alt="image" src="https://github.com/DJumanto/JanjiTemu-App/assets/100863813/042ec3f6-cd83-4042-a920-8314b80b8104">

* Menambahkan eksekusi ``os.system("/bin/bash")`` pada proses encoding string.

Setelah itu file kita simpan dan eksekusi sebagai root

Hasilnya adalah, ketika mereka akan mengeksekusi ``host_info = base64.b64encode(host_info_to_str.encode('ascii'))``. Kita akan mendapatkan shell sebagai root akibat library hijacking.

<img width="835" alt="image" src="https://github.com/DJumanto/JanjiTemu-App/assets/100863813/eb41f217-57be-416e-9108-4a64949d3e07">