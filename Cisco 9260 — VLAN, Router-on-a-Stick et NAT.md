# Cisco 9260 — VLAN, Router-on-a-Stick et NAT

> ⚠️ **Configuration ancienne :** ce lab correspond à une ancienne configuration réalisée avec un switch Cisco 9260 et un routeur Cisco. Il est conservé comme exemple de mise en pratique des VLAN, du routage inter-VLAN et du NAT/PAT.

## Présentation

Cette configuration met en place deux réseaux locaux séparés par **VLAN** :

- **VLAN 10** — `192.168.10.0/24`
  - Un PC
  - Un serveur Proxmox
- **VLAN 20** — `192.168.20.0/24`
  - Un second PC

Le switch est relié au routeur sur le port `GigabitEthernet1/0/1` du switch et `GigabitEthernet0/0` du routeur.

La liaison entre le switch et le routeur est configurée en **trunk 802.1Q**. Le routage inter-VLAN est réalisé par le routeur à l'aide de **sous-interfaces** (*Router-on-a-Stick*).

Le routeur dispose également d'une interface WAN `GigabitEthernet0/1`, utilisée comme interface **NAT outside** pour permettre aux machines des deux VLAN d'accéder à Internet.

## Architecture

```text
                         Internet
                            │
                         G0/1
                       NAT outside
                            │
                       ┌────┴────┐
                       │ Routeur │
                       └────┬────┘
                            │ G0/0
                         802.1Q
                          TRUNK
                            │
                     Gi1/0/1 │
                       ┌─────┴─────┐
                       │  Switch   │
                       │ Cisco 9260│
                       └─────┬─────┘
                         │       │
                    Gi1/0/10   Gi1/0/20
                       │          │
                    VLAN 10     VLAN 20
                       │          │
                 PC + Proxmox    PC
```

## Plan d'adressage

| Élément | VLAN | Adresse |
|---|---:|---|
| Réseau VLAN 10 | 10 | `192.168.10.0/24` |
| Passerelle VLAN 10 | 10 | `192.168.10.254` |
| Réseau VLAN 20 | 20 | `192.168.20.0/24` |
| Passerelle VLAN 20 | 20 | `192.168.20.254` |
| Routeur → Internet | — | DHCP / selon le réseau WAN |

---

# Partie Switch

## Création des VLAN

```cisco
vlan 10
 name VLAN10
exit

vlan 20
 name VLAN20
exit
```

## Ports d'accès

Le port `Gi1/0/10` est utilisé pour le VLAN 10 :

```cisco
interface gigabitEthernet 1/0/10
 switchport mode access
 switchport access vlan 10
exit
```

Le port `Gi1/0/20` est utilisé pour le VLAN 20 :

```cisco
interface gigabitEthernet 1/0/20
 switchport mode access
 switchport access vlan 20
exit
```

## Liaison trunk vers le routeur

Le port `Gi1/0/1` transporte les VLAN 10 et 20 vers le routeur :

```cisco
interface gigabitEthernet 1/0/1
 switchport mode trunk
exit
```

---

# Partie Routeur

## Interface WAN

L'interface `G0/1` est utilisée pour la connexion vers l'extérieur et définie comme interface NAT outside :

```cisco
interface gigabitEthernet 0/1
 ip nat outside
 no shutdown
exit
```

## Sous-interface VLAN 10

```cisco
interface gigabitEthernet 0/0.10
 description VLAN10
 encapsulation dot1q 10
 ip address 192.168.10.254 255.255.255.0
 ip nat inside
exit
```

Cette sous-interface sert de **passerelle par défaut** aux machines du VLAN 10.

## Sous-interface VLAN 20

```cisco
interface gigabitEthernet 0/0.20
 description VLAN20
 encapsulation dot1q 20
 ip address 192.168.20.254 255.255.255.0
 ip nat inside
exit
```

Cette sous-interface sert de **passerelle par défaut** aux machines du VLAN 20.

L'interface physique `G0/0` doit également être active :

```cisco
interface gigabitEthernet 0/0
 no shutdown
exit
```

---

# Configuration NAT/PAT

Les deux réseaux privés sont autorisés à utiliser le NAT :

```cisco
access-list 1 permit 192.168.10.0 0.0.0.255
access-list 2 permit 192.168.20.0 0.0.0.255
```

Les deux ACL sont ensuite utilisées pour effectuer du **NAT Overload (PAT)** vers l'adresse de l'interface WAN `G0/1` :

```cisco
ip nat inside source list 1 interface gigabitEthernet 0/1 overload
ip nat inside source list 2 interface gigabitEthernet 0/1 overload
```

Ainsi, plusieurs machines des VLAN 10 et 20 peuvent partager l'adresse IP de l'interface WAN pour accéder à Internet.

---

# Fonctionnement

Le chemin d'une requête Internet depuis une machine du VLAN 10 est donc :

```text
PC VLAN 10
    │
    ▼
Switch — VLAN 10
    │
    │ 802.1Q
    ▼
Routeur G0/0.10
    │
    │ NAT/PAT
    ▼
Routeur G0/1
    │
    ▼
Internet
```

Le même principe s'applique au VLAN 20 via la sous-interface `G0/0.20`.

Le routeur assure également le **routage entre les VLAN 10 et 20**, puisque chacun possède sa propre interface de niveau 3 sur le routeur.

---

# Commandes de vérification

### Switch

```cisco
show vlan brief
show interfaces trunk
show running-config
```

### Routeur

```cisco
show ip interface brief
show ip route
show ip nat translations
show ip nat statistics
show access-lists
```

## Points importants à retenir

- **VLAN** → séparation logique des réseaux sur le switch.
- **Trunk** → transporte plusieurs VLAN entre le switch et le routeur.
- **`encapsulation dot1q`** → permet au routeur d'identifier le VLAN reçu sur le trunk.
- **Sous-interface** → sert de passerelle de niveau 3 pour chaque VLAN.
- **`ip nat inside`** → côté réseaux privés.
- **`ip nat outside`** → côté WAN.
- **ACL NAT** → détermine quels réseaux privés sont concernés.
- **`overload`** → permet à plusieurs machines de partager l'adresse WAN.