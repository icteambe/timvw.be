---
title: "Keep Your Bose SoundTouch Speaker Alive After the Shutdown"
date: 2026-02-17
draft: false
tags: ["bose", "soundtouch", "self-hosting", "reverse-engineering", "docker"]
categories: ["self-hosting"]
---

Bose is shutting down the SoundTouch cloud servers on [May 6, 2026](https://www.bose.co.uk/en_gb/landing_pages/soundtouch-eol.html). Once they do, your TuneIn presets stop working, the SoundTouch app loses most functionality, and you can no longer configure presets.

The good news: you can fix all of this. [SoundCork](https://github.com/timvw/soundcork) is a self-hosted replacement for the Bose cloud servers. It runs on anything — a Raspberry Pi, a NAS, a Docker host, a Kubernetes cluster. Your speaker talks to your server instead of Bose, and everything keeps working.

This post walks through the setup. It took me about an hour from start to finish with a SoundTouch 20.

## What Still Works Without Any Changes

These features are independent of the cloud servers:
- **Spotify Connect** — cast from the Spotify app to your speaker
- **Bluetooth** and **AUX** input
- **AirPlay** (on supported models)
- The speaker's **local API on port 8090** — [bose](https://github.com/timvw/bose) is a CLI tool I wrote that uses this for volume, presets, power control

## What Breaks (And What SoundCork Fixes)

- **TuneIn radio presets** — the BMX server that resolves station URLs is already returning 404s
- **Preset management** in the SoundTouch app — can no longer set TuneIn presets
- **Spotify presets** — fixable with a workaround (see below)

## The Setup

### 1. Get SSH Access to Your Speaker

SoundTouch speakers run Linux. On firmware 27.x, you need a USB stick trick to enable SSH:

1. Format a USB stick as FAT32
2. Create an empty file called `remote_services` (no extension)
3. **macOS users**: remove hidden junk files — `mdutil -i off /Volumes/USB && rm -rf /Volumes/USB/.fseventsd /Volumes/USB/.Spotlight-V100`
4. Power off the speaker, insert the USB, power on
5. Wait 60 seconds, then: `ssh root@<speaker-ip>` (no password)

Make it permanent: `touch /mnt/nv/remote_services`

During my setup, I also had to connect via Ethernet rather than WiFi. I changed both at the same time (cleaned the USB and switched to Ethernet), so I can't say for certain which was the fix.

Full details: [Speaker Setup Guide](https://github.com/timvw/soundcork/blob/main/docs/speaker-setup.md)

### 2. Extract Your Speaker Data

SoundCork needs four XML files from your speaker:

```bash
# From the speaker's local API (port 8090)
curl http://<speaker-ip>:8090/presets > Presets.xml
curl http://<speaker-ip>:8090/recents > Recents.xml
curl http://<speaker-ip>:8090/info > DeviceInfo.xml

# From SSH (contains auth tokens not exposed via the API)
ssh root@<speaker-ip> cat /mnt/nv/BoseApp-Persistence/1/Sources.xml > Sources.xml
```

Place them in SoundCork's data directory — see the [examples/](https://github.com/timvw/soundcork/tree/main/examples) directory for the expected format.

### 3. Deploy SoundCork

```bash
docker run -d --name soundcork \
  -p 8000:8000 \
  -v ./data:/soundcork/data \
  -e base_url=http://your-server:8000 \
  -e data_dir=/soundcork/data \
  ghcr.io/timvw/soundcork:main
```

The image is multi-arch (amd64 + arm64), so it works on a Raspberry Pi. See the [Deployment Guide](https://github.com/timvw/soundcork/blob/main/docs/deployment.md) for Docker Compose, Kubernetes, and bare metal options.

### 4. Redirect Your Speaker

```bash
ssh root@<speaker-ip>
rw  # make filesystem writable
vi /opt/Bose/etc/SoundTouchSdkPrivateCfg.xml
```

Change all server URLs to point to your SoundCork instance. Reboot the speaker.

Your speaker now sends all cloud traffic to your server. No data goes to Bose.

## Spotify: It's Complicated

There are two separate Spotify systems on SoundTouch. **Spotify Connect** (cast from the Spotify app) always works — it doesn't touch Bose servers.

**Spotify presets** may fail initially with SoundCork. The fix: play one song via Spotify Connect first. This authenticates the speaker's embedded Spotify client over your local network, and after that, presets work. You may need to repeat this after a reboot.

Details: [Spotify Guide](https://github.com/timvw/soundcork/blob/main/docs/spotify.md)

## Bose Server Status (February 2026)

As of writing, the servers aren't fully shut down yet (that's May 6), but some APIs are already gone:

| Server | Status |
|--------|--------|
| streaming.bose.com (marge) | Alive |
| content.api.bose.io (bmx) | API removed — 404s |
| worldwide.bose.com (updates) | API removed — 404s |

SoundCork handles all of these locally, so it doesn't matter whether Bose's servers are up or down.

## What's Next

SoundCork is a fork of [deborahgu/soundcork](https://github.com/deborahgu/soundcork). I've added Docker support, a smart proxy with circuit breaker for testing, and the documentation you see here. PRs are going upstream.

Bose has also published [official API documentation](https://assets.bosecreative.com/m/496577402d128874/original/SoundTouch-Web-API.pdf) to help the community build tools — which is a good sign.

If you have a SoundTouch speaker, now is the time to set this up — while the servers are still running and you can verify everything works.
