# Création du Serveur ChirpStack et du Serveur ThingsBoard

## Introduction

Ce guide vous accompagnera pas à pas dans la création et la configuration d'un serveur **ChirpStack** et d'un serveur **ThingsBoard** sur une machine Linux sous **Ubuntu 24.04**, en utilisant Docker et Docker Compose. Vous apprendrez à installer les prérequis, déployer les conteneurs nécessaires, créer manuellement un profil de dispositif dans ChirpStack, configurer les applications pour communiquer avec des dispositifs LoRaWAN comme le **Maduino Zero**, et configurer ThingsBoard pour visualiser et gérer les données reçues.

## Prérequis

- **Serveur Linux** (machine virtuelle ou physique) avec **Ubuntu 24.04**.
- **Accès administrateur** (sudo) sur la machine.
- **Connexion Internet** pour télécharger les packages et les images Docker.

## Installation de Docker, Docker Compose et Configuration des Services

Vous pouvez automatiser l'installation de Docker, Docker Compose, et la configuration initiale en exécutant directement le script `setup.sh` depuis le dépôt GitHub en utilisant la commande suivante :

### Exécution du script `setup.sh` avec `wget`

Étant donné que ce script est sécurisé et utilisé dans le cadre d'un cours au cégep, vous pouvez l'exécuter directement en utilisant la commande suivante :

```bash
sh -c "$(wget https://raw.githubusercontent.com/tgeinfo/documentation/main/chirpstack-thingsboard/setup.sh -O -)"
```

**Remarque :** Cette commande télécharge et exécute le script `setup.sh` en une seule étape. Assurez-vous d'avoir les droits administrateur (sudo) pour que le script puisse installer les packages nécessaires et configurer les services.

Le script `setup.sh` effectue les opérations suivantes :

- Met à jour et met à niveau le système Ubuntu.
- Installe Docker et Docker Compose.
- Démarre et active le service Docker.
- Clone le dépôt GitHub `tgeinfo/scripts` en utilisant une extraction partielle pour ne récupérer que le dossier `chirpstack-thingsboard`.
- Déplace le contenu du dossier cloné vers le répertoire courant et nettoie les fichiers temporaires.
- Crée les répertoires nécessaires et modifie les permissions.
- Télécharge l'image Docker de ThingsBoard avec PostgreSQL.
- Exécute le script de mise à niveau de ThingsBoard.
- Supprime le conteneur `mytb` s'il existe.
- Démarre les services définis dans le fichier `docker-compose.yml`.

**Note :** Cette commande doit être exécutée dans le répertoire où vous souhaitez installer les services.

### Contenu du fichier `docker-compose.yml`

Assurez-vous que le fichier `docker-compose.yml` est correctement configuré. Voici le contenu corrigé :

```yaml
version: '3.0'

services:
  mytb:
    restart: always
    image: "thingsboard/tb-postgres"
    ports:
      - "8081:9090"        # ThingsBoard accessible sur le port 8081
      - "1884:1883"        # MQTT port (modifié pour éviter les conflits)
      - "7070:7070"
      - "5683-5688:5683-5688/udp"
    environment:
      TB_QUEUE_TYPE: in-memory
    volumes:
      - ./mytb-data:/data
      - ./mytb-logs:/var/log/thingsboard

  chirpstack:
    image: chirpstack/chirpstack:4
    command: -c /etc/chirpstack
    restart: unless-stopped
    volumes:
      - ./configuration/chirpstack:/etc/chirpstack
      - ./lorawan-devices:/opt/lorawan-devices
    depends_on:
      - postgres
      - mosquitto
      - redis
    environment:
      - MQTT_BROKER_HOST=mosquitto
      - REDIS_HOST=redis
      - POSTGRESQL_HOST=postgres
    ports:
      - 8080:8080  # ChirpStack accessible sur le port 8080

  chirpstack-gateway-bridge:
    image: chirpstack/chirpstack-gateway-bridge:4
    restart: unless-stopped
    ports:
      - 1700:1700/udp
    volumes:
      - ./configuration/chirpstack-gateway-bridge:/etc/chirpstack-gateway-bridge
    environment:
      - INTEGRATION__MQTT__EVENT_TOPIC_TEMPLATE=us915_1/gateway/{{ .GatewayID }}/event/{{ .EventType }}
      - INTEGRATION__MQTT__STATE_TOPIC_TEMPLATE=us915_1/gateway/{{ .GatewayID }}/state/{{ .StateType }}
      - INTEGRATION__MQTT__COMMAND_TOPIC_TEMPLATE=us915_1/gateway/{{ .GatewayID }}/command/#
    depends_on:
      - mosquitto

  chirpstack-gateway-bridge-basicstation:
    image: chirpstack/chirpstack-gateway-bridge:4
    restart: unless-stopped
    command: -c /etc/chirpstack-gateway-bridge/chirpstack-gateway-bridge-basicstation-eu868.toml
    ports:
      - 3001:3001
    volumes:
      - ./configuration/chirpstack-gateway-bridge:/etc/chirpstack-gateway-bridge
    depends_on:
      - mosquitto

  chirpstack-rest-api:
    image: chirpstack/chirpstack-rest-api:4
    restart: unless-stopped
    command: --server chirpstack:8080 --bind 0.0.0.0:8090 --insecure
    ports:
      - 8090:8090
    depends_on:
      - chirpstack

  postgres:
    image: postgres:14-alpine
    restart: unless-stopped
    volumes:
      - ./configuration/postgresql/initdb:/docker-entrypoint-initdb.d
      - ./postgresqldata:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=root

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --save 300 1 --save 60 100 --appendonly no
    volumes:
      - ./redisdata:/data

  mosquitto:
    image: eclipse-mosquitto:2
    restart: unless-stopped
    ports:
      - 1883:1883
    volumes:
      - ./configuration/mosquitto/config/:/mosquitto/config/
```

