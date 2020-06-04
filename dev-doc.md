---
title: Vývojářská dokumentace
layout: default
nav_order: 3
---

# Vývojářská dokumentace
Zde je popsáno jak je aplikace strukturována a jak funguje.

## Struktura
Aplikace využívá třívrstvou architekturu. Na datové vrstvě je použit Apache Solr, na aplikační vrstvě Spring Boot a na prezentační vrstvě Vue a Vuetify. Aplikace využívá Docker a Docker Compose pro kontejnerizaci, aby bylo možné aplikaci snadno nasadit. V rámci aplikace byla navíc vytvořena aplikace pro úpravu dat.

### Adresářová struktura:
```
RUIAN-search/
├── CSVModifier
│   ├── CSVModifier.iml
│   ├── Dockerfile
│   ├── dependency-reduced-pom.xml
│   ├── pom.xml
│   ├── src
│   └── target
├── LICENSE
├── README.md
├── backend
│   ├── Dockerfile
│   ├── pom.xml
│   ├── ruian-search.iml
│   ├── src
│   └── target
├── coreConfig
│   └── conf
├── docker-compose.yml
├── frontend
│   ├── Dockerfile
│   ├── README.md
│   ├── babel.config.js
│   ├── dist
│   ├── nginx.conf
│   ├── nginxConfig
│   ├── node_modules
│   ├── package-lock.json
│   ├── package.json
│   ├── public
│   ├── src
│   └── vue.config.js
├── index.sh
└── init.sh
```

- `CSVModifier` obsahuje zdrojové kódy pro aplikaci, která slouží pro úpravu dat.
- `backend` obsahuje zdrojové kódy backendu aplikace. 
- `coreConfig` obsahuje konfiguraci Apache Solr
- `frontend` obsahuje zdrojové kódy frontendu aplikace.
- `index.sh` slouží k indexaci a úpravě dat.
- `init.sh` slouží ke spuštění aplikace.


## Docker
Aplikace je rozdělena do 3 kontejnerů (služeb) - solr, backend, frontend. Jelikož je kontejnerů více, je pro jejich správu použit Docker compose. Definice a nastavení služeb se nachází v souboru `Docker-compose.yml`. Služby backend a frontend používají `Dockerfile` pro sestavení image. Backend využívá `backend/Dockerfile` a frotend využívá `frontend/Dockerfile`.
Všechny služby mají definovaný healthcheck, pomocí kterého je možné sledovat, jestli je služba dostupná. Healthcheck je následně využit ve skriptu pro spuštění aplikace.

### Reverzní proxy
Kontejner **frontend** slouží zároveň jako reverzní proxy. To umožňuje přístup k frontendu a backendu ze stejného portu. Aplikace využívá Nginx. Nastavení Nginx se nachází ve složce `frontend/nginxConf`.

## Skript pro spuštění aplikace
Jedná se o Bash skript, který volá příkaz `docker-compose up`. Skript průběžně kontroluje stav služeb. Jakmile jsou všechny služby dostupné, vypíše do příkazové řádky "Application is ready".

`init.sh`
```bash
#!/bin/bash
getContainerHealth () {
  docker inspect --format "{{json .State.Health.Status }}" $1
}

waitContainer () {
  while STATUS=$(getContainerHealth $1); [ "$STATUS" != "\"healthy\"" ]; do
    if [ "$STATUS" = "\"unhealthy\"" ]; then
      echo "Failed!"
      exit 1
    fi
    sleep 1
  done
  echo "$1 is ready"
}

waitContainers () {
  waitContainer solr
  waitContainer backend
  waitContainer frontend
}

if [ "$1" == "build" ]; then
	docker-compose up --build -d
else
	docker-compose up -d
fi
echo "Docker-compose is running"
waitContainers
echo "Application is ready"
```

Pokud se skript zavolá s argumentem build, dojde k sestavení kontejnerů, to je třeba, pokud se provedou nějaké změny v kódu. Funkce `waitContainers` volá funkci `waitContainer` pro jednotlivé kontejnery. Funkce `waitContainer` kontroluje stav kontejnerů, k volání funkce je nutné přidat název kontejneru. Pokud by se přidal nový kontejner, bylo by třeba přidat volání `waitContainer` i pro něj. 

Kontejner se spustí nezávisle na funkci `waitContainer` jen se může stát, že kontejner nebude ještě dostupný i když skript vypíše "Application is ready".

## Indexace dat
Pro indexaci dat slouží skript `index.sh`. Aby skript fungoval správně, je nutné sestavit image, který umožní úpravu dat. Image stačí sestavit jednou. Další build je nutný pouze pokud provedete změny v aplikaci pro úpravu dat. Ke skriptu je možné přidat argument `build` pro sestavení image.

