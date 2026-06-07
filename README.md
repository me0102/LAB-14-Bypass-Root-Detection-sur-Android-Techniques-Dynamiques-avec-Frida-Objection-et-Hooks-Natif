# LAB-14-Bypass-Root-Detection-sur-Android-Techniques-Dynamiques-avec-Frida-Objection-et-Hooks-Natif
# Sécurité Mobile Android — Contournement de la Détection Root

> Méthodologie complète pour neutraliser la détection root et analyser des applications Android durcies à l'aide de Frida, Objection et Medusa sur émulateur.

---

## Sommaire

- [Configuration de l'environnement](#configuration-de-lenvironnement)
- [Arsenal technique](#arsenal-technique)
- [Mise en place](#mise-en-place)
- [UnCrackable Niveau 1](#uncrackable-niveau-1)
- [UnCrackable Niveau 3](#uncrackable-niveau-3)
- [Référence des scripts](#référence-des-scripts)
- [Problèmes fréquents et solutions](#problèmes-fréquents-et-solutions)

---

## Configuration de l'environnement

| Composant | Version / Détails |
|---|---|
| Système d'exploitation | Windows 11 |
| Émulateur Android | Android Emulator 5554 (Android 8.1.0 / API 27 / x86) |
| Frida Client | 17.9.1 |
| Objection | v1.12.4 |
| Medusa | dev |
| Applications cibles | `owasp.mstg.uncrackable1`, `owasp.mstg.uncrackable3` |

---

## Arsenal technique

**Frida** — Moteur d'instrumentation dynamique permettant d'injecter du code JavaScript dans des processus Android en cours d'exécution.

**Objection** — Suite d'exploration mobile en temps réel construite sur Frida. Elle expose des commandes de haut niveau pour automatiser des tâches courantes comme le contournement du root ou l'épinglage SSL.

**Medusa** — Framework modulaire basé sur Frida, regroupant plus de 124 modules prêts à l'emploi pour l'analyse approfondie d'applications Android.

**ADB** — Pont de débogage Android permettant de piloter l'émulateur et de gérer les installations d'APK.

---

## Mise en place

### 1. Vérifier la connexion ADB

```bash
adb devices
# Sortie attendue :
# emulator-5554   device
```

### 2. Déployer les APK cibles

```bash
adb install uncrackable1.apk
adb install uncrackable3.apk
```

### 3. Désinstaller une application

```bash
adb uninstall owasp.mstg.uncrackable1
```

### 4. Démarrer frida-server sur l'émulateur

```bash
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell /data/local/tmp/frida-server &
```

### 5. Tester la connexion Frida

```bash
frida -U -f owasp.mstg.uncrackable1 -l hello.js
```

> **Remarque :** L'option `--no-pause` a été supprimée dans les versions récentes de Frida. Il suffit de l'omettre.

---

## UnCrackable Niveau 1

### Objectif

Désactiver la détection root et extraire la chaîne secrète enfouie dans l'application.

### Approche 1 — Frida (scripts manuels)

**Étape 1 — Neutraliser la détection root :**

```bash
frida -U -f owasp.mstg.uncrackable1 -l bypass_root_basic.js
```

Sortie console attendue :
```
[+] Build.TAGS -> release-keys
[*] RootBeer non présent
[+] Runtime.exec hooks installés
[+] Bypass Java installé
[+] File.exists bypass: /sbin/su
[+] File.exists bypass: /system/bin/su
[+] File.exists bypass: /system/xbin/su
[+] File.exists bypass: /system/app/Superuser.apk
```

<img width="1548" height="609" alt="Résultat bypass root Level 1" src="https://github.com/user-attachments/assets/751c649a-ea1d-422f-a694-7c9bb33bd613" />

**Étape 2 — Intercepter la routine de déchiffrement :**

Intégrer le fragment suivant dans votre script Frida pour capturer le secret lors de son déchiffrement :

```javascript
Java.perform(function() {
    var b = Java.use("sg.vantagepoint.a.a");
    b.a.implementation = function(key, enc) {
        var result = this.a(key, enc);
        var secret = "";
        for (var i = 0; i < result.length; i++) {
            secret += String.fromCharCode(result[i]);
        }
        console.log("[+] Secret déchiffré : " + secret);
        return result;
    };
});
```

### Approche 2 — Objection (recommandé pour débuter)

```bash
objection -n owasp.mstg.uncrackable1 start
```

Dans la console interactive d'Objection :

```
android root disable
android hooking watch class_method sg.vantagepoint.uncrackable1.a.a --dump-args --dump-return
```

<img width="1339" height="508" alt="Objection console Level 1" src="https://github.com/user-attachments/assets/185aa403-780a-4952-b0ab-8e3e6df10d25" />

Saisir n'importe quel mot de passe dans l'application — le vrai secret s'affiche immédiatement dans la console.

### Approche 3 — Medusa

```bash
python medusa.py
# Sélectionner le périphérique : 3 (emulator-5554)
# Sélectionner l'application : 0 (owasp.mstg.uncrackable1)
```

Dans la console Medusa :

```
use root_detection/universal_root_detection_bypass
run
```

<img width="1273" height="811" alt="Medusa root bypass" src="https://github.com/user-attachments/assets/44c01234-de71-49c9-9eb5-d270a72a7537" />

---

## UnCrackable Niveau 3

### Objectif

Contourner simultanément la détection root **et** la protection anti-altération native reposant sur `abort()` via un thread `pthread`.

### Complexité supplémentaire par rapport au Niveau 1

- Une bibliothèque native effectue des contrôles d'intégrité dans un thread d'arrière-plan
- L'application peut appeler `abort()` directement au niveau natif — les hooks Java sur `System.exit` sont insuffisants à eux seuls
- Les exports de `libc.so` peuvent varier selon les émulateurs x86

### Méthode — Frida avec deux scripts combinés

```bash
frida -U -f owasp.mstg.uncrackable3 -l bypass_root_basic.js -l bypass_native.js
```

**bypass_native.js** doit impérativement :
1. Hooker `System.exit` et `Runtime.exit` à l'intérieur de `Java.perform`
2. Hooker `fopen` et `access` depuis `libc.so` **en dehors** de `Java.perform`

> **Erreur classique :** Appeler `Module.findExportByName` à l'intérieur de `Java.perform` provoque un `TypeError: not a function` sur les émulateurs x86.

**Structure correcte :**

```javascript
// ✅ Hooks natifs EN DEHORS de Java.perform
function safeHook(lib, name, callback) {
    var addr = Module.findExportByName(lib, name);
    if (addr && !addr.isNull()) {
        try {
            Interceptor.attach(addr, callback);
            console.log("[+] Hook OK: " + name);
        } catch(e) {
            console.log("[!] Hook échoué: " + name + " -> " + e.message);
        }
    } else {
        console.log("[!] " + name + " introuvable dans " + lib);
    }
}

safeHook("libc.so", "fopen", { ... });
safeHook("libc.so", "access", { ... });

// ✅ Hooks Java À L'INTÉRIEUR de Java.perform
Java.perform(function() {
    Java.use("java.lang.System").exit.implementation = function(code) {
        console.log("[+] System.exit(" + code + ") bloqué");
    };
});
```

### Diagnostiquer les symboles libc disponibles

Si les hooks échouent, lister ce qui est réellement exporté sur votre émulateur :

```javascript
Module.enumerateExports("libc.so").forEach(function(e) {
    if (e.name.indexOf("open") !== -1 || e.name.indexOf("access") !== -1) {
        console.log(e.name + " -> " + e.address);
    }
});
```

---

## Référence des scripts

### `bypass_root_basic.js`

Neutralise les contrôles de détection root au niveau Java :

- Falsifie `Build.TAGS` en `release-keys`
- Intercepte `Runtime.exec` pour bloquer les appels à `su`
- Détourne `File.exists` pour renvoyer `false` sur les chemins root connus
- Détecte la présence de la bibliothèque RootBeer et la contourne si nécessaire

### `bypass_native.js`

Neutralise les contrôles de détection root au niveau natif :

- Détourne `fopen` et `access` dans `libc.so` pour filtrer les chemins sensibles (`/sbin/su`, `/system/bin/su`, `/system/app/Superuser.apk`, etc.)
- Intercepte `System.exit` et `Runtime.exit` pour empêcher les crashs déclenchés par l'anti-tampering

---

## Problèmes fréquents et solutions

### Argument `--no-pause` non reconnu

Supprimé depuis Frida 17+. Il suffit d'omettre le flag :
```bash
frida -U -f owasp.mstg.uncrackable1 -l script.js
```

### `TypeError: not a function` dans bypass_native.js

**Cause :** `Module.findExportByName` est appelé à l'intérieur de `Java.perform`.  
**Solution :** Déplacer tous les hooks natifs en dehors du bloc `Java.perform`.

### `Process crashed: Trace/BPT trap` / SIGABRT

**Cause :** Le thread natif anti-tampering déclenche `abort()` avant que les hooks Java ne soient en place.  
**Solution :** Hooker `System.exit` et `Runtime.exit` tout en interceptant les fonctions natives de `libc.so` avant que le thread de contrôle d'intégrité ne s'active.

### `Module root-bypass not found` dans Medusa

**Cause :** Nom de module incorrect.  
**Solution :** Utiliser la commande `search root` dans Medusa pour identifier le nom exact :
```
root_detection/universal_root_detection_bypass
```

### Avertissement de dépréciation de `objection -g`

**Cause :** Le flag `-g` est obsolète dans les versions récentes d'Objection.  
**Solution :** Remplacer par `-n` :
```bash
objection -n owasp.mstg.uncrackable1 start
```

---

## Références

- [OWASP MASTG](https://mas.owasp.org/MASTG/)
- [Documentation Frida](https://frida.re/docs/home/)
- [Dépôt Objection](https://github.com/sensepost/objection)
- [Dépôt Medusa](https://github.com/Ch0pin/medusa)
- [Applications UnCrackable](https://github.com/OWASP/owasp-mastg/tree/master/Crackmes)
