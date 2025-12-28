=== Подготовка ===
# Загрузившись в GParted Live отформатировать диск по такой схеме:
https://pixvid.org/images/2025/12/23/5bhvO.png

# Перезагрузка в Arch Live
# Применяем шрифт с поддержкой кириллицы.
setfont cyr-sun16

# Проверка доступности интернта.
ip link

# Проверка синхронизации часов.
timedatectl

# Сверка дисков
fdisk -l
# В случае неправильных Типов "Type" у разделов выполнить их правку через:
fdisk /dev/sda
https://pixvid.org/images/2025/12/23/5bqQF.png

# Создание и монтирование разделов
## Создание и монтирование зашифрованного корневого раздела.
cryptsetup -v luksFormat /dev/sda2
cryptsetup open /dev/sda2 root
## Создание файловой системе на разблокированном LUKS
mkfs.btrfs /dev/mapper/root
## Монтирование корневого тома в /mnt
mount /dev/mapper/root /mnt
## Проверка работает ли маппинг (?) как надо
umount /mnt
cryptsetup close root
cryptsetup open /dev/sda2 root
mount /dev/mapper/root /mnt

## Монитирование разделов учитывая, что у нас btrfs
## Так как корень смонтирован на пред. этапе, переход к созданию подтомов
## Cоздаю подтома
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@pkg
btrfs subvolume create /mnt/@swap
## Произвожу размонтирование с целью монтирования как подтомов.
umount /mnt

## Монтирование подтомов и создание папок
mount -o subvol=@,compress=zstd /dev/mapper/root /mnt

mount --mkdir -o subvol=@home,compress=zstd /dev/mapper/root /mnt/home
mount --mkdir -o subvol=@log,compress=zstd /dev/mapper/root /mnt/var/log
mount --mkdir -o subvol=@pkg,compress=zstd /dev/mapper/root /mnt/var/cache/pacman/pkg
mount --mkdir -o subvol=@swap,compress=zstd /dev/mapper/root /mnt/swap

## Добавление файла подкачки
btrfs filesystem mkswapfile --size 16g --uuid clear /mnt/swap/swapfile
## Активация файла подкачки
swapon /mnt/swap/swapfile
## Крайне важно не забыть добавить в список загрузки fstab

## Настройка раздела boot
mount --mkdir /dev/sda1 /mnt/boot

# Если я хочу отключить сжатие, я прописываю:
# btrfs property set [путь] compression no
# Если я хочу отключить CoW, я прописываю chattr +С
# Проверка осуществляется командой lsattr -d [путь]
# Есть отключён CoW отключено и сжатие.
# -R chattr применяет рекурсивно 
========================

# Обновим зеркала
reflector --verbose --download-timeout 4 -c France,Germany, --sort rate -a 6 -l 20 -p https --save /etc/pacman.d/mirrorlist

# Устанавливаем консольную расскладку и шрифт по умолчанию.
## Это делается заранее, так как mkinitcpio ругается на отсутствие этого файла.
mkdir /mnt/etc
cat > /mnt/etc/vconsole.conf << EOF
KEYMAP=us
FONT=cyr-sun16
EOF
# Устанавливаем основные пакеты до chroot
pkgs=(
    base linux-zen linux-firmware nano              # Базовые утилиты
    btrfs-progs dosfstools exfatprogs               # Утилиты для работы с ФС
    f2fs-tools e2fsprogs nilfs-utils                # Утилиты для работы с ФС
    linux-firmware-marvell linux-firmware-qcom      # Прошивки для устройств
)
pacstrap -K /mnt "${pkgs[@]}"

# Генерация списка разделов для автоматического монтирования при запуске системы
## Список сделанных монтирований | сверить с вручную написанными 
genfstab -U /mnt
## Определяю UUID в переменных
BTRFS_UUID=$(blkid -s UUID -o value /dev/mapper/root) && echo "btrfs: $BTRFS_UUID"
EFI_UUID=$(blkid -s UUID -o value /dev/sda1) && echo "EFI: $EFI_UUID"
## Вношу в файл ручную конфигурацию
cat > /mnt/etc/fstab << EOF
# Добавление конфигурации fstab
# <file system> <dir> <type> <options> <dump> <pass>

