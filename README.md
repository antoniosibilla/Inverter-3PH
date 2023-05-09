 
POLITECNICO DI BARI
DIPARTIMENTO DI INGEGNERIA ELETTRICA E DELL’INFORMAZIONE – BARI
CORSO DI LAUREA IN ING. AUTOMAZIONE (ROBOTICS)

PROJECT IN
EMBEDDED CONTROL:

PROPELLER ARM




Prof. Ing. Luca De Cicco

Studente:
Antonio Sibilla



ANNO ACCADEMICO 2020/2021
Abstract 
Lo scopo di questo Progetto è quello di stabilizzare l’angolo di beccheggio di un asta vincolata ad un punto(pivot) la quale può ruotare attorno a quest’ultimo. All’estremità della stessa è collegata un motore cc pilotato attraverso un doppio ponte h con a bordo un L293D.








 
Index
Sommario
Index	3
1.	Fisica del Sistema	4
2.	Controllo di velocità su un motore cc	4
3.	Controllo di velocità su un motore cc	5
4.	Hardware utilizzato	6
5.	Lettura angolo di Tilt	6
6.	Diagramma a blocchi	8
7.	Macro codice	10
8.	Interfaccia GUI	13
9.	Ricezione con DMA	15
10.	Meccanica del sistema	16
11.	Progetti futuri	17
12.	Tentativo di realizzazione ESC	18
13.	Teoria Inverter	20
14.	Configurazione Timer	24
15.	Problema Overshoot	25
16.	Feedback di posizione	26

 
	Fisica del Sistema
Il Sistema si compone di un asta vincolata in un punto P, libera di ruotare attorno a quest’ultimo ed un motore all’estremità.
L’attuatore in questione è proprio il motore in cc in grado di generare una portanza e quindi di spingere il tutto verso l’alto. Maggiore è la portanza che genera il motore, Maggiore sarà l’angolo a cui il Sistema giunge.
 
Figure 1 - Stato dell'arte
Ipotizziamo che si voglia portare il Sistema con un angolo pari a 0(parallelo all’orizzonte) in questo caso la forza che il motore deve sviluppare è ≃f_g. 
NB. Non si va a considerare il peso della struttura in quanto peso_braccio≪peso_motore . 

	Controllo di velocità su un motore cc 
Il metodo più semplice per il controllo di velocità di un motore cc è la variazione di tensione con cui alimentare il motore. Avendo a disposizione una tensione continua V_dcfornita da un batteria la velocità del motore sarebbe pressoché costante, mentre per variarla è necessario utilizzare un segnale PWM.
La  PWM  è  una  tecnica  che  consente  di  generare  segnali  di  comando  ai  terminali  di  controllo del transistor.  Con  questa  tecnica  è  possibile  modificare  la  larghezza/durata  di  tali  impulsi. Così facendo è possibile variare la velocità del motore avendo a disposizione una tensione fissa, ma tramite la PWM ed il teorema del valor medio la tensione applicata al motore risulterà inferiore a V_dc.

	Controllo di velocità su un motore cc

 Attraverso Eagle è stato creato uno schematico che contiene 3 ponti H, 1 a componenti discreti e un doppio ponte H attraverso l’integrato L293D. 
Attraverso questo integrato è possibile controllare fino a 2 motori in tutte le direzioni di marcia. Nel caso in questione questo chopper è stato sufficiente in quanto le potenze in gioco sono abbondantemente al di sotto della capacità dell’circuito. 


 











 
	Hardware utilizzato
	Sensore: LSM-6DS3 (ST-MICROELECTRONICS). 
Il sensore in questione è un IMU di tipo MEMS. È equipaggiato con un giroscopio a 3 assi ed un accellerometro a 3 assi. Attraverso questi due sensori è possibile misurare l’angolo di tilt del sistema e quindi effettuare una retroazione nel sistema per andare a calcolare la tensione da fornire al motore per il controllo a catena chiusa.
	Board: Nucleo-64 F446RE 
