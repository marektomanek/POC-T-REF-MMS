# Instalace a uprava projektu pri migraci z LB V3 na V4

## Predpoklady
* Globalne instalovana verze LB4 (npm i -g @loopback/cli)
* Bezici kontejnery DB2, Kafka, konfigurovane podle prechozich standardnich instrukci

## Omezeni
* LB4 neumi generovat vice repositories na stejnym datasource -> Kafka repository s transakcni podporou je vytvorena rucne
* Projekt neni pripraven pro beh v OC kontejneru

## Zmeny proti projektu LB3
* Model databaze (tabulka MATERIAL) se vytvori pomoci prikazu **lb4 discover**. Nevytvari se rucne.

## Kroky pro aktivaci hot reload pri vyvoji:
* npm install nodemon --save-dev
* Novy script do sekce scripts v package.json: “localDev”: “nodemon .”,

## Spusteni v rezimu hot reload:
Osobne to delam v ramci integrovaneho terminalu Visual code, ktery si rozdelim na vice oken
* Prvni okno: npm run build:watch
* Druhe okno: npm run localDev
* Treti okno: pro prikazy LB4 xx

# Zakladni priprava projektu
* Vytvorit aplikaci (skeleton): lb4 app
	* Potvrdit predvolene hodnoty
	* Bohuzel projekt se vytvori v pod-adresari (nutno rucne prekopirovat zpet do root adresare projektu)

* Vytvorit DB2 datasource: lb4 datasource
	* Pojmenovat db2
	* Zadat zname hodnoty pro db testdb

* Overit zakladni funkcionalitu
	*	Pomoci prikazu: npm run localDev -> http://localhost:3000
	* Zkusit hot reload: napr. pridat radek v souboru **src/application.ts**

# Zmena databaze, pridani ciselniku skladu
Nove bude reseni demonstrovat i relacni vazbu mezi vice tabulkami. (pridat tabulku číselník MVM. Dva sloupce MVM, NAZEV. Následně vytvořili joined dotaz nad tabulkou Material. Cílem je to, abychom při ziskání dat z Material měli za sloupcem MVM i název MVM z číselníku MVM. )
* V kontejneru DB2 provest:
	* db2 connect
	* CREATE TABLE CISMVM (MVM CHAR(3) NOT NULL ,NAZEV VARCHAR(30),PRIMARY KEY(MVM));
	* DB2 drop TABLE material
	* CREATE TABLE MATERIAL (ID INT GENERATED BY DEFAULT AS IDENTITY NOT NULL, MVM CHAR(3) NOT NULL, KMAT VARCHAR(12), MNOZSTVI DECIMAL(10,3), HMOTNOST DECIMAL(10,3), PRIMARY KEY (ID), FOREIGN KEY MVM (MVM) REFERENCES CISMVM ON DELETE NO ACTION)

	* INSERT INTO CISMVM (MVM, NAZEV) VALUES ('wh1', 'Sklad c.1')
	* INSERT INTO CISMVM (MVM, NAZEV) VALUES ('wh2', 'Sklad c.2')
	* INSERT INTO MATERIAL (MVM, KMAT, MNOZSTVI, HMOTNOST) VALUES ('wh1', 'material1', 22, 44)
	* INSERT INTO MATERIAL (MVM, KMAT, MNOZSTVI, HMOTNOST) VALUES ('wh2', 'material22', 44, 88)
	* INSERT INTO MATERIAL (MVM, KMAT, MNOZSTVI, HMOTNOST) VALUES ('wh3', 'material33', 66, 99)

	* SELECT MATERIAL.ID, material.kmat, CISMVM.NAZEV FROM material, cismvm WHERE cismvm.mvm = MATERIAL.mvm ORDER BY MATERIAL.kmat

# Discovery existujici databaze DB2
* Vytvorit MODEL pro CISMV pomoci prikazu lb4 discover
	* Vybrat model **CISMVM**
