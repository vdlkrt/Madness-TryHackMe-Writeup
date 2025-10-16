# Madness-TryHackMe-Writeup

Nmap -sC -sV -T5 -p- 10.10.176.148  -o nmap.txt 

Découverte de 2 ports ouverts : 22 et 80 ( SSH et HTTP); Apache 2.4.18, et OPENSSH 7.2p2.

gobuster dir -u 10.10.176.148 -w /usr/share/wordlists/dirb/common.txt -t 90 
Aucun résultats de gobuster.

Bloqué. Je relis donc le code source de la page plus attentivement.

Bingo, "They will never find me" dans le code source. qui nous ramène a une image cassée sur le haut de la page. J'aurai du faire plus attention.


wget http://madness.com/thm.jpg

file thm.jpg (data)

strings thm.jpg

Rien d'intéressant, mais la commande file me laisse penser que le fichier est corrompu pour X ou Y raisons.

Je décide directement d'utiliser "steghide extract -sf thm.jpg" mais on me demande un passphrase.

Je check les magic numbers grace a "hexedit thm.jpg" et je découvre quelque chose d'étrange.

En effet, les magics numbers affichés sont : "89 50 4e 47" ce sont la signature des fichiers PNG et non JPEG.
Grace a cette ressource : https://gist.github.com/leommoore/f9e57ba2aa4bf197ebc5 je trouve les magic numbers d'un fichier JPEG, et je remplace les anciens via hexedit.

je vérifie : file thm.jpg 
output : thm.jpg: JPEG image data
Je devrais donc etre en mesure d'ouvrir l'image.
J'essaye mais ça ne marche pas.

Après recherches, je trouve ceci : https://en.wikipedia.org/wiki/List_of_file_signatures (CTRL+F : JPEG)
Encore hexedit : cette fois ci je rentre comme valeur : FF D8 FF E0 00 10 41 46
file thm.jpg output : thm.jpg: JPEG image data, baseline, precision 8, 400x400, components 3
Ca m'a l'air mieux.

En ouvrant l'image cette fois ci bingo ! on nous indique un hidden directory : /th1s_1s_h1dd3n
En arrivant sur la page indiquée, on nous demande de rentrer un secret.
Le premier réflexe que j'ai est de regarder le code source : on tombe sur "<!-- It's between 0-99 but I don't think anyone will look here-->"
Je lance burpsuite, puis je me dis que ffuf sera plus pratique..

je crée ma liste : seq 0 99 > list.txt
je teste le param dans l'URL : http://madness.com/th1s_1s_h1dd3n/?secret=1
Le résultat est bien affiché sur la page, c'est validé.
je lance ffuf : ffuf -u 'http://madness.com/th1s_1s_h1dd3n/?secret=FUZZ' -w /home/kali/list.txt -mc 200 (-mc 200 sert a filtrer les réponses HTTP 200)

l'output de ffuf indique 45 mots pour tous, sauf 73. J'entre donc http://madness.com/th1s_1s_h1dd3n/?secret=73, et bingo.
Mais encore dans batons dans les roues : Urgh, you got it right! But I won't tell you who I am! y2RPJ4QaPF!B
Il faut maintenant décoder y2RPJ4QaPF!B
La j'ai compris pourquoi la box s'appelle madness. J'ai essayé de décoder "y2RPJ4QaPF!B" pendant longtemps. Puis je me suis rappelé de ma tentative de steghide qui me demandait un passphrase. J'ai réessayé, et bingo. 

steghide extract -sf thm.jpg :wrote extracted data to "hidden.txt".

cat hidden.txt : Fine you found the password! 

"Here's a username 

wbxre

I didn't say I would make it easy for you!"

Petit roublard..

Petit rappel de notre scan nmap, on sait qu'il y'a un SSH sur le port 22.
on essaye de se connecter via SSH, en utilisant le meme mdp que sur le steghide, mais sans succès..
Puis je me rappelle de mes tentatives de décoder avec le ROT Cipher, (le hint TryHackMe confirma ma piste) et j'obtiens : joker

Ici j'ai bloqué pendant très longtemps. Mais j'ai fini par trouver.
Ayant compris pourquoi "Madness" assez tôt, j'ai décidé de réfléchir comme un psychopathe et de tout reprendre depuis le début. Je me suis rendu compte après longtemps que toutes mes pistes étaient épuisées. J'ai donc décidé de faire quelque chose que je ne pensais même pas possible, mais qui m'a paru évident quand j'ai revu l'image. Oui, l'image de https://tryhackme.com/room/madness, sur la page de la box.
Je l'ai téléchargée dans le doute, puis j'ai essayé:

steghide info 5iW7kC8.jpg
Il m'a demandé un passphrase, et je n'en ai pas cru mes yeux..
Ne sachant pas comment l'obtenir, j'ai demandé l'aide de ChatGPT, qui m'a recommandé stegseek
Ca n'a pas raté : stegseek --crack 5iW7kC8.jpg 

"StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: ""
[i] Original filename: "password.txt".
[i] Extracting to "5iW7kC8.jpg.out".
cat 5iW7kC8.jpg.out
"I didn't think you'd find me! Congratulations!

Here take my password

*axA&GF8dP"

La, je revis.
Je réessaye de me connecter via SSH. 
ssh joker@10.10.176.148 password : *axA&GF8dP


DIEU. MERCI. ENFIN.
Premier ressenti : j'ai très peur de la "madness" de l'escalade de privilèges..
Ma machine THM a expiré tellement je me suis creusé la tête. Changement d'IP.

Chaque chose en son temps, tout d'abord le user flag.
tout simplement cat user.txt : THM{d5781e53b130efe2f94f9b0354a5e4ea}

sudo -l : pas de sudo pour joker.
Enumerer les binaires qui ont des permissions SUID : find / -perm -u=s -type f 2>/dev/null
output:
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/bin/vmware-user-suid-wrapper
/usr/bin/gpasswd
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/sudo
/bin/fusermount
/bin/su
/bin/ping6
/bin/screen-4.5.0
/bin/screen-4.5.0.old
/bin/mount
/bin/ping
/bin/umount


(Je ne le savais pas, mais j'avais la réponse sous les yeux. J'ai cherché /bin/screen-4.5.0 sur gtfobins, mais aucun résultat. C'était la bonne réponse.)

Après beaucoup d'énumération manuelle ainsi que via LinPeas et de fausses pistes, j'ai décidé de rechercher /bin/screen-4.5.0 exploit sur Google, et j'ai trouvé.

J'ai donc tout simplement copié collé le code de cet exploit : https://www.exploit-db.com/exploits/41154 dans un fichier nommé exploit.sh, (directement créé avec nano sur la machine distante)

On modifie les permissions avec chmod +x, puis on exécute le fichier avec ./exploit.sh 

on vérifie : id 
uid=0(root) gid=0(root) groups=0(root),1000(joker)
Bingo !

Et voilà. Je me suis un peu trop creusé la tête sur l'escalade de privilèges, du fait de la "Madness" précédente..
Flag root : cat /root/root.txt
THM{5ecd98aa66a6abb670184d7547c8124a} 

Une box qui m'a permis de m'apprendre à revenir en arrière, et penser en dehors des cheminements classiques. J'ai découvert des outils de stéganographie très utiles. J'ai aussi appris qu'il fallait garder la tête froide et rester objectif. Rester simple et efficace, et quand rien ne marche, savoir changer totalement sa manière de penser. Je retiens aussi qu'il faut que je recherche absolument CHAQUE potentielles pistes en profondeur, pas seulement sur gtfobins etc.

Surement la box la plus amusante que j'ai pu faire !



