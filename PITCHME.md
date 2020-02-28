# Tecniche di collaborazione tra thread

---

@snap[north-west]
### Produttore/Consumatore
Come facciamo a far si che due thread collaborino? <br>Ad esempio un thread, *Thread A*, produce qualcosa di cui il *Thread B* ha bisogno.<br><br> 
Il *Thread A* è il produttore e il *Thread B* è il consumatore. <br><br> I due thread si devono coordinare: il consumatore deve aspettare che il produttore 
abbia prodotto, prima di poter prelevare l'elemento.
@snapend
---
@snap[west]
Il *Thread B*, consumatore e *Thread A*, produttore, eseguono questo codice, dove *obj* è qualche oggetto:

```java
// Thread B
if ( resultIsAvailable() == false )
    obj.wait(); // wait for notification that the result is available
useTheResult();
```

```
// Thread A
generateTheResult();
obj.notify(); // send out a notification that the result is available
```

Ora c'è qualche subdola race condition in questo codice. 
@snapend

---

@snap[north-west]
```java
// Thread B: consumatore
if ( resultIsAvailable() == false )
    obj.wait(); // wait for notification that the result is available
useTheResult();
```

```
// Thread A: produttore
generateTheResult();
obj.notify(); // send out a notification that the result is available
```
@snapend

@snap[south-west text-07]
I due thread potrebbero eseguire il loro codice nel seguente ordine:
@ol[]
1. Il *Thread A* controlla **resultIsAvailable()** è trova che il risultato non è disponibile, così decide di invocare **obj.wait()**, ma prima che lo faccia, @note[nota 1]
1. Il *Thread B* termina di generare il risultato e invoca **obj.notify()** @note[nota 2]
1. Il *Thread A* chiama **obj.wait()** per aspettare la notifica che il risultato è disponibile. @note[note 3]
@olend

Nello Step 3, il *Thread A* sta aspettando per una notifica che non arriverà mai, perché **notify()** è già stata invocata nello Step 2. 
Questo è un tipo di deadlock che lascia il *Thread A* aspettare per sempre.

@snapend

---
@snap[north-west text-07]
Quindi ci vuole qualche tipo di sincronizzazione. La soluzione è includere sia il codice di *Thread A* che di *Thread B* nello statement **synchronized**, ed è molto naturale sincronizzare sullo stesso oggetto, obj, che è utilizzato per le chiamate di **wait()** e **notify()**. 
<br><br>In effetti, siccome la sincronizzazione sarebbe quasi sempre obbligatoria quando  si utilizzano wait() e notify(), **Java lo rende un requisito obbligatorio**. 
<br><br>
In Java, un thread può legalmente invocare **obj.wait()** e **obj.notify()** solo se il thread ha acquisito il lock di sincronizzazione associato all'oggetto **obj**. 
<br><br>Se non ha acquisito prima il lock, allora viene lanciata un'eccezione. (L'eccezione è di tipo *java.lang.IllegalMonitorStateException*, che è di tipo runtime 
e quindi non deve essere gestita obbligatoriamente). Una ulteriore complicazione è che il metodo wait() può lanciare l'eccezione *java.lang.InterruptedException* 
e quindi deve essere invocata nello statement di  try che cattura quella eccezione.
@snapend
---

@snap[north-west  text-09]
### Produttore/consumatore semplificato (un solo elemento)
Facciamo ad esempio il caso di un risultato calcolato da un thread che deve essere reso disponibile ad un altro thread. 
Assumendo che c'è una variabile condivisa **sharedResult** che è utilizzata per trasferire il risultato dal produttore al consumatore. 
Quando il risultato è pronto, il produttore setta il valore della variabile a un valore non nullo. 
Il consumatore può controllare se il risultato è pronto controllando se il valore della variabile **sharedResult** è nullo. 
Utilizziamo la variabile **lock** per sincronizzare.
@snapend

---

@snap[north-west]
Codice produttore:
```java
makeResult = generateTheResult(); // Not synchronized!
synchronized(lock) {
    sharedResult = makeResult;
    lock.notify();
}
```
Codice consumatore:
```
synchronized(lock) {
    while ( sharedResult == null ) {
        try {
            lock.wait();
        }
        catch (InterruptedException e) {
            }
    }
    useResult = sharedResult;
}
useTheResult(useResult); // Not synchronized!
```
@snapend

---

@snap[north-west text-08]
Da notare una cosa: **lock.wait()** non finisce finché **lock.notify()** non è invocato, 
ma siccome entrambi i metodi sono chiamati all'interno di blocchi synchronized che si sincronizzano sullo stesso oggetto, 
non dovrebbe essere impossibile per entrambi i metodi essere eseguiti contemporaneamente?<br><br> 
Infatti, **lock.wait()** è un caso speciale: quando un thread chiama **lock.wait()**, esso cede il lock che teneva sull'oggetto sincronizzato.<br><br> 
Questo dà a un altro thread la possibilità di eseguire il blocco **synchronized(lock)** che contiene lo statement **lock.notify()**.  
Dopo che il secondo thread (il produttore) esce dal blocco sincronizzato, il lock è ritornato al thread consumer così che possa continuare.
@snapend

---

@snap[north-west text-08]
### Produttore/consumatore
In un problema produttore/consumatore completo, molti risultati sono prodotti da uno o più produttori e sono consumati da uno o più consumatori. 

Invece di avere un unico oggetto condiviso, *sharedResult*, abbiamo una lista di oggetti che sono stati prodotti e non ancora consumati.<br><br>

La struttura dati che implementa questa problematica di produttore e consumatore è quella di una **lista** in cui i produttori mettono gli elementi prodotti e i consumatori prendono gli elementi da consumare.

@snapend
---