La board in questione si presta particolarmente bene per progetti embedded visto il suo rapporto prestazioni/costo elevato rispetto ad altri competitor.

	Lettura angolo di Tilt 
	Accellerometro: Attraverso l’accellerometro è possibile conoscere le forze rispetto i 3 assi (X Y Z) in un istante t_k e quindi ricavarsi attraverso delle semplici relazioni matematica l’inclinazione del sistema. 
L’accellerometro in quanto tale è soggetto alla forza di gravità verso il basso, ma è coinvolto anche in accellerazioni dovute alla non perfezione dei giunti e della propagazione delle vibrazioni dal motore verso il sensore, le vibrazioni sono tali da affogare la misura nel rumore di misura.
Per loro natura gli accellerometri sono dei sensori la cui uscita è molto rumorosa e quindi inutilizzabile se presa singolarmente.
	Giroscopio: Attraverso il giroscopio è possibile misurare la velocità a cui ruota il sensore ed integrarlo nel tempo, eseguendo la seguente formula θ=θ+ω_i*dt con i=x,y,z.
In questo caso però la formula reale diventa θ=θ+(ω_i+b)*dt dove b è il bias che ogni giroscopio possiede. Questo bias è sempre diverso da 0 quindi integrando un quantità maggiore di 0 nel tempo, la stima dell’angolo tende a divergere.
 
Figure 4 - Drift Gyro
In quest’immagine si vede come l’angolo ottenuto tramite l’accellerometro(BLU) è molto rumoroso e le vibrazioni che si vengono a creare sul braccio amplificano questo problema rendendo la misure poco utile.
Tramite il solo giroscopio(Verde) la stima dell’angolo diverge dopo circa 8 secondi, dovuto proprio al bias precedentemente citato; in pratica i 2 sensori presi singolarmente non sono di nessun aiuto.
	Sensor fusion: Un filtro complementare è un modo semplice per combinare i sensori, in quanto è la funzione lineare di un filtro giroscopio passa alto e un filtro accelerometro passa basso. 
 
Figure 5 - Schema complementary filter

Dati dell'accelerometro rumorosi con le alte frequenze vengono quindi filtrate a breve termine e attenuate da una lettura del giroscopio più fluida. Nel codice in questione è stato utilizzato un filtro con le seguenti caratteristiche:
Angolo=0.98 *(Angolo+gyro Data*dt)+0.02*(acc Data)
 
	Diagramma a blocchi






Dalla teoria dei controlli è ovvio che se si vuole un errore a regime nullo è necessario utilizzare un regolatore di tipo PI(almeno). In questo progetto è stato utilizzata una libreria PID proprio per annullare un offset a regime. Utilizzando un regolatore di tipo PI è necessario effettuare una prima fase di calibrazione dei parametri Kp e Ki, avvalendosi ad esempio del 2° metodo di Zigler-Nichols. 
Regolazione dei parametri:
Per tale regolazione è stato utilizzato il 2° metodo di Zigler-Nichols cioè è stato aumentato il valore di Kp fino al valore di Ku cioè il valore di Kp per il quale il sistema diviene persistentemente oscillante. Il parametro  Ku  limite ritrovato è stato 7 ed il tempo di oscillazione è di 1 secondo, dalla tabella di Zigler-Nichols si ricavano i seguenti risultati
 
Figure 6 - Zigler Nichols' table
È stato utilizzato il metodo “NO OVERSHOOT” ottenendo quindi i seguenti valori Kp=1.4, Ki=2.8, Kd=0.5.
N.B. Questi valori utilizzati in un primo momento non sono risultati soddisfacenti e quindi sono stati variati in maniera try and error dallo sviluppatore. Dei valori che sono stati considerati accettabili(a limite della stabilità, e che consentissero un movimento più fluido ed un control effort minore) sono: Kp=2.5, Ki=16.5, Kd=0.1 .  La costante integrale rende il sistema più lento nella risposta ma consente un Δu più contenuto.
Questi valori ottenuti , si riferiscono ad un alimentazione di 7.4V . 
	Macro codice
Il codice si divide in poche funzioni richiamate all’interno del main:
	Init_LSM: in questa funzione si controlla che il sensore sia correttamente collegato ed attivo alla board, tramite una lettura nel registro WHO_AM_I deve restituire il valore 0x
