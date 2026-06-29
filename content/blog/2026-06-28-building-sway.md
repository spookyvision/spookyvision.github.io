+++
title = "Building sway from source on Ubuntu derivates"
slug = "building-sway"
+++
## Not for the faint of heart

Sadly, the [official sway build instructions](https://github.com/swaywm/sway/wiki/Development-Setup) are a bit minimalistic and do not take into account that some system packages may be too old. I'm currently running Pop!OS (though once I find the time I'll replace that with Archbtw) and it was a bit of an adventure, as its most recent release is 24.04-LTS.

Even with other distros though, you will be in for a few surprises if you want to have a full-featured build with all optional items enabled…

Since I didn't expect things to get so involved I didn't keep a proper track record from the beginning, so I'm using my shell history and educated guesses. Expect things to be out of order and incomplete - [feedback](https://chaos.social/@dngrs/116828883787861727) welcome.

## Rough reconstruction of steps taken

### System packages that still can be used

```sh
sudo apt build-dep sway

# aaaand some additional packages required by the deps we are going to build.
# Can probably be trimmed, and some things may be missing that I already had installed, e.g. m4

sudo apt install bison libxcb-xkb-dev libxcb-util1 libxcb-util-dev xcb-proto lua5.4 liblua5.4-dev hwdata liblcms2-dev \
glslang-tools libcairo2-dev libcap-dev libdbus-1-dev libdisplay-info-dev libevdev-dev libgdk-pixbuf2.0-dev libjson-c-dev \
libpam0g-dev libpango1.0-dev libpcre2-dev libpixman-1-dev libseat-dev libsystemd-dev libvulkan-dev libwayland-egl1 libxcb-ewmh-dev scdoc
```

### Build dependencies from source

Most dependencies can be built with a simple `meson setup build/ --prefix=/usr/local`, followed by `ninja -C build/ install`. Neither require running as root, `ninja` will ask for privileges when installing. sweet. For `wayland` we're skipping docs, as this cuts down on further dependencies.

When possible, clones are done with `--depth=1` to speed things up (this only fetches `HEAD` and not the entire history).

Notable exceptions:
- `wlroots` requires an older version of `libliftoff` for some reason
- `libdisplay-info` has subprojects that must be downloaded
- `libxkbcommon` HEAD requires a more recent `meson` than the distro provides, and I really didn't feel like going down *that* path, so I randomly picked an older release
- Xorg `macros`: uses autoconf (argh)
- `libxcb-errors` uses autoconf (argh) and won't find dependent macros installed to `/usr/local`. Fortunately this is easily fixed by setting `ACLOCAL_PATH`. It also requires manual submodule (argh) init

```sh
# wayland
git clone --depth=1 https://gitlab.freedesktop.org/wayland/wayland.git
cd wayland
meson setup build/ --prefix=/usr/local -Ddocumentation=false
ninja -C build/ install
cd ..

# wayland-protocols
git clone --depth=1 https://gitlab.freedesktop.org/wayland/wayland-protocols.git
cd wayland-protocols
meson setup build/ --prefix=/usr/local
ninja -C build/ install

# drm
git clone --depth=1 https://gitlab.freedesktop.org/mesa/drm
cd drm
meson setup build/ --prefix=/usr/local
ninja -C build/ install
cd ..

# libxkbcommon
git clone --depth=1 https://github.com/xkbcommon/libxkbcommon.git
cd libxkbcommon
git checkout xkbcommon-1.11.0
meson setup build/ --prefix=/usr/local
ninja -C build/ install
cd ..

# Xorg macros
git clone https://gitlab.freedesktop.org/xorg/util/macros
cd macros
./autogen.sh
./configure --prefix=/usr/local
make
sudo make install
export ACLOCAL_PATH=/usr/local/share/aclocal
cd ..

# libxcb-errors
git clone --depth=1 https://gitlab.freedesktop.org/xorg/lib/libxcb-errors
cd libxcb-errors
git submodule update --init
./autogen.sh
./configure --prefix=/usr/local
make
sudo make install
cd ..

# pixman
git clone --depth=1 https://gitlab.freedesktop.org/pixman/pixman.git
cd pixman
meson setup build/ --prefix=/usr/local
ninja -C build/ install
cd ..

# libinput
git clone --depth=1 https://gitlab.freedesktop.org/libinput/libinput
meson setup build/ --prefix=/usr/local
ninja -C build/ install
cd ..

# libdisplay-info
git clone --depth=1 https://gitlab.freedesktop.org/emersion/libdisplay-info.git
cd libdisplay-info
meson subprojects download
meson setup build/ --prefix=/usr/local
ninja -C build/ install
cd ..

# libliftoff
git clone https://gitlab.freedesktop.org/emersion/libliftoff.git
cd libliftoff
git checkout v0.4
meson setup build/ --prefix=/usr/local
ninja -C build/ install
cd ..
```

### Build sway

```sh
git clone --depth=1 https://github.com/swaywm/sway.git
meson subprojects download
meson setup build/ --prefix=/usr/local
ninja -C build/ install
```


Finally. After `meson setup` you should hopefully see this:
```
wlroots 0.21.0-dev

    drm-backend      : YES
    x11-backend      : YES
    libinput-backend : YES
    xwayland         : YES
    gles2-renderer   : YES
    vulkan-renderer  : YES
    gbm-allocator    : YES
    udmabuf-allocator: YES
    session          : YES
    color-management : YES
    xcb-errors       : YES
    egl              : YES
    libliftoff       : YES

sway 1.13-dev

    gdk-pixbuf: YES
    tray      : YES
    man-pages : YES

  Subprojects
    wlroots   : YES
```

### Running it

```
$ /usr/local/bin/sway
/usr/local/bin/sway: error while loading shared libraries: libwlroots-0.21.so: cannot open shared object file: No such file or directory
```

Oh no, our shiny new dependency libraries cannot be found! Well then.

```sh
LD_LIBRARY_PATH=/usr/local/lib/x86_64-linux-gnu:/usr/local/lib /usr/local/bin/sway
```

Hurray, almost there!

### WHAT DO YOU MEAN, ALMOST THERE?!

Welp, one of the reasons to run a modern sway is that it supports screen sharing of application windows since version 0.12 (released in May 2026). But that's not enough - we also need a wlroots desktop portal that understands this.

### OKAY, BACK TO THE DRAWING BOARD

- remove the system package, it will get in the way
- build and install `xdg-desktop-por`- no wait, install its dependencies first 🙄
- add a systemd unit for our custom build

here's my `~/.config/systemd/user/xdg-desktop-portal-wlr.service`:

```systemd
[Unit]
Description=Portal service (wlroots implementation)
PartOf=graphical-session.target
After=graphical-session.target
ConditionEnvironment=WAYLAND_DISPLAY

[Service]
Type=dbus
BusName=org.freedesktop.impl.portal.desktop.wlr
Environment=LD_LIBRARY_PATH=/usr/local/lib/x86_64-linux-gnu:/usr/local/lib
ExecStart=/usr/local/libexec/xdg-desktop-portal-wlr
Restart=on-failure
```

once that's in place, install the portal:

```sh
systemctl --user stop xdg-desktop-portal-wlr.service
sudo apt remove xdg-desktop-portal-wlr

git clone --depth=1 https://github.com/benhoyt/inih.git
cd inih
meson setup build/ --prefix=/usr/local 
ninja -C build/ install
cd ..

git clone --depth=1 https://github.com/emersion/xdg-desktop-portal-wlr.git
cd xdg-desktop-portal-wlr
meson setup build/ --prefix=/usr/local 

systemctl daemon-reload
```

for completeness' sake: this portal needs to be explicitly set as preferred, like so (`~/.config/xdg-desktop-portal/sway-portals.conf`):

```ini
[preferred]
# use xdg-desktop-portal-gtk for every portal interface
default=gtk
# except for the xdg-desktop-portal-wlr supplied interfaces
org.freedesktop.impl.portal.Screencast=wlr
org.freedesktop.impl.portal.Screenshot=wlr
```