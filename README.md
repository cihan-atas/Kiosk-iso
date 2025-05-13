# Debian Tabanlı Otomatik VLC Kiosk ISO Oluşturma Kılavuzu

Bu kılavuz, Ubuntu bir sistem üzerinde `live-build` kullanarak Debian tabanlı, özel bir ISO dosyası oluşturmayı adım adım açıklar. Oluşturulacak ISO'nun özellikleri:

*   Minimal Debian sistemi (örn: Bookworm veya Bullseye).
*   `buluthan` adında bir kullanıcı.
*   Sistem başlangıcında `buluthan` kullanıcısı otomatik olarak ve şifresiz giriş yapacak.
*   Giriş yaptıktan sonra VLC, tam ekran, arayüzsüz, OSD'siz ve döngü modunda önceden belirlenmiş bir videoyu otomatik olarak oynatmaya başlayacak.
*   Fare imleci gizlenecek.

## 1. Ön Gereksinimler (Ubuntu Sisteminize Kurulacaklar)

Bu işlemleri yapacağınız Ubuntu sisteminde aşağıdaki paketlerin kurulu olması gerekir:

```bash
sudo apt update
sudo apt install -y live-build debootstrap squashfs-tools xorriso git cifs-utils
```

*   `live-build`: Debian Live imajları oluşturmak için ana araç.
*   `debootstrap`: Temel bir Debian sistemi kurmak için kullanılır.
*   `squashfs-tools`: Live imajın sıkıştırılmış dosya sistemini oluşturmak için.
*   `xorriso`: ISO imajını oluşturmak için.
*   `git`: (Opsiyonel) Bu kılavuzu veya örnek yapılandırmaları klonlamak için.
*   `cifs-utils`: (Opsiyonel) Video dosyasını bir ağ paylaşımından almak için gerekebilir, genellikle doğrudan ISO'ya ekleyeceğiz.

## 2. Proje Dizini Oluşturma

Çalışma alanımızı oluşturalım:

```bash
mkdir ~/debian-vlc-kiosk
cd ~/debian-vlc-kiosk
```

## 3. `live-build` Yapılandırması

`live-build`'i yapılandırmak için `lb config` komutunu kullanacağız.

```bash
lb config noauto \
    --architecture amd64 \
    --distribution bookworm \
    --archive-areas "main contrib non-free non-free-firmware" \
    --debian-installer false \
    --iso-application "Debian VLC Kiosk" \
    --iso-publisher "Buluthan Custom Build" \
    --bootappend-live "boot=live components username=buluthan autologin quiet splash" \
    --memtest none \
    --win32-loader false
```

Açıklamalar:
*   `noauto`: Yapılandırma seçeneklerini manuel olarak belirteceğimizi söyler.
*   `--architecture amd64`: 64-bit sistem için. 32-bit için `i386` kullanabilirsiniz.
*   `--distribution bookworm`: Kullanılacak Debian sürümü (şu anki kararlı sürüm). `bullseye` (eski kararlı) de olabilir.
*   `--archive-areas "main contrib non-free non-free-firmware"`: VLC codec'leri ve olası kapalı kaynak donanım sürücüleri için `non-free` ve `non-free-firmware` depolarını ekler.
*   `--debian-installer false`: ISO'ya Debian yükleyicisini dahil etmeyerek boyutu küçültür.
*   `--iso-application`, `--iso-publisher`: ISO hakkında meta veriler.
*   `--bootappend-live "..."`: Önyükleme parametreleri. `username=buluthan autologin` burada önemli, ancak systemd ile daha sağlam bir otomatik giriş de yapılandıracağız. `quiet splash` önyükleme mesajlarını gizler.
*   `--memtest none`: Memtest'i kaldırır.
*   `--win32-loader false`: Windows için önyükleyiciyi kaldırır.

## 4. Paket Listesini Tanımlama

Minimal bir sistem ve VLC için gerekli paketleri içeren bir liste oluşturacağız.

`config/package-lists/mykiosk.list.chroot` adında bir dosya oluşturun ve içine şunları yazın:

```
# Minimal Xorg ve yardımcıları
xserver-xorg-core
xinit
x11-xserver-utils
xserver-xorg-video-fbdev # Genel fallback video sürücüsü
# İsteğe bağlı: donanımınıza özel sürücüler:
# xserver-xorg-video-intel
# xserver-xorg-video-amdgpu
# xserver-xorg-video-nouveau

# VLC Media Player
vlc

# Fare imlecini gizlemek için
unclutter

# Otomatik giriş ve temel sistem için
systemd-sysv
# firmware-linux-nonfree # Gerekirse bazı donanımlar için
# network-manager # Kablolu/kablosuz ağ için gerekirse, kiosk için genellikle gereksiz
```

