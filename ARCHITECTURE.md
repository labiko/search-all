# Architecture — SaaS Pointage Biometrique

## Contexte

SaaS de pointage biometrique pour la Guinee-Conakry.
Pointeuse : **[Horus E1-FP](https://zkteco.technology/en/product/horus-e1-fp/)** (reconnaissance faciale + empreinte, 4G natif avec Nano SIM, batterie 8h, serie FacePro).

Variantes disponibles :
- [Horus E1](https://zkteco.technology/en/product/horus/) — visage uniquement
- [Horus E1-FP](https://zkteco.technology/en/product/horus-e1-fp/) — visage + empreinte ← **modele choisi**
- [Horus E1-RFID](https://zkteco.technology/en/product/horus-e1-rfid/) — visage + badge RFID

La pointeuse communique **directement avec le cloud** via 4G grace au **protocole PUSH (Attendance PUSH Communication Protocol)**.
**Pas de PC local necessaire.** Le device est le client HTTP — il fait des requetes vers notre serveur.

---

## Choix technique : TA PUSH SDK

ZKTeco propose trois options pour integrer le Horus E1-FP :

| | SDK Standalone (zkemkeeper.dll) | UTimeMaster | TA PUSH SDK |
|---|---|---|---|
| **Principe** | PC local communique avec la pointeuse via UDP:4370 | Logiciel web ZKTeco avec API REST | La pointeuse pousse les donnees directement vers notre serveur via HTTP/4G |
| **PC bridge** | ✅ Obligatoire | Selon config | ❌ **Pas necessaire** |
| **4G natif** | Non utilise (LAN uniquement) | Selon config | ✅ **Oui** |
| **Cout** | $400 | Licence par device | $400 |
| **Integration cloud** | Impossible directement | Via serveur ZKTeco | ✅ **Directe (HTTP)** |

**Decision : TA PUSH SDK ($400)** — pour les raisons suivantes :
1. **Pas de PC bridge** : la pointeuse parle directement au cloud via 4G
2. **Architecture simplifiee** : moins de composants = moins de pannes
3. **4G natif** : la pointeuse utilise sa carte SIM pour communiquer, pas besoin de WiFi/LAN
4. **Integration directe** avec Vercel (Next.js) via HTTP POST
5. **Ideal pour la Guinee** : fonctionne partout ou il y a du reseau mobile

---

## Vue d'ensemble

```
┌──────────────────────────────────────┐
│            CLOUD (Internet)          │
│                                      │
│  ┌───────────┐     ┌──────────────┐  │
│  │  Vercel   │     │  Supabase    │  │
│  │ (Next.js) │────>│  (BDD)       │  │
│  │ Dashboard │     │              │  │
│  │           │     │              │  │
│  │ /iclock/cdata    <── INSERT     │  │
│  │ /iclock/getrequest ── SELECT   │  │
│  │ /iclock/devicecmd <── UPDATE   │  │
│  └───────────┘     └──────────────┘  │
│        ^                             │
│        │                             │
└────────┼─────────────────────────────┘
         │
    HTTP GET/POST (texte tabule)
    via 4G (Nano SIM)
         │
┌────────┼─────────────────────────────┐
│  SITE  │                             │
│        │                             │
│  ┌─────┴──────────┐                  │
│  │   Horus E1-FP  │                  │
│  │   (Pointeuse)  │                  │
│  │                │                  │
│  │  4G Nano SIM   │                  │
│  │  Batterie 8h   │                  │
│  │  Face + FP     │                  │
│  └────────────────┘                  │
│                                      │
│  Pas de PC. Pas de WiFi.             │
│  Juste la pointeuse + une SIM 4G.   │
│                                      │
└──────────────────────────────────────┘
```

---

## Composants

### 1. Dashboard (Vercel / Next.js)

Interface web utilisee par les RH pour :
- Gerer les employes (ajout, suppression)
- Lancer un enrolement (visage, empreinte)
- Consulter les pointages
- Voir le statut de la pointeuse en temps reel

Le dashboard expose des **API Routes** compatibles avec le protocole PUSH ZKTeco :
- `GET /iclock/cdata?SN=xxx` — initialisation du device (echange de configuration)
- `POST /iclock/cdata?SN=xxx&Stamp=xxx` — reception des pointages et donnees utilisateurs
- `GET /iclock/getrequest?SN=xxx` — le device poll pour recuperer les commandes en attente
- `POST /iclock/devicecmd?SN=xxx` — le device confirme l'execution d'une commande

**Important :** Le device est toujours le client HTTP. Le serveur ne contacte jamais le device directement.
Les commandes sont stockees en BDD (table `device_commands`) et envoyees quand le device poll.

### 2. Supabase (BDD)

Base de donnees PostgreSQL :
- Stocke employes, pointages, statuts
- API REST pour le dashboard
- Realtime pour les mises a jour en temps reel du dashboard

Tables principales :
| Table | Role |
|-------|------|
| `sites` | Infos du site (adresse, SN pointeuse, statut) |
| `employees` | Liste des employes et leur code pointeuse |
| `attendances` | Historique des pointages |
| `device_commands` | File d'attente des commandes a envoyer au device (polling) |
| `device_status` | Etat actuel de la pointeuse |

**Note :** La table `device_commands` est necessaire — le protocole PUSH fonctionne par polling.
Le device appelle `GET /iclock/getrequest` a intervalle regulier pour recuperer les commandes en attente.

### 3. Horus E1-FP (Pointeuse)

La pointeuse communique **directement avec Vercel** via 4G :
- **Push** : envoie les pointages en temps reel vers notre API (HTTP POST)
- **Pull** : recoit les commandes (ajout/suppression users) depuis notre API
- Fonctionne de maniere **autonome** avec une Nano SIM 4G
- Batterie 8h pour les coupures d'electricite
- Stocke les pointages localement si le reseau est indisponible, et les renvoie a la reconnexion

---

## Flux de fonctionnement

### Flux 1 — Ajouter un employe

```
1. Le RH clique "Ajouter Jean Camara" sur le dashboard

2. Vercel :
   a. INSERT dans Supabase table employees (code: '001', name: 'Jean Camara')
   b. INSERT dans Supabase table device_commands :
      {type: 'add_user', payload: {PIN: '001', Name: 'Jean Camara'}, status: 'pending'}

3. La pointeuse poll GET /iclock/getrequest?SN=xxx
   → Vercel repond avec : C:123:DATA UPDATE USERINFO PIN=001\tName=Jean Camara

4. La pointeuse execute la commande et confirme via POST /iclock/devicecmd

5. Vercel met a jour device_commands.status = 'completed'

6. Le dashboard affiche "Employe ajoute avec succes"
```

**Note :** Le delai depend de l'intervalle de polling du device (a configurer).

### Flux 2 — Enrolement visage

```
1. Le RH clique "Enroler le visage" pour Jean Camara

2. Vercel INSERT device_commands : {type: 'enroll_face', payload: {PIN: '001'}}

3. La pointeuse poll et recoit la commande d'enrolement

4. La pointeuse affiche "Placez votre visage devant la camera"

5. L'employe se presente devant la pointeuse

6. La pointeuse capture le visage et enregistre le template

7. La pointeuse confirme via POST /iclock/devicecmd → Vercel met a jour enrolled = true
```

### Flux 3 — Un employe pointe (quotidien)

```
1. L'employe se presente devant la pointeuse

2. La pointeuse reconnait son visage (ou empreinte)

3. La pointeuse envoie un HTTP POST vers Vercel POST /iclock/cdata?SN=xxx&Stamp=26
   Corps (texte tabule, pas JSON) :
   001\t2026-03-05 08:02:00\t0\t15\t0\t1

   Champs : PIN, Time, Status, Verify (15=face), Workcode, Reserved

4. Vercel parse le texte tabule et insere dans Supabase (table attendances)

5. Vercel repond "OK"

6. Le dashboard affiche le pointage en temps reel
```

### Flux 4 — Supprimer un employe

```
1. Le RH clique "Supprimer" sur le dashboard

2. Vercel INSERT device_commands : {type: 'delete_user', payload: {PIN: '001'}}

3. La pointeuse poll et recoit la commande de suppression

4. La pointeuse supprime l'employe et ses donnees biometriques

5. La pointeuse confirme via POST /iclock/devicecmd

6. Vercel supprime l'employe de Supabase
```

### Flux 5 — Pointeuse hors-ligne (coupure reseau)

```
1. La pointeuse perd le reseau 4G (coupure, zone blanche)

2. Les employes continuent de pointer normalement
   → les pointages sont stockes localement sur la pointeuse

3. Le reseau revient

4. La pointeuse reprend la transmission depuis le point d'arret (mecanisme Stamp)
   POST /iclock/cdata?SN=xxx&Stamp=dernierStamp

5. Vercel insere les pointages dans Supabase (avec deduplication via Stamp)

6. Le dashboard se met a jour
```

---

## Gestion des pannes

| Situation | Comportement |
|-----------|-------------|
| **Coupure 4G** | Les pointages sont stockes localement sur la pointeuse. Renvoi automatique a la reconnexion. |
| **Coupure electricite** | La pointeuse a 8h de batterie. Elle continue de fonctionner et de pointer. |
| **Pointeuse eteinte** | Le dashboard detecte l'absence de heartbeat. Les commandes en attente seront traitees au redemarrage. |
| **Serveur Vercel indisponible** | La pointeuse stocke les pointages et retente. Vercel a un uptime de 99.99%. |
| **Doublon de pointage** | Le backend Vercel deduplique (meme employee_code + meme timestamp = ignore). |

---

## Stack technique

| Couche | Technologie |
|--------|-------------|
| Dashboard | Next.js (Vercel) |
| Base de donnees | Supabase (PostgreSQL) |
| Protocole pointeuse ↔ cloud | **Attendance PUSH Communication Protocol v4.5 (HTTP GET/POST via 4G)** |
| Format des donnees | Texte tabule (\\t = separateur champs, \\n = separateur lignes) |
| Pointeuse | Horus E1-FP (Nano SIM 4G, batterie 8h) |

**Composants supprimes** (par rapport a l'ancienne architecture) :
- ~~PC Windows local~~
- ~~Bridge Python~~
- ~~zkemkeeper.dll (Standalone SDK)~~
- ~~Supabase Realtime (WebSocket vers PC)~~
- ~~Table device_commands~~ → **retablie** (necessaire pour le mecanisme de polling)

---

## Schema de la base de donnees

### sites
| Colonne | Type | Description |
|---------|------|-------------|
| id | UUID | Identifiant unique |
| name | VARCHAR(100) | Nom du site |
| address | TEXT | Adresse physique |
| device_sn | VARCHAR(50) | Numero de serie de la pointeuse |
| device_status | VARCHAR(20) | 'online' ou 'offline' |
| last_heartbeat | TIMESTAMP | Dernier signe de vie de la pointeuse |
| created_at | TIMESTAMP | Date de creation |

### employees
| Colonne | Type | Description |
|---------|------|-------------|
| id | UUID | Identifiant unique |
| employee_code | VARCHAR(20) | Code utilise sur la pointeuse (unique) |
| name | VARCHAR(100) | Nom complet |
| site_id | UUID | Site de rattachement |
| enrolled | BOOLEAN | Biometrie enregistree ou non |
| enrolled_at | TIMESTAMP | Date d'enrolement |
| created_at | TIMESTAMP | Date de creation |

### attendances
| Colonne | Type | Description |
|---------|------|-------------|
| id | UUID | Identifiant unique |
| employee_code | VARCHAR(20) | Code de l'employe |
| site_id | UUID | Site du pointage |
| punch_time | TIMESTAMP | Heure du pointage |
| punch_type | VARCHAR(10) | 'IN', 'OUT', 'BREAK_OUT', 'BREAK_IN', 'OT_IN', 'OT_OUT' |
| verify_type | VARCHAR(20) | 'face', 'fingerprint', 'card', 'password' |
| device_sn | VARCHAR(50) | Numero de serie de la pointeuse |
| created_at | TIMESTAMP | Date d'insertion |

### device_commands (file de commandes — polling)
| Colonne | Type | Description |
|---------|------|-------------|
| id | UUID | Identifiant unique |
| site_id | UUID | Site concerne |
| command_type | VARCHAR(30) | 'add_user', 'delete_user', 'enroll_face', 'enroll_finger', 'restart' |
| payload | JSONB | Donnees de la commande (ex: {PIN: '001', Name: 'Jean'}) |
| status | VARCHAR(20) | 'pending', 'sent', 'completed', 'failed' |
| result | JSONB | Resultat retourne par le device |
| created_at | TIMESTAMP | Date de creation |
| sent_at | TIMESTAMP | Date d'envoi au device (via polling) |
| completed_at | TIMESTAMP | Date de confirmation par le device |

### device_status
| Colonne | Type | Description |
|---------|------|-------------|
| id | UUID | Identifiant unique |
| site_id | UUID | Site (unique) |
| is_connected | BOOLEAN | Pointeuse joignable ou non |
| user_count | INTEGER | Nombre d'utilisateurs enregistres |
| face_count | INTEGER | Nombre de visages enregistres |
| fp_count | INTEGER | Nombre d'empreintes enregistrees |
| last_ping | TIMESTAMP | Dernier ping reussi |

---

## Alternative evaluee : CAMS Biometrics

Nous avons aussi evalue CAMS Biometrics (modele Hawking Plus f38+) comme alternative :

| Critere | ZKTeco Horus E1-FP | CAMS f38+ |
|---------|-------------------|-----------|
| **4G natif** | ✅ Nano SIM | ❌ Routeur WiFi externe |
| **Batterie** | ✅ 8h | ❌ Non |
| **API directe cloud** | ✅ TA PUSH SDK | ✅ Web API 3.0 |
| **PC bridge** | ❌ Pas besoin | ❌ Pas besoin |
| **Prix device** | ~$359 (Microless Global) | 220 USD |
| **Prix API/SDK** | $400 | 120-300 USD |
| **Portabilite** | ✅ Portable | ❌ Fixe |
| **Livraison** | En attente | 3-4 jours DHL |

**Decision : Horus E1-FP** — le 4G natif et la batterie 8h sont decisifs pour la Guinee.
CAMS reste une **option de secours** si ZKTeco ne repond pas ou si le prix est trop eleve.

---

## Historique des decisions

| Date | Decision | Raison |
|------|----------|--------|
| 2026-03-01 | Choix initial : SDK Standalone + PC bridge | Architecture classique ZKTeco |
| 2026-03-02 | Evaluation UTimeMaster | Reponse ZKTeco proposant 2 options |
| 2026-03-02 | Rejet UTimeMaster | Vendor lock-in, polling uniquement, cout recurrent |
| 2026-03-03 | Evaluation CAMS Biometrics | Alternative avec Web API directe |
| 2026-03-03 | **Pivot vers TA PUSH SDK** | ZKTeco confirme que le Horus E1-FP supporte le push direct via 4G. Plus besoin de PC bridge. |
| 2026-03-03 | **Architecture finale : Horus E1-FP + TA PUSH SDK + Vercel + Supabase** | Architecture simplifiee, 4G natif, pas de PC, pas de middleware |
| 2026-03-04 | Evaluation Hikvision DS-K1T341CMF | ISAPI = LAN only, ISUP = middleware proprietaire. Incompatible cloud direct. **Rejete.** |
| 2026-03-04 | Re-evaluation CAMS Biometrics | Cloud CAMS obligatoire comme intermediaire. Vendor lock-in + licence annuelle. **Rejete.** |
| 2026-03-04 | Re-evaluation Anviz CrossChex Cloud | Pas d'API gestion users. Webhooks non fiables. **Rejete.** |
| 2026-03-04 | **Decision : achat device Microless Global + SDK direct ZKTeco** | Device ~$359 sur Microless + SDK TA PUSH direct aupres de ZKTeco. Mail envoye a sales@zkteco.com. |

---

## Achat du device — Lien direct

**Microless Global** (Dubai, livraison mondiale 4-8 jours) :
[https://global.microless.com/product/zkteco-horus-e1-fp-wireless-time-attendance-terminal-facial-recognition-face-finger-card-3-meter-range-scanning-supports-4g-wifi-bluetooth-gps-mask-detection-horus-e1-fp/](https://global.microless.com/product/zkteco-horus-e1-fp-wireless-time-attendance-terminal-facial-recognition-face-finger-card-3-meter-range-scanning-supports-4g-wifi-bluetooth-gps-mask-detection-horus-e1-fp/)

**Prix :** $358.90 USD (~€330) + frais d'import/TVA au checkout

---

## Alternatives evaluees et rejetees

### Hikvision DS-K1T341CMF (~$247)
- Face + Empreinte + Badge, WiFi/Ethernet
- **Rejete** : ISAPI est un protocole LAN, pas cloud. ISUP necessite un serveur middleware proprietaire (C/C++). Incompatible avec Vercel serverless.

### CAMS Biometrics Hawking Plus f38+ (~$220 + $120-300 API)
- Face + Empreinte, WiFi
- **Rejete** : Le cloud CAMS est un intermediaire obligatoire (Device → CAMS Cloud → Notre serveur). Licence annuelle $60-120/an. Latence 15-60s pour les commandes. Pas de 4G natif.

### Anviz CrossChex Cloud (FaceDeep 3, W2 Face)
- Face + RFID (FD3) ou Face + Empreinte (W2)
- **Rejete** : Pas d'API pour gerer les users (lecture seule). Webhooks non fiables. Cloud Anviz obligatoire. $99/an par device.

**Conclusion : Le ZKTeco Horus E1-FP avec TA PUSH SDK reste la seule option qui permet un push HTTP direct vers notre serveur sans intermediaire.**

---

## Protocole PUSH — Details techniques

### Document de reference
- **Titre :** Attendance PUSH Communication Protocol
- **Version logiciel :** 2.4.2
- **Version document :** 4.5 (juillet 2022)
- **Pages :** 158 (on a l'intro de 18 pages, document complet apres signature NDA)

### Endpoints du protocole (cotes serveur — nos API Routes)

| Route | Methode | Role |
|-------|---------|------|
| `/iclock/cdata?SN=xxx&options=all` | GET | Initialisation — echange de config (Stamp, options) |
| `/iclock/cdata?SN=xxx&Stamp=xxx` | POST | Reception des pointages et donnees (texte tabule) |
| `/iclock/getrequest?SN=xxx` | GET | Le device poll pour recuperer les commandes en attente |
| `/iclock/devicecmd?SN=xxx` | POST | Le device confirme l'execution d'une commande |

### Format des commandes (serveur → device via polling)
```
C:{CmdID}:{CmdDesc}
Exemple : C:123:DATA UPDATE USERINFO PIN=001\tName=Jean Camara
```

### Format des pointages (device → serveur via POST)
```
Texte tabule : PIN\tTime\tStatus\tVerify\tWorkcode\tReserved
Exemple : 001\t2026-03-05 08:02:00\t0\t15\t0\t1
```

Verify types : 1=empreinte, 15=visage

### Mecanisme de reprise (Stamp)
- Le serveur renvoie un `Stamp` a chaque echange
- Le device stocke ce Stamp et reprend la transmission depuis ce point si deconnexion
- Garantit zero perte de donnees meme en cas de coupure reseau

### Commandes disponibles (Section 12 du protocole complet)
- **Gestion users** : ajout, modification, suppression
- **Templates biometriques** : empreinte, visage, veine, paume
- **Enrolement a distance** : empreinte, carte, visage
- **Photos utilisateur** : upload/download
- **Controle device** : reboot, deverrouillage porte, annulation alarme
- **Mise a jour firmware** : a distance
- **Configuration** : options du device

### Points a verifier (reunion Jessica — lundi 10 mars)
- Support HTTPS (Vercel ne sert qu'en HTTPS)
- Support nom de domaine (pas seulement IP)
- Intervalle de polling configurable (delai de livraison des commandes)

---

## Documentation SDK de reference

- SDK Standalone (zkemkeeper.dll) : inclus dans le ZIP du repo (reference uniquement, **non utilise**)
- **Attendance PUSH Communication Protocol v4.5** : intro recue de ZKTeco Europe, document complet apres NDA
- **Contact ZKTeco Europe** : Jessica Perret — jessica.perret@zkteco.eu — +34 650 082 720
- CAMS Web API 3.0 : [camsbiometrics.com/application/biometric-web-api.html](https://camsbiometrics.com/application/biometric-web-api.html) (reference alternative, **rejete**)