# Разделы BTRFS
# /dev/mapper/root
UUID=${BTRFS_UUID}  /   btrfs   rw,noatime,compress=zstd:3,discard=async,space_cache=v2,subvol=/@    0 0
# /dev/mapper/root
UUID=${BTRFS_UUID}  /home   btrfs   rw,noatime,compress=zstd:3,discard=async,space_cache=v2,subvol=/@home    0 0
# /dev/mapper/root
UUID=${BTRFS_UUID}  /var/log   btrfs    rw,noatime,compress=zstd:3,discard=async,space_cache=v2,subvol=/@log 0 0
# /dev/mapper/root
UUID=${BTRFS_UUID}  /var/cache/pacman/pkg   btrfs   rw,noatime,compress=zstd:3,discard=async,space_cache=v2,subvol=/@pkg 0 0
# /dev/mapper/root
UUID=${BTRFS_UUID}  /swap   btrfs   rw,noatime,compress=zstd:3,discard=async,space_cache=v2,subvol=/@swap    0 0

# Раздел boot
UUID=${EFI_UUID}  /boot   vfat    rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro    0 2
# SWAP
/swap/swapfile  none    swap    defaults    0 0
EOF

# Переход в chroot
arch-chroot /mnt

# Настройка времени.
## Создаём символическую ссылку с указанием региона и города для корректного часового пояса.
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
## Настройка службы systemd-timesyncd для сихронизации системных часов.
### Запускаю и добавляю в автозапуск службу.
systemctl enable systemd-timesyncd.service
### Тут возможно редактирование конфигурации /etc/systemd/timesyncd.conf но стоит оставить настройки по умолчанию.
## Синхронизируем аппаратные часы.
hwclock --systohc

# Настройка локализации
## В файле /etc/locale.gen расскомментировать нужные мне локализации.
sed -i 's/#\s*en_US\.UTF-8 UTF-8/en_US\.UTF-8 UTF-8/' /etc/locale.gen
sed -i 's/#\s*ru_RU\.UTF-8 UTF-8/ru_RU\.UTF-8 UTF-8/' /etc/locale.gen
grep -B 1 -A 1 'en_US\.UTF-8 UTF-8' /etc/locale.gen
grep -B 1 -A 1 'ru_RU\.UTF-8 UTF-8' /etc/locale.gen
## Генерируем локали
locale-gen
## Устанавливаем предпочтительный язык системы.
cat > /etc/locale.conf << EOF
LANG=ru_RU.UTF-8
EOF

# Добавление репозитория x86 пакетов multilib
## Требуется в файле /etc/pacman.conf расскоментировать строки:
sed -i '/^#\s*\[multilib\]/,/^#\s*Include/ s/^#\s*//' /etc/pacman.conf
grep -B 2 -A 2 '\[multilib\]' /etc/pacman.conf
## Обновляем репозитории.
pacman -Syu

# Имя ПК
cat > /etc/hostname << EOF
arch-pc1
EOF

# Добавление пароля root
passwd root

# Установка пакетов через pacman
## Базовые пакеьы
pkgs_BASE=(
    amd-ucode sudo                      # необходимо для нормальной работы
    base-devel git                      # Базовые утилиты
    man-db man-pages texinfo            # пакеты для работы с документацией
    7zip tar libarchive zip unzip unrar # пакеты для работы с архивами
    reflector                           # обновление зеркал для pacman
    snapper                             # утилита для рабоата с снимками btrfs
)
pacman -S --needed "${pkgs_BASE[@]}"

# Добавляение пользователя sudo
groupadd plugdev # требование solaar
groupadd informant # требование пакета AUR.
useradd -m -G wheel,plugdev,informant -s /bin/bash plasterr
sed -i '/^#\s*%wheel\s*ALL=(ALL:ALL)\s*ALL/ s/^#\s*//' /etc/sudoers
grep -B 1 -A 1 '%wheel\s*ALL=(ALL:ALL)\s*ALL' /etc/sudoers
passwd plasterr

# Продолжаю настройку btrfs
## Вход от лица пользователя
su - plasterr
## Создание subvolume и настройка их на отключение CoW и сжатия
btrfs subvolume create /home/plasterr/.cache
chattr +C /home/plasterr/.cache
lsattr -d /home/plasterr/.cache
btrfs subvolume create /home/plasterr/Загрузки
chattr +C /home/plasterr/Загрузки
lsattr -d /home/plasterr/Загрузки
btrfs subvolume create /home/plasterr/.virtualmachines_data
chattr +C /home/plasterr/.virtualmachines_data
lsattr -d /home/plasterr/.virtualmachines_data
btrfs subvolume create /home/plasterr/.games_data
chattr +C /home/plasterr/.games_data
lsattr -d /home/plasterr/.games_data
ls -la
## Добавление конфигураций snapper для получения снимков
sudo snapper --no-dbus -c home create-config /home
sudo snapper --no-dbus -c root create-config / 
exit

