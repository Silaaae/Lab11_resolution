# Lab 11 — Bypass de la Détection de Root Android avec Frida

**Cours :** Sécurité des applications mobiles
**Outil :** Frida + frida-server
**Cible :** Émulateur Android x86_64

---

## Étape 1 — Installation de Frida et vérification de l'environnement

Frida a été installé via pip et la connexion ADB avec l'émulateur a été vérifiée.

<img width="702" height="257" alt="image" src="https://github.com/user-attachments/assets/360c606a-d4cc-49ed-8b7c-f031cd343c0f" />

---

## Étape 2 — Déploiement de frida-server sur l'émulateur

Le binaire a été pushé dans `/data/local/tmp/`, rendu exécutable, puis lancé. `frida-ps -Uai` confirme que le serveur tourne.

<img width="1462" height="182" alt="image" src="https://github.com/user-attachments/assets/d6f317ae-5dc6-4f97-9742-ce6705c3bce4" />

---

## Étape 3 — Exécution du script de bypass root

Le script `bypass_root.js` a été injecté via Frida. Les hooks Java ont été installés : `Build.TAGS`, `File.exists()`, `Runtime.exec()`, `RootBeer.isRooted()`.

<img width="796" height="366" alt="image" src="https://github.com/user-attachments/assets/595d7481-2e04-4112-826c-770f3bb838a4" />

---

## Étape 4 — Résultat : détection de root contournée

<img width="772" height="306" alt="image" src="https://github.com/user-attachments/assets/e1fc5751-e539-487f-8730-1b207737b5ab" />

---

## Étape 5 — Script Anti-Frida

Un script `anti_frida.js` a masqué la présence de Frida en hookant `System.getenv()` et `Socket.connect()` pour bloquer les ports 27042/27043.

```javascript
Java.perform(function() {
  try {
    const Sys = Java.use('java.lang.System');
    Sys.getenv.overload('java.lang.String').implementation = function (name) {
      if (name && name.toLowerCase().indexOf('frida') !== -1) {
        console.log('[+] Hiding env var', name);
        return null;
      }
      return this.getenv(name);
    };
  } catch (e) {}
  try {
    const Socket = Java.use('java.net.Socket');
    Socket.connect.overload('java.net.SocketAddress').implementation = function (addr) {
      try {
        const s = addr.toString();
        if (s.indexOf(':27042') !== -1 || s.indexOf(':27043') !== -1) {
          console.log('[+] Blocked connect to', s);
          throw new Error('Connection refused');
        }
      } catch (_) {}
      return this.connect(addr);
    };
  } catch (e) {}
});
```

---

## Conclusion

Ce lab a démontré comment utiliser Frida pour contourner les mécanismes de détection de root. Les hooks Java ont neutralisé toutes les vérifications courantes, et le script anti-Frida a masqué la présence de l'instrumentation.
