---
layout: default
title: Model
nav_order: 4
---

# Model 
{: .fs-9 }

L'objectiu del model és recomanar productes de H&M als seus clients. És a dir, donat un conjunt d'observacions usuari-article, proporcionar un ranking adequat per cada usuari.
Per fer-ho, utilitzarem la llibreria de **TensorFlow Recommenders (TFRS)**, que permet crear models de sistemes de recomanació.


En general, els sistemes de recomanació consten de dues fases:

•	**Retrieval Stage**: en aquesta etapa es crea un model amb el qual s'obté un conjunt inicial de centenars de candidats, d'entre tots els candidats possibles. L'objectiu principal del model és eliminar de manera eficient tots els candidats pels quals l'usuari no estigui interessat.
Els Retrieval Models normalment consten de dos sub-models: 
-	*Query Model*: per les dades de consulta 
-	*Candidate Model*: per les dades candidates. 
Construirem i entrenarem un model de dues torres o dos submodels (torre de consultes i torre candidata) utilitzant el conjunt de dades explicat anteriorment
Per implementar-ho, utilitzarem els mòduls de TensorFlow següents: tfrs.tasks.Retrieval i tfrs.metrics.FactorizedTopK

•	**Ranking Stage**: en aquesta etapa s'analitzen les sortides del Retrieval Model i s'afinen per seleccionar el millor conjunt de recomanacions. És a dir, l'objectiu principal d'aquesta fase és reduir el conjunt d'elements que poden interessar a l'usuari.
En aquest cas, per recuperar els millors candidats d'una consulta determinada, utilitzarem la llibreria de TensorFlow ScaNN. En concret, per cada usuari, recuperarem 12 etiquetes, que es corresponen als articles previstos que un client pot comprar en els següents 7 dies.

En aquest model, utilitzarem el dataset de “transactions”. Aquest dataset el tractem i el dividim en dos conjunts de test i train.

~~•	El subconjunt d'entrenament s'utilitza per entrenar les dades que donem al sistema de recomanació, per tant, són dades “visibles” pel model. Els resultats obtinguts amb aquest subconjunt no s'han de tenir en compte, ja que no representen el rendiment real.
•	El subconjunt de test són dades no visibles pel model, amb les quals es mesura el rendiment real del sistema. L'objectiu és maximitzar el conjunt d'entrenament per poder tenir més dades i ajustar millor el model, però també es vol maximitzar el conjunt de tests per poder obtenir millors resultats en tenir més informació sobre la qual mesurar el sistema.~~

En un sistema de recomanació, una bona manera de dividir-lo és en un moment indicat T. És a dir, les dades fins al moment T s'utilitzen per predir les següents observacions. El nostre T es 2020-09-01.

~~transactions_train.head()
transactions_test.head()~~

A continuació, busquem els clients i els articles únics presents dels datatset *customers* i *articles* . És important que tant els clients com els articles siguin únics perquè necessitem mapejar els valors de les variables categòriques per després utilitzar-les en el nostre model com a vectors. És a dir, necessitem un diccionari d'aquestes dues variables per crear el nostre model. Són elements vectoritzats que s'inseriran i faran servir com a referència a les capes del model

<div class="code-example" markdown="1">
```js
// Javascript code with syntax highlighting.
  unic_customer_id = df_customer.customer_id.unique()
  unic_article_id = df_article.article_id.unique()
  article_slices = tf.data.Dataset.from_tensor_slices(dict(df_article[
  articles = article_slices.map(lambda x: x['article_id'])
```
</div>

Retrival Stage
Ara que ja hem preparat les dades, creem el model. . L'arquitectura d'aquest model és clau. Com hem mencionat anteriorment, utilitzarem un Retrieval Modelconstituit per dos sub-models. Així doncs, podem crear cada model per separat (Query Model i Candidate Model) i després combinar-los en un model final.
La idea principal és el que es representa a la figura següent:
Dibujo 1
En particular, pel nostre cas:
Dibujo 2
En primer lloc, fixem la dimensió de l'embedding:
embedding_dimension = 64

Aquesta es un híperparàmetre del nostre model, per tant, hem fet algunes proves per veure quina seria la dimensió de embedding que s’ajusta millor al nostre model. Finalment, podem concloure que es 64. 
A continuació, definim els models dos submodels, el query model y el candidate model. Per crear el query model utilitzem les capes de preprocessament de Keras, que ens permet convertir els ID dels clients en números enters. Després, mitjançant la capa d'embedding, es converteixen en embeddings d'usuaris. 
customer_model = tf.keras.Sequential([
  tf.keras.layers.StringLookup(
      vocabulary=unic_customer_id, mask_token=None),  
  tf.keras.layers.Embedding(len(unic_customer_id) + 1, embedding_dimension)
])
La creació del candidate model segueix el mateix procés que el query model, però utilitzant ara la llista d'ID d'articles únics:
article_model = tf.keras.Sequential([
  tf.keras.layers.StringLookup(
      vocabulary=unic_article_id, mask_token=None),
  tf.keras.layers.Embedding(len(unic_article_id) + 1, embedding_dimension)
])

