# Questions pour Jessica Perret — Reunion lundi 10 mars 13h

**Interlocutrice :** Jessica Perret, Business Development Manager, ZKTeco Europe
**Contact :** jessica.perret@zkteco.eu | +34 650 082 720
**Format :** Visio Teams

---

## Contexte a rappeler en debut de reunion

- On developpe un **SaaS de pointage biometrique** pour la Guinee-Conakry
- Architecture : **Horus E1-FP (4G) → HTTP → Vercel/Next.js → Supabase**
- Pas de PC local, pas de UTimeMaster, pas d'ADMS
- On a recu et analyse l'intro du protocole PUSH (18 pages sur 158)
- Phase POC : **1 terminal** pour commencer

---

## PRIORITE 1 — Questions bloquantes (deal-breakers potentiels)

### 1. HTTPS / TLS
**Le Horus E1-FP supporte-t-il les connexions HTTPS (TLS/SSL) ?**
- Notre serveur (Vercel) ne sert qu'en HTTPS (port 443)
- Si le device ne supporte que le HTTP simple, on ne peut pas se connecter directement
- Si pas de HTTPS : quelle solution ? Proxy ? Port specifique ?

### 2. Nom de domaine vs IP
**Le device accepte-t-il un nom de domaine (ex: `monapp.vercel.app`) ou seulement une adresse IP ?**
- Vercel utilise des IPs dynamiques derriere un CDN
- Si le device n'accepte que des IPs fixes, c'est un probleme

### 3. Horus H1 vs Horus E1-FP
**Le modele "Horus H1" que vous mentionnez est-il le meme que le "Horus E1-FP" ?**
- On veut le modele avec **visage + empreinte digitale + 4G**
- Le E1-FP est-il bien compatible avec le protocole PUSH ?

---

## PRIORITE 2 — Documentation et NDA

### 4. Document complet du protocole PUSH
**Peut-on obtenir le document complet (158 pages) rapidement ?**
- On a l'intro (18 pages) — sections 1 a 4
- On a besoin des sections 5 (initialisation), 11 (upload data), 12 (commandes)
- Sans le document complet, on ne peut pas commencer le developpement

### 5. NDA
**Quel est le processus pour le NDA ?**
- On est prets a signer
- Delai de signature → delai de reception de la doc complete ?

### 6. Exemples de code / serveur de reference
**Existe-t-il un exemple d'implementation serveur ?**
- Un sample code (Python, Node.js, PHP) qui implemente les endpoints /iclock/*
- Ou un serveur de test pour valider l'integration avant d'avoir le device physique

---

## PRIORITE 3 — Architecture et configuration

### 7. Intervalle de polling
**Quel est l'intervalle de polling configurable sur le device ?**
- Le device appelle GET /iclock/getrequest a quelle frequence ?
- Minimum / maximum configurable ?
- Ca determine le delai entre "le RH clique Ajouter" et "le device recoit la commande"

### 8. Timeout HTTP
**Quel est le timeout HTTP du device ?**
- Si notre serveur met plus de X secondes a repondre, le device timeout ?
- Important car Vercel a des cold starts (1-3 secondes)

### 9. Donnees offline
**Quand le device a stocke des pointages offline et se reconnecte :**
- Envoie-t-il tout en un seul POST ou plusieurs requetes ?
- Quelle est la taille maximale d'un POST ?
- Le mecanisme Stamp gere-t-il bien la reprise ?

### 10. Certificats HTTPS
**Si HTTPS est supporte, le device gere-t-il le renouvellement de certificats ?**
- Vercel renouvelle ses certificats TLS automatiquement
- Le device fait-il du certificate pinning ?

---

## PRIORITE 4 — Fonctionnalites

### 11. Enrolement a distance
**L'enrolement visage/empreinte peut-il etre declenche a distance via le protocole PUSH ?**
- Section 12.6 mentionne "Remote Enrollment Commands"
- Ca fonctionne sur le Horus E1-FP ?

### 12. Envoi de photos utilisateur
**Peut-on envoyer une photo d'employe au device via le protocole PUSH ?**
- Pour l'affichage sur l'ecran lors de la verification
- Section 11.13 mentionne "Uploading User Photo"

### 13. Mise a jour firmware a distance
**Le firmware du Horus E1-FP peut-il etre mis a jour via le protocole PUSH ?**
- Section 12.8.2 mentionne "Online Update"
- Important pour la maintenance a distance en Guinee

---

## PRIORITE 5 — Commercial

### 14. Prix du SDK / Licence PUSH
- **Combien coute la licence PUSH SDK ?**
- Est-ce un paiement unique ou annuel ?
- Par device ou universel (illimite) ?

### 15. Achat du device
- **Peut-on acheter le Horus E1-FP directement chez ZKTeco Europe ?**
- Quel prix ?
- Delai de livraison a Paris ?

### 16. Support technique
- **Le SDK inclut-il du support technique pour l'integration ?**
- On developpe un serveur custom (pas WDMS/ZKECO)
- Aura-t-on de l'aide si on a des problemes d'integration ?

### 17. Simulateur / Environnement de test
- **Existe-t-il un simulateur du protocole PUSH ?**
- Pour tester nos API sans avoir le device physique
- Accelererait enormement le developpement

---

## Notes a prendre pendant la reunion

| Question | Reponse | Action |
|----------|---------|--------|
| HTTPS supporte ? | | |
| Domaine accepte ? | | |
| H1 = E1-FP ? | | |
| Doc complete quand ? | | |
| NDA processus ? | | |
| Intervalle polling ? | | |
| Timeout HTTP ? | | |
| Prix SDK ? | | |
| Prix device ZKTeco Europe ? | | |
| Support integration ? | | |
| Simulateur dispo ? | | |
| Enrolement a distance ? | | |

---

## Apres la reunion

- [ ] Signer le NDA
- [ ] Obtenir le document complet (158 pages)
- [ ] Obtenir un devis device + SDK
- [ ] Commencer le developpement des API routes si HTTPS + domaine confirmes
- [ ] Commander le device
