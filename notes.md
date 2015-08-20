# Create a REST API to visualize data with Elasticsearch and Ramses

This is a post to show how to leverage Elasticsearch for data visualization in a web application. It uses Ramses, a Pyramid-based web framework written in Python. Ramses allows programming beginners to generate production-ready REST APIs that leverage Elasticsearch for all endpoints to eliminate boilerplate and provide for advanced data manipulation. If you'd like to read a tutorial for beginners, please go here: https://realpython.com/blog/python/create-a-rest-api-in-minutes-with-pyramid-and-ramses/

You can see the completed code for this example here: https://github.com/chrstphrhrt/ramses-elasticsearch

## Datasets

Looking around for datasets to use for this example, I learned that the UN Development Programme publishes two interesting variations on the Human Development Index. The original HDI is what we hear about in media stories with titles like "Best Countries to Live in for Quality of Life." One interesting extension to the HDI is called the Inequality-adjusted Human Development Index (IHDI), which downgrades the normal HDI score of countries proportionally to their level of inequality in the distribution of life expectancy, education, and income. Another very interesting compliment to the HDI is called the Gender Inequality Index (GII), which also downgrades the the level of human development according to women's reproductive health, empowerment, and labour market participation. While the IHDI is a true extension to the HDI because it uses the same metrics, the GII is a compliment because the metrics are different. Both the IHDI and GII were developed using the same underlying theoretical framework, so they are nevertheless comparable.

I thought it would be interesting to find out if there is a correlation between gender inequality and overall inequality by comparing the two scores.

Disclaimer: these data are published by the UNDP and are missing all kinds of important metrics for other axes of oppression like race, language, religion, disability, and non-binary gender identities. If any experts, academics, statisticians or intersectional feminists want me to correct anything please get in touch by sending me an email to chris@brandicted.com.

## Setup

For this guide we need to satisfy five dependencies before diving in.

* We need to have Python installed on the system.
* We need to have Elasticsearch running with a default configuration.
* We need either PostgreSQL or MongoDB running with a default configuration.
* We need to have `virtualenv` installed.
* We need to have `npm` (the nodejs package manager) installed.

For this example, I have chosen Postgres, but Mongo works too.

Now, do the following in a terminal:

```
$ mkdir humandev-gender
$ cd humandev-gender
$ virtualenv venv
$ source venv/bin/activate
```

After that, install Ramses and generate a new blank project.

````
(venv)$ pip install ramses
(venv)$ pcreate -s ramses_starter humgen
<output>: Select '1' to use Postgres.
(venv)$ cd humgen/
(venv)$ pserve local.ini
```

Now you will see a server running. In a new terminal, see what a basic request and response looks like. I will use [httpie](https://github.com/jkbrzt/httpie) in this example.

```
$ http :6543/api/items
```

Yay! A real REST API that does nothing useful yet.

Now let's fetch the data we want to analyze so we can model it in the API.

```
$ cd humgen/
$ mkdir data
$ cd data
$ wget http://data.undp.org/api/views/n8fa-gx39/rows.json -O ihdi.json
$ wget http://data.undp.org/api/views/ku9i-8fxp/rows.json -O gii.json
```

We need to extract the schemas from these datasets so that our API will know the structure of the data we want to analyze. For this I found a nice little nodejs utility called `json-schema-generator`.

Let's delete the boilerplate `items.json` schema, create a directory to hold our new schemas, and then generate them from the datasets we downloaded.

```
$ cd ..
$ rm items.json
$ mkdir schemas
$ cd schemas/
$ npm install -g json-schema-generator
$ json-schema-generator ../data/ihdi.json -o ihdi.json
$ json-schema-generator ../data/gii.json -o gii.json
```
