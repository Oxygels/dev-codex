---
title: "Pointeurs de fonction"
subtitle: "Les pointeurs de fonction en C"
summary: "Article résumant les pointeurs de fonction en C"
date: 2021-01-24T22:16:49+01:00
draft: false
author: "Oxygel.fr"
description: "Les pointeurs de fonction en C"
---

## Origine
On a l'habitude d'écrire des fonctions ou procédures en C. 

Celles-ci nous permettent d'effectuer des traitements conditionnels sur des données d'entrée/sortie.
Ces données peuvent très bien être :
1. Constantes et codées en dur dans le programme.
2. Des paramètres que l'on passe à la fonction.

Supposons que dans un projet vous êtes amené à écrire une fonction comme celle-ci :

```c
int version_one(/*params*/)
{
    // Du code que l'on a factorisé et que l'on peut résumer en un appel
    code_commun(/* params */);
    
    // La partie qui change et que l'on ne peut donc factoriser
    code_variant1(/* params */);
}
```

Vous vous rendez compte que vous avez plein de fonctions qui lui ressemblent car vous y avez écrit plusieurs fois un même code commun.

En tant que bon développeur, on factorise le plus possible ce code commun (`code_commun()`) 
mais on se retrouve avec une partie qui est différente selon les fonctions (`code_variant()`)

