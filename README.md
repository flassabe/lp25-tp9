# TP C n°4 : communication entre processus

Dans ce TP, l'objectif est d'écrire des programmes capables d'interagir avec le système ou entre eux.

Nous y verrons notamment la lecture de données envoyées au proggramme par un `pipe`, l'interception de signaux destinés au programme, l'utilisation des FIFO, et le multiplexages sur des entrées multiples.

## Ouvrir la sortir d'un pipe depuis un programme

Pour ouvrir le pipe en lecture (donc réceptionner le données qui y ont été envoyées), il est nécessaire de lire l'entrée standard, par exemple en appelant la fonction `fgets` sur le descripteur `stdin`.

## Exercice 1 - ouvrir le flux d'un *pipe*

Le programme de cet exercice sera nommé `pipe-cat.c` et pourra soit lire et afficher un fichier passé en argument du programme, soit afficher ce qui lui est transmis par un pipe.

## Exercice 2 - *pipe* et pointeur de fonction

Ce second programme est nommé `pipe-functer.c` et reprend le principe du premier exercice, mais il va cette fois appliquer à chaque caractère du flux d'entrée une fonction définie par un pointeur de fonction.

Le pointeur de fonction aura le type suivant :
```c
typedef void (*ptr_function)(char);
```
Cette déclaration définit un type nommé `ptr_function` qui pointe sur une fonction renvoyant `void` et prenant un `char` en paramètre.

Par exemple, la fonction :
```c
void make_caps(char c) {
	putc(toupper(c), stdout	);
}
```
va afficher le caractère transmis en capitale et on peut déclarer un pointeur de fonction comme :
```c
int main(int argc, char *argv[]) {
	ptr_function mon_ptr_fonction = make_caps;
	int c;
	while ((c = fgetc(stdin)) != EOF) {
		mon_ptr_fonction((char) c);
	}
	return 0;
}
```

Définir deux autres fonctions `make_small` (affiche tout en minuscule), et `remove_lines` (n'affiche pas les retours à la ligne) et tester les appels à ces fonctions via le pointeur (en changeant la fonction affectée lors de l'affectation de `mon_ptr_fonction`). Utilisez un fichier texte avec quelques lignes et un mélange de majuscules et minuscules.

## Exercice 3 - interception des signaux

Cet exercice vise à intercepter le signal d'interruption du programme pour terminer ce dernier correctement en libérant les ressources du programme. Pour cela, nous utiliserons l'API C permettant de manipuler les signaux.

Le programme que vous devez écrire est un programme qui ouvre un fichier texte en écriture, puis boucle indéfiniment sur le comportement suivant :

 - demander à l'utilisateur de saisir une phrase
 - écrire la phrase dans le fichier

Le programme s'arrête quand l'utilisateur presse la combinaison de touches `Ctrl-c`, ce qui envoie le signal `SIGINT` au programme pour lui demander de se fermer.

### Étape 1 : pas de gestion du signal

Pour débuter, copier dans my-signal.c, compiler et exécuter le programme suivant :

```c
#include <stdio.h>

#define BUFFER_SIZE 150

int main(int argc, char *argv[]) {
	FILE *f = fopen("dump.txt", "w");
	if (f) {
		char buffer[BUFFER_SIZE];
		while (1) {
			scanf("%s", buffer);
			fprintf(f, "%s", buffer);
		}
	}
	fclose(f);
	return 0;
}
```

Saisir quelques lignes de texte, puis quitter le programme avec la combinaison de touches `Ctrl-c`. Lire le contenu du fichier dump.txt. Qu'observez-vous ?

### Étape 2 : gérer le signal avec `signal`

Ajouter au programme une variable globale nommée `sigint_trigerred` et de type et qualificateur `volatile sig_atomic_t` qui servira de condition de continuation de la boucle (vous remplacerez donc `while (1)` par `while (!sigint_triggered)`).

Écrire une fonction pour gérer le signal `SIGINT`, qui basculera la valeur de `sigint_triggered` de 0 à 1. le prototype de cette fonction sera `void sigint_handler(int signal)`.

Lier cette fonction au signal `SIGINT` à l'aide de la fonction `signal`. Tester à nouveau le programme. Qu'observez-vous ?

### Étape 3 : utiliser `sigaction`

La dernière modification consiste à utiliser sigaction pour résoudre les écueils observés dans les deux premières étapes. Pour cela vous devez créer une structure de type `struct sigaction` qui appliquera la fonction `sigint_handler` précédemment définie au signal `SIGINT`. Vous devrez également affecter le flag `SA_RESETHAND` au champ `sa_flags` de la structure, et la valeur `NULL` à son champ `sa_restorer`.

Vous remplacerez ensuite l'appel à `signal` par un appel à la fonction `sigaction` (cf. `man sigaction` pour plus d'informations).

Testez cette nouvelle version du programme et observez le résultat. Il reste un dernier défaut (la dernière phrase entrée est écrite deux fois dans le fichier de sortie)

