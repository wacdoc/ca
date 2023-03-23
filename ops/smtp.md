# Creeu el vostre propi servidor d'enviament de correu SMTP

## preàmbul

SMTP pot comprar directament serveis de proveïdors de núvol, com ara:

* [Amazon SES SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [Ali núvol correu electrònic push](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

També podeu crear el vostre propi servidor de correu: enviament il·limitat, baix cost global.

A continuació, mostrem pas a pas com construir el nostre propi servidor de correu.

## Selecció del servidor

El servidor SMTP autoallotjat requereix una IP pública amb els ports 25, 456 i 587 oberts.

Els núvols públics d'ús habitual han bloquejat aquests ports de manera predeterminada i és possible que es puguin obrir emetent una ordre de treball, però després de tot és molt problemàtic.

Recomano comprar a un amfitrió que tingui aquests ports oberts i admeti la configuració de noms de domini inversos.

Aquí us recomano [Contabo](https://contabo.com) .

Contabo és un proveïdor d'allotjament amb seu a Munic, Alemanya, fundat l'any 2003 amb preus molt competitius.

Si trieu l'euro com a moneda de compra, el preu serà més barat (un servidor amb 8 GB de memòria i 4 CPU costa uns 529 iuans a l'any, i la tarifa d'instal·lació inicial és gratuïta durant un any).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

Quan feu una comanda, comenteu que `prefer AMD` i el servidor amb CPU AMD tindrà un millor rendiment.

A continuació, prendré el VPS de Contabo com a exemple per demostrar com crear el vostre propi servidor de correu.

## Configuració del sistema Ubuntu

El sistema operatiu aquí és Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

Si el servidor a ssh mostra `Welcome to TinyCore 13!` (tal com es mostra a la figura següent), vol dir que el sistema encara no s'ha instal·lat. Si us plau, desconnecteu ssh i espereu uns minuts per iniciar sessió de nou.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

Quan aparegui `Welcome to Ubuntu 22.04.1 LTS` , la inicialització s'ha completat i podeu continuar amb els passos següents.

### [Opcional] Inicialitzeu l'entorn de desenvolupament

Aquest pas és opcional.

Per comoditat, he posat la instal·lació i la configuració del sistema del programari d'ubuntu a [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) .

Executeu l'ordre següent per instal·lar-lo amb un sol clic.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

Usuaris xinesos, utilitzeu l'ordre següent, i l'idioma, la zona horària, etc. s'establiran automàticament.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo habilita IPV6

Habiliteu IPV6 perquè SMTP també pugui enviar correus electrònics amb adreces IPV6.

editeu `/etc/sysctl.conf`

Modifiqueu o afegiu les línies següents

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Seguiu amb [el tutorial de contabo: Afegir connectivitat IPv6 al vostre servidor](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

Editeu `/etc/netplan/01-netcfg.yaml` , afegiu-hi unes quantes línies com es mostra a la figura següent (el fitxer de configuració predeterminat de Contabo VPS ja té aquestes línies, només heu de deixar de comentar-les).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

A continuació `netplan apply` per fer efectiva la configuració modificada.

Després de la configuració correcta, podeu utilitzar `curl 6.ipw.cn` per veure l'adreça ipv6 de la vostra xarxa externa.

## Clonar les operacions del dipòsit de configuració

```
git clone https://github.com/wactax/ops.soft.git
```

## Genereu un certificat SSL gratuït per al vostre nom de domini

L'enviament de correu requereix un certificat SSL per a l'encriptació i la signatura.

Utilitzem [acme.sh](https://github.com/acmesh-official/acme.sh) per generar certificats.

acme.sh és una eina de signatura de certificats automatitzada de codi obert,

Introduïu el magatzem de configuració ops.soft, executeu `./ssl.sh` i es crearà una carpeta `conf` **al directori superior** .

Trobeu el vostre proveïdor de DNS a [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) , editeu `conf/conf.sh` .

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

A continuació, executeu `./ssl.sh 123.com` per generar certificats `123.com` i `*.123.com` per al vostre nom de domini.

La primera execució instal·larà automàticament [acme.sh](https://github.com/acmesh-official/acme.sh) i afegirà una tasca programada per a la renovació automàtica. Podeu veure `crontab -l` , hi ha una línia com la següent.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

El camí per al certificat generat és com `/mnt/www/.acme.sh/123.com_ecc。`

La renovació del certificat cridarà a l'script `conf/reload/123.com.sh` , editeu aquest script, podeu afegir ordres com `nginx -s reload` per actualitzar la memòria cau del certificat d'aplicacions relacionades.

## Creeu un servidor SMTP amb chasquid

[chasquid](https://github.com/albertito/chasquid) és un servidor SMTP de codi obert escrit en llenguatge Go.

Com a substitut dels antics programes de servidor de correu com Postfix i Sendmail, chasquid és més senzill i fàcil d'utilitzar, i també és més fàcil per al desenvolupament secundari.

Executeu `./chasquid/init.sh 123.com` s'instal·larà automàticament amb un sol clic (substituïu 123.com pel vostre nom de domini d'enviament).

## Configura la signatura de correu electrònic DKIM

DKIM s'utilitza per enviar signatures de correu electrònic per evitar que les cartes siguin tractades com a correu brossa.

Després que l'ordre s'executi correctament, se us demanarà que establiu el registre DKIM (com es mostra a continuació).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

Només cal que afegiu un registre TXT al vostre DNS (com es mostra a continuació).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## Veure l'estat del servei i els registres

 `systemctl status chasquid` Veure l'estat del servei.

L'estat de funcionament normal és el que es mostra a la figura següent

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` o `journalctl -xeu chasquid` poden veure el registre d'errors.

## Configuració inversa del nom de domini

El nom de domini invers és per permetre que l'adreça IP es resolgui amb el nom de domini corresponent.

L'establiment d'un nom de domini invers pot evitar que els correus electrònics s'identifiquin com a correu brossa.

Quan es rep el correu, el servidor receptor realitzarà una anàlisi inversa del nom de domini a l'adreça IP del servidor d'enviament per confirmar si el servidor d'enviament té un nom de domini invers vàlid.

Si el servidor d'enviament no té un nom de domini invers o si el nom de domini invers no coincideix amb l'adreça IP del servidor d'enviament, el servidor receptor pot reconèixer el correu electrònic com a correu brossa o rebutjar-lo.

Visiteu [https://my.contabo.com/rdns](https://my.contabo.com/rdns) i configureu com es mostra a continuació

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

Després de configurar el nom de domini invers, recordeu configurar la resolució directa del nom de domini ipv4 i ipv6 al servidor.

## Editeu el nom d'amfitrió de chasquid.conf

Modifiqueu `conf/chasquid/chasquid.conf` al valor del nom de domini invers.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

A continuació, executeu `systemctl restart chasquid` per reiniciar el servei.

## Còpia de seguretat de conf al repositori git

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

Per exemple, faig una còpia de seguretat de la carpeta conf al meu propi procés github de la següent manera

Creeu primer un magatzem privat

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

Introduïu el directori conf i envieu-lo al magatzem

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## Afegeix remitent

correr

```
chasquid-util user-add i@wac.tax
```

Es pot afegir un remitent

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### Comproveu que la contrasenya estigui configurada correctament

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

Després d'afegir l'usuari, s'actualitzarà `chasquid/domains/wac.tax/users` , recordeu enviar-lo al magatzem.

## DNS afegeix un registre SPF

SPF (Sender Policy Framework) és una tecnologia de verificació de correu electrònic utilitzada per prevenir el frau de correu electrònic.

Verifica la identitat d'un remitent de correu comprovant que l'adreça IP del remitent coincideix amb els registres DNS del nom de domini que diu ser, evitant que els estafadors enviïn correus electrònics falsos.

Afegir registres SPF pot evitar que els correus electrònics s'identifiquin com a correu brossa tant com sigui possible.

Si el vostre servidor de noms de domini no admet el tipus SPF, només cal que afegiu un registre de tipus TXT.

Per exemple, el SPF de `wac.tax` és el següent

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

SPF per `_spf.wac.tax`

`v=spf1 a:smtp.wac.tax ~all`

Tingueu en compte que aquí he `include:_spf.google.com` , perquè més endavant configuraré `i@wac.tax` com a adreça d'enviament a la bústia de Google.

## Configuració DNS DMARC

DMARC és l'abreviatura de (Domain-based Message Authentication, Reporting & Conformance).

S'utilitza per capturar rebots SPF (potser causats per errors de configuració, o algú altre està fent veure que ets tu per enviar correu brossa).

Afegeix el registre TXT `_dmarc` ,

El contingut és el següent

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

El significat de cada paràmetre és el següent

### p (política)

Indica com s'han de gestionar els correus electrònics que fallan en la verificació SPF (Sender Policy Framework) o DKIM (DomainKeys Identified Mail). El paràmetre p es pot establir en un dels tres valors:

* cap: no es pren cap acció, només es retorna el resultat de la verificació al remitent mitjançant el mecanisme d'informes per correu electrònic.
* Quarantena: posa el correu que no ha passat la verificació a la carpeta de correu brossa, però no rebutjarà el correu directament.
* rebutjar: rebutja directament els correus electrònics que no superin la verificació.

### fo (Opcions de fallada)

Especifica la quantitat d'informació que retorna el mecanisme d'informes. Es pot establir en un dels valors següents:

* 0: informe dels resultats de la validació de tots els missatges
* 1: només informa dels missatges que no s'hagin verificat
* d: només informeu els errors de verificació del nom de domini
* s: només informa dels errors de verificació SPF
* l: només informa dels errors de verificació de DKIM

### rua & ruf

* rua (URI d'informes per a informes agregats): adreça de correu electrònic per rebre informes agregats
* ruf (URI d'informes per a informes forenses): adreça de correu electrònic per rebre informes detallats

## Afegiu registres MX per reenviar correus electrònics a Google Mail

Com que no he pogut trobar una bústia corporativa gratuïta que admeti adreces universals (Catch-All, pot rebre qualsevol correu electrònic enviat a aquest nom de domini, sense restriccions de prefixos), he utilitzat chasquid per reenviar tots els correus electrònics a la meva bústia de Gmail.

**Si teniu la vostra pròpia bústia de correu d'empresa de pagament, no modifiqueu el MX i ometeu aquest pas.**

Editeu `conf/chasquid/domains/wac.tax/aliases` , configureu la bústia de reenviament

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` indica tots els correus electrònics, `i` és el prefix de l'adreça de correu electrònic de l'usuari remitent creat anteriorment. Per reenviar el correu, cada usuari ha d'afegir una línia.

A continuació, afegiu el registre MX (apunto directament a l'adreça del nom de domini invers aquí, tal com es mostra a la primera línia de la figura següent).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

Un cop completada la configuració, podeu utilitzar altres adreces de correu electrònic per enviar correus electrònics a `i@wac.tax` i `any123@wac.tax` per veure si podeu rebre correus electrònics a Gmail.

Si no, comproveu el registre de chasquid ( `grep chasquid /var/log/syslog` ).

## Envieu un correu electrònic a i@wac.tax amb Google Mail

Després que Google Mail rebé el correu, esperava respondre amb `i@wac.tax` en comptes d'i.wac.tax@gmail.com.

Visiteu [https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) i feu clic a "Afegeix una altra adreça de correu electrònic".

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

A continuació, introduïu el codi de verificació rebut pel correu electrònic al qual s'ha reenviat.

Finalment, es pot configurar com a adreça de remitent predeterminada (juntament amb l'opció de respondre amb la mateixa adreça).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

D'aquesta manera, hem completat l'establiment del servidor de correu SMTP i al mateix temps fem servir Google Mail per enviar i rebre correus electrònics.

## Envieu un correu electrònic de prova per comprovar si la configuració és correcta

Introduïu `ops/chasquid`

Executeu `direnv allow` instal·lar dependències (direnv s'ha instal·lat en el procés d'inicialització d'una clau anterior i s'ha afegit un ganxo a l'intèrpret d'ordres)

després córrer

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

El significat dels paràmetres és el següent

* usuari: nom d'usuari SMTP
* pass: contrasenya SMTP
* a: destinatari

Podeu enviar un correu electrònic de prova.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

Es recomana utilitzar Gmail per rebre correus electrònics de prova per comprovar si les configuracions tenen èxit.

### Xifratge estàndard TLS

Com es mostra a la figura següent, hi ha aquest petit bloqueig, el que significa que el certificat SSL s'ha habilitat correctament.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

A continuació, feu clic a "Mostra el correu electrònic original"

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

Com es mostra a la figura següent, la pàgina de correu original de Gmail mostra DKIM, la qual cosa significa que la configuració de DKIM és correcta.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

Comproveu el Rebut a la capçalera del correu electrònic original i podreu veure que l'adreça del remitent és IPV6, la qual cosa significa que IPV6 també s'ha configurat correctament.
