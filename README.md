# LAB 13 — Bypass de la Détection de Root Android avec Objection

**Nom :** Ismaïl | **Date :** 19 Mai 2026 | **Module :** Sécurité Mobile / Pentest Android

---

## Étape 1 — Installation et vérification des outils

Installation d'Objection via pip, puis vérification de Frida et ADB.

```powershell
pip install --upgrade objection
pip show objection
frida --version
adb devices
```

![Objection 1.12.4, Frida 17.9.8 installés — emulator-5554 détecté](screenshots/etape1_versions.png)

---

## Étape 2 — Connexion Objection à l'app cible

Lancement d'Objection en mode attach sur `com.pwnsec.firestorm`.

```powershell
objection -g com.pwnsec.firestorm explore
```

![Bannière Objection v1.12.4 — invite com.pwnsec.firestorm (run) on (Android: 11) [usb] #](screenshots/etape2_objection_prompt.png)

---

## Étape 3 — Recherche des méthodes de détection root

Exploration du runtime pour identifier les méthodes `isRoot` présentes dans l'app.

```
android hooking search methods isRoot
```

![Liste des méthodes isRoot trouvées dans le runtime Android](screenshots/etape3_hooking_search_isroot.png)

---

## Étape 4 — Bypass root detection et SSL Pinning

Désactivation de la détection root et du SSL Pinning via Objection.

```
android root disable
android sslpinning disable
android hooking search classes root
```

![Jobs root-detection-disable et android-sslpinning-disable enregistrés avec succès](screenshots/etape3_root_disable_sslpinning.png)

---

## Conclusion

Objection injecte des hooks Frida dans le runtime Java de l'app pour masquer l'état rooté de l'appareil — sans modifier l'APK. Une protection efficace nécessite des vérifications natives (C/C++) combinées à une détection anti-instrumentation.