### Étape 4 : ne pas écrire deux fois la même phrase

Implémenter une des solutions suivantes pour terminer l'exercice :

 - remplacer la boucle `while` par une boucle `do...while`
 - tester la valeur de `sigint_triggered` avant de faire le fprintf
 - affecter le caractère nul au premier caractère de `buffer` après l'avoir écrit dans le fichier.

Par ailleurs, le programme actuel n'est pas très fiable car il permet facilement de l'attaquer en débordant le buffer de lecture. Remplacer `scanf` et `fprintf` par des fonctions qui limitent la taille des données lues/écrites, par exemple `read` et `write`

## Exercice 4 : multiplexage des entrées/sorties

Il est fréquent que des programmes recoivent des données sur plusieurs canaux différents. La capacité à les traiter, quelque soit l'ordre d'arrivée des données sur les canaux est le multiplexage.

### 5.1 Préparation du contexte

Pour les exercices qui suivent, nous utiliserons des FIFO, qui sont un type de fichier particulier destiné à être consommé à la volée. Ce qui est envoyé dans le fichier, par exemple avec une redirection, est lu par le lecteur du fichier, puis enlevé.

Créer un programme `lecture-fifo.c` qui ouvre une FIFO en lecture (il suffit de l'ouvrir comme un fichier standard avec l'option "r" pour la lecture de caractères). Pour lire dans ce fichier, nous utiliserons la fonction `read`, qui ne lit pas dans un pointeur de `FILE` mais dans un descripteur de fichier.