**Remarques importantes :**

- **ThingsBoard** est configuré pour être accessible sur le port **8081** de l'hôte, mappé au port **9090** du conteneur.
- Toutes les références dans la documentation ont été mises à jour pour refléter ce changement de port.

## Accès aux Interfaces Web

Après le déploiement, les interfaces web seront accessibles aux adresses suivantes :

- **ChirpStack** : `http://[IP_DU_SERVEUR]:8080`
- **ThingsBoard** : `http://[IP_DU_SERVEUR]:8081`

## Configuration du Serveur ChirpStack et de ThingsBoard

### Configuration de ChirpStack

#### 1. Accéder à l'interface web de ChirpStack

```
http://[IP_DU_SERVEUR]:8080
```

#### 2. Se connecter avec les identifiants par défaut

- **Nom d'utilisateur** : `admin`
- **Mot de passe** : `admin`

#### 3. Créer un profil de dispositif

Comme le profil de dispositif **Maduino Zero 915** n'est pas fourni par défaut, vous devez le créer manuellement.

##### Étapes pour créer un profil de dispositif :

1. **Naviguez vers "Device-profiles"** dans le menu de gauche.
2. **Cliquez sur "Add device-profile"** en haut à droite.
3. **Remplissez les informations générales :**

   - **Name** : `Maduino Zero 915` (ou un autre nom descriptif)
   - **Region** : Sélectionnez la région correspondant à votre dispositif (par exemple, `US915` ou `EU868` selon votre fréquence)
   - **MAC version** : Sélectionnez la version LoRaWAN supportée par le dispositif (par exemple, `LoRaWAN 1.0.2`)
   - **PHY version** : Sélectionnez la version PHY correspondante (par exemple, `RFU`)
   - **Max EIRP** : Laissez la valeur par défaut ou ajustez selon votre réglementation locale

4. **Paramètres supplémentaires :**

   - **Supports OTAA** : Cochez si votre dispositif utilise OTAA (Over-The-Air Activation)
   - **Supports Class-B/C** : Cochez si applicable
   - **Payload codec** : Si vous utilisez un codec spécifique (comme CayenneLPP), sélectionnez `Custom JavaScript codec` et entrez le code nécessaire

5. **Cliquez sur "Create device-profile"** pour enregistrer le profil.

#### 4. Créer une application

1. **Naviguez vers "Applications"** dans le menu de gauche.
2. **Cliquez sur "Add application"**.
3. **Remplissez les champs :**

   - **Name** : Entrez un nom pour votre application
   - **Description** : (Optionnel) Entrez une description
   - **Service profile** : Sélectionnez le profil de service par défaut

4. **Cliquez sur "Create application"**.

#### 5. Ajouter un dispositif (device)

