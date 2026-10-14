# Mixxx Unofficial (snap)

Unofficial snap of [Mixxx](https://mixxx.org), the free open-source DJ software.
It repackages the official Ubuntu 24.04 build from the Mixxx stable PPA
([ppa:mixxx/mixxx](https://launchpad.net/~mixxx/+archive/ubuntu/mixxx)) for
amd64 and arm64. Not affiliated with or endorsed by the Mixxx Development Team.

    sudo snap install mixxx-unofficial
    sudo snap connect mixxx-unofficial:alsa      # MIDI controllers, direct ALSA output
    sudo snap connect mixxx-unofficial:jack1     # optional, JACK
    sudo snap connect mixxx-unofficial:raw-usb   # optional, USB bulk controllers
    sudo snap connect mixxx-unofficial:removable-media  # music on /media, /mnt

## Automatic updates

`.github/workflows/build-and-publish.yml` runs every 12 hours:

1. `check-upstream` reads the newest `mixxx` source published for noble in the
   PPA (Launchpad API) and compares it with the version in the `stable` channel
   of `mixxx-unofficial`.
2. If they differ, `build-and-publish` builds the snap on native amd64 and arm64
   runners, installs it, runs an `ldd` sweep inside the snap runtime plus
   `mixxx --version`, and uploads to `edge`.
3. `promote-to-stable` promotes `edge` to `stable`.

The version is taken from the PPA package at build time (`adopt-info`), so new
releases need no commit here.

Required repository secret: `SNAPCRAFT_STORE_CREDENTIALS`
(`snapcraft export-login --snaps=mixxx-unofficial --channels=edge,stable creds.txt`).

## Limitations

This is a repackaged PPA build running under strict confinement. Compared with
a native install (PPA, Flatpak, distro package), expect the following:

- **HID controllers.** Controllers that Mixxx drives through `/dev/hidraw*`
  (devices whose Mixxx mapping is a HID mapping rather than MIDI) are
  not reachable: no snapd interface grants generic hidraw access. MIDI
  controllers work once `alsa` is connected; USB bulk controllers need
  `raw-usb`.
- **No udev rules.** The upstream `69-mixxx-usb-uaccess.rules` file cannot be
  installed from a snap, so device permissions come only from the host system.
- **Manual interfaces.** `alsa`, `jack1`, `raw-usb` and `removable-media` are not connected
  automatically; without `alsa` there are no MIDI devices and no direct `hw:`
  output.
- **Audio latency.** PortAudio in the PPA build has no PulseAudio host API.
  The ALSA `default` device is routed to PulseAudio/PipeWire
  (`other-files/asound.conf`), which works but adds latency. For live use pick
  a `hw:` device (needs `alsa`) or JACK (needs `jack1`; with PipeWire the host
  must provide `pipewire-jack`). `pasuspender` from the upstream launcher is
  not used.
- **Realtime scheduling.** The snap has no rtkit/`sys_nice` access, so it cannot
  raise its audio thread priority; buffer underruns at very low latency are more likely
  than with a native install.
- **File access.** Music is readable from `$HOME` (non-hidden paths) and from
  `/media` and `/mnt` only after `snap connect mixxx-unofficial:removable-media`.
  Hidden folders in `$HOME` and other system paths are not visible.
- **Settings location.** The Mixxx library and settings live in
  `~/snap/mixxx-unofficial/current/.mixxx`, not `~/.mixxx`. Copy an existing
  `~/.mixxx` there to migrate (absolute track paths still have to be reachable
  from inside the snap).
- **Wayland.** The UI runs through XWayland (`QT_QPA_PLATFORM=xcb`,
  `DISABLE_WAYLAND=1`); `qt6-wayland` is not shipped. This is deliberate for
  now: Mixxx still has open native-Wayland bugs (waveform shader crashes on
  `wayland-egl`, QML skin crashes, broken drag/scroll, HiDPI issues on KDE),
  see [mixxxdj/mixxx Wayland issues](https://github.com/mixxxdj/mixxx/issues?q=is%3Aissue+is%3Aopen+wayland+in%3Atitle).
  XWayland works on every Wayland desktop that ships it (GNOME, KDE).
  **TODO:** once those are fixed upstream, stage `qt6-wayland`, drop
  `DISABLE_WAYLAND`/`QT_QPA_PLATFORM` and switch to native Wayland.
- **Version lag.** Only the stable PPA is tracked (no 2.6 beta / nightly
  builds), and a new release reaches the snap within about 12 hours after it is
  published in the PPA for Ubuntu 24.04.
- **Unofficial.** Bugs specific to this packaging belong in this repository's
  issue tracker, not upstream.

## License

Packaging files in this repository: GPL-3.0 (see `LICENSE`).
Mixxx itself: GPL-2.0-or-later, Copyright (C) 2001-2026 Mixxx Development Team.
