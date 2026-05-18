# Lab_13_MobileSecurity
# LAB 13 — Bypass de la Détection de Root Android avec Objection

---

## 1. Page de garde

| Champ   | Détail                                                    |
|---------|-----------------------------------------------------------|
| **Titre**  | Bypass de la Détection de Root Android avec Objection  |
| **Nom**    | Ismaïl                                                 |
| **Date**   | 19 Mai 2026                                            |
| **Module** | Sécurité Mobile / Pentest Android                      |

---

## 2. Objectif du TP

Ce TP a pour but de comprendre et de contourner les mécanismes de détection de root dans une application Android. On utilise **Objection**, un framework de pentest mobile basé sur **Frida**, pour injecter des hooks Java au sein d'une application cible et faire croire à celle-ci que l'appareil n'est pas rooté.

L'application cible utilisée est **Firestorm** (`com.pwnsec.firestorm`), installée sur un émulateur Android.

---

## 3. Environnement de travail

| Élément              | Détail                                 |
|----------------------|----------------------------------------|
| **Système d'exploitation** | Windows 11                       |
| **Émulateur**        | Android Emulator — Pixel 5 (Android 11)|
| **ADB**              | Android Platform Tools                 |
| **Frida**            | Version 17.9.8                         |
| **Objection**        | Version 1.12.4                         |
| **Application cible**| `com.pwnsec.firestorm`                 |
| **Shell utilisé**    | PowerShell                             |

---

## 4. Réalisation du TP

### Étape 1 — Installer Objection et vérifier les outils

On commence par installer Objection via pip, puis on vérifie que tous les outils sont bien en place : Frida, Objection et ADB.

**Commandes exécutées :**
```powershell
pip install --upgrade objection
pip show objection
frida --version
adb devices
```

**Résultat :** Objection v1.12.4 est installé, Frida est en version 17.9.8, et l'émulateur `emulator-5554` est bien détecté.

![Vérification des versions — Objection, Frida, ADB](screenshots/etape1_versions.png)

---

### Étape 2 — Démarrer frida-server et connecter Objection

On lance frida-server sur l'émulateur, puis on connecte Objection à l'application cible `com.pwnsec.firestorm` en mode **attach** (l'app est déjà en cours d'exécution).

**Commandes exécutées :**
```powershell
# Côté PC — connexion Objection à l'app
objection -g com.pwnsec.firestorm explore
```

**Résultat :** La bannière Objection v1.12.4 s'affiche et l'invite de commande interactive apparaît :
```
com.pwnsec.firestorm (run) on (Android: 11) [usb] #
```

![Bannière Objection et invite interactive sur com.pwnsec.firestorm](screenshots/etape2_objection_prompt.png)

---

### Étape 3 — Explorer les méthodes de root detection

Avant de lancer le bypass, on explore les méthodes de détection de root présentes dans l'application à l'aide de la commande de recherche d'Objection.

**Commande exécutée dans la console Objection :**
```
android hooking search methods isRoot
```

**Résultat :** Objection liste toutes les méthodes contenant `isRoot` dans le runtime Android. Parmi elles, on trouve des méthodes appartenant à `InputMethodService` et d'autres classes système, ce qui confirme que l'app inspecte l'état root de l'appareil.

![Recherche des méthodes isRoot dans le runtime Android](screenshots/etape3_hooking_search_isroot.png)

---

### Étape 4 — Bypass de la détection root (android root disable)

On exécute la commande principale du bypass. Objection installe automatiquement des hooks Frida qui :
- Interceptent les vérifications de fichiers suspects (`/system/xbin/su`, `/sbin/su`, etc.)
- Modifient la valeur de `android.os.Build.TAGS` pour renvoyer `release-keys`
- Neutralisent les appels à `Runtime.exec()` pour bloquer l'exécution de commandes `su`

On désactive aussi le **SSL Pinning** et on recherche les classes liées au root.

**Commandes exécutées dans la console Objection :**
```
android root disable
android sslpinning disable
android hooking search classes root
```

**Résultat :** Objection enregistre les jobs avec succès :
- `Job 288511 — root-detection-disable` ✅
- `Job 224455 — android-sslpinning-disable` ✅
- La recherche de classes root s'exécute en parallèle.

![Bypass root disable + sslpinning disable + hooking search classes](screenshots/etape3_root_disable_sslpinning.png)

---

## 5. Résultats obtenus

| Exercice | Résultat |
|----------|----------|
| Installation Objection + Frida | ✅ Objection 1.12.4 et Frida 17.9.8 installés |
| Connexion à l'app via Objection | ✅ Invite `com.pwnsec.firestorm (run) on (Android: 11)` obtenue |
| Exploration des méthodes root | ✅ Méthodes `isRoot` trouvées via `android hooking search methods` |
| Bypass root detection | ✅ Job `root-detection-disable` enregistré avec succès |
| Désactivation SSL Pinning | ✅ Job `android-sslpinning-disable` enregistré avec succès |

---

## 6. Explication technique — Que fait `android root disable` ?

Objection utilise Frida pour injecter du code JavaScript directement dans le runtime de l'application. Concrètement, les hooks installés :

1. **Interceptent `java.io.File.exists()`** — Quand l'app cherche des fichiers comme `/system/xbin/su` ou `/sbin/su`, la méthode retourne `false` au lieu de `true`.
2. **Modifient `android.os.Build.TAGS`** — La valeur est forcée à `release-keys` (valeur d'un appareil non rooté), au lieu de `test-keys`.
3. **Bloquent `Runtime.getRuntime().exec()`** — Les exécutions de commandes comme `which su` ou `su` retournent une erreur.
4. **Patchent les libs de sécurité** — Des bibliothèques populaires comme RootBeer voient leurs méthodes `isRooted()` retourner `false`.

```
su
```

---

## 7. Conclusion

Ce TP nous a permis de comprendre comment fonctionne la détection de root dans une application Android et comment la contourner efficacement avec **Objection** et **Frida**.

En quelques commandes seulement, on a réussi à :
- Injecter des hooks Java dans une application en cours d'exécution
- Masquer l'état rooté de l'émulateur
- Explorer le runtime de l'app pour identifier les mécanismes de sécurité

Ce type d'attaque montre que la détection de root côté Java est insuffisante si elle n'est pas combinée avec des vérifications natives (C/C++) et une protection contre l'instrumentation dynamique (anti-Frida).

---

## Structure du projet

```
Lab13/
├── README.md
├── bypass_native.js          ← Script Frida pour hooks natifs (bonus)
└── screenshots/
    ├── etape1_versions.png                  ← pip show objection + frida --version + adb devices
    ├── etape2_objection_prompt.png          ← Bannière Objection + invite interactive
    ├── etape3_hooking_search_isroot.png     ← android hooking search methods isRoot
    └── etape3_root_disable_sslpinning.png  ← android root disable + sslpinning disable
```
