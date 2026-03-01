# Architecture — SaaS Pointage Biometrique

## Contexte

SaaS de pointage biometrique pour la Guinee-Conakry.
Pointeuse : **[Horus E1-FP](https://zkteco.technology/en/product/horus-e1-fp/)** (reconnaissance faciale + empreinte, 4G natif, batterie 8h, serie FacePro).
La pointeuse communique uniquement via le SDK Windows **zkemkeeper.dll** (composant COM/ActiveX) en UDP sur le port 4370.

Variantes disponibles :
- [Horus E1](https://zkteco.technology/en/product/horus/) — visage uniquement
- [Horus E1-FP](https://zkteco.technology/en/product/horus-e1-fp/) — visage + empreinte ← **modele choisi**
- [Horus E1-RFID](https://zkteco.technology/en/product/horus-e1-rfid/) — visage + badge RFID

Un **PC Windows local** fait le pont (bridge) entre la pointeuse et le cloud.

---

## Vue d'ensemble

```
┌─────────────────────────────────┐
│         CLOUD (Internet)        │
│                                 │
│  ┌───────────┐   ┌───────────┐  │
│  │  Vercel   │   │ Supabase  │  │
│  │ (Next.js) │──>│  (BDD +   │  │
│  │ Dashboard │   │ Realtime) │  │
│  └───────────┘   └─────┬─────┘  │
│                        │        │
└────────────────────────┼────────┘
                         │
                    WebSocket
                    (Realtime)
                         │
┌────────────────────────┼────────┐
│         SITE LOCAL               │
│                        │        │
│              ┌─────────┴──────┐ │
│              │  PC Windows    │ │
│              │  (Python)      │ │
│              │  Bridge Service│ │
│              └─────────┬──────┘ │
│                        │        │
│                   UDP : 4370    │
│                   (zkemkeeper)  │
│                        │        │
│              ┌─────────┴──────┐ │
│              │   Horus E1     │ │
│              │  (Pointeuse)   │ │
│              └────────────────┘ │
│                                 │
└─────────────────────────────────┘
```

---

## Composants

### 1. Dashboard (Vercel / Next.js)

Interface web utilisee par les RH pour :
- Gerer les employes (ajout, suppression)
- Lancer un enrolement (visage, empreinte)
- Consulter les pointages
- Voir le statut de la pointeuse en temps reel

Le dashboard n'a **aucun contact direct** avec la pointeuse.
Il ecrit des **commandes** dans Supabase, que le PC local viendra lire.

### 2. Supabase (BDD + Realtime)

Supabase joue le role central :
- **Base de donnees** : stocke employes, pointages, commandes, statuts
- **Realtime (WebSocket)** : notifie le PC local instantanement quand une nouvelle commande est inseree
- **API REST** : utilisee par le dashboard Vercel pour les operations CRUD

Tables principales :
| Table | Role |
|-------|------|
| `sites` | Infos du site (adresse, IP pointeuse, statut bridge) |
| `employees` | Liste des employes et leur code pointeuse |
| `attendances` | Historique des pointages |
| `device_commands` | File d'attente des commandes a envoyer a la pointeuse |
| `device_status` | Etat actuel de la pointeuse (connectee, nb utilisateurs, etc.) |

### 3. PC Windows (Bridge Python)

Programme Python qui tourne en permanence sur un PC au sein du site.
Il fait la passerelle entre Supabase (cloud) et la pointeuse (reseau local).

Ses responsabilites :
- **Ecouter les commandes** venant de Supabase via Realtime (WebSocket)
- **Executer les commandes** sur la pointeuse via zkemkeeper.dll
- **Remonter les pointages** : quand un employe pointe, le bridge insere le pointage dans Supabase
- **Heartbeat** : signaler regulierement que le bridge est en ligne
- **Reconnexion automatique** : si la pointeuse ou internet se deconnecte, le bridge retente

### 4. Horus E1 (Pointeuse)

La pointeuse est sur le reseau local du site.
Le PC communique avec elle en UDP sur le port 4370 via le SDK zkemkeeper.dll.
La pointeuse ne parle **jamais directement au cloud**.

---

## Flux de fonctionnement

### Flux 1 — Ajouter un employe

```
1. Le RH clique "Ajouter Jean Camara" sur le dashboard
2. Vercel envoie a Supabase :
   - INSERT dans la table employees (code: '001', name: 'Jean Camara')
   - INSERT dans la table device_commands :
     {type: 'add_user', payload: {code: '001', name: 'Jean Camara'}, status: 'pending'}

3. Supabase notifie le PC via Realtime (WebSocket)

4. Le PC recoit la commande, met le status a 'processing'

5. Le PC execute via zkemkeeper.dll :
   a. EnableDevice(1, false)          -- desactive la pointeuse pendant l'operation
   b. SetStrCardNumber(cardNumber)    -- (optionnel) associe un badge RFID
   c. SSR_SetUserInfo(1, '001', 'Jean Camara', '', 0, true)
      - Param 1 : machine number (1 en TCP, ignore)
      - Param 2 : employee_code (BSTR, max ~PIN2Width~ caracteres)
      - Param 3 : nom
      - Param 4 : mot de passe (vide = pas de mot de passe)
      - Param 5 : privilege (0=user, 1=enroller, 2=admin, 3=superadmin)
      - Param 6 : actif (true)
   d. RefreshData(1)                  -- force la pointeuse a recharger ses donnees
   e. EnableDevice(1, true)           -- reactive la pointeuse

6. zkemkeeper envoie la commande en UDP a la pointeuse
7. La pointeuse confirme (retour true)
8. Le PC met le status a 'completed' dans Supabase
9. Le dashboard affiche "Employe ajoute avec succes"
```

### Flux 2 — Enrolement visage

Le Horus E1 est un appareil **FacePro** (reconnaissance faciale en lumiere visible).
L'enrolement se fait **directement sur la pointeuse** : l'employe se place devant la camera.

```
1. Le RH clique "Enroler le visage" pour Jean Camara
2. Vercel insere dans device_commands :
   {type: 'enroll_face', payload: {code: '001'}, status: 'pending'}

3. Le PC recoit la commande via Realtime

4. Le PC execute via zkemkeeper.dll :
   a. CancelOperation()              -- annule toute operation en cours
   b. L'enrolement facial se fait physiquement sur la pointeuse
      (l'employe se place devant la camera, la pointeuse capture le visage)

   Note : pour les templates face (IFACE), le SDK utilise :
   - SetUserFace / SetUserFaceStr avec dwFaceIndex = 50 (valeur fixe obligatoire)
   - Chaque utilisateur a ~15 templates sous differents angles (~37 Ko total)

   Pour les FacePro (visible light), l'alternative est l'envoi de photo :
   - Format fichier : verify_biophoto_9_{user_id}.jpg
   - Via SendUserFacePhoto(machineNumber, filePath)

5. La pointeuse affiche "Placez votre visage devant la camera"
6. L'employe presente son visage
7. La pointeuse enregistre le visage et confirme
   - Evenement OnEnrollFingerEx declenche (malgre le nom, couvre aussi le visage)
   - ActionResult = 0 : succes, autres valeurs : echec
8. Le PC met status = 'completed' et met a jour enrolled = true dans employees
```

### Flux 3 — Enrolement empreinte

```
1. Le RH clique "Enroler empreinte" pour Jean Camara (doigt index droit)
2. Vercel insere dans device_commands :
   {type: 'enroll_finger', payload: {code: '001', finger_index: 1}, status: 'pending'}

3. Le PC recoit la commande via Realtime

4. Le PC execute via zkemkeeper.dll :
   a. CancelOperation()
   b. SSR_DelUserTmpExt(1, '001', 1)  -- supprime le template existant si besoin
   c. StartEnrollEx('001', 1, 1)
      - Param 1 : employee_code
      - Param 2 : finger_index (0-9, un par doigt)
      - Param 3 : flag (0=invalide, 1=valide, 3=empreinte de contrainte)

5. La pointeuse entre en mode enrolement, l'employe scanne 3 fois son doigt

6. Evenement OnEnrollFingerEx declenche :
   - sEnrollNumber : code de l'employe
   - iFingerIndex : index du doigt
   - iActionResult : 0=succes
   - iTemplateLength : taille du template

7. Le PC appelle StartIdentify() pour remettre la pointeuse en mode normal
8. Le PC met status = 'completed' dans Supabase
```

### Flux 4 — Un employe pointe (quotidien)

```
1. L'employe se presente devant la pointeuse

2. La pointeuse reconnait son visage (ou empreinte, ou badge)

3. La pointeuse declenche l'evenement OnAttTransactionEx avec ces parametres :
   - EnrollNumber (BSTR) : code de l'employe (ex: '001')
   - IsInValid (LONG) : 0=valide, 1=invalide
   - AttState (LONG) : statut du pointage
       0 = Check-In (entree)
       1 = Check-Out (sortie)
       2 = Break-Out (debut pause)
       3 = Break-In (fin pause)
       4 = OT-In (debut heures sup)
       5 = OT-Out (fin heures sup)
   - VerifyMethod (LONG) : methode de verification
       0 = mot de passe
       1 = empreinte digitale
       15 = visage (face)
       2 = carte RFID
   - Year, Month, Day, Hour, Minute, Second : horodatage
   - WorkCode (LONG) : code travail (0 si non utilise)

4. Le PC capture l'evenement via zkemkeeper.dll

5. Le PC mappe les valeurs et insere dans Supabase :
   - Table attendances :
     employee_code = '001'
     punch_time = '2026-03-01 08:02:00'
     verify_type = 'face'         (VerifyMethod 15 → 'face')
     punch_type = 'IN'            (AttState 0 → 'IN')

6. Le dashboard affiche le pointage en temps reel
```

**Mapping VerifyMethod → verify_type :**
| VerifyMethod | verify_type |
|-------------|-------------|
| 0 | password |
| 1 | fingerprint |
| 2 | card |
| 15 | face |

**Mapping AttState → punch_type :**
| AttState | punch_type |
|---------|------------|
| 0 | IN |
| 1 | OUT |
| 2 | BREAK_OUT |
| 3 | BREAK_IN |
| 4 | OT_IN |
| 5 | OT_OUT |

### Flux 5 — Supprimer un employe

```
1. Le RH clique "Supprimer" sur le dashboard
2. Vercel insere dans device_commands :
   {type: 'delete_user', payload: {code: '001'}, status: 'pending'}

3. Le PC recoit la commande via Realtime

4. Le PC execute via zkemkeeper.dll :
   a. EnableDevice(1, false)
   b. SSR_DeleteEnrollData(1, '001', 12)
      - dwBackupNumber = 12 : supprime l'utilisateur complet
        (empreintes + carte + mot de passe + visage)
      Variantes :
        0-9 = supprimer une empreinte specifique (index du doigt)
        10 = supprimer le mot de passe uniquement
        11 = supprimer toutes les empreintes
        12 = supprimer l'utilisateur complet
        13 = supprimer toutes les empreintes (version optimisee, SSR_DeleteEnrollDataExt)
   c. RefreshData(1)
   d. EnableDevice(1, true)

5. La pointeuse supprime l'employe et ses donnees biometriques
6. Le PC met status = 'completed'
7. Vercel supprime l'employe de la table employees
```

### Flux 6 — Heartbeat (surveillance continue)

```
Toutes les 30 secondes :

1. Le PC interroge la pointeuse via zkemkeeper.dll :
   - GetDeviceStatus(1, 2, ref value)   → nombre d'utilisateurs  (dwStatus=2)
   - GetDeviceStatus(1, 21, ref value)  → nombre de visages      (dwStatus=21)
   - GetDeviceStatus(1, 3, ref value)   → nombre d'empreintes    (dwStatus=3)
   - GetSerialNumber(1, ref sn)         → numero de serie

   Autres infos disponibles (dwStatus) :
     1 = nombre d'administrateurs
     4 = nombre de mots de passe
     6 = nombre d'enregistrements de pointage
     7 = capacite max empreintes
     8 = capacite max utilisateurs
     22 = capacite max visages

2. Le PC met a jour dans Supabase :
   - Table device_status :
     {is_connected: true, user_count: 45, face_count: 42, fp_count: 30, last_ping: now}
   - Table sites :
     {bridge_status: 'online', bridge_last_seen: now}

3. Le dashboard affiche l'etat en temps reel

4. Si la pointeuse ne repond pas :
   - Le PC met is_connected = false
   - Le PC tente une reconnexion : Disconnect() puis Connect_Net(IP, 4370)
```

---

## Detail technique du SDK zkemkeeper.dll

### Installation prealable sur le PC

Le SDK est un composant **COM (ActiveX)**. Il doit etre enregistre sur Windows avant utilisation :

1. Copier les DLL du SDK dans `%windir%\SysWOW64` (Windows 64-bit)
2. Ouvrir un terminal administrateur
3. Executer : `regsvr32.exe %windir%\SysWOW64\zkemkeeper.dll`

Alternative : executer le script `Register_SDK.bat` fourni avec le SDK en mode administrateur.

**Version du SDK :** 6.3.1.55 (la plus recente, supporte les firmwares avec cle COM securisee)

### Acces depuis Python

Le bridge Python accede au SDK via COM :
```
win32com.client.Dispatch("zkemkeeper.ZKEM.1")
```
Ou via comtypes comme alternative.

### Connexion a la pointeuse

```
Connect_Net(IP, Port) → bool
- IP : adresse IP de la pointeuse sur le reseau local
- Port : 4370 (defaut, UDP)
- Retour : true si connexion reussie

Disconnect() → void
- Deconnecte et libere les ressources
```

Si un mot de passe COM est configure sur la pointeuse :
```
SetCommPasswordEx(commKey)    -- definir le mot de passe cote PC (BSTR, alphanumerique)
Connect_Net(IP, Port)         -- se connecter ensuite
```

### Ecoute des evenements temps reel

Le bridge doit s'abonner aux evenements pour capturer les pointages :

```
RegEvent(machineNumber, 65535)    -- 65535 = tous les evenements
```

Masques d'evenements disponibles :
| Masque | Evenement |
|--------|-----------|
| 1 | OnAttTransactionEx (pointage) |
| 2 | OnFinger (doigt detecte) |
| 4 | OnNewUser (nouvel utilisateur) |
| 8 | OnEnrollFingerEx (enrolement termine) |
| 16 | OnKeyPress (touche pressee) |
| 256 | OnVerify (verification en cours) |
| 512 | OnFingerFeature (qualite empreinte) |
| 1024 | OnDoor / OnAlarm |
| 65535 | Tous les evenements |

### Evenement principal : OnAttTransactionEx

Declenche a chaque pointage reussi :
```
OnAttTransactionEx(
    EnrollNumber,    -- BSTR : code employe
    IsInValid,       -- LONG : 0=valide
    AttState,        -- LONG : 0=IN, 1=OUT, 2=BREAK_OUT, 3=BREAK_IN
    VerifyMethod,    -- LONG : 0=password, 1=fingerprint, 2=card, 15=face
    Year, Month, Day, Hour, Minute, Second,
    WorkCode         -- LONG : code travail
)
```

### Pattern obligatoire pour les operations sur la pointeuse

Avant toute operation critique (ajout/suppression d'utilisateurs, templates) :

```
1. EnableDevice(machineNumber, false)    -- desactive la pointeuse (bloque empreinte/clavier/carte)
2. ... effectuer l'operation ...
3. RefreshData(machineNumber)            -- force le rechargement des donnees
4. EnableDevice(machineNumber, true)     -- reactive la pointeuse
```

### Codes d'erreur principaux (GetLastError)

| Code | Description |
|------|-------------|
| 0 | Succes / pas de donnees |
| -1 | SDK non initialise, reconnecter |
| -2 | Erreur lecture/ecriture |
| -4 | Espace insuffisant |
| -5 | Donnees deja existantes |
| -6 | Mauvais mot de passe COM |
| -100 | Non supporte ou donnees inexistantes |
| -201 | Device occupe (busy) |
| -307 | Timeout de connexion |

---

## Gestion des pannes

| Situation | Comportement |
|-----------|-------------|
| **Pointeuse eteinte / deconnectee** | Le bridge detecte la perte de connexion au prochain heartbeat. Il passe `device_status.is_connected = false`. Les commandes restent en `pending` et seront executees quand la pointeuse revient. Le bridge tente une reconnexion toutes les 30 secondes. |
| **PC perd internet** | Les pointages sont captures localement mais ne peuvent pas etre envoyes. A la reconnexion, le bridge rattrape les evenements et les pousse vers Supabase. Les commandes en attente dans Supabase seront recuperees a la reconnexion. |
| **Commande echoue (timeout > 60s)** | Le bridge met `status = 'failed'` avec le detail de l'erreur dans le champ `result` (code erreur SDK inclus). Le dashboard affiche l'erreur au RH. |
| **PC eteint** | Le champ `bridge_last_seen` n'est plus mis a jour. Le dashboard detecte que le bridge est offline apres quelques minutes sans heartbeat. |
| **SDK crash (code erreur -1)** | Le bridge capture l'exception, log l'erreur, appelle `Disconnect()` puis retente `Connect_Net()`. |
| **Device occupe (code erreur -201)** | Le bridge attend quelques secondes et retente l'operation. La pointeuse est temporairement occupee (enrolement en cours, menu ouvert, etc.). |
| **Mauvais mot de passe COM (code -6)** | Le bridge log l'erreur. Il faut verifier que `SetCommPasswordEx` est appele avec le bon mot de passe avant `Connect_Net`. |

---

## Stack technique

| Couche | Technologie |
|--------|-------------|
| Dashboard | Next.js (Vercel) |
| Base de donnees | Supabase (PostgreSQL) |
| Communication cloud ↔ PC | Supabase Realtime (WebSocket) |
| Bridge (PC local) | Python + win32com / comtypes |
| SDK pointeuse | zkemkeeper.dll v6.3.1.55 (COM/ActiveX) |
| Protocole pointeuse | UDP port 4370 |
| Pointeuse | Horus E1 (serie FacePro, visible light face) |

---

## Schema de la base de donnees

### sites
| Colonne | Type | Description |
|---------|------|-------------|
| id | UUID | Identifiant unique |
| name | VARCHAR(100) | Nom du site |
| address | TEXT | Adresse physique |
| device_ip | VARCHAR(15) | IP de la pointeuse sur le reseau local |
| device_port | INTEGER | Port UDP (defaut: 4370) |
| device_sn | VARCHAR(50) | Numero de serie de la pointeuse |
| bridge_status | VARCHAR(20) | 'online' ou 'offline' |
| bridge_last_seen | TIMESTAMP | Dernier signe de vie du bridge |

### employees
| Colonne | Type | Description |
|---------|------|-------------|
| id | UUID | Identifiant unique |
| employee_code | VARCHAR(20) | Code utilise sur la pointeuse (unique) |
| name | VARCHAR(100) | Nom complet |
| site_id | UUID | Site de rattachement |
| enrolled | BOOLEAN | Biometrie enregistree ou non |
| enrolled_at | TIMESTAMP | Date d'enrolement |

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

### device_commands
| Colonne | Type | Description |
|---------|------|-------------|
| id | UUID | Identifiant unique |
| site_id | UUID | Site cible |
| command_type | VARCHAR(30) | 'add_user', 'delete_user', 'enroll_face', 'enroll_finger', 'restart_device' |
| payload | JSONB | Donnees de la commande |
| status | VARCHAR(20) | 'pending', 'processing', 'completed', 'failed' |
| result | JSONB | Resultat de l'execution (inclut le code erreur SDK si echec) |
| processed_at | TIMESTAMP | Date de traitement |

### device_status
| Colonne | Type | Description |
|---------|------|-------------|
| id | UUID | Identifiant unique |
| site_id | UUID | Site (unique) |
| is_connected | BOOLEAN | Pointeuse joignable ou non |
| user_count | INTEGER | Nombre d'utilisateurs enregistres (GetDeviceStatus dwStatus=2) |
| face_count | INTEGER | Nombre de visages enregistres (GetDeviceStatus dwStatus=21) |
| fp_count | INTEGER | Nombre d'empreintes enregistrees (GetDeviceStatus dwStatus=3) |
| last_ping | TIMESTAMP | Dernier ping reussi |

---

## Reference rapide des fonctions SDK utilisees par le bridge

| Operation | Fonction SDK | Parametres cles |
|-----------|-------------|----------------|
| Connexion | `Connect_Net(IP, Port)` | IP pointeuse, port 4370 |
| Deconnexion | `Disconnect()` | -- |
| Mot de passe COM | `SetCommPasswordEx(key)` | Appeler avant Connect_Net |
| Ajouter utilisateur | `SSR_SetUserInfo(machNo, code, name, pwd, privilege, enabled)` | privilege: 0=user |
| Supprimer utilisateur | `SSR_DeleteEnrollData(machNo, code, 12)` | 12=suppression complete |
| Enrolement empreinte | `StartEnrollEx(code, fingerIndex, flag)` | flag: 1=valide |
| Fin enrolement | `StartIdentify()` | Remet en mode normal |
| Annuler operation | `CancelOperation()` | -- |
| Template visage get | `GetUserFaceStr(machNo, code, 50, data, length)` | index 50 obligatoire |
| Template visage set | `SetUserFaceStr(machNo, code, 50, data, length)` | index 50 obligatoire |
| Photo visage (FacePro) | `SendUserFacePhoto(machNo, filePath)` | fichier: verify_biophoto_9_{id}.jpg |
| Ecouter evenements | `RegEvent(machNo, 65535)` | 65535 = tous |
| Statut device | `GetDeviceStatus(machNo, dwStatus, ref value)` | dwStatus: 2=users, 3=fp, 21=faces |
| Numero de serie | `GetSerialNumber(machNo, ref sn)` | -- |
| Bloquer pointeuse | `EnableDevice(machNo, false)` | Avant operation critique |
| Debloquer pointeuse | `EnableDevice(machNo, true)` | Apres operation |
| Rafraichir donnees | `RefreshData(machNo)` | Apres modification |
| Redemarrer | `RestartDevice(machNo)` | -- |
| Eteindre | `PowerOffDevice(machNo)` | -- |
| Derniere erreur | `GetLastError(ref errorCode)` | Apres chaque echec |
