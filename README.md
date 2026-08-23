# LocalCue

A fully offline replacement for Pixel Magic Cue. Runs Gemma 3 1B on your phone,
watches the messaging apps you choose, and shows a cue when something you asked
it to remember is relevant to the conversation.

---

## The security guarantee

`AndroidManifest.xml` does not declare `android.permission.INTERNET`.

On Android, network access is enforced by the kernel. A process whose package
lacks that permission is not placed in the `inet` Linux group and therefore
cannot open a socket. This is not a policy or a promise — it is not physically
possible for this app, or any library inside it, to transmit a byte off your
device. Attempting it throws a `SecurityException`.

Everything else follows from that:

- Memory lives in `/data/data/dev.localcue/files/memory.json`, inside the app's
  private UID sandbox. No other app can read it.
- `allowBackup="false"` plus explicit exclusion rules keep it out of cloud backup
  and device-to-device transfer.
- No analytics, no crash reporting, no third-party SDK except Google's MediaPipe
  inference runtime, which performs local computation only.

## What it can read, and when

| Source | When | Stored? |
|---|---|---|
| Notifications from apps you tick | Whenever one arrives | **No.** Used to decide whether to cue, then discarded. |
| Your screen | Only when you tap the tile or notification button | Yes, that's the point |
| Text shared to the app | Only when you share it | Yes |

The accessibility service is registered so it *can* read the screen on demand.
It does not act on screen events — `onAccessibilityEvent` is empty.

## Battery design

1. **The prefilter.** Before the model is ever woken, a keyword-overlap check
   runs against memory. No overlap, no inference. Most messages cost nothing.
2. **Idle unload.** The model closes after 3 minutes of inactivity, returning
   ~600 MB of RAM. Nothing stays resident.
3. **Gates.** Skipped in battery saver, below your battery threshold, and
   (by default) when the screen is off.
4. **Cooldown.** One cue per app per N seconds, adjustable.
5. **No polling, no timers, no foreground service, no wake locks.**

---

## Building it

You do not need a computer or Android Studio. GitHub compiles it for you.

### 1. Create the repository

On github.com, create a new **empty** repo called `LocalCue`. Do not add a README.

### 2. Push this project from Termux

```
pkg install git
git config --global user.email "you@example.com"
git config --global user.name "You"
cd ~/LocalCue
git init
git add .
git commit -m "initial"
git branch -M main
git remote add origin https://github.com/YOURNAME/LocalCue.git
git push -u origin main
```

GitHub will ask for a password — use a **personal access token**, not your
account password. Create one at github.com → Settings → Developer settings →
Personal access tokens → Tokens (classic) → scope `repo`.

### 3. Collect the APK

Open your repo → **Actions** tab → the running build → wait ~5 minutes →
scroll to **Artifacts** → download `LocalCue-apk` → unzip → install.

Android will warn about installing from an unknown source. That's expected.

### 4. Get the model

Download this file on your phone:

**`Gemma3-1B-IT_multi-prefill-seq_q4_ekv2048.task`**
from `huggingface.co/litert-community/Gemma3-1B-IT`

It's about 550 MB and needs a free Hugging Face account plus accepting Google's
Gemma licence. Save it to Downloads.

### 5. Set it up

Open LocalCue and work down the buttons:

1. **Choose model file** — pick the `.task` file. It copies into private storage
   (takes a minute), after which you can delete the download.
2. **Grant notification access** — find LocalCue, enable it.
3. **Grant screen capture** — Accessibility settings → LocalCue → on.
4. **Apps that can trigger cues** — tick your messaging apps.
5. Long-press the Quick Settings panel and drag the **Remember this screen**
   tile in.

### 6. Use it

- On any screen worth remembering, pull down and tap the tile (or the button in
  the persistent LocalCue notification).
- In your notes app, select a note → Share → LocalCue.
- Then, when a message arrives mentioning something you saved, a cue appears.

Long-press any memory in the app to view or delete it.

---

## Changing the model

Any MediaPipe `.task` bundle works. Gemma 3n E2B (3.1 GB) is more capable and
handles images; it is slower and uses far more RAM. Just pick the different file
in step 5 — no rebuild needed.

## Known limits

- Cues appear as notifications. No app can draw inside another app's UI without
  system privileges, so a true inline chip is not possible.
- Notes apps have no public API. Sharing text to LocalCue is the supported route.
- First inference after idle includes model load time, roughly 2–4 seconds.
