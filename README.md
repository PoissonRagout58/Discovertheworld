# DTW — Discover The World

App "fog of war" GPS : la carte du monde démarre noire, et se révèle en satellite
(Esri World Imagery, gratuit, sans clé API) autour de chaque endroit où tu es
réellement passé. Le rayon de révélation est un simple cercle de 150 m
(modifiable via `VISION_RADIUS_M` dans `index.html`).

Un seul fichier `index.html` (vanilla JS, comme Immersion/DevHub), pas de build.

## 1. Créer le projet Firebase

1. https://console.firebase.google.com → **Ajouter un projet**.
2. **Authentication** → Sign-in method → active **Email/Password**.
3. **Firestore Database** → Créer une base, mode production, région Europe (`eur3`).
4. Project settings → **Tes applications** → Ajouter une app Web (</>) → copie
   l'objet `firebaseConfig` qui s'affiche.
5. Colle-le dans `index.html`, tout en haut du `<script>`, à la place des
   `REPLACE_ME` :

```js
const firebaseConfig = {
  apiKey: "...",
  authDomain: "...",
  projectId: "...",
  storageBucket: "...",
  messagingSenderId: "...",
  appId: "..."
};
```

## 2. Déployer les règles de sécurité

Dans la console Firebase → Firestore → **Règles**, colle le contenu de
`firestore.rules` puis publie. Ça garantit qu'un utilisateur ne peut **jamais**
lire ou écrire les données d'un autre profil (chaque zone révélée est en plus
immuable après création, donc infalsifiable a posteriori).

Avec la CLI, si tu préfères :

```bash
npm install -g firebase-tools
firebase login
firebase init firestore   # choisis ton projet, garde firestore.rules
firebase deploy --only firestore:rules
```

## 3. Tester en local

```bash
cd dtw
python3 -m http.server 8000
```

Ouvre `http://localhost:8000`. **Important** : `navigator.geolocation` exige
HTTPS (sauf `localhost`), donc pour tester en conditions réelles sur ton
téléphone il te faut du HTTPS — même solution que pour FreshTrack :
Cloudflare Tunnel, ou héberge directement sur **Firebase Hosting** (gratuit,
HTTPS automatique) :

```bash
firebase init hosting     # public dir = dtw/
firebase deploy --only hosting
```

## 4. Champ de vision ajustable

Le rayon de révélation n'est plus figé à 150 m en dur dans le code : il y a un
bouton réglages (icône engrenage, en bas à droite) avec un slider de 30 à
300 m, réglable en direct pendant que tu regardes la carte se dévoiler.

Réglage par défaut : **80 m**, plus réaliste qu'un rayon fixe de 150 m pour
une visibilité piétonne (en ville, les bâtiments limitent souvent la vue à
60-100 m ; en zone dégagée tu peux monter à 150-200 m). Le réglage est
sauvegardé sur ton profil Firestore, donc il persiste d'une session à l'autre.

Un nouveau point n'est enregistré que tous les `rayon × 0,6` mètres environ
(visible dans le panneau de réglage) — assez pour que le chemin révélé reste
continu, sans empiler des cercles à moitié superposés qui donneraient
l'impression de découvrir bien plus large que ce que tu as vraiment parcouru.

**Remarque sur "% de la planète" et "Zone explorée"** : ce sont des sommes
d'aires de cercles (π×r² par point), donc une légère surestimation puisque
les cercles voisins se chevauchent un peu — un ordre de grandeur correct,
pas une mesure de pixels exacte de la zone réellement visible à l'écran.

## 5. Boucles fermées (marche autour d'une zone)

Pendant que le tracking est actif, si ton trajet revient à moins de
`LOOP_CLOSE_THRESHOLD_M` (20 m) d'un point que tu as déjà parcouru dans la
même sortie, l'app considère que tu as "refermé une boucle" — l'intégralité
de la zone à l'intérieur du polygone que tu viens de tracer est révélée d'un
coup, en plus de ton champ de vision habituel. Pratique pour un parc, un
pâté de maisons ou un rond-point sans avoir à quadriller chaque mètre carré.

