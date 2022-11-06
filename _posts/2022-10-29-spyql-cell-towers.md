---
layout: post
title: Command-line data analytics made easy 
date: 2022-11-01 00:00:00
description: With SQL Powered by Python 
categories: data-processing command-line python csv json

og_title: Command-line data analytics made easy
description: With SQL Powered by Python 
og_image: /assets/img/kepler_h3_cell_towers.png
og_type: website
---

The command-line is incredibly powerful when it comes to data processing. Still, many of us working with data do not take advantage of it. I can think of some reasons:

* **Poor readability**: the focus is on minimising how much you need to type and not so much on how readable a sequence of commands is;
* **Steep learning curve**: many commands, with many options;
* **Looks outdated:** some of these tools are around since the 70s and target delimited text files (and not modern formats like JSON and YAML).

These motivated me to write a command-line tool that focus on readability, easiness to learn and modern data formats, while leveraging the command-line ecosystem. On top of that, it also leverages the Python ecosystem! Meet [SPyQL - SQL with Python in the middle](https://github.com/dcmoura/spyql):

{% highlight sql %}SELECT 
  date.fromtimestamp(purchase_ts) AS purchase_date,
  sum_agg(price * quantity) AS total
FROM csv('my_purchases.csv')
WHERE department.upper() == 'IT' and purchase_ts is not Null
GROUP BY 1
ORDER BY 1
TO json
{% endhighlight %}


# SPyQL in Action

I think the best way for getting to know SPyQL and getting comfortable with the command-line is to open a terminal and solve a problem. In this case, we will try to understand the geographical distribution of cell towers. Let's start!

## Setup

Let's start by installing SPyQL:

```sh
$ pip3 install -U spyql
```

and check its version:

```sh
$ spyql --version
spyql, version 0.8.1
```

Let's also install [MatplotCLI](https://github.com/dcmoura/matplotcli), a utility for creating plots from the command-line that leverages [Matplotlib](https://matplotlib.org):

```sh
$ pip3 install -U matplotcli
```

Finally, we will download some sample data (you can alternatively copy-paste the URL to your browser and download the file from there):

```sh
$ wget https://raw.githubusercontent.com/dcmoura/blogposts/master/spyql_cell_towers/sample.csv
```

This CSV file contains data about cell towers that were added to the [OpenCellid](https://www.opencellid.org/) database on 2022 September 10 (`OCID-diff-cell-export-2022-09-10-T000000.csv` file from the [OpenCellid](https://www.opencellid.org/) project redistributed without modifications under the [Creative Commons Attribution-ShareAlike 4.0 International License](https://creativecommons.org/licenses/by-sa/4.0/)).

## Inspecting the data

Let's look to the data by getting the first 3 lines of the file:

```sh
$ head -3 sample.csv
radio,mcc,net,area,cell,unit,lon,lat,range,samples,changeable,created,updated,averageSignal
GSM,262,2,852,2521,0,10.948628,50.170324,15762,200,1,1294561074,1662692508,0
GSM,262,2,852,2501,0,10.940241,50.174076,10591,200,1,1294561074,1662692508,0
```

You could do this same operation with SPyQL:

```sh
$ spyql "SELECT * FROM csv LIMIT 2" < sample.csv
```

or

```sh
$ spyql "SELECT * FROM csv('sample.csv') LIMIT 2"
```

Notice that we are telling to get 2 rows of data and not 3 rows of the file (where the first is the header). 

One advantage of SPyQL is that we can change the output format easily. Let's change the output to JSON and look to the first record:

```sh
$ spyql "SELECT * FROM csv('sample.csv') LIMIT 1 TO json(indent=2)"
{
  "radio": "GSM",
  "mcc": 262,
  "net": 2,
  "area": 852,
  "cell": 2521,
  "unit": 0,
  "lon": 10.948628,
  "lat": 50.170324,
  "range": 15762,
  "samples": 200,
  "changeable": 1,
  "created": 1294561074,
  "updated": 1662692508,
  "averageSignal": 0
}
```

## Querying the data

Let's first count how many records we have:

```sh
$ spyql "SELECT count_agg(*) AS n FROM csv('sample.csv')"
n
45745
```

Notice that aggregation functions have the suffix `_agg` to avoid conflicts with Python's functions like `min`, `max` and `sum`.
Now, let's count how many cell towers we have by radio type:

```sh
$ spyql "SELECT radio, count_agg(*) AS n FROM csv('sample.csv') GROUP BY 1 
ORDER BY 2 DESC TO pretty"
radio        n
-------  -----
GSM      31549
LTE      12996
UMTS      1182
CDMA        16
NR           2
```

Notice the pretty printing output. We can plot the above results by setting the output format to JSON and piping results into [MatplotCLI](https://github.com/dcmoura/matplotcli):

```sh
$ spyql "SELECT radio, count_agg(*) AS n FROM csv('sample.csv') GROUP BY 1 
ORDER BY 2 DESC TO json" | plt "bar(radio, n)"
```


<div class="row">
	<div class="col-2 mt-3 mt-md-0"/>
	<div class="col-8 mt-3 mt-md-0">
		{% include figure.html path="assets/img/matplotcli_count_by_type.png" title="Plot with MatplotCLI" class="img-fluid rounded z-depth-1" %} 
	</div>
	<div class="col-2 mt-3 mt-md-0"/>
</div>
<div class="caption">
Matplolib plot created by MatplotCLI using the output of a SPyQL query
</div>

How easy was that? **:-)**

## Querying the data: a more complex example

Now, let's get the top 5 countries with more cell towers added on that day:

```sh
$ spyql "SELECT mcc, count_agg(*) AS n FROM csv('sample.csv') GROUP BY 1 ORDER BY 2 DESC LIMIT 5 TO pretty"
  mcc      n
-----  -----
  262  24979
  440   5085
  208   4573
  310   2084
  311    799
```

MCC stands for Mobile Country Code (262 is the code for Germany). The first digit of the MCC identifies the region. Here's an exert from [Wikipedia](https://en.wikipedia.org/wiki/Mobile_country_code):

```
0: Test networks
2: Europe
3: North America and the Caribbean
4: Asia and the Middle East
5: Australia and Oceania
6: Africa
7: South and Central America
9: Worldwide
```

Let's copy paste the above list of regions and create a new file named `mcc_geo.txt`. On the mac this is as easy as `$ pbpaste > mcc_geo.txt`, but you can also paste this into a text editor and save it. 

Now, let's ask SPyQL to open this file as a CSV and print its contents:

```sh
$ spyql "SELECT * FROM csv('mcc_geo.txt') TO pretty"
  col1  col2
------  -------------------------------
     0  Test networks
     2  Europe
     3  North America and the Caribbean
     4  Asia and the Middle East
     5  Australia and Oceania
     6  Africa
     7  South and Central America
     9  Worldwide
```

SPyQL detected that the separator is a colon and that the file has no header. We will use the *colN* syntax to address the *Nth* column. 

Now, let's create a single JSON object with as many key-value pairs as input rows. Let the 1st column of the input be the *key* and the 2nd column be the *value*, and save the result to a new file:

```sh
$ spyql "SELECT dict_agg(col1,col2) AS json FROM csv('mcc_geo.txt') TO json('mcc_geo.json', indent=2)"
```

We can use `cat` to inspect the output file:

```sh
$ cat mcc_geo.json                                               
{
  "0": "Test networks",
  "2": "Europe",
  "3": "North America and the Caribbean",
  "4": "Asia and the Middle East",
  "5": "Australia and Oceania",
  "6": "Africa",
  "7": "South and Central America",
  "9": "Worldwide "
}
```

Basically, we have aggregated all input lines into a Python dictionary, and then saved it as a JSON file. Try removing the `AS json` alias from the `SELECT` to understand why we need it :-) 

Now, let's get the statistics by region instead of by MCC. For this, we will load the JSON file that we just created (with the `-J` option) and do a dictionary lookup:

```sh
$ spyql -Jgeo=mcc_geo.json "SELECT geo[mcc//100] AS region, count_agg(*) AS n 
FROM csv('sample.csv') GROUP BY 1 ORDER BY 2 DESC TO pretty"
region                               n
-------------------------------  -----
Europe                           35601
Asia and the Middle East          5621
North America and the Caribbean   3247
Australia and Oceania              894
Africa                             381
South and Central America            1
```

We do an integer division by 100 to get the 1st digit of the MCC and then we lookup this digit on the JSON we just created (which is loaded as a Python dictionary). This how we do a JOIN in SPyQL, via a dictionary lookup :-)