# Провекрка поддержки TRIM и его включение.
lsblk --discard
## Если в DISC-GRAN и DISC-MAX не нулевые значение, это означает поддержку TRIM.
## Запускаем TRIM | проверка службы | просмотр интервала
systemctl enable fstrim.timer
systemctl status fctrim.timer
cat /usr/lib/systemd/system/fstrim.timer

## Драйвера GPU + их x86 версии
pacman -S --needed \
    mesa lib32-mesa \
    vulkan-radeon lib32-vulkan-radeon

## KDE Plasma и основные пакеты
pkgs_KDE=(
    plasma-meta kde-system-meta                     # базовое ПО KDE
    phonon-qt6-vlc                                  # бэкэнд VLC был в руквдстве KDE
    dolphin baloo dolphin-plugins kio-admin         # dolphine и дополнения
    kio-gdrive kompare                              # dolphine и дополнения
    purpose                                         # добавляет в dolphine share
    ffmpegthumbs icoutils kdegraphics-thumbnailers  # различное медиа и миниатюры
    kdesdk-thumbnailers kimageformats libheif       # различное медиа и миниатюры
    libappimage qt6-imageformats resvg taglib       # различное медиа и миниатюры
    ffmpeg gst-libav gst-plugins-bad                # кодеры и декодеры медиа
    gst-plugins-base gst-plugins-good               # кодеры и декодеры медиа
    gst-plugins-ugly                                # кодеры и декодеры медиа
    kdeconnect                                      # kdeconnect

    arch-install-scripts                            # скрипты arch LiveCD напр genfstab
    sddm sddm-kcm qt6-declarative                   # всё для экрана блокировки.
    noto-fonts noto-fonts-emoji ttf-roboto          # шрифты
    fcitx5-im                                       # метод ввода
    snapper btrfs-assistant                         # создание снимков btrfs
    flatpak                                         # просто flatpak
    gparted                                         # управление дисками
    unp                                             # универсальный разархиватор
    flameshot                                       # утилита создания скриншотов
)
pacman -S --needed "${pkgs_KDE[@]}"
### Запуск службы sddm
systemctl enable sddm.service

## Пакеты предложенные в archinstall
pkgs_ARCHINSTALL=(
    smartmontools                                   # проверка здоровья накопителей
    wget wireless_tools wpa_supplicant xdg-utils    # предлогались archinstall | noto
    btop vim openssh                                # предлогались archinstall | htop был заменён на btop
)
pacman -S --needed "${pkgs_ARCHINSTALL[@]}"

## Выборка из kde-multimedia-meta
pkgs_KDE_multimedia_meta=(
    kmix                                            # микшер звука
    kdenlive                                        # видео редактор
    kwave                                           # аудио редактор
)
pacman -S --needed "${pkgs_KDE_multimedia_meta[@]}"

## Выборка из kde-graphics-meta
pkgs_KDE_graphics_meta=(
    arianna                      # электронная читалка книг
    colord-kde                   # что-то связанное с colord !!!
    gwenview                     # просмотр фото
    kcolorchooser                # палитра цветов
    kolourpaint                  # paint
    kruler                       # экранная линейка
    okular                       # просмотр pdf и прочее
    svgpart                      # отображение svg
)
pacman -S --needed "${pkgs_KDE_graphics_meta[@]}"

## Выборка из kde-utilities-meta
pkgs_KDE_utilities_meta=(
    ark                         # архиватор
    isoimagewriter              # запись iso
    kalk                        # калькулятор
    kate                        # текстовый редактор
    kbackup                     # содание бэкапов
    kcharselect                 # таблица символов
    kclock                      # часы
    kdebugsettings              # debug настройки
    kdf                         # мониторинг дисков
    kdialog                     # вызов диалоговых окон
    kfind                       # поиск файлов
    kgpg                        # что-то связано с подписями
    konsole                     # терминал KDE
    ktimer                      # таймер с запуском команд
    markdownpart                # компонент поддержки markdown
    sweeper                     # очистка временныз файлов
    yakuake                     # выпадающий терминал
)
pacman -S --needed "${pkgs_KDE_utilities_meta[@]}"

# Из группы kde-network
pkgs_KDE_network=(
    kdenetwork-filesharing      # Позволяет шерить папку через samba
    krdc                        # Подключение по rpd + vnc
    krfb                        # Позволяет мне шерить свой ПК через vnc
    kio-zeroconf                # насколько понимаю что-то делает сам с сетью
)
pacman -S --needed "${pkgs_KDE_network[@]}"

