---
theme: default
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://cover.sli.dev
# some information about your slides (markdown enabled)
title: Welcome to duck nodejs event loop
titleTemplate: '%s - Duck Event Loop'
info: |
  ## Duck Nodejs Event loop
  Presentation slides for developers.
author: stephen deletang
keywords: nodejs eventLoop
presenter: true
browserExporter: build
download: true,
exportFilename: ducker-event-loop-exported
export:
  format: pdf
  timeout: 30000
  dark: false
  withClicks: false
  withToc: false
twoslash: true
selectable: true
record: dev
contextMenu: true
wakelock: true
overviewSnapshots: false
colorSchema: auto
routerMode: history
aspectRation: 16/9
canvasWidth: 980
themeConfig:
  primary: '#5d8392'
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
---

# Duck Nodejs Event loop

Presentation slides for developers

<div @click="$slidev.nav.next" class="mt-12 py-1" hover:bg="white op-10">
  Press Space for next page <carbon:arrow-right />
</div>

<div class="abs-br m-6 text-xl">
  <button @click="$slidev.nav.openInEditor()" title="Open in Editor" class="slidev-icon-btn">
    <carbon:edit />
  </button>
  <a href="https://github.com/stephen-shopopop/duck-nodejs-event-loop" target="_blank" class="slidev-icon-btn">
    <carbon:logo-github />
  </a>
</div>

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---
layout: center
---

## Reactor pattern

