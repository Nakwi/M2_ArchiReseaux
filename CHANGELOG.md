# Changelog — NovaCorp (site AIX)

Toutes les étapes du projet NovaCorp : conception (schéma, adressage), puis configuration de la maquette Cisco Packet Tracer (`novacorp.pkt`).

Les étapes sont classées dans l'ordre chronologique.

---

## [0.1] — 08-09 octobre 2026 — Schéma d'architecture

### Ajouté
- **Schéma d'architecture** (`NovaCorp.drawio`) :
  - 2 pare-feux (FW-AIX-01/02) en sortie vers Internet (Cloud).
  - 2 cœurs L3 (SW-AIX-05/06), reliés entre eux par 2 liens.
  - 4 switchs d'accès : SW-AIX-01/02 (utilisateurs), SW-AIX-03/04 (serveurs), chacun raccordé aux deux cœurs.
  - Équipements : téléphones IP, bornes Wi-Fi + laptops, 3 hyperviseurs (VM serveurs), NAS.
  - Légende des VLANs par couleur.

---

## [0.2] — 08-09 octobre 2026 — Adressage IP, inventaire et nommage

### Ajouté
- **Plan d'adressage IPv4 / IPv6** (`IPV4-IPV6_Nommage_Inventaire.xlsx`, onglet « IPV4 & IPV6 ») :

| VLAN | Nom | Réseau IPv4 | Passerelle | Préfixe IPv6 |
|---|---|---|---|---|
| 9 | VOIP | 192.168.9.0/24 | 192.168.9.254 | fd3a:91c4:5e07:9::/64 |
| 10 | PC | 192.168.10.0/24 | 192.168.10.254 | fd3a:91c4:5e07:10::/64 |
| 40 | RESEAUX | 192.168.40.0/27 | 192.168.40.30 | fd3a:91c4:5e07:40::/64 |
| 70 | HPV | 192.168.70.0/29 | 192.168.70.6 | fd3a:91c4:5e07:70::/64 |
| 80 | SRV | 192.168.80.0/24 | 192.168.80.254 | fd3a:91c4:5e07:80::/64 |
| 100 | IT | 192.168.100.0/26 | 192.168.100.62 | fd3a:91c4:5e07:100::/64 |

- **Inventaire** (onglet « Inventaire ») : IP fixes des équipements.
  - HPV-AIX-01 à 03 : 192.168.70.1 à .3
  - FW-AIX-01/02 : 192.168.40.1 et .2
  - SW-AIX-01 à 06 : 192.168.40.3 à .8
  - AP-AIX-01 à 10 : 192.168.40.9 à .18
  - NAS-AIX-01 : 192.168.80.1
  - TEL-AIX-01 à 65 : 192.168.9.1 à .65
- **Convention de nommage** (onglet « Nommage ») : format `SSS-VVV-00` (type – trigramme ville – incrément), villes AIX, PEY, MRS. Préfixes : SRV, DC, PC, SW, FW, AP, TEL, HPV, NAS.

---

## [0.3] — 08-09 octobre 2026 — Infrastructure de base sur Cisco Packet Tracer

### Ajouté
- **Maquette Packet Tracer** (`novacorp.pkt`) :
  - FW-AIX-01/02 : routeurs PT8200.
  - SW-AIX-05/06 : switchs L3 Cisco 3560-24PS.
  - SW-AIX-01 à 04 : switchs Cisco 2960-24TT.
  - HPV-AIX-01 à 03 (Server-PT), TEL-AIX-01 à 04 (Home-VoIP-PT), AP-AIX-01 à 04 (AccessPoint-PT), PC-AIX-01 à 04 (Laptop-PT).
  - Câblage conforme au schéma : double attachement accès → cœurs, 2 liens inter-cœurs (Fa0/7-8), chaque FW raccordé aux deux cœurs, hyperviseurs raccordés à SW-AIX-03 et 04.

---

## [0.4] — 08-09 octobre 2026 — VLANs et trunks 802.1Q

