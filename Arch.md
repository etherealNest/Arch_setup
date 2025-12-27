=== Начало установки ===
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

_---МОНТИРОВАНИЕ---_
# Монитирование разделов учитывая, что у нас btrfs
## Монтирую / .
mount /dev/sda2 /mnt
## Cоздаю подтома .
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@pkg
## Произвожу размонтирование с целью монтирования как подтомов.
umount /dev/sda2

## Монтирование подтомов и создание папок
mount -o subvol=@,compress=zstd /dev/sda2 /mnt

mkdir -p /mnt/home && mkdir -p /mnt/var/log && mkdir -p /mnt/var/cache/pacman/pkg

mount -o subvol=@home,compress=zstd /dev/sda2 /mnt/home
mount -o subvol=@log,compress=zstd /dev/sda2 /mnt/var/log
mount -o subvol=@pkg,compress=zstd /dev/sda2 /mnt/var/cache/pacman/pkg

## Cоздаю папку boot и монтирую /boot.
mkdir -p /mnt/boot && mount /dev/sda1 /mnt/boot

## Активируем раздел SWAP
swapon /dev/sda3

# Если я хочу отключить сжатие, я прописываю:
# btrfs property set [путь] compression no
# Если я хочу отключить CoW, я прописываю chattr +С
# Проверка осуществляется командой lsattr -d [путь]
# Есть отключён CoW отключено и сжатие.
# -R chattr применяет рекурсивно 
---______________---
========================

=== Установка ===

# Обновим зеркала
reflector --verbose --download-timeout 4 -c France,Germany, --sort rate -a 6 -l 20 -p https --save /etc/pacman.d/mirrorlist
// Стоит не забыть создать после сервис в системе.

# Устанавливаем консольную расскладку и шрифт по умолчанию.
## Это делается заранее, так как ядро будет ругаться на отсутсвие этого файла.
mkdir /mnt/etc
nano /mnt/etc/vconsole.conf
+ KEYMAP=us
+ FONT=cyr-sun16
# Устанавливаем основные пакеты до chroot
pkgs=(
    base linux-zen linux-firmware nano              # Базовые утилиты
    btrfs-progs dosfstools exfatprogs               # Утилиты для работы с ФС
    f2fs-tools e2fsprogs nilfs-utils                # Утилиты для работы с ФС
    linux-firmware-marvell linux-firmware-qcom      # Прошивки для устройств
)
pacstrap -K /mnt "${pkgs[@]}"

# Генерация списка разделов для автоматического монтирования при запуске системы
genfstab -U /mnt >> /mnt/etc/fstab

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
## Проверяем
timedatectl status

# Настройка локализации
## В файле /etc/locale.gen расскомментировать нужные мне локализации.
nano /etc/locale.gen
!= "#" en_US.UTF-8 UTF-8
!= "#" ru_RU.UTF-8 UTF-8
## Генерируем локали
locale-gen
## Устанавливаем предпочтительный язык системы.
nano /etc/locale.conf
+ LANG=ru_RU.UTF-8

# Добавление репозитория x86 пакетов multilib
## Требуется в файле /etc/pacman.conf расскоментировать строки:
nano /etc/pacman.conf
!= "#" [multilib]
!= "#" Include = /etc/pacman.d/mirrorlist
## Обновляем репозитории.
pacman -Syu

# Имя ПК
nano /etc/hostname
+ [имя]

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
groupadd plugdev
groupadd informant # требование пакета AUR.
useradd -m -G wheel,plugdev,informant -s /bin/bash plasterr
passwd plasterr
nano /etc/sudoers
!= "#" %wheel      ALL=(ALL:ALL) ALL 

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
## Отключаем непрервный TRIM.
nano /etc/fstab
## Убираем из строчек discard
!="discard"

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

    sddm sddm-kcm qt6-declarative                   # всё для экрана блокировки.
    noto-fonts noto-fonts-emoji                     # шрифты
    noto-fonts-extra ttf-roboto                     # шрифты
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
)
pacman -S --needed "${pkgs_NETWORK[@]}"
## Указываю явно использовать iwd в конфиге NM
nano /etc/NetworkManager/conf.d/wifi_backend.conf
+ [device]
+ wifi.backend=iwd
## Запустить службу зависимость для Networkmanager
systemctl enable systemd-resolved.service # Отвечает за DNS и тд.
## Создание символической ссылки на stub systemd-resolved, который подхватит Networkmanager
ln -sf ../run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
// ссылку попарвить
// создавать ссылку в chroot не получиться, делать это нужно не находясь в chroot
// \\ ln -sf ../run/systemd/resolve/stub-resolv.conf /mnt/etc/resolv.conf
// \\  ссылка на замечание (https://wiki.archlinux.org/title/Systemd-resolved#DNS)
## Запуск службы Networkmanager
systemctl enable NetworkManager.service 
// ТРЕБУЕТ ПРЕДВАРИТЕЛЬНО ЗАПУСКА systemd-resolved 
// ТРЕБУЕТ plasma-nm 
// так же проверить ли запущено networkManager-wait-online.service после основного запуска
// убедиться что не запущено systemd-networkd.service (ссылка на замечание https://wiki.archlinux.org/title/Network_configuration#Network_management)
## Запуск службы firewalld
systemctl enable firewalld.service
# Настройка Bluetooth
## Установка пакетов и запуск службы
pacman -S --needed bluez bluez-utils
systemctl enable bluetooth.service

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

