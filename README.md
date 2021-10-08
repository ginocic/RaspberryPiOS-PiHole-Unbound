# Installazione e configurazione del sistema operativo Raspberry Pi OS, Pi-Hole e Unbound
## Installare e configurare Raspberry Pi OS
Seguire quanto descritto [in questa guida](https://gist.github.com/ginocic/6c1f6e845266ac262f8b532d7405ddc7)

## Installazione e configurazione di Pi-Hole
Per l'installazione e la configurazione di Pi-Hole, ho seguito la [documentazione ufficiale](https://docs.pi-hole.net/).

### Prerequisiti
Riferimento alla documentazione ufficiale: [Prerequisites](https://docs.pi-hole.net/main/prerequisites/)

### Applicare le seguenti regole al firewall (Raspberry Pi OS, di solito, installa di default IPTables)
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
