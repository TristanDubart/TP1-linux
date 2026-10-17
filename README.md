## B.1 Justifications du partitionnement

#Pourquoi séparer /var et surtout /var/log du volume racine ?
Le volume /var/log est le plus exposé à la saturation car il reçoit tous les journaux du système et des applications. Si un service s'emballe et génère d'énormes fichiers journaux, isoler /var/log empêche de saturer le volume racine /, ce qui bloquerait complètement le serveur (par exemple, impossibilité d'écrire des fichiers temporaires ou de s'authentifier).

#Pourquoi la partition /boot reste-t-elle une partition classique ?
Elle doit rester en dehors du groupe de volumes LVM pour que le chargeur d'amorçage (comme GRUB) puisse lire le noyau et lancer le système simplement, avant même que les pilotes LVM plus complexes ne soient chargés en mémoire.   

#Pourquoi attribuer 2 Gio de mémoire d'échange (swap) sur une machine de 2 Gio de RAM ?
Il est de bonne pratique d'aligner la taille du swap sur la taille de la mémoire vive. Cela permet au système de décharger la RAM en cas de fort besoin ponctuel sans déclencher immédiatement le tueur de processus (OOM Killer).   

#Pourquoi ne pas allouer la totalité du disque ?
L'espace libre restant du disque est volontairement mis de côté pour permettre des extensions futures. Il servira concrètement à l'agrandissement en ligne des volumes lors du TP 2 et éventuellement à la prise d'instantanés LVM.   

#Quel volume risque de saturer en premier, et comment l'agrandir sans coupure ?
C'est le volume /var/log qui risque de saturer en premier en production. Grâce à LVM, on pourra puiser dans l'espace non alloué du groupe de volumes pour agrandir le volume logique à chaud (sans éteindre le serveur). 
