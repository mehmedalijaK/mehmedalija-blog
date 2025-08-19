---
title: "Statička vs. Dinamička Analiza: Dva Ključna Pristupa Kvalitetnom Kodu"
date: 2025-08-17T22:00:00+02:00
draft: false
translationKey: "static-dynamic-analysis-post"
tags: ["programiranje", "kotlin", "analiza-koda", "softverski-inzenjering"]
categories: ["Kompajleri"]
description: "Razlike između statičke i dinamičke analize koda, dva stuba na kojima počiva razvoj pouzdanog softvera."
---

Dobro došli na moj blog! Kao prvu objavu, želim da podelim jedan od centralnih koncepata iz mog diplomskog rada: razliku između statičke i dinamičke analize koda.

### Statička Analiza: Inspekcija Koda bez izvršavanja

Statička analiza podrazumeva ispitivanje koda **bez njegovog izvršavanja**. Ovo je trenutak kao da posmatramo samo arhitektonski plan jedan zgrade, bez početka izgradnje iste.

Pri čemu se koriste strukture poput apstraktnog-sintaksnog stabla (AST), kako bi se otkrile greške i nepravilnosti u:
*   Sintaksi
*   Strukturi koda
*   Programerskom stilu

Ovaj pristup nam omogućava da uhvatimo čitave klase problema pre nego što kod uopšte bude pokrenut.

### Dinamička Analiza: Testiranje Koda "u akciji"

S druge strane, dinamička analiza podrazumeva posmatranje ponašanja programa tokom stvarnog izvršavanja. Vraćajući se na analogiju od pre, ovo bi bilo kao testiranje izgrađene kuće pod realnim okolnostima – puštanje vode, paljenje svih uređaja i provera da li sve funkcioniše kako treba.

U ovu kategoriju spadaju:
*   Unit i integracioni testovi
*   Alati za praćenje performansi (profileri)
*   Analiza pokrivenosti testovima (*coverage*), koja identifikuje delove koda koji nisu testirani
*   Detekcija *runtime* izuzetaka

### Najbolje od oba sveta: Zašto ih koristiti zajedno?

U savremenoj praksi, najefikasniji pristupi kombinuju statičku i dinamičku analizu — statička omogućava rano otkrivanje problema u strukturi i logici koda, dok dinamička pruža uvid u konkretno ponašanje programa tokom izvršavanja. Prilikom razvoja mobilnih aplikacija u Kotlinu, statička analiza pomaže u detekciji potencijalnih grešaka, dok se dinamička vrši kroz debagovanje, testiranje i profilisanje performansi. Kombinovanjem ova dva pristupa dobija se stabilan, pouzdan i kvalitetan softver