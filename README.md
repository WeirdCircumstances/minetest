# minetest

In dieser Anleitung geht es um eine technische Beschreibung, mit der eine aktuelle Minetest-Version auf einem Linux Server installiert werden kann.

Voraussetzungen:
- ein Linux Server (hier Debian oder Ubuntu)
- 1 CPU, 2 GB Ram
- curl
- wget
- git
- ufw
- rsync
- Zugang per ssh

Optional:
- ein Webserver (hier caddy)
- grafana
- telegraf

Minetest ist eine durch Minecraft inspirierte Spielengine. Ähnlich wie bei dem Vorbild ist die Grafik blockartig. Das Spiel kann auch durch Mods und Texture Packs erweitert werden.

Anders als Minecraft ist das Spiel Open Source und kann kostenfrei installiert werden.
Das Spiel läuft auf Windows, Linux, Mac und auf Android. Es ist sehr sparsam mit Systemressourcen und läuft auch auf älteren Geräten noch gut.
Auch die Anforderungen an den Server sind geringer. Minetest für das jumblr-Projekt läuft ohne Probleme auf einem Server mit zwei CPUs und 4 GB Ram für etwa 50 gleichzeitige Nutzer.

# Installation

Die Anleitung ist für eine aktuelle auf Debian basierte Distribution gültig.

Die von der Distribution verwaltete Minetest-Server Version kann mit folgendem Aufruf installiert werden:

```
sudo apt install minetest-server
```

Nach der Installaltion ist der Server bereit und kann eine erste Spielwelt hosten.
Der Aufruf für eine einzelne Welt ist

```
sudo systemctl start minetest-server.service
```

In dieser Anleitung geht es darum mehrere Minetest-Server parallel anzulegen und unanbhängig voneinader zu verwalten.

Dazu werden eigene systemd-Konfigurationen benötigt, eine für jeden Server.
Es wird eine neue Konfiguration angelegt mit

```
sudo nano /etc/systemd/system/30001.service
```

Diese bekommt folgenden Inhalt:

```
[Unit]
Description = Minetest Server on Port 30000
After=network-online.target
Wants=network-online.target systemd-networkd-wait-online.service

StartLimitIntervalSec=5
StartLimitBurst=5

[Service]
User=minetest
Group=minetest

Type=simple

Restart=on-failure
RestartSec=5s

ExecStart=/home/minetest/services/30000.sh

[Install]
WantedBy=multi-user.target
```

Mit dieser Konfiguration wird der Server mit dem Namen 30000.sh gestartet. Falls dabei ein Problem auftritt, wird es fünf mal alle fünf Sekunden versucht. So ist sichergestellt, dass der Service ausfallsicherer betrieben werden kann und trotzdem nach durchgehend immer wieder auftretenen Problemen irgendwann gestoppt wird.

Nach dem Anlegen oder der Modifikation einer systemctl Datei muss der passende Deamon neu gestartet werden, damit die Änderungen bekannt gemacht werden:

```
sudo systemctl daemon-reload
```

Der einzelne Minetest-Server wird nun gestoppt und für den Start deaktiviert:

```
sudo systemctl stop minetest-server.service
sudo systemctl disable minetest-server.service
```

Für dieses Projekt sollen mehrere Welten gleichzeitg gehostet werden. Der Prozess soll aus Sicherheitsgründen nur mit Benutzerrrechten laufen. Zusätzlich sollen einfach Backups, Zugriffe auf die Logs und Start- und Stop-Skrite möglich sein.

Dazu wird zunächst ein neuer Nutzer angelegt:

```
sudo adduser minetest
```

Nach der Einrichtung wird mit dem Befehl 

```
su minetest
```

zu diesem Benutzer gewechelt.
Mit dem Kommando

```
cd
```

kann schnell in das passende Home-Verzeichnis gewechselt werden (Kontrolle mit pwd).

Hier werden zunächst mehrere Verzeichnisse angelegt:

```
mkdir minetest services logs
```

Das minetest Verzeichnis wird alle Welten und die passenden Konfigurationen zum Minetest-Server enthalten. Services enthält alle Tools, mit denen der Server automatisiert neu gestartet werden kann, Backups erstellt werden und das Log gefüllt. Das Verzeichnis log schließlich enthält alle Logs der jeweiligen Server.

Zuerst werden die Service Dateien angelegt. Jeder Server bekommt dazu einen eindeutigen Namen. Per default startet der erste Server auf dem Port 30000, was sich gut als eindeutige Kennung eignet.

Jeder Server bekommt eine Datei nach folgendem Schema:

