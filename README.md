Écrire un shell en C

15 janvier 2018

Vous vous êtes déjà demandé comment ce terminal fonctionne ? Une question plus précise serait : comment fonctionne ce shell ? Je me suis posé la même question, et il m’a fallu quelques recherches et beaucoup de lecture pour comprendre. J’ai écrit un shell en C, d’abord basique, puis j’y ai ajouté quelques fonctionnalités supplémentaires. Voici le processus que j’ai suivi pour écrire un shell de base en C.
Shell

Qu’est-ce qu’un shell ? De manière très simple, on peut le définir comme un outil ou un programme par lequel vous pouvez (et devriez) interagir avec le système d’exploitation. Cette définition est assez vague, mais elle donne une idée générale.
Prérequis

    Connaissances en programmation C

    gcc

    Éditeur de texte

Démarrage du shell

Voici la fonction principale de la boucle. Voyons comment les choses sont implémentées et comment elles fonctionnent au démarrage du shell. Tout d’abord, une invite permet à l’utilisateur de savoir que le terminal est prêt à recevoir des commandes. Elle peut être largement personnalisée, mais pour simplifier, nous utiliserons un symbole > classique.

void loop() {
   char *line;
   char **args;
   int status = 1;

   do {
      printf("> ");
      line = read_line();
      flag = 0;
      args = split_lines(line);
      status = dash_launch(args);
      free(line);
      free(args);
   } while (status);
}

Voyons les parties importantes. On déclare deux pointeurs char : line (une chaîne simple) et args (un tableau 2D). La variable line contiendra la commande saisie par l’utilisateur grâce à la fonction read_line(), expliquée ci-dessous. La variable status stocke la valeur de retour des fonctions appelées lors de l'exécution de la commande. Si l'utilisateur entre la commande exit, la fonction correspondante renverra 0, ce qui fera sortir de la boucle et quittera le shell.

Les deux dernières lignes de la boucle do-while libèrent la mémoire utilisée.
Lecture des commandes utilisateur

char *read_line() {
  int buffsize = 1024;
  int position = 0;
  char *buffer = malloc(sizeof(char) * buffsize);
  int c;

  if (!buffer) {
    fprintf(stderr, "%sdash: Erreur d'allocation%s\n", RED, RESET);
    exit(EXIT_FAILURE);
  }

  while (1) {
    c = getchar();
    if (c == EOF || c == '\n') {
      buffer[position] = '\0';
      return buffer;
    } else {
      buffer[position] = c;
    }
    position++;

    if (position >= buffsize) {
      buffsize += 1024;
      buffer = realloc(buffer, buffsize);

      if (!buffer) {
        fprintf(stderr, "dash: Erreur d'allocation\n");
        exit(EXIT_FAILURE);
      }
    }
  }
}

Ici, on commence par allouer dynamiquement un buffer de 1024 octets. La mémoire est allouée dynamiquement car on ne peut pas prédire la longueur de la commande entrée.

La boucle while lit les caractères un par un avec getchar(). Si on atteint EOF ou un saut de ligne (\n), on termine la chaîne avec \0 et on la retourne. Sinon, on stocke le caractère dans le buffer.

Si le buffer atteint sa taille maximale, on la double et on réalloue avec realloc.
Découpage de la commande (tokenization)

On va maintenant découper la chaîne en mots (tokens) à l’aide de strtok.

Exemple :

str1 = strtok("ceci est un test", " ");  // str1 -> "ceci"
str1 = strtok(NULL, " ");                // str1 -> "est"
str1 = strtok(NULL, " ");                // str1 -> "un"

Code :

char **split_line(char *line) {
  int buffsize = 1024, position = 0;
  char **tokens = malloc(buffsize * sizeof(char *));
  char *token;

  if (!tokens) {
    fprintf(stderr, "%sdash: Erreur d'allocation%s\n", RED, RESET);
    exit(EXIT_FAILURE);
  }

  token = strtok(line, TOK_DELIM);
  while (token != NULL) {
    tokens[position] = token;
    position++;

    if (position >= buffsize) {
      buffsize += TK_BUFF_SIZE;
      tokens = realloc(tokens, buffsize * sizeof(char *));
      if (!tokens) {
        fprintf(stderr, "%sdash: Erreur d'allocation%s\n", RED, RESET);
        exit(EXIT_FAILURE);
      }
    }
    token = strtok(NULL, TOK_DELIM);
  }

  tokens[position] = NULL;
  return tokens;
}

