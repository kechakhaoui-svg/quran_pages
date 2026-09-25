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

3. **Audio (Al Kouchi / Kazabri)** — rien à copier. Ces deux récitateurs n'existent pas
   sur verse.mp3quran.net (un mp3 par verset) ; l'app utilise leurs mp3 par sourate sur
   mp3quran.net (`koshi/`, `omar_warsh/`, voir `Reciter.kt`) et le minutage officiel de
   chaque verset (API `ayat_timing`, read 16 et 80). Chaque verset est lu comme un extrait
   du mp3 de sa sourate : seuls les octets des versets écoutés sont téléchargés, puis
   gardés dans un cache disque (`AudioCache`, 2 Go max) pour la relecture hors-ligne.

4. **Icône de l'application** — non fournie ici (nécessite plusieurs résolutions). Dans
   Android Studio : clic droit sur `res` → New → Image Asset, pour générer les
   `mipmap/ic_launcher*`.

5. **Métadonnées des pages** — `mushaf_531_database.json` ne sert plus qu'à la liste des
   numéros de page : ses champs `startAyah`/`endAyah` sont faux. Les versets de chaque
   page sont déduits des fichiers `texte_page/page_NNN.txt` (`MetadataLoader`), une fois,
   puis mis en cache dans `files/page_ranges_*.txt`.

## Ce qui est fait (Phase 1)

| Exigence | Fichier(s) | Remarque |
|---|---|---|
| Interface entièrement en arabe | `MushafApplication.kt`, `values/strings.xml` | Langue forcée via `AppCompatDelegate.setApplicationLocales`, indépendante de la langue du téléphone |
| Balayage gauche→droite = page suivante, droite→gauche = page précédente | `activity_main.xml` (`layoutDirection="rtl"`) + `MainActivity.setupPager()` | Le sens RTL de `ViewPager2` inverse nativement le geste ; aucune détection de geste ad hoc |
| Page active + 5 avant + 5 après en mémoire | `PageCacheManager.kt` | Recycle explicitement les `Bitmap` hors fenêtre (pas seulement une limite de taille comme `LruCache`) |
| Adaptation à l'écran, marges système respectées | `MainActivity.applySystemBarMargins()` | Utilise `WindowInsetsCompat` — pas de mode plein écran "edge-to-edge" |
| Bouton réglages en haut → menu de configuration | `SettingsBottomSheet.kt` | Feuille de bas d'écran (Material) |
| Récitation : choix du lecteur, page courante + suite, ou plage de versets | `RecitationBottomSheet.kt`, `AudioController.kt` | Le lecteur tourne les pages au fil de la récitation ; barre lecture/pause/arrêt en bas |
| Répétition (chaque verset / plage entière) + vitesse x0.5–x2 | `RecitationBottomSheet`, `AudioController` | |
| Aller à une page (appui sur le numéro de page) | `MainActivity.showGoToPageDialog()` | |
| Fihris des sourates | `MainActivity.showSurahList()` | |
| Recherche d'un mot (sans tashkil, tolérante hamza/alif) → ouverture de la page | `SearchActivity.kt`, `QuranTextRepository.search()` | Tests : `QuranSearchTest` |

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
- Surlignage du verset en cours sur l'image de la page (nécessiterait les coordonnées
  des versets sur chaque image).
