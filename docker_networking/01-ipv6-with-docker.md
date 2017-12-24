# todo: translate to english

## original german text from my blog post https://www.puzzle.ch/blog/articles/2017/06/13/docker-container-mit-ipv6-anbinden 


Die aktuelle Verbreitung von IPv6 liegt laut Google <a href="https://www.google.com/intl/en/ipv6/statistics.html#tab=per-country-ipv6-adoption&amp;tab=per-country-ipv6-adoption">in der Schweiz bei 27.4%</a>. Da sollte man sich als Betreiber einer Webseite oder eines Cloud-Dienstes schon langsam Gedanken darüber machen, wie man seine Services auch per IPv6 an seine Kunden bringt.

Ich zeige in diesem Artikel auf, wie man Docker vorbereitet und dann seinen Containern eine IPv6-Adresse verpasst. Alle gezeigten Befehle sollten auf einem Ubuntu 16.04 funktionieren. Bei anderen Distributionen und Versionen können die zu editierenden Dateien und auszuführenden Befehle natürlich unterschiedlich sein!<!--more-->
<h2>Voraussetzungen</h2>
<h3>Server mit geroutetem IPv6-Netzwerk</h3>
Natürlich muss man zuerst einmal einen Server in Betrieb haben, der schon per IPv6 angebunden ist. Das heisst, einerseits muss der Server-Provider das unterstützen und andererseits muss der Dual-Stack-Betrieb im Betriebssystem aktiviert sein.

Am einfachsten findet man das mit ifconfig heraus. Wenn bei der Ausgabe etwas Ähnliches wie hier zu sehen ist, dann scheint alles bereit zu sein:
<pre class="">root@dockerhost /home/docker # ifconfig
eth0 Link encap:Ethernet HWaddr 90:1b:0e:00:aa:bb 
 inet addr:88.99.11.22 Bcast:88.99.11.255 Mask:255.255.255.255
 inet6 addr: fe80::921b:eff:febd:f718/64 Scope:Link
 inet6 addr: 2a01:4f8:a:b::2/64 Scope:Global</pre>
Wichtig ist hier die Adresse mit dem Scope <em><strong>Global</strong></em>. Adressen, die mit fe80 oder fd00 beginnen oder den Scope <em>Link</em> haben, sind lokale Adressen, mit denen man nicht vom Internet aus erreichbar ist. Im obigen Beispiel sehen wir, dass wir ein ganzes /64-Netz zur Verfügung haben, nämlich das <strong><em>2a01:4f8:a:b::/64</em></strong>.

Ob die Adresse auch korrekt funktioniert, findet man am einfachsten mit einem Ping heraus:
<pre class="">root@dockerhost /home/docker # ping6 -n -c4 google.com
PING google.com(2a00:1450:4001:811::200e) 56 data bytes
64 bytes from 2a00:1450:4001:811::200e: icmp_seq=1 ttl=56 time=5.19 ms
64 bytes from 2a00:1450:4001:811::200e: icmp_seq=2 ttl=56 time=5.19 ms
64 bytes from 2a00:1450:4001:811::200e: icmp_seq=3 ttl=56 time=5.19 ms
64 bytes from 2a00:1450:4001:811::200e: icmp_seq=4 ttl=56 time=5.20 ms

--- google.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3003ms
rtt min/avg/max/mdev = 5.194/5.196/5.201/0.088 ms</pre>
<h3>Aktuelle Docker-Version</h3>
Es muss mindestens die Version 1.10 von Docker installiert sein. Erst diese unterstützt es, eigene IP-Adressen beim Erstellen eines Containers anzugeben.
<h2>Docker vorbereiten</h2>
Folgende Schritte müssen wir durchführen, damit die Docker Engine was mit IPv6 anfangen kann:
<ul>
 	<li>Route für Interface <em>docker0</em> hinzufügen:
<pre class="">ip -6 route add 2a01:4f8:a:b::/64 dev docker0</pre>
<ul>
 	<li>Wir routen damit allen Traffic, der über IPv6 an das Netz <strong><em>2a01:4f8:a:b::/64</em></strong> kommt
an das Interface <em>docker0</em> weiter</li>
</ul>
</li>
 	<li>Forwarding von IPv6-Paketen im Kernel  aktivieren:
<pre class="">sysctl net.ipv6.conf.default.forwarding=1
sysctl net.ipv6.conf.all.forwarding=1
</pre>
</li>
 	<li>Beim Docker-Daemon IPv6 aktivieren und ihm das Subnetz zuweisen. Dafür editiert man die Datei /etc/default/docker
wie folgt:
<pre class=""># Use DOCKER_OPTS to modify the daemon startup options.
DOCKER_OPTS="--dns 8.8.8.8 --dns 8.8.4.4 --ipv6 --fixed-cidr-v6='2a01:4f8:a:b::/64'"</pre>
</li>
 	<li>Restart des Docker-Daemon:
<pre class="">systemctl docker restart</pre>
</li>
 	<li>Ein Docker-Netzwerk erstellen, das IPv6 aktiviert hat:
<pre class="">docker network create --driver bridge --ipv6 --subnet=2a01:4f8:a:b:c::/80 ipv6</pre>
<ul>
 	<li>Damit erstellen wir ein Docker-Netzwerk mit dem Namen <em>ipv6</em> und dem Subnetz <strong><em>2a01:4f8:a:b:c::/80</em></strong></li>
 	<li>Eventuell wollen wir mehrere Docker-Netzwerke mit unterschiedlichen Subnetzen machen. Darum wählen wir eine
