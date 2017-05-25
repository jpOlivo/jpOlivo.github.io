---
layout: post
title:  "BigData sobre Datos Abiertos"
date:   2017-05-25 09:53:00
categories: BigData
---

En 2016, mientras cursaba la especializacion en Arquitectura de Software, tuve la oportunidad de involucrarme con estos dos nuevos paradigmas desde el punto de vista practico. Hasta ese momento, conocia acerca de los conceptos, tecnologias y tecnicas que gobiernan estos paradigmas, pero salvo algun *Proof of Concept* sobre alguna tecnologia en particular, nunca habia trabajado con ninguna de ellas bajo un fin practico.

En esta oportunidad tuve que dise&ntilde;ar y desarrollar una solucion haciendo uso de fuentes de datos masivas pertenecientes a algun dominio de especial interes (Geociencia, Redes Sociales, Economia, e-Commerce, Medicina, Medio Ambiente y Clima, Fisica, Astronomia, Quimica, Seguridad, Politica, etc.)

El dominio elegido fue la alimentacion, y el desarrollo consistio en crear una aplicacion que permitiese descubrir patrones, habitos y comportamientos acerca de como se alimentan las personas en diferentes poblaciones, con el objetivo de concientizar acerca de la importancia de la alimentacion, y que contribuya a la elaboracion de programas y politicas (por parte de los paises y/o organismos) que permitan abordar problemas de malnutricion y mejorar la salud de las personas.

El trabajo esta basado en [OpenFoodFacts](https://world.openfoodfacts.org/), una base de datos abierta NoSQL que crece dia a dia y que alberga la informacion de productos alimenticios distribuidos por todo el mundo, y en el [Toolbox to the Data Analyst](http://arjon.es/2015/08/23/vagrant-spark-zeppelin-a-toolbox-to-the-data-analyst/) desarrollado por @arjones. 

Conociendo la informacion nutricional de cada alimento y conociendo donde ese alimento es consumido, es posible conocer que tipo de dieta (basada en vegetales, basada en carnes, etc.) predomina en cada pais, los lugares con mayor indice de consumo de sal, azucar, grasas, etc.

El procesamiento de los datos se realizo a traves de Apache Spark. Primeramente, los datos persistidos en una base de datos MongoDB fueron cargados en el contexto de Spark (a traves de la libreria [Spark-MongoDB](https://github.com/Stratio/Spark-MongoDB) desarrollada por Stratio) y representados por medio del concepto de DataFrames 

~~~~~~
## load the lib third-party for access to mongoDB
z.load("com.stratio.datasource:spark-mongodb_2.10:0.9.2")

import org.apache.spark.sql.SQLContext._
import org.apache.spark.sql._

## load the data to a spark dataframe
val options = Map("host" -> "10.0.2.2:27017", "database" -> "openfoodfacts", "collection" -> "products")
val df = sqlContext.read.format("com.stratio.datasource.mongodb").options(options).load

~~~~~~~~~~~~

Un DataFrame es una coleccion distribuida de datos organizados en columnas (conceptualmente equivalente a una tabla en el modelo relacional) y que admite operaciones SQL sobre los datos.

![spark-dataframe](/assets/images/spark-dataframe.png)

La manipulacion de los DataFrames por medio de Spark permitio acotar la informacion y crear un subconjunto de datos reducido para ser consultado a traves de SparkSQL.

~~~~~~
...
case class Product(product:String, countriesSold:String, sugar100g:String, sodium100g:String, transFat:String, category:String, nutritionScore:String)

val products = productsText.map(line => line.split("\t")).filter(s=>s(0)!="code").filter(_.size == 159).filter(s=>s(102)!="" & s(117)!="").flatMap(s => s(33).split(",").map(part => Product(s(7), part, s(102), s(117), s(99), s(60), s(158)))) 

products.toDF().registerTempTable("products")
~~~~~~~~~~~~


El interprete SparkSQL provisto por Apache Zeppelin, permite acceder al contexto de Spark y con ello a los DataFrames que residen en el. De esta manera es posible consultar la informacion a traves de operaciones SQL como de si de una tabla relacional se tratara.

![average-salt-bycountry](/assets/images/avg-salt-bycountry.png)


El resultado de las consultas anteriores se puede visualizar a traves de las vistas *built-in* (grafico de barras, grafico de dispersion, representacion tabular, etc.) provistas por Apache Zeppelin.

![how-much-salt-eat](/assets/images/how-much-salt-eat.png)

Este trabajo es tan solo un ejemplo de como podemos integrar algunas herramientas y tecnologias de BigData sobre fuentes de datos abiertas para desarrollar soluciones innovadoras que pongan al descubierto nuevo conocimiento en diferentes campos o escenarios.

Una copia completa del trabajo puede descargarse en el siguiente [link](https://www.slideshare.net/JuanPabloOlivera1/como-puede-ayudar-el-big-data-al-sector-alimentario)

