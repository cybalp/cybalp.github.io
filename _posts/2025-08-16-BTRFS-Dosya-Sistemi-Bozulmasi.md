---
layout: post
title: "BTRFS Dosya Sistemi Bozulması"
date: 2025-08-16 13:00
categories: [Olay Analizleri, Rapor]
tags: [Sistem Kurtarma, BTRFS, Dosya Sistemleri]
---

<audio controls>
  <source src="/assets/music/Desperado.mp3" type="audio/mpeg">
  Tarayıcınız ses etiketini desteklemiyor
</audio>

## OLAY ÖZETİ

Son kullanıcının bilgisayarı, herhangi bir görünür sorun yokken kapatıldı. Saatler sonra tekrardan bilgisayar açıldığında disk şifresi girildikten sonra "emergency shell" (acil durum kabuğu) ekranında kaldı. Sistem, "root" dosya sistemini bağlayamadığını belirten kritik hatalar (`mount: ... fsconfig() failed`, `ERROR: Failed to mount 'UUID=...'`) gösterdi. 

Bu rapor, olayın adım adım analizini, kök nedenin belirlenmesini ve sorunun çözülmesi için uygulanan teknik adımları detaylandırmaktadır.

## İlk Gözlem ve Hipotez Geliştirme

Bilgisayarın önyükleme sırasında "emergency shell" ekranına düştüğü görüldü. Hata mesajı, sistemin UUID'si (`UUID=9d...`) ile belirtilen kök bölümünü bağlayamadığını gösteriyordu.

![hata mesajı](assets/img/post1/hatamesaji.png)

**İlk Hipotez**

Sorun, /etc/fstab dosyasındaki bir UUID (Universally Unique Identifier) yanlışlığıydı. Yanlış UUID nedeniyle sistem, kök bölümü bulamıyor ve önyükleme işlemi başarısız oluyordu. 

## Teşhis ve Adli Bilişim Adımları

Olayı analiz etmek ve sistemi kurtarmak için **Arch Linux Live** ortamı kullanıldı. Bu, şüpheli sistemden bağımsız olarak teşhis yapma imkanı sağladı. 

![live ortamı](assets/img/post1/liveortami.png)

**1. Adım: Disk Bölümlerinin Tespiti**

`fdisk -l` komutu kullanılarak disk listeleri incelendi. Ana disk (`/dev/nvme0n1`) iki bölümden oluşuyordu. 
- EFI sistemi olarak kullanılan bölüm -> `nvme0n1p1`
- Linux Dosya Sistemi -> `nvme0n1p2`

![fdisk -l komutu](assets/img/post1/fdisk -l command.png)

Ardından `blkid` komutu ile UUID taraması yapıldı. `/dev/nvme0n1p2` diskinin doğru UUID'sinin `645f...` olduğu tespit edildi. Bu da önyükleme hatasındaki `9d...` UUID'sinden farklıydı. Ayrıca diskin LUKS ile şifrelendiği de biliniyordu.

![blkid komutu](assets/img/post1/blkid command.png)

**2. Adım: Şifreli Bölümü Açma**

LUKS ile şifreli part'ı (`/dev/nvme0n1p2`), `cryptsetup` komutu ile önce açmamız, ardından diski onarmamız gerekiyordu. Bu sebeple aşağıdaki komutu kullanarak diski açtım.

`sudo cryptsetup open /dev/nvme0n1p2 cryptroot` 

--- Burada şifre açma işlemi esnasında bir isim vermem gerekiyordu. Bu ismi `cryptroot` olarak verdim. 

**2. Adım: Dosya Sistemini Bağlama ve fstab Dosyasını Düzenleme**

Önce kendi root bölümümü yani /dev/nvme0n1p2 diskimi Live sistemine bağlamam gerekiyordu. Bunun için bir dizin oluşturdum ve bu dizine diskimi bağladım.

`sudo mkdir /mnt/archiso`
`sudo mount /dev/mapper/cryptroot /mnt/archiso`

![mount komutu](assets/img/post1/mount.png)

Yukarıdaki hata ile karşılaştım.

`BTRFS: error (device dm-0) in btrfs_replay_log:2100: errno=-5 IO failure (Failed to recover log tree)`

Bu hata dosya sisteminin kendisinde bir bozulma olduğunu gösteriyor. Dosya sisteminin log ağacını kurtaramadığını ve bir I/O hatasıyla karşı karşıya olduğunu belirtiyor. I/O hatası olduğunu araştırmalarımdan sonra `errno=-5`'ten anlayabildim.

`BTRFS: error (device dm-0 state E): open_ctree failed: -5`

Bu hata ise yukarıdaki durumu doğrular nitelikte. Sistem de ciddi bir şekilde hata olduğunu bir hata ile bir daha vurguluyor.

**mount: /mnt/archiso: can't read superblock on /dev/mapper/cryptroot**

