# FTPS load balancing op BIG-IP

Configuratie van een virtual server die FTPS-verkeer proxiet én load balancet over meerdere
backend-servers, inclusief de aandachtspunten die specifiek bij FTP(S) load balancing komen kijken.

---

## Inhoud

- [Explicit vs implicit FTPS](#explicit-vs-implicit-ftps)
- [Waarom een FTP-profiel](#waarom-een-ftp-profiel)
- [De kern: persistence](#de-kern-persistence)
- [Configuratie (tmsh)](#configuratie-tmsh)
- [Aandachtspunten](#aandachtspunten)
- [Offload vs re-encrypt](#offload-vs-re-encrypt)

---

## Explicit vs implicit FTPS

"FTPS" is dubbelzinnig. De configuratie hieronder gaat uit van **explicit FTPS**.

| Variant | Beschrijving | BIG-IP gedrag |
|---|---|---|
| **Explicit FTPS** (FTPES, AUTH TLS) | Client verbindt plain op poort 21 en doet daarna `AUTH TLS` om te upgraden. De gangbare variant. | FTP-profiel kan in de control channel meekijken. |
| **Implicit FTPS** | TLS meteen vanaf de start, klassiek op poort 990. | BIG-IP kan **niet** meekijken (alles versleuteld vanaf byte 1). Terugvallen op pure L4-loadbalancing met alleen source-persistence. |

---

## Waarom een FTP-profiel

FTP opent voor elke transfer een tweede verbinding — de **data channel** — op een ander, dynamisch
poortnummer (active of passive). Het FTP-profiel:

- leest de control channel mee;
- ziet de `PASV`/`227`- of `EPSV`/`229`-respons;
- opent automatisch de juiste pinhole;
- stuurt die datasessie naar **dezelfde poolmember** als de control channel.

Zonder dit profiel breekt FTP achter een load balancer.

---

## De kern: persistence

Het belangrijkste punt bij load balancing. De data channel **moet** naar dezelfde backend-server
als de control channel, anders mislukt de transfer. Daarom: **source address persistence** op de pool.

Het FTP-profiel koppelt data- aan control channel binnen één sessie; source-persistence borgt dat
opeenvolgende verbindingen van dezelfde client consistent op dezelfde backend landen — belangrijk bij
clients die meerdere control channels openen.

---

## Configuratie (tmsh)

### Pool met meerdere FTPS-backends

```tcl
ltm pool ftps_pool {
    members {
        10.10.20.11:21 { address 10.10.20.11 }
        10.10.20.12:21 { address 10.10.20.12 }
        10.10.20.13:21 { address 10.10.20.13 }
    }
    monitor ftp
    load-balancing-mode least-connections-member
}
```

### Source address persistence (cruciaal)

```tcl
ltm persistence source-addr ftps_source_persist {
    timeout 600
    # match-across-services aanzetten als control- en datachannel
    # als losse virtuals worden gezien
    match-across-services enabled
    # persistence-records spiegelen naar de standby unit (HA failover)
    mirror enabled
}
```

### Client SSL profiel (TLS van de client termineren)

```tcl
ltm profile client-ssl ftps_clientssl {
    cert-key-chain {
        default {
            cert /Common/ftps.example.com.crt
            key /Common/ftps.example.com.key
            chain /Common/ca-bundle.crt
        }
    }
    ciphers DEFAULT
}
```

### Server SSL profiel (opnieuw TLS naar de backend)

```tcl
ltm profile server-ssl ftps_serverssl {
    ciphers DEFAULT
}
```

### FTP-profiel

```tcl
ltm profile ftp ftps_profile {
    translate-extended enabled
}
```

### Virtual server

```tcl
ltm virtual vs_ftps {
    destination 192.0.2.50:21
    ip-protocol tcp
    profiles {
        tcp { }
        ftps_profile { }
        ftps_clientssl {
            context clientside
        }
        ftps_serverssl {
            context serverside
        }
    }
    pool ftps_pool
    persist {
        ftps_source_persist { }
    }
    source-address-translation {
        type automap
    }
    translate-address enabled
    translate-port enabled
}
```

---

## Aandachtspunten

### SNAT/AutoMap is hier bijna verplicht
Met meerdere backends en een data channel die teruggeopend wordt, wil je dat al het verkeer via de
self-IP van de BIG-IP loopt, zodat return traffic gegarandeerd terugkomt. Zonder SNAT moet de default
gateway van **elke** poolmember de BIG-IP zijn — foutgevoelig bij meerdere servers.

### Passive mode + backend port range
In passive mode bepaalt de **backend** het datapoort-bereik. Dat bereik moet open staan tussen BIG-IP
en de poolmembers. Het FTP-profiel volgt de poort die de server adverteert, dus zorg dat de firewall
tussen BIG-IP en pool dat range toelaat. Bij re-encryptie naar de backend leest BIG-IP de `227`-respons
binnen de TLS-sessie — dat werkt zolang het FTP-profiel actief is.

### Health monitor
Gebruik de ingebouwde `ftp`-monitor (eventueel met username/password en een op te halen bestand), zodat
een backend die wel op poort 21 luistert maar geen geldige FTP-sessie aankan, netjes uit de pool valt.

---

## Offload vs re-encrypt

| Keuze | Hoe | Wanneer |
|---|---|---|
| **TLS offload** | Laat het `ftps_serverssl`-profiel weg; BIG-IP stuurt plain FTP naar de backends. | Schaalt het mooist, backends hoeven geen certificaten te beheren. Alleen acceptabel als het netwerk tussen BIG-IP en pool vertrouwd is. |
| **Re-encrypt** | Houd het `ftps_serverssl`-profiel zoals hierboven; end-to-end versleuteld. | Iets zwaarder, maar data blijft ook intern versleuteld. |

> Het voorbeeld hierboven gaat uit van **explicit FTPS met re-encryptie**. Draai je implicit FTPS op 990,
> dan kan het FTP-profiel niet meekijken en moet je naar een L4-aanpak met alleen source-persistence.