**Not:** `xserver-xorg-video-fbdev` çoğu durumda çalışır. Eğer belirli bir grafik kartınız varsa (Intel, AMD, Nvidia-açık kaynak) ilgili paketi (`xserver-xorg-video-intel`, `xserver-xorg-video-amdgpu`, `xserver-xorg-video-nouveau`) ekleyebilirsiniz.

## 5. Kullanıcı Oluşturma ve Otomatik Giriş Yapılandırması (Hook)

`buluthan` kullanıcısını oluşturacak, şifresini ayarlayacak (veya şifresiz bırakacak) ve otomatik giriş için systemd servisini yapılandıracak bir hook script'i oluşturalım.

`config/hooks/live/0100-kiosk-user.hook.chroot` adında bir dosya oluşturun:

```bash
#!/bin_sh
set -e

# "buluthan" kullanıcısını oluştur, ev dizini ile, bash kabuğu ile
# ve video, audio, input gruplarına ekle
useradd -m -s /bin/bash -G video,audio,input,sudo buluthan

# İsteğe bağlı: buluthan kullanıcısına bir şifre ata (örn: "kiosk")
# Güvenlik için bu şifreyi değiştirin veya otomatik giriş sonrası kilitleyin.
# Şifresiz bırakmak için bu satırları yorumlayın veya /etc/shadow dosyasını düzenleyin.
echo "buluthan:kiosk" | chpasswd

# TTY1'de "buluthan" kullanıcısı için otomatik giriş
mkdir -p /etc/systemd/system/getty@tty1.service.d
cat << EOF > /etc/systemd/system/getty@tty1.service.d/override.conf
[Service]
ExecStart=
ExecStart=-/sbin/agetty --autologin buluthan --noclear %I \$TERM
Type=idle
EOF

# .bash_profile dosyasını oluşturarak X'i otomatik başlat
cat << EOF > /home/buluthan/.bash_profile
if [ -z "\$DISPLAY" ] && [ "\$(tty)" = "/dev/tty1" ]; then
  # X'in zaten çalışıp çalışmadığını kontrol et (örn. bir çökme sonrası yeniden giriş)
  if ! pgrep -x "Xorg" > /dev/null; then
    exec startx
  fi
fi
EOF
chown buluthan:buluthan /home/buluthan/.bash_profile
chmod 644 /home/buluthan/.bash_profile

# .xinitrc dosyasını oluşturarak VLC'yi kiosk modunda başlat
# Oynatılacak video dosyasının yolu ISO içinde /opt/kiosk_video/video.mp4 olacak
VIDEO_PATH="/opt/kiosk_video/video.mp4"

cat << EOF > /home/buluthan/.xinitrc
#!/bin/sh

# Ekran koruyucuyu ve güç yönetimini devre dışı bırak
xset s noblank
xset s off
xset -dpms

# Fare imlecini 0.5 saniye sonra gizle
unclutter -idle 0.5 -root &

# VLC'yi tam ekran, dekorasyonsuz, OSD'siz ve döngüde başlat
# Eğer VLC bir şekilde kapanırsa, startx da kapanır ve kullanıcı konsola düşer.
# Daha sağlam bir çözüm için VLC'yi bir while döngüsü içine alabilirsiniz.
exec vlc --fullscreen --no-video-title-show --no-video-deco --no-osd --loop "\$VIDEO_PATH"
EOF
chown buluthan:buluthan /home/buluthan/.xinitrc
chmod +x /home/buluthan/.xinitrc

exit 0
```
Bu script'i çalıştırılabilir yapın:
```bash
chmod +x config/hooks/live/0100-kiosk-user.hook.chroot
```
**Önemli Notlar:**
*   `echo "buluthan:kiosk" | chpasswd` satırı `buluthan` kullanıcısına `kiosk` şifresini atar. Eğer tamamen şifresiz bir erişim istiyorsanız ve konsola düşme durumunda bile şifre sormasın diyorsanız, bu satırı kaldırabilir ve `/etc/shadow` dosyasını düzenleyerek `buluthan` kullanıcısının şifre alanını boşaltabilirsiniz (`passwd -d buluthan` komutu chroot içinde hook ile çalıştırılabilir). Ancak, otomatik giriş zaten TTY1 için ayarlandığından bu genellikle yeterlidir.
*   `VIDEO_PATH` değişkeni, ISO içine yerleştireceğimiz video dosyasının yolunu gösterir.

## 6. Video Dosyasını ISO'ya Ekleme

