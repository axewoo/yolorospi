## 1. Arborescence des Fichiers du Projet

Voici la structure complète de votre espace de travail ROS 2 (~/ros2_hailo_ws) incluant les fichiers créés et modifiés :
Plaintext

~/ros2_hailo_ws/
├── src/
│   └── hailo_person_detection/
│       ├── package.xml                 # Déclaration des dépendances (rclpy, sensor_msgs, etc.)
│       ├── setup.py                    # Configuration du package et point d'entrée de l'exécutable
│       ├── setup.cfg                   # Fichier de configuration standard pour Python sous ROS 2
│       ├── launch/
│       │   └── hailo_launch.py         # [Optionnel/Suggéré] Script de lancement avec reconnexion automatique (respawn)
│       └── hailo_person_detection/
│           ├── __init__.py
│           └── hailo_inference.py      # Code principal : Capture ROS 2, Inférence Hailo-8 et dessin OpenCV
└── home/
    └── raspespion/
        └── yolov8n.hef                 # Le modèle YOLOv8n compilé pour le hardware Hailo

Rôle des fichiers clés :

    hailo_inference.py : C'est le cœur applicatif. Il s'abonne au flux de la caméra, convertit les images compressées, les formate pour YOLOv8, pilote l'accélérateur matériel Hailo-8, décode le post-processing NMS natif de la puce, dessine les boîtes de détection et publie le flux annoté.

    hailo_launch.py : Permet de lancer simultanément le driver de la caméra et le nœud Hailo. Il intègre la sécurité de redémarrage automatique en cas de défaillance matérielle.

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