```
#!/bin/bash
/usr/lib/minetest/minetestserver \
--world ~/minetest/worlds/30000/ \
--config ~/minetest/worlds/30000/minetest.conf \
--port 30000 \
--logfile /home/minetest/log/30000.log
```

Die Datei wird passend benannt, zum Beispiel nach dem Port, über den die Welt angeboten wird. In dem Fall also 30000.sh
Jede weitere Welt wird einfach um ein erhöht. Die zweite Welt ist dann 300001.sh usw.

Als nächstes muss eine leere Log-Datei angelegt werden, in die zukünftig alle Meldungen geschrieben werden:

```
touch ~/log/30000.log
```

Die Server sollen automatisch starten, Backups erzeugen und Abends wieder herunter fahren. Dazu werden zwei Skripte erzeugt, die das in Zukunft für uns erledigen.

Das erste Skript erhält den Namen cron-start.sh und bekommt folgenden Inhalt:

```
#!/bin/bash

systemctl start 30000
systemctl start 30001
```

Das Start-Skript enthält schon einen zweiten Server, um zu demonstrieren, wie die Konfiguration dann aussehen sollte.

Das zweite Skript erhält den Namen cron-stop.sh und bekommt folgenden Inhalt:

```
#!/bin/bash
systemctl stop 30000
systemctl stop 30001

rsync -av /home/minetest/minetest/30000 /var/www/site/backups/
rsync -av /home/minetest/minetest/30001 /var/www/site/backups/

zip -r /var/www/site/backups/minetest-Port30000-$(date +%d.%m.%Y-%H:%M:%S).zip /var/www/site/backups/30000
zip -r /var/www/site/backups/modtest-Port30001-$(date +%d.%m.%Y-%H:%M:%S).zip /var/www/site/backups/30001

rm -r /var/www/site/backups/30000
rm -r /var/www/site/backups/30001
```

Das Skript stoppt beide Server, kopiert den Inhalt an einen anderen Ort (hier expemlarisch an einen Webserver mit den Namen site). Danach wird eine Zip-Datei aus jedem Backup mit dem aktuellen Datum und der Uhrzeit erstellt. Die Backups werden nachher über den Webserver verfügbar sein.
Die nicht komprimierten Backups werden anschließend wieder gelöscht.

Alle in dem Ordner services vorhandenen Dateien müssen nun als ausführbar markiert werden:

```
chmod +x 30000.sh cron-start.sh cron-stop.sh
```

Nun müssen noch die Ordner erstellt werden, in denen die Welten und die Konfiguration liegen sollen:

```
mkdir -r ~/minetest/worlds/30000
```

Der Prozess wiederholt sich für alle weiteren Welten.

Die Konfiguration einer jeden Welt wird in dem Ordner mit dem Namen der Welt unter ~/minetest/ abgelegt. In Fall der Welt 30000 also im Ordner ~/minetest/worlds/30000.
Die Konfiguration ist zentraler Bestandteil einer jeden Minetest-Welt. In ihr werden Mods und Rechte verwaltet.

Eine vollständige Konfiguration mit dem Namen minetest.conf sieht so aus:

```
enu_last_game = minetest
motd = Willkomen auf dem außerkultigen Server! Infos zum Projekt auf Kulti-rockt.de
#give_initial_stuff = true
enable_rollback_recording = true
deprecated_lua_api_handling = legacy
static_spawnpoint = -552, 9, -143
default_privs = shout,spawn,home,zoom,fast,interact,student
creative_mode = false
disallow_empty_password = true
default_password = DasStandardpasswort
enable_damage = true
language = de
#mg_name = flat

#PERFORMANCE
max_users = 120
#max_block_send_distance = 9
#block_send_optimize_distance = 4 ##ggf auskommentieren oder Wert erhöhen

name = kultiadmin
server_name = Der Außerkultige
server_description = Die Spielewelt für Außerkultige
mainmenu_last_selected_world = 1
port = 30000
enable_server = true
secure.trusted_mods = wiki
#load_mod_mg = true
#load_mod_mg_villages = true
#world_config_selected_mod = 51

server_announce = false
#ipv6_server = true
#serverlist_url = v6.servers.minetest.net
#bind_address = theebi.ignorelist.com
#curl_timeout = 5000
#debug_log_level = verbose
#das ist ein Kommentar
```