StringLookup() es una capa de preprocessament que mapa característiques de cadena amb índexs enters.

Ara que ja tenim els dos submodels creats, els unifiquem per crear el nostre model de recomanació. 
TensorFlow Recommenders exposa una classe de model base (tfrs.models.Model) que facilita la creació dels models. El que cal fer és configurar els components en el mètode init i implementar el mètode compute_loss, agafant les característiques sense processar i tornant un valor de pèrdua.
El model base s'encarrega de crear el cicle d'entrenament apropiat per tal que s'ajusti al nostre model.
class RetrivalModel(tfrs.Model):
    
    def __init__(self, customer_model, article_model):
        super().__init__()
        self.article_model: tf.keras.Model = article_model
        self.customer_model: tf.keras.Model = customer_model
        self.task = tfrs.tasks.Retrieval(
        metrics=tfrs.metrics.FactorizedTopK(
            candidates=articles.batch(128).map(self.article_model),            
            ),
        )        

    def compute_loss(self, features: Dict[str, tf.Tensor], training=False) -> tf.Tensor:
    
        customer_embeddings = self.customer_model(features["customer_id"])    
        article_embeddings = self.article_model(features["article_id"])

        # Task calcula la pèrdua i les mètriques
        return self.task(customer_embeddings, article_embeddings,compute_metrics=not training)

FactorizedTopK() calcula les mètriques dels K candidats principals que apareixen mitjançant un model de recuperació.

Finalment, amb el model ja definit, podem ajustar i avaluar el model utilitzant les crides estàndard de Keras:
model = RetrivalModel(customer_model, article_model)
model.compile(optimizer=tf.keras.optimizers.Adagrad(learning_rate=0.1))

Podem veure que el nostre model te un learning rate (rati de aprenentatge) de 0.1. Aquest es un altre hiperparàmetre del model, el qual hem fet proves per buscar el paràmetre més òptim per aquest.
El learning rate és el percentatge de canvi amb què s'actualitzen els pesos en cada iteració, en altres paraules, cada que es realitza una iteració en el procés d'entrenament s'han d'actualitzar els pesos de l'entrada per poder donar cada cop una millor aproximació. Si posem un valor molt proper a un podria cometre errors i no obtindríem un model de predicció adequat, però si posem un valor molt petit aquest entrenament podria ser massa trigat per acostar-nos a una predicció acceptable.

Per altra banda, apliquem shuffle, batch i cache en els conjunts definits anteriorment de test i train. Podem veure que el batch es un altre híperparàmetre, el qual també hem fet proves per trobar el més adequat. Que es BACH (crear grupos...)

Ara ja podem començar amb l’entrenament. Per això hem de decidir el híperparàmetre de quantes iteració ha de tenir el model (epoch). Un epoch implica un cicle complet del conjunt de dades d'entrenament que es compon de lots i iteracions del conjunt de dades, i el nombre d'epoch necessaris perquè un model s'executi de manera eficient es basa en les dades en si i l'objectiu del model.
Les xarxes neuronals d'aprenentatge profund con la nostra s'entrenen mitjançant l'algoritme de descens de gradient estocàstic. El learning rate juntament amb el nombre d'iteracions (epoch) són els paraments que s'utilitzen a l'algorisme del gradient descendent. El descens del gradient estocàstic és un algorisme d'optimització que estima el gradient d'error per a l'estat actual del model utilitzant exemples del conjunt de dades d'entrenament i, a continuació, actualitza els pesos del model mitjançant l'algorisme de retropropagació d'errors, anomenat simplement retropropagació.

history = model.fit(
    train_ds,    
    epochs=num_epochs,
    verbose=1)

A continuació, avaluem el model. ¿no ha ido como qeremos?
model.evaluate(test_ds, return_dict=True)

Ranking Statge
En aquesta fase, com hem explicat al principi s'analitzen les sortides del Retrieval Model i s'afinen per seleccionar el millor conjunt de recomanacions. En aquest cas, per recuperar els millors candidats d'una consulta determinada, utilitzarem la llibreria de TensorFlow ScaNN. En concret, per cada usuari, recuperarem 12 etiquetes, que es corresponen als articles previstos que un client pot comprar en els següents 7 dies.
scann_index = tfrs.layers.factorized_top_k.ScaNN(model.customer_model, k = 12 )
scann_index.index_from_dataset(
  tf.data.Dataset.zip((articles.batch(100), articles.batch(100).map(model.article_model)))
)

ScanNN() troba índex de recuperació aproximat per a un model de recuperació factoritzat.
