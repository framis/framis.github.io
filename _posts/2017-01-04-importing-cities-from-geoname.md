---
layout: post
title:  "Importing world cities from Geoname dataset"
date:   2016-12-26 08:00:00 -0700
categories: geoname
---

[Geoname](http://www.geonames.org/) is an open-source dataset containing more than 11 million places, including country informations.

In this article, we will see how we can leverage this dataset to get the list of the cities in the world merged with some additionnal data, such as the postalCodes. 

<!--more-->

# The problem

The data we are going to import is located into different files:

* allCountries.zip -> that unzips to allCountries.txt
* or XX.zip where zip is the country code, ex: US.zip

The places are in CSV format but it lack most useful information. The country is represented by its two letter code. The administrative information (state, region, etc) are represented by a code. The postalCode information is missing.

Therefore, we will need to get relevant data from other files and perform relevant joins:

* http://download.geonames.org/export/dump/countryInfo.txt for countries
* http://download.geonames.org/export/dump/admin1CodesASCII.txt for Administrative 1 (state or region depending on the country)
* http://download.geonames.org/export/dump/admin2Codes.txt for Administrative 2 info (county in the US)
* http://download.geonames.org/export/zip/allCountries.zip for postalCode info

## Simple approach with a Shell script

Most people who are dealing with Geonames data use a simple shell script and save data in a RDBMS such as MySQL or PostgreSQL (useful if your are using postgis plugin for geolocation).

A good source of information is:
http://codigofuerte.github.io/GeoNames-MySQL-DataImport/

To download the files, the use of wget over curl is preferred because it can resume the download if failed. In order to make this approach more automatic, you can easily Dockerize the script as well as the database.

Once the data is imported into your local database, you can use your favorite client and start playing with the data, doing your joins and export the materialized view containing just cities.

The main advantage of this approach is its simplicity.

The main drawback is that you are tied to have a RDBMS for performing your data-processing, which is not very scalable. Moreover, you will need to build complex joins and your SQL queries will get inefficient and complex.

That's why most people actually use a programming language to perform the data-processing such as Foursquare's [twofishes](https://github.com/foursquare/fsqio/tree/master/src/jvm/io/fsq/twofishes) in scala

## Data Processing with Go

I could have used Python or Scala but I chose Golang. I am pretty fond of Communicating Sequential Processes framework popularized by Hoare (more info [book here](http://usingcsp.com/cspbook.pdf), [research article here](http://fi.ort.edu.uy/innovaportal/file/20124/1/55-hoare_csp_old_article.pdf)). Golang channels and goroutines makes it easy to implement.

My approach is the following:

* download parse the CSV files and load into memory small datasets useful for joins: countryInfo, admin1CodesASCII, admin2Codes and store the data in a Map where the key is the field used for joining the data (2 letter ISO code for country, and code for admin1CodesASCII and admin2Codes).
* download and unzip the geoname file (316M and unzipped more than 1G)
* process the file line by line in a streaming fashing using channels. Each line is sent a channel.
* the line is parsed from csv to a slice
* the slice is mapped to a struct
* the struct is validated and filtered to keep cities only (there is a 'P' class field, doc [here](http://www.geonames.org/export/codes.html))
* because there are still many invalid places we do further processing: we keep only places with a population > 0 (we realized that the population field is reliably > 0 for actual cities), and we use hierarchy.zip to fix a few more duplicates (some cities are multiple times in the file, for instance Marseille and its arrondissement in France)
* the City struct is enriched with country, Administrative1 and 2 and postalCode data
* the City can undergo further processing if needed, for instance it can be crossed checked manually or with another data-source
* the City is indexed to a database, in our case we send the data to Algolia search engine, but we could use ElasticSearch instead.
* send the data-processing errors to another channel for logging or ad-hoc resolution

You can find a quick and dirty version of the code at [framis/gocity](https://github.com/framis/gocity)

A few remarks, I am downloading and unzipping the file with Go code. I could also use a shell script instead. However, doing everything in Go reduces the number of dependencies.

We would have benefitted a lot to get a datasource gzipped instead of zipped. We could have unzipped it in a streaming fashion without having to download and unzip the full file. The data-processing pipeline would have been much faster.

We use Goroutines to handle downloads and data-processing in parallel to avoid I/O wait.

This approach allow the better customization but we are pretty much reinventing the wheel if the use-case is simple. Our use-case is too complex for the shell scripting approach, but not compelx enough to require building a software form scratch. Therefore, the last approach leverage Spark framework that is really good to process data at large scale.

## Using Spark

As we saw before, the Golang approach is highly scalable but requires a decent number of LOC. I am not even including tests, which I should.

Spark framework is super good at importing data from various data-sources and performing distributed joins. This is exactly our use-case here.

Moreover, the Dataframe API offers a good high-level API that is much easier to understand that custom golang code. It also offers a good client for developping and testing the code as well as many hosting solutions for production use.

The code takes less than 140 lines, provided we download the files with a bash script.

```scala
import org.apache.spark.SparkContext
import org.apache.spark.SparkContext._
import org.apache.spark.SparkConf
import org.apache.spark.sql.SQLContext
import org.apache.spark.sql.functions.udf
import org.apache.spark.sql.types.{StructType, StructField, StringType, IntegerType, DateType, DoubleType, FloatType}

object SimpleApp {
  def main(args: Array[String]) {
    val conf = new SparkConf().setAppName("Geoname")
    val sc = new SparkContext(conf)

    val CITY_CODE = "P"
    val GEONAME_PATH = "download/allCountries.txt"
    val COUNTRY_PATH = "download/countryInfo.txt"
    val ADMIN_ONE_PATH = "download/admin1CodesASCII.txt"
    val ADMIN_TWO_PATH = "download/admin2Codes.txt"
    val POSTAL_CODE_PATH = "download/zip/allCountries.txt"

    val geonameSchema = StructType(Array(
        StructField("geonameid", IntegerType, false),
        StructField("name", StringType, false),
        StructField("asciiname", StringType, true),
        StructField("alternatenames", StringType, true),
        StructField("latitude", FloatType, true),
        StructField("longitude", FloatType, true),
        StructField("fclass", StringType, true),
        StructField("fcode", StringType, true),
        StructField("country", StringType, true),
        StructField("cc2", StringType, true),
        StructField("admin1", StringType, true),
        StructField("admin2", StringType, true),
        StructField("admin3", StringType, true),
        StructField("admin4", StringType, true),
        StructField("population", DoubleType, true), // Asia population overflows Integer
        StructField("elevation", IntegerType, true),
        StructField("gtopo30", IntegerType, true),
        StructField("timezone", StringType, true),
        StructField("moddate", DateType, true)))

    val adminSchema = StructType(Array(
        StructField("code", StringType, true),
        StructField("name", StringType, true)))

    val countrySchema = StructType(Array(
        StructField("iso_alpha2", StringType, false),
        StructField("iso_alpha3", StringType, true),
        StructField("iso_numeric", StringType, true),
        StructField("fips_code", StringType, true),
        StructField("name", StringType, true),
        StructField("capital", StringType, true),
        StructField("areainsqkm", StringType, true),
        StructField("population", StringType, true),
        StructField("continent", StringType, true),
        StructField("tld", StringType, true),
        StructField("currency", StringType, true),
        StructField("currencyName", StringType, true),
        StructField("phone", StringType, true),
        StructField("postalCodeFormat", StringType, true),
        StructField("postalCodeRegex", StringType, true),
        StructField("geonameId", StringType, true),
        StructField("languages", StringType, true),
        StructField("neighbours", StringType, true),
        StructField("equivalentFipsCode", StringType, true)))

    val postalCodeSchema = StructType(Array(
        StructField("country", StringType, false),
        StructField("postal_code", StringType, true),
        StructField("name", StringType, true),
        StructField("admin1_name", StringType, true),
        StructField("admin1_code", StringType, true),
        StructField("admin2_name", StringType, true),
        StructField("admin2_code", StringType, true),
        StructField("admin3_name", StringType, true),
        StructField("admin3_code", StringType, true),
        StructField("latitude", StringType, true),
        StructField("longitude", StringType, true),
        StructField("accuracy", IntegerType, true)))

    val getAdminOneCode = udf( (country: String, adminOne: String) => { country + "." + adminOne } )
    val getAdminTwoCode = udf( (country: String, adminOne: String, adminTwo: String) => { country + "." + adminOne + "." + adminTwo } )

    val sqlContext = new SQLContext(sc)
    import sqlContext.implicits._

    val geonames = sqlContext.read
        .option("header", false)
        .option("quote", "")
        .option("delimiter", "\t")
        .option("maxColumns", 22)
        .schema(geonameSchema)
        .csv(GEONAME_PATH)
        .filter($"fclass"===CITY_CODE)
        .filter($"population">0)
        .withColumn("adminOneCode", getAdminOneCode($"country", $"admin1"))
        .withColumn("adminTwoCode", getAdminTwoCode($"country", $"admin1", $"admin2"))

    val countries = sqlContext.read
        .option("header", false)
        .option("delimiter", "\t")
        .schema(countrySchema)
        .csv(COUNTRY_PATH)

    val adminOne = sqlContext.read
        .option("header", "false")
        .option("delimiter", "\t")
        .schema(adminSchema)
        .csv(ADMIN_ONE_PATH)

    val adminTwo = sqlContext.read
        .option("header", "false")
        .option("delimiter", "\t")
        .schema(adminSchema)
        .csv(ADMIN_TWO_PATH)

    val postalCodes = sqlContext.read
        .option("header", "false")
        .option("delimiter", "\t")
        .schema(postalCodeSchema)
        .csv(POSTAL_CODE_PATH)
        .dropDuplicates(Seq("name", "country", "admin1_code"))    

    geonames.createOrReplaceTempView("geonames")
    countries.createOrReplaceTempView("countries")
    adminOne.createOrReplaceTempView("adminOne")
    adminTwo.createOrReplaceTempView("adminTwo")
    postalCodes.createOrReplaceTempView("postalCodes")

    val cities = sqlContext.sql(
          "SELECT g.geonameid, g.name, g.alternatenames, g.latitude, g.longitude, g.population, c.name as country_name, g.country as country_code, a1.name as admin1_name, g.admin1 as admin1_code, a2.name as admin2_name, p.postal_code " +
          "FROM geonames g " + 
          "LEFT OUTER JOIN countries c ON g.country = c.iso_alpha2 " + 
          "LEFT OUTER JOIN adminOne a1 ON g.adminOneCode = a1.code " +
          "LEFT OUTER JOIN adminTwo a2 ON g.adminTwoCode = a2.code " +
          "LEFT OUTER JOIN postalCodes p ON g.name = p.name AND g.country = p.country AND g.admin1 = p.admin1_code")
        .dropDuplicates(Seq("geonameid"))
        .cache()

    cities.createOrReplaceTempView("cities")

    // DATA INTEGRITY TEST
    val frenchCities = cities.filter($"country_code"==="FR").cache()
    assert(frenchCities.count() > 33000 && frenchCities.count() < 36000, "French cities count should be between 33000 and 36000")

    sqlContext.sql("SELECT * FROM cities WHERE name='Paris'").show()

    sc.stop()
  }
}
```

You can of course add more data integrity tests. Another approach would be to use newer Dataset api instead of Dataframes, so that the data is typed and to avoid such lines `.dropDuplicates(Seq("name", "country", "admin1_code"))` with hard-coded column names.

The working app is available at [framis/geonames-spark](https://github.com/framis/geonames-spark)



