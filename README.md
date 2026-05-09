# LAB 15 — Analyse Dynamique Android : Inspection TLS/HTTPS et Bypass SSL Pinning

## Description

Ce laboratoire a pour objectif d’apprendre à :

- Installer et configurer Frida
- Intercepter le trafic HTTPS Android
- Configurer un proxy HTTP(S)
- Bypasser le SSL Pinning
- Analyser les communications réseau d’une application Android

---

# Objectifs

À la fin de ce laboratoire, vous serez capables de :

- Déployer `frida-server`
- Connecter Frida à un appareil Android
- Utiliser Burp Suite ou mitmproxy
- Injecter des scripts JavaScript Frida
- Désactiver le SSL Pinning
- Inspecter les requêtes HTTPS d’une application mobile

---

# Prérequis

## Logiciels nécessaires

- Python 3
- ADB
- Frida
- Burp Suite ou mitmproxy
- Téléphone Android ou émulateur

## Configuration Android

- Options développeur activées
- Débogage USB activé

---

# Étape 1 — Installation de Frida

## Installation côté PC

```bash
python -m pip install --upgrade frida frida-tools
```

## Vérification

```bash
frida --version
```

```bash
python -c "import frida; print(frida.__version__)"
```

---

# Étape 2 — Préparation Android

## Vérifier la connexion ADB

```bash
adb devices
```

## Identifier l’architecture CPU

```bash
adb shell getprop ro.product.cpu.abi
```

### Exemples d’architectures

- arm64-v8a
- armeabi-v7a
- x86_64

---

# Étape 3 — Déployer frida-server

## Télécharger frida-server

Télécharger la version correspondant à celle installée sur le PC :

https://github.com/frida/frida/releases

> Important : la version de `frida-server` doit être identique à celle affichée par `frida --version`.

---

## Transférer le serveur

```bash
adb push frida-server /data/local/tmp/
```

## Donner les permissions

```bash
adb shell chmod 755 /data/local/tmp/frida-server
```

## Lancer frida-server

```bash
adb shell "/data/local/tmp/frida-server -l 0.0.0.0"
```

---

# Étape 4 — Vérification Frida

## Forward des ports

```bash
adb forward tcp:27042 tcp:27042
```

```bash
adb forward tcp:27043 tcp:27043
```

## Tester la connexion

```bash
frida-ps -Uai
```

---

# Étape 5 — Configuration du Proxy HTTPS

## Configuration Burp Suite

Configurer le proxy :

- Adresse : `127.0.0.1`
- Port : `8080`

---

## Configuration du proxy Android

Dans les paramètres Wi-Fi :

- Proxy : Manuel
- Adresse IP : IP du PC
- Port : `8080`

---
<img width="504" height="931" alt="Screenshot 2026-05-08 193510" src="https://github.com/user-attachments/assets/20e39ca9-7cbd-4b82-a7fd-a94f35af4cb5" />

# Étape 6 — Installer le certificat CA
<img width="466" height="922" alt="Screenshot 2026-05-08 194003" src="https://github.com/user-attachments/assets/ebc2825e-0317-44da-9c50-b5e14bf36e98" />

## Exporter le certificat Burp

Dans Burp Suite :

```text
Proxy → Options → Import / export CA certificate
```

Exporter le certificat au format DER ou CER.

---

## Installer le certificat sur Android

1. Copier le certificat sur le téléphone
2. Ouvrir :

```text
Paramètres → Sécurité → Installer un certificat
```

3. Installer comme certificat CA utilisateur

---
<img width="475" height="107" alt="image" src="https://github.com/user-attachments/assets/be78e9d5-b027-497c-9393-a74562058a00" />

# Étape 7 — Lancer l’application sous Frida

## Trouver le package

```bash
frida-ps -Uai
```

## Démarrer l’application

```bash
frida -U -f com.example.app
```

---

# Étape 8 — Script universel SSL Pinning Bypass

## Créer le fichier

```text
ssl_bypass.js
```

## Contenu du script

```javascript
Java.perform(function () {

    console.log("[*] SSL Pinning Bypass Loaded");

    var X509TrustManager = Java.use('javax.net.ssl.X509TrustManager');
    var SSLContext = Java.use('javax.net.ssl.SSLContext');

    var TrustManager = Java.registerClass({
        name: 'com.example.TrustManager',
        implements: [X509TrustManager],
        methods: {
            checkClientTrusted: function () {},
            checkServerTrusted: function () {},
            getAcceptedIssuers: function () {
                return [];
            }
        }
    });

    var TrustManagers = [TrustManager.$new()];

    var SSLContext_init =
        SSLContext.init.overload(
            '[Ljavax.net.ssl.KeyManager;',
            '[Ljavax.net.ssl.TrustManager;',
            'java.security.SecureRandom'
        );

    SSLContext_init.implementation = function (
        keyManager,
        trustManager,
        secureRandom
    ) {

        console.log('[+] SSLContext intercepted');

        SSLContext_init.call(
            this,
            keyManager,
            TrustManagers,
            secureRandom
        );
    };
});
```

---

# Étape 9 — Injecter le script

```bash
frida -U -f com.example.app -l ssl_bypass.js
```

---
<img width="605" height="261" alt="image" src="https://github.com/user-attachments/assets/b191190b-6278-420a-8ba1-602a55024e68" />

# Étape 10 — Vérification

Dans Burp Suite, vous devez maintenant voir :

- Les requêtes HTTPS
- Les réponses API
- Les headers HTTP
- Les tokens
- Les données JSON

---

# Étape 11 — Cas avancés

Certaines applications utilisent des protections supplémentaires :

- OkHTTP Pinning
- TrustKit
- Conscrypt
- BoringSSL
- OpenSSL natif

Dans ces cas :

- Utiliser des hooks spécifiques
- Hooker les fonctions natives
- Utiliser Objection
- Utiliser Frida Gadget

---

# Dépannage

## Erreur : Version mismatch

### Message d’erreur

```text
unable to communicate with remote frida-server
```

### Solution

Utiliser exactement la même version de Frida :

- côté PC
- côté Android

---

## Erreur : Device unauthorized

### Commandes

```bash
adb kill-server
```

```bash
adb start-server
```

Reconnecter ensuite le téléphone et accepter la clé RSA.

---

## Frida détecté par l’application

### Solutions possibles

- Objection
- Frida Gadget
- Renommer `frida-server`
- Utiliser des bypass anti-Frida

---

# Bonnes pratiques

- Utiliser uniquement dans un cadre légal
- Tester uniquement sur vos propres applications
- Respecter les politiques de sécurité
- Ne jamais intercepter des données sans autorisation

---

# Résumé des commandes

## Installer Frida

```bash
python -m pip install frida frida-tools
```

## Vérifier les appareils

```bash
adb devices
```

## Lancer frida-server

```bash
adb shell "/data/local/tmp/frida-server -l 0.0.0.0"
```

## Voir les applications

```bash
frida-ps -Uai
```

## Injecter le bypass SSL Pinning

```bash
frida -U -f com.example.app -l ssl_bypass.js
```

---

# Auteur

LAB 15 — Sécurité des applications mobiles  
Analyse Dynamique Android : TLS/HTTPS & SSL Pinning
