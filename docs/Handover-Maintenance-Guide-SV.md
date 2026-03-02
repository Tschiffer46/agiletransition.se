# Överlämnings- och underhållsguide

**Dokumentversion:** 1.0  
**Senast uppdaterad:** 2026-03-02  
**Upprättad av:** ATM AB (Agile Transition AB, org nr 559378-3045)  
**Målgrupp:** Underhållsorganisation som tar över ansvaret för de fyra webbplatserna

---

## Innehållsförteckning

1. [Introduktion och lösningsöversikt](#1-introduktion-och-lösningsöversikt)
2. [Arkitektur](#2-arkitektur)
3. [Teknikstack per webbplats](#3-teknikstack-per-webbplats)
4. [Driftsättningsmetod](#4-driftsättningsmetod)
5. [Serveråtkomst och användare](#5-serveråtkomst-och-användare)
6. [Säkerhet](#6-säkerhet)
7. [Hosting och infrastruktur](#7-hosting-och-infrastruktur)
8. [Programuppgraderingar och underhåll](#8-programuppgraderingar-och-underhåll)
9. [Felsökningsreferens](#9-felsökningsreferens)
10. [Viktiga länkar och åtkomstpunkter](#10-viktiga-länkar-och-åtkomstpunkter)
11. [Hur man gör innehållsändringar](#11-hur-man-gör-innehållsändringar)
12. [Kontaktformulärsbackend](#12-kontaktformulärsbackend)

---

## 1. Introduktion och lösningsöversikt

**ATM AB (Agile Transition AB, org nr 559378-3045)** har utformat och byggt samtliga fyra webbplatser som beskrivs i detta dokument. Guiden är avsedd att ge en underhållsorganisation allt den behöver för att driva, uppdatera och felsöka lösningen framöver.

### De fyra webbplatserna

| Webbplats | Domän | GitHub-repository | Typ |
|-----------|-------|-------------------|-----|
| Agile Transition | `agiletransition.se` | `Tschiffer46/agiletransition.se` | Statisk HTML (inget byggsteg) |
| AZ Profil | `azprofil.agiletransition.se` | `Tschiffer46/azprofil` | React + Vite statisk webbplats |
| Padel to Business (azP2B) | `azp2b.agiletransition.se` | `Tschiffer46/azP2B` | React + Vite statisk webbplats |
| Hemsidor | `hemsidor.agiletransition.se` | `Tschiffer46/hemsidor` | React + Vite statisk webbplats |

Alla fyra webbplatser är **helt statiska** — det finns ingen serverbaserad applikationslogik och ingen databas. Innehållet serveras som färdigbyggda HTML/CSS/JS-filer.

---

## 2. Arkitektur

### Trafikflöde

```
Användare → Cloudflare (DNS + CDN + HTTPS) → Hetzner-server (89.167.90.112) → Nginx Proxy Manager → Docker-containrar (nginx:alpine som serverar statiska filer)
```

När en besökare öppnar en av de fyra webbplatserna passerar begäran genom följande lager:

1. **Cloudflare** löser upp domännamnet (DNS), cachar statiska resurser (CDN) och terminerar HTTPS-anslutningen från webbläsaren.
2. **Hetzner-servern** (IP `89.167.90.112`) tar emot begäran från Cloudflare.
3. **Nginx Proxy Manager** (kör som en Docker-container på servern) granskar `Host`-headern och vidarebefordrar begäran till rätt webbplats-container.
4. **Webbplats-containern** (en Docker-container med `nginx:alpine`) läser de statiska filerna från `dist/`-katalogen på disk och returnerar dem till webbläsaren.

### Viktiga arkitekturfakta

- Alla fyra webbplatser körs på **en enda Hetzner-server** (IP: `89.167.90.112`).
- Servern kör **Ubuntu** med **Docker + Docker Compose**.
- Alla webbplatser definieras i **en gemensam `docker-compose.yml`** på `/home/deploy/hosting/docker-compose.yml`.
- Varje webbplats är en separat Docker-container med `nginx:alpine` som serverar statiska filer från sin egen `dist/`-katalog.
- **Nginx Proxy Manager** tillhandahåller ett webbgränssnitt på `http://89.167.90.112:81` och hanterar all trafikroutning, SSL-certifikat (Let's Encrypt) och domänmappning.
- Det finns **ingen vanlig Nginx installerad på värdsystemet** — inga `/etc/nginx/`-konfigurationsfiler finns på servern.
- Alla containrar delar ett Docker-nätverk med namnet `web`.
- **Cloudflare** hanterar DNS, CDN-cachning och HTTPS-anslutningen från användaren till Cloudflare. Let's Encrypt-certifikat (hanterade av Nginx Proxy Manager) sköter anslutningen från Cloudflare till servern.

---

## 3. Teknikstack per webbplats

### agiletransition.se

- Vanlig HTML — **inget byggsteg krävs**
- Driftsätts som råa HTML-filer via `rsync`
- En `CNAME`-fil finns i repositoryt (historisk GitHub Pages-artefakt; primär hosting är Hetzner)

### azprofil.agiletransition.se

- React 19 + TypeScript + Vite 7 + Tailwind CSS 3
- `react-i18next` — stöd för svenska och danska
- `react-router-dom` — klientsidans routing
- Formspree — backend för kontaktformulär
- **Driftsättningssökväg på servern:** `~/hosting/sites/client-azprofil/dist/`

### azp2b.agiletransition.se (Padel to Business)

- React 19 + TypeScript + Vite 6 + Tailwind CSS 3
- `react-router-dom` — klientsidans routing
- Endast svenska (inline i18n i `src/i18n.ts`)
- **Driftsättningssökväg på servern:** `~/hosting/sites/client-azp2b/dist/`
- > **Notera:** Denna webbplats byggdes ursprungligen med Next.js + Docker + MariaDB, men förenklades till en statisk Vite-webbplats (PR #12). Den nuvarande versionen har **ingen databas**.

### hemsidor.agiletransition.se

- React 19 + TypeScript + Vite 6 + Tailwind CSS 3
- Formspree för kontaktformulär (formulär-ID: `xreawgqr`)
- Endast svenska
- **Driftsättningssökväg på servern:** `~/hosting/sites/hemsidor/dist/`

---

## 4. Driftsättningsmetod

### Översikt

Alla fyra repositorier använder **GitHub Actions** för automatisk driftsättning. När kod pushas till `main`-grenen aktiveras arbetsflödet som definieras i `.github/workflows/deploy.yml` i respektive repository.

### Vad arbetsflödet gör

1. **Checka ut** repository-koden.
2. **Konfigurera Node.js 20** (endast för React-webbplatser; hoppas över för den statiska HTML-webbplatsen).
3. **`npm ci`** — installera exakta beroendeversioner från `package-lock.json`.
4. **`npm run build`** — kompilera React/TypeScript/Vite-projektet till statiska filer i `dist/`.
5. **Konfigurera SSH-nyckel** från GitHub Secrets.
6. **`rsync`** — ladda upp `dist/`-mappen (eller repo-roten för `agiletransition.se`) till Hetzner-servern.

### Driftsättningsegenskaper

- **Driftsättning utan driftstopp**: `rsync` uppdaterar filer på plats medan `nginx:alpine`-containern fortsätter att hantera förfrågningar.
- **Flaggan `--delete`** på `rsync` säkerställer att filer som tagits bort från repositoryt också tas bort från servern.
- Typisk driftsättningstid: **~2 minuter** från push till live.

### Obligatoriska GitHub Secrets

Följande tre hemligheter måste konfigureras i varje repositorys **Settings → Secrets and variables → Actions**. Värdena är identiska i alla fyra repositorier.

| Hemlighet | Beskrivning |
|-----------|-------------|
| `SERVER_HOST` | Hetzner-serverns IP-adress (`89.167.90.112`) |
| `SERVER_USER` | SSH-användarnamn (`deploy`) |
| `SERVER_SSH_KEY` | Privat SSH-nyckel för `deploy`-användaren |

> ⚠️ **Om någon av dessa hemligheter saknas eller är felaktig misslyckas driftsättningen.** Kontrollera GitHub Actions-loggen för detaljer.

---

## 5. Serveråtkomst och användare

| Användare | Syfte | Åtkomst |
|-----------|-------|---------|
| `root` | Serveradministration | SSH via Terminus |
| `deploy` | Alla driftsättningar — GitHub Actions ansluter som denna användare | SSH via Terminus eller automatiskt via GitHub Actions |

- **Rekommenderad SSH-klient:** Terminus
- `deploy`-användaren måste vara medlem i `docker`-gruppen. Verifiera med:

  ```bash
  # Kör som root
  groups deploy
  ```

  Om `docker` inte listas, lägg till användaren:

  ```bash
  # Kör som root
  usermod -aG docker deploy
  ```

- **SSH-autentisering sker enbart med nyckel** (lösenordsinloggning är inaktiverat). Den privata nyckeln som lagras i GitHub-hemligheten `SERVER_SSH_KEY` motsvarar en publik nyckel i `/home/deploy/.ssh/authorized_keys`.

---

## 6. Säkerhet

| Ämne | Detaljer |
|------|----------|
| **SSH-åtkomst** | Enbart nyckelbaserad autentisering — lösenordsinloggning är inaktiverat |
| **Cloudflare** | Tillhandahåller DDoS-skydd, CDN-cachning och SSL-terminering för slutanvändare |
| **Let's Encrypt** | SSL-certifikat mellan Cloudflare och servern, hanteras automatiskt av Nginx Proxy Manager |
| **GitHub Secrets** | SSH-nycklar och serveruppgifter lagras som GitHub Actions-hemligheter — publiceras aldrig i källkoden |
| **Docker-containrar** | Körs med `restart: unless-stopped`; statiska filvolymer monteras skrivskyddade (`:ro`) |
| **Nginx Proxy Manager** | "Block Common Exploits" är aktiverat på alla proxyvärdar |
| **Ingen databas eller känsliga data** | Alla fyra webbplatser är helt statiska — inga användardata lagras på servern |

> ⚠️ **Publicera aldrig SSH-nycklar, lösenord eller API-token i något repository.** Använd alltid GitHub Secrets.

---

## 7. Hosting och infrastruktur

| Komponent | Detaljer |
|-----------|----------|
| **Serverleverantör** | Hetzner (hetzner.com) |
| **Serverns IP** | `89.167.90.112` |
| **Operativsystem** | Ubuntu |
| **DNS och CDN** | Cloudflare (dash.cloudflare.com) |
| **Docker** | Docker + Docker Compose installerat på servern |
| **Nginx Proxy Manager** | Kör som en Docker-container; tillgänglig på `http://89.167.90.112:81` |

### Cloudflare DNS-konfiguration

Alla domäner har **A-poster** som pekar på `89.167.90.112`. **Proxystatus är aktiverad** (orange moln PÅ) för alla domäner, vilket innebär att all trafik passerar genom Cloudflare innan den når servern.

### Nginx Proxy Manager

- Tillgänglig på `http://89.167.90.112:81` (webbgränssnitt)
- Varje webbplats har en **Proxy Host**-post som mappar domännamnet till containernamnet på port 80
- Hanterar utfärdande och förnyelse av SSL-certifikat via Let's Encrypt
- Kontrollera dashboarden regelbundet för att bekräfta att alla proxyvärdar är aktiva och att SSL-certifikaten är giltiga

---

## 8. Programuppgraderingar och underhåll

Det här avsnittet är centralt för att hålla lösningen säker och driftsäker på lång sikt.

### Vad behöver uppgraderas och när

#### 1. Ubuntu OS-paket

Säkerhetsuppdateringar bör appliceras regelbundet på serverns operativsystem.

```bash
# Kör som root
sudo apt update && sudo apt upgrade -y
```

- **Rekommenderad frekvens:** Månadsvis, eller omedelbart när en säkerhetsvarning publiceras
- **Valfritt:** Aktivera `unattended-upgrades` för automatiska säkerhetsuppdateringar:

  ```bash
  # Kör som root
  apt install unattended-upgrades
  dpkg-reconfigure --priority=low unattended-upgrades
  ```

#### 2. Docker och Docker Compose

Uppdatera när nya stabila versioner släpps.

```bash
# Kör som root — kontrollera nuvarande versioner
docker --version
docker compose version
```

Följ [Dockers officiella dokumentation](https://docs.docker.com/engine/install/ubuntu/) för uppgradering på Ubuntu.

#### 3. Nginx Proxy Manager

Uppdatera Nginx Proxy Manager-containeravbildningen.

```bash
# Kör som deploy, i /home/deploy/hosting/
docker compose pull proxy-manager
docker compose up -d proxy-manager
```

#### 4. nginx:alpine-webbplatscontainrar

Webbplatscontainrarna använder `nginx:alpine` (latest-taggen). Hämta den senaste avbildningen och återskapa containrarna:

```bash
# Kör som deploy, i /home/deploy/hosting/
docker compose pull
docker compose up -d
```

> ⚠️ Att köra `docker compose up -d` efter en pull återskapar endast de containrar vars avbildningar har ändrats. Befintliga containrar fortsätter att hantera trafik tills de ersätts — driftstoppet är minimalt.

#### 5. Node.js i GitHub Actions

GitHub Actions-arbetsflödena är för närvarande konfigurerade med **Node.js 20**. När Node 20 når slutet av sin livscykel, uppdatera fältet `node-version` i alla fyra `deploy.yml`-filer:

```yaml
# .github/workflows/deploy.yml — ändra denna rad
- uses: actions/setup-node@v4
  with:
    node-version: '20'   # Uppdatera till nästa LTS-version (t.ex. '22')
```

#### 6. npm-beroenden i React-repositorierna

Varje React-repository har `npm`-beroenden som med tiden kan samla säkerhetssårbarheter.

```bash
# Kör lokalt i respektive React-repository
npm audit
```

- Använd `npm update` för mindre- och patchuppdateringar
- Använd verktyg som **Dependabot** eller **Renovate** för automatiserade pull requests
- **Uppgraderingar av major-versioner** (React, Vite, Tailwind CSS) bör testas lokalt innan de slås samman till `main`

### Hur man vet när uppgraderingar behövs

| Källa | Vad att bevaka |
|-------|----------------|
| **GitHub Dependabot** | Aktivera säkerhetsvarningar och versionsuppdateringar på alla fyra repositorier: *Settings → Security → Dependabot* |
| **Ubuntu** | Prenumerera på [Ubuntus säkerhetsmeddelanden](https://ubuntu.com/security/notices) eller aktivera `unattended-upgrades` |
| **Docker** | Följ [Dockers versionsinformation](https://docs.docker.com/engine/release-notes/) |
| **Nginx Proxy Manager** | Kontrollera [NPM:s releases på GitHub](https://github.com/NginxProxyManager/nginx-proxy-manager/releases) |
| **Let's Encrypt-certifikat** | Förnyas automatiskt av Nginx Proxy Manager — kontrollera NPM-dashboarden regelbundet |
| **Hetzner** | Bevaka [Hetzners statussida](https://www.hetzner.com/status) och e-postmeddelanden |

### Rekommenderad månatlig underhållschecklista

- [ ] SSH in på servern som `root`, kör `sudo apt update && sudo apt upgrade -y`
- [ ] Som `deploy`, gå till `/home/deploy/hosting/` och kör `docker compose pull`
- [ ] Kör `docker compose up -d` för att tillämpa eventuella uppdaterade avbildningar
- [ ] Öppna Nginx Proxy Manager (`http://89.167.90.112:81`) — verifiera att alla proxyvärdar är gröna och att SSL-certifikaten är giltiga
- [ ] Kontrollera GitHub Actions-körningar i varje repository — verifiera att alla senaste driftsättningar lyckades
- [ ] Kör `npm audit` lokalt i varje React-repository för att söka efter sårbarheter
- [ ] Granska eventuella öppna Dependabot-PR:er eller säkerhetsvarningar på GitHub

---

## 9. Felsökningsreferens

| Problem | Lösning |
|---------|---------|
| **Webbplatsen visar 502 Bad Gateway** | Containern körs inte — SSH in som `deploy` och kör `docker compose ps` i `/home/deploy/hosting/`. Starta containern om den är stoppad: `docker compose up -d <container-namn>` |
| **Webbplatsen visar gammalt innehåll efter driftsättning** | Starta om containern: `docker compose down <container-namn> && docker compose up -d <container-namn>` |
| **SSL-certifikatet har gått ut** | Öppna Nginx Proxy Manager → fliken *SSL Certificates* → begär certifikatet för den berörda domänen på nytt |
| **GitHub Actions-driftsättning misslyckas** | Kontrollera Actions-loggen i repositoryt. Verifiera att alla tre hemligheter (`SERVER_HOST`, `SERVER_USER`, `SERVER_SSH_KEY`) är korrekt inställda. Kontrollera att SSH-nyckeln inte har gått ut eller roterats. |
| **`permission denied` på docker-kommandon** | Kör `usermod -aG docker deploy` som root, logga sedan ut och in igen som `deploy` |
| **Tom sida efter driftsättning** | Verifiera att `dist/`-mappen innehåller filer: `ls ~/hosting/sites/<webbplatsnamn>/dist/` |
| **Kan inte komma åt Nginx Proxy Manager** | Öppna `http://89.167.90.112:81` i en webbläsare. Om sidan inte kan nås, SSH in och kör `docker compose ps` för att kontrollera om `proxy-manager`-containern körs |

---

## 10. Viktiga länkar och åtkomstpunkter

| Resurs | URL / Plats |
|--------|-------------|
| **Cloudflare Dashboard** | https://dash.cloudflare.com |
| **Nginx Proxy Manager** | http://89.167.90.112:81 |
| **GitHub-repositorier** | https://github.com/Tschiffer46 |
| **Hetzner-serverns IP** | `89.167.90.112` |
| **Docker Compose-fil** | `/home/deploy/hosting/docker-compose.yml` |
| **Webbplatsfiler — agiletransition** | `~/hosting/sites/client-agiletransition/dist/` |
| **Webbplatsfiler — azprofil** | `~/hosting/sites/client-azprofil/dist/` |
| **Webbplatsfiler — azp2b** | `~/hosting/sites/client-azp2b/dist/` |
| **Webbplatsfiler — hemsidor** | `~/hosting/sites/hemsidor/dist/` |

---

## 11. Hur man gör innehållsändringar

### Generellt arbetsflöde (alla webbplatser)

1. Öppna GitHub-repositoryt för den webbplats du vill uppdatera (https://github.com/Tschiffer46).
2. Redigera relevant fil — antingen via GitHub:s webbredigerare eller genom att klona repositoryt lokalt.
3. Spara (commit) och pusha ändringen till `main`-grenen.
4. GitHub Actions bygger och driftsätter automatiskt inom ungefär **2 minuter**.
5. Verifiera ändringen på den live-satta webbplatsen.

### agiletransition.se (vanlig HTML)

Det räcker att redigera `index.html` direkt i repositoryt. Inget byggsteg krävs — filen laddas upp som den är.

### React-webbplatser (azprofil, azP2B, hemsidor)

- **Sidinnehåll** finns vanligtvis i komponentfiler under `src/components/`
- **Text/översättningar** för azprofil finns i översättningsfiler (använda av `react-i18next`)
- **Text** för azP2B finns i `src/i18n.ts`
- Efter redigering, spara och pusha till `main` — GitHub Actions-arbetsflödet hanterar bygget och driftsättningen automatiskt

> ⚠️ **Pusha inte otestade ändringar direkt till `main` för React-webbplatser.** Testa lokalt först med `npm run dev` och pusha sedan. Ett misslyckat bygge gör att driftsättningen avbryts och den live-satta webbplatsen uppdateras inte förrän nästa lyckade push.

---

## 12. Kontaktformulärsbackend

| Webbplats | Lösning för kontaktformulär |
|-----------|-----------------------------|
| **azprofil** | Formspree (formspree.io) — formulärsvar vidarebefordras till konfigurerad e-postadress |
| **hemsidor** | Formspree (formulär-ID: `xreawgqr`) — formulärsvar vidarebefordras till konfigurerad e-postadress |
| **azP2B** | `mailto:`-länkar — ingen formulärbackend |
| **agiletransition.se** | Inget kontaktformulär |

**Formspree** är en tredjepartstjänst. Formulär-ID:n och tillhörande e-postadresser konfigureras i Formspree-dashboarden (formspree.io). Vid överlämning, säkerställ att:

1. Uppgifterna till Formspree-kontot överlämnas till ansvarig part.
2. E-postadressen som tar emot formulärsvar uppdateras till rätt adress.
3. Formulär-ID:n i koden stämmer överens med Formspree-dashboarden (om formulären återskapas måste ID:n i källkoden uppdateras och webbplatsen driftsättas på nytt).