Attraverso questa funzione si vanno a scrivere dei bit all’interno della tabella dei registri, proprio per configurare i sensori, ad esempio  il fondoscala, oppure il rate di lettura.
 
Figure 7
Dovendo utilizzare una larghezza di banda(per l’accellerometro) di 416 Hz è necessario settare i bit ODR_XL3[3:0] pari a 0110. I Bit relativi al fondo scala sono i successivi 2 e per utilizzare il fondoscala minimo (±2g) è necessario settare FS_SEL[1:0] entrambi a 0.
Nel caso in questione è stato necessario scrivere nel registro CTRL1_XL il valore: 001100000 in binario, che in esadecimale diviene 0x60; Questo viene scritto tramite il comando di alto livello della libreria  HAL.
Data=0x60; // fisso Ouput Data Rate(0110), fsr (00),BW (00)
HAL_I2C_Mem_Write(&hi2c1, LSM_ADDR, CTRL1_XL, 1, &Data, 1, 1000);

Per configurare il giroscopio il procedimento è analogo. 
Read_Value: in questa funzione si leggono i valori dai registri OUTx_x_x. Gli output delle misure sono inserite nella tabella dei registri in modo contiguo quindi è possibile leggere tutti i dati insieme e poi dividerli all’interno di un array. 
Figure 8
  
Come si può notare dalla mappa dei registri, l’ordine con cui vengono scritti i registri è “inverso” rispetto a quello comprensibile dagli esseri umani, infatti la parte meno significativa precede quella più significativa. Per rendere comprensibili questi numeri è necessario effettuare lo swap di 8 bit in modo da ordinare correttamente i numeri letti.  
if(HAL_I2C_Mem_Read(&hi2c1,LSM_ADDR,OUTX_L_G,1,Rec_Data1,12,HAL_MAX_DELAY) == HAL_OK)  
{  
Gyro_X_RAW = (int16_t) (Rec_Data1[0] | Rec_Data1[1] << 8);  
Gyro_X = Gyro_X_RAW*4.375*(2 >> 1)/1000;  
}  
Si nota come il valore di Gyro_X  è ottenuto tramite la concatenazione di due byte, dove il secondo byte viene shiftato in avanti in modo da ottenere un numero comprensibile.  
  
Esempio	Lettura:  0x80|0x40;  
Vero    :   0x40|0x60;   
  
PID_Compute: al fine di calcolare la variabile di attuazione u(k) è stata utilizzata una libreria PID per ARM Cortex-M(STM32) disponibile su GitHub la quale permette di inizializzare un controllore pid e fissare i valori dei parametri Kp, Ki, Kd, min(u(k)), max(u(k)), Ts, etc… . 
 
	Interfaccia GUI
Per il controllo del propeller arm è stato utilizzato un ambiente di sviluppo della NI(National Instruents) il quale utilizza un linguaggio di programmazione grafico, chiamato linguaggio G.
Lo sviluppo del software su LabVIEW è di semplice concezione, infatti si sfrutta la comunicazione VISA (Virtual Instrument software architecture) attraverso la quale è possibile scrivere e leggere dei messaggi sulla uart della board nucleo attraverso la connessione USB.  
 
