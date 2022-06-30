---
layout: default
title: Conclusions
nav_order: 5
---

# Conclusions
{: .fs-9 }

-	S’ha realitzat un estudi dels 3 datasets “transactions”, “article” i “customers” on s’han entès i trobat relació i paràmetres representatius.
-	Es presenta un model de recomanació de productes de H&M on donat un conjunt d'observacions usuari-article, es proporciona un ranking adequat per cada usuari. Per fer-ho s'ha utilitzat la llibreria de TensorFlow Recommenders.
-	El sistema de recomanació consta en dues fases. Retrieval Stage: en aquesta etapa es crea un model amb el qual s’obté un conjunt inicial de centenars de candidats, d’entre tots els candidats possibles. Ranking Stage: en aquesta etapa s’analitzen les sortides del Retrieval Model i s’afinen per seleccionar el millor conjunt de recomanacions.
-	En el Retrieval Stage és defineix un model de dues torres o dos submodels (Query Model i Candidate Model). Per implementar-ho, s’han utilitzat els mòduls de TensorFlow següents: tfrs.tasks.Retrieval i tfrs.metrics.FactorizedTopK.
-	En el Ranking Stage, per recuperar els millors candidats d’una consulta determinada, utilitzarem la llibreria de TensorFlow ScaNN. En concret, per cada usuari, recuperarem 12 etiquetes, que es corresponen als articles previstos que un client pot comprar en els següents 7 dies.
-	Per definir aquest model, s’han tocat els següents hiperparàmetres:
1.	embedding_dimension: 32 i 64
2.	Learning rate: 0.01, 0.05 i 0.1
3.	Algoritmes de optimització: Adam, Adamax i Adagrad
4.	Epoch: 5, 10, 15 i 20
-	Per concloure, tot i les proves realitzades, la eficiència del model es podria millorar. 

## Alternatives
Com hem comentat abans podríem millorar el model presentat. Aquí proposem diferents alternatives per fer-ho:
-	Es pot provar de veure con afecta l'eficiència tocant més els hiperparàmetres esmentats anteriorment.
-	També es podria fer proves canviant o afegint capes de Keras al model. Com per exemple, Long short-term memory (LSTM) o Dense.
- Considerem que hi ha certes variables que poden influir positivament en els resultats. Entre elles destaquem:
  - **Edat dels consumidors**: donar un pes important al tipus de producte que es compra segons l'edat dels consumidors (en particular, per rangs d'edat).
  - **Popularitat dels productes**: donar un pes més elevat a aquells productes (o tipus de productes) que es venen amb més freqüència al llarg del temps.
  - **Articles comprats junts**: estudiar quins articles acostumen a comprar-se de manera conjunta i aplicar-ho al model pot millorar els resultats.
  - **Data de compra dels productes**: tenir en compte si els productes es continuen venent o ja s'han deixat de produir.



## Experiència personal

En aquest apartat volem comentar alguns dels inconvenients que ens hem trobat al llarg de la realització del treball: 

En primer lloc, la gran quantitat de dades. Aquest fet fa que el procés d’execució de qualsevol model sigui molt lent, i per tant, no facilita treballar d’una manera àgil. Per tant, dificulta el fet de poder fer diferents proves, cosa que implica que el resultat del model sigui molt millorable.

Per altra banda, hem trobat grans dificultats per afegir la metadata al model, per tal de millorar els resultats.
