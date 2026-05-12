# Lab11_resolution
Lab 11 — Bypass de la Détection de Root Android avec Frida
Cours : Sécurité des applications mobiles
Outil : Frida + frida-server
Cible : Émulateur Android x86_64

Étape 1 — Installation de Frida et vérification de l'environnement
Frida a été installé via pip et la version a été vérifiée, ainsi que la connexion ADB avec l'émulateur.
Show Image

Étape 2 — Déploiement de frida-server sur l'émulateur
Le binaire frida-server.1-android-x86_64 (déjà extrait) a été pushé dans /data/local/tmp/, rendu exécutable, puis lancé en arrière-plan. La commande frida-ps -Uai confirme que le serveur tourne et liste les applications disponibles.
Show Image

Étape 3 — Exécution du script de bypass root (Plan B — Frida pur)
Le script bypass_root.js a été injecté via Frida sur l'application cible. Les hooks Java suivants ont été installés avec succès :

Build.TAGS → retourne release-keys au lieu de test-keys
File.exists() → retourne false pour les chemins suspects (/system/xbin/su, /sbin/su, etc.)
Runtime.exec() → bloque toute tentative d'exécution de su ou busybox
RootBeer.isRooted() → retourne false

Show Image

Étape 4 — Résultat : détection de root contournée
Après injection du script, l'application ne détecte plus l'environnement rooté. Les vérifications qui retournaient "ROOTED" passent désormais sans blocage.
Show Image

Étape 5 — Script Anti-Frida (protection supplémentaire)
Un script complémentaire anti_frida.js a également été utilisé pour masquer la présence de Frida lui-même, en hookant deux points clés :

System.getenv() : retourne null pour toute variable d'environnement contenant frida
Socket.connect() : bloque toute connexion vers les ports Frida habituels (27042, 27043)

javascriptJava.perform(function() {
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

Conclusion
Ce lab a démontré comment utiliser Frida pour contourner les mécanismes de détection de root dans une application Android. L'approche Plan B (Frida pur, sans Medusa) s'est avérée pleinement fonctionnelle : les hooks Java ont neutralisé toutes les vérifications courantes, et le script anti-Frida a permis de masquer la présence de l'instrumentation elle-même.