# Установка загрузчика
## В корне от chroot выполеить команду
bootctl install
## Настройка загрузчика
nano /boot/loader/loader.conf
+ default  arch.conf
+ timeout  5
+ console-mode max
+ editor   no
## Добавление записи
nano /boot/loader/entries/arch.conf
+ title   Arch Linux
+ linux   /vmlinuz-linux-zen
+ initrd  /amd-ucode.img
+ initrd  /initramfs-linux-zen.img
+ options root="LABEL=Arch" rw rootflags=subvol=@

# Установка остальных пакетов
pkgs_oth=(
    pacman-contrib      # набор утилит pacman | pactree -rs

    steam               # steam


    obsidian            # Заметки

    adguardhome         # DNS фильтрация
    qbittorrent         # торрент-клиент

    solaar              # софт для управлением устройствами Logitech
    code                # редактор кода

    baobab              # анализ использования диска
    vlc vlc-plugins-all # VLC и плагины
)
pacman -S --needed ${pkgs_oth[@]}

    protonplus          # конфигурация для запуска игр
    portproton          # настройка и запуск игр
    syncthingtray-qt6   # синхронизация


# Пакеты Paru
## Вход от имени пользователя
su - plasterr
## Установка самого Paru
git clone https://aur.archlinux.org/paru.git && cd paru && makepkg -si && cd .. && rm -rf paru
## Пакеты Paru
pkgs_PARU=(
    informant                   # не позволяет обновиться при новостях
    pamac-aur                   # управление пакетами
    ffmpeg-full                 # Полный пакет ffmpeg
                                # .это рекомендовалось в Arch Wiki в ffmpeg
    kde-thumbnailer-apk          # Для отображения ярлыков apk файлов
    raw-thumbnailer              # Для отображения raw файлов
                                 # .Взято из Wiki Dolphine
    noisetorch                  # Шумоподавление микрофона
    # ayugram-desktop            # AyuGram
    vmware-keymaps              # Ставится как зависимость к vmware
    vmware-workstation          # VM Ware
    librewolf-bin                # Браузер
    protontricks                # Взято и образа Bazzite                    
    proton-ge-custom-bin        # Кастомная версия proton
    ttf-gentium-basic           # Шрифт указанный в wiki libreoffice
    gdown                       # Прямая загрузка по ссылок Google Drive
    betterdiscord-installer     # Установщик betterdiscord
    obs-studio-git              # OBS Studio
    ab-download-manager-bin     # Альтернатива ADM | требует расширения
    wondershaper-git            # Ограничитель пропускной способности сети
    nordvpn-bin                 # VPN
    portmaster-bin              # Portmaster
    peazip                      # Архиватор
    ttf-meslo-nerd-font-powerlevel10k # Шрифты для темы zsh
    )
paru -S ${pkgs_PARU[@]}

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

# Установка пакетов Flatpak
flatpak install flathub io.github.flattool.Warehouse # управление flatpak пакетами
flatpak install flathub com.github.tchx84.Flatseal # разрешения flatpak пакетов
flatpak install flathub io.github.kolunmi.Bazaar # менеджер пакетов
flatpak install flathub org.kde.CrowTranslate # переводчик
flatpak install flathub net.ankiweb.Anki # Anki
flatpak install flathub com.bitwarden.desktop # Bitwarden
flatpak install flathub io.github.mezoahmedii.Picker # Random

# Настройка Reflactor на автообновление зеркал
## Настройка параметров запуска службы. В конфиг файле закомментировать сток значения
sudo nano /etc/xdg/reflector/reflector.conf
+ -c France,Germany,
+ --sort rate
+ -a 6
+ -l 20
+ -p https
+ --save /etc/pacman.d/mirrorlist
systemctl start reflector.service
systemctl enable reflector.timer
systemctl status reflector.timer
