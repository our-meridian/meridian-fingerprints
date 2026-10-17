# Checking the web page by hand

Your messages are sealed in your browser by the page's JavaScript, so you are trusting that the server sent you the code Meridian published. Meridian publishes a fingerprint (a SHA-256 hash) of every file the page can run, in a small public repository of their own, `our-meridian/meridian-fingerprints`, where whoever runs the server cannot change it. (Meridian's source repository is private; this one holds only the fingerprints and this guide.) A page check in your browser (a service worker) compares each file to those fingerprints automatically; this guide is how to do the same yourself. The design is in `docs/encryption.md` section 5 of the source repository.

## What is published

- `index.json` names the newest release.
- `<n>.json` is release `n`: the Content Security Policy the pages are sent, and the SHA-256 of each file by the path it is served at (`/sw.js`, `/static/meridian-crypto.js`, and so on).
- Read them on GitHub, on the `main` branch of `our-meridian/meridian-fingerprints`, or at `https://raw.githubusercontent.com/our-meridian/meridian-fingerprints/main/index.json`. A release is never edited once it is on `main`.

## Checking one file

Replace `meridian.example` with your Meridian's address. In a terminal:

```sh
curl -s https://meridian.example/static/meridian-crypto.js | sha256sum
curl -s https://raw.githubusercontent.com/our-meridian/meridian-fingerprints/main/1.json | grep meridian-crypto.js
```

The two hashes must be the same (use the number `index.json` gives as `latest`). Do this for `/sw.js` and every file in the manifest; a loop does it:

```sh
site=https://meridian.example
m=https://raw.githubusercontent.com/our-meridian/meridian-fingerprints/main/1.json
curl -s "$m" | python3 -c 'import json,sys; [print(h, p) for p, h in json.load(sys.stdin)["files"].items()]' |
while read -r want path; do
  got=$(curl -s "$site$path" | sha256sum | cut -d" " -f1)
  [ "$got" = "$want" ] && echo "ok       $path" || echo "DIFFERS  $path"
done
```

If the server is mid-update it may serve the next release while GitHub's `latest` is still catching up (or the reverse); try the previous number too. Anything that matches neither is a problem.

## Checking the policy

```sh
curl -sI https://meridian.example/login | grep -i content-security-policy
```

It must equal the `csp` line of the manifest, which allows no inline script or style, nothing from another site, and no `unsafe-eval`. Pages with a looser policy could run code that is not in the list.

## Checking that no page runs anything else

View a page's source (`view-source:`). Every `<script>` must have a `src` starting `/static/` (the one exception is `<script type="application/json">`, which is data) and an `integrity="sha384-..."` attribute; there must be no `onclick=`-style attributes, no `<style>` and no `style=`. Browsers enforce this through the policy anyway.

## What your browser does for you

On your first visit the page installs a service worker, then reloads under it. From then on it checks every script and style the page loads, and the policy of every page, against the fingerprints it was installed with. The foot of each page says which version it has checked ("Page code version 3, checked against the fingerprints published on GitHub"). If something does not match:

- The file is not run. If it was the page itself, you see a page from the checker, not from the server, saying "This page has been changed".
- If GitHub lists a newer release and the server serves all of it, the checker moves to that release by itself. That is how updates arrive.
- A server that has been rolled back to older code is refused too, because the checker never goes back to an older release.

If you see the warning: do not link a phone or write messages. Check the files by hand as above, and tell whoever runs Meridian. If you have just seen an update announced, wait a minute and reload.

## What this does not cover

- Your very first visit, before the checker is installed, is trusted. If you want to be sure, check the files by hand the first time.
- A server could still offer a different checker when the browser looks for a new one, as the browser does that check itself. The hand check above covers `/sw.js`.
- It checks what the page runs, not what it says: a server can still show you wrong words or refuse to serve.
- Private browsing windows that switch service workers off cannot be checked.