Garde-fous pour éviter les faux positifs et les abus :
- Il faut avoir marché au moins `LOOP_MIN_PATH_M` (60 m) et posé au moins
  `LOOP_MIN_POINTS` (6 points GPS) avant qu'une fermeture soit même recherchée
  — sinon rester immobile quelques secondes (bruit GPS) déclencherait des
  boucles minuscules en continu.
- Une boucle de moins de 100 m² est ignorée (probablement du bruit GPS plutôt
  qu'une vraie boucle).
- Une boucle de plus de `MAX_LOOP_AREA_M2` (2 km²) est ignorée — sans ce
  plafond, croiser ton propre trajet des kilomètres plus loin dans la
  journée révélerait tout ce qu'il y a entre les deux, ce qui casserait tout
  l'intérêt du principe "il faut vraiment explorer".
- Une fois une boucle détectée, le trajet suivi repart de zéro à partir du
  point de fermeture, pour pouvoir détecter d'autres boucles distinctes dans
  la même sortie (faire le tour de deux pâtés de maisons d'affilée, par
  exemple).

L'aire du polygone est calculée avec la formule du lacet (shoelace) sur une
projection plane locale — largement suffisant pour des boucles de quelques
centaines de mètres, l'erreur de projection est négligeable à cette échelle.

Comme pour les zones révélées classiques, aucune trace continue n'est
stockée : seuls les sommets du polygone final (le contour de la boucle) sont
sauvegardés dans `users/{uid}/loops/{loopId}`, jamais le chemin GPS complet
avec ses horodatages.

## 6. Carnet d'exploration (stats)

Tape sur la carte "Zone explorée" en bas à gauche pour déplier :

- **% de la planète découverte** — surface révélée / 510,1 M km² (Terre entière,
  océans inclus). Les chiffres seront minuscules au début (attends-toi à du
  `1.2e-7 %`), c'est normal et plutôt marrant à regarder grandir.
- **Distance parcourue** — cumul de toutes tes sorties, en `+= à chaque
  déplacement` filtré pour ignorer le bruit GPS (mouvements < 3 m ignorés) et
  les sauts improbables (> ~43 km/h entre deux points, filtré comme glitch).
- **Pas estimés** — distance totale / 0,75 m (longueur de foulée moyenne,
  `STRIDE_M` dans le code si tu veux l'ajuster à la tienne).
- **Record d'une sortie** — la plus longue distance parcourue en une seule
  session de tracking (du moment où tu appuies sur le bouton GPS jusqu'à ce
  que tu l'arrêtes).

La distance de la session en cours est calculée en local en temps réel ; elle
n'est écrite dans Firestore qu'**une fois à l'arrêt du tracking** (pas à
chaque mètre), pour limiter les écritures. Si l'app est fermée brutalement en
plein tracking, cette session n'est pas comptabilisée — à améliorer plus tard
avec une sauvegarde périodique si besoin.

## 7. Conformité RGPD

Trois nouveaux fichiers : `confidentialite.html` et `mentions-legales.html`
(pages légales, à héberger à côté d'`index.html`), plus les règles Firestore
mises à jour.

**À faire avant de partager l'app au-delà de toi-même :**
- Remplace tous les `[PLACEHOLDER]` dans `confidentialite.html` et
  `mentions-legales.html` par ton nom et ton email de contact.
- Vérifie que ton projet Firestore est bien en région Europe (`eur3`), sinon
  les données ne restent pas dans l'UE (voir section 1).
- Dans la console Firebase → **Authentication → Templates**, pour chaque
  modèle (Vérification de l'adresse e-mail, Réinitialisation du mot de passe,
  Changement d'adresse e-mail) : ouvre le sélecteur de langue en haut du
  panneau et choisis **Français**. Le code force déjà `auth.languageCode =
  'fr'`, donc Firebase enverra automatiquement la version française de ces
  modèles une fois sélectionnée côté console — les deux réglages sont
  nécessaires (l'un ne suffit pas sans l'autre).

**Ce qui est déjà implémenté dans l'app :**
- **Consentement explicite** à l'inscription (case à cocher obligatoire, liée
  à la politique de confidentialité), avec horodatage stocké (`consentAt`).
- **Droit à la portabilité** : bouton "Exporter mes données" dans le profil,
  télécharge un JSON avec ton profil, tes zones révélées et tes zones
  capturées.
- **Droit à l'effacement** : bouton "Supprimer mon compte" — supprime tes
  zones révélées, libère les zones de territoire que tu possèdes, supprime ton
  profil, puis ton compte Firebase Auth. Irréversible, avec confirmation.
- **Minimisation des données** : toujours pas de trace GPS continue stockée
  (inchangé), seulement des positions ponctuelles.
- **Transparence sur la visibilité publique** : le texte de consentement et la
  politique précisent clairement que les zones capturées (nom, couleur,
  position) sont visibles par tous les joueurs — contrairement aux zones
  révélées (fog of war) et aux stats, qui restent strictement privées.
- **Emails d'authentification en français** : email de vérification envoyé
  automatiquement à l'inscription, lien "Mot de passe oublié ?" sur l'écran de
  connexion. Les deux passent par les modèles Firebase (voir réglage console
  ci-dessus pour qu'ils s'affichent en français).

**Limite technique honnête à connaître** : `currentUser.delete()` (suppression
du compte Firebase Auth) échoue si la session est trop ancienne
(`auth/requires-recent-login`) — Firebase l'exige par sécurité. Le message
d'erreur invite alors à se reconnecter avant de réessayer ; c'est géré dans le
code mais bon à savoir si tu testes.

## 8. Choix de design assumés

- **Imagerie satellite : Esri World Imagery.** Gratuit, sans clé API, sans
  quota bloquant pour un usage perso — contrairement à Mapbox (clé + quota
  gratuit limité) ou Google Maps (payant au-delà d'un seuil). Si un jour tu
  veux du Mapbox pour du style custom, il suffit de remplacer l'URL du
  `L.tileLayer`.
- **Confidentialité par construction :** on ne stocke jamais ta trace GPS
  continue (positions horodatées en continu), seulement les zones déjà
  révélées (point + rayon). Un point n'est enregistré que si tu es à plus de
  ~82 m (`MIN_JUMP_M`) du dernier point connu, et les positions GPS imprécises
  (>60 m, `MAX_ACCURACY_M`) sont ignorées.
- **Brouillard :** canvas HTML5 par-dessus la carte, peint en noir, avec des
  trous en dégradé radial (`destination-out`) découpés à chaque position
  révélée — d'où le bord "carte déchirée" plutôt qu'un cercle net.

## 9. Vers l'app Android (WebView, comme FreshTrack)

Deux points spécifiques à gérer par rapport à FreshTrack :

- **Géolocalisation en WebView** : par défaut désactivée. Dans ton
  `WebChromeClient`, override `onGeolocationPermissionsShowPrompt` et appelle
  `callback.invoke(origin, true, false)` après avoir vérifié/demandé la
  permission Android `ACCESS_FINE_LOCATION` (et idéalement
  `ACCESS_BACKGROUND_LOCATION` si tu veux que ça tourne app fermée — mais ça
  demande une justification Play Store).
- **HTTPS obligatoire** : comme pour la caméra dans FreshTrack, sers le fichier
  via Firebase Hosting ou un tunnel HTTPS plutôt qu'en `http://192.168.0.13`.

## Prochaines étapes possibles

- Bouton "centrer sur ma position" dédié (actuellement le recentrage ne se
  fait qu'au premier fix GPS).
- Stats supplémentaires (distance parcourue, plus longue session).
- Export/partage d'une carte explorée (image PNG du fog révélé).
- Mode "amis" : partager sa carte explorée avec un profil précis (nécessiterait
  d'assouplir les règles Firestore pour un accès en lecture croisé, à faire
  avec précaution).
