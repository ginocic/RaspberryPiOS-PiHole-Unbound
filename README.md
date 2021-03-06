# Installazione e configurazione del sistema operativo Raspberry Pi OS, Pi-hole e Unbound
## Installare e configurare Raspberry Pi OS
Seguire quanto descritto [in questa guida](https://github.com/ginocic/Preparazione-Raspberry)

## Installazione e configurazione di Pi-hole
Per l'installazione e la configurazione di Pi-hole, ho seguito la [documentazione ufficiale](https://docs.pi-hole.net/).

### Prerequisiti
Riferimento alla documentazione ufficiale: [Prerequisites](https://docs.pi-hole.net/main/prerequisites/)

Applicare le seguenti regole al firewall (Raspberry Pi OS, di solito, installa di default IPTables)
```bash
sudo su -
iptables -I INPUT 1 -s 192.168.0.0/16 -p tcp -m tcp --dport 80 -j ACCEPT
iptables -I INPUT 1 -s 127.0.0.0/8 -p tcp -m tcp --dport 53 -j ACCEPT
iptables -I INPUT 1 -s 127.0.0.0/8 -p udp -m udp --dport 53 -j ACCEPT
iptables -I INPUT 1 -s 192.168.0.0/16 -p tcp -m tcp --dport 53 -j ACCEPT
iptables -I INPUT 1 -s 192.168.0.0/16 -p udp -m udp --dport 53 -j ACCEPT
iptables -I INPUT 1 -p udp --dport 67:68 --sport 67:68 -j ACCEPT
iptables -I INPUT 1 -p tcp -m tcp --dport 4711 -i lo -j ACCEPT
iptables -I INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
ip6tables -I INPUT -p udp -m udp --sport 546:547 --dport 546:547 -j ACCEPT
ip6tables -I INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
reboot
```

### Installazione
Riferimento alla documentazione ufficiale: [Installation](https://docs.pi-hole.net/main/basic-install/)

```bash
curl -sSL https://install.pi-hole.net | bash
pihole -a -p [NUOVA_PASSWORD]
```

Testare il corretto funzionamento di Pi-hole all'indirizzo "[NOME_HOST]/admin" o [INDIRIZZO_IP]/admin

### Aggiornamento
Riferimento alla documentazione ufficiale: [Updating](https://docs.pi-hole.net/main/update/)

Per effettuare l'aggiornamento, inserire il seguente comando

```bash
pihole -up
```

## Installazione e configurazione di Unbound
Riferimento alla documentazione ufficiale: [unbound](https://docs.pi-hole.net/guides/dns/unbound/)

### Installazione
```bash
sudo apt install unbound -y
```

### Configurazione
Creare il file di configurazione
```bash
sudo touch /etc/unbound/unbound.conf.d/pi-hole.conf
```

Copiare le righe seguenti nel file appena creato
```
server:
    # If no logfile is specified, syslog is used
    # logfile: "/var/log/unbound/unbound.log"
    verbosity: 0

    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes

    # May be set to yes if you have IPv6 connectivity
    do-ip6: no

    # You want to leave this to no unless you have *native* IPv6. With 6to4 and
    # Terredo tunnels your web browser should favor IPv4 for the same reasons
    prefer-ip6: no

    # Use this only when you downloaded the list of primary root servers!
    # If you use the default dns-root-data package, unbound will find it automatically
    #root-hints: "/var/lib/unbound/root.hints"

    # Trust glue only if it is within the server's authority
    harden-glue: yes

    # Require DNSSEC data for trust-anchored zones, if such data is absent, the zone becomes BOGUS
    harden-dnssec-stripped: yes

    # Don't use Capitalization randomization as it known to cause DNSSEC issues sometimes
    # see https://discourse.pi-hole.net/t/unbound-stubby-or-dnscrypt-proxy/9378 for further details
    use-caps-for-id: no

    # Reduce EDNS reassembly buffer size.
    # Suggested by the unbound man page to reduce fragmentation reassembly problems
    edns-buffer-size: 1472

    # Perform prefetching of close to expired message cache entries
    # This only applies to domains that have been frequently queried
    prefetch: yes

    # One thread should be sufficient, can be increased on beefy machines. In reality for most users running on small networks or on a single machine, it should be unnecessary to seek performance enhancement by increasing num-threads above 1.
    num-threads: 1

    # Ensure kernel buffer is large enough to not lose messages in traffic spikes
    so-rcvbuf: 1m

    # Ensure privacy of local IP ranges
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10
```

Riavviare Unbound e testare la risoluzione DNS
```bash
sudo service unbound restart 
dig pi-hole.net @127.0.0.1 -p 5335
```
La prima richiesta potrebbe essere abbastanza lenta ma le richieste successive, anche ad altri domini sotto lo stesso TLD, dovrebbero essere significativamente pi?? veloci.

### Test validazione
Si pu?? testare la validazione DNSSEC usando
```bash
dig sigfail.verteiltesysteme.net @127.0.0.1 -p 5335
```
Questo comando dovrebbe riportare lo status `SERVFAIL`

```bash
dig sigok.verteiltesysteme.net @127.0.0.1 -p 5335
```
Questo comando dovrebbe riportare lo status `NOERROR`

Se entrambi sono i comandi riportano gli status correttamente allora la validazione DNSSEC funziona.

### Configurazione di Pi-hole
* Accedere alla pagina **[NOME_HOST]/admin** oppure **[INDIRIZZO_IP]/admin**
* Cliccare su:
    - `Settings`
    - `DNS`
* Deselezionare gli tutti `Upstream DNS Servers` eventualmente presenti.
* Selezionare `Custom 1 (IPv4)`
* Aggiungere `127.0.0.1#5335` nella casella di testo
* Cliccare su `Save` (in fondo alla pagina)
* Aprire una sessione ssh oppure accedere al terminale:
  ```bash
  cat /etc/resolv.conf
  ```

Se il comando restituisce qualcosa diverso da
```
# Generated by resolvconf
nameserver 127.0.0.1
```
Allora eseguire i seguenti comandi
````bash
sudo systemctl disable unbound-resolvconf.service
sudo systemctl stop unbound-resolvconf.service
sudo systemctl restart dhcpcd
sudo systemctl enable unbound-resolvconf.service
sudo systemctl start unbound-resolvconf.service
````

Rieseguendo il comando `cat /etc/resolv.conf`, l'output dovrebbe essere:
```
# Generated by resolvconf
nameserver 127.0.0.1
```

# Note finali
Leggere la [documentazione ufficiale](https://docs.pi-hole.net/) per eventuali ulteriori informazioni su Pi-hole e Unbound. La guida ?? stata redatta molto bene. Semplice da seguire ma anche adeguatamente tecnica in modo da capire ci?? che si sta facendo.
