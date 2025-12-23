# Immich

## Hardware setup

Enable wake-on-lan (see dev-notes).

Configure the laptop to not suspend if the lid is closed:

```sh
sudo mkdir -p /etc/systemd/logind.conf.d
sudo tee /etc/systemd/logind.conf.d/lid-close-action.conf > /dev/null <<EOF
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
EOF
```

## Installation and configuration

User: _reg e-mail address_, password _N3_.

1. Configure storage template
1. Disable face recognition (under 'Machine Learning Settings' -> 'Facial Recognition')
1. Enable face import (under 'Metadata Settings')
1. Mount external libraries as _read-only_ volumes

## Downloading and fixing Google Photos

1. Unzip archive
    ```sh
    cd "$HOME/Pictures/Foto's/Nieuw"
    unzip takeout-20250718T190838Z-1-001.zip
    ```
1. If you want, replace `_` with `'` in album directory names
1. **Check for missing albums shared with me but not created by me**
1. **Check for missing photos in shared albums I created to which other people added**
1. Clone `google-photos-migrate`
    ```sh
    cd ~/docker-compose/immich
    git clone https://github.com/garzj/google-photos-migrate
    ```
1. In `src/config/langs.ts', add the following entry:
    ```ts
      nl: {
        editedSuffix: 'bewerkt',
        photosDir: 'Google Foto_s',
        untitledDir: 'Naamloos',
      },
    ```
1. <del>In `Dockerfile`, change the base image to `docker.io/node:24-bookworm` (in contrast to what the README says, Node 18 doesn't work).</del>
1. Build the Docker image
    ```sh
    docker build -f Dockerfile -t localhost/google-photos-migrate:latest .
    ```
1. Run the full migration:
    ```sh
    takeout_dir="$HOME/Pictures/Foto's/Nieuw/2025-Takeout-test/"
    mkdir "${takeout_dir}/output" "${takeout_dir}/error"
    docker run --rm -it --security-opt=label=disable \
        -v "$(readlink -e "${takeout_dir}"):/takeout" \
        -v "$(readlink -e "${takeout_dir}/output"):/output" \
        -v "$(readlink -e "${takeout_dir}/error"):/error" \
        localhost/google-photos-migrate:latest \
        full '/takeout' '/output' '/error'
    ```
1. Check for files in `/errors`, fix and rerun. Possible causes:
    * Multiple original files with the same name. Solution: rename the image, the metadata JSON file name _and_ the file name within the metadata JSON file.
1. Check for files in subdirectories of `/output` named 'duplicate-*'. Usually, these are the originals of edited photos and can be deleted.
1. Change the time zone of all photos from UTC to the local time zone of wherever they were taken (usually Europe/Amsterdam):
    ```sh
    (cd timezone-adjuster && uv run shift_exif_timezone.py Europe/Amsterdam "$takeout_dir/output" "$takeout_dir/output-tz-adjusted")
    ```
1. Check if `$takeout_dir/output` and `$takeout_dir/output-tz-adjusted` have the same files (2x find + diff)
1. Create albums in Immich (see below)

TODO HIER BEZIG:

Old photos:
1. Find all files w/o EXIF timestamps --> set timestamp from Digikam and/or from file name
1. Move wedding photos to `$HOME/Pictures/Foto's/albums` after checking/fixing the EXIF timestamps


## Creating albums for external libraries

https://github.com/Salvoxia/immich-folder-album-creator

```sh
docker run \
  -e API_URL="http://scheleaapub.fritz.box:2283/api/" \
  -e API_KEY="wOHY7ReXTt6nMHYd9qqkLbsmGpRwx7WNZuFgdvIL9Jc" \
  -e ROOT_PATH="/media/D/Afbeeldingen/Foto's/albums" \
  salvoxia/immich-folder-album-creator:latest \
  /script/immich_auto_album.sh
```

/usr/src/app/external


## Importing albums using the CLI

```sh
sudo docker run -it -v $HOME/temp-linux/immich/example-folders-to-upload:/import:ro -e IMMICH_INSTANCE_URL=http://scheleaapub.fritz.box:2283/api -e IMMICH_API_KEY=3Mm6TLlRxTwIE5IVRSHD73u2SHyawc1GnoxlVf5dA ghcr.io/immich-app/immich-cli:latest upload -r -a /import
```
