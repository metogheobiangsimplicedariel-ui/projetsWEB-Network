# Rapport Technique : Architecture Réseau Campus Sécurisée

**Auteur :** METOGHE OBIANG Simplice Dariel
**Date :** 18 Janvier 2026  
**Sujet :** Conception et déploiement d'une infrastructure réseau multi-bâtiments avec isolation logique stricte.

---

## 1. Présentation du Projet

Ce projet vise à concevoir et déployer l'infrastructure réseau d'un campus composé de 4 bâtiments. Le défi principal réside dans la gestion de multiples entités (écoles) cohabitant au sein des mêmes bâtiments (notamment le Bâtiment 4), ce qui impose une segmentation réseau rigoureuse.

**Objectifs techniques :**

1.  **Interconnexion :** Routage dynamique OSPF entre les bâtiments.
2.  **Autonomie :** Distribution d'adresses IP (DHCP) gérée localement par étage.
3.  **Sécurité :** Isolation des réseaux étudiants (Pédagogie) via des ACLs strictes ("Zero Trust" vers l'interne).

---

## 2. Architecture Technique

### 2.1 Topologie Logique

L'architecture repose sur un modèle hiérarchique :

- **Accès :** Commutateurs de couche 2 distribuant les VLANs.
- **Distribution/Cœur Local :** Routeurs d'étage assurant le routage inter-VLAN (Router-on-a-Stick).
- **Cœur de Réseau :** Routeurs de bâtiment et routeur central interconnectés en OSPF.

### 2.2 Plan d'Adressage (VLSM)

Le campus utilise un bloc d'adressage privé global de classe B : **10.15.0.0/16**.
Ce bloc est subdivisé en sous-réseaux de taille `/24` pour chaque VLAN, permettant d'héberger jusqu'à 254 hôtes par entité.

| Niveau          | Adresse Réseau | Masque | Utilisation              |
| :-------------- | :------------- | :----- | :----------------------- |
| **Global**      | `10.15.0.0`    | `/16`  | Adresse du Campus entier |
| **Sous-Réseau** | `10.15.28.0`   | `/23`  | Exemple : EDS            |
| **Réseau-VLAN** | `10.15.28.0`   | `/24`  | Exemple : Admin École 1  |
| **Réseau-VLAN** | `10.15.28.0`   | `/24`  | Exemple : Peda École 1   |

---

## 3. Implémentation Technique

Cette section détaille les configurations types appliquées aux équipements d'un étage (Exemple : École 1).

### 3.1 Configuration du Commutateur (Switch d'Access)

Le switch assure la séparation physique des flux via les VLANs et l'acheminement vers le routeur via un lien Trunk.

```cisco
enable
configure terminal

! 1. Création des VLANs
vlan 10
 name ADMINISTRATION
exit
vlan 20
 name PEDAGOGIE
exit

! 2. Affectation des ports (Access)
! Ports pour l'Administration
interface range FastEthernet0/1-10
 switchport mode access
 switchport access vlan 10
 exit

! Ports pour la Pédagogie
interface range FastEthernet0/11-20
 switchport mode access
 switchport access vlan 20
 exit

! 3. Lien vers le Routeur (Trunk)
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 end
```

### 3.2 Configuration du Routeur d'Étage (Services & Routage)

#### A. Interfaces Virtuelles (Router-on-a-Stick)

```cisco
enable
configure terminal

! Activation de l'interface physique
interface GigabitEthernet0/0
 no shutdown
 exit

! Gateway Admin (VLAN 10)
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 10.15.28.1 255.255.255.0
 exit

! Gateway Peda (VLAN 20)
interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 10.15.29.1 255.255.255.0
 exit

```

#### B. Service DHCP Local (Autonome)

Le routeur distribue lui-même les IP, garantissant le fonctionnement du réseau local même en cas de coupure du lien vers le bâtiment.

```cisco
! Exclusion des passerelles
ip dhcp excluded-address 10.15.28.1
ip dhcp excluded-address 10.15.29.1

! Pool ADMIN
ip dhcp pool admin
 network 10.15.28.0 255.255.255.0
 default-router 10.15.28.1
 exit

! Pool PEDA
ip dhcp pool peda
 network 10.15.29.0 255.255.255.0
 default-router 10.15.29.1
 exit

```

#### C. Routage OSPF (Zone 0)

```cisco
 router ospf 1
 router-id 11.11.11.11

 ! Annonce des réseaux locaux
 network 10.15.28.0 0.0.0.255 area 0
 network 10.15.29.0 0.0.0.255 area 0

 ! Annonce du lien d'interconnexion (vers routeur Bâtiment)
 network 177.77.95.0 0.0.0.3 area 0

 passive-interface GigabitEthernet0/0.10
 passive-interface GigabitEthernet0/0.20
 exit

```

---

## 4. Sécurité et Filtrage (ACL)

La sécurité repose sur le principe du **moindre privilège** appliqué aux réseaux étudiants.

### 4.1 Stratégie de Filtrage

L'objectif est d'empêcher toute communication latérale (entre écoles ou étages) tout en maintenant les accès nécessaires.

- **Autorisé :** Trafic vers l'Admin Local (pour accès serveurs pédagogiques/fichiers).
- **Autorisé :** Trafic vers Internet (Any).
- **Interdit :** Tout trafic vers les autres réseaux privés (10.0.0.0/8).

### 4.2 Script de l'ACL (Appliqué sur le Routeur)

```cisco
! Création de l'ACL Étendue
ip access-list extended SECURITE_ECOLE

 ! 1. Exception : Autoriser l'accès à l'Admin LOCAL uniquement
 ! Source : Peda (10.15.29.0) -> Destination : Admin (10.15.28.0)
 permit ip 10.15.29.0 0.0.0.255 10.15.28.0 0.0.0.255

 ! 2. Isolation : Bloquer l'accès à tout le reste de l'intranet
 deny ip any 10.0.0.0 0.255.255.255

 ! 3. Sortie : Autoriser l'accès Internet
 permit ip any any
 exit

! Application sur l'interface Pédagogie (Sens entrant/IN)
interface GigabitEthernet0/0.20
 ip access-group SECURITE_ECOLE in
 exit

```

---

## 5. Tests et Validation

### 5.1 Tests

#### 5.2.1 **Test 1 : Ping Admin -> Admin Distant**

```cisco
C:\>ipconfig

FastEthernet0 Connection:(default port)

   Connection-specific DNS Suffix..:
   Link-local IPv6 Address.........: FE80::260:3EFF:FED0:DA04
   IPv6 Address....................: ::
   IPv4 Address....................: 10.15.28.3
   Subnet Mask.....................: 255.255.255.0
   Default Gateway.................: ::
                                     10.15.28.1

Bluetooth Connection:

   Connection-specific DNS Suffix..:
   Link-local IPv6 Address.........: ::
   IPv6 Address....................: ::
   IPv4 Address....................: 0.0.0.0
   Subnet Mask.....................: 0.0.0.0
   Default Gateway.................: ::
                                     0.0.0.0
C:\>ping 10.15.32.2

Pinging 10.15.32.2 with 32 bytes of data:

Reply from 10.15.32.2: bytes=32 time<1ms TTL=125
Reply from 10.15.32.2: bytes=32 time<1ms TTL=125
Reply from 10.15.32.2: bytes=32 time<1ms TTL=125
Reply from 10.15.32.2: bytes=32 time<1ms TTL=125

Ping statistics for 10.15.32.2:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms

C:\>
```

#### 5.2.2 **Test 2 : Ping Peda -> Admin Local**

```cisco
C:\>ipconfig

FastEthernet0 Connection:(default port)

   Connection-specific DNS Suffix..:
   Link-local IPv6 Address.........: FE80::240:BFF:FED7:33D2
   IPv6 Address....................: ::
   IPv4 Address....................: 10.15.29.3
   Subnet Mask.....................: 255.255.255.0
   Default Gateway.................: ::
                                     10.15.29.1

Bluetooth Connection:

   Connection-specific DNS Suffix..:
   Link-local IPv6 Address.........: ::
   IPv6 Address....................: ::
   IPv4 Address....................: 0.0.0.0
   Subnet Mask.....................: 0.0.0.0
   Default Gateway.................: ::
                                     0.0.0.0

C:\>ping 10.15.28.2

Pinging 10.15.28.2 with 32 bytes of data:

Request timed out.
Reply from 10.15.28.2: bytes=32 time<1ms TTL=127
Reply from 10.15.28.2: bytes=32 time=3ms TTL=127
Reply from 10.15.28.2: bytes=32 time<1ms TTL=127

Ping statistics for 10.15.28.2:
    Packets: Sent = 4, Received = 3, Lost = 1 (25% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 3ms, Average = 1ms

C:\>
```

#### 5.2.3 **Test 3 : Ping Peda -> Autre École**

````cisco

```cisco
C:\>ipconfig

FastEthernet0 Connection:(default port)

   Connection-specific DNS Suffix..:
   Link-local IPv6 Address.........: FE80::240:BFF:FED7:33D2
   IPv6 Address....................: ::
   IPv4 Address....................: 10.15.29.3
   Subnet Mask.....................: 255.255.255.0
   Default Gateway.................: ::
                                     10.15.29.1

Bluetooth Connection:

   Connection-specific DNS Suffix..:
   Link-local IPv6 Address.........: ::
   IPv6 Address....................: ::
   IPv4 Address....................: 0.0.0.0
   Subnet Mask.....................: 0.0.0.0
   Default Gateway.................: ::
                                     0.0.0.0
C:\>ping 10.15.23.2

Pinging 10.15.23.2 with 32 bytes of data:

Reply from 10.15.29.1: Destination host unreachable.
Reply from 10.15.29.1: Destination host unreachable.
Reply from 10.15.29.1: Destination host unreachable.
Reply from 10.15.29.1: Destination host unreachable.

Ping statistics for 10.15.23.2:
    Packets: Sent = 4, Received = 0, Lost = 4 (100% loss),

````

### 5.2 Résumé des tests

| Test                            | Source        | Destination   | Résultat Attendu                |
| ------------------------------- | ------------- | ------------- | ------------------------------- |
| **Ping Admin -> Admin Distant** | Admin École 1 | Admin École 2 | **Succès** (Routage OSPF OK)    |
| **Ping Peda -> Admin Local**    | Peda École 1  | Admin École 1 | **Succès** (ACL Permit Ligne 1) |
| **Ping Peda -> Autre École**    | Peda École 1  | Peda École 2  | **Bloqué** (ACL Deny Ligne 2)   |

## 6. Conclusion

L'architecture déployée répond aux exigences de performance et de sécurité. L'usage du **DHCP local** assure la résilience des services de base, tandis que le routage **OSPF** garantit une connectivité fluide entre les bâtiments. Enfin, la mise en place rigoureuse des **ACLs** assure que chaque environnement pédagogique reste cloisonné, prévenant ainsi les risques de sécurité interne.
