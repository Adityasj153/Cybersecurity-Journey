# Email Authentication Audit & Spoof Defense Lab

A fully self-hosted lab demonstrating how SPF, DKIM, and DMARC actually work — by building two independent mail servers and a private DNS server from scratch, exploiting a deliberately weak configuration with a forged "CEO wire transfer" email, then hardening the domain and proving the attack is blocked.

No cloud services. No paid tools. No purchased domain. Every layer — DNS, mail transport, DKIM signing, DMARC evaluation — is self-hosted and inspectable.

📄 **Full narrative write-up:** [link to your Medium article]

---

## Table of Contents

- [Why this exists](#why-this-exists)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Repo structure](#repo-structure)
- [Setup: Phase 1 — Environment](#phase-1--environment)
- [Setup: Phase 2 — Mail servers + DNS](#phase-2--mail-servers--dns)
- [Phase 3 — The exploit](#phase-3--the-exploit)
- [Phase 4 — Hardening](#phase-4--hardening)
- [Phase 5 — Verification](#phase-5--verification)
- [Key finding: what DMARC doesn't protect](#key-finding-what-dmarc-doesnt-protect)
- [Evidence](#evidence)
- [License](#license)

---

## Why this exists

Business Email Compromise (BEC) — attackers impersonating executives or vendors to trigger fraudulent wire transfers — is one of the most financially damaging categories of cybercrime, and the underlying protocol gap is simple: **email's `From:` header is just text with no built-in verification.** SPF, DKIM, and DMARC exist to close that gap, but they're widely misunderstood and frequently misconfigured, even by experienced engineers.

This lab audits a domain in a deliberately weak state, demonstrates a realistic exploit, hardens it correctly, and proves — with real captured SMTP transactions — exactly what the fix does and does not protect against.

## Security Objective

The lab demonstrates a practical email-authentication audit rather than only showing DNS records.

The central question is:

> What changes when a domain moves from weak authentication to SPF/DKIM/DMARC enforcement, and what limitation remains when the sending path itself is authorized?

The final experiment showed that an unauthorized external container was rejected at the SMTP layer, while a forged local-part sent through the authorized sender path could still pass domain-level authentication.

That distinction is the most important finding of the lab.

## Architecture

```
                    ┌──────────────────────┐
                    │   lab-dns (BIND9)    │
                    │   172.28.0.10        │
                    │                      │
                    │  zones:              │
                    │   lab.local          │
                    │   receiver.local     │
                    └──────────┬───────────┘
                               │ DNS resolution
              ┌────────────────┴───────────────────┐
              │                                    │
   ┌──────────▼──────────────┐          ┌──────────▼───────────────┐
   │   mail-sender           │          │   mail-receiver          │
   │   172.28.0.20           │   SMTP   │   172.28.0.30            │
   │   Postfix + Dovecot     │ ───────► │   Postfix + Dovecot      │
   │   OpenDKIM + OpenDMARC  │          │   OpenDKIM + OpenDMARC   │
   │   lab.local             │          │   receiver.local         │
   └─────────────────────────┘          └──────────────────────────┘

   ┌────────────────────────────┐
   │  attacker (ephemeral)      │
   │  172.28.0.99 (unauthorized)│
   │  spun up only for Phase 5  │
   └────────────────────────────┘
```

All three permanent containers run on a single Docker bridge network (`172.28.0.0/24`). The attacker container is created on-demand with `docker run` and destroyed after each test (`--rm`), deliberately outside the trusted SPF range.

The lab follows an **attack → harden → verify → investigate** workflow:

1. Start with a deliberately weak SPF configuration.
2. Send a forged `ceo@lab.local` message.
3. Observe that the message is accepted.
4. Add SPF, DKIM and DMARC controls.
5. Repeat the spoofing attempt from an unauthorized container.
6. Verify SMTP-level rejection.
7. Test the same forged local-part through an authorized sending path.
8. Document what domain-level authentication does — and does not — establish.

## Prerequisites

- Docker Engine + Docker Compose v2 (`docker compose version` should work)
- ~2GB free RAM for the three containers
- `dig` (install via `apt install dnsutils` if missing)
- Linux/WSL2/VM environment recommended (tested on Ubuntu 24.04)

## Repository Structure

```text
Email-Authentication-Spoof-Defense/
├── README.md
├── LICENSE
├── .gitignore
├── docker-compose.yml
├── bind/
│   ├── named.conf.options
│   ├── named.conf.local
│   └── zones/
│       ├── db.lab.local
│       └── db.receiver.local
├── sender/
│   └── config/
├── receiver/
│   └── config/
│       └── postfix-main.cf
├── evidence/
└── docs/
```
---

## Phase 1 — Environment

```bash
# Install Docker Engine (Ubuntu)
curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh
sudo usermod -aG docker $USER
sudo systemctl reboot -i   # or a full reboot — required to pick up group membership

# Verify
docker run hello-world
docker compose version

mkdir -p ~/email-auth-lab && cd ~/email-auth-lab
docker pull ghcr.io/docker-mailserver/docker-mailserver:latest
```

## Phase 2 — Mail servers + DNS

Full `docker-compose.yml`:

```yaml
services:
  dns:
    image: ubuntu/bind9:latest
    container_name: lab-dns
    hostname: dns
    volumes:
      - ./bind/named.conf.local:/etc/bind/named.conf.local
      - ./bind/named.conf.options:/etc/bind/named.conf.options
      - ./bind/zones:/etc/bind/zones
    networks:
      lab-net:
        ipv4_address: 172.28.0.10
    restart: unless-stopped
    # Note: no ports published to host — systemd-resolved already binds :53
    # on most Linux hosts. Containers reach BIND over the internal network only.

  mail-sender:
    image: ghcr.io/docker-mailserver/docker-mailserver:latest
    container_name: mail-sender
    hostname: mail.lab.local
    domainname: lab.local
    dns:
      - 172.28.0.10
    ports:
      - "25:25"
      - "587:587"
    volumes:
      - ./sender/mail-data:/var/mail
      - ./sender/mail-state:/var/mail-state
      - ./sender/mail-logs:/var/log/mail
      - ./sender/config:/tmp/docker-mailserver
      - /etc/localtime:/etc/localtime:ro
    environment:
      - ENABLE_OPENDKIM=1
      - ENABLE_OPENDMARC=1
      - ENABLE_POLICYD_SPF=1
      - PERMIT_DOCKER=network
      - ONE_DIR=1
    cap_add:
      - NET_ADMIN
    networks:
      lab-net:
        ipv4_address: 172.28.0.20
    restart: unless-stopped

  mail-receiver:
    image: ghcr.io/docker-mailserver/docker-mailserver:latest
    container_name: mail-receiver
    hostname: mail.receiver.local
    domainname: receiver.local
    dns:
      - 172.28.0.10
    ports:
      - "2525:25"
    volumes:
      - ./receiver/mail-data:/var/mail
      - ./receiver/mail-state:/var/mail-state
      - ./receiver/mail-logs:/var/log/mail
      - ./receiver/config:/tmp/docker-mailserver
      - /etc/localtime:/etc/localtime:ro
    environment:
      - ENABLE_OPENDKIM=1
      - ENABLE_OPENDMARC=1
      - ENABLE_POLICYD_SPF=1
      # Deliberately NOT "network" — see receiver/config/postfix-main.cf.
      # PERMIT_DOCKER=network auto-trusts 172.16.0.0/12, which silently skips
      # SPF/DKIM/DMARC evaluation for anything on the lab's own Docker subnet.
      # That defeats the point of the lab, since mail-sender's traffic would
      # never actually be checked. Set to "none" and override mynetworks below.
      - PERMIT_DOCKER=none
      - ONE_DIR=1
    cap_add:
      - NET_ADMIN
    networks:
      lab-net:
        ipv4_address: 172.28.0.30
    restart: unless-stopped

networks:
  lab-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/24
```

`bind/named.conf.options`:
```
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-recursion { any; };
    forwarders { 8.8.8.8; 1.1.1.1; };
    dnssec-validation auto;
    listen-on { any; };
    listen-on-v6 { any; };
};
```

`bind/named.conf.local`:
```
zone "lab.local" {
    type master;
    file "/etc/bind/zones/db.lab.local";
};
zone "receiver.local" {
    type master;
    file "/etc/bind/zones/db.receiver.local";
};
```

Bring it up and create the required mailboxes (`docker-mailserver` refuses to start Dovecot with zero accounts):

```bash
docker compose up -d
docker exec mail-sender setup email add admin@lab.local 'YourPassword123'
docker exec mail-receiver setup email add admin@receiver.local 'YourPassword123'
docker compose ps   # both should report (healthy)
```

## Phase 3 — The exploit

**`bind/zones/db.lab.local` — vulnerable ("before") state:**
```
$TTL    3600
@   IN  SOA dns.lab.local. admin.lab.local. ( 2026091801 3600 1800 604800 3600 )
@       IN  NS      dns.lab.local.
dns     IN  A       172.28.0.10
@       IN  MX  10  mail.lab.local.
mail    IN  A       172.28.0.20
@       IN  TXT     "v=spf1 +all"
```

Send a forged executive-impersonation email — `ceo@lab.local` is not a real mailbox anywhere in this lab:

```bash
docker exec mail-sender swaks \
  --to admin@receiver.local \
  --from ceo@lab.local \
  --server mail.receiver.local \
  --header "Subject: URGENT: Wire Transfer Approval Needed" \
  --body "Please process the attached wire transfer immediately. - CEO"
```

**Result:** delivered cleanly.
```
Authentication-Results: mail.receiver.local; dmarc=none (p=none dis=none) header.from=lab.local
Received-SPF: Pass (mailfrom) identity=mailfrom; client-ip=172.28.0.20;
  envelope-from=ceo@lab.local; receiver=receiver.local
```

## Phase 4 — Hardening

Generate a real DKIM key pair:
```bash
docker exec mail-sender setup config dkim
docker exec mail-sender cat /tmp/docker-mailserver/opendkim/keys/lab.local/mail.txt
```

Replace `bind/zones/db.lab.local` with the hardened version:
```
$TTL    3600
@   IN  SOA dns.lab.local. admin.lab.local. ( 2026091802 3600 1800 604800 3600 )
@       IN  NS      dns.lab.local.
dns     IN  A       172.28.0.10
@       IN  MX  10  mail.lab.local.
mail    IN  A       172.28.0.20

; Restrictive SPF — only the real mail server may send as this domain
@       IN  TXT     "v=spf1 ip4:172.28.0.20 -all"

; DKIM public key (value from `setup config dkim` above)
mail._domainkey IN TXT ( "v=DKIM1; h=sha256; k=rsa; "
    "p=<your generated public key>" )

; Strict-alignment DMARC policy — reject on failure
_dmarc  IN  TXT     "v=DMARC1; p=reject; adkim=s; aspf=s; rua=mailto:dmarc-reports@lab.local"
```

```bash
docker compose restart dns mail-sender
docker exec mail-sender dig @172.28.0.10 lab.local TXT
docker exec mail-sender dig @172.28.0.10 mail._domainkey.lab.local TXT
docker exec mail-sender dig @172.28.0.10 _dmarc.lab.local TXT
```

## Phase 5 — Verification

**Legitimate mail still works** (sanity check):
```bash
docker exec mail-sender swaks --to admin@receiver.local --from admin@lab.local --server mail.receiver.local
# → dmarc=pass (p=reject dis=none)
```

**Simulated external attacker** — a throwaway container on an IP with no SPF authorization:
```bash
docker run --rm --network email-auth-lab_lab-net --ip 172.28.0.99 ubuntu:24.04 bash -c \
  "apt-get update -qq && apt-get install -y -qq swaks && \
   swaks --to admin@receiver.local --from ceo@lab.local --server 172.28.0.30 \
     --header 'Subject: URGENT: Wire Transfer Approval Needed' \
     --body 'Please process the attached wire transfer immediately. - CEO'"
```

**Result:**
```
<** 550 5.7.23 <admin@receiver.local>: Recipient address rejected:
Message rejected due to: SPF fail - not authorized.
```

Blocked at the SMTP protocol level — never accepted onto the receiving server.

## Key finding: what DMARC doesn't protect

Re-running the identical forged "CEO" email from the *authorized* sending server (`172.28.0.20`) instead of the unauthorized attacker IP **still passes** every check, including DMARC:

```
Authentication-Results: mail.receiver.local; dmarc=pass (p=reject dis=none) header.from=lab.local
```

SPF and DMARC alignment validate the **sending domain**, not the **mailbox local-part**. Anyone with access to a server or account already authorized to relay mail for a domain can still impersonate any other address `@thatdomain` — `ceo@`, `billing@`, `admin@`, anything. This is a genuine limitation of domain-level email authentication, not a misconfiguration, and it's the exact gap that behavior-based tools (Sublime Security, Abnormal Security, Microsoft Defender's newer LLM-based filtering) are built to address on top of SPF/DKIM/DMARC.

| Scenario | Sender IP | SPF | DMARC | Outcome |
|---|---|---|---|---|
| Baseline spoof (before hardening) | 172.28.0.20 (authorized, `+all`) | Pass | none | Delivered, unflagged |
| Same-domain impersonation (after hardening) | 172.28.0.20 (authorized) | Pass | pass | Delivered — domain-level checks can't see the local-part |
| External attacker (after hardening) | 172.28.0.99 (unauthorized) | **Fail** | n/a | **Blocked — 550 SPF fail** |

# 8. What This Lab Demonstrates

### Before hardening

```text
Forged From:
ceo@lab.local
      │
      ▼
Weak SPF (+all)
      │
      ▼
Authentication not enforced
      │
      ▼
Message delivered
```

### After hardening

```text
Unauthorized sender
172.28.0.99
      │
      ▼
SPF check
      │
      ▼
SPF FAIL
      │
      ▼
SMTP 550 rejection
```

### Authorized-path impersonation

```text
Forged local-part
ceo@lab.local
      │
      ▼
Authorized sending path
172.28.0.20
      │
      ▼
Domain authentication passes
      │
      ▼
Message delivered
      │
      ▼
Human identity still requires
additional controls
```

---

# 9. Security Lessons

- SPF should identify authorized sending infrastructure rather than using permissive policies such as `+all`.
- DKIM provides cryptographic signing for messages.
- DMARC connects authentication results with the visible `From:` domain through alignment.
- Receiver trust configuration matters: internal network membership should not automatically bypass authentication testing.
- SMTP-level rejection provides strong evidence that the unauthorized sender was blocked.
- Domain authentication and human identity are different security questions.
- Email authentication should be combined with additional identity, authorization and detection controls.

---

# 10. Evidence

The `evidence/` directory is intended to contain screenshots documenting:

1. Lab architecture
2. Vulnerable spoofing attempt
3. Authentication results
4. SPF configuration
5. DKIM configuration
6. DMARC configuration
7. External attacker attempt
8. SMTP `550` rejection
9. Authorized-path impersonation finding

Keep screenshots free of real credentials, private keys and unnecessary personal information.

---

# 11. Reproducibility Notes

The project uses mutable Docker image tags in the supplied implementation. For a production-quality reproduction, pin tested image versions after establishing the exact versions used in the lab.

The project intentionally does not fabricate configuration files whose complete contents were not part of the supplied implementation. Copy the actual `db.receiver.local` and sender configuration from the lab environment before treating the repository as a complete one-command reproduction.

---

## Project Focus

**Security domains:** Email Security · SOC · Blue Team · Network Security

**Technologies:** Docker · Docker Compose · BIND9 · Postfix · Dovecot · OpenDKIM · OpenDMARC · Swaks · DNS · SPF · DKIM · DMARC

**Core workflow:**

```text
Build → Attack → Observe → Harden → Verify → Investigate → Document
```

## Author

**Aditya**

GitHub: `Adityasj153`

---

## License

MIT