@snap[north-west text-08]
### Blocking queue
Questa lista deve essere sincronizzata per evitare race condition ma in più deve far si che se la lista è vuota 
e un consumatore cerca di prelevare un elemento dalla lista, il consumatore sia messo in wait (bloccato) fino a che 
un produttore non inserisce un elemento e notifica l'inserimento: per questo motivo, questa struttura, si chiama **blocking queue**.
<br>
![width=700](assets/img/Produttore_consumatore_connection_pool.PNG)
@snapend

---?gist=MassimoCappellano/34a86252db41dc4d28d4b21424246ea7&lang=Java&title=Implementazione di una blocking queue
@[4](variabile interna di tipo LinkedList)
@[19-25](metodo take, per prendere elemento dalla coda)
@[12-17](metodo add, per aggiungere elemento alla coda)
@[6-10](metodo clean, per svuotare la coda)

---

@snap[north-west text-07]
### Il metodo notifyAll()
E' possibile che diversi thread siano in attesa di notifiche. 
Una chiamata a **obj.notify()** risveglia solo un thread che era in waiting su obj. 
Se si vuole risvegliare tutti i thread che erano in waiting su *obj*, bisogna invocare **obj.notifyAll()**. 
obj.notify() funziona nell'esempio sopra perché solo i thread consumatori potevano essere bloccati. 
Abbiamo bisogno che anche un solo thread dei consumatori sia risvegliato e non importa quale consumatore prenda l'elemento.<br><br> 
Consideriamo invece una blocking queue di **capacità massima limitata**, dove sia il consumatore che il produttore possono essere bloccati. 
Quando un elemento è aggiunto a una coda, noi vogliamo che il thread consumatore è  notificato, non invece magari un altro produttore. 
Una soluzione è chiamare la **notifyAll()** invece della notify(), che notificherà a tutti i thread, inclusi i consumatori che aspettano in waiting.
@snapend

---?gist=MassimoCappellano/0092be9c4c2f53658c1cdcb1c751e9a5&lang=Java&title=Esempio utilizzo notifyAll()
@[2-19](variabili interne condivise: message e empty)
@[11-25](metodo take, per prendere elemento dalla coda)
@[27-42](metodo put, per aggiungere elemento alla coda)
---?gist=MassimoCappellano/9a14eafac275a29c107e46b67dcc2599&lang=Java&title=Blocking queue con limite massimo di elementi 
@[6-7](variabili interne condivise: queue e limit)
@[13-21](metodo enqueue)
@[23-32](metodo dequeue)

---
@snap[north-west]
Java ha due classi che implementano la blocking queue: 
**LinkedBlockingQueue** e **ArrayBlockingQueue**.<br><br> 
Questi sono tipi parametrici per permettere di specificare i tipi di elementi che la coda può contenere.<br><br> 
Entrambe le classi sono definite nel pacchetto **java.util.concurrent** ed entrambe implementano l'interfaccia **BlockingQueue**.
@snapend

---
 
@snap[north-west]
Un **ArrayBlockingQueue** ha una capacità massima che è specificata quando è creata.<br> 
Per esempio, per creare una coda bloccante che può contenere fino a 25 elementi di tipo *ItemType*, si può dire:
```java
ArrayBlockingQueue<ItemType> bqueue = new ArrayBlockingQueue<>(25);
```
<br>
Una **LinkedBlockingQueue** crea, invece, una blocking queue con capacità illimitata.<br> 
Per esempio:
```java
LinkedBlockingQueue<ItemType> bqueue = new LinkedBlockingQueue<>();
```
@snapend

---
@snap[north-west text-09]
Se *bqueue* è una coda bloccante che appartiene a una delle due classi, allora le seguenti operazioni sono definite:
@ul[](false)
- *bqueue.take()* - Rimuove un elemento dalla coda e lo ritorna risultato. Se la coda è vuota quando questo metodo è invocato, il thread che ha invocato il metodo **rimane bloccato** finché un elemento non sarà disponibile. Questo metodo lancia una *InterruptedException* se un thread è interrotto mentre è bloccato.
- *bqueue.put(item)* - Aggiunge un elemento alla coda. Se la coda ha una capacità limitata ed è piena, il thread che ha chiamato questo metodo, **rimane bloccato** finché non si libera posto nella coda. Questo metodo lancia una *InterruptedException* se il thread è interrotto mentre era bloccato.
@ulend
@snapend

---

@snap[north-west text-09]
@ul[](false)
- *bqueue.add(item)* - Aggiunge un elemento alla coda, se lo spazio è disponibile. Se la coda ha una capacità limitata e è piena, lancia una *IllegalStateException*. Questo metodo non è bloccante.
- *bqueue.clear()* - Rimuove tutti gli elementi dalla coda e li scarta.
@ulend
@snapend

---

@snap[north-west text-08]
### Concurrent Queue
Un' altra classe che rappresenta una coda è **ConcurrentLinkedQueue<E>** nel package **java.util.concurrent**.<br>

```java
ConcurrentLinkedQueue<E> queue = new ConcurrentLinkedQueue<>();
```
Questa classe non è di tipo blocking queue: il metodo per aggiungere un elemento x alla coda è **queue.add(x)**. 
Il metodo per rimuovere un elemento dalla testa della coda è **queue.poll()**. 
La chiamata **queue.poll()** ritorna *null* se la coda è vuota; quindi **poll()** può essere utilizzato per testare se la coda è vuota 
o ritrovare l'elemento altrimenti.

@snapend

---
@snap[north-west]
#### Riepilogo
@snapend

@snap[south span-100 text-06]
[Vai a @fa[external-link]](https://checksound.gitbook.io/corsojava/i-thread-e-concorrenza)
@snapend