Quand il est question d’events-loop on parle souvent de [Reactor pattern](https://en.wikipedia.org/wiki/Reactor_pattern) depuis 1996 (c'est le principe qui définit les fondamentaux et qui pourra notamment vous permettre de comprendre d’autres pattern de concurrence comme [Proactor](https://en.wikipedia.org/wiki/Proactor_pattern)).

Dans le cadre d’une Event-loop/Reactor on parlera souvent aussi d'algorithme [Round-robin](https://en.wikipedia.org/wiki/Round-robin_scheduling) et Demultiplexing d'évènements.

---
layout: center
---

## Reactor event loop

Le réacteur prend en input un évènement (lire un fichier, envoyer un paquet sur le réseau) qui aura un cycle de vie prédéfini au sein de la loop en fonction de sa nature (et de l’implémentation). Les I/O bloquant seront, la plupart du temps, gérés au sein d’abstractions bas niveau fournies par le système comme epoll, kqueue et event ports (tout dépend du système d’exploitation cible). Quand il n’est pas possible d’utiliser les ressources du système, des threads seront bien souvent créés.

Une fois le traitement terminé le réacteur s’occupera de déclencher le callback lié à l’évènement pour signaler que le traitement est terminé (avec succès ou en erreur). Je parle ici de callback pour rester bas niveau, mais il peut s’agir d’une Promise/Future ou de toute autre structure ayant pour objectif de gérer la résolution d’un événement Asynchrone.

Lien bonus pour les motivés: [EN Reactor - An Object Behavioral Pattern for Demultiplexing and Dispatching Handles for Synchronous Events](https://www.dre.vanderbilt.edu/~schmidt/PDF/reactor-siemens.pdf)

---
layout: center
---

## Histoire

La notion d'[event](https://en.wikipedia.org/wiki/Event_(computing)), event-driven et [event-loop](https://en.wikipedia.org/wiki/Event_loop) ne date pas d’hier et les premières apparitions date des années 80 (même si le pattern est devenu fort populaire depuis une dizaine d’années grâce à l’apparition de lib comme Libuv ou plus récemment Tokio sur Rust).

Il existe très probablement des librairies équivalentes ou des implémentations très sérieuses sur les différents runtime (Python, Ruby, PHP, Lua, Perl etc). Le langage de programmation [Julia](https://julialang.org) est d’ailleurs basé sur Libuv.

Aujourd’hui il devient très clair que le pattern a fait ses preuves et qu’il est très largement apprécié par les développeurs du monde entier pour construire des programmes concurrents (même s’il faut toujours garder en tête qu’il y aura toujours des points forts ainsi que des points faibles).

---
layout: center
---

## Libuv

Libuv est donc la librairie qui est utilisée dans Node.js pour l’event-loop. Son fonctionnement ne vous impacte pas directement dans votre code (elle est transparente pour les développeurs… c’est l’objectif de Node.js ^^).

Il est important de comprendre comment elle fonctionne a minima car l’exécution des différentes phases va définir comment votre code fonctionnera et dans quel ordre il sera exécuté (ce qui vous permettra de résoudre le challenge de l’introduction).

Le schéma ci-dessous est un schéma que j’ai construit pour représenter les différentes phases de l’event-loop (vous noterez la claire séparation entre votre code, la loop et le système d’exploitation).

Sur le sujet je vous recommande d’aller lire en premier lieu les pages suivantes :

- EN [Débutant] [The Node.js Event Loop, Timers, and process.nextTick()](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)
- EN [Débutant] [Libuv design overview (documentation officielle de Libuv)](https://docs.libuv.org/en/v1.x/design.html).
- EN [Débutant] [An introduction to libuv](https://nikhilm.github.io/uvbook/An%20Introduction%20to%20libuv.pdf).

---
layout: two-cols
---

## Petit challenge, pouvez-vous deviner l’ordre des logs ?

**resultat:**

```shell
E
A 0
A 1
A 2
A 3
A 4
A 5
A 6
A 7
A 8
A 9
J
F
H
G
I
D
B
C
```

:: right ::

```ts
async function a(val) {
    console.log("A", val);
}
setImmediate(() => console.log("B"));

new Promise((res) => {
    for (let id = 0; id < 1e9; id++) {}
    setImmediate(() => console.log("C"));
    process.nextTick(() => res("D"));
    console.log("E");
}).then(console.log);

queueMicrotask(() => console.log("F"));

(async(res) => {
    for (let id = 0; id < 1e6; id++) {}
    process.nextTick(() => console.log("G"));
    return "H";
})().then(console.log);

process.nextTick(() => console.log("I"));
const promises = [];
let n = 0;
for (; n < 10; n++) promises.push(a(n));

console.log("J");
Promise.all(promises);
```

---
layout: center
---

## L'event loop

L' [event loop](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick) comprend les phases suivantes :

- Minuteries
- Rappels en attente
- Au ralenti/préparer
- Sondage
- Vérifier
- Fermer les rappels,
- Connexions et données entrantes

La phase la plus importante est la première: les minuteurs. Les minuteurs sont des fonctions de rappel enregistrées avec "setTimeout()" ou "setInterval()".

Ils permettent également de surveiller la boucle d'événements avec la possibilité de planifier les données, ce qui constitue un excellent moyen de vérifier si un événement est inactif. La boucle d'événements exécute ensuite les temporisateurs expirés et vérifie à nouveau les rappels en attente.

---

Les rappels d'E/S sont vérifiés en premier lors de la phase d'interrogation, suivis des rappels "setImmediate()".

Node.js dispose également d'un rappel spécifique, process.nextTick(), qui s'exécute après chaque boucle. Ce rappel a la priorité la plus élevée.

Pendant la phase d'interrogation, la boucle d'événements recherche les événements qui ont terminé leurs tâches asynchrones et sont prêts à être traités.

Nous passons ensuite à la phase de vérification, au cours de laquelle la boucle d'événements exécute tous les rappels enregistrés avec "setImmediate()".

Les rappels de fermeture sont associés à la fermeture des connexions réseau ou à la gestion des erreurs lors des événements d'E/S. La boucle d'événements recherche ensuite les temporisateurs programmés.

La boucle continue ensuite, gardant l’application réactive et non bloquante.

---
layout: center
---

# Flux normal d'une requête HTTP

Lorsqu'une requête arrive dans Node.js, elle est traitée de manière synchrone, et la réponse suit ensuite un processus similaire. En revanche, lorsqu'une requête doit appeler la base de données, elle s'exécute de manière asynchrone.

Cela signifie que pour chaque requête, il existe deux processus synchrones et un processus asynchrone. Généralement, le temps de réponse peut être calculé à partir de la formule suivante :

> Temps de réponse = 2SP + 1AP

Où SP est le traitement synchrone et AP est le traitement asynchrone.

L'attente d'E/S n'est pas prise en compte car la boucle d'événements s'exécute de manière synchrone.

---

Si, par exemple, un serveur reçoit trois requêtes à la fois, combien de temps faudra-t-il pour traiter la dernière requête ?

La première requête est traitée, tandis que les deuxième et troisième requêtes sont mises en file d'attente. Ces dernières sont ensuite traitées dans l'ordre d'arrivée, en attendant la fin du traitement de la requête précédente.

Le temps de traitement de chaque requête selon la formule standard sera respectivement de 30 ms, 50 ms et 70 ms, la boucle d'événements étant exécutée de manière synchrone.
Pour calculer le temps de réponse de la dernière requête, quel que soit le nombre de requêtes, vous pouvez appliquer la formule suivante :

> Temps de réponse = SPx\ 2 + ASx + (SPx-1*2)*

Où x est le numéro de la demande.

---

Une solution possible pour réduire ce temps d'exécution est de faire évoluer vos serveurs en fonction de l'utilisation du CPU : cependant, la création de nouveaux serveurs prend du temps et entraîne souvent une sous-utilisation des ressources car il peut y avoir de la capacité disponible dans votre système malgré une utilisation à 100 %.

La raison est simple : Node.js s'exécute sur plusieurs threads, avec un ramasse-miettes et un optimiseur de processeur fonctionnant séparément. Cela signifie qu'au sein de Node.js, une grande quantité de CPU peut être disponible avant que le système ne commence à ralentir significativement.

---

Une autre solution utilise ce package [node-metrics](https://www.npmjs.com/package/@stephen-shopopop/node-metrics).

Lorsque le package reçoit plusieurs requêtes après un certain temps, l'utilisation des événements dépasse la limite au bout de 0,98 s. Passé ce délai, toute requête entrante reçoit un code d'état de réponse de 503.

Imaginez que vous ayez plusieurs requêtes : la boucle d'événements pourrait avoir accumulé plus de deux secondes de retard. Un utilisateur pourrait être gêné par une telle attente. Dans ce cas, vous pouvez renvoyer une réponse pour l'informer que le serveur ne répondra pas à la requête.

---
layout: center
---

## Surveiller l'utilisation de la boucle d'événements

Vous pouvez calculer l'ELU à l'aide de la [bibliothèque perf_hooks](https://nodejs.org/api/perf_hooks.html) . Cette fonction renvoie une valeur décimale comprise entre 0 et 1, indiquant la quantité de boucle d'événements utilisée.

Sinon vous pouver utiliser le plugin [node-metrics](https://www.npmjs.com/package/@stephen-shopopop/node-metrics) et vous trouvez sur [repo](https://github.com/stephen-shopopop/node-metrics/tree/main/example) de nombreux examples.

---
layout: end
---