Figure 9 - Interfaccia grafica Labview
Attraverso il pannello di controllo è possibile scegliere l’angolo a cui stabilizzare il braccio, i parametri relativi al regolatore PID, cioè Kp,Ki e Kd. 

 
Figure 10 - Schema a blocchi Labview 
In questo schema a blocchi si riconoscono dei blocchi con su scritto “Visa” tali blocchi sono i responsabili della counicazione seriale tra il pc e la board Nucleo-64. 
Partendo dalla sinistra ritroviamo per primo il blocco “Visa configure serial Port” il quale è quello che inizializza la porta seriale sulla risorsa specificata in “Visa Resource name”, con un baudrate di 115200.
Continuando si trova il blocchetto “Visa Open” il quale apre la sessione specificata in precedenza. Il blocco “Flush I/O” effettua il flush di eventuali bit in coda.
Il blocco “Visa Write” è il blocco che manda i bit ricevuti dal blocco “Format into string”, quest’ultimo crea una stringa di lunghezza fissa, in maniera tale da scatenare un interrupt sulla board alla fine di ogni trasmissione ed aggiornare tutti i dati.
Esempio: al fine di mantenere una sintassi uguale per tutte le combinazioni di numeri, è stata utilizzata la tecnica dello Zero Padding cioè aggiungere degli zeri a sinistra del numero in questione. 
Tramite i knob sull’interfaccia grafica è possibile scegliere dei valori che vanno da 0 a 99.9 ma è evidente che se si volesse scegliere il numero 1, esso ha bisogno di una sola cifra, e la sintassi non è più rispettata. Un ingresso del blocco format into string è appunto il format stringS  il quale consente di specificare il tipo di dato in uscita da questo blocco. Utilizzando il comando %04.1f si ordina di utilizzare 4 cifre in totale, e laddove non siano sufficienti vengono aggiunti degli zeri a sinistra del numero da inviare. 
45.9  OK ……3.0 NO03.0OK
Una volta eseguita la scrittura sulla seriale, la comunicazione viene chiusa tramite il blocco Visa Close e successivamente vengono raccolti eventuali errori tramite un Error Handler

Spostandoci dal lato board la ricezione (e trasmissione) dei dati viene affidata ad una periferica, il DMA controller. 
	Ricezione con DMA
	Esempio: se si vuole inviare(o ricevere) dei dati da un sensore esterno(ad esempio una videocamera) il trasferimento dei dati stessi potrebbe richiedere parecchio tempo, ma in particolare la chiamata alla funzione HAL_UART_Transmit() è una chiamata bloccante, il che significa che comincia a mandare i dati non appena viene conclusa l’operazione precedente, e va avanti solo dopo aver completato la trasmissione, non lasciando la CPU ad altre operazioni, questo metodo è detto “polling mode”, ed è una chiamata di tipo bloccante. Questo implica che  se la CPU è impegnata esclusivamente nella trasmissione dei dati, non è possibile effettuare un operazione critica molto veloce. L’alternativa al polling mode è il DMA mode, cioè viene sfruttata un’altra periferica che non utilizza la cpu per la trasmissione/ricezione dei dati, quindi non è bloccante, ed è pnesata per applicazioni mission critical	.

 
Figure 11 - Esempio DMA

In questo programma viene creato un “while loop” parallelo al main loop. 
Prima che comincia il main loop viene richiamata la funzione HAL_UART_Recieve_DMA(…) la quale abilita la ricezione dei messaggi sulla seriale, e dopo di questo comando, parte il main.
Il messaggio che viene mandato da LabVIEW verso la board ha una struttura ben definita ed è una stringa composta da solo numeri i quali sono i parametri Kp,Ki,Kd e ref. Queste stringhe hanno una lunghezza fissa, ad esempio 14. Alla ricezione del 14-esimo byte, viene chiamato un interrupt che indica il completamento del buffer di ricezione, ed una volta ricevuti tutti, vengono estrapolati i parametri del sistema. Una volta ricavati questi parametri viene ri-chiamata la funzione HAL_UART_Recieve_DMA(…) proprio per ri-abilitare la ricezione dei messaggi da parte della seriale.

Esempio:
char rx_buffer[14];
…
HAL_UART_Receive_DMA(&huart2, (uint8_t *)rx_buffer,sizeof(rx_buffer));
while (1) {…}
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
kp = strtof(rx_buffer, &pEnd);
…
HAL_UART_Receive_DMA(huart, (uint8_t *)rx_buffer, sizeof(rx_buffer));
}


Ad esempio, la stringa può avere la seguente forma: 13.3 45.5 65.6
Questo indica che i valori da assegnare alle variabili sono: Kp=13.3, Ki=45.5, Kd=65.6.

	Meccanica del sistema
Nella realizzazione meccanica si è voluto modificare la parte che permette la rotazione del braccio sull’asta.
 