Bu hata, bir önceki hataların sonucudur. `superblock`, bir dosya sisteminin meta verilerini (boyutu, türü, konumu, vb.) depolayan en kritik yapıdır. Eğer `superblock` okunamıyorsa, dosya sistemi mount edilemez.

Sonuç olarak burada tek sorun UUID sorunu değil, hatta UUID sorunu 'muhtemelen' sistemde yaşanmış olan ani kapatmanın sonucu. Yani `fstab`'de bulunan hatayı düzeltsek bile alttaki dosya sisteminin bozuk olması, sistemin açılmasını engelleyecek.

**3. Adım: Diskin Onarılması**

Onarımı, disk henüz Live ortamımıza bağlı değilken yapmamız gerekiyor ki `sudo mount /dev/mapper/cryptroot /mnt/archiso` komutundan aldığımız hata ile **diskimizin henüz bağlı olmadığını** bilebiliriz.

`sudo btrfs check --repair /dev/mapper/cryptroot`

Yukarıdaki komut ile hasarlı dosya sistemini onarmaya çalıştım. Riskli bir durum olduğu ve veri kaybına sebep olacağı şeklinde bir uyarı aldım. Eğer diskimde düşündüğümün aksine donanımsal bir hata olsaydı, verilerimi kaybedebilirdim. Ancak 'kişisel tercihim' olarak bunu göze aldım ve devam ettim. 

![diskin onarılması](assets/img/post1/diskonarilmasi.png)

`sudo mount /dev/mapper/cryptroot /mnt/archiso` komutunu tekrar çalıştırdım ve herhangi bir hata ile karşılaşmadım. 

![Tekrar mount denemesi](assets/img/post1/mounttry.png)

`sudo nano /mnt/cryptroot/@/etc/fstab` komutunu kullanarak dosyayı düzenledim.  

![diskin onarilmasi 2](assets/img/post1/diskonarilmasi2.png)

Görmüş olduğumuz gibi diğer üç satır, sırasıyla `/home`, `/var/cache` ve `/var/log` gibi alt dizinleri işaret ediyor. Bunların hepsi aynı UUID'yi kullanıyor çünkü BTRFS dosya sistemi **alt bölümleri (subvolumes)** destekler ve bu alt bölümlerin hepsi aynı temel diskin üzerinde bulunur.

Ancak kök (root) bölümünde siyah daire içine alınmış olan UUID sorunun asıl kaynağı. Hata, sistemin ana kök bölümü bağlama hatası olduğu için ilgili `9d...` UUID'sini daha öncesinde `blkid` komutu ile bulduğumuz `645f...` UUID'si ile değiştiriyoruz. 

Daha sonra yeniden başlatmadan önce Live sisteminin, diskimizle olan tüm bağlantısını kesiyoruz. 

`sudo umount /dev/mapper/cryptroot`

Ardından...

`reboot` ile sistemi yeniden başlatıyoruz.

## KÖK NEDEN

Bilgisayarı genel olarak terminal açıp direk `shutdown` komutuyla kapatıp, `reboot` komutuyla yeniden başlatıyorum. Ancak bu problem başıma gelmeden önce 'shutdown' komutunu yazdıktan hemen sonra dizüstü bilgisayarımın kapağını kapattım. Sistem, kapanmadan önce diskteki açık dosyaları kapatır, önbellekteki verileri diske yazar ve log defterindeki tüm değişiklikleri ana dosya sistemine yazar. Bilgisayarın kapatıldığında anında kapanmamasının, biraz beklemesinin, hatta windowsta yuvarlak simgenin kısa da olsa dönmesinin ana nedeni de bu işlemlerdir.  Her anında kapattığımızda (mesela bilgisayarın güç tuşuna basılı tutarak) böyle bir sorunla karşılaşmıyoruz ancak bazen sistem ne yapacağını şaşırıyor. Tekrar açtığımızda da bize 'dur bakalım! Benim dosya sistemimi bozdun, bunu düzeltmeden eğer bu bilgisayarı açarsan, verilerin bütünlüğü bozulabilir! Veri kaybı yaşayabilirsin!' diyor..


## SONUÇ

Olayın temel nedeni, sistemin düzgün bir şekilde kapatılmaması sonucu oluşan dosya sistemi bozulmasıydı. Bu bozulma, BTRFS dosya sisteminin ana meta verilerini (`superblock`, `log tree`) okunamaz hale getirdi. Bu durum, sistemin `fstab` dosyasındaki doğru UUID'yi bile kullanamamasına neden oldu. Başarılı bir onarımdan sonra, daha önce gözden kaçan `fstab` dosyasındaki yanlış UUID sorunu da düzeltildi ve sistem normal işlevine döndü.

## Çıkarımlar ve Gelecek İçin Dersler

Tavsiyem; veri bütünlüğünüzü sağlamak için bilgisayarınızı bayıltmayın, siz ona uyumasını söyleyin, o zaten uyur. Kendinize ve bilgisayarınıza iyi bakın :)