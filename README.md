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
- [Implicit FTPS (L4-variant)](#implicit-ftps-l4-variant)

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

---

## Implicit FTPS (L4-variant)

Bij implicit FTPS is de verbinding **vanaf byte 1 versleuteld** (klassiek op poort 990). BIG-IP kan dan
niet in de control channel meekijken, dus:

- **Geen FTP-profiel** — er valt niets te lezen, de `227`/`229`-responses zitten in de TLS-stroom.
- **Geen automatische data-channel pinholes** — BIG-IP weet niet welke datapoort de server adverteert.
- **Geen client-/server-SSL profielen** als je puur op L4 passthrough doet — de TLS blijft end-to-end
  tussen client en backend, BIG-IP raakt de payload niet aan.

Het wordt daarmee een **L4 passthrough met source-persistence**. De load balancing werkt nog, maar de
"intelligentie" rond de data channel valt weg. Dat heeft consequenties voor de poortconfiguratie.

### Aanpak

1. **Control channel op 990** load balancen met source-persistence.
2. **Data channel** afdwingen via een **passive port range** op de backends, en dat range óók als
   virtual server(s) op de BIG-IP publiceren met dezelfde persistence. Source-persistence zorgt dat de
   datasessie bij dezelfde backend als de control channel landt.
3. Op de FTP-servers het passive range vastzetten (bijv. `40000–40100`) en die `PASV`-respons het
   **externe** (virtual server) adres laten adverteren — anders krijgt de client een intern IP terug.

### Configuratie (tmsh)

```tcl
# Pool — health monitor op tcp (geen ftp-monitor, want geen plain control channel)
ltm pool ftps_implicit_pool {
    members {
        10.10.20.11:990 { address 10.10.20.11 }
        10.10.20.12:990 { address 10.10.20.12 }
        10.10.20.13:990 { address 10.10.20.13 }
    }
    monitor tcp
    load-balancing-mode least-connections-member
}

# Source-persistence — bindt control- én datachannel aan dezelfde backend.
# match-across-virtuals zodat de losse data-VS dezelfde persistence-record gebruikt.
ltm persistence source-addr ftps_implicit_persist {
    timeout 600
    match-across-virtuals enabled
    match-across-services enabled
    mirror enabled
}

# Control channel: poort 990, pure L4 passthrough (alleen tcp-profiel)
ltm virtual vs_ftps_implicit_control {
    destination 192.0.2.50:990
    ip-protocol tcp
    profiles {
        tcp { }
    }
    pool ftps_implicit_pool
    persist {
        ftps_implicit_persist { }
    }
    source-address-translation {
        type automap
    }
    mirror enabled
}

# Data channel: passive port range, zelfde pool, zelfde persistence
ltm virtual vs_ftps_implicit_data {
    destination 192.0.2.50:40000-40100
    ip-protocol tcp
    profiles {
        tcp { }
    }
    pool ftps_implicit_pool
    persist {
        ftps_implicit_persist { }
    }
    source-address-translation {
        type automap
    }
    mirror enabled
}
```

### Aandachtspunten implicit

- **Passive range op de backend en op de firewall** moet exact overeenkomen met het range op de
  data-VS (`40000–40100` in het voorbeeld). Mismatch = transfers die hangen.
- **PASV-adres adverteren.** Stel op de FTP-server in dat hij het publieke/VS-adres in de `PASV`-respons
  zet (bij vsftpd `pasv_address`, bij ProFTPD `MasqueradeAddress`). BIG-IP kan dit hier niet corrigeren
  omdat het de versleutelde control channel niet kan herschrijven.
- **Geen `ftp`-monitor** — die spreekt plain FTP en faalt op een implicit-TLS-poort. Gebruik `tcp` of een
  `https`/ssl-monitor op 990.
- **Active mode werkt niet betrouwbaar** door deze opzet; dwing passive mode af bij de clients.

> Kort samengevat: explicit FTPS is de nette variant waar BIG-IP echt waarde toevoegt via het FTP-profiel.
> Implicit FTPS dwingt je tot L4 passthrough waarbij de backends en firewall het zware werk doen — kies
> waar mogelijk explicit.