* Vytvorit repository (CismvmRepository) pro danny model pomoci prikazu **lb4 repository**
* Vyvorit standardni controller **lb4 controller** (CiselnikMvn -> REST Controller with CRUD functions -> Cismvm -> CismvmRepository -> mvm -> string -> n -> //cismvms) pro danny model umoznujici zakladni CRUD operace nad tabulkou CISMVM
* Otestovat novy controller pomoci web exploreru

* Vytvorit MODEL pro MATERIAL pomoci prikazu lb4 discover
	* Vybrat model **MATERIAL**
* Vytvorit repository (MaterialRepository) pro danny model pomoci prikazu **lb4 repository**
* Vyvorit standardni controller **lb4 controller** (MaterialStandard -> REST Controller with CRUD functions -> Material -> MaterialRepository -> id -> number -> y -> /materials) pro danny model umoznujici zakladni CRUD operace nad tabulkou MATERIAL
* Otestovat novy controller pomoci web exploreru

### Omezeni&Varovani
Bohuzel automaticke generovani vykazuje drobne chyby, ktere je nutno rucne opravit v souboru **/src/models/material.model.ts**. Jedna se opravu spatne prirazeneho atributu **precision** u datovych typu DECIMAL, a ne-povinneho atributu **id** (required: true -> required: false). **POZOR: decimal je standardne prevaden na string v js/json formatu**

# Rucni vytvoreni "controlleru" pro integraci s Kafka a vytvoreni join relace material <-> cismvm
Principy:
* Logika souvisejici s Kafka bude umistena do dvou komponent typu [service](https://loopback.io/doc/en/lb4/Services.html)
* Na priklade join query material, cismvm bude demonstrovana moznost pristupu k jazyka SQL pomoci standardnich funci kompoenty typu repository

### Zdrojovy kod je k dispozici:
* Repository s transakcni podporou: **src/repositories/material.with.tx.repository.ts**
* Sluzby souviceji s kafkou: **src/services/...**
* Custom controller: **src/controllers/material.with.tx.repository.ts**

### Omezeni:
* Je implementovan pouze jednoduchy insert noveho materialu a jeho odeslani do Kafky topic v ramci jedne transakce
* Z obchodni logiky v3 je pouze k dispozi modelova implementace vyberu scenare ve sluzbe **src/services/scenario-simulator.service.ts**

### Zjednoduseny postup
Pro vytvoreni transakcni funkcionality analogicke s V3 (v ramci existujici db transacke ulozit data do kafky a nasledne realizovat rizeny commit/rollback) jsou nutne tyto kroky:
* Pridat npm moduly pro Kafku: npm install kafka-node
* Vytvorit novou repository s podporou transakci:
	* Kopirovat /src/repositories/material.repository.ts do **material.with.tx.repository.ts**
	* Prislusne upravit soubory **material.with.tx.repository.ts** a **index.ts** (klicove: zmenit DefaultCrudRepository -> **DefaultTransactionalRepository**)
* Vytvorit 2x sluzbu:
	* Pomoci prikazu **lb4 service** vytvorit obe sluzby typu **Local service class bound to application context**
	* Zkopirovat kod z prislusneho souboru z LB4 vetve
* Vytvorit prazdny controller **lb4 controller** (MaterialPostWithKafkaSubmit -> Empty Controller)
	* Zkopirovat kod z prislusneho souboru z LB4 vetve
* Overit funcionalitu pomoci Exploreru pro oba endpointy:
	* GET /get-materials-by-mvm​/{mvm}
	* POST /post-material-submit-kafka

## Priklady a dalsi zdroje
Jednoduchy query string pro Explorer pro get /materials:
	{
  "fields": {
    "hmotnost": true,
    "id": true,
    "kmat": true,
    "mnozstvi": true,
    "mvm": true
  },
  "offset": 0,
  "limit": 100,
  "skip": 0
	}


	* CREATE TABLE CISMVM2 (MVM CHAR(3) NOT NULL ,NAZEV VARCHAR(30),PRIMARY KEY(MVM));
	* DB2 drop TABLE material
	* CREATE TABLE MATERIAL2 (ID INT GENERATED BY DEFAULT AS IDENTITY NOT NULL, MVM CHAR(3) NOT NULL, KMAT VARCHAR(12), MNOZSTVI DECIMAL(10,3), HMOTNOST DECIMAL(10,3), PRIMARY KEY (ID))

	* INSERT INTO CISMVM (MVM, NAZEV) VALUES ('wh1', 'Sklad c.1')
	* INSERT INTO CISMVM (MVM, NAZEV) VALUES ('wh2', 'Sklad c.2')
	* INSERT INTO MATERIAL (MVM, KMAT, MNOZSTVI, HMOTNOST) VALUES ('wh1', 'material1', 22, 44)
	* INSERT INTO MATERIAL (MVM, KMAT, MNOZSTVI, HMOTNOST) VALUES ('wh2', 'material22', 44, 88)

	* INSERT INTO MATERIAL2 (MVM, KMAT, MNOZSTVI, HMOTNOST) VALUES ('wh1', 'material1', 22, 44)
	* INSERT INTO MATERIAL2 (MVM, KMAT, MNOZSTVI, HMOTNOST) VALUES ('wh2', 'material22', 44, 88)

	* SELECT MATERIAL.ID, material.kmat, CISMVM.NAZEV FROM material, cismvm WHERE cismvm.mvm = MATERIAL.mvm ORDER BY MATERIAL.kmat


"database": {
		  "client": "pg",
		  "connection": {
			"database": "architektDEV6",
			"host": "6c8ac4f9-6af3-4f04-8471-397f4ac43d84.bc28ac43cf10402584b5f01db462d330.databases.appdomain.cloud",
			"port": 32635,
			"password": "477369dc0347a52245c40a136dee4c4f2cf00047fb17d6c24d67131e5e5334c6",
			"user": "ibm_cloud_c8556321_c2f9_4f02_9b92_b4acd9184ff4",
			"ssl": true
		  }
	}
