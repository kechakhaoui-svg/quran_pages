# المصحف الشريف — application Android (Kotlin natif)

Projet Android Studio complet pour la Phase 1 (lecture) et le squelette de la Phase 2
(test de récitation). Ce code n'a **pas pu être compilé ni testé sur un appareil réel**
dans l'environnement où il a été écrit (pas de SDK Android disponible ici) : ouvrez-le
dans Android Studio, laissez Gradle synchroniser, et corrigez les éventuelles erreurs de
syntaxe mineures avant la première compilation.

## Mise en place avant de compiler

1. **Métadonnées** — copiez dans `app/src/main/assets/quran_meta/` :
   - `mushaf_531_database.json` (celui déjà corrigé et validé précédemment)
   - `surahs.json`
   - `quran_muhammadi.json` — **déjà inclus dans ce zip** (assets/quran_meta/), c'est le
     texte de référence mot à mot utilisé pour la Phase 2 (voir `QuranTextRepository.kt`) :
     pas besoin des 514 fichiers `page_XXX.txt` séparés, ce fichier unique suffit et évite
     d'avoir à filtrer les titres de sourate / insertions de basmala qu'ils contiennent.

2. **Images des 531 pages** — à copier sur l'appareil, PAS dans `assets/` (531 images en
   pleine résolution alourdiraient l'APK de façon déraisonnable). Au premier lancement,
   copiez-les vers `context.getExternalFilesDir(null)/pages/NNN.jpg` — par exemple via un
   petit écran d'installation qui décompresse un .zip embarqué en `assets/`, ou en les
   plaçant manuellement sur l'appareil de test pendant le développement
   (`adb push pages/ /sdcard/Android/data/com.warsh.mushaf/files/pages/`).

3. **Audio Al Kouchi** — vos fichiers mp3 (déjà découpés par sourate/verset) doivent être
   copiés vers `.../files/audio_alkouchi/` avec le nommage `SSS_AAA.mp3` (3 chiffres
   sourate, 3 chiffres verset — ex. `002_255.mp3`). **Si vos fichiers ont un autre nom**,
   ouvrez `AudioRepository.kt` et changez seulement la fonction `localFileName()`.

4. **Icône de l'application** — non fournie ici (nécessite plusieurs résolutions). Dans
   Android Studio : clic droit sur `res` → New → Image Asset, pour générer les
   `mipmap/ic_launcher*`.

5. **Kazabri (عمر القزابري)** — aucun fichier local n'a été fourni. L'app interroge en
   direct l'API publique **mp3quran.net** (`AudioRepository.resolveKazabriServer()`) pour
   trouver son moshaf en rewaya Warsh, puis télécharge et met en cache les mp3 par
   SOURATE ENTIÈRE (pas par verset — voir limite ci-dessous). Recherche effectuée durant
   ce travail : sa présence sur mp3quran.net n'a pas pu être confirmée à 100 % (l'API
   liste plus de 600 récitateurs) ; au premier essai réel, si la recherche échoue, un
   message d'erreur clair s'affiche et vous pourrez soit corriger le mot-clé de recherche
   dans le code, soit fournir vous-même une autre source.

## Ce qui est fait (Phase 1)

| Exigence | Fichier(s) | Remarque |
|---|---|---|
| Interface entièrement en arabe | `MushafApplication.kt`, `values/strings.xml` | Langue forcée via `AppCompatDelegate.setApplicationLocales`, indépendante de la langue du téléphone |
| Balayage gauche→droite = page suivante, droite→gauche = page précédente | `activity_main.xml` (`layoutDirection="rtl"`) + `MainActivity.setupPager()` | Le sens RTL de `ViewPager2` inverse nativement le geste ; aucune détection de geste ad hoc |
| Page active + 5 avant + 5 après en mémoire | `PageCacheManager.kt` | Recycle explicitement les `Bitmap` hors fenêtre (pas seulement une limite de taille comme `LruCache`) |
| Adaptation à l'écran, marges système respectées | `MainActivity.applySystemBarMargins()` | Utilise `WindowInsetsCompat` — pas de mode plein écran "edge-to-edge" |
| Bouton réglages en haut → menu de configuration | `SettingsBottomSheet.kt` | Feuille de bas d'écran (Material) |
| Lecture audio page / depuis verset choisi | `AudioController.kt`, `AudioRepository.kt` | Local (Al Kouchi) ou téléchargé à la demande et mis en cache (Kazabri) |
| Répétition + vitesse configurables | `AudioController` (ExoPlayer `PlaybackParameters`), `SettingsBottomSheet` | |

## Ce qui est fait (Phase 2 — squelette à valider)

- `WordAligner.kt` : alignement mot à mot (programmation dynamique, même principe que
  l'alignement page par page utilisé plus tôt dans ce projet), détecte mots omis/ajoutés.
- `SpeechToTextEngine.kt` : utilise `android.speech.SpeechRecognizer` (moteur Google
  intégré à Android). **Limite importante, documentée dans le fichier** : ce moteur est
  conçu pour la parole courante, pas pour la récitation coranique avec tajwid — sa
  précision réelle sur ce contenu n'est pas garantie et doit être testée avec de vrais
  utilisateurs avant de considérer cette fonctionnalité comme fiable. L'interface
  `SpeechToTextEngine` isole ce point pour permettre de basculer facilement vers un
  service de reconnaissance plus spécialisé si besoin.
- `RecitationCheckActivity.kt` : affiche le passage mot par mot, colore en rouge (barré =
  omis, souligné = ajouté), calcule un score (`ScoreCalculator.kt`).
- **Manque à faire** : l'écran qui construit la liste de mots attendus à partir d'une
  sourate/verset choisi par l'utilisateur, et le bouton qui lance
  `RecitationCheckActivity` avec `EXTRA_EXPECTED_WORDS` rempli. Câblage volontairement
  laissé à faire une fois le texte de référence (celui déjà extrait des 531 pages)
  intégré au projet.

## Aller plus loin (non fait dans cette passe)

- Lecture audio en arrière-plan / contrôles à l'écran verrouillé (nécessite un
  `MediaSessionService` Media3 complet, retiré du manifeste pour rester honnête sur ce
  qui est réellement implémenté).
- Sélecteur sourate/verset pour "lecture à partir de" (UI non fournie, seule la logique
  `AudioController.playFrom()` existe).
- Barre de progression / contrôles pause-reprise dans `bottomBar` (le conteneur existe
  dans `activity_main.xml`, vide pour l'instant).
