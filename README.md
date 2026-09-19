# 🖥️ Building a Mail Lab from Scratch: VMware → DNS → IIS → hMailServer → Thunderbird

**A step-by-step, screenshot-guided lab**

*Security+ SY0-701 — Day 3 Lab, Section 9:00–12:00*

---

## 📑 Table of Contents

1. [What You Will Build](#1-what-you-will-build)
2. [Key Ideas Before You Start](#2-key-ideas-before-you-start)
3. [Part A — Create the Virtual Machine](#part-a--create-the-virtual-machine)
4. [Part B — Configure the Network Inside the VM](#part-b--configure-the-network-inside-the-vm)
5. [Part C — Install the DNS Server Role](#part-c--install-the-dns-server-role)
6. [Part D — Disable the Firewall and Reach the VM Remotely](#part-d--disable-the-firewall-and-reach-the-vm-remotely)
7. [Part E — Transfer the Lab Files onto the VM](#part-e--transfer-the-lab-files-onto-the-vm)
8. [Part F — Build a DNS Zone by Hand (practice zone)](#part-f--build-a-dns-zone-by-hand-practice-zone)
9. [Part G — Stand Up the Training Bank Sites (bdo.ph.com / bpi.ph.com)](#part-g--stand-up-the-training-bank-sites-bdophcom--bpiphcom)
10. [Part H — Install hMailServer](#part-h--install-hmailserver)
11. [Part I — Install Thunderbird](#part-i--install-thunderbird)
12. [Part J — Configure hMailServer: Domain and Accounts](#part-j--configure-hmailserver-domain-and-accounts)
13. [Part K — Configure Thunderbird's Two Accounts](#part-k--configure-thunderbirds-two-accounts)
14. [Part L — Send and Receive a Test Email](#part-l--send-and-receive-a-test-email)
15. [The Big Picture](#the-big-picture)
16. [Troubleshooting](#troubleshooting)
17. [Check Your Understanding](#check-your-understanding)
18. [Lab Checklist](#lab-checklist)
19. [Glossary](#glossary)

---

## 1. What You Will Build

By the end of this lab you will have, all inside **one Windows Server VM**:

- ✅ A virtual machine built from an ISO, with a static IP and two network adapters
- ✅ A working **DNS Server role**, hosting your own zones
- ✅ Two **look-alike banking websites** (`bdo.ph.com`, `bpi.ph.com`) running in **IIS** over HTTPS with self-signed certificates — a controlled, offline exercise in how phishing infrastructure is built, so you can recognize and defend against it
- ✅ **hMailServer**, a full mail server, with a domain and two mailboxes
- ✅ **Thunderbird**, configured with both mailboxes, sending mail between them

> [!IMPORTANT]
> The "bank" sites in Part G are **training clones only**. They exist inside an isolated lab VM with a self-signed certificate and are never exposed to the real internet. The point is to understand, from the builder's side, how a convincing fake site and matching mail domain go together — so you can spot the same pattern when defending a real network.

---

## 2. Key Ideas Before You Start

### 🧩 One VM, many roles

This whole lab happens inside a **single virtual machine** — everything from the DNS server to the mail server to the fake websites lives on one Windows Server install. The "m" you'll see in some of the raw lab notes (`10.m.1.8`, `azureM.com`) is a **placeholder for your assigned student/PC number**. In these screenshots that placeholder becomes a real number — for example `M = 61` would give `10.61.1.8` and `azure61.com`. Substitute your own assigned number throughout.

### 🗺️ The pipeline

```
ISO  →  VM  →  Static IP  →  DNS role  →  DNS zones  →  IIS websites
                                                              │
                                                              ▼
                                              hMailServer  →  Thunderbird  →  Test email
```

Every later part depends on the one before it: the mail server can't be reached by name until DNS resolves it, and Thunderbird can't connect until hMailServer is running and a domain/account exist.

---

## Part A — Create the Virtual Machine

### Step 1 — Open VMware and create a new VM

Open VMware Workstation → **Create a New Virtual Machine** → **Custom**.

### Step 2 — Set hardware

| Setting | Value |
|---|---|
| Memory | 8 GB |
| Processors | 2 processors × 4 cores |
| Network adapter | Add **two**: one **Bridged** (replicate physical network connection) and one **NAT** |
| Install media | Point the virtual DVD drive at the Windows Server ISO, e.g. `D:\__RivanApps\SERVER_EVAL_x64FRE_en-us.iso` |

> [!NOTE]
> Two network adapters matter later. In this lab's screenshots, **Ethernet0** ends up as the internal lab network (`10.M.1.8`) and **Ethernet1** as the bridged adapter reaching the classroom network (`208.8.8.211`). Keep track of which adapter is which in your own VM.

### Step 3 — Install Windows Server

Power on the VM, choose **Custom install**, and let Setup run. Sign in as `Administrator` once it finishes.

---

## Part B — Configure the Network Inside the VM

### Step 4 — Open adapter settings

Run → `ncpa.cpl` → open **Ethernet0**.

### Step 5 — Set a static IPv4 address

- Disable IPv6 on the adapter (uncheck it).
- Open the IPv4 properties and set:

| Field | Value |
|---|---|
| IP address | `10.M.1.8` *(replace `M` with your own assigned PC number)* |
| Subnet mask | `255.255.255.0` |
| Default gateway | leave blank |
| Preferred DNS server | `127.0.0.1` (the VM will be its own DNS server) |

### Step 6 — Confirm with ipconfig

![Windows IP Configuration output showing Ethernet0 with IPv4 address 10.M.1.8 and Ethernet1 with IPv4 address 208.8.8.211](screenshots/01-windows-ip-configuration-output-showing-ethernet0.png)

*Real output from this lab: **Ethernet0** = `10.M.1.8` (the static address you just set, no default gateway) and **Ethernet1** = `208.8.8.211` (the bridged adapter's DHCP address, used to reach the classroom network). Both matter later — the first is where your DNS/mail server will live, the second is how the instructor's PC or a classmate can reach your VM.*

---

## Part C — Install the DNS Server Role

### Step 7 — Add the DNS Server role

Server Manager → **Manage** → **Add Roles and Features** → **Next** through the wizard → **Server Roles** → check **DNS Server** → **Next**, **Next**, **Install**. Wait for it to finish.

### Step 8 — Verify the role works

```
ping 10.M.1.10
nmap -v 10.M.1.10
```

These are quick reachability checks against another host on the lab network (your instructor's reference server, or a classmate's VM) — not strictly required for DNS to function, but a good habit before building on top of the role.

---

## Part D — Disable the Firewall and Reach the VM Remotely

This section exists so the physical/host PC ("**tunay na PC**" — the real PC, as opposed to the VM) can reach into the VM over the network: to copy files in, and later to browse to the sites you'll build.

### Step 9 — Confirm the firewall is currently blocking you

From the **physical PC**, try to ping the VM's bridged address and it fails:

![Command Prompt showing four failed pings to 208.8.8.211, all Request timed out, followed by pinging 208.8.8.211 again with 4 successful replies](screenshots/02-command-prompt-showing-four-failed-pings.png)

*Top half: `ping 208.8.8.211` — 100% loss, because the VM's Windows Firewall is blocking ICMP. Bottom half: the same ping after the firewall is disabled — 0% loss. This screenshot captures the before/after in one window.*

### Step 10 — Disable the firewall from PowerShell ISE

Inside the VM, open **Windows PowerShell ISE** as Administrator → **Edit → Show Script Pane**. Type the command and run it (▶ or **F5**):

![PowerShell ISE script pane with one line: Set-NetFirewallProfile -name private,public,domain -enabled false](screenshots/03-powershell-ise-script-pane-with-one.png)

```powershell
Set-NetFirewallProfile -name private,public,domain -enabled false
```

This turns the firewall off for **all three profiles** (private, public, domain) at once — appropriate for an isolated lab VM, never for a production or internet-facing machine.

### Step 11 — Comment it out and verify

Once it's run, put `#` in front of that line so you don't accidentally run it again, and add a second line to check the current state:

![PowerShell ISE script pane, line 1 commented out with #, line 2 reading Get-NetFirewallProfile](screenshots/04-powershell-ise-script-pane-line-1.png)

```powershell
#Set-NetFirewallProfile -name private,public,domain -enabled false
Get-NetFirewallProfile
```

You can also confirm visually — search **Firewall & network protection**:

![Windows Security firewall panel, Domain network shows Firewall is off, with a red warning that the device may be unsafe](screenshots/05-windows-security-firewall-panel-domain-network.png)

*The red banner and "Firewall is off" under Domain network confirm it. This warning is expected and fine for a disposable lab VM.*

### Step 12 — Confirm remote access from the host

Back on the physical PC, `Win+R` → `\\208.8.8.211\c$` opens the VM's C: drive over the network:

![File Explorer browsing \\208.8.8.211\c$ showing PerfLogs, Program Files, Program Files (x86), Users, Windows folders](screenshots/06-file-explorer-browsing-20888211c-showing-perflogs.png)

If this folder opens without a password prompt or error, the firewall change worked and you have a path to copy files onto the VM.

---

## Part E — Transfer the Lab Files onto the VM

### Step 13 — Copy the three files across

Using the `\\<VM-IP>\c$` share from the previous step, copy three files onto the VM's `C:\`:

![File Explorer on the VM's own C: drive showing DNS-CONFIG-main(1) folder, Thunderbird Setup 138.0.exe, and hMailServer-5.6.8-B2574.exe](screenshots/07-file-explorer-on-the-vms-own.png)

| File | What it is |
|---|---|
| `DNS-CONFIG-main` | A zipped GitHub repo of PowerShell scripts and site content used later in Part G |
| `Thunderbird Setup 138.0.exe` | The Thunderbird installer, used in Part I |
| `hMailServer-5.6.8-B2574.exe` | The mail server installer, used in Part H |

If `DNS-CONFIG-main` arrives as a `.zip`, right-click → **Extract All** (WinRAR or the built-in Windows extractor both work) before continuing.

---

## Part F — Build a DNS Zone by Hand (practice zone)

Before scripting the real bank-lookalike zones, this lab has you build one zone manually so the pieces (zone → host record → alias → MX record) make sense before automating them.

### Step 14 — Open DNS Manager and add a new zone

Server Manager → **Tools → DNS**. Right-click **Forward Lookup Zones → New Zone…** → **Next** through **Primary zone**:

![New Zone Wizard welcome screen](screenshots/08-new-zone-wizard-welcome-screen.png)

### Step 15 — Name the zone

![New Zone Wizard Zone Name screen with "azureM.com" typed into the Zone name field](screenshots/09-new-zone-wizard-zone-name-screen.png)

*Zone name: `azureM.com` — the "M" placeholder has been replaced with this student's assigned number.*

Finish the wizard. The zone appears, **Running**, as a Standard Primary zone:

![DNS Manager showing azureM.com listed as Standard Primary, Running, Not Signed](screenshots/10-dns-manager-showing-azuremcom-listed-as.png)

### Step 16 — Add a host (A) record

Right-click the zone → **New Host (A or AAAA)…**:

![New Host dialog, empty, inside the azureM.com zone](screenshots/11-new-host-dialog-empty-inside-the.png)

Fill in a name and the VM's own IP:

![New Host dialog filled: Name "ns", FQDN "ns.azureM.com.", IP address 10.M.1.8](screenshots/12-new-host-dialog-filled-name-ns.png)

Click **Add Host** — you'll get a confirmation, and the record appears in the zone alongside the default SOA and NS records:

![DNS zone record list showing SOA, NS, and the new Host (A) record ns pointing to 10.M.1.8](screenshots/13-dns-zone-record-list-showing-soa.png)

### Step 17 — Add an alias (CNAME) and test it

Right-click → **New Alias (CNAME)…**, point `www` at the host you just made:

![New Resource Record dialog, Alias (CNAME) tab, Alias name "www", FQDN "www.azureM.com."](screenshots/14-new-resource-record-dialog-alias-cname.png)

Test from the command line — pinging the alias resolves through to the host record and answers:

![Command Prompt: ping www.azureM.com resolving to ns.azureM.com [10.M.1.8], 4 replies, 0% loss](screenshots/57-ping-www-azurem-com-resolving.png)

### Step 18 — Add a Mail Exchanger (MX) record

Right-click → **New Mail Exchanger (MX)…**:

![New Resource Record dialog, Mail Exchanger tab, FQDN azureM.com, mail server ns.azureM.com, priority 10](screenshots/15-new-resource-record-dialog-mail-exchanger.png)

This tells anything doing mail delivery for `azureM.com` to hand mail to `ns.azureM.com`. You won't run a mail server on this practice zone — the point here is just to see the dialog before repeating the pattern for real in Part G.

---

## Part G — Stand Up the Training Bank Sites (bdo.ph.com / bpi.ph.com)

This is where the three scripts from `DNS-CONFIG-main\bank-phishing-sites` come in: `createDNSZONES.ps1`, `createWEBSITE.ps1`, and `CERTIFICATEWEBSITE.ps1`. Each is written as a **template** using a placeholder domain (`ngcpM.ph`) that you find-and-replace before running — first for `bdo.ph.com`, then again for `bpi.ph.com`.

> [!NOTE]
> The `bank-phishing-sites` folder also contains ready-made lookalike page sets for several real banks and e-wallets (this lab uses `bdo` and `bpi`). This is deliberate: **security professionals build and study convincing fakes in a locked-down lab so they can recognize the real thing in the wild.** None of this is hosted anywhere but your own VM.

### Step 19 — Open the DNS zone script template

Browse into `DNS-CONFIG-main\bank-phishing-sites` in PowerShell ISE's Open dialog — note the other bank folders sitting alongside it:

![Open file dialog inside DNS-CONFIG-main\bank-phishing-sites, listing folders for coinsph, gcash, homecredit, landbank, paymaya, pdax, psbank, securitybank, and files CERTIFICATEWEBSITE.ps1, createDNSZONES.ps1, createWEBSITE.ps1](screenshots/16-open-file-dialog-inside-dns-config-mainbank-phishing-sites-listing.png)

Open `createDNSZONES.ps1`. It starts out templated like this:

![createDNSZONES.ps1 open in ISE showing the ngcpM.ph placeholder template with A, CNAME records and Cisco device records](screenshots/17-creatednszonesps1-open-in-ise-showing-the.png)

```powershell
Add-DnsServerPrimaryZone -Name "ngcpM.ph" -ZoneFile "ngcpM.ph.dns"
add-DnsServerResourceRecord -zonename ngcpM.ph -A -name ns    -ipv4address 10.m.1.8
add-DnsServerResourceRecord -zonename ngcpM.ph -Cname -name www  -hostname ns.ngcpM.ph
add-DnsServerResourceRecord -zonename ngcpM.ph -Cname -name imap -hostname ns.ngcpM.ph
add-DnsServerResourceRecord -zonename ngcpM.ph -Cname -name pop  -hostname ns.ngcpM.ph
add-DnsServerResourceRecord -zonename ngcpM.ph -Cname -name smtp -hostname ns.ngcpM.ph
###FOR CISCO DEVICES DNS RECORDS;
add-DnsServerResourceRecord -zonename ngcpM.ph -A -name cb   -ipv4address 10.m.1.4
...
```

The `imap` / `pop` / `smtp` aliases are the whole reason this section exists before hMailServer: Thunderbird will look up exactly these names later.

### Step 20 — Find-and-replace the placeholder, twice

**First replace** — the domain:

![Replace dialog: Find "ngcpM.ph", Replace with "bdo.ph.com"](screenshots/18-replace-dialog-find-ngcpmph-replace-with.png)

**Second replace** — the machine number in the IP addresses:

![Replace dialog mid-edit with bdo.ph.com already applied to some lines](screenshots/19-replace-dialog-mid-edit-with-bdophcom-already.png)

After both replaces, the script is fully resolved and ready to run:

![createDNSZONES.ps1 fully edited: Add-DnsServerPrimaryZone -Name "bdo.ph.com", records using 10.M.x.x addresses](screenshots/20-creatednszonesps1-fully-edited-add-dnsserverprimaryzone--name-bdophcom.png)

Press **F5** to run it. The `bdo.ph.com` zone now exists with its aliases and the Cisco-device records:

![DNS Manager showing bdo.ph.com zone populated with ap, c1, c2, cb, cm, ct, ed, imap, ns, p1, p2, pop, smtp, www records](screenshots/21-dns-manager-showing-bdophcom-zone-populated.png)

Test the mail alias resolves:

![ping imap.bdo.ph.com resolving through ns.bdo.ph.com [10.M.1.8], 4 replies](screenshots/58-ping-imap-bdo-ph-com-resolving.png)

> [!TIP]
> If you run the script a **second time** by accident, PowerShell prints errors in red like `ResourceExists`. That's not a failure — it means those records are already there. This is the same "turns red, means it already applied" behavior mentioned in the original lab notes.

### Step 21 — Add a bare-domain host record, then the MX record

`bdo.ph.com` itself (no subdomain) also needs a host record before mail can be addressed to it directly:

![New Host dialog: Name blank, FQDN "bdo.ph.com.", IP address 10.M.1.8](screenshots/22-new-host-dialog-name-blank-fqdn.png)

Then add the MX record for the domain:

![New Resource Record, Mail Exchanger tab: FQDN bdo.ph.com, mail server ns.bdo.ph.com, priority 10](screenshots/23-new-resource-record-mail-exchanger-tab.png)

### Step 22 — Edit and run the website script

Open `createWEBSITE.ps1`. In its original template form it installs the Web-Server (IIS) feature and creates a site under the placeholder name:

```powershell
Install-WindowsFeature -name Web-Server -includeManagementTools
New-Website -name "ngcpM.ph" -hostheader "www.ngcpM.ph" -physicalpath "d:\webs\datingbiz"
```

Replace the placeholder and point `-physicalpath` at the actual `bdo` content folder — copy the folder's address bar path out of File Explorer:

![File Explorer inside DNS-CONFIG-main\bank-phishing-sites\bdo showing index and login page files](screenshots/24-file-explorer-inside-dns-config-mainbank-phishing-sitesbdo-showing-index.png)

![createWEBSITE.ps1 edited, physicalpath pointing to C:\DNS-CONFIG-main(1)\DNS-CONFIG-main\bank-phishing-sites\bdo](screenshots/25-createwebsiteps1-edited-physicalpath-pointing-to-cdns-config-main1dns-config-mainbank-phishing-sitesbdo.png)

Run it (**F5**). IIS Manager now shows the site is live:

![Internet Information Services (IIS) Manager start page, connected to the server](screenshots/26-internet-information-services-iis-manager-start.png)

### Step 23 — Browse to the site

Open Edge and visit `http://www.bdo.ph.com`:

![Browser tab "BDO Online — Bank Anytime" showing a mock BDO banking homepage at www.bdo.ph.com](screenshots/27-browser-tab-bdo-online-bank-anytime.png)

DNS resolved the name, IIS served the page — the loop closes.

### Step 24 — Add HTTPS with a self-signed certificate

Open `CERTIFICATEWEBSITE.ps1`:

```powershell
# Create self-signed certificate for www.bpi.ph.com
$cert = New-SelfSignedCertificate -DnsName "www.bpi.ph.com" -CertStoreLocation "Cert:\LocalMachine\My"

# Bind HTTPS to website with correct cert
$siteName = "bpi.ph.com"
$hostname = "www.bpi.ph.com"
$certThumbprint = $cert.Thumbprint

# Ensure SSL binding
New-WebBinding -Name $siteName -Protocol https -Port 443 -HostHeader $hostname

# Bind certificate to HTTPS binding
Push-Location IIS:\SslBindings
New-Item "0.0.0.0!443!$hostname" -Thumbprint $certThumbprint -SSLFlags 1
Pop-Location
```

![CERTIFICATEWEBSITE.ps1 open in ISE with the full self-signed certificate + HTTPS binding script, console showing bpi.ph.com site successfully bound](screenshots/28-certificatewebsiteps1-open-in-ise-with-the.png)

Find/replace `bpi` → `bdo` (this script's placeholder happens to start life pointed at `bpi`, so you may replace in the opposite direction depending on which site you do first) and run it. Browse to `https://www.bdo.ph.com` — the padlock confirms it:

![Browser padlock panel for https://www.bdo.ph.com reading "Connection is secure"](screenshots/29-browser-padlock-panel-for-httpswwwbdophcom-reading.png)

### Step 25 — Repeat everything in this Part for bpi.ph.com

Run the same three scripts again, this time replacing `bdo` → `bpi` throughout:

![Replace dialog inside createDNSZONES.ps1: Find "bdo", Replace with "bpi"](screenshots/30-replace-dialog-inside-creatednszonesps1-find-bdo.png)

Both zones now exist side by side:

![DNS Manager Forward Lookup Zones showing azureM.com, bdo.ph.com, and bpi.ph.com all Running](screenshots/31-dns-manager-forward-lookup-zones-showing.png)

`createWEBSITE.ps1` after its own replace:

![createWEBSITE.ps1 edited for bpi.ph.com, physicalpath pointing to the bpi content folder](screenshots/32-createwebsiteps1-edited-for-bpiphcom-physicalpath-pointing.png)

Browse to the result — a second look-alike login page, now with its own domain and cert:

![Browser tab "BPI Login Clone" showing a mock BPI login page at www.bpi.ph.com](screenshots/33-browser-tab-bpi-login-clone-showing.png)

And the certificate script, re-run for `bpi`:

![Replace dialog inside CERTIFICATEWEBSITE.ps1: Find "bdo", Replace with "bpi"](screenshots/34-replace-dialog-inside-certificatewebsiteps1-find-bdo.png)

![Browser padlock panel for https://www.bpi.ph.com reading "Connection is secure"](screenshots/35-browser-padlock-panel-for-httpswwwbpiphcom-reading.png)

> [!NOTE]
> ✅ **Result of Part G:** two fully working, HTTPS-secured, DNS-resolvable domains — `bdo.ph.com` and `bpi.ph.com` — each with a real IIS site and a matching mail-friendly DNS zone (`imap.`, `pop.`, `smtp.` aliases already in place). Part J will hang a real mailbox off `bdo.ph.com`.

---

## Part H — Install hMailServer

### Step 26 — Enable the .NET Framework 3.5 prerequisite

hMailServer needs .NET Framework 3.5. Server Manager → **Manage → Add Roles and Features** → **Next** through to **Features** → check **.NET Framework 3.5 Features** → **Install**:

![Add Roles and Features Wizard, Select features screen, .NET Framework 3.5 Features checked](screenshots/36-add-roles-and-features-wizard-select.png)

### Step 27 — Run the hMailServer installer

Double-click `hMailServer-5.6.8-B2574.exe` from `C:\`:

![hMailServer Setup Wizard welcome screen, hMailServer 5.6.8-B2574](screenshots/37-hmailserver-setup-wizard-welcome-screen-hmailserver.png)

Click through **Next**. When asked for the database type, use the built-in option:

![Setup - hMailServer, Select database server type, "Use built-in database engine (Microsoft SQL Compact)" selected](screenshots/38-setup---hmailserver-select-database-server.png)

### Step 28 — Set the administrator password

hMailServer creates its own admin account, separate from Windows:

![Setup - hMailServer, hMailServer Security screen, password and confirm password fields](screenshots/39-setup---hmailserver-hmailserver-security-screen.png)

> Password used in this lab: `C1sc0123`. Write down whatever you choose — you'll need it every time you open hMailServer Administrator.

### Step 29 — Install and finish

![Setup - hMailServer, Ready to Install screen, destination C:\Program Files (x86)\hMailServer](screenshots/40-setup---hmailserver-ready-to-install.png)

![Setup - hMailServer, Completing the Setup Wizard, Finish button, Run hMailServer Administrator checked](screenshots/41-setup---hmailserver-completing-the-setup.png)

---

## Part I — Install Thunderbird

### Step 30 — Run the Thunderbird installer

Double-click `Thunderbird Setup 138.0.exe`:

![Mozilla Thunderbird Setup Wizard welcome screen](screenshots/42-mozilla-thunderbird-setup-wizard-welcome-screen.png)

### Step 31 — Choose Standard setup

![Mozilla Thunderbird Setup, Setup Type screen, Standard option selected](screenshots/43-mozilla-thunderbird-setup-setup-type-screen.png)

**Standard** installs Thunderbird with its common defaults — fine for this lab. Finish the wizard.

---

## Part J — Configure hMailServer: Domain and Accounts

### Step 32 — Turn off auto-ban (lab convenience only)

Open **hMailServer Administrator**, enter the password from Step 28. Go to **Settings → Advanced → Auto-ban**, uncheck **Enabled**, and **Save**:

![hMailServer Administrator, Settings > Advanced > Auto-ban screen, Enabled checkbox unchecked](screenshots/44-hmailserver-administrator-settings-advanced-auto-ban-screen.png)

> [!WARNING]
> Auto-ban temporarily blocks an IP address after too many failed logins — a real anti-abuse feature. Disabling it here is purely so repeated test logins during the lab don't lock you out of your own server. **Leave it enabled on any server that isn't a throwaway lab.**

### Step 33 — Add the domain

Right-click **Domains → Add**, type the domain, and **Save**:

![hMailServer Administrator, Domains, Add new domain screen with Domain field showing bdo.ph.com](screenshots/45-hmailserver-administrator-domains-add-new-domain.png)

*Domain: `bdo.ph.com` — the same domain whose DNS zone (with `imap.`, `pop.`, `smtp.` aliases) you built in Part G.*

### Step 34 — Add the "customer support" account

Under the new domain → **Accounts → Add**:

![hMailServer Administrator, new account form under bdo.ph.com, Address field cs, Maximum size 100 MB](screenshots/46-hmailserver-administrator-new-account-form-under.png)

| Field | Value |
|---|---|
| Address | `cs` (becomes `cs@bdo.ph.com`) |
| Password | `pass` |
| Maximum size | `100` MB |

Save. The next screenshot (Step 35) shows `cs@bdo.ph.com` already sitting in the tree once saved.

### Step 35 — Add the "client" account

Repeat, this time using your own name as the address:

![hMailServer Administrator, second account form, Address field "Emman", Maximum size 100 MB, Save button highlighted](screenshots/47-hmailserver-administrator-second-account-form-address.png)

This becomes `emman@bdo.ph.com` — swap in your own name.

---

## Part K — Configure Thunderbird's Two Accounts

### Step 36 — Start adding the customer-support account

Open Thunderbird → **Set Up Your Existing Email Address**:

![Thunderbird Account Setup, Your full name "anne", Email address cs@bdo.ph.com, password entered](screenshots/48-thunderbird-account-setup-your-full-name.png)

| Field | Value |
|---|---|
| Full name | `anne` |
| Email address | `cs@bdo.ph.com` |

Click **Configure manually** rather than letting Thunderbird auto-detect — the mail server is local and needs exact settings.

### Step 37 — Fill in the manual server settings

![Thunderbird manual configuration screen: Incoming IMAP hostname, Outgoing SMTP hostname, both under bdo.ph.com](screenshots/49-thunderbird-manual-configuration-screen-incoming-imap.png)

Use the mail aliases created back in Part G:

| Section | Field | Value |
|---|---|---|
| Incoming server | Protocol | IMAP |
| Incoming server | Hostname | `imap.bdo.ph.com` |
| Outgoing server | Hostname | `smtp.bdo.ph.com` |

![Manual configuration filled in: incoming imap.bdo.ph.com, outgoing smtp.bdo.ph.com](screenshots/50-manual-configuration-filled-in-incoming-imapbdophcom.png)

### Step 38 — Re-test and confirm

Click **Re-test**. A green banner confirms Thunderbird found working ports automatically — **IMAP port 143**, **SMTP port 587**, Normal password authentication:

![Green success banner "The following settings were found by probing the given server", IMAP port 143, SMTP port 587](screenshots/51-green-success-banner-the-following-settings.png)

Click **Done**.

### Step 39 — Add the client (recipient) account the same way

New Account → fill in the second mailbox:

![Thunderbird Account Setup, full name "emman", email address emman@bdo.ph.com](screenshots/52-thunderbird-account-setup-full-name-emman.png)

**Configure manually** again, same hostnames, this time logging in as the second mailbox:

![Manual configuration for the emman account: imap.bdo.ph.com / smtp.bdo.ph.com, username emman](screenshots/53-manual-configuration-for-the-emman-account.png)

Finish and add the account. You can always start a new one from the menu (☰ → **New Account**):

![Thunderbird hamburger menu showing New Account, Create, Open from File, View options](screenshots/54-thunderbird-hamburger-menu-showing-new-account.png)

---

## Part L — Send and Receive a Test Email

### Step 40 — Compose from the customer-support mailbox

With `cs@bdo.ph.com` selected, **Write a new message**:

![Thunderbird compose window, From anne <cs@bdo.ph.com>, To emman@bdo.ph.com, empty subject and body](screenshots/55-thunderbird-compose-window-from-anne-csbdophcom.png)

| Field | Value |
|---|---|
| From | `anne <cs@bdo.ph.com>` |
| To | `emman@bdo.ph.com` |
| Subject / body | anything you like |

Click **Send**.

### Step 41 — Confirm delivery in the other mailbox

Switch to `emman@bdo.ph.com`'s Inbox:

![Thunderbird folder pane showing cs@bdo.ph.com and emman@bdo.ph.com accounts, emman's Inbox showing 1 unread message from anne, subject "unpaid balance from bdo family"](screenshots/56-thunderbird-folder-pane-showing-csbdophcom-and.png)

The message from `anne <cs@bdo.ph.com>` lands in `emman@bdo.ph.com`'s Inbox with an unread badge. Both mailboxes live in the same Thunderbird window because both accounts are on the same install — that's normal for this lab and doesn't reflect how two separate people would use it in real life.

> [!NOTE]
> ✅ **Result:** a message travelled from `cs@bdo.ph.com`, through hMailServer's SMTP service (`smtp.bdo.ph.com`), and landed in `emman@bdo.ph.com`'s mailbox, retrieved over IMAP (`imap.bdo.ph.com`). Every piece — DNS aliases, the mail server, and the mail client — did its job.

---

## The Big Picture

```
DNS ZONE (bdo.ph.com)                MAIL SERVER (hMailServer)            MAIL CLIENT (Thunderbird)
──────────────────────               ──────────────────────────           ─────────────────────────
ns      → 10.M.1.8                  Domain: bdo.ph.com                   Account 1: cs@bdo.ph.com
imap    → ns.bdo.ph.com   ─────────► listens on IMAP (143)      ◄───────  connects via imap.bdo.ph.com
smtp    → ns.bdo.ph.com   ─────────► listens on SMTP (587)      ◄───────  connects via smtp.bdo.ph.com
www     → ns.bdo.ph.com              Accounts: cs, emman                  Account 2: emman@bdo.ph.com
MX      → ns.bdo.ph.com (priority 10)
```

| Layer | What it does | What breaks if it's wrong |
|---|---|---|
| **DNS zone** | Turns `imap.bdo.ph.com` into an IP address | Thunderbird can't even find the server |
| **MX record** | Tells other mail servers where to deliver mail for the domain | Only matters for mail from *outside* this VM |
| **hMailServer** | Actually stores mailboxes and speaks IMAP/SMTP | No mail server = nothing to connect to, regardless of DNS |
| **Thunderbird account** | The client that logs in and shows you the mailbox | Wrong hostname/port here fails even if everything else is correct |

---

## Troubleshooting

**❌ `ping` to the VM times out from the host PC**

The Windows Firewall inside the VM is still on. Revisit Part D — confirm `Get-NetFirewallProfile` shows all three profiles disabled, and check Windows Security's Firewall & network protection page directly.

**❌ Website loads over `http://` but not `https://`**

The certificate script (Part G, Step 24) either wasn't run for that domain, or the find/replace missed a line. Re-open `CERTIFICATEWEBSITE.ps1`, confirm every `bpi`/`bdo` reference matches the site you're testing, and re-run it.

**❌ Thunderbird's Re-test never turns green**

- Confirm hMailServer is actually running (check its Administrator window, or `services.msc` for the hMailServer service).
- Confirm the DNS aliases resolve: `ping imap.bdo.ph.com` and `ping smtp.bdo.ph.com` from inside the VM.
- Double check the account exists in hMailServer under the correct domain, and the password matches.

**❌ Test email never arrives**

- Check the **To** address is spelled exactly right (`emman@bdo.ph.com`, not `emman@bdo.ph`).
- Open hMailServer Administrator → **Utilities → Diagnostics** to run its built-in self-check.
- Confirm Auto-ban (Part J, Step 32) isn't blocking your own repeated test attempts.

**❌ DNS script errors with "ResourceExists" in red**

Not a real error — it means you (or a previous run) already created that record. Safe to ignore, or delete the zone and start fresh if you want a clean run.

---

## Check Your Understanding

**1.** Why does Thunderbird need `imap.bdo.ph.com` to resolve in DNS *before* it can connect to hMailServer?

<details><summary>Answer</summary>

Thunderbird only knows the hostname you typed. Without a DNS record turning that hostname into an IP address, there's nothing for it to connect to — DNS resolution has to succeed before any mail protocol conversation can even start.
</details>

**2.** What's the difference between the MX record and the `www` alias in the `bdo.ph.com` zone?

<details><summary>Answer</summary>

`www` is for browsers finding the *website*. MX tells other mail systems where to deliver *mail* for the domain. They can point at the same server (as here, both resolving to `ns.bdo.ph.com`) but they answer different questions.
</details>

**3.** Why did `createWEBSITE.ps1` need its `-physicalpath` changed, not just the domain name?

<details><summary>Answer</summary>

`-physicalpath` points IIS at the actual folder of HTML files to serve. The domain/hostname and the folder are two separate settings — changing the domain doesn't automatically change which files IIS shows for it.
</details>

**4.** Is `https://www.bdo.ph.com` in this lab actually as trustworthy as the real BDO website?

<details><summary>Answer</summary>

No. The padlock only confirms the connection is *encrypted* with a certificate — it says nothing about who controls the site. This certificate is **self-signed**, meaning the VM vouched for itself; a real browser outside this lab would warn that it isn't issued by a trusted authority. That gap between "looks secure" and "is who it claims to be" is exactly the lesson.
</details>

**5.** Both `cs@bdo.ph.com` and `emman@bdo.ph.com` are visible in the same Thunderbird window. Would that be normal outside a lab?

<details><summary>Answer</summary>

No — normally each mailbox belongs to a different person on a different computer. Having both in one Thunderbird install is a lab convenience so one machine can play both sender and recipient.
</details>

---

## Lab Checklist

- [ ] Created the VM with 8 GB RAM, 2×4 cores, bridged + NAT adapters
- [ ] Set a static IP on Ethernet0 and disabled IPv6
- [ ] Installed the DNS Server role
- [ ] Disabled the firewall and confirmed remote `\\IP\c$` access
- [ ] Copied `DNS-CONFIG-main`, the Thunderbird installer, and the hMailServer installer onto the VM
- [ ] Built the `azureM.com` practice zone by hand (host, alias, MX)
- [ ] Ran `createDNSZONES.ps1`, `createWEBSITE.ps1`, `CERTIFICATEWEBSITE.ps1` for `bdo.ph.com`
- [ ] Repeated all three scripts for `bpi.ph.com`
- [ ] Confirmed both sites load over HTTPS with a secure padlock
- [ ] Installed hMailServer and set the admin password
- [ ] Installed Thunderbird
- [ ] Added the `bdo.ph.com` domain in hMailServer
- [ ] Created the `cs` and client accounts in hMailServer
- [ ] Configured both accounts manually in Thunderbird (imap./smtp. hostnames)
- [ ] Got a green "settings found" result on Re-test for both accounts
- [ ] Sent a test email from `cs@bdo.ph.com` and confirmed it arrived

---

## Glossary

| Term | Meaning |
|---|---|
| **DNS zone** | A container of name-to-IP (and other) records for one domain |
| **A record** | Maps a hostname directly to an IPv4 address |
| **CNAME (alias)** | Maps a hostname to *another hostname*, which is then resolved in turn |
| **MX record** | Tells mail systems which server handles mail for a domain, with a priority |
| **IIS** | Internet Information Services — Windows' built-in web server |
| **Self-signed certificate** | An HTTPS certificate a server issues to itself, rather than a trusted certificate authority issuing it — encrypts the connection but doesn't prove identity |
| **hMailServer** | A free, self-hosted mail server for Windows |
| **IMAP** | The protocol a mail client uses to read/sync mail stored on the server |
| **SMTP** | The protocol used to send mail out to a server |
| **Auto-ban** | hMailServer's feature that temporarily blocks an IP after repeated failed logins |

---

*Security+ SY0-701 — Day 3 Lab, Section 9:00–12:00. Screenshots are from one student's walkthrough (VM name `w22-man`, PC number M); your own IPs, domain names, and folder paths will differ accordingly.*