# Установка LibreOffice
pkgs_OFFICE=(
    libreoffice-fresh libreoffice-fresh-ru 
    # Пакеты шрифтов указанные в Wiki. Один находится в AUR!
    ttf-caladea ttf-carlito ttf-dejavu ttf-liberation
    ttf-linux-libertine-g noto-fonts
    adobe-source-code-pro-fonts adobe-source-sans-fonts
    adobe-source-serif-fonts
)
pacman -S --needed "${pkgs_OFFICE[@]}"

# Настройка сети
## Установка пакетов
pkgs_NETWORK=(
    networkmanager          # необходимо для работы сети
    iwd                     # управляет подключением wi-fi вместо wpa_supplicant
    firewalld               # фаерволл
    systemd-resolvconf      # для работы некоторых VPN | взято из wiki
)
pacman -S --needed "${pkgs_NETWORK[@]}"
## Указываю явно использовать iwd в конфиге NM
cat > /etc/NetworkManager/conf.d/wifi_backend.conf << EOF
[device]
wifi.backend=iwd
EOF
## Запустить службу зависимость для Networkmanager
systemctl enable systemd-resolved.service # Отвечает за DNS и тд.
## Создание символической ссылки на stub systemd-resolved, который подхватит Networkmanager | СОЗДАНИЕ с chroot
ln -sf ../run/systemd/resolve/stub-resolv.conf /mnt/etc/resolv.conf
## Запуск службы Networkmanager | требует ЗАПУСКА systemd-resolved 
systemctl enable NetworkManager.service 
## Настройка DNS
### Явно заставляю использовать systemd-resolved
cat > /etc/NetworkManager/conf.d/dns.conf << EOF
[main]
dns=systemd-resolved
EOF
### Создаю конфиг по которому будет опредяться DNS systemd-resolved
mkdir -p /etc/systemd/resolved.conf.d/
cat > /etc/systemd/resolved.conf.d/dns_main.conf << EOF
[Resolve]
DNS=9.9.9.9#dns.quad9.net
Domains=~.
FallbackDNS=
DNSOverTLS=yes
EOF
## Запуск службы firewalld
systemctl enable firewalld.service
## Настройка Bluetooth
### Установка пакетов и запуск службы
pacman -S --needed bluez bluez-utils
systemctl enable bluetooth.service
## Заметка: в cat /etc/resolv.conf указан ip на котором принимает запросы systemd-resolved
## Сам systemd-resolved можно настроить на использование adguardhome

# Настройка PipeWire
## Установка основного пакета и доп пакетов указанных в wiki
pacman -S --needed pipewire lib32-pipewire
    wireplumber \
    pipewire-audio \
    pipewire-alsa \
    pipewire-pulse \
    pipewire-jack lib32-pipewire-jack
## Проверка запуска сокета
systemctl --user status pipewire-pulse.socket
## Просто интересный сайт для настройки наушников
https://autoeq.app/
## Установка easyeffects для настройки в системе и его плагинов
pacman -S easyeffects \
    calf lsp-plugins-lv2 mda.lv2 yelp zam-plugins-lv2
## Уже в системе из AUR будет скачан пакет шумодав noisetorch

# Редактирование /etc/mkinitcpio.conf | добавление хуков для работы LUSK
nano /etc/mkinitcpio.conf
## Добавление модулей для работы клавиатуры с USB HUB
MODULES=(usbhid xhci_hcd)
## Редактирование хуков
HOOKS=(base systemd keyboard autodetect microcode modconf kms keymap sd-vconsole block sd-encrypt filesystems fsck)
### keyboard стоит до autodetect, для работы всех клавиатур даже после загрузки.
### sd-vconsole стоит перед block, а sd-encrypt после
## Пересборка образа initramfs
mkinitcpio -P

# Установка загрузчика
## В корне от chroot выполеить команду
bootctl install
## Настройка загрузчика
cat > /boot/loader/loader.conf << EOF
default  arch.conf
timeout  5
console-mode max
editor   no
EOF
## Добавление записи
### Определение переменных
LUKS_UUID=$(blkid -s UUID -o value /dev/sda2) && echo "LUKS: $LUKS_UUID"
### Запись
cat > /boot/loader/entries/arch.conf << EOF
title   Arch Linux
linux   /vmlinuz-linux-zen
initrd  /amd-ucode.img
initrd  /initramfs-linux-zen.img
options rd.luks.name=${LUKS_UUID}=root root=/dev/mapper/root rd.luks.options=discard
EOF

