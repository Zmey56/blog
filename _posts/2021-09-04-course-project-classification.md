---
toc: true
layout: post
comments: true
description: Course project "MySQL database for the financial instruments market."
categories: [course project, mysql, geekbrain]
title: Course project "MySQL database for the financial instruments market."
---

According to the task presented, I needed to come up with a topic and create a database based on it.Here is a general description of the project:

**Requirements for the course project:**
1. Make a general text description of the database and the tasks it solves;
2. the minimum number of tables is 10;
3. scripts for creating a database structure (with primary keys, indexes, foreign keys);
4. create an ER Diagram for the database;
5. scripts for filling the database with data;
6. scripts of characteristic samples (including groupings, joins, nested tables);
7. submissions (minimum 2);
8. stored procedures/triggers;

*Examples: describe the data storage model of a popular website: kinopoisk, booking.com , wikipedia, online store, geekbrains, public services...*

I took [the article] as a basis (https://vertabelo.com/blog/a-data-model-for-trading-stocks-funds-and-cryptocurrencies /) posted on a foreign source. It shows the basic scheme. It remains only to translate it into SQL. First of all, the translation of the theory.

Trading cryptocurrencies, buying stocks and the like are extremely popular these days, as it is perceived as an easy profit. Prices are currently rising, but we can't know when that will change. On the other hand, we know that this will happen at some point. But we are not here to make financial forecasts. Instead, we'll talk about a data model that can be used to support trading in cryptocurrencies and financial instruments such as stocks or fund stocks.

### 1. General text description of the database and the tasks to be solved

**What You Need to Know About Trading Currencies and Stocks**

Technological improvements over the past few decades have had a significant impact on trade. There are currently many online trading platforms that you can use. Most of today's trading is done virtually – you can see paper stocks in museums, but it is unlikely that you will see stocks that you buy in paper form. And you don't have to pack your bags and go to Wall Street or any other stock exchange to make a deal. From the comfort of your computer or mobile device, you can buy or sell derivative financial instruments (such as bonds, stocks or commodities).

Most transactions (sale of derivative financial instruments) follow the same rules. There are sellers and buyers. If they agree on the price, the deal will take place. After the transaction, the price of this derivative financial instrument will be recalculated, and the process will continue with new traders. Stocks and other derivative financial instruments work the same way.

What is cryptocurrency? You've probably heard of bitcoin and other cryptocurrencies. But what is it? Cryptocurrencies are similar to virtual currencies, but they are not tied to real world currencies (such as euros or dollars). Instead, users can trade cryptocurrencies among themselves as tokens. Then they can negotiate a sale that will turn their tokens into real money. These sales function in exactly the same way as the stock and stock transactions described above.

This topic is complex, and there may be many details in our model (for example, records of documents and transactions). I'm going to make it simple; I'm not going to implement any automated trading or any formulas to create new prices after a trading event.

Let's move on to the code. According to the article , the database consists of three blocks:

* CURRENCIES
* TRADERS
* ITEMS

Now let's write the code for each block:

###2. Scripts for creating a database structure

**CURRENCIES**

~~~~sql
CREATE DATABASE IF NOT EXISTS coursework_portfolio;
USE coursework_portfolio;
~~~~

First I create the country from which the trade is carried out

~~~~sql
DROP TABLE IF EXISTS country;

CREATE TABLE country(
	id SERIAL PRIMARY KEY,
	country VARCHAR(128)
) COMMENT = 'The country from which the trade is carried out';
~~~~

The currency that the user uses for trading

~~~~sql
DROP TABLE IF EXISTS currency_used;


CREATE TABLE currency_used(
	id SERIAL PRIMARY KEY,
	country_id BIGINT UNSIGNED NOT NULL COMMENT 'Key to another table',
	currency_id INT UNSIGNED COMMENT 'Key to another table',
	data_from DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT 'Start date of currency usage',
	data_to DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT 'The end date of currency usage. If NULL, then the currency is still in use'
) COMMENT = 'Currency used for purchase';

DESC currency_used ;
~~~~

Current and historical exchange rates between currency pairs are stored.

~~~~sql
DROP TABLE IF EXISTS currency_rate;

CREATE TABLE currency_rate(
	id SERIAL PRIMARY KEY,
	currency_id INT UNSIGNED COMMENT 'Key to another table',
	base_currency_id INT UNSIGNED COMMENT 'Key to another table',
	rate DECIMAL(16,6) NOT NULL DEFAULT 0 COMMENT 'Currency exchange rate',
	ts DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT 'The time at which this course was fixed'
) COMMENT = 'Currency exchange rate';
~~~~

Store all the currencies that we have ever used for trading.

~~~~sql
DROP TABLE IF EXISTS currency;

CREATE TABLE currency(
	id INT UNSIGNED PRIMARY KEY,
	code VARCHAR(8) NOT NULL UNIQUE COMMENT 'The code used for the unique designation of the currency',
	name VARCHAR(128) NOT NULL UNIQUE COMMENT 'The unique name of this currency',
	is_active BOOL DEFAULT FALSE COMMENT 'If the currency is currently active in our system',
	is_base_currency BOOL DEFAULT FALSE COMMENT 'If this currency is the base currency of our system.'
) COMMENT = 'Currency exchange rate';
~~~~

**ITEMS**

The item tables define all the products available for trading and their current status. It also records all the changes that have occurred with these products over time.

~~~~sql
DROP TABLE IF EXISTS item;

CREATE TABLE item(
	id SERIAL PRIMARY KEY,
	code VARCHAR(64) NOT NULL UNIQUE COMMENT 'The code used for the unique designation of the product (shares, mutual funds, etc.)',
	name VARCHAR(255) NOT NULL UNIQUE COMMENT 'Full name',
	is_active BOOL DEFAULT FALSE COMMENT 'Is this product available for trading or not',
	currency_id INT UNSIGNED COMMENT 'Refers to the currency used as the base currency for this product',
	details TEXT COMMENT 'All additional information (for example, the number of shares issued) in text format.'
) COMMENT = 'Available products';
~~~~

The price table tracks all price changes over time.

~~~~sql
DROP TABLE IF EXISTS price;

CREATE TABLE price(
	id SERIAL PRIMARY KEY,
	item_id BIGINT UNSIGNED COMMENT 'Refers to the currency used as the base currency for this product',
	currency_id INT UNSIGNED COMMENT 'Refers to the currency used as the base currency for this product',
	buy DECIMAL(16,6) NOT NULL DEFAULT 0 COMMENT 'Purchase rate',
	sell DECIMAL(16,6) NOT NULL DEFAULT 0 COMMENT 'Selling rate',
	ts DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT 'The time at which the transaction at the last price was fixed'
) COMMENT = 'Price change';
~~~~

Report table

~~~~sql
DROP TABLE IF EXISTS report;

CREATE TABLE report(
	id SERIAL PRIMARY KEY,
	trading_data DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT 'Report date',
	item_id BIGINT UNSIGNED COMMENT 'Refers to the currency used as the base currency for this product',
	currency_id INT UNSIGNED COMMENT 'Refers to the currency used as the base currency for this product',
	first_price DECIMAL(16,6) DEFAULT NULL COMMENT 'Initial price',
	last_price DECIMAL(16,6) DEFAULT NULL COMMENT 'Last price',
	min_price DECIMAL(16,6) DEFAULT NULL COMMENT 'Minimum price',
	max_price DECIMAL(16,6) DEFAULT NULL COMMENT 'Maximum price',
	avg_price DECIMAL(16,6) DEFAULT NULL COMMENT 'Average price',
	total_amount DECIMAL(16,6) DEFAULT NULL COMMENT 'The total amount paid for this product during the reporting period.',
	quantity DECIMAL(16,6) DEFAULT NULL COMMENT 'The number of products sold during this reporting period.'
) COMMENT = 'Report';
~~~~

**TRADERS**

Table of traders

~~~~sql
DROP TABLE IF EXISTS trader;

CREATE TABLE trader (
	id SERIAL PRIMARY KEY,
    firstname VARCHAR(50) COMMENT 'Name',
    lastname VARCHAR(50) COMMENT 'Surname',
    user_name VARCHAR(50) NOT NULL UNIQUE COMMENT 'Everyone's login is unique',
    email VARCHAR(120) NOT NULL UNIQUE,
    confirmation_code VARCHAR(120) NOT NULL COMMENT 'The code sent to the user to complete the registration process.',
    time_registered DATETIME DEFAULT CURRENT_TIMESTAMP,
    time_confirmed DATETIME DEFAULT CURRENT_TIMESTAMP,
    country_id BIGINT UNSIGNED COMMENT 'The country in which he lives.',
    preffered_currency_id BIGINT UNSIGNED COMMENT 'The currency that the trader prefers'
) COMMENT 'users';
~~~~

A list of all the products that the trader currently owns

~~~~sql
DROP TABLE IF EXISTS current_inventory;

CREATE TABLE current_inventory (
	id SERIAL PRIMARY KEY,
	trader_id BIGINT UNSIGNED COMMENT 'Link to the trader',
	item_id BIGINT UNSIGNED COMMENT 'Product link',
	quantity DECIMAL(16,6) DEFAULT NULL COMMENT 'Number of products'
) COMMENT 'List of products';
~~~~

Trading Event

~~~~sql
DROP TABLE IF EXISTS trade;

CREATE TABLE trade (
	id SERIAL PRIMARY KEY,
	item_id BIGINT UNSIGNED COMMENT 'Product link',
	seller_id BIGINT UNSIGNED DEFAULT NULL COMMENT 'Link to the trader',
	buyer_id BIGINT UNSIGNED COMMENT 'Link to the trader',
	quantity DECIMAL(16,6) DEFAULT NULL COMMENT 'Number of products',
	unit_price DECIMAL(16,6) DEFAULT NULL COMMENT 'Price per unit',
	description TEXT COMMENT 'All additional information (for example, the number of shares issued) in text format.',
	offer_id BIGINT UNSIGNED COMMENT 'Transaction Identifier'
) COMMENT 'Transactions';
~~~~

Accounting for all offers

~~~~sql
DROP TABLE IF EXISTS offer;

CREATE TABLE offer (
	id SERIAL PRIMARY KEY,
	item_id BIGINT UNSIGNED COMMENT 'Product link',
	trader_id BIGINT UNSIGNED DEFAULT NULL COMMENT 'Link to the trader',
	quantity DECIMAL(16,6) DEFAULT NULL COMMENT 'Number of products',
	buy BOOL DEFAULT FALSE,
	sell BOOL DEFAULT FALSE,
	price DECIMAL(16,6) DEFAULT NULL COMMENT 'Desired price per unit',
	ts DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT 'When was it exposed',
	is_active BOOL DEFAULT FALSE COMMENT 'Is this offer still valid'
) COMMENT 'Transactions';
~~~~

I also decided to add the birthday column to traders

~~~~sql
ALTER TABLE trader
  DROP COLUMN birthday;

ALTER TABLE trader
ADD COLUMN birthday DATE ;
~~~~

### 3. Relationship creation scripts

~~~~sql
USE coursework_portfolio;
~~~~

one-to-many relationship currency - currency_used

~~~~sql
ALTER TABLE currency_used ADD CONSTRAINT currency_used_country_fk FOREIGN KEY(country_id) REFERENCES country(id);
~~~~

one-to-many relationship currency - currency_used

~~~~sql
ALTER TABLE currency_used
ADD CONSTRAINT currency_used_currency_fk
FOREIGN KEY(currency_id)
REFERENCES currency(id);
~~~~

one-to-many relationship currency - currency_rate

~~~~sql
ALTER TABLE currency_rate ADD CONSTRAINT currency_currency_rate_fk FOREIGN KEY(currency_id) REFERENCES currency(id);
ALTER TABLE currency_rate ADD CONSTRAINT currency_currency_rate_base_fk FOREIGN KEY(base_currency_id) REFERENCES currency(id);
~~~~

one - to- many currency - price relationship

~~~~sql
ALTER TABLE price ADD CONSTRAINT currency_price_fk FOREIGN KEY(currency_id) REFERENCES currency(id);
~~~~

one-to-many currency - item relationship

~~~~sql
ALTER TABLE item ADD CONSTRAINT currency_item_fk FOREIGN KEY(currency_id) REFERENCES currency(id);
~~~~

one- to- many relationship price - item

~~~~sql
ALTER TABLE price ADD CONSTRAINT item_price_fk FOREIGN KEY(item_id) REFERENCES item(id);
~~~~

one-to-many relationship report - item

~~~~sql
ALTER TABLE report ADD CONSTRAINT report_item_fk FOREIGN KEY(item_id) REFERENCES item(id);
~~~~

one - to - many report - currency relationship

~~~~sql
ALTER TABLE report ADD CONSTRAINT currency_report_fk FOREIGN KEY(currency_id) REFERENCES currency(id);
~~~~

one-to-many trader - currency connection

~~~~sql
ALTER TABLE trader ADD CONSTRAINT country_trader_fk FOREIGN KEY(country_id) REFERENCES country(id);
~~~~

one-to-many relationship trade - trader

~~~~sql
ALTER TABLE trade ADD CONSTRAINT trader_trade_sell_fk FOREIGN KEY(seller_id) REFERENCES trader(id);
~~~~

one-to-many relationship trade - trader

~~~~sql
ALTER TABLE trade ADD CONSTRAINT trader_trade_buy_fk FOREIGN KEY(buyer_id) REFERENCES trader(id);
~~~~

one - to - many trade - offer relationship

~~~~sql
ALTER TABLE trade ADD CONSTRAINT offer_trade_fk FOREIGN KEY(offer_id) REFERENCES offer(id);
~~~~

one-to-many relationship trade - item
~~~~sql
ALTER TABLE trade ADD CONSTRAINT item_trade_fk FOREIGN KEY(item_id) REFERENCES item(id);
~~~~

one-to-many relationship offer - trader

~~~~sql
ALTER TABLE offer ADD CONSTRAINT trader_offer_fk FOREIGN KEY(trader_id) REFERENCES trader(id);
~~~~

one-to- many relationship offer - item

~~~~sql
ALTER TABLE offer ADD CONSTRAINT item_offer_fk FOREIGN KEY(item_id) REFERENCES item(id);
~~~~

one-to-many relationship Current_inventory - trader

~~~~sql
ALTER TABLE current_inventory ADD CONSTRAINT trader_current_inventory_fk FOREIGN KEY(trader_id) REFERENCES trader(id);
~~~~

one-to-many relationship Current_inventory - item

~~~~sql
ALTER TABLE current_inventory ADD CONSTRAINT item_current_inventory_fk FOREIGN KEY(item_id) REFERENCES item(id);
~~~~

### 4. Creating an ERDiagram for a database

The result is the following scheme:
---

![](/images/ER_diagram.png)

### 5. Scripts for filling the database with data

The filling was done through a popular site for generating fake data [Dummy Data for MYSQL Database] (http://filldb.info /)

~~~~sql
CREATE DATABASE coursework_portfolio;

-- MariaDB dump 10.17  Distrib 10.4.15-MariaDB, for Linux (x86_64)
--
-- Host: mysql.hostinger.ro    Database: u574849695_20
-- ------------------------------------------------------
-- Server version	10.4.15-MariaDB-cll-lve

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!40101 SET NAMES utf8mb4 */;
/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
/*!40103 SET TIME_ZONE='+00:00' */;
/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;

--
-- Table structure for table `country`
--
USE coursework_portfolio;

DROP TABLE IF EXISTS `country`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `country` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `country` varchar(128) COLLATE utf8mb4_unicode_ci DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=11 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='The country from which the trade is carried out';
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `country`
--

LOCK TABLES `country` WRITE;
/*!40000 ALTER TABLE `country` DISABLE KEYS */;
INSERT INTO `country` VALUES (1,'Bermuda'),(2,'Christmas Island'),(3,'Holy See (Vatican City State)'),(4,'Macedonia'),(5,'Qatar'),(6,'Tokelau'),(7,'Romania'),(8,'Central African Republic'),(9,'Uganda'),(10,'Saint Pierre and Miquelon');
/*!40000 ALTER TABLE `country` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `currency`
--

DROP TABLE IF EXISTS `currency`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `currency` (
  `id` int(10) unsigned NOT NULL,
  `code` varchar(8) COLLATE utf8mb4_unicode_ci NOT NULL COMMENT 'The code used for the unique designation of the currency',
  `name` varchar(128) COLLATE utf8mb4_unicode_ci NOT NULL COMMENT 'The unique name of this currency',
  `is_active` tinyint(1) DEFAULT 0 COMMENT 'If the currency is currently active in our system',
  `is_base_currency` tinyint(1) DEFAULT 0 COMMENT 'If this currency is the base currency of our system.',
  PRIMARY KEY (`id`),
  UNIQUE KEY `code` (`code`),
  UNIQUE KEY `name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Currency exchange rate';
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `currency`
--

LOCK TABLES `currency` WRITE;
/*!40000 ALTER TABLE `currency` DISABLE KEYS */;
INSERT INTO `currency` VALUES (0,'htxs','eaque',0,0),(1,'nnlp','soluta',1,0),(2,'dtlh','quis',0,0),(3,'wimu','animi',0,0),(4,'lnag','earum',0,0),(5,'ekbd','molestiae',1,0),(6,'nivw','nam',1,0),(7,'zldi','consequuntur',1,1),(8,'zyrs','qui',1,1),(9,'uvfn','blanditiis',1,1);
/*!40000 ALTER TABLE `currency` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `currency_rate`
--

DROP TABLE IF EXISTS `currency_rate`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `currency_rate` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `currency_id` int(10) unsigned DEFAULT NULL COMMENT 'Key to another table',
  `base_currency_id` int(10) unsigned DEFAULT NULL COMMENT 'Key to another table',
  `rate` decimal(16,6) NOT NULL DEFAULT 0.000000 COMMENT 'Currency exchange rate',
  `ts` datetime DEFAULT current_timestamp() COMMENT 'The time at which this course was fixed',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=11 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Currency exchange rate';
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `currency_rate`
--

LOCK TABLES `currency_rate` WRITE;
/*!40000 ALTER TABLE `currency_rate` DISABLE KEYS */;
INSERT INTO `currency_rate` VALUES (1,0,0,248964.149700,'1992-06-09 14:41:36'),(2,1,1,390368.888800,'2017-09-18 05:43:25'),(3,2,2,0.000000,'2001-11-16 22:01:24'),(4,3,3,50.506542,'1979-06-15 23:49:48'),(5,4,4,81.498779,'1984-06-23 02:36:52'),(6,5,5,584941.290982,'1970-04-04 16:47:28'),(7,6,6,6265204.580000,'1979-09-29 09:25:05'),(8,7,7,16.154948,'2010-12-21 00:43:41'),(9,8,8,8.630000,'1986-04-16 05:48:55'),(10,9,9,2.800000,'1984-10-06 10:07:42');
/*!40000 ALTER TABLE `currency_rate` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `currency_used`
--

DROP TABLE IF EXISTS `currency_used`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `currency_used` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `country_id` bigint(20) unsigned NOT NULL COMMENT 'Key to another table',
  `currency_id` int(10) unsigned DEFAULT NULL COMMENT 'Key to another table',
  `data_from` datetime DEFAULT current_timestamp() COMMENT 'Start date of currency usage',
  `data_to` datetime DEFAULT current_timestamp() COMMENT 'The end date of currency usage. If NULL, then the currency is still in use',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=11 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Currency used for purchase';
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `currency_used`
--

LOCK TABLES `currency_used` WRITE;
/*!40000 ALTER TABLE `currency_used` DISABLE KEYS */;
INSERT INTO `currency_used` VALUES (1,1,0,'1972-07-14 21:53:10','1989-06-25 02:18:23'),(2,2,1,'1970-03-19 12:37:26','2018-12-11 16:34:23'),(3,3,2,'1989-06-15 13:50:05','1985-06-10 17:07:32'),(4,4,3,'2008-09-26 16:36:58','1992-01-21 09:35:20'),(5,5,4,'1984-09-02 16:32:24','2019-07-25 03:33:31'),(6,6,5,'2018-10-02 04:32:51','1973-08-23 18:43:00'),(7,7,6,'2004-07-04 22:17:20','1990-11-19 16:48:34'),(8,8,7,'2007-06-25 10:34:33','1997-09-16 08:51:40'),(9,9,8,'1977-10-01 11:13:18','1985-03-05 19:28:04'),(10,10,9,'1987-03-28 10:28:24','2003-02-05 18:20:56');
/*!40000 ALTER TABLE `currency_used` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `current_inventory`
--

DROP TABLE IF EXISTS `current_inventory`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `current_inventory` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `trader_id` bigint(20) unsigned DEFAULT NULL COMMENT 'Link to the trader',
  `item_id` bigint(20) unsigned DEFAULT NULL COMMENT 'Product link',
  `quantity` decimal(16,6) DEFAULT NULL COMMENT 'Number of products',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=11 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='List of products';
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `current_inventory`
--

LOCK TABLES `current_inventory` WRITE;
/*!40000 ALTER TABLE `current_inventory` DISABLE KEYS */;
INSERT INTO `current_inventory` VALUES (1,1,1,8421.000000),(2,2,2,969106.000000),(3,3,3,9235206.000000),(4,4,4,92961.000000),(5,5,5,145700.000000),(6,6,6,0.000000),(7,7,7,19133.000000),(8,8,8,0.000000),(9,9,9,26582.000000),(10,10,10,78483052.000000);
/*!40000 ALTER TABLE `current_inventory` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `item`
--

DROP TABLE IF EXISTS `item`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `item` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `code` varchar(64) COLLATE utf8mb4_unicode_ci NOT NULL COMMENT 'The code used for the unique designation of the product (shares, mutual funds, etc.)',
  `name` varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL COMMENT 'Full name',
  `is_active` tinyint(1) DEFAULT 0 COMMENT 'Is this product available for trading or not',
  `currency_id` int(10) unsigned DEFAULT NULL COMMENT 'Refers to the currency used as the base currency for this product',
  `details` text COLLATE utf8mb4_unicode_ci DEFAULT NULL COMMENT 'All additional information (for example, the number of shares issued) in text format.',
  PRIMARY KEY (`id`),
  UNIQUE KEY `code` (`code`),
  UNIQUE KEY `name` (`name`)
) ENGINE=InnoDB AUTO_INCREMENT=11 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Available products';
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `item`
--

LOCK TABLES `item` WRITE;
/*!40000 ALTER TABLE `item` DISABLE KEYS */;
INSERT INTO `item` VALUES (1,'kzgb','doloremque',1,0,NULL),(2,'krcn','omnis',1,1,NULL),(3,'ywfp','rerum',0,2,NULL),(4,'bbjq','eaque',0,3,NULL),(5,'hsib','quis',1,4,NULL),(6,'dnuf','quia',1,5,NULL),(7,'wnfb','numquam',0,6,NULL),(8,'pefi','quos',0,7,NULL),(9,'vbrv','expedita',1,8,NULL),(10,'afem','esse',0,9,NULL);
/*!40000 ALTER TABLE `item` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `offer`
--

DROP TABLE IF EXISTS `offer`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `offer` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `item_id` bigint(20) unsigned DEFAULT NULL COMMENT 'Product link',
  `trader_id` bigint(20) unsigned DEFAULT NULL COMMENT 'Link to the trader',
  `quantity` decimal(16,6) DEFAULT NULL COMMENT 'Number of products',
  `buy` tinyint(1) DEFAULT 0,
  `sell` tinyint(1) DEFAULT 0,
  `price` decimal(16,6) DEFAULT NULL COMMENT 'Desired price per unit',
  `ts` datetime DEFAULT current_timestamp() COMMENT 'When was it exposed',
  `is_active` tinyint(1) DEFAULT 0 COMMENT 'Is this offer still valid',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=11 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Transactions';
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `offer`
--

LOCK TABLES `offer` WRITE;
/*!40000 ALTER TABLE `offer` DISABLE KEYS */;
INSERT INTO `offer` VALUES (1,1,1,4492564.690000,0,0,3560.226016,'1978-04-16 05:11:17',0),(2,2,2,88065.115876,0,1,25673.801940,'2012-01-27 00:48:33',0),(3,3,3,57342.670793,1,1,5968955.094700,'1994-04-03 09:01:57',0),(4,4,4,0.841573,0,1,43.341753,'2016-01-27 13:50:13',1),(5,5,5,21273859.908703,0,1,1003.024800,'1976-04-11 21:39:31',0),(6,6,6,330151.950000,0,0,0.908693,'2005-12-14 22:57:53',1),(7,7,7,270663059.774970,1,0,105.000000,'1988-11-26 14:32:08',0),(8,8,8,242300815.081650,1,1,418.591556,'2009-01-04 11:59:22',0),(9,9,9,43.108410,1,0,175276231.966730,'1980-11-30 22:09:40',1),(10,10,10,35074063.000000,0,0,3151.711086,'2007-11-07 11:28:43',1);
/*!40000 ALTER TABLE `offer` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `price`
--

DROP TABLE IF EXISTS `price`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `price` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `item_id` bigint(20) unsigned DEFAULT NULL COMMENT 'Refers to the currency used as the base currency for this product',
  `currency_id` int(10) unsigned DEFAULT NULL COMMENT 'Refers to the currency used as the base currency for this product',
  `buy` decimal(16,6) NOT NULL DEFAULT 0.000000 COMMENT 'Purchase rate',
  `sell` decimal(16,6) NOT NULL DEFAULT 0.000000 COMMENT 'Selling rate',
  `ts` datetime DEFAULT current_timestamp() COMMENT 'The time at which the transaction at the last price was fixed',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=11 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Price change';
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `price`
--

LOCK TABLES `price` WRITE;
/*!40000 ALTER TABLE `price` DISABLE KEYS */;
INSERT INTO `price` VALUES (1,1,0,833143.663441,7083.640000,'1974-02-28 12:14:53'),(2,2,1,506349.000000,13.956034,'1989-08-25 08:13:42'),(3,3,2,0.000000,3.100000,'1977-01-24 01:27:13'),(4,4,3,30754203.895000,22564.976107,'2005-03-20 21:58:47'),(5,5,4,607602067.694800,692.700000,'1995-05-31 22:30:59'),(6,6,5,55.747000,131.327840,'2011-06-04 13:19:14'),(7,7,6,4047695.239467,9294.278000,'1991-08-02 18:28:19'),(8,8,7,2053.700000,42688565.419750,'1995-09-23 02:06:23'),(9,9,8,29.743971,5.057628,'2002-05-03 05:17:56'),(10,10,9,56795556.100000,3272.168110,'1994-03-20 04:59:16');
/*!40000 ALTER TABLE `price` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `report`
--

DROP TABLE IF EXISTS `report`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `report` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `trading_data` datetime DEFAULT current_timestamp() COMMENT 'Report date',
  `item_id` bigint(20) unsigned DEFAULT NULL COMMENT 'Refers to the currency used as the base currency for this product',
  `currency_id` int(10) unsigned DEFAULT NULL COMMENT 'Refers to the currency used as the base currency for this product',
  `first_price` decimal(16,6) DEFAULT NULL COMMENT 'Initial price',
  `last_price` decimal(16,6) DEFAULT NULL COMMENT 'Last price',
  `min_price` decimal(16,6) DEFAULT NULL COMMENT 'Minimum price',
  `max_price` decimal(16,6) DEFAULT NULL COMMENT 'Maximum price',
  `avg_price` decimal(16,6) DEFAULT NULL COMMENT 'Average price',
  `total_amount` decimal(16,6) DEFAULT NULL COMMENT 'The total amount paid for this product during the reporting period.',
  `quantity` decimal(16,6) DEFAULT NULL COMMENT 'The number of products sold during this reporting period.',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=11 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Report';
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `report`
--

LOCK TABLES `report` WRITE;
/*!40000 ALTER TABLE `report` DISABLE KEYS */;
INSERT INTO `report` VALUES (1,'2008-09-12 05:40:01',1,0,0.000000,0.000000,11284.100000,0.000000,2113910.285725,33.000000,0.000000),(2,'2005-08-05 11:49:42',2,1,7.124000,17.382430,2.353250,2569858.500000,15130.120000,0.000000,0.000000),(3,'1985-04-14 09:07:10',3,2,43490527.885671,1452.364336,26509.073543,135.213446,341326328.324550,33705.000000,45819518.090000),(4,'2018-02-14 00:52:19',4,3,649342.981000,3.424990,991.284828,4063.439875,1324038.780000,9267588.571890,133722.790635),(5,'2003-12-24 09:46:13',5,4,2413.800000,3.000000,170787253.391930,221895.722237,2782544.825000,196.304264,85386572.462400),(6,'2014-11-14 05:28:06',6,5,4.320848,0.000000,120899.185986,351820.630000,44.353438,0.000000,2229.320343),(7,'2000-11-12 21:49:52',7,6,69682369.400000,211162.000000,119454.460000,37212568.372855,2133133.264035,153722219.685720,5557523.960000),(8,'2021-07-08 17:06:56',8,7,341.300000,1861.743223,1.337996,49.590397,8522.873563,2269.709000,16999.627000),(9,'2003-12-11 22:56:40',9,8,20354369.167190,3144.140230,0.800314,305.165000,1.136590,215.070000,0.000000),(10,'1999-04-10 21:50:09',10,9,4736112.251411,1569.689600,52377.000310,492851.324047,0.000000,33864985.940000,68705694.321598);
/*!40000 ALTER TABLE `report` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `trade`
--

DROP TABLE IF EXISTS `trade`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `trade` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `item_id` bigint(20) unsigned DEFAULT NULL COMMENT 'Product link',
  `seller_id` bigint(20) unsigned DEFAULT NULL COMMENT 'Link to the trader',
  `buyer_id` bigint(20) unsigned DEFAULT NULL COMMENT 'Link to the trader',
  `quantity` decimal(16,6) DEFAULT NULL COMMENT 'Number of products',
  `unit_price` decimal(16,6) DEFAULT NULL COMMENT 'Price per unit',
  `description` text COLLATE utf8mb4_unicode_ci DEFAULT NULL COMMENT 'All additional information (for example, the number of shares issued) in text format.',
  `offer_id` bigint(20) unsigned DEFAULT NULL COMMENT 'Transaction Identifier',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=11 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Transactions';
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `trade`
--

LOCK TABLES `trade` WRITE;
/*!40000 ALTER TABLE `trade` DISABLE KEYS */;
INSERT INTO `trade` VALUES (1,1,1,1,2.460000,58902033.493045,'Dolores officiis quas necessitatibus qui amet. Id error atque laborum ea maiores rerum voluptatem eum. Et tempora pariatur quaerat laborum. Placeat ipsa tenetur maiores architecto.',1),(2,2,2,2,28389.028300,64.144119,'Dolorem perspiciatis consequatur eaque est corporis adipisci. Quae numquam quo provident alias natus eligendi. Voluptas ratione velit eveniet aperiam.',2),(3,3,3,3,267929.007257,3002.000000,'Distinctio facere ullam nostrum commodi. Mollitia ea quo aut labore eaque. Aliquam dolores porro ut magni vitae dolore a nostrum.',3),(4,4,4,4,9942452.040000,572582523.427500,'Cumque fugit similique culpa et ad. Temporibus dolores ut sint ipsum voluptas a et. Dolore et totam tempora dolorem ea. Unde dolore velit perspiciatis exercitationem.',4),(5,5,5,5,679040271.000000,73901.498440,'Totam quisquam unde possimus voluptatibus quos praesentium provident. Et magnam aspernatur aut cupiditate eaque et non. Minima ex ut unde ut magni.',5),(6,6,6,6,11415.933040,34761.863528,'Necessitatibus laudantium fugit porro. A et reprehenderit iusto ut deleniti ut. Et inventore aut culpa quibusdam eveniet aut nulla.',6),(7,7,7,7,4.820000,77.000000,'Corporis provident natus neque et. Libero nostrum aut minima impedit. Libero quos et beatae eos. Facere et officia quis et quia.',7),(8,8,8,8,667543.999204,422918.783910,'Autem rerum est nostrum placeat nulla. Velit cumque eius a enim. Dolor harum dignissimos sunt ea. Consectetur rerum eius ratione rerum nisi ipsam perspiciatis.',8),(9,9,9,9,280068485.000000,0.000000,'Voluptatem veritatis consectetur libero autem reiciendis. Animi qui aut animi sed perspiciatis. Reprehenderit officia iure occaecati molestiae ipsam ipsa. Quo qui laudantium provident veniam aut.',9),(10,10,10,10,0.000000,107806.940000,'Ipsa ut veniam deserunt quis. Ipsam labore aspernatur cupiditate et molestiae. Animi voluptatem vel reprehenderit tempora. At doloribus facere voluptatem dignissimos dolorem.',10);
/*!40000 ALTER TABLE `trade` ENABLE KEYS */;
UNLOCK TABLES;

--
-- Table structure for table `trader`
--

DROP TABLE IF EXISTS `trader`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `trader` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `firstname` varchar(50) COLLATE utf8mb4_unicode_ci DEFAULT NULL COMMENT 'Name',
  `lastname` varchar(50) COLLATE utf8mb4_unicode_ci DEFAULT NULL COMMENT 'Surname',
  `user_name` varchar(50) COLLATE utf8mb4_unicode_ci NOT NULL COMMENT 'Everyone's login is unique',
  `email` varchar(120) COLLATE utf8mb4_unicode_ci NOT NULL,
  `confirmation_code` varchar(120) COLLATE utf8mb4_unicode_ci NOT NULL COMMENT 'The code sent to the user to complete the registration process.',
  `time_registered` datetime DEFAULT current_timestamp(),
  `time_confirmed` datetime DEFAULT current_timestamp(),
  `country_id` bigint(20) unsigned DEFAULT NULL COMMENT 'The country in which he lives.',
  `preffered_currency_id` bigint(20) unsigned DEFAULT NULL COMMENT 'The currency that the trader prefers',
  PRIMARY KEY (`id`),
  UNIQUE KEY `user_name` (`user_name`),
  UNIQUE KEY `email` (`email`)
) ENGINE=InnoDB AUTO_INCREMENT=11 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='users';
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `trader`
--

LOCK TABLES `trader` WRITE;
/*!40000 ALTER TABLE `trader` DISABLE KEYS */;
INSERT INTO `trader` VALUES (1,'Gideon','Hettinger','nweissnat','hilda.schulist@example.com','sdaj','1994-07-02 14:22:41','1973-09-17 17:51:44',1,0),(2,'Antwon','Rogahn','eva.stoltenberg','grussel@example.org','cddo','1979-12-08 07:45:35','1990-02-28 17:41:55',2,1),(3,'Letitia','Cremin','bergnaum.tyra','berniece.feeney@example.com','jrip','2005-05-27 20:49:06','1990-05-07 05:33:38',3,2),(4,'Braulio','Hessel','andreanne97','margarita71@example.net','zqnt','2006-01-12 00:24:22','2021-04-16 22:24:14',4,3),(5,'Clementine','Zboncak','orion53','sawayn.keara@example.net','tkwo','2004-09-08 13:05:57','2009-04-21 11:36:19',5,4),(6,'Alvis','Gutkowski','ikovacek','ikovacek@example.net','supc','2009-10-11 20:46:23','2003-04-14 08:48:12',6,5),(7,'Raphael','Sanford','mcdermott.providenci','leonie94@example.org','ibjn','1975-03-27 19:01:54','2000-12-20 15:01:59',7,6),(8,'Ruthie','Dietrich','lyost','powlowski.hillary@example.org','mdzm','2005-02-19 04:07:58','1993-08-30 12:18:06',8,7),(9,'Keagan','Gutmann','kpredovic','kody.gibson@example.com','ejxu','2001-09-05 00:29:11','2016-07-10 15:52:51',9,8),(10,'Marianne','Ziemann','jace.kunde','qbergnaum@example.com','qozb','1991-04-13 01:13:57','1987-11-14 13:35:07',10,9);
/*!40000 ALTER TABLE `trader` ENABLE KEYS */;
UNLOCK TABLES;
/*!40103 SET TIME_ZONE=@OLD_TIME_ZONE */;

/*!40101 SET SQL_MODE=@OLD_SQL_MODE */;
/*!40014 SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS */;
/*!40014 SET UNIQUE_CHECKS=@OLD_UNIQUE_CHECKS */;
/*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */;
/*!40101 SET CHARACTER_SET_RESULTS=@OLD_CHARACTER_SET_RESULTS */;
/*!40101 SET COLLATION_CONNECTION=@OLD_COLLATION_CONNECTION */;
/*!40111 SET SQL_NOTES=@OLD_SQL_NOTES */;

-- Dump completed on 2021-09-02 19:36:59

UPDATE trader SET country_id = (FLOOR(1 + RAND() * 3));
UPDATE offer SET item_id = (FLOOR(1 + RAND() * 3));
UPDATE offer SET trader_id = (FLOOR(1 + RAND() * 3));
UPDATE offer SET price = (FLOOR(1 + RAND() * 100000));
UPDATE offer SET buy = 0 WHERE sell = 1;
UPDATE offer SET buy = 1 WHERE sell = 0;
UPDATE trade SET seller_id = (FLOOR(1 + RAND() * 3));
UPDATE trade SET buyer_id = (FLOOR(1 + RAND() * 3));
UPDATE trade SET buyer_id = (FLOOR(1 + RAND() * 3)) WHERE buyer_id = seller_id ;
UPDATE trade SET quantity = (FLOOR(1 + RAND() * 1000000));
UPDATE trade SET unit_price = (FLOOR(1 + RAND() * 1000));
UPDATE trade SET item_id = (FLOOR(1 + RAND() * 3));
UPDATE trade SET offer_id = (FLOOR(1 + RAND() * 3));
UPDATE report SET currency_id = (FLOOR(1 + RAND() * 3));
UPDATE trader SET birthday = CURRENT_DATE() - INTERVAL (FLOOR(20 + RAND() * 60)) YEAR;
~~~~

### 6. Scripts of characteristic samples

Of course, I created the first samples from my logic, but of course I need a lot more samples

Distribution of traders by country

~~~~sql
CREATE VIEW trader_country AS
SELECT c.country, COUNT(*) AS number_people FROM trader t JOIN country c ON t.country_id = c.id GROUP BY c.country ;
~~~~

Distribution of the number of purchase offers by traders

~~~~sql
DROP VIEW trader_offer_buy;
CREATE VIEW trader_offer_buy AS
SELECT user_name,  COUNT(o.buy) AS sum_offer_buy FROM trader t JOIN offer o ON t.id = o.trader_id WHERE o.buy = 1 GROUP BY t.user_name ORDER BY sum_offer_buy DESC;
~~~~

Distribution of the number of offers for sale by traders

~~~~sql
DROP VIEW trader_offer_sell;
CREATE VIEW trader_offer_sell AS
SELECT user_name,  COUNT(o.sell) AS sum_offer_sell FROM trader t JOIN offer o ON t.id = o.trader_id WHERE o.sell = 1 GROUP BY t.user_name ORDER BY sum_offer_sell DESC;

SELECT * FROM trader_country tc ;

SELECT * FROM trader_offer_sell ;

SELECT * FROM trader_offer_buy tob  ;
~~~~

### 7. Creating Triggers

Trigger to check that age is not an empty value

~~~~sql
DROP TRIGGER IF EXISTS trader_age;

DELIMITER //

CREATE TRIGGER trader_age BEFORE UPDATE ON trader
FOR EACH ROW
BEGIN
	IF NEW.birthday IS NULL THEN
		SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'It is necessary to add the date of birth';
    END IF;
END//

DELIMITER ;
~~~~

Trigger to verify that the trader is of legal age

~~~~sql
DROP TRIGGER IF EXISTS trader_age;

DELIMITER //

CREATE TRIGGER trader_age BEFORE UPDATE ON trader
FOR EACH ROW
BEGIN
    IF NEW.birthday >= CURRENT_DATE() - INTERVAL 18 YEAR THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Wait until you turn 18';
    END IF;
END//

DELIMITER ;
~~~~

Trigger to check that a trader does not simultaneously put the same position on sale and purchase at once

~~~~sql
DROP TRIGGER IF EXISTS trader_age;

DELIMITER //

CREATE TRIGGER trader_age BEFORE UPDATE ON trader
FOR EACH ROW
BEGIN
    IF NEW.sell = 1 AND NEW.buy = 1 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'You cannot make a purchase and a sale at the same time';
    END IF;
END//

DELIMITER ;
~~~~

Trigger to check that the trader has chosen either buy or sell

~~~~sql
DROP TRIGGER IF EXISTS trader_age;

DELIMITER //

CREATE TRIGGER trader_age BEFORE UPDATE ON trader
FOR EACH ROW
BEGIN
    IF NEW.sell = 0 AND NEW.buy = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'It is necessary to choose one of the operations';
    END IF;
END//

DELIMITER ;
~~~~
That's the whole code. Rated excellent. At the end, a quote from the teacher:

> It's good that the indexes were taken care of.
ENGINE = InnoDB - default value. it is not necessary to write it. but it's good that you know about this option.
In SQL, it is generally accepted to name the fields of tables in the singular, and tables can be called in the plural. it is important to adhere to the chosen style (all in units or all in many hours).
It makes sense to save the most popular queries (often executed) in the form of representations.
It's good that views and triggers have been implemented, not everyone reaches this point.
Good luck in further training!
