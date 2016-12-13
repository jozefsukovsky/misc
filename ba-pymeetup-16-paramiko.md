<!-- $theme: default -->

Paramiko
========
# Bratislava Python Meetup #16
<small>Jozef Sukovský jozef@measuredsearch.com</small>

---

# Čo dokáže Paramiko
- implementácia SSH v2 protokolu v Pythone
- client-side funkcionalita
- server-side funkcionalita

# Kde nám pomôže
- automatizovanie opakovateľných úloh cez SSH
- bezpečný transfer súborov cez SSH
- tunelovanie spojení

Viac teórie na http://www.paramiko.org
<!-- page_number: true -->

---
# Príprava prostredia
```
mkdir meetup
cd meetup
virtualenv --no-site-packages env
source env/bin/activate
pip install paramiko==2.1.0
```

---
# Jednoduché pripojenie s heslom
```
import paramiko

client = paramiko.SSHClient()
client.load_system_host_keys()
try:
    client.connect(
        hostname='jump.0th.place',
        username='meetup',
        password='meetupsr123'
    )
    client.close()
except paramiko.ssh_exception.SSHException as exc:
    print exc.message
```
Vyvolá výnimku, pretože vzdialený host je pre nás neznámy (nie je v known_hosts ak máme OpenSSH)

---
```
import paramiko

client = paramiko.SSHClient()
client.load_system_host_keys()
client.set_missing_host_key_policy(paramiko.WarningPolicy())
try:
    client.connect(
        hostname='jump.0th.place',
        username='meetup',
        password='meetupsr123'
    )
    client.close()
except paramiko.ssh_exception.SSHException as exc:
    print exc.message
```

---
```
import paramiko

client = paramiko.SSHClient()
client.load_system_host_keys()
client.load_host_keys('/Users/jsu/.ssh/known_hosts')
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
try:
    client.connect(
        hostname='jump.0th.place',
        username='meetup',
        password='meetupsr123'
    )
    client.close()
except paramiko.ssh_exception.SSHException as exc:
    print exc.message
```
Bude akceptovať aj neznámy host vrátane ďalších pripojení recyklovaním client objektu

---
# Poznámky ku host kľúčom
- load_system_host_keys sa pozerá do konfigurácie GlobalKnownHostsFile, read-only
- load_host_keys prečíta parameter, do ktorého pri AutoAddPolicy nový kľúč zapíše
- zapisuje v ssh-rsa formáte, na macu (asi aj na iných) nevie pracovať s ecdsa
- - použite ľubovoľný súbor

---
```
import paramiko

client = paramiko.SSHClient()
client.load_system_host_keys()
client.load_host_keys('/Users/jsu/.ssh/known_hosts')
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
client.connect(
    hostname='jump.0th.place',
    username='meetup',
    password='meetupsr123'
)

stdin, stdout, stderr = client.exec_command('hostname')
print stdout.read().strip()
client.close()

```

---
# Stiahnutie kľúča cez SFTP
```
import os
import paramiko

client = paramiko.SSHClient()
client.load_system_host_keys()
client.load_host_keys('/Users/jsu/.ssh/known_hosts')
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
client.connect(
    hostname='jump.0th.place',
    username='meetup',
    password='meetupsr123'
)

ftp = client.open_sftp()
ftp.get('meetupkey', './meetupkey')
os.chmod('./meetupkey', 0600)
client.close()

```

---
# Pripojenie s kľúčom
```
import paramiko

client = paramiko.SSHClient()
client.load_system_host_keys()
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
client.connect(
    hostname='jump.0th.place',
    username='meetup',
    key_filename='meetupkey'
)

stdin, stdout, stderr = client.exec_command('hostname')
print stdout.read().strip()
client.close()
```

---
# Pripojenie cez Jumphost 1/2
```
import paramiko

client = paramiko.SSHClient()
client.load_system_host_keys()
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
client.connect(
    hostname='jump.0th.place',
    username='meetup',
    key_filename='meetupkey')
transport = client.get_transport()
dest_addr = ('internal.0th.place', 22)
local_addr = ('127.0.0.1', 22222)
channel = transport.open_channel(
    'direct-tcpip', dest_addr, local_addr)

# pokracovanie na dalsej strane
```

---
# Pripojenie cey Jumphost 2/2
```
# pokracovanie

internal = paramiko.SSHClient()
internal.load_system_host_keys()
internal.set_missing_host_key_policy(paramiko.AutoAddPolicy())
internal.connect(
    '127.0.0.1',
    port=22222,
    username='meetup',
    key_filename='meetupkey',
    sock=channel)
stdin, stdout, stderr = internal.exec_command('hostname')
print 'internal: %s' % stdout.read().strip()
stdin, stdout, stderr = client.exec_command('hostname')
print 'jump: %s' % stdout.read().strip()
internal.close()
client.close()
```
---
# Poznámky ku kanálom
- mali by ste mať AllowTcpForwarding na yes, inak to nepôjde
- najviac rozumov dá paramiko/demos/forward.py


---
# Interaktivita
- ak je možnosť, vyhnite sa jej použitím alternatívnej aplikácie
- ak je možnosť volať parametre, volajte ich (napr top -u namiesto u príkazu)
- ak sa nedá, snažte sa parametre odovzdať jednorázovo, napr.
```
$ zkCli.sh
delete /keys/exe/keyname
quit
```
nahraďte
```
<<EOF
delete /keys/exe/keyname
quit
EOF
```

---
# Príklad interakcie
```
# zrecyklujeme client objekt z minulosti
stdin, stdout, stderr = client.exec_command(
    'top', get_pty=True)
stdin.write('q')
stdin.flush()
print stdout.read().strip()
```
V praxi nemá veľký význam, ilustruje štandardný vstup a výstup

---
# Čo ak nedostaneme EOF
```
stdin, stdout, stderr = client.exec_command('top', get_pty=True)
stdin.write('uuuuuuuu\n\n')
stdin.write('meetup\n')
stdin.flush()
endtime = time() + 10
while not stdout.channel.eof_received:
    sleep(1)
    if time() > endtime:
        stdout.channel.close()
        break
print stdout.read()

```

---
# Poznámky k interaktivite
- niektoré aplikácie, najmä očakávajúce presmerovanie výstupu do súboru aj svoj stdout smerujú do stderr
- niektoré aplikácie nepošlú eof a kanál zostane visieť
- ak je možnosť, naozaj sa jej vyhnite

---
Pre pokročilejšie príklady navštívte priamo veľmi dobrú dokumentáciu k paramiko alebo/a príklady na https://github.com/paramiko/paramiko/tree/master/paramiko/demos