Oynatılacak video dosyasını (`video.mp4` olarak adlandıralım) ISO'nun içine yerleştireceğiz.

Proje ana dizininde (`~/debian-vlc-kiosk`) aşağıdaki dizin yapısını oluşturun:
`config/includes.chroot/opt/kiosk_video/`

```bash
mkdir -p config/includes.chroot/opt/kiosk_video
```

Şimdi, oynatmak istediğiniz `video.mp4` dosyasını bu `config/includes.chroot/opt/kiosk_video/` dizinine kopyalayın.
Örnek olarak boş bir dosya oluşturalım (siz gerçek videonuzla değiştirin):

```bash
# GERÇEK VİDEO DOSYANIZI BURAYA KOPYALAYIN VEYA TAŞIYIN
# cp /path/to/your/actual/video.mp4 config/includes.chroot/opt/kiosk_video/video.mp4

# Test için geçici bir dosya (yer tutucu):
touch config/includes.chroot/opt/kiosk_video/video.mp4
echo "Bu geçici bir video dosyasıdır. Lütfen gerçek videonuzla değiştirin." > config/includes.chroot/opt/kiosk_video/video.mp4
```

`live-build`, `config/includes.chroot/` altındaki her şeyi oluşturulacak sistemin kök dizinine (`/`) kopyalayacaktır. Yani `video.mp4` dosyası ISO içinde `/opt/kiosk_video/video.mp4` adresinde olacaktır.

## 7. ISO Dosyasını Oluşturma

Artık ISO dosyasını oluşturmaya hazırız. Proje ana dizininde (`~/debian-vlc-kiosk`) aşağıdaki komutu çalıştırın:

```bash
sudo lb build
```

Bu işlem biraz zaman alabilir, çünkü gerekli Debian paketleri indirilecek ve sistem oluşturulacaktır. İşlem tamamlandığında, `live-image-amd64.hybrid.iso` (veya benzeri bir isimde) ISO dosyası proje ana dizininizde oluşacaktır.

Eğer bir hata alırsanız veya yapılandırmayı değiştirip yeniden denemek isterseniz, önce temizlik yapmanız gerekebilir:
```bash
sudo lb clean --purge
```
Sonra `sudo lb build` komutunu tekrar çalıştırın.

## 8. Test Etme

Oluşan ISO dosyasını bir sanal makinede (VirtualBox, QEMU/KVM, VMware) veya USB belleğe yazarak gerçek bir bilgisayarda test edebilirsiniz.

**QEMU ile Hızlı Test:**
```bash
qemu-system-x86_64 -m 2G -boot d -cdrom live-image-amd64.hybrid.iso -vga virtio
```
Sanal makine başladığında, Debian önyükleme yapmalı, `buluthan` kullanıcısı otomatik olarak giriş yapmalı ve VLC tam ekran modunda `/opt/kiosk_video/video.mp4` dosyasını oynatmaya başlamalıdır.

## Ek Notlar ve Özelleştirmeler

*   **Video Değiştirme:** Farklı bir video oynatmak için `config/includes.chroot/opt/kiosk_video/video.mp4` dosyasını istediğiniz video ile değiştirip `sudo lb build` komutunu tekrar çalıştırmanız yeterlidir.
*   **Daha Fazla Paket:** Eğer başka araçlara veya kütüphanelere ihtiyacınız varsa, bunları `config/package-lists/mykiosk.list.chroot` dosyasına ekleyebilirsiniz.
*   **Donanım Yazılımı (Firmware):** Bazı kablosuz kartlar veya özel donanımlar için ek firmware paketleri gerekebilir. `firmware-linux-nonfree` veya daha spesifik `firmware-*` paketlerini (örn: `firmware-iwlwifi`) paket listenize ekleyebilirsiniz.
*   **VLC Seçenekleri:** `.xinitrc` içindeki `vlc` komutunun seçeneklerini (örn: `--no-interact` ile etkileşimi tamamen kapatmak) ihtiyacınıza göre değiştirebilirsiniz.
*   **Sorun Giderme:**
    *   Eğer X başlamazsa veya VLC açılmazsa, `Ctrl+Alt+F2` ile başka bir sanal konsola geçip `buluthan` kullanıcısı (şifre: `kiosk` veya ayarladığınız şifre) ile giriş yapın. `/home/buluthan/.xsession-errors` (eğer oluşmuşsa) veya `journalctl -xe` komutlarıyla hata günlüklerini kontrol edin.
    *   `live-build` sırasında hata alırsanız, `build.log` dosyasını inceleyin.

Bu adımlar, istenen özelliklere sahip minimal bir Debian VLC kiosk ISO'su oluşturmanıza yardımcı olacaktır.