# Установка остальных пакетов
pkgs_oth=(
    pacman-contrib      # набор утилит pacman | pactree -rs
    steam               # steam
    obsidian            # Заметки
    qbittorrent         # торрент-клиент
    solaar              # софт для управлением устройствами Logitech
    code                # редактор кода
    baobab              # анализ использования диска
    gimp gimp-help-hu   # gimp
    vlc vlc-plugins-all # VLC и плагины
)
pacman -S --needed ${pkgs_oth[@]}

# Включение многоядерной сборки Make
## Узнать точнее количество потоков
nproc
## Установить полученное количество в конфиг
sudo nano /etc/makepkg.conf
- #MAKEFLAGS="-j2"
+ MAKEFLAGS="-j12"

# Настройка Reflactor на автообновление зеркал
## Настройка параметров запуска службы. В конфиг файле закомментировать сток значения
cat > /etc/xdg/reflector/reflector.conf << EOF
-c France,Germany,
--sort rate
-a 6
-l 20
-p https
--save /etc/pacman.d/mirrorlist
EOF
systemctl start reflector.service
systemctl enable reflector.timer
systemctl status reflector.timer

# Пакеты Paru
## Установка самого Paru
git clone https://aur.archlinux.org/paru.git && cd paru && makepkg -si && cd .. && rm -rf paru
## Пакеты Paru
### Пакетов много, ставлю их все по отдельности
paru -S informant                   # не позволяет обновиться при новостях
paru -S pamac-aur                   # управление пакетами
paru -S ffmpeg-full                 # Полный пакет ffmpeg | это рекомендовалось в Arch Wiki в ffmpeg
paru -S kde-thumbnailer-apk         # Для отображения ярлыков apk файлов
paru -S raw-thumbnailer             # Для отображения raw файлов | Взято из Wiki Dolphine
paru -S noisetorch                  # Шумоподавление микрофона
paru -S ayugram-desktop             # AyuGram
paru -S vmware-keymaps vmware-workstation # vmware-keymaps cтавится как зависимость к vmware
paru -S librewolf-bin               # Браузер
paru -S protonplus                  # конфигурация для запуска игр
paru -S portproton                  # настройка и запуск игр
paru -S protontricks                # Взято и образа Bazzite
paru -S proton-ge-custom-bin        # Кастомная версия proton
paru -S ttf-gentium-basic           # Шрифт указанный в wiki libreoffice
paru -S gdown                       # Прямая загрузка по ссылок Google Drive
paru -S betterdiscord-installer     # Установщик betterdiscord
paru -S obs-studio-git              # OBS Studio
paru -S ab-download-manager-bin     # Альтернатива ADM | требует расширения
paru -S wondershaper-git            # Ограничитель пропускной способности сети
paru -S nordvpn-bin                 # VPN
paru -S peazip                      # Архиватор
paru -S ttf-meslo-nerd-font-powerlevel10k # Шрифты для темы zsh
paru -S syncthingtray-qt6           # синхронизация
paru -S fsearch                     # местный аналог everything
paru -S xnviewmp                    # просмотр фото

## Установка шрифтов Windows 11
paru -G ttf-ms-win11 && cd ttf-ms-win11
gdown --fuzzy "https://drive.google.com/file/d/15EFnB7dgbaoaT9C6liTnjNJyWpEPlrTA/view?usp=drive_link"
7z -pdeutschistleicht x enqf89qtyjrm.7z && rm -rf enqf89qtyjrm.7z
makepkg -si --skipinteg
cd .. && rm -rf ttf-ms-win11

## Помощник по btrfs 
paru -S btrfsmaintenance-git
### Включаем его как сервис
sudo systemctl enable btrfsmaintenance-refresh

# Пакеты треюующие особой настройки после установки
pacman -S adguardhome               # DNS фильтрация
paru -S portmaster-bin              # Portmaster
paru -S nordvpn-bin                 # VPN

# Установка пакетов Flatpak
flatpak install flathub io.github.flattool.Warehouse # управление flatpak пакетами
flatpak install flathub com.github.tchx84.Flatseal # разрешения flatpak пакетов
flatpak install flathub io.github.kolunmi.Bazaar # менеджер пакетов
flatpak install flathub org.kde.CrowTranslate # переводчик
flatpak install flathub net.ankiweb.Anki # Anki
flatpak install flathub com.bitwarden.desktop # Bitwarden
flatpak install flathub io.github.mezoahmedii.Picker # Random





# Проверка работы trim
genfstab -U # проверяем наличие discard=async в монтировании
cat /sys/fs/btrfs/YOUR_UUID/discard # при отсутсвии в монтировании проверка в sysfs
dmsetup table root # проверка LUKS
sudo fstrim -v / # тест на практике
