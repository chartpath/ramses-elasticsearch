# Create a REST API to visualize data with Elasticsearch and Ramses

This is a post to show how to leverage Elasticsearch for data visualization in a web application. It uses Ramses, a Pyramid-based web framework written in Python. Ramses allows programming beginners to generate production-ready REST APIs that leverage Elasticsearch for all endpoints to eliminate boilerplate and provide for advanced data manipulation. If you'd like to read a tutorial for beginners, please go here: https://realpython.com/blog/python/create-a-rest-api-in-minutes-with-pyramid-and-ramses/

## The Data

Looking around for datasets to use for this example, I learned that the UN Development Programme publishes two interesting variations on the Human Development Index. The original HDI is what we hear about in media stories with titles like "Best Countries to Live in for Quality of Life." One interesting extension to the HDI is called the Inequality-adjusted Human Development Index (IHDI), which downgrades the normal HDI score of countries proportionally to their level of inequality in the distribution of life expectancy, education, and income. Another very interesting compliment to the HDI is called the Gender Inequality Index (GII), which also downgrades the the level of human development according to women's reproductive health, empowerment, and labour market participation. While the IHDI is a true extension to the HDI because it uses the same metrics, the GII is a compliment because the metrics are different. Both the IHDI and GII were developed using the same underlying theoretical framework, so they are nevertheless comparable.

I thought it would be interesting to find out if there is a correlation between gender inequality and overall inequality by comparing the two scores.

Disclaimer: these data are published by the UNDP and are missing all kinds of important metrics for other axes of oppression like race, language, religion, disability, and non-binary gender identities. If any experts, academics, statisticians or intersectional feminists want me to correct anything please get in touch by sending me an email to chris@brandicted.com.

## Setup

For this guide we need to satisfy four dependencies before diving in. First, we need to have Python installed on the system. Second, we need to have Elasticsearch running with a default configuration. Third, we need either PostgreSQL or MongoDB running with a default configuration. Fourth, we need to have `virtualenv` installed, which can be done with the command `sudo pip install virtualenv`.

For this example, I have chosen Postgres, but Mongo works too.

Now, try the following in a terminal:

```
$ mkdir humandev-gender
$ cd humandev-gender
$ virtualenv venv
$ source venv/bin/activate
```

After that,

````
(venv)$ pip install ramses
(venv)$ pcreate -s ramses_starter humgen
<output>: 1
(venv)$ cd humgen/
(venv)$ pserve local.ini
```

Now you will see a server running.

In a new terminal,

```



```