Figure 12 - Sistema realizzato
Come si può notare dalle foto il punto di fulcro del sistema non corrisponde al centro di istantanea rotazione del sistema, questo incide negativamente sulle prestazioni del sistema, in quanto il plant ad un angolo di ≃0° è già in una condizione di instabilità, poiché data la lunghezza del braccio, ed i pesi non ottimamente ripartiti, bastano pochissime rotazioni per far ribaltare il sistema. È stato realizzato questo giunto tra l’asta ed il braccio tramite una stampante 3d, e si è scelto di ancorare il braccio nella parte superiore del giunto e non in maniera radiale per massimizzare la robustezza dello stesso.
  
Figure 13 - Giunto asse-braccio			Figure 14 - culla sensore
Nelle foto precedenti si notano i render in 3d dei componenti stampati per la realizzazione del progetto.
	Progetti futuri	
Il sistema ad oggi viene equipaggiato con un motore in cc recuperato da un quadricottero dismesso. Questo componente è formato da più parti, e a causa di queste l’asta utilizzata trasmette molte vibrazioni al sensore stesso. Il motore a vuoto gira a velocità troppo alte per l’applicazione di un quadricottero quindi viene accoppiato ad un sistema di trasmissione in grado di diminuire la velocità dello stesso ed aumentarne la coppia. Questo accoppiamento dato da ruote dentate di plastica abbastanza economiche genera delle vibrazioni molto fastidiose, ed inoltre nel mentre della calibrazioni uno dei micromotori utilizzati è defunto a causa del tempo prolungato di utilizzo(spazzole bruciate). Qualsiasi cambio di input non è immediato. Inoltre questo motore non è in grado di generare una forza di lift elevata, quindi il braccio deve essere sempre di dimensioni contenute.
La soluzione proposta è quella di sostituire il motore cc con un motore brushless di piccole dimensioni. Nella sostituzione del motore è necessario mettere in conto che il controllo dello stesso non viene effettuato più mediante un ponte H ma bensì attraverso un ESC. 

	Tentativo di realizzazione ESC
 
Figure 15 - Schema 1/3 ESC
È stato realizzato 1/3 di esc all’interno del simulatore Pspice, e ne sono state verificate le specifiche di progetto.
 
Figure 16 - Oscilloscopio Pspice
Come è possibile notare, il segnale in blu è il segnale che genera un ipotetico uC ed il segnale rosso è il segnale in uscita dal gate driver.
In questo progetto è stato utilizzato un gate driver IR2110 che consente di pilotare una gamba dell’inverter, in modo da accoppiare il sss al uC. Questo schema viene ripetuto 3 volte, una per ogni fase del motore.
 
Figure 17 - Creazione PCB
Nella figura seguente si riconoscono le 3 gambe dell’inverter; in questa fase è stato necessario adattare il più possibile le capacità parassite date dalle lunghezze(e forme) delle piste, in quanto da sole possono compromettere il perfetto funzionamento del sistema.
 
Figure 18 – Board completata
	Teoria Inverter
Un inverter è un sistema tempo discreto, in grado di pilotare delle macchine elettriche AC(tempo continue); queste macchine elettriche sono assimilabili ad un motore sincrono trifase senza eccitazione, quindi quello che necessitano in ingresso sono 3 sinuisoidi sfasate di 120°, ma com’è possibile generare dei segnali del genere tramite un uC?
Generazione look-up table
Al fine di generare dei segnali simili ad una sinusoide è stato pensato un percorso inverso rispetto a quello che succede su un ADC; un adc restituisce un valore compreso tra 0 e 2^n-1 per esplicitare la tensione letta al proprio ingresso. In questo caso si sfrutta il teorema del valor medio per avere una tensione “discontinua” istante per istante, ma se opportunamente filtrata si ottiene un’onda sinusoidale.(Tecnica SPWM)
 
Figure 19 - "Dimostrazione" valor medio
Attraverso uno script MATLAB è stata generata una tabella che contiene i valori da andare ad inserire all’interno del registro di ARR del timer in questione.
  
Figure 20 					Figure 21
In questo programma veniva utilizzato il timer 1(Timer avanzato) che consente la generazione di PWM_negata ed il timer2(GP) come Timebase.
 