Netzlänge von <strong>/80</strong> und den Suffix <strong>c</strong> für dieses Subnetz. Für weitere Netze
würden wir einfach einen anderen Suffix benutzen.</li>
 	<li>Den Teil <em>c</em> können wir frei wählen. Theoretisch können wir auch noch weitere Bytes des Subnetzes
bestimmen, dann muss aber auch die Länge nach dem / angepasst werden</li>
</ul>
</li>
</ul>
Sofern das alles geklappt hat, sollte der Befehl <strong>docker network inspect</strong> nun etwas in der Art liefern:
<pre class="">root@dockerhost /home/docker # docker network inspect ipv6
[{
 "Name": "ipv6",
 "Id": "8777534b045364232f1b9706172ed0ac8c68ce5c60bb2ab538994858e4c1b2e4",
 "Created": "2017-05-12T09:15:59.367836503+02:00",
 "Scope": "local",
 "Driver": "bridge",
 "EnableIPv6": true,
 "IPAM": {
 "Driver": "default",
 "Options": {},
 "Config": [{
     "Subnet": "172.19.0.0/16",
     "Gateway": "172.19.0.1"
   },
   {
     "Subnet": "2a01:4f8:a:b:c::/80"
   }
 ]},
 "Internal": false,
 "Attachable": false,
 "Ingress": false,
 "Containers": {},
 "Options": {},
 "Labels": {}
}]</pre>
<h2>Container dem Netzwerk hinzufügen</h2>
Zum Schluss können wir einen laufenden Container dem soeben erstellten Netzwerk hinzufügen:
<pre class="">docker network connect --ip6 '2a01:4f8:a:b:c::5' ipv6 [container_name]
</pre>
<p class="">Damit wird der Container mit dem Namen [<em>container_name]</em> unter der Adresse <em>2a01:4f8:a:b:c::5</em> erreichbar sein.</p>
<p class="">Die Voraussetzung dafür ist natürlich, dass die Software im Container auch mit IPv6 umgehen kann. Diese muss nämlich auch über IPv6 auf dem Interface zuhören. Falls man eine Listen-Address konfigurieren muss, dann gibt man dort [::] ein, was das IPv6-Äquivalent zu 0.0.0.0 ist. Am besten konsultiert man die Dokumentation des Services, den man laufen lassen möchte. Oft ist es auch nur eine Einstellung in einer Konfigurationsdatei.</p>
<p class="">Muss ein Container nicht über IPv4 erreichbar und damit nicht im default-Netzwerk hinzugefügt sein, kann man auch direkt beim Start des Containers das Netzwerk konfigurieren:</p>

<pre class="">docker run \
 -d \
 -p 51472:51472 \
......
 --network ipv6 \
 --ip6 '2a01:4f8:a:b:c::5' \
 --name [container_name] \
 [image_name]</pre>
<p class="">Um zu prüfen, ob das Hinzufügen geklappt hat, kann man wiederum den Befehl <strong>docker network inspect</strong> benutzen.</p>
<p class="">Ob der Service danach auch wirklich über IPv6 erreichbar ist, prüft man am besten mit einem Tool wie <a href="http://ipv6-test.com/validate.php">ipv6-test.com</a></p>

<h3>Edit (20.06.2017):</h3>
Wie im Kommentar von Roland korrekt erwähnt wurde, ergeben sich durch IPv6 mit Docker neue Herausforderungen, was Security und Wartbarkeit betrifft.

Der Hauptgrund dafür ist das Fehlen des NAT-Supports für IPv6 in Docker. Beziehungsweise ist ja NAT nur eine Krücke, um das Problem der beschränkten Anzahl öffentlicher Adressen in IPv4 einzuschränken. Deshalb wird NAT möglicherweise gar nie nativ in Docker eingebaut werden für IPv6 (siehe dazu <a href="https://github.com/moby/moby/issues/25407">dieses Issue auf Github</a>).

Was bedeutet das Fehlen von IPv6-NAT nun für meine Container?
<ul>
 	<li>Bisher ist man sich von Docker gewohnt, dass nur die Ports, die man ganz explizit (also mit <strong><em>-p xxx:xxx</em></strong> oder <em><strong>-P</strong></em>) freischaltet/weiterleitet auch von aussen erreichbar sind. Bei IPv6 ist dies nicht so. Gibt man einem Container eine öffentliche IPv6-Adresse, dann sind standardmässig <strong>alle (!!!) Ports</strong>, die der Container öffnet und an diese Adresse bindet, von aussen erreichbar. Also nicht nur die, die im EXPOSE-Statement des Dockerfiles angegeben sind, sondern alle, die der Prozess öffnet.
Will man also die gleiche Sicherheit wie NAT, dann muss man noch explizit Firewall-Rules definieren, welche die gleiche Isolation des Containers bewirken.</li>
 	<li>Ein weiterer Nachteil gegenüber NAT ist das Handling der DNS-Einträge. Bisher war es einem relativ egal, welche Adresse ein Container intern genau gekriegt hatte. Im DNS trug man sowieso einfach die öffentliche IP des Docker-Hosts ein. Da mit v6 aber jeder Container eine öffentliche Adresse kriegt, muss diese im DNS eingetragen werden. Und damit sie nicht bei jedem Neuanlegen des Containers ändert, will man sie dem Container statisch zuordnen. Das heisst, man muss sich auch da wieder deutlich mehr Gedanken machen und es funktioniert nicht einfach alles "out of the box" wie man es sich bisher gewohnt war.</li>
</ul>
Wie genau diese Herausforderungen am besten angegangen werden, wird sich zeigen. Es gibt beispielsweise Projekte, die versuchen die gleiche <a href="https://github.com/robbertkl/docker-ipv6nat">NAT-Funktionalität für IPv6</a> zu implementieren.