# Task Voice

A single-page Progressive Web App (PWA). Tap the button once to start
recording your voice, tap it again to stop and automatically send the
recording to an n8n webhook as a multipart form upload. While recording you
can pause/resume or discard and restart from scratch.

No frameworks, no build step, no external libraries or CDN dependencies.
Just static files.

## Files

- `index.html` — the entire app (markup, styles, and JavaScript)
- `manifest.json` — PWA manifest (lets it be installed on the Android home screen)
- `sw.js` — service worker for basic offline support
- `icon.svg` — microphone emoji app icon placeholder

## Before you deploy: set your webhook URL and secret token

Open [index.html](index.html) and find the configuration block near the top
of the `<script>` section:

```js
// =========================================================
// CONFIGURATION - EDIT THESE TWO VALUES BEFORE DEPLOYING
// =========================================================
const N8N_WEBHOOK_URL = "https://n8n.srv1322057.hstgr.cloud/webhook/80acff81-4ad5-478b-8323-095bc9029d86"; // Replace this with your actual value
const SECRET_TOKEN = "DavidVoiceMessageApp"; // Replace this with your actual value
// =========================================================
```

Replace `N8N_WEBHOOK_URL` with the production webhook URL from your n8n
Webhook node, and `SECRET_TOKEN` with the secret value your n8n workflow
expects to receive in the `X-Secret-Token` header. Save the file before
deploying.

> Note: because this is a static, client-side app, the secret token is
> visible to anyone who views the page source. Treat it as a shared-secret
> deterrent rather than a strong security boundary. If you need real
> authentication, put a proxy or auth layer in front of the webhook.

## Deploying to Vercel

### Option A: Vercel CLI

1. Install the CLI if you don't have it: `npm install -g vercel`
2. From this project folder, run:
   ```
   vercel
   ```
3. Follow the prompts (link or create a project). Choose the defaults —
   this is a static site, no build command or output directory needed.
4. For a production deployment:
   ```
   vercel --prod
   ```

### Option B: Vercel dashboard (no CLI)

1. Push this folder to a GitHub/GitLab/Bitbucket repository.
2. Go to [vercel.com/new](https://vercel.com/new) and import the repository.
3. Framework preset: choose **Other** (no build step is required).
4. Leave the build command empty and output directory as the repo root.
5. Click **Deploy**.

After deployment, Vercel gives you a URL like `https://task-voice.vercel.app`.
Open it on an Android Chrome device.

## Installing on Android home screen

1. Open the deployed URL in Chrome on Android.
2. Tap the three-dot menu → **Add to Home screen** (or **Install app**).
3. The app will launch standalone (no browser address bar) using the
   microphone icon.

## Updating the webhook URL or secret token later

1. Edit the two constants described above in `index.html`.
2. Redeploy:
   - CLI: `vercel --prod`
   - Dashboard: push the change to your connected git branch; Vercel
     redeploys automatically.
3. On the Android device, the new service worker version will pick up the
   updated `index.html` the next time the app is opened with a network
   connection (it caches the previous version for offline use until then).

## How it works

- **Tap once** on the main button to start recording. The app requests
  microphone access via `navigator.mediaDevices.getUserMedia({ audio: true })`
  and starts a `MediaRecorder`.
- While recording, two extra controls appear:
  - **⏸ Pause / ▶ Resume** — pauses the recording at its current length via
    `MediaRecorder.pause()`; tapping it again calls `MediaRecorder.resume()`
    and continues recording onto the same clip (it does not start over).
  - **🗑 Restart** — discards everything recorded so far and immediately
    starts a brand new recording from zero. Nothing is sent for a discarded
    take.
- **Tap the main button again** (now labeled "Tap to Stop & Send") to stop
  and automatically send the recording.
- The app picks the best supported audio format using
  `MediaRecorder.isTypeSupported`, preferring `audio/webm;codecs=opus`
  (Android Chrome's native format), with `audio/mp4` and other fallbacks.
- On stop, the audio `Blob` is wrapped in `FormData`, and a `POST` request is
  sent to `N8N_WEBHOOK_URL` with header `X-Secret-Token: <SECRET_TOKEN>`.
- Status text cycles through **Ready → Recording → (Paused) → Sending →
  Done**, or **Error** with a **Retry** button if the request fails.
- The timestamp of the last successful submission is stored in
  `localStorage` and shown in small text at the bottom of the screen.
- If the browser doesn't support `MediaRecorder` or `getUserMedia`, the
  button is disabled and a clear error message is shown.
- The service worker caches the app shell (`index.html`, `manifest.json`,
  `icon.svg`) for offline loading, and shows "You are offline" status if
  there's no network connection when you try to record or send. The
  webhook POST request always goes directly to the network and is never
  intercepted or cached by the service worker.

## Local testing

Because the app uses `getUserMedia`, most browsers require either
`localhost` or HTTPS to grant microphone access. Serve the folder locally,
for example:

```
npx serve .
```

or with Python:

```
python -m http.server 8080
```

Then open `http://localhost:8080` in Chrome.