Skript stáhne zip soubor s daty RÚIAN z aplikace pro [Nahlížení do katastru nemovitostí](https://nahlizenidokn.cuzk.cz/StahniAdresniMistaRUIAN.aspx). Zip soubor obsahuje přes 6000 CSV souborů. Po rozbalení skript převede kódování souborů z Windows-1250 do UTF-8 pomocí nástroje iconv. Následně se v dočasném kontejneru provede provede úprava souborů. Přidají se dva nové sloupce, jeden sloupec s názvem identifikace, který obsahuje číslo domovní a případně číslo orientační, pokud jej daný záznam obsahuje. Druhý sloupec obsahuje souřadnice zkonvertované z Křovákova zobrazení do WGS-84.

```
docker run --rm -v "$PWD/temp:/temp" -v "$PWD/data:/data" csvmodifier
```
Do kontejneru se namapují složky `temp` a `data`. Ve složce `temp` jsou vstupní data. Aplikace pro úpravu dat uloží upravená data do složky `data`.

Poté skript smaže existující data v Apache Solr a pomocí Post Tool nahraje nová data.

## Aplikace pro úpravu dat
Zdrojové kódy aplikace se nachází ve složce `CSVModifier`. Aplikace je napsaná v jazyce Java a využívá knihovnu Geotools pro transformaci souřadnic.

Aplikace se skládá ze tří tříd.
- `CSVModifier` hlavní třída.
- `Modifier` třída, která provádí čtení CSV souborů a zápis do nich.
- `CoordinatesConverter` třída sloužící k transformaci souřadnic

Součástí složky `CSVModifier` je `Dockerfile`, pomocí kterého je možné sestavit image aplikace.

## Apache Solr
Instance Apache Solr obsahuje jedno jádro s názvem `ruian`. Pomocí Docker volumes je do kontejneru promapována upravená konfigurace, která se nachází ve složce `CoreConfig`. Složka `CoreConfig/conf` obsahuje upravené schéma `managed-schema`. Do schématu byl přidán nový typ pole `custom_string`. Dále byly přidány definice nových polí, aby se data správně indexovala.

```xml
<fieldType name="custom_string" class="solr.TextField">
<analyzer type="index">
	<tokenizer class="solr.KeywordTokenizerFactory"/>
	<filter class="solr.LowerCaseFilterFactory"/>
	<filter class="solr.EdgeNGramFilterFactory" 
			minGramSize="1" maxGramSize="35" />
</analyzer>
<analyzer type="query">
	<tokenizer class="solr.KeywordTokenizerFactory"/>
	<filter class="solr.LowerCaseFilterFactory"/>
</analyzer>
</fieldType>
```

Apache Solr využívá Docker volumes, aby nebylo třeba data indexovat po každém restartu aplikace.

`Docker-compose.yml`
```yml
version: '3'

services:
  solr:
    container_name: solr
    image: solr:8.3
    ports:
      - 8983:8983
    volumes:
	  - data:/var/solr

...

volumes:
  data:
```

## Backend
Backend slouží k přístupu k datům v Apache Solr, využívá k tomu knihovnu Spring Data Solr.

Aplikace je rozdělená na repozitáře, služby a kontrolery. Kontrolery definují HTTP API, pomocí kterého je možné s aplikací komunikovat. Kontrolery skrze služby volají příslušné repozitáře. Repozitáře slouží k přístupu k datům. Aplikace obsahuje dva repozitáře. Repozitář `AddressRepository.java` rozšiřuje `SolrCrudRepository`. Dotazy se zde vytváří na základě názvu metod, nebo podle anotací. V repozitáři `AddressCustomRepository.java` se vytváří dotazy manuálně, aby bylo možné výsledky dotazů seskupovat.

### Připojení k Apache Solr
Nastavení připojení k Apache Solr je definováno ve třídě `config/SolrConfig.java`. Ve třídě `model/Address.java` je definována struktura dokumentu Apache Solr, který má aplikace očekávat. Na základě této třídy se generují z dokumentů Java objekty.

### HTTP API
Rozhraní je popsáno pomocí Open API. Dokumentace je dostupná zde: [https://app.swaggerhub.com/apis-docs/letomas/Address-search-RUIAN/1.0.0-oas3](https://app.swaggerhub.com/apis-docs/letomas/Address-search-RUIAN/1.0.0-oas3)

## Frontend
Frontend využívá Vue a Vuetify pro tvorbu uživatelského rozhraní.
Dále používá Vuex pro správu stavu aplikace. Nastavení Vuex se nachází v `src/store/index.js`. Směrování je definováno v souboru `src/router/index.js`.

Ve složce `src/views` se nacházejí komponenty, které se používají při směrování. Ve složce `src/components` se nachází pomocné komponenty.

V rámci frontendu vznikly dva pomocné moduly. Jeden slouží pro konzumaci HTTP API: `src/api.js`. Konzumace rozhraní byla oddělena do modulu kvůli modularitě. Takto je snadné modul upravit (není třeba měnit kód všude, kde se posílají požadavky na backend), nebo jej vyměnit za jiný. Druhý modul slouží pro skládání adres dle vyhlášky č. 359/2011 Sb. `src/addressBuilder.js`.