1. **Dans votre application**, cliquez sur l'onglet **"Devices"**.
2. **Cliquez sur "Add device"**.
3. **Dans l'onglet "Device", remplissez les champs :**

   - **Device name** : Entrez un nom pour votre dispositif
   - **Device EUI** : Générer un **Device EUI** en cliquant sur le bouton **Random** (icône de flèche tournante). **Conservez ce Device EUI**.
   - **Device-profile** : Sélectionnez le profil de dispositif que vous venez de créer (`Maduino Zero 915`)

4. **Cliquez sur "Create device"**.
5. **Dans l'onglet "Activation", sélectionnez "OTAA"**, puis :

   - **AppKey** : Générer une **App Key** en cliquant sur le bouton **Random**. **Conservez cette App Key**.

6. **Cliquez sur "Set device-keys"** pour enregistrer les clés.

**Note :** La clé **AppEUI** est généralement `0000000000000000` dans ChirpStack, sauf si spécifié autrement.

### Configuration de l'Intégration avec ThingsBoard

#### 1. Configurer l'intégration dans ChirpStack

- **Dans votre application**, allez dans l'onglet **"Integrations"**.
- **Cliquez sur le bouton **"+"** à côté de **"ThingsBoard"**.
- **Dans le champ "ThingsBoard URL"**, entrez l'adresse de votre serveur ThingsBoard :

  ```
  http://[IP_DU_SERVEUR]:8081
  ```

- **Cliquez sur "Add"** pour enregistrer l'intégration.

#### 2. Configurer un dispositif dans ThingsBoard

##### Accès à l'interface web de ThingsBoard

1. **Accédez à l'interface web de ThingsBoard :**

   ```
   http://[IP_DU_SERVEUR]:8081/
   ```

2. **Connectez-vous avec les identifiants par défaut :**

   - **Nom d'utilisateur** : `tenant@thingsboard.org`
   - **Mot de passe** : `tenant`

##### Création d'un profil de dispositif dans ThingsBoard

Avant de créer un dispositif, il est recommandé de créer un **Device Profile** personnalisé pour gérer les données spécifiques à votre dispositif.

1. **Accédez à "Device Profiles" :**

   - Cliquez sur **"Device profiles"** dans le menu de gauche sous **"Entities"**.

2. **Créer un nouveau profil de dispositif :**

   - Cliquez sur **"+"** en haut à droite pour ajouter un nouveau profil.
   - **Remplissez les champs :**

     - **Name** : Entrez un nom pour le profil (par exemple, `Maduino Zero Profile`)
     - **Description** : (Optionnel) Ajoutez une description.

   - **Cliquez sur "Add"**.

3. **Configurer le profil de dispositif :**

   - **Telemetry** : Définissez les clés de télémétrie que votre dispositif va envoyer.
   - **Attributes** : Si votre dispositif envoie des attributs spécifiques, vous pouvez les définir ici.
   - **Transport Configuration** : Laissez par défaut si vous utilisez MQTT.

4. **Enregistrez les modifications**.

##### Création d'un dispositif dans ThingsBoard

1. **Allez dans "Devices" :**

   - Cliquez sur **"Devices"** dans le menu de gauche.

2. **Créer un nouveau dispositif :**

   - Cliquez sur **"+"** en haut à droite pour ajouter un nouveau dispositif.
   - **Remplissez les champs :**

     - **Name** : Entrez un nom pour votre dispositif.
     - **Type** : (Optionnel) Spécifiez un type.
     - **Device Profile** : Sélectionnez le profil que vous venez de créer (`Maduino Zero Profile`).

   - **Cliquez sur "Add"**.

3. **Obtenir le jeton d'accès :**

   - **Cliquez sur le dispositif créé** pour afficher les détails.
   - **Cliquez sur l'onglet "Credentials"**.
   - **Copiez le "Access token"** et **conservez ce jeton**.

##### Configuration des tableaux de bord et widgets dans ThingsBoard

1. **Accédez à "Dashboards" :**

   - Cliquez sur **"Dashboards"** dans le menu de gauche.

2. **Créer un nouveau tableau de bord :**

   - Cliquez sur **"+"** en haut à droite pour ajouter un nouveau tableau de bord.
   - **Entrez un nom** pour le tableau de bord (par exemple, `Maduino Zero Dashboard`).
   - **Cliquez sur "Add"**.