### Ajouté
- Hostnames : SW-AIX-01 à SW-AIX-06, FW-AIX-01 et FW-AIX-02.
- `vtp mode transparent` sur les 6 switchs.
- VLANs créés sur les 6 switchs :

| ID | Nom |
|---|---|
| 9 | VOIP |
| 10 | PC |
| 40 | RESEAUX |
| 70 | HPV |
| 80 | SRV |
| 100 | IT |

- Trunks 802.1Q (`switchport nonegotiate`, VLANs autorisés : 9, 10, 40, 70, 80, 100) :
  - SW-AIX-01 à 04 : Fa0/1-2 (uplinks vers les cœurs).
  - SW-AIX-05 et 06 : Fa0/3-8 (vers l'accès + liaison inter-cœurs), encapsulation `dot1q`.

### Validé
- `show interfaces trunk` et `show vlan brief` sur les 6 switchs.

---

## [0.5] — 08-09 octobre 2026 — Rapid PVST+ et LACP

### Ajouté
- `spanning-tree mode rapid-pvst` sur les 6 switchs.
- Répartition des root bridges :

| VLANs | Root primaire (24576) | Root secondaire (28672) |
|---|---|---|
| 9, 10, 100 | SW-AIX-05 | SW-AIX-06 |
| 40, 70, 80 | SW-AIX-06 | SW-AIX-05 |

- **LACP** entre SW-AIX-05 et SW-AIX-06 : Fa0/7-8 en `channel-group 1 mode active` → Port-channel1 (trunk dot1q).

### Corrigé
- Convergence STP bloquée après passage en Rapid PVST+ (SW-AIX-01 voyait SW-06 comme root du VLAN 9) → résolu par `shutdown` / `no shutdown` des uplinks.
- Port-channel bloqué en `(SD)` / ports `(s)` malgré une config identique des deux côtés → résolu par `write memory` + `reload` des deux cœurs.
- `switchport nonegotiate` retiré de Po1 et Fa0/7-8 (bloquait la négociation LACP dans Packet Tracer).

### Validé
- `show etherchannel summary` : `Po1(SU) LACP Fa0/7(P) Fa0/8(P)`.
- `show spanning-tree` : SW-05 root pour 9/10/100, SW-06 root pour 40/70/80 ; uplinks d'accès en `Root FWD` / `Altn BLK`.

---

## [0.6] — 08-09 octobre 2026 — Ports d'accès et protections STP

### Ajouté
- **SW-AIX-01 / 02** : Fa0/3-4 → VLAN 9 (téléphones), Fa0/5-6 → VLAN 10 (bornes Wi-Fi).
- **SW-AIX-03 / 04** : Fa0/3-5 → VLAN 70 (hyperviseurs).
- PortFast + BPDU Guard sur tous les ports d'accès.
- **Root Guard** sur Fa0/3-6 de SW-AIX-05 et SW-AIX-06 (ports vers l'accès).
- Ports inutilisés en `shutdown` sur les 6 switchs.

---

## [0.7] — 26-27 octobre 2026 — Couche 3 : routage, HSRP, pare-feux

### Ajouté
- `ip routing` sur SW-AIX-05 et SW-AIX-06.
- **SVI + HSRP** sur les deux cœurs (le cœur actif est aligné sur le root STP du VLAN) :

| VLAN | SW-AIX-05 | SW-AIX-06 | VIP | Actif |
|---|---|---|---|---|
| 9 VOIP | 192.168.9.252 | 192.168.9.253 | 192.168.9.254 | SW-05 |
| 10 PC | 192.168.10.252 | 192.168.10.253 | 192.168.10.254 | SW-05 |
| 100 IT | 192.168.100.60 | 192.168.100.61 | 192.168.100.62 | SW-05 |
| 40 RESEAUX | 192.168.40.7 | 192.168.40.8 | 192.168.40.30 | SW-06 |
| 70 HPV | 192.168.70.4 | 192.168.70.5 | 192.168.70.6 | SW-06 |
| 80 SRV | 192.168.80.252 | 192.168.80.253 | 192.168.80.254 | SW-06 |

- Priorité HSRP `110` + `preempt` sur le cœur actif, `100` + `preempt` sur le backup.
- Fa0/1-2 des cœurs (vers les FW) : accès VLAN 40, PortFast, BPDU Guard.
- Route par défaut sur les cœurs : `0.0.0.0/0 → 192.168.40.20` (VIP des pare-feux).
- **FW-AIX-01** : Gi0/0/0 `192.168.40.1/27`, HSRP groupe 41 actif (priorité 110).
- **FW-AIX-02** : Gi0/0/0 `192.168.40.2/27`, HSRP groupe 41 standby.
- VIP pare-feux : `192.168.40.20`.
- Routes statiques sur les FW vers les 5 réseaux internes via `192.168.40.30`.
- IP de management des switchs d'accès (VLAN 40), passerelle `192.168.40.30` :
  SW-AIX-01 `.3`, SW-AIX-02 `.4`, SW-AIX-03 `.5`, SW-AIX-04 `.6`.

### Modifié
- Gi0/0/1 des deux pare-feux en `shutdown` : un routeur ne peut pas avoir deux interfaces dans le même sous-réseau. Les liens croisés FW ↔ cœur opposé sont inactifs, la redondance est assurée par HSRP.

### Validé
- `show standby brief` : SW-05 actif pour 9/10/100, SW-06 actif pour 40/70/80.
- Ping des switchs d'accès vers `192.168.40.30` et `192.168.40.20` : OK.

---

## [0.8] — 26-27 octobre 2026 — DHCP

### Ajouté
- **DHCP sur SW-AIX-05 et SW-AIX-06** : pools `VOIP` (VLAN 9) et `PC` (VLAN 10), passerelle = VIP HSRP (`.254`).
- Plages réparties entre les deux cœurs pour la redondance :

| Cœur | VLAN 9 | VLAN 10 |
|---|---|---|
| SW-AIX-05 | 192.168.9.1 – .127 | 192.168.10.1 – .127 |
| SW-AIX-06 | 192.168.9.128 – .250 | 192.168.10.128 – .250 |

- Adresses exclues : `.251` à `.254` (IP des cœurs + VIP HSRP).
- Téléphones (TEL-AIX-01 à 04) et laptops (PC-AIX-01 à 04) passés en client DHCP.

### Validé
- `show ip dhcp binding` : 4 téléphones et 4 laptops ont reçu une IP.
- Ping PC-AIX (VLAN 10) → passerelle `192.168.10.254` : OK.
- Ping PC-AIX (VLAN 10) → TEL-AIX (VLAN 9) : OK (TTL 127, routage inter-VLAN fonctionnel).

## [0.9] — 17 Novembre 2026 - Récapitulatif des protocoles en place
 
| Protocole | Couche | Où | Rôle |
|---|---|---|---|
| 802.1Q | 2 | Trunks entre les 6 switchs | Transporter plusieurs VLANs sur un même câble |
| Rapid PVST+ (802.1w) | 2 | Les 6 switchs | Empêcher les boucles. Root : SW-05 pour 9/10/100, SW-06 pour 40/70/80 |
| ↳ PortFast + BPDU Guard | 2 | Ports TEL, AP, HPV, FW | Activer les ports tout de suite, et les couper si on y branche un switch |
| ↳ Root Guard | 2 | SW-05/06, ports vers l'accès | Empêcher un switch d'accès de devenir root |
| LACP (802.3ad) | 2 | SW-05 ↔ SW-06 (Po1) | Regrouper les deux câbles entre les cœurs en un seul lien |
| Routage inter-VLAN (SVI) | 3 | SW-05 et SW-06 | Faire communiquer les VLANs entre eux |
| HSRP | 3 | SW-05/06 et FW-01/02 | Garder une passerelle disponible si un équipement tombe |
| Routage statique | 3 | Cœurs et pare-feux | Route par défaut vers les FW, et routes des FW vers les réseaux internes |
| DHCP | 7 | SW-05 et SW-06 | Donner automatiquement une IP aux téléphones et aux laptops |
