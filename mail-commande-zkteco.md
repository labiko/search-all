# Mail de commande ZKTeco

**A :** france@bydemes.com
**Objet :** Demande de devis — ZKTeco Horus E1-FP (1 unite)

---

Bonjour,

Je me permets de vous contacter en tant que distributeur officiel ZKTeco en France.

Nous developpons une solution SaaS de pointage biometrique pour le marche ouest-africain (Guinee-Conakry). Nous sommes en phase pilote et souhaitons commander le materiel suivant :

## Demande de devis

| Produit | Quantite |
|---------|----------|
| **ZKTeco Horus E1-FP** (Reconnaissance faciale + Empreinte digitale) | 1 unite |

**Livraison :** Paris, Ile-de-France

## Questions techniques (a confirmer avant achat)

Avant de finaliser la commande, nous avons besoin de confirmer les points suivants concernant la compatibilite SDK :

1. **SDK Standalone (zkemkeeper.dll) :** Le Horus E1-FP est-il compatible avec le SDK Standalone/Offline (zkemkeeper.dll v6.3.1.x) pour une communication via reseau local (UDP port 4370) ? Nous devons piloter l'appareil depuis un PC Windows sur le meme reseau LAN.

2. **Gestion des utilisateurs a distance :** Peut-on ajouter/supprimer des utilisateurs a distance via le SDK (fonctions SSR_SetUserInfo, SSR_DeleteEnrollData) ?

3. **Evenements temps reel :** L'appareil declenche-t-il des evenements OnAttTransactionEx en temps reel lorsqu'un employe pointe (visage ou empreinte) ?

4. **Enrolement visage via SDK :** L'enrolement facial peut-il etre declenche a distance via le SDK (SetUserFace / SendUserFacePhoto), ou doit-il obligatoirement se faire sur l'ecran de la pointeuse ?

5. **Enrolement empreinte via SDK :** La fonction StartEnrollEx fonctionne-t-elle sur le Horus E1-FP pour l'enrolement d'empreintes en ligne ?

6. **Version SDK recommandee :** Quelle version du SDK conseillez-vous pour le Horus E1-FP ? Nous disposons actuellement de la documentation des versions 6.3.1.43 et 6.3.1.55.

## Contexte de notre projet

Notre architecture repose sur un bridge Python tournant sur un PC Windows local qui communique avec le Horus E1-FP via le SDK (UDP:4370) et synchronise les donnees vers notre plateforme cloud. Nous n'utiliserons **pas** ZKTeco ADMS ni ZKBioTime — nous avons besoin d'un acces SDK direct a l'appareil.

Nous souhaiterions recevoir :
- Un devis incluant la livraison a Paris
- Le delai de livraison estime
- La confirmation de la compatibilite SDK

Je vous remercie par avance pour votre retour.

Cordialement
