# Technical CTI Case Study: Deconstructing an Automated Task-Scam Infrastructure Network

**Initial Incident Identification:** August 4, 2026  
**Follow-Up Telemetry Window:** October 27, 2026  
**Analyst / Investigator:** BRS404  
**Primary Target Cluster:** `app.l4bkoj45.sbs` / Associated Gname `.sbs` TLD Network  

---

## 1. Executive Summary

This case study documents an end-to-end technical investigation into an active task-and-optimization fraud campaign. Operating under the guise of daily logistics task completions (spoofing brands like AliExpress and MoneyGram), the threat group targets victims primarily across Spanish-speaking markets (Argentina/LATAM). 

The infrastructure leverages automated batch subdomains (`app.`, `ios.`, `svip.`) over cheap `.sbs` string-soup domains registered through Gname, fronted by Cloudflare Anycast WAF proxies. Active incident response, payload analysis, and passive telemetry (Censys, CT logs, Tria.ge sandbox verification, and port scanning) successfully exposed origin hosting on Sharktech (AS46844), Tier-1 gateway redirectors (`shopdd.sbs`), multi-CA TLS automation failovers, out-of-band laundering vectors, and identity theft at the registration level.

---

## 2. Threat Landscape & Social Engineering Mechanics

* **Targeting & Localization:** The landing pages strictly enforce Spanish localization regardless of request origin ("Asignando su ruta, por favor espere - modo normal"). Initial victim engagement originated in Argentina, with sandbox execution verified across European Tria.ge nodes.
* **Brand Impersonation:** The web app pairs fake AliExpress package processing tasks with MoneyGram branding to establish false legitimacy.
* **The Task-Scam / Money Laundering Funnel:**
  1. **Hook:** Promised financial compensation for rating or approving delivery packages via social media advertising (Facebook, Instagram).
  2. **Trust Building:** Scammers request the victim’s Argentine banking identifier (CBU) and issue a small initial payout to build confidence ("Customer Acquisition Tax").
  3. **The Premium Task Lock (Negative Balance Trap):** The system feeds normal micro-tasks to build engagement, eventually hitting the victim with a randomized "high-value / combination task" that plunges their account balance or credit pool into the negative.
  4. **Financial Extortion:** The platform freezes all withdrawals (holding both the victim's initial funds and simulated profits hostage). To unlock the account and complete the task, the victim is forced to deposit real money out-of-pocket.
  5. **Sunk Cost Exploitation:** The platform inflates the apparent commission value of the locked task, manipulating the victim into feeling they are just one more deposit away from a massive payout.
  6. **Out-of-Band (OOB) Control:** All operational communications are funneled directly to Telegram.
  7. **Payment Isolation:** Cash-out and top-up vectors utilize delayed-trace payment mechanisms (MoneyGram, gift cards, and crypto rail placeholders).

---

## 3. Initial Incident & Abuse Mitigation Baseline (August 2026)

* **Incident Discovery:** Intercepted the live malicious endpoint (`app.l4bkoj45.sbs`), analyzed session behavior, and executed active victim intervention to halt pending transactions.
* **Victim Guidance & Escalation:** Advised the victim to immediately notify their banking institution regarding incoming BIC/SWIFT/IBAN transfer loops and to submit a formal incident report to local cybercrime authorities in Argentina.
* **Takedown Filings:** Executed cross-platform abuse filings targeting Cloudflare WAF, Gname Registrar, Google Trust & Safety, Google SafeBrowsing, MSRC, and Telegram handle abuse channels.
* **Domain Seasoning Analysis:** RDAP data confirmed the apex domain was registered on April 14, 2026, and sat dormant for ~4 months until weaponization on August 4, 2026—a deliberate tactic to bypass Newly Registered Domain (NRD) security filters.
* **Fraudulent Registration Identity:** WHOIS records showed the registrar account was created using potentially fraudulent or stolen identity records belonging to a Thai national.
* **Source Code Analysis:** Payload inspection revealed a lightweight web application containing minimal dynamic execution logic and embedded Chinese code comments, indicating the deployment of commercial or AI-assisted phishing kits.

---

## 4. Cryptographic Telemetry & Certificate Transparency (August vs. October 2026)

Comparative analysis of historical telemetry (August 4, 2026) versus follow-up telemetry (October 27, 2026) demonstrates continuous, automated certificate rotation despite registrar-level domain suspension.

### Wildcard CT Provisioning
* **crt.sh Query Behavior:** Direct API queries returned entries exclusively for the apex (`l4bkoj45.sbs`) and wildcard (`*.l4bkoj45.sbs`).
* **Operational Impact:** The actor uses wildcard TLS certificates to dynamically spawn subdomains (`app.`, `ios.`, `svip.`) on reverse proxies without leaving explicit subdomain entries in public Certificate Transparency logs.

### Certificate Rotation Timeline
* **August 4, 2026 Telemetry:**
  * **Issuer:** Let's Encrypt (ACME DV)
  * **Chain:** `ISRG Root X1 → ISRG Root X2 → Root YE → YE2 → End-Entity`
  * **Serial Number:** `06092ECC07130AC7B0AADA5ACD11D45FF95E`
  * **SHA-256 Fingerprint:** `22:34:9F:16:08:8C:AB:1D:57:5C:E8:F4:86:CB:1D:9B:04:39:BE:69:1D:75:C4:D2:95:38:0A:59:99:34:97:B4`
* **October 27, 2026 Telemetry:**
  * **Domain Status:** Marked Inactive (Registrar hold post-abuse filings).
  * **Issuer:** Let's Encrypt (`YE1`)
  * **Algorithm / Key:** EC 256-bit (ECDSA P-256)
  * **Chain:** `ISRG Root X1 → ISRG Root X2 → Root YE → YE1 → End-Entity`
  * **Serial Number:** `06E3E6D9F07BCBD3C7E0A64154218360AB3D`
  * **Validity Range:** Aug 8, 2026 – Jan 6, 2027
  * **SHA-256 Fingerprint:** `38:40:12:BB:71:C2:04:04:BC:78:51:78:71:08:D9:15:D7:81:56:95:1F:B9:9F:38:0E:D0:E3:34:E0:1A:23:F9`

---

## 5. Origin Infrastructure & Multi-Tier Routing Model

Sandbox behavioral runs and passive DNS mapping revealed a multi-tiered infrastructure model:

1. **Tier-1 Gateway Redirector Node (`shopdd.sbs`):**
   * **Role:** Operates as a front-end redirection gateway to absorb initial traffic and ad clicks.
   * **Behavior:** Sandbox analysis confirms `shopdd.sbs` (and its `www` CNAME) executes an immediate dynamic redirect pointing directly into the active `ios.` wildcard task portal on the `l4bkoj45.sbs` cluster.
   * **Infrastructure:** Registered via Gname under privacy proxy services; fronted by Cloudflare WAF on ports `80` and `443` with an active Let's Encrypt DV certificate.
2. **Tier-2 Task Execution Infrastructure (`*.l4bkoj45.sbs`):**
   * **Role:** Hosts the core Spanish task-lock application logic, user session handlers, and payment deposit prompts.
3. **Origin Backend Server (`70.39.73.94` / Sharktech AS46844):**
   * **Environment:** OpenBSD operating system running OpenSSH (Port 22) and F5 Nginx web services.
   * **Active Listening Ports:** `22`, `80`, `888`, `10601–10613` (Exposed high-port HTTP listeners used for backend administration or internal reverse-proxy routing).
   * **Historical Telemetry Note (`hotelcascada.com`):** Passive DNS records previously mapped this origin IP to `1ie5jnc.hotelcascada.com`. Investigation confirms that `hotelcascada.com` has since lapsed, been acquired by GoDaddy via Domains By Proxy, and reassigned to Afternic domain parking infrastructure on AWS (AS16509) serving a standard "domain for sale" landing page. This historical hostname relationship is purely coincidental collateral and is unrelated to the active campaign cluster.

---

## 6. Comprehensive Indicators of Compromise (IoCs)

| Category | Indicator / Artifact | Operational Context |
| :--- | :--- | :--- |
| **Primary Domain** | `app.l4bkoj45.sbs` | Target task-scam landing page |
| **Wildcard Scope** | `*.l4bkoj45.sbs` | Wildcard scope (`app`, `ios`, `svip`) |
| **Tier-1 Gateway** | `shopdd.sbs` | Gateway redirector node (Routes to `ios.` cluster) |
| **Origin Backend IP** | `70.39.73.94` | Direct origin IP hosted on Sharktech (AS46844) |
| **Registrar** | Gname.com Pte. Ltd. (IANA ID: 1923) | Domain registrar (Stolen Thai identity / Privacy proxy) |
| **Registry / TLD** | ShortDot SA (`.sbs`) | TLD Registry operator |
| **Fronting ASN** | Cloudflare, Inc. (AS13335) | WAF / Reverse Proxy Protection |
| **Origin ASN** | Sharktech (AS46844) | Direct origin backend hosting |
| **Exposed Ports** | `22`, `80`, `888`, `10601–10613` | Backend origin management ports |
| **Aug Cert Fingerprint** | `22:34:9F:16:08:8C:AB:1D:57:...` | August Let's Encrypt `YE2` SHA-256 |
| **Oct Cert Fingerprint** | `38:40:12:BB:71:C2:04:04:BC:...` | October Let's Encrypt `YE1` SHA-256 |

---

## 7. External Technical References

* **Tria.ge Sandbox Analysis (Task Endpoint):** `https://tria.ge/260807-wlfr4atwdt/behavioral1`
* **Tria.ge Sandbox Analysis (shopdd Endpoint):** `https://tria.ge/261027-jw39xsbv6z/behavioral1`
* **Censys Platform Cluster Scan:** `https://platform.censys.io/search?q=%22l4bkoj45.sbs%22`
* **VirusTotal Domain Graph:** `https://www.virustotal.com/graph/app.l4bkoj45.sbs`
* **CentralNIC RDAP Telemetry:** `https://rdap.centralnic.com/sbs/domain/l4bkoj45.sbs`
* **DNSChecker References:** `https://dnschecker.org/all-dns-records-of-domain.php?query=app.l4bkoj45.sbs&rtype=ALL&dns=google`
* **DNSChecker IP Location Report:** `https://dnschecker.org/ip-location.php?ip=104.21.49.55`