## Leveraging Python libs on your queries

Another advantage of SPyQL is that we can leverage the Python ecosystem. Let's try to do some more geographical statistics. Let's count towers by H3 cell (resolution 5) for Europe. First, we need to install the [H3 lib](https://github.com/uber/h3-py):

```sh
$ pip3 install -U h3
```

Then, we can convert latitude-longitude pairs into H3 cells, count how many towers we have by H3 cell, and save results into a CSV:

```sh
$ spyql "IMPORT h3 SELECT h3.geo_to_h3(lat, lon, 5) AS cell, count_agg(*) AS n 
FROM csv('sample.csv') WHERE mcc//100==2 GROUP BY 1 TO csv('towers_by_h3_res5.csv')"
```

Visualising these results is fairly simple with Kepler. Just go to [kepler.gl/demo](https://kepler.gl/demo) and open the above file. You should see something like this:

<div class="row">
	<div class="col-sm mt-3 mt-md-0">
		{% include figure.html path="assets/img/kepler_h3_cell_towers.png" title="Kepler visualization of aggregations by H3 cell (resolution 5) from SPyQL" class="img-fluid  rounded z-depth-1" %}
	</div>
</div>
<div class="caption">
Kepler visualization of aggregations by H3 cell (resolution 5) from SPyQL
</div>


# Final words

I hope you enjoyed SPyQL and that I could show you how simple it makes to query data from the command-line. In this first post about SPyQL, we are just scratching the surface. There is a lot more we can do. We have barely leveraged the Shell and Python ecosystems in this article. And we worked with a small file (SPyQL can handle GB-size files without compromising system resources). So, stay tuned!

Try out SPyQL and reach back to me with your thoughts. Thank you!

# Resources

* SPyQL repo: [github.com/dcmoura/spyql](https://github.com/dcmoura/spyql)

* SPyQL documentation: [spyql.readthedocs.io](https://spyql.readthedocs.io)

* MatplotCLI repo: [github.com/dcmoura/matplotcli](https://github.com/dcmoura/matplotcli)


---


SPyQL is open-source ([MIT license](https://github.com/dcmoura/spyql/blob/master/LICENSE)).
If you like the project, please consider giving it a [star](https://github.com/dcmoura/spyql). I celebrate every star and every fork the project gets :-). All kind of contributions are welcomed, just reach out to me before you start coding. Thank you!