Finalement on se retrouve avec plusieurs fonctions qui appellent chacune le code commun 
```c
int version_une(/*params*/)
{
    code_commun(/* params */);
    code_variant1(/* params */);
}

int version_deux(/*params*/)
{
    code_commun(/* params */);
    code_variant2(/* params */);
}

int version_trois(/*params*/)
{
    code_commun(/* params */);
    code_variant3(/* params */);
}
```
{{< admonition type=note title="Remarque" open=true >}}
Ca ressemble de loin aux problèmes que résout la [programmation générique](https://en.cppreference.com/w/cpp/language/templates) mais nous en parlerons plus tard.
Pour le moment la raison d'existence des multiples versions n'est pas à cause d'un type générique mais plutôt de la volonté d'avoir un comportement différent selon la version qu'on utilise.
{{< /admonition >}}


## Exemple
Tout de suite un exemple pour avoir en tête un repère.

{{< admonition type=example title="Exemple" open=true >}}
Imaginons un jeu où un personnage a une position et souhaite avancer, on imagine pour le moment 3 moyens de locomotion :
- Ses jambes
- Son vélo
- Sa voiture

De plus, chaque fois qu'un personnage se déplace, il doit attendre avant d'arriver, ce temps d'attente étant constant selon le véhicule.
{{< /admonition >}}

On se retrouve donc avec cette mise en application
```c
// ====PRMITIVES====

#define MAX_TAB 50

#include <string.h>

typedef struct {
    int x, y;
} Position;

typedef struct {
    Position pos;
    char mail[MAX_TAB]; // Supposé CSTR (NULL TERMINATED STRING)
} Joueur;

const Joueur POISSON_ROUGE = { .pos = {0,0}, .mail = "poisson@bocal.home"};

/**
 * Envoie un mail
 * @param[in] addr_dest L'adresse de destination
 * @param[in] long      Longueur de l'adresse de destination
 */
void envoyer_message(Joueur exp, Joueur dest) { /* ... */ }

/**
 * Bloque le thread pendant une certaine durée
 * @param[in] duree La durée à attendre en secondes
 */
void attendre(int duree) { /* ... */ }

// ====DEPLACEMENTS====

/**
 * Déplace un joueur vers une position en voiture
 * @param[in,out] j     Le joueur à déplacer
 * @param[in]     arr   La position d'arrivée
 */
void se_deplacer_en_voiture(Joueur* j, Position arr) { 
    envoyer_message(j,POISSON_ROUGE);
    
    // verifier_plein();
    // verifier_pneus();
    attendre(5);

    j->pos = arr;
 }

/**
 * Déplace un joueur vers une position à vélo
 * @param[in,out] j     Le joueur à déplacer
 * @param[in]     arr   La position d'arrivée
 */
void se_deplacer_en_velo(Joueur* j, Position arr) {
    envoyer_message(j, POISSON_ROUGE);
    
    // verifier_pneus();
    attendre(15);

    j->pos = arr;
 }

/**
 * Déplace un joueur vers une position à pieds
 * @param[in,out] j     Le joueur à déplacer
 * @param[in]     arr   La position d'arrivée
 */
void se_deplacer_a_pieds(Joueur j*, Position arr) {
    envoyer_message(j, POISSON_ROUGE);
    attendre(30);

    j->pos = arr;
}
```
Le code qui varie ne dépend pas uniquement d'une donnée, ça aurait été le cas si il fallait **juste** attendre plus longtemps entre chaque véhicule. Dans ce cas on aurait juste eu une fonction se déplacer avec un paramètre `int duree` mais ça aurait été trop facile.

Des vérifications supplémentaires doivent être effectuées selon le véhicule, ce qui paraît beaucoup plus réaliste, on ne prend pas les mêmes précautions entre une voiture et simplement marcher dans la rue *(c'est vrai que la vitesse joue un peu)*

## Solutions possibles
Nous avons donc 3 fonctions pour se déplacer, ce nombre est bien **fini** et une factorisation semble tout de même possible.

### Compacter
Nous allons compacter tous ces traitements possibles dans une seule et même fonction `se_deplacer()`
Cela donnerait : 
```c
typedef enum { Pieds, Velo, Voiture } Locomotion;

void se_deplacer(Joueur* j, Position arr, Locomotion moyen) { 
    envoyer_message(j,POISSON_ROUGE);
    switch(moyen) {
        case Pieds:
            attendre(30);
        break;

        case Velo:
            // verifier_pneus();
            attendre(15);
        break;

        case Voiture:
            // verifier_plein();
            // verifier_pneus();
            attendre(5);
        break;

        // Arrive si on modifie l'enum et qu'on appelle cette fonction sans avoir définit les nouveaux cas
        default:
            exit(1); // dans stdlib.h
    }
    j->pos = arr;
}
```

Cette solution semble satisfaisante pour le moment. Cependant on se rend compte qu'elle n'est viable que pour un nombre fini de véhicule mais aussi et surtout **court**. 
Au niveau de la maintenance c'est un désastre, chaque fois que l'on ajoute une variante de `Locomotion` il faut encore agrandir cette fonction, on sent plutôt qu'on a caché le problème sous le tapis.

Regardons d'autres solutions permettant de faciliter la maintenance pendant les mises à jour. On voudrait pouvoir ajouter quelque chose sans avoir besoin de se souvenir des 2000 endroits où il faudrait effectuer des modifications.

### Pointeurs de fonction
Nous allons donc séparer le code précédent de l'implémentation des moyens de locomotion.
Ainsi on a

```c
// joueur.c
#define MAX_TAB 50

typedef struct {
    int x, y;
} Position;

typedef struct {
    Position pos;
    char mail[MAX_TAB]; // Supposé CSTR (NULL TERMINATED STRING)
} Joueur;

const Joueur POISSON_ROUGE = { .pos = {0,0}, .mail = "poisson@bocal.home"};

void se_deplacer(Joueur j, Position arr, void (*deplacement)(void)) { 
    envoyer_message(j,POISSON_ROUGE);
    (*deplacement)(j,arr);
    j.pos = arr;
}
```

```c
// locomotion.c

void depl_voiture() {
    // verifier_pneu();
    // verifier_plein();
    attendre(5):
}

void depl_velo() {
    // verifier_pneus();
    attendre(15);
 }

void depl_pieds() {
    attendre(30);
}
```
La syntaxe `void (*deplacement)(void)` représente un pointeur de procédure nommé `déplacement`(retourne void sans paramètre de sortie) qui ne prend aucun paramètre.
Il s'utilise comme suit:
```c
void depl_pieds() {
    attendre(30);
}
se_deplacer(j,pos, &depl_pieds)
```

L'utilisation de l'opérateur adresse `&` est optionnel pour les pointeurs de fonctions mais le faire me semble plus lisible pour passer un pointeur de fonction. 

{{< admonition type=warning title="Attention !" open=true >}}
A ne surtout pas confondre avec `void (*deplacement)()` qui annonce prendre n'importe quelle fonction tant que le type de retour est respecté (ici `void`), c'est une sorte de pointeurs de fonction passe-partout.

La différence entre les deux dans notre cas est qu'un passage d'une fonction incorrecte (par exemple prenant un `int` en entrée) résultera en un .... **warning** (sur gcc 10.2 avec `-Wall -Wextra`) avec `void (*deplacement)(void)`, en revanche, au niveau de l'appel par le pointeur de fonction, c'est une erreur de compilation (du type trop d'argument). Tandis qu'aucun warning pour `void (*deplacement)()`, c'est le but.
{{< /admonition >}}

Désormais chaque fois que l'on voudra ajouter des moyens de locomotions, on écrira une fonction supplémentaire dans `locomotion.c` (et dans le .h correspondant mais j'en parle pas ici car ce n'est pas l'objectif).
A chaque appel, de `se_deplacer()`, on passera une fonction de déplacement comme celles ci-dessus.

En quoi cette solution est-elle différente du problème original ?

1. On ne répète plus le code commun
2. On n'appelle plus différentes fonctions mais une seule et même fonction dont les arguments varient.
   La maintenance est donc plus facile et en plus on peut laisser du code client créer ses propres locomotions (mods par exemple).

## Pointeurs de fonction pour structure générique
Dès qu'on manipule une structure de la forme 
```c
struct Generique {
    void* valeur;
    // Autres champs
};
```
On est obligé d'avoir une disjonction de cas selon le type de `valeur` car on ne traite justement pas de la même manière un `int*`, un `FILE*` ou même un `int (*tab)[50]` (pointeur vers un tableau 2D d'entier) .

Dans notre exemple, nous avions imaginé ce besoin d'avoir plusieurs versions, ici ce n'est pas de l'imagination mais une obligation, la méthode des pointeurs de fonction est donc aussi préconisé.

{{< admonition type=note title="Remarque" open=true >}}
Pour finir, j'ai parlé précédemment de la programmation générique, la programmation générique avec une [implémentation réifiée](https://medium.com/@darrenwedgwood/java-generics-explained-c18a929fcd68#:~:text=Reified%20Generics%20is%20a%20generics,to%20Java%27s%20Type%2DErased%20Generics.) comme en C++ ou en C# écrit pour nous ce code qui varie mais pour chaque type utilisé.
Nous nous contentons d'écrire le comportement de chaque type à l'aide de l'héritage / l'implémentation d'interface (comme en C# ou traits en Rust) pour définir des fonctions au nom commun mais au comportement différent pour chaque type [(cf. surcharge d'opérateur)](https://en.cppreference.com/w/cpp/language/operators).

Cette approche est dite "type safe" à partir du moment où l'on peut **contraindre** ces génériques mais on en parlera dans un autre article.
{{< /admonition >}}

{{< admonition type=abstract title="Résumé" open=true >}}
1. Une variable est une donnée, une fonction/procédure est un traitement conditionnel sur une/plusieurs donnée(s).
2. Il faut toujours chercher à factoriser le plus possible son code pour des raisons d'ergonomie, de maintenance mais aussi d'efficacité (un code moins long est plus facile à auditer et à stocker).
3. Si on veut laisser le code client implémenter ses propres traitements, l'utilisation de pointeurs de fonction est privilégiée
4. Dans une structure générique, nous n'avons pas vraiment le choix d'utiliser des pointeurs sur fonction pour effectuer une disjonction de cas selon les types que peut prendre le champ générique.
{{< /admonition >}}