Understanding the Effects of Gender Inequality on Global Human Development

This is a post to show how to leverage Elasticsearch for data visualization in a web application. It uses Ramses, a Pyramid-based web framework written in Python. Ramses allows programming beginners to generate production-ready REST APIs that leverage Elasticsearch for all endpoints to eliminate boilerplate and provide for advanced data manipulation. If you'd like to read a tutorial for beginners, please go here: https://realpython.com/blog/python/create-a-rest-api-in-minutes-with-pyramid-and-ramses/

Disclaimer about privilege: I'm a white cis male, and therefore I'm not an expert on issues related to race or gender oppression. I'm also a software developer, not a scientist. So in case I have written anything that experts disagree with on intersectionality or statistics, I automatically defer to them in advance and am extremely grateful if they are able to share comments publicly so that this post can be updated. Also, there are many limitations to the depth of the data presented, which can be seen on the data publisher's site, on Wikipedia, and in the academic literature, so those are all discounted for my purposes here.

I have recently begun looking around for open datasets that would be fun and meaningful to play with to show off my favourite database-slash-search-engine, Elastic. To my delight, I found that the UN Development Programme has some interesting stuff. They publish the well-known Human Development Index that everyone cites when threatening to move to Scandinavia every time our less-enlightened governments of the world drop the ball on basic decency.

In my search, I learned that it's not just the old HDI anymore. They also publish the IHDI, the Inequality-adjusted Human Development Index. What's better, they also publish the Gender Inequality Index. Both of these datasets are statistically tied-into the original HDI, which makes them comparable in at least one interesting way.

That is, how does global gender equality relate to overall global equality in human development?

To be more specific, how do the combination of average national inequalities in reproductive health, women's political empowerment, and women's workplace participation relate to the combination of general average national inequalities in health, education and income?

Let's find out!

For this guide we need to satisfy four dependencies before diving in. First, we need to have Python 3 installed on the system. Second, we need to have Elasticsearch running with a default configuration. Third, we need either PostgreSQL or MongoDB running with a default configuration. Fourth, we need to have `virtualenv` installed, which can be done with the command `sudo pip install virtualenv`.

For this example, I have chosen Postgres, but Mongo works too.

Now, try the following in a terminal:

```
$ mkdir humandev-gender
$ cd humandev-gender
$ virtualenv -p python3 venv
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