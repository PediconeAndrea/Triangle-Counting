# Triangle-Counting

## La strategia dell'algoritmo
Otteniamo &Gamma;<sup>+</sup>(u) &forall; u &isin; U, ovvero l'insieme formato tutti i v &isin; &Gamma;(u) tali che: d(v) > d(u), tra questi si selezionano poi gli insiemi tali per cui: &thinsp;
|&Gamma;<sup>+</sup>(u)| &ge; k-1. Ottenuto questo, attraverso passaggi logici, ricaviamo G<sup>&#43;</sup>(u), ovvero il sottografo indotto da &Gamma;<sup>&#43;</sup>(u), e poi contiamo le k-1 cliques in G<sup>+</sup>(u).
***








## Il dataset  
Dove trovarlo: https://snap.stanford.edu/data/loc-Gowalla.html
Il grafo di riferimento è concepito come indiretto, e ha 196591 nodi, 950327 archi e 2273138 triangoli. Il file che scarichiamo contiene però il grafo diretto, quindi possiede 1900654 archi, e i nodi di ogni arco sono separati da uno spazio. 




## Indicazioni per l'uso


Per una maggiore leggibilità del grafo, con il seguente codice abbiamo posto come elemento separatore dei due nodi la virgola.

```
JavaRDD<String> gowalla = jsc.textFile("data/Gowalla_edges.txt");
gowalla = gowalla.map(x->new String(x.split("	")[0]+ "," + x.split("	")[1]));
gowalla.saveAsTextFile("Gowalla");
```


Per la creazione del grafo su Neo4j si richiede un file contenente la lista dei nodi del grafo. Per ottenerla, utilizziamo il seguente codice:

```
JavaRDD<String> dGrafo = jsc.textFile("data/Gowalla.txt");	
JavaRDD<String> dNodi = dGrafo.map(x -> new String(x.split(",")[0],1)).reduceByKey((x,y)->x+y).map(x -> new String(x._1 + "," + x._1));
dNodi.saveAsTextFile("GowallaNodi");
```

Dal momento che Neo4j è in grado di tralasciare la distinzione tra grafi diretti e indiretti nel momento dell'interrogazione mediante query, per ottimizzare i tempi abbiamo modificato il file scaricato dal sito in modo da rendere il grafo indiretto; per fare ciò abbiamo utilizzato la classe Wrapper "Arco".
```
JavaRDD<String> dGrafo = jsc.textFile("data/Gowalla.txt");	
JavaRDD<Arco> dArco = dGrafo.map(x -> new Arco(x.split(",")[0],x.split(",")[1]));
List<Arco> A = dArco.collect();
List<String> Grafo = new ArrayList<String>();
 for (Arco a : A) {
  if (Integer.parseInt(a.getIdEntrata()) < Integer.parseInt(a.getIdUscita())) {
  Grafo.add(a.getIdEntrata() + "," + a.getIdUscita());
 }
JavaRDD<String> dGrafo1 = jsc.parallelize(Grafo);
dGrafo1.saveAsTextFile("GowallaArchi");
```
Una volta ottenuti i file GowallaNodi.txt e GowallaArchi.txt, li abbiamo trasformati in file .csv e utilizzando la libreria **apoc** di Neo4j li abbiamo caricati sul software.




##Due strade alternative 


Abbiamo scelto di seguire due strategie che forniscono in maniera diversa l'input per l'algoritmo Clique Counting. La prima genera l'input direttamente con Spark, mentre nella seconda è Neo4j che svolge la parte iniziale di preparazione per l'input dell'applicazione. Successivamente, una volta terminata l'applicazione Spark, Neo4j è stato usato nuovamente come strumento per validare i risultati ottenuti.


1. Preparazione dell'input con Spark
Dopo aver caricato il file Gowalla.txt sull'applicazione ContaTriangoli.java, inizia la parte di codice che ha lo scopo di produrre in output una lista in cui in ogni riga si trova il generico elemento {((u,v); d(u), d(v)}. 
Per fare ciò, abbiamo eseguito i seguenti passaggi: 
**CALCOLO DI (NODO; GRADO)**: al grafo diretto abbiamo applicato una funzione lambda che restituisce un oggetto in cui in chiave si trova il nodo in entrata, e in valore l'intero "1"; successivamente, mediante una reduceByKey, abbiamo ottenuto una lista in cui vi è in chiave il nodo, e in valore il suo grado; per semplicità, abbiamo poi convertito quest'ultimo in una stringa. (u; d(u))
**CALCOLO DI (ARCO; GRADI DEI DUE NODI)**: Abbiamo poi creato due liste differenti in cui entrambe hanno come chiave l'arco, e come valore rispettivamente il grado del nodo in entrata e il grado del nodo in uscita. Con un join, intersecando per chiave le due liste, abbiamo ottenuto una lista in cui in chiave si trova l'arco, e in valore i gradi dei relativi nodi. Sempre per comodità, abbiamo poi convertito questo oggetto in una lista di stringhe. 




2. Preparazione dell'input con Neo4j
**CALCOLO DI (NODO; GRADO)**: dopo aver creato il grafo su Neo4j, abbiamo eseguito una query per assegnare come attributo ad ogni nodo del grafo il relativo grado. Questa operazione  è stata ottimizzata utilizzando il comando node.degree() della libreria apoc. 
**CALCOLO DI (ARCO; GRADI DEI DUE NODI)**: per esportare la lista in cui il generico elemento è del tipo {((u,v); d(u), d(v)}, abbiamo dato come argomento del comando export.csv.query() della libreria apoc la query che permette di ottenere questo oggetto. Il file ArcoGradi.csv risultante sarà l'input del codice contenuto nel file ContaTriangoli_NeoSpark.java.
(per permetterne il caricamento su GitHub è stato suddiviso nei due file ArcoGradi_pt1.txt e ArcoGradi_pt2.txt).


##Implementazione dell'algoritmo Map-Reduce
L'algoritmo, sia da un punto di vista operativo, sia da un punto di vista concettuale, è diviso in tre round. 

