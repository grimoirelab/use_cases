Creating a simple Kibana dashboard for Discourse
================================================

Simple instructions to set up a Kibana dashboard using p2o tool from [GrimoireELK](https://github.com/grimoirelab/grimoireelk)

Install the tools
-----------------

**Reminder**: Grimoire tools work with Python3.x

**Install perceval**

Since version 0.3.1 you can install it using `pip`
```
$ sudo pip install perceval
```

Or you can

```
$ git clone https://github.com/grimoirelab/perceval.git
```

And follow [Perceval instructions](https://github.com/grimoirelab/perceval) to install it

**Install GrimoireELK**

Just:
```
$ git clone https://github.com/grimoirelab/grimoireelk.git
```

**Install and run ElasticSearch and Kibana**

We will be using `.zip` files for:
* [ElasticSearch 2.4.1](https://www.elastic.co/downloads/elasticsearch)
* [Kibana 4.6.1](https://www.elastic.co/downloads/kibana)

Running them is as simple as:
```
$ ./elasticsearch-2.4.1/bin/elasticsearch
$ ./kibana-4.6.1-linux-x86_64/bin/kibana
```

**Get git data and produce ES index**

For example, for Mozilla Hispano Discourse instance:
```
~/GrimoireELK/utils$ python3 p2o.py --enrich --index discourse_moz_hispano -e http://localhost:9200 --no_inc --debug discourse 'https://foro.mozilla-hispano.org/'
```

It will gather data and create a raw index and then enrich it to produce a new index in your local ES called `discourse_moz_hispano_enrich`

**Visualize git data**

Once `p2o` finishes, open your local Kibana instance at `http://localhost:5601`

Add `discourse_moz_hispano_enrich` index pattern, and set `girmoire_creation_date` as `Time field name`.

**Play with the data**

Now you can play with the data using Kibana.

**Using dashboard templates**

If you wanna try a pre-configured dashboard, download [this .json template](discourse-template-dashboard.json).

Open it in a text editor and search and replace `template` by whatever `{name}` you have used for your `discourse_{name}_enrich` index you have created..

Save it as a new `.json` file, and import the file in Kibana using the `Import` option in the Settings/Objects tab

This will create a new set of visualizations and dashboard parametrized for the index you have created:
![Discourse dashboard](discourse-dashboard.jpg)

Play with it, break it, improve it ...
