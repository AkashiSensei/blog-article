# Solved: Five Reconnect Attempts and Codex Mobile Stuck Waiting for Desktop

> [Original Chinese article on Zhihu](https://zhuanlan.zhihu.com/p/2038938826651989256) · Last edited on 2026-08-04 at 13:37.
>
> This English version is an AI-generated translation for reference only and has not been published.

I'd been eager to try Claude Code's mobile version ever since it came out. Cursor also lets you connect remotely to Cursor on your computer through a mobile browser, but it requires enabling On-Demand Usage, which costs extra.

I've always wanted to keep working on my computer from my phone. QClaw had helped with part of that workflow. Once Codex supported control from a phone, I had to try it immediately.

However, I ran into a problem during setup: the ChatGPT app on my phone kept saying it was waiting for the desktop, while the desktop app remained in the state where it was waiting for me to scan the QR code.

## Suspected Cause

My hypothesis is that the phone stays on `Waiting for desktop` because Codex on the desktop has not successfully established or maintained the persistent WebSocket / streaming connection needed for remote-control.

I believe this has the same underlying cause as the five `reconnecting` attempts at the start of each conversation. I only made the connection after another user pointed it out. My understanding is that the WebSocket / streaming connection fails to establish, and after five attempts, Codex falls back to HTTP mode to continue working.

There are many possible causes. In my case, TUN mode was not enabled in my proxy client, and the proxy configuration in `~/.codex/.env` was incorrect. I had written:

```text
HTTP_PROXY=127.0.0.1:7897
HTTPS_PROXY=127.0.0.1:7897
```

These values are missing the URL scheme, such as `http://`. In my setup, HTTP requests could still go through, but establishing a WebSocket connection failed. The correct format should look like this:

```text
HTTP_PROXY=http://127.0.0.1:7897
HTTPS_PROXY=http://127.0.0.1:7897
```

## What Worked for Me

I now enable TUN mode, delete or clear `~/.codex/.env`, and restart Codex. That resolved the issue for me, with TUN handling proxy routing at the system level.

## Another Option

Another approach is to explicitly route Codex through the proxy. I have not used this method myself, but other users have reported success. Writing the following values into `~/.codex/.env`, or launching remote-control with these environment variables, may also work:

```text
HTTP_PROXY=http://127.0.0.1:7897
HTTPS_PROXY=http://127.0.0.1:7897
ALL_PROXY=http://127.0.0.1:7897
NO_PROXY=localhost,127.0.0.1,::1
```

Adjust the port to match your proxy client. This format is intended for an HTTP or mixed proxy port.

## Or Let Codex Troubleshoot It

You can also give Codex the analysis in this post and ask it to inspect your current proxy configuration, ensuring that remote-control can establish a persistent connection through the proxy.

After making changes, remember to fully quit and restart Codex.