3. **Ajouter des widgets :**

   - **Ouvrez le tableau de bord** que vous venez de créer.
   - **Cliquez sur le bouton "Add new widget"** (icône **"+"**).
   - **Sélectionnez un widget** adapté pour vos données (par exemple, un graphique, un indicateur numérique, etc.).
   - **Configurez le widget :**

     - **Data source** : Sélectionnez le dispositif que vous avez créé.
     - **Telemetry keys** : Sélectionnez les clés de télémétrie que vous souhaitez afficher.
     - **Apparence** : Personnalisez l'apparence du widget si nécessaire.

   - **Enregistrez le widget**.

4. **Organisez le tableau de bord :**

   - **Déplacez et redimensionnez les widgets** pour organiser le tableau de bord selon vos préférences.
   - **Enregistrez les modifications** en cliquant sur l'icône de disque en haut à droite.

### Finaliser la configuration dans ChirpStack

- **Dans votre dispositif** dans ChirpStack, allez dans l'onglet **"Configuration"**, puis **"Variables"**.
- **Cliquez sur "**+ Add variable**"**.
- **Dans le champ "Key"**, entrez `ThingsBoardAccessToken`.
- **Dans le champ "Value"**, collez le jeton d'accès copié précédemment.
- **Cliquez sur "Add"** pour enregistrer la variable.

**Résultat :** Votre dispositif devrait maintenant envoyer les données à ChirpStack, qui les transmettra à ThingsBoard. Vous pourrez visualiser les données sur votre tableau de bord personnalisé dans ThingsBoard.

## Vérification du Fonctionnement

- **Dans ChirpStack :**

  - **Utilisez les onglets "Device data" et "Live LoRaWAN frames"** du dispositif pour vérifier que les données sont bien reçues.

- **Dans ThingsBoard :**

  - **Allez dans le dispositif concerné**.
  - **Vérifiez l'onglet "Latest telemetry"** pour vous assurer que les données sont reçues.
  - **Accédez à votre tableau de bord** pour visualiser les données à travers les widgets que vous avez configurés.

## Profil de Dispositif sur ChirpStack

En créant manuellement le profil de dispositif, vous pouvez personnaliser les paramètres selon les besoins de votre dispositif. Si vous utilisez un codec spécifique pour le décodage des payloads (comme **CayenneLPP**), assurez-vous de le configurer dans le profil de dispositif :

1. **Dans le profil de dispositif**, allez à l'onglet **"Codec"**.
2. **Sélectionnez "CayenneLPP"**.



## Informations sur le Maduino Zero

Le **Maduino Zero** comporte deux microcontrôleurs :

- **SAMD21G18A** : Utilisé pour programmer la carte et pour communiquer via USB avec le PC.
- **ASR6501** : Utilisé pour la communication LoRaWAN avec des passerelles telles que le Sentrius.

Les communications entre les deux microcontrôleurs se font via l'UART (Serial). Les ports sont :

- `SerialUSB` : Communication série avec le PC.
- `Serial1` : Communication série entre les microcontrôleurs (baudrate : 115200).

Un code d'exemple est disponible pour envoyer des commandes AT au microcontrôleur **RA-07H** via la connexion série USB, ce qui facilite le dépannage et la compréhension du fonctionnement du Maduino Zero.

## Informations Diverses

### Identifiants par Défaut

- **ChirpStack :**

  - **Admin** : `admin` / `admin`

- **ThingsBoard :**

  - **System Administrator** : `sysadmin@thingsboard.org` / `sysadmin`
  - **Tenant Administrator** : `tenant@thingsboard.org` / `tenant`
  - **Customer User** : `customer@thingsboard.org` / `customer`

- **Sentrius :**

  - **Login** : `sentrius` / `RG1xxx`

## Références

- [CayenneLPP](https://github.com/myDevicesIoT/CayenneLPP)
- [Arduino LoRaWAN Device Library](https://github.com/TheThingsNetwork/arduino-device-lib)
- [ASR650X AT Command Introduction](https://www.hoperf.com/data/upload/back/20190605/ASR650X%20AT%20Command%20Introduction-20190605.pdf)
- [Maduino Zero LoRaWAN](https://github.com/Makerfabs/Maduino-Zero-Lorawan/tree/Ra07)
- [Github chirpstack docker](https://github.com/chirpstack/chirpstack-docker)
- [Documentation ChirpStack](https://www.chirpstack.io/docs/)
- [Documentation ThingsBoard](https://thingsboard.io/)
