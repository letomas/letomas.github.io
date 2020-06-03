---
title: Administrátorská dokumentace
layout: default
nav_order: 4
---

# Administrátorská dokumentace
Administrátorská dokumentace obsahuje požadavky pro stuštění aplikace a intrukce pro její nasazení.

Odkaz na Github: [https://github.com/letomas/RUIAN-search](https://github.com/letomas/RUIAN-search)

Commit: `8b12c970f8e953fb3f9cc2b8fb6b2464db5cc0e9`

### Navigace
  - [Požadavky](#po%c5%beadavky)
  - [Spuštění](#spu%c5%a1t%c4%9bn%c3%ad)
  - [Indexace dat](#indexace-dat)
  - [Konfigurace](#konfigurace)

## Požadavky
- Linux kernel verze 3.10 nebo vyšší
- Docker
- Docker Compose
- Bash
- Curl
- Zip

## Spuštění
>**Upozornění:** Pro spuštění je nutné zachovat unixové konce řádků (LF). Pokud máte v gitu nastavené automatické převádění na CRLF, vypněte toto nastavení pomocí příkazu `git config --global core.autocrlf input`. Popřípadě zkonvertujte konce řádků pomocí nástroje dos2unix nebo textového editoru.

Po stažení/naklonování projektu z [Githubu](https://github.com/letomas/RUIAN-search) je možné aplikaci spustit pomocí skriptu:
```
./init.sh
```
Při provedení změn je nutné přidat argument `build`:
```
./init.sh build
```
Tento skript volá docker-compose up -d (případně ještě s argumentem --build) a kontroluje stav kontejnerů. Jakmile jsou všechny kontejnery připravené, vypíše se do příkazové řádky: `Application is ready`. Při prvním spuštětní a při aktualizaci dat je třeba použít příslušný skript viz [indexace dat](#indexace-dat).


## Indexace dat
K nahrání a aktualizaci dat slouží následující skript:
```
./index.sh
```
Skript potřebuje ke spuštění zbuildit image, který umožní úpravu CSV souborů. Image stačí zbuildit jednou. Další build je nutný pouze pokud provedete změny v aplikaci pro úpravu dat.

 Image je možné zbuildit buď přidáním argumentu `build`:
```
./index.sh build
```
, nebo manuálně tímto příkazem:
```
docker build --tag csvmodifier ./CSVModifier/
```
Pokud se rozhodnete build image provést manuálně, musíte jej provést před spuštěním skriptu pro indexaci.

Skript stáhne zip soubor, který osahuje přes 6000 csv souborů, jeden soubor pro každou obec v ČR. Zip soubor je nutné rozbalit. Csv soubory se musí upravit (zkonvertovat kódování z Windows-1250 na UTF-8 a přidat sloupce s identifikací a zkonvertovanými souřadnicemi) a následně nahrát do Solru pomocí Post toolu (nástroj pro nahrávání souborů do Solru přes příkazovou řádku). Celý proces může zabrat přibližně 20 minut. Data se uchovávají v Docker volume, takže při restartu aplikace není třeba data znovu indexovat. Data jsou aktualizována měsičně, poslední den v měsíci.

Po spuštění je aplikace dostupná z [http://localhost:8000](http://localhost:8000)

## Konfigurace
Aplikace je ve výchozím stavu dostupná z portu 8000. Toto nastavení je možné změnit v `Docker-compose.yml`. Je třeba změnit nastavení portů u služby **vue-app**.