Zeilen, welche mit einer Raute (#) beginnen sind auskommentiert und werden nicht ausgeführt. Zu den einzelnen Details der Konfigurationsdatei empfiehlt es sich zusätzlich die Ressourcen den Minetest-Wikis zu studieren.
Wichtig sind vor allem die Zeilen

```
disallow_empty_password = true
default_password = DasStandardpasswort
port = 30000
server_announce = false
```

So wird verhindert, dass der Server ohne Passwort betreten werden kann, der Port definiert und auch ganz wichtig, der Server nicht in der offiziellen Liste aller Minetest-Server erscheint.

Hiermit ist die Konfiguration über den Benutzer minetest abgeschlossen. Es kann mit dem Befehl 

```
exit
```

in den root Account zurück gewechelt werden. Alternativ auch mit

```
sudo su
```

Es fehlen nun noch die Cron-Dienste, die den Start und Stopp der Server verwalten. Die Dienste werden mit dem Benutzer root bearbeitet:

```
sudo crontab -e
```

Wenn eine Abfrage kommt, welcher Editor verwendet werden soll, dann nano auswählen.
Die Datei wird nun am Ende um folgenden Inhalt ergänzt:

```
# sunday - thursday
30 22 * * 0-4 /home/minetest/services/cron-stop.sh

# friday - saturday
30 23 * * 5-6 /home/minetest/services/cron-stop.sh

 0  6 * * * /home/minetest/services/cron-start.sh
```

Der Server startet damit jeden Tag um 6 Uhr morgens. Normalerweise läuft er dann bis 22:30 Uhr, ausser Freitag und Samstags, dann bleibt er noch bis 23:30 Uhr online.

Nun fehlt nur noch den Server zu aktivieren mit

```
sudo systemctl enable 30000.service
```

Der Server kann nun auch zum Test gestartet oder gestoppt werden, normalerweise wird das in Zukunft per cron-Job gemacht.

```
sudo systemctl start 30000.service
sudo systemctl stop 30000.service
```

Die Minetest-Welt kann nun mit einem der verfügbaren Minetest-Clients besucht werden. Dazu wird die IP-Adresse oder besser ein Name des Servers benötigt. Die IP-Adresse erfährt man mit dem Befehl 

```
ifconfig
```

Die Adresse steht zu beginnt des Ausdrucks hinter der Bezeichnung inet.
Der Port entspricht dem aus der Konfiguration, hier also 30000.
Es kann ein Beutzername frei gewählt werden. Das Passwort ist zunächst das aus der Konfiguration ( hier default_password = DasStandardpasswort), kann und sollte aber später über den Einstellungsdialog aus Minetest heraus geändert werden.

Der Status der Spielinstanz ist sichtbar mit dem Befehl

```
tail -f /home/minetest/log/30000.log
```

Auftretene Probleme sind damit frühzeitig sichtbar, auch ohne dass es in den meisten Fällen schon zu sichtbaren Problemen führt.

Es empfiehlt sich von Zeit zu Zeit den Server per ssh zu besuchen und die Logdateien zu überprüfen und Systemaupdates mit der Befehlskette

```
apt update && apt full-upgrade && apt autoremove && apt autoclean
```

zu installieren.

Optional und richtig spannend wird es, wenn der Welt Mods (kurz für Modifikationen oder Erweiterungen) hinzugefügt werden. 
Hier ein Auszug der Datei world.mt

```
creative_mode = false
auth_backend = sqlite3
player_backend = sqlite3
gameid = minetest
enable_damage = true
backend = sqlite3
load_mod_default = false
load_mod_mobs_animal = 1
load_mod_weather = false
load_mod_boost_cart = false
load_mod_areas = 1
load_mod_farming = 1
load_mod_wildlife = 1
load_mod_mobs = 1
load_mod_mobkit = 1
load_mod_windmill = 1
load_mod_tubelib_addons2 = false
load_mod_tubelib_addons1 = false
load_mod_tubelib = false
load_mod_techpack_warehouse = false
load_mod_smartline = false
load_mod_sl_controller = false
load_mod_lcdlib = false
load_mod_technic_chests = false
load_mod_extranodes = false
load_mod_streets = 1
load_mod_street_signs = 1
load_mod_frame = 1
load_mod_fireflies = false
load_mod_tubelib_addons3 = false
load_mod_technic = false
load_mod_advtrains_train_track = false
load_mod_advtrains_train_japan = false
load_mod_concrete = false
load_mod_fachwerk = false
load_mod_mesecons_gates = 1
load_mod_mesecons_button = 1
load_mod_assets = false
load_mod_mesecons_extrawires = false
load_mod_advtrains_train_subway = false
load_mod_gravelsieve = false
load_mod_advtrains_line_automation = false
load_mod_mesecons = 1
load_mod_bridges = 1
load_mod_mesecons_receiver = 1
load_mod_advtrains_train_industrial = false
load_mod_advtrains_signals_ks = false
load_mod_advtrains_luaautomation = false
load_mod_safer_lua = false
load_mod_advtrains_itrainmap = false
load_mod_advtrains_interlocking = false
load_mod_mesecons_torch = 1
load_mod_technic_worldgen = false
load_mod_advtrains = false
load_mod_techpack_stairway = false
load_mod_food = false
load_mod_carpets = 1
load_mod_mesecons_stickyblocks = 1
load_mod_mesecons_materials = 1
load_mod_mesecons_insulated = 1
load_mod_mesecons_random = 1
load_mod_gates_long = false
load_mod_mesecons_hydroturbine = 1
load_mod_moretrees = false
load_mod_c_doors = 1
load_mod_hangglider = 1
load_mod_advtrains_train_steam = false
load_mod_ilights = 1
load_mod_mobs_humans = false
load_mod_mesecar = false
load_mod_mesecons_alias = 1
load_mod_mesecons_blinkyplant = 1
load_mod_mesecons_commandblock = 1
load_mod_mesecons_detector = 1
load_mod_mesecons_doors = 1
load_mod_technic_cnc = false
load_mod_mesecons_switch = 1
load_mod_mesecons_lightstone = 1
load_mod_mesecons_delayer = 1
load_mod_mesecons_fpga = 1
load_mod_mesecons_lamp = 1
load_mod_food_basic = false
load_mod_mesecons_luacontroller = 1
load_mod_mesecons_solarpanel = 1
load_mod_mesecons_microcontroller = 1
load_mod_wrench = false
load_mod_sailing_kit = 1
load_mod_mesecons_movestones = 1
load_mod_mesecons_mvps = 1
load_mod_water_life = 1
load_mod_mesecons_wires = 1
load_mod_digiterms = false
load_mod_mesecons_noteblock = 1
load_mod_mesecons_pistons = 1
load_mod_mesecons_pressureplates = 1
load_mod_mesecons_walllever = 1
load_mod_mokapi = 1
load_mod_petz = 1
load_mod_mesecons_powerplant = 1
load_mod_emote = false
load_mod_pontoons = 1
load_mod_mobs_monster = false
load_mod_tubelib2 = false
load_mod_ts_workshop = 1
load_mod_instant_ores = false
load_mod_steles = false
load_mod_signs_road = 1
load_mod_ontime_clocks = 1
load_mod_font_metro = 1
load_mod_font_api = 1
load_mod_display_api = 1
load_mod_basic_materials = 1
load_mod_signs_lib = 1
load_mod_signs = 1
load_mod_signs_api = 1
load_mod_digilines = false
load_mod_biome_lib = false
load_mod_boards = false
load_mod_pipeworks = 1
load_mod_unifieddyes = 1
load_mod_default = false
load_mod_vehicles = false
load_mod_cannons = false
load_mod_ma_pops_furniture = 1
load_mod_extra_doors = 1
load_mod_castle_weapons = false
load_mod_elevator = 1
```

Mods können mit true oder 1 aktiviert werden und 0 oder false deaktiviert werden. Mods können leider nicht direkt über den Server installiert werden. Dazu empfielt sich, die Mods mit dem Minetest-Client Programm herunterzuladen und dann auf den Server zu übertragen. Das Programm rsync empfielt sich dafür besonders. Nicht jede Kombination aus Mods funktioniert. Manchmal sind auch noch weitere Mods, genannt Abhängigkeiten notwendig, damit die Mods auch funktionieren. Eine genaue Analyse der Logdatei ist dazu notwendig. Manchmal zeigen sich Probleme auch erst im laufenden Betrieb.

Der Server sollte nun mit einer Firewall abgesichert werden.
Ganz wichtig, und ich schreibe das hier, weil das schon mehrfach vorgekommen ist, muss ssh zu dem Zeitpunkt bereits endgültig eingerichtet sein.
Der Dienst ssh ist am sichersten, wenn der Zugang per Passwort verboten ist und nur per Schlüssel möglich ist. 
Zur Einrichtung eignet sich die offizielle Anleitung sehr gut: https://www.ssh.com/ssh/key/

Der Firwall ufw müssen verschiedene Dienste hinzugefügt werden, bevor sie aktiviert werden darf.

Mit dem Aufruf

```
ufw allow ssh
```

wird der Zugang per ssh erlaubt werden. Das ist essentiell, da man sich sonst aus dem Server ausperrt und schlimmstenfalls alle Arbeit verloren geht.

Danach müssen noch weitere Dienste hinzugefügt werden

```
ufw allow 80
ufw allow 443
ufw allow 30000:30010
```

Die Argumente hinter allow stehen jeweils für einen Port (80 für http, 443 für https) oder für eine Reihe von Ports 30000:30010 also für alle Ports von 30000 bis 30010.

Die Firewall kann anschließend mit 

```
ufw enable
```

aktiviert werden. Danach sollte noch kurz überprüft werden, ob alle Dienste wie erwarte konfigurtiert wurden mit

```
ufw list
```

bzw indem die Dienste von aussen benutzt werden. Das heißt, funktioniert eine Websiete noch (sofern vorhanden) und ist Minetest noch erreichbar. Sollten hierbei Probleme auftreten, sollte die Konfiguration nocheinmal überprüft werden, bevor der Zugang per ssh beendet wird.

Hat alles bis hier hin geklappt, ist Minetest erfolgreich eingerichtet. 

Herzlichen Glückwunsch!

# Optionale Funktionen

Der Minetest-Server funktioniert nun mit einer oder mehreren Welten, je nach Konfiguration. Es kann sinnvoll sein auf dem Server zusätzliche Dienste anzubieten. So hat sich herausgestellt, dass es ganz gut ist eine kleine Webseite mit westenlichen Informationen bereitzustellen. Zum Beispiel wenn Mods hinzugefügt wurden oder wenn allgemeine Informationen nötig sind.

Hier wurde der super schlanke und trotzdem mächtige Webserver caddy benutzt. Er eignet sich, da automatisch TLS Zertifikate erzeugt werden und damit eine große Fehlerquelle eliminiert wird. Caddy versteht die Markdown-Language, eine beschreibene Sprache für Textformatierung. Auch dieses Dokument ist in Markdown geschrieben, so we alle Dokumente auf Github.

Zur Installation von Caddy empfielt es sich der sehr guten offiziellen Anleitung zu folgen: https://caddyserver.com/docs/install

Der Webserver caddy ist mit diesen Befehlen im Caddyfile (unter /var/www/site) bereits eingerichtet und voll funktionsfähig:

```
NameDerWebseite.de www.NameDerWebseite.de {
	encode zstd gzip

	templates
	file_server browse

	reverse_proxy /grafana/* localhost:3000
}

```

Damit ein Name angezeigt werden kann, muss vorher eine Domain gekauft werden und entsprechend eingerichtet sein, damit die Namensauflösung für die aktuelle IP-Adresse funktioniert. Die Einrichtung einer Domain hängt stark von Anbieter ab, der dann ensprechende Informationen zur Einrichtung bereithält.

Der Webserver benötigt nun nur noch die Dateien, die die Webseite bilden. Dafür sind nur noch zwei weitere Dateien nötig. 
Einmal die index.html 

```
{{$pathParts := splitList "/" .OriginalReq.URL.Path}}
{{$markdownFilename := default "index" (slice $pathParts 2 | join "/")}}
{{$markdownFilePath := printf "%s.md" $markdownFilename}}
{{$markdownFile := (include $markdownFilePath | splitFrontMatter)}}
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8" />
	<link id="pagestyle" rel="stylesheet" type="text/css" href="css/dark.css">
	<link rel="stylesheet" type="text/css" href="css/extra.css">
	<link rel="stylesheet" href="github-markdown.css">
	<script src="js/mdpage.js"></script>
        <meta http-equiv="X-UA-Compatible" content="IE=edge" />
        <title>Kulti-rockt! (.de)</title>
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <style>
	.markdown-body {
		box-sizing: border-box;
		min-width: 200px;
		max-width: 980px;
		margin: 0 auto;
		padding: 45px;
	}

	@media (max-width: 767px) {
		.markdown-body {
			padding: 15px;
		}
	}
        </style>
    </head>
    <body>

		{{markdown $markdownFile.Body}}
            </article>
	</main>
    </body>
</html>
```

Und einmal die index.md, die mit einem beliebigen Text gefüllt werden kann, der mit Markdown-Syntax interpretiert wird. 
Als Einführung zu Markdown und der Syntax ist https://www.markdowntutorial.com/ eine gute Quelle.

Wenn der Server noch Hintergrundinformationen über ein Dashboard dargestellen soll, dann eigent sich Grafana mit dem Telegraf. Die Einrichtung geht aber über die Möglichkeiten dieses Dokuments hinaus.

Eine Webseite mit caddy, Markdown, Grafana und Telegraf kann dann so aussehen:

https://kulti-rockt.de/
