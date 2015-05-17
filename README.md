# Ghost-Blog bei Uberspace mit Passwort schützen
Viele betreiben ein Blog auf Basis von Ghost bei Uberspace, eingerichtet nach der [Anleitung im dortigen Wiki](https://wiki.uberspace.de/cool:ghost). Das Blog soll aber nicht immer öffentlich zugänglich sein, z.B. wenn es um Auslandsaufenthalte, Kinderfotos oder vereinsinterne Neuigkeiten geht. Bei einem "normalen" Blogsystem auf PHP-Basis ist dies meist entweder durch Plugins, oder durch einen Verzeichnisschutz von Apache via .htaccess-Datei möglich. Passende Plugins gibt es für Ghost aktuell nicht, aber bis zur Version 0.49 war es möglich, in die angelegte .htaccess-Datei (~/html/.htaccess) ein paar Zeilen zur HTTP-Basic-Authentifizierung zu ergänzen.
Setzt man dies bei einer aktuellen Version um, funktioniert der Passwortschutz grundsätzlich auch wie erwartet. Das Problem: Nun ist das Einloggen ins Backend (/ghost) nicht mehr möglich. Dies liegt am neu eingeführten OAuth für das Backend (siehe [Beitrag im offiziellem Forum](https://ghost.org/forum/installation/15795-password-protected-blog-produces-error-while-trying-to-login/7/)).
##Was nun?
Es gibt zwei verschiedene Möglichkeiten, auf eine werde ich näher eingehen: Es ist möglich, Ghost direkt als npm-package zu verwenden. Dann kann man sich eine eigene Node-Anwendung basteln, die Ghost und ein Paket zur HTTP-Basic-Authentifizierung vorraussetzt und damit den Passwortschutz für Ghost erzeugen (siehe [Beitrag im offiziellem Forum](https://ghost.org/forum/installation/5105-solved-any-way-to-add-a-simple-authentication-method/12/)). Dies ist allerdings etwas komplexer und man müsste von der uberspace-Anleitung abweichen, weswegen ich die aus meiner Sicht einfacherere Methode gewählt habe:  
##Die Anleitung: Passwortschutz einrichten, /ghost-Verzeichnis ausnehmen
Um nun den Passwortschutz einzurichten & trotzdem noch auf das Admin-Interface zugreifen zu können, müssen wir das /ghost-Verzeichnis vom Schutz ausnehmen. Wenn man direkten Zugriff auf die Konfiguration hätte, könnte man dies [über die Location-Direktive angeben](https://ghost.org/forum/installation/15795-password-protected-blog-produces-error-while-trying-to-login/10/), dies ist in .htaccess-Dateien allerdings nicht möglich und damit entfällt diese Methode.  
Daher erstellen wir zunächst eine Subdomain, mit der wir zukünftig auf das Backend von Ghost zugreifen:  

    [lisbeth@amnesia ~]$ cd ~/html
    [lisbeth@amnesia html]$ mkdir ghost.lisbeth.amnesia.de
    [lisbeth@amnesia html]$ cd ghost.lisbeth.amnesia.de
    [lisbeth@amnesia ghost.lisbeth.amnesia.de]$ cat <<__EOF__ >> ~/html/.htaccess
    RewriteEngine On
    RewriteCond %{REQUEST_URI} "/ghost/" [OR]
    RewriteCond %{REQUEST_URI} "/content/"
    RewriteRule ^(.*) http://localhost:12345/\$1 [P]
    __EOF__

"ghost.lisbeht.amnesia.de" ist in diesem Fall der Name der Subdomain, über die wir zukünftig auf das Backend zugreifen, 12345 ist die Portnummer. Die Anweisungen in der .htaccess-Datei erlauben den Zugriff, wenn der Pfad `/ghost` (für das Backend) oder `/content` (für Bilder) enthält. Der direkte Zugriff auf die Blogseiten ist nicht möglich. Hinweis: Damit schaffen wir eine Sicherheitslücke, denn über http://ghost.lisbeth.amnesia.uberspace.de/content ist nun der Zugriff auf die Bilder möglich, sofern der genaue Pfad bekannt ist. Lässt man das `[OR] RewriteCond %{REQUEST_URI} "/content/"` weg, ist dies nicht mehr möglich, dann können wir beim Bearbeiten die Bilder nicht in der Vorschau sehen.  
Im zweiten Schritt richten wir nun wie oben erwähnt den Passwortschutz für Ghost ein. Dazu ergänzen wir in der .htaccess-Datei im ~/html-Verzeichnis (oder unter der Subdomain, unter der wir Ghost installiert haben) folgende Zeilen (einfach mit einem beliebigen Texteditor oben einfügen):

    AuthType Basic
    AuthName "Ghost Blog"
    AuthUserFile /var/www/virtual/lisbeth/html/.htpasswd
    Require valid-user

Mit `htpasswd -c /var/www/virtual/lisbeth/html/.htpasswd lisbeth` legen wir dann eine passende .htpasswd-Datei mit den zulässigen Benutzern an (in diesem Fall ein Benutzer "lisbeth", das Passwort wird abgefragt).  

Damit wir nicht immer "ghost.lisbeth.amnesia.de/ghost/" (der schließende Slash ist wichtig) aufrufen müssen, können wir noch eine Umleitung von http://[blog-url]/ghost erstellen. Dazu müssen wir zwischen `RewriteEngine On` und `RewriteRule` folgende zwei Zeilen ergänzen:
    RewriteCond %{REQUEST_URI} "/ghost"
    RewriteRule ^(.*) http://ghost.lisbeth.amnesia.uberspace.de/ghost/ [L,R=301] [L]

Wenn wir nun http://lisbeth.amnesia.uberspace.de/ghost aufrufen und mit dem richtigen Passwort eingeloggt sind, werden wir automatisch zum Backend umgeleitet.  
Fertig! Auf das Blog kann nun nur noch mit Passwort zugegriffen werden, während das Einloggen ins Backend ohne Probleme möglich ist.