Figure 22
 
Figure 23 

Esempio:
Trig_Freq=F_clk/((PSC+1)(ARR+1))
out_signal=(Trig_freq)/NS

 
Figure 24
 
Figure 25 - interrupt sul timer 2
Nella figura 24 si mostra come ad ogni interrupt scatenato dal timer 2(timebase) venivano cambiati i valori nel registro CCRx(Control compare register) per il prossimo ciclo.
 
Figure 26 - Risultato vero e filtrato
 
	Configurazione Timer
Per via della componente altamente induttiva data dal motore e dai mosfet, il toggle di stato sui mosfet non è istantaneo ma è necessario applicare dei “dead time” affinché non si ottenga un cortocircuito sulla gamba. Questo dead-time è possibile impostarlo soltanto sui timer avanzati e lo si modifica attraverso l’ioc ;
 
Figure 27
 
Figure 28
 
	Problema Overshoot 
Durante i test dell’inverter a tensione ridotta è saltato subito all’occhio il valore dell’uscita delle varie fasi che era affetto da un overshoot assai marcata ed un oscillazione poco smorzata. 
In questo caso si è registrato un overshoot del 83%. Esempio: Immaginiamo di alimentare un brushless con Vn=100V, utilizzando un DC_BUS da 100V ci potrebbero essere dei picchi di tensione di circa 190V, e questa sovratensione perforerebbe il dielettrico rendendo inutilizzabile la macchina.
 
Figure 29 – Overshoot
Questi overshoot possono essere eliminati in parecchi modi: possibile implementare uno snubber,cioè un circuito RC posto sulla gamba inferiore; attraverso questo sistema si va a spostare il polo “vicino” all’asse imaginario. Un metodo più semplice, ma meno efficiente è quello di sostituire la resistenza di gate, con una più grossa, in questo modo si rende l’inverter più lento, ma complessivamente più sicuro.
 
Figure 30 - Tensione di Gate
Dopo aver risolto tutti i vari piccoli problemi si è giunti al test finale, e cioè quello di implementare una legge di controllo che riesca a regolare la forza di lift del motore.
	Feedback di posizione (rotorica)
Come è possibile notare, il programma sembrava svolgere abbastanza bene il suo lavoro, ma non è stato tenuto in conto il problema della posizione del rotore. Affinché il motore ruoti ad una data velocità con una determinata coppia è necessario conoscere la direzione degli assi d(iretto) e q(uadratura) in questo modo è possibile andare ad imprimere una certa id oppure una certa iq a seconda della posizione del rotore. In questo modo, non tenendo conto del motore e far ruotare i vettori id e iq a velocità fissa(senza feedback) se il motore perde il sincronismo(a seguito di un disturbo) sarebbe stato impossibile da far ripartire senza trascinare il motore ad una ω_meccanica≃ ω_elettrica/np. 
Questo problema sarebbe stato possibile da arginare attraverso un qualsiasi metodo di controllo di posizione. Data la natura del sistema, utilizzare un sensore per la misura della posizione è impossible, in quanto non è possibile aggiungere dei sensori effetto hall all’interno della carcassa del motore, ed è anche di difficile realizzazzione l’inserimento di un encoder assoluto calettato sull’albero del motore. Una possibile soluzione è quella della lettura della BEMF(Back electromotive force). Questa è la tensione indotta dal movimento del magnete permanente nell’unica fase del motore non alimentata. Attuando un circuito di ZCD(Zero crossing detecting) è possibile rilevare l’istante più adatto al cambio di stato degli interruttori a stato solido. 
 
Figure 31 - Spiegazione ZCD
 
Figure 32 - Schema ZCD
Con un circuito del genere è possibile avere un output “alto” quando la tensione sulla fase C è < 0V, quindi abilitando un interrupt è possibile chiamare una callback a seconda dell’interrupt che lo ha scatenato. In ogni caso la callback eseguirà un cambio di stato di tutti gli sss, evitando così di perdere il sincronismo in presenza di coppia resistente maggiore.
 
Parte Meccanica Brushless
 
Figure 33 - 3D render
 
Figure 34 – Stampa ultimata
