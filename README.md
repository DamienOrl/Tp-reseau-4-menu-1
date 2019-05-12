# Tp reseau 4 menu 1: Sécurisation

* [Infra de base](#infrastructure-de-départ)
* [Renforcement](#renforcement-de-linfrastructure)
* [Inspection du trafic réseau](#hé-la-qui-va-là?)

## Infrastructure de départ

Les manipulations pour obtenir notre infrastructure de base sont disponibles [dans la partie 4 du TP précédent.](https://github.com/DamienOrl/TP-reseau-3/blob/master/README.md#iv-lab-final)

## Renforcement de l'infrastructure
* [Enlever les SPoF](#nettoyage-des-spof-single-point-of-failure)
* [Aggrégation de liens entre les switches](#utilisation-du-lacp-link-aggregation-control-protocol)
* [VRRP entre les routeurs](#mise-en-place-du-vrrp-virtual-router-redundancy-protocol)

### Nettoyage des SPoF (Single Point of Failure)
Dans notre architecture, utiliser quatre routeurs ne sert pas vraiment, vu sa taille assez réduite. Nous ne garderons donc que `R1` et `R4`, les deux routeurs les plus importants (l'un étant relié au NAT, l'autre aux clients et serveurs)!

Ensuite, nous devons relier `R4` au NAT et lui donner une IP via DHCP:

```
R4#(config)# interface fastEthernet 1/0
R4(config-if)# ip nat outside
R4(config-if)# ip address dhcp
R4(config-if)# no shut
R4(config-if)# exit

R4(config)# interface fastEthernet 0/0
R4(config-if)# ip nat inside
R4(config-if)# exit
```

Enfin, nous paramétrons `R1` pour en faire un routeur "on a stick" comme `R4` que nous relions à `IOU2`. Ainsi, lorsque l'un des routeurs n'est plus en état de marche, l'autre est capable de prendre le relais!

### Utilisation du LACP (Link Aggregation Control Protocol)

Ce procédé consiste à aggréger plusieurs ports entre eux afin d'améliorer la disponibilité de la liaison entre deux appareils. De cette manière, elle reste opérationnelle même si une des connexions devient défaillante!

Nous allons mettre en place ce protocole sur la liaison entre les deux switches:

```
IOU1(config)#interface range fastEthernet 1/2 - 3
IOU1(config-if-range)#channel-group 1 mode active
IOU1(config-if-range)#channel-protocol lacp
```

```
IOU2(config)#interface range fastEthernet 2/1 - 2
IOU2(config-if-range)#channel-group 1 mode active
IOU2(config-if-range)#channel-protocol lacp
```

**Et voilà!** Nous avons fait fusionner deux ports, formant désormais un PortChannel!
<img src="https://media.giphy.com/media/MY6LrEW1m6rgA/giphy.gif" width="350">

### Mise en place du VRRP (Virtual Router Redundancy Protocol)

Ce protocole permet d'assigner une adresse IP virtuelle à un groupe de routeurs, ainsi qu'un ordre de priorité entre eux afin de déterminer qui doit prendre en charge le trafic réseau.

Sur `R4`:
```
interface f0/0.10
ip address 10.1.1.253 255.255.255.0
no shut
vrrp 100 ip 10.1.1.254
vrrp 100 preempt

interface f0/0.20
ip address 10.2.2.253 255.255.255.0
no shut
vrrp 200 ip 10.2.2.254
vrrp 200 preempt
```


Et sur `R1`:
```
interface f0/0.10
ip address 10.1.1.251 255.255.255.0
no shut
vrrp 100 ip 10.1.1.254
vrrp 100 priority 90

interface f0/0.20
ip address 10.2.2.251 255.255.255.0
no shut
vrrp 200 ip 10.2.2.254
vrrp 200 priority 90
```

**Et c'est bon!** Maintenant, si `R4` tombe en panne, `R1` est promu maître et devient responsable de la gestion du trafic!

----
Après ces modifications, notre infrastructure ressemble désormais à ça:
<img src="Captures/Infra renforcée.png" />

## Hé la qui va là?

Pour que nous puissions nous assurer de la confidentialité des échanges(et voir si on essaye pas de nous faire une attaque de type DoS), nous allons mettre en place du port mirroring sur nos deux switches. Ce procédé nous permet de conserver une copie du trafic réseau afin de pouvoir l'analyser ensuite.

L'objectif est de stocker le trafic en provenance du VLAN 10 dans `server1` et celui arrivant du VLAN 20 dans `server2`:

```
IOU1(config)# monitor session 1 source int ethernet 1/1 both
IOU1(config)# monitor session 1 destination int ethernet 0/0
```

Avec ces deux commandes, `IOU1` envoie désormais une copie du trafic en provenance et à destination de l'interface `ethernet 1/1` à l'interface `ethernet 0/0`. Maintenant, il faut faire en sorte de faire pareille depuis `IOU2`...

```
IOU2(config)# monitor session 1 source int ethernet 1/1 both
IOU2(config)# monitor session 1 destination int PortChannel 1
```

`IOU1` n'a plus qu'à intercepter et à envoyer vers `ethernet 1/0`, où est branché `server2`:

```
IOU1(config)# monitor session 2 source int PortChannel 1 tx (pour n'enregistrer que ceux transmis)
IOU1(config)# monitor session 2 destination int ethernet 1/0
```

**Booyah!** Nous pouvons surveiller ce qui passe par nos deux swiches maintenant! <img src="https://media.giphy.com/media/NS7gPxeumewkWDOIxi/giphy.gif" width="350">