Quitter le shell

Une fonction simple qui renvoie 0 :

int dash_exit(char **args)
{
	return 0;
}

Exécution des commandes

Voici l’exécution de la commande :

int dash_execute(char **args) {
  pid_t cpid;
  int status;

  if (strcmp(args[0], "exit") == 0)
    return dash_exit(args);

  cpid = fork();

  if (cpid == 0) {
    if (execvp(args[0], args) < 0)
      printf("dash: commande introuvable : %s\n", args[0]);
    exit(EXIT_FAILURE);
  } else if (cpid < 0)
    printf(RED "Erreur lors du fork" RESET "\n");
  else
    waitpid(cpid, &status, WUNTRACED);

  return 1;
}

    fork() crée un processus enfant.

    Dans le processus enfant (cpid == 0), on utilise execvp() pour exécuter la commande.

    waitpid() dans le parent attend la fin de l’enfant.

Code complet du shell

#include <stdio.h>
#include <string.h>
#include <stdlib.h>

#define TOK_DELIM " \t\r\n"
#define RED "\033[0;31m"
#define RESET "\e[0m"

char *read_line();
char **split_line(char *);
int dash_exit(char **);
int dash_execute(char **);

int dash_execute(char **args) {
  pid_t cpid;
  int status;

  if (strcmp(args[0], "exit") == 0)
    return dash_exit(args);

  cpid = fork();

  if (cpid == 0) {
    if (execvp(args[0], args) < 0)
      printf("dash: commande introuvable : %s\n", args[0]);
    exit(EXIT_FAILURE);
  } else if (cpid < 0)
    printf(RED "Erreur lors du fork" RESET "\n");
  else
    waitpid(cpid, &status, WUNTRACED);

  return 1;
}

int dash_exit(char **args) {
  return 0;
}

char **split_line(char *line) {
  int buffsize = 1024, position = 0;
  char **tokens = malloc(buffsize * sizeof(char *));
  char *token;

  if (!tokens) {
    fprintf(stderr, "%sdash: Erreur d'allocation%s\n", RED, RESET);
    exit(EXIT_FAILURE);
  }

  token = strtok(line, TOK_DELIM);
  while (token != NULL) {
    tokens[position] = token;
    position++;

    if (position >= buffsize) {
      buffsize += 1024;
      tokens = realloc(tokens, buffsize * sizeof(char *));
      if (!tokens) {
        fprintf(stderr, "%sdash: Erreur d'allocation%s\n", RED, RESET);
        exit(EXIT_FAILURE);
      }
    }

    token = strtok(NULL, TOK_DELIM);
  }

  tokens[position] = NULL;
  return tokens;
}

char *read_line() {
  int buffsize = 1024, position = 0;
  char *buffer = malloc(sizeof(char) * buffsize);
  int c;

  if (!buffer) {
    fprintf(stderr, "%sdash: Erreur d'allocation%s\n", RED, RESET);
    exit(EXIT_FAILURE);
  }

  while (1) {
    c = getchar();
    if (c == EOF || c == '\n') {
      buffer[position] = '\0';
      return buffer;
    } else {
      buffer[position] = c;
    }
    position++;

    if (position >= buffsize) {
      buffsize += 1024;
      buffer = realloc(buffer, buffsize);
      if (!buffer) {
        fprintf(stderr, "dash: Erreur d'allocation\n");
        exit(EXIT_FAILURE);
      }
    }
  }
}

void loop() {
  char *line;
  char **args;
  int status = 1;

  do {
    printf("> ");
    line = read_line();
    args = split_line(line);
    status = dash_execute(args);
    free(line);
    free(args);
  } while (status);
}

int main() {
  loop();
  return 0;
}

Conclusion

Ce projet est avant tout un exercice d’apprentissage plutôt qu’un shell complet. Vous n’utiliserez probablement jamais un shell aussi simple, mais maintenant vous comprenez mieux comment fonctionnent vos shells préférés sous le capot.

J’ai aussi écrit une version plus avancée de ce shell avec support des pipes, de l’historique et d’autres commandes internes.
Contribuer

Si vous trouvez des erreurs ou pensez que cet article peut être amélioré, n’hésitez pas à me contacter.

Copyright © Danish Prakash 2025