Pour obtenir ce descripteur, vous utiliserez la fonction `fileno` qui prend en paramètre un `FILE *` déjà ouvert avec `fopen`, et qui retourne son descripteur (c'est un entier).

Le reste du programme consiste en une boucle infinie (un `while(1)` fera l'affaire), à l'intérieur de laquelle vous appellerez la fonction `read` et en afficherez le résultat à chaque fois qu'elle retourne (i.e. quand elle a lu quelque chose).

Pour tester le fonctionnement du programme, créez une FIFO dans le répertoire duquel vous exécutez le programme, grâce à la commande `mkfifo fifo1` qui créera la FIFO nommée `fifo1`. Vous devez voir cette dernière en affichant la liste de vos fichiers avec la commande `ls`. Vous pouvez ensuite lancer le programme et, depuis un second terminal, envoyer du texte dans la FIFO (commande `echo "quelque chose" > fifo1`). Le programme doit alors afficher ce que vous lui avez envoyé.

Ajoutez ensuite une gestion du signal `SIGINT` pour terminer proprement le programme quand vous l'interrompez avec `Ctrl-c` ou tout autre moyen de lui envoyer le signal `SIGINT`.

### 5.2 Ajout d'une seconde FIFO

Nous allons maintenant ajouter une seconde FIFO (`mkdir fifo2`), que nous ouvrirons dans le programme (toujours en lecture) et pour lequel nous ajouterons également une lecture (après la lecture du descripteur de `fifo1`).

Testez le programme avec les commandes :

```bash
echo "blah" > fifo1
echo "hello world!" > fifo2
```

```bash
echo "blah" > fifo2
echo "hello world!" > fifo1
```

Que se passe-t-il dans le second cas ? Le problème est que les appels à `read` sont bloquants, comme la plupart des fonctions de lecture le sont par défaut.

### 5.3 Multiplexage par polling

Pour permettre la lecture dans un ordre indéterminé, nous allons utiliser la méthode la plus naïve, qui consiste à rendre les descripteurs de fichiers non bloquants afin que l'appel à `read` ne bloque pas en attente d'une donnée. Pour ceci, nous ouvrirons directement les fichiers avec `open` :

```c
int fifo1_desc = open("fifo1", O_READONLY | O_NONBLOCK); // 1st flag can be O_RDWR
// Then same for FIFO2
```

Ensuite, dans la boucle, nous allons utiliser `read` et n'afficher un résultat que si la valeur de retour est strictement supérieure à `0`. Sinon, cela veut dire qu'il n'y a rien à lire, `read` ayant retourné car le fichier est en mode non bloquant.

Écrire un programmé nommé `multiplexage-polling.c` qui met en application le polling et testez le en envoyant des données dans les FIFO avec des ordres différents pour vous assurer que cela fonctionne. Puis, regardez la charge CPU du programme (commande bash `top`, puis appui sur la touche `c` pour classer par occupation du CPU de la plus forte à la plus faible). Que remarquez vous ? Est-ce intéressant ?

### 5.4 Multiplexage avec `select`

Pour résoudre le problème de multiplexage et de charge CPU, nous allons utiliser la fonction `select` qui permet de scruter plusieurs descripteurs et de retourner quand un descripteur (ou plusieurs) ont des données à consommer.

Lisez le manuel de la fonction `select` (`man 3 select`). En quelques mots, cette fonction nécessite qu'on lui passe des ensembles de descripteurs de fichiers, qu'elle renverra ensuite modifiés avec un marquage pour les descripteurs qui ont du contenu à lire (`select`  gère aussi l'écriture, mais nous ne le verrons pas ici).

Son utilisation se fait en 3 temps (qui doivent être répétés dans la boucle de lecture) :

 - Initialisation des `fd_set` (les ensembles de descripteurs de fichiers)
 - Appel à `select` en lui passant les ensembles de descripteurs de fichiers
 - Après retour de `select`, parcours des ensembles de FD (_file descriptors_ pour descripteurs de fichiers) pour trouver lesquels ont des données à lire, et lecture effective des données en attente.

**Remarque :** pour permettre le fonctionnement de `select`, vos fichiers devront être ouverts en mode `"r+"` sans quoi `select` retournera sans arrêt alors qu'il n'y a rien à lire.

#### Étape 1 : préparation des `fd_set`

Un ensemble de FD est de type `fd_set` et est manipulé grâce à des macros. Avant d'entrer dans la boucle, le `fd_set` doit être déclaré et initialisé.

```c
fd_set read_fds;
/*...*/
while(1) {
	FD_ZERO(&read_fds);
	FD_SET(fifo1_desc, &read_fds);
	FD_SET(fifo2_desc, &read_fds);
	/**/
}
```

Cet extrait de code permet de déclarer le FD set, puis de le réinitialiser au début de la boucle, afin de lui réaffecter les descripteurs de fichiers sur `fifo1` et `fifo2`.

#### Étape 2 : appel de `select`

La fonction `select` prend en premier paramètre la valeur du plus grand DF +1, en second paramètre un pointeur vers l'ensemble de FD en lecture (ici, `read_fds`), puis un pointeur vers le `fd_set` en écriture (ici, il sera `NULL`), puis un pointeur vers le `fd_set` des exceptions (aussi `NULL` dans notre cas), puis un pointeur vers une structure `timeval`. Cette structure définit à quel moment `select` retournera (avec comme valeur de retour `0`) même si rien n'a été lu sur les FD surveillés par `select`. On remarque que l'appel à `select` avec aucun descripteur à surveiller permet de faire un timer.

Dans le cadre de notre TP, l'appel sera similaire à ceci :

```c
int max_fd = (fifo1_desc > fifo2_desc) ? fifo1_desc : fifo2_desc;
struct timeval select_timeout = {.tv_sec=1, .tv_usec=0}; // timeout: 1 second, 0 µs
int select_result = select(max_fd+1, &read_fds, NULL, NULL, &select_timeout);

/* ... Do something with select results ... */
```

#### Étape 3 : utilisation du résultat de `select`

Il y a principalement 3 cas à vérifier concernant le résultat de `select` :

 - La valeur de retour `-1` : `select` a généré une erreur, il est raisonnable de considérer à ce moment là que le programme doit être arrêté.
 - La valeur de retour `0` (uniquement si un timeout est passé en paramètre) : `select` n'a rien trouvé à lire sur ses descripteurs dans la durée du timeout. Il a donc retourné, mais aucun descripteur ne sera disponible à la lecture.
 - Une valeur `>0` : `select` a des données à lire sur un ou plusieurs descripteurs (un nombre égal à la valeur de retour). Il faut alors parcourir les `fd_set` et tester si les descripteurs qui y sont enregistrés sont "actifs", c'est-à-dire qu'ils ont des données en attente de lecture.

Les deux premiers cas sont traités selon la logique de votre application. Pour le dernier cas, le traitement est généralement d'une forme similaire à ceci (basé sur le contexte de notre exemple) :

```c
if (FD_ISSET(fifo1_desc, &reaf_fds)) // If FD has data available
	if (read(fifo1_desc, buffer, BUFFER_SIZE) > 0) // Try to read
		printf("%s", buffer); // You shall add a '\0' before printing since read doesn't do it
if (FD_ISSET(fifo2_desc, &reaf_fds))
	if (read(fifo2_desc, buffer, BUFFER_SIZE) > 0)
		printf("%s", buffer); // You shall add a '\0' before printing since read doesn't do it
```

Puis, on recommence la boucle, jusqu'à son interruption par une condition particulière ou un signal.

À l'aide des instructions ci-dessus, faites en sorte d'écrire un programme nommé `multiplexage-select.c` qui utilise `select` pour multiplexer les lectures sur des FIFO. Vous pouvez ajouter des FIFO, tester de les alimenter dans n'importe quel ordre, etc. Regardez également la consommation CPU de votre programme.

## Conclusion

Dans ce TP, vous avez vu tout d'abord comment communiquer entre le système et vos programmes. Puis, vous avez vu deux manières de multiplexer des lectures de données dans un programme en C (il en existe d'autres). La fonction `select` est aussi utilisable pour la lecture sur l'entrée standard (le descripteur de filchier étant `stdin`), ainsi que sur les _sockets_, qui sont une API de communication réseau. Cette API est d'ailleurs abordée dans une UV de la filière réseau de la formation en informatique de l'UTBM.
