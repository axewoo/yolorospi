## 1. Arborescence des Fichiers du Projet

```text
home/
└── ros2_hailo_ws/
    └── src/
        └── hailo_person_detection/
            ├── package.xml                 # Déclaration des dépendances ROS 2 (rclpy, sensor_msgs...)
            ├── setup.py                    # Configuration du package et points d'entrée des exécutables
            ├── setup.cfg                   # Configuration standard d'installation Python pour ROS 2
            ├── launch/
            │   └── hailo_launch.py         # Script de lancement avec gestion du crash de la caméra (respawn)
            └── hailo_person_detection/
                ├── __init__.py
                └── hailo_inference.py      # Nœud principal (Inférence Hailo-8 + Traitement NMS + Affichage OpenCV)
└── yolov8n.hef             # Le modèle YOLOv8n compilé pour le hardware Hailo-8
```
Rôle des fichiers clés :

* **`hailo_inference.py`** : C'est le cœur applicatif. Ce nœud ROS 2 s'abonne au topic de la caméra (`/camera/image/compressed`), convertit l'image, exécute l'inférence sur la puce Hailo-8, décode le tenseur NMS natif, dessine les boîtes de détection via OpenCV et republie l'image annotée.
* **`hailo_launch.py`** : Permet de lancer simultanément le driver de votre caméra et le nœud d'inférence Hailo. Il intègre une sécurité `respawn=True` pour reconnecter automatiquement la caméra en cas de coupure ou de surchauffe matérielle.
* **`yolov8n.hef`** : Le fichier du modèle YOLOv8n converti et optimisé spécifiquement pour l'architecture du processeur Hailo.

## 2. Les Pièges Rencontrés et Résolutions (R RETEX / Post-Mortem)

Au cours du développement, nous avons fait face à 4 pièges techniques majeurs (logiciels et matériels). Voici leur documentation pour éviter qu'ils ne se reproduisent :

🧠 Piège 1 : Confusion de syntaxe dans l'API Hailo (Médiatisation des types)

    Le Problème : Erreurs d'importation et de crashs au démarrage à cause d'une confusion entre les classes de configuration (InputVStreamParams) et les variables d'instance créées par le script.

    La Solution : Respecter strictement la convention Hailo :

        Les Classes importées prennent une Majuscule et sont au singulier : InputVStreamParams.

        Les Variables d'instances de votre nœud prennent une minuscule et se terminent par un "s" : self.input_vstreams_params.

📊 Piège 2 : La structure "poupée russe" du post-processing NMS natif Hailo

    Le Problème : Le flux vidéo s'affichait mais aucune boîte YOLO n'apparaissait. Le terminal indiquait en boucle : Sortie NMS 'yolov8n...' (1 éléments) : [Elément 0: list]. Le modèle .hef intégrait le NMS (Non-Maximum Suppression) directement dans le silicium, encapsulant les matrices de détection dans une liste, elle-même stockée dans une liste de Batch (taille 1). Le code initial attendait un tableau NumPy brut et ignorait donc les données.

    La Solution : Descendre d'un niveau dans la structure en extrayant d'abord nms_data = tensor[0]. Ensuite, analyser dynamiquement la forme (shape) des sous-tableaux pour mapper automatiquement :

        Les coordonnées des boîtes (Format normalisé Hailo : [ymin, xmin, ymax, xmax]).

        Les scores de confiance (filtrés à >30%).

        Les identifiants de classes (IDs).

📺 Piège 3 : Le gel de l'affichage OpenCV en cas d'anomalie

    Le Problème : Dès qu'une exception ou une frame corrompue arrivait dans le bloc de calcul (try:), le script sautait l'instruction d'affichage cv2.imshow(), ce qui faisait figer l'image à l'écran, donnant l'impression que le programme avait planté.

    La Solution : Isolation complète du bloc d'affichage. Les fonctions cv2.imshow() et cv2.waitKey(1) ont été déplacées tout à la fin de la fonction, en dehors et après le bloc try/except de l'inférence. Même si l'accélérateur Hailo ou le décodage NMS rencontre une erreur passagère, le flux vidéo reste fluide et ne fige jamais.

🌡️ Piège 4 : Crash cyclique de la caméra IMX307 + MStar SSC335 (Toutes les 2 min)

    Le Problème : La caméra coupait le flux et disparaissait du réseau de manière ultra-régulière toutes les deux minutes. Ce comportement était d'origine matérielle : le processeur MStar/SigmaStar SSC335 de la caméra entrait en surchauffe thermique (thermal throttling) à cause de l'effort soutenu pour encoder et diffuser le flux vidéo en continu vers ROS 2, déclenchant le Watchdog matériel (reboot forcé).

    La Solution : 1.  Matérielle : Ajout/vérification d'un dissipateur thermique (radiateur alu) sur le SoC SSC335 et fiabilisation de l'alimentation 12V.
    2.  Configuration : Allègement de la charge de la caméra en baissant le Framerate à 15 FPS (amplement suffisant pour l'inférence) et réduction du Bitrate.
    3.  Logicielle (Tolérance aux pannes) : Configuration du nœud driver dans le fichier Launch avec l'argument respawn=True et respawn_delay=2.0. Si la caméra sature et redémarre, ROS 2 ressuscite le pilote automatiquement en tâche de fond dès que la caméra est de nouveau accessible.

## Configuration Matérielle (Hardware)

L'architecture de ce projet repose sur une chaîne d'acquisition et de traitement embarquée performante, conçue pour minimiser la latence tout en garantissant une grande fiabilité industrielle :

* **Calculateur Principal - Raspberry Pi 5 :** Le cœur informatique du système, chargé d'exécuter l'environnement ROS 2, de gérer l'acquisition des flux vidéo compressés et d'orchestrer la communication entre les différents nœuds.
* **Accélérateur IA - Raspberry Pi AI HAT (Hailo-8, 26 TOPS) :** Connecté via l'interface PCIe du Raspberry Pi 5, ce module dédié à l'intelligence artificielle déploie une puissance de calcul de 26 TOPS (Tera Operations Per Second). C'est lui qui prend en charge l'inférence et le post-processing (NMS) en temps réel du modèle YOLOv8n, libérant ainsi totalement le processeur principal.
* **Switch Réseau & Alimentation - Brainboxes PE-515 :** Un commutateur Ethernet industriel PoE (Power over Ethernet) non managé à 5 ports. Il assure à la fois la transmission ultra-rapide des paquets réseau et l'alimentation électrique stable et sécurisée de la caméra IP directement via le câble RJ45.
* **Capteur de Vision - Caméra IMX307 + MStar SSC335 :** Une caméra IP équipée du capteur Sony Starvis IMX307 (reconnu pour ses performances exceptionnelles en basse luminosité) associé au processeur SigmaStar SSC335. Configurée de manière optimale à 15 FPS avec un dissipateur thermique pour éviter tout emballement thermique, elle capture et diffuse le flux vidéo traité ensuite par notre pipeline.

---

## Inspiration et Crédits

Ce projet s'appuie de manière significative sur les concepts et l'architecture du projet open-source **`yoloros`**. Ce dépôt a constitué une base de référence technique indispensable pour comprendre l'intégration des modèles de la famille YOLO au sein de l'écosystème de communication de ROS 2. En s'inspirant de sa logique de nœuds et de la manipulation des types de messages d'images (`sensor_msgs`), nous avons pu concevoir un pont robuste et réactif entre le flux compressé de la caméra industrielle et les capacités de détection d'objets en direct de la puce Hailo-8.
