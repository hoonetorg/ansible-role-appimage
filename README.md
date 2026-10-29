# ansible-role-appimage

Desktop apps as AppImages on openSUSE Leap (x86_64), once per host for all users, always the version the vendor
publishes as latest, checked against the vendor's checksum (and signature where the vendor signs):

| app | file | checked with | menu entry |
|---|---|---|---|
| `signal` | `/opt/appimage/Signal.AppImage` | sha512 from `latest-linux.yml` + GPG signature (`.gpg`) of Signal's key, fingerprint `4B16 B723 2DFA A439 AD79 1002 EF9F 501F 13EE D94C` (key in `files/`) | yes |
| `kdrive` | `/opt/ksuite/kDrive.AppImage` | `<url>.sha256` | no |
| `kchat` | `/opt/ksuite/kChat.AppImage` | sha512 from `latest-linux.yml` | no |
| `kmeet` | `/opt/ksuite/kMeet.AppImage` | sha512 from `latest-linux.yml` | no |

The Infomaniak apps (kDrive, kChat, kMeet) are installed exactly as ansible-role-ksuite did it (same URLs, checks
and files in `/opt/ksuite`), so a host switched from ksuite to this role keeps its files unchanged. Details on
those apps (several users, per-user first start, kDrive + SELinux, Sentry): see ansible-role-ksuite's README.

## How

Per app listed in `apps`:

1. find the current version (Infomaniak: release JSON; Signal: `https://updates.signal.org/desktop/latest-linux.yml`)
2. download only when the installed file differs (= update):
   - Infomaniak: `get_url` straight to the target with the vendor checksum
   - Signal: download to a temporary file next to the target (sha512 checked), download its `.gpg` signature,
     `gpgv --keyring <role key>` must report a valid signature of the pinned key fingerprint, then the file is moved
     into place (atomic). Bad download or signature: the run fails ("NOT installed"), the installed file stays
     unchanged, temporary files are removed. Check mode only reports the update, nothing is downloaded.
3. menu entry (Signal): `/usr/local/share/applications/appimage-signal.desktop` (`StartupWMClass=signal`, link
   handlers `sgnl://`, `signalcaptcha://`), icon `/usr/local/share/appimage/signal.png` taken from the verified
   AppImage with `unsquashfs` (the AppImage is not run) after every update or when it is missing; owner/mode of
   icon and launcher are checked on every run.

Apps not in `apps` are removed again (AppImage, menu entry, icon). The user data in the homes stays - the run shows
a cleanup note (as "changed", visible at the default verbosity) and saves it on the controller next to the ansible
log: `<directory of log_path>/cleanup/<host>/appimage-<id>-<YYYYmmdd-HHMMSS>.txt` (nsbldr:
`/srv/ansible/logs/cleanup/…`).

Packages: `fuse` (fusermount for the AppImage runtime), `squashfs` (unsquashfs, only with a menu entry app).

## Signal

- per user: start "Signal" from the menu, link it with the phone (QR code: Signal on the phone → Settings → Linked
  devices). Data: `~/.config/Signal` (message history of this computer, keys).
- sandbox: Signal's start script adds `--no-sandbox` to the whole app only if unprivileged user namespaces don't
  work (`unshare -Ur true`); on Leap they do, so the main process, zygotes and GPU process are sandboxed. One process
  still shows `--no-sandbox`: the renderer of Signal's main window - Signal creates that window with
  `webPreferences: {sandbox: false}` (in its `app.asar`; all other Signal windows use `sandbox: true`), the same in
  every package format (deb, Flatpak, AppImage); node integration stays off there. Check (expected: 1, a renderer):
  `ps -eo args | grep -i '[s]ignal' | grep -- '--no-sandbox' | grep -o -- '--type=[a-z-]*'` → `--type=renderer`.
  More processes with `--no-sandbox` (main process, zygotes) = the start script's fallback triggered.
- updates come with the next ansible run (the AppImage is root-owned, so Signal's own updater should not be able to
  replace it - what Signal shows then: still to be checked on the VM).
  Signal Desktop builds expire after about 90 days - run the role at least monthly.
- menu entry: a new entry/icon shows up in GNOME right away; a changed entry (start command, icon) only after the
  next logout/login - GNOME keeps the old one until then (`gtk-launch appimage-signal` starts the current file).

## Variables

```yaml
appimage:
  apps: [kdrive, kchat, kmeet, signal]
```

All defaults: `defaults/main.yml`; the app catalog (URLs, keys, menu entries, cleanup notes): `vars/main.yml`.

## Check

```bash
ls -l /opt/ksuite/ /opt/appimage/
curl -s https://updates.signal.org/desktop/latest-linux.yml | head -3     # current Signal version
desktop-file-validate /usr/local/share/applications/appimage-signal.desktop
```

## License

Apache-2.0

Created with the help of AI
