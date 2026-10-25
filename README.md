name: Construire l'ISO DarckOS Linux

on:
  workflow_dispatch:

jobs:
  iso:
    runs-on: ubuntu-latest
    container:
      image: debian:trixie
      options: --privileged
    timeout-minutes: 120
    steps:
      - uses: actions/checkout@v4

      - name: Installer les outils
        run: |
          apt-get update
          apt-get install -y --no-install-recommends \
            unzip git make gcc libc6-dev ca-certificates curl procps \
            live-build debootstrap squashfs-tools xorriso cpio zip \
            grub-efi-amd64-bin grub-pc-bin grub-common mtools dosfstools \
            isolinux syslinux syslinux-common

      - name: Décompresser le projet
        run: unzip -q darckos-linux-1.0.0.zip -d projet

      - name: Compiler V (version fixée)
        run: |
          git clone https://github.com/vlang/v.git /opt/v
          git -C /opt/v checkout 5bd67093f97f3574b36bc1a9f19f56a9c0a4ad07
          make -C /opt/v

      - name: Compiler DarckOS Tools puis l'auto-tester
        working-directory: projet
        run: |
          make V=/opt/v/v
          make test V=/opt/v/v

      - name: Construire l'ISO
        working-directory: projet
        run: |
          live/prepare.sh build/darckos
          cd live
          lb config
          lb build

      - name: Empreinte SHA-256
        working-directory: projet/live
        run: sha256sum ./*.iso | tee darckos-linux.iso.sha256

      - uses: actions/upload-artifact@v4
        with:
          name: darckos-linux-iso
          path: |
            projet/live/*.iso
            projet/live/darckos-linux.iso.sha256
          retention-days: 14
          compression-level: 0
