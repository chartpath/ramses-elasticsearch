# Make an Elasticsearch-powered REST API for any data with Ramses

In this short guide, I'm going to show you how to download a dataset and create a REST API. This new API uses Elasticsearch to power the endpoints, so you can build a product around your data without having to directly expose Elasticsearch in production. This allows for proper authentication, authorization, custom logic, other databases, and even auto-generated client libraries.

[Ramses](https://github.com/brandicted/ramses) is a bit like [Parse](https://parse.com/) for open source. It provides the convenience of a "backend as a service", except that you run the server yourself and have full access to the internals.

<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/2/2a/Igualtat_de_sexes.svg/2000px-Igualtat_de_sexes.svg.png" alt="Gender Equality" width=200 align=right hspace=30 />I recently came across the Gender Inequality Index, an interesting dataset published by the UN Development Programme. This dataset is a twist on the classic Human Development Index. The HDI ranks countries based on their levels of lifespan, education and income. The GII, on the other hand, ranks countries based on how they stack up in terms of gender (in)equality. The metrics in the GII are a combination of women's reproductive health, social empowerment, and labour force participation. Unfortuately, this dataset is missing non-binary gender identities, so our exploration will be a bit limited until that information is added. You can [read more about the dataset](http://hdr.undp.org/en/content/gender-inequality-index-gii) from the United Nations Development Programme. 

This dataset enables us to dig into some really interesting questions. Let's make a REST API out of the GII and then query it in fun ways using the Elasticsearch query DSL via the endpoint URL.

You can jump ahead to see [the completed code for this example here] (https://github.com/chrstphrhrt/ramses-elasticsearch).

## Set up the project

Before we dive in, make sure these pieces are in place:

* Recent versions of Elasticsearch and PostgreSQL installed and running in the background with default configurations.
* Python 2.7, 3.3 or 3.4 and [virtualenv](https://virtualenv.pypa.io/en/latest/)
* A CLI HTTP client (I'm using [httpie](https://github.com/jkbrzt/httpie) but you can use curl if you prefer)

Open a terminal, install Ramses, and create your new project:

```bash
$ mkdir gii_project && cd gii_project
$ virtualenv venv
$ source venv/bin/activate
(venv)$ pip install ramses
(venv)$ pcreate -s ramses_starter gii_api
```

Choose Postgres (option 1) as your database and open the new project in a text editor to look around. Then, start the server to make sure it works. It should look something like this:

```bash
(venv)$ cd gii_api
(venv)$ pserve local.ini
...
Starting server in PID 40098.
serving on http://0.0.0.0:6543
```

## Model and post the data

There are two main files in the boilerplate project right now: `api.raml`, and `items.json`. `api.raml` is a [RAML](http://raml.org/) file, which is a DSL for describing REST APIs in YAML. It configures your endpoints.  `items.json` is the schema that describes the fake boilerplate "Item" model.

We are going to replace these files with real ones based on the GII data.

First, download the data to the root of the project (`gii_api/`), and rename the old `items.json` to  `gii_schema.json`.

```bash
$ wget https://raw.githubusercontent.com/chrstphrhrt/ramses-elasticsearch/master/gii_api/gii_data.json
...
2015-08-31 15:58:34 (198 KB/s) - 'gii_data.json' saved [75659]
$ mv items.json gii_schema.json
```

**Note**: you can also get the data directly from the [UNDP site](http://hdr.undp.org/en/content/gender-inequality-index-gii). I cleaned it up a little for this guide.

Now edit the `gii_schema.json` file to describe the fields we see in the raw data. Look in `gii_data.json` for the field names and types.

Here's the first record for example:

```json
{
  "labour_force_participation_rate_aged_15_and_above_male_2012" : "69.5",
  "hdi_rank" : "1",
  "gender_inequality_index_value_2013" : "0.068",
  "gender_inequality_index_rank_2013" : "9",
  "population_with_some_secondary_ed_aged_25_and_up_fem_2005_2012" : "97.4",
  "country" : "Norway",
  "_2010_2015_adolescent_birth_rate_births_per_1_000_women_aged_15_19" : "7.8",
  "labour_force_participation_rate_aged_15_and_above_female_2012" : "61.5",
  "_2013_share_of_seats_in_parliament_held_by_women" : "39.6",
  "population_with_some_secondary_ed_aged_25_and_up_male_2005_2012" : "96.7",
  "_2010_maternal_mortality_ratio_deaths_per_100_000_live_births" : "7"
}
```

Now look in `gii_schema.json`:

```json
{
    "type": "object",
    "title": "Item schema",
    "$schema": "http://json-schema.org/draft-04/schema",
    "required": ["id", "name"],
    "properties": {
        "id": {
            "type": ["integer", "null"],
            "_db_settings": {
                "type": "id_field",
                "required": true,
                "primary_key": true
            }
        },
        "name": {
            "type": "string",
            "_db_settings": {
                "type": "string",
                "required": true
            }
        },
        "description": {
            "type": ["string", "null"],
            "_db_settings": {
                "type": "text"
            }
        }
    }
}
```

We want to replace the "properties" section with the field names and types from our dataset. Update the `title` and `required` fields, and make the `country` field the primary key, like so:

```json
{
    "type": "object",
    "title": "GII Country schema",
    "$schema": "http://json-schema.org/draft-04/schema",
    "required": ["country"],
    "properties": {
        "labour_force_participation_rate_aged_15_and_above_male_2012": {
            "_db_settings": {
                "type": "float"
                }
        },
        "hdi_rank": {
            "_db_settings": {
                "type": "integer"
            }
        },
        "gender_inequality_index_value_2013": {
            "_db_settings": {
                "type": "float"
            }
        },
        "gender_inequality_index_rank_2013": {
            "_db_settings": {
                "type": "float"
            }
        },
        "population_with_some_secondary_ed_aged_25_and_up_fem_2005_2012": {
            "_db_settings": {
                "type": "float"
            }
        },
        "country": {
            "_db_settings": {
                "type": "string",
                "primary_key": true
            }
        },
        "_2010_2015_adolescent_birth_rate_births_per_1_000_women_aged_15_19": {
            "_db_settings": {
                "type": "float"
            }
        },
        "labour_force_participation_rate_aged_15_and_above_female_2012": {
            "_db_settings": {
                "type": "float"
            }
        },
        "_2013_share_of_seats_in_parliament_held_by_women": {
            "_db_settings": {
                "type": "float"
            }
        },
        "population_with_some_secondary_ed_aged_25_and_up_male_2005_2012": {
            "_db_settings": {
                "type": "float"
            }
        },
        "_2010_maternal_mortality_ratio_deaths_per_100_000_live_births": {
            "_db_settings": {
                "type": "integer"
            }
        }
    }
}
```

**Note:** the `_db_settings` is Ramses-specific and is used to tell the DB engine(s) how to configure themselves. There are a lot of fields that can be set in addition to the types. For a [full list](https://ramses.readthedocs.org/en/stable/fields.html), have a look at the docs. 

Now that we have data and a schema to describe it, let's hook up the endpoints. Open `api.raml` and change the boilerplate Item endpoint to use the GII countries schema instead.

Before:

```yaml
#%RAML 0.8
---
title: gii_api API
documentation:
    - title: gii_api REST API
      content: |
        Welcome to the gii_api API.
baseUri: http://{host}:{port}/{version}
version: v1
mediaType: application/json
protocols: [HTTP]

/items:
    displayName: Collection of items
    get:
        description: Get all item
    post:
        description: Create a new item
        body:
            application/json:
                schema: !include items.json

    /{id}:
        displayName: Collection-item
        get:
            description: Get a particular item
        delete:
            description: Delete a particular item
        patch:
            description: Update a particular item
```

And after:

```yaml
#%RAML 0.8
---
title: gii_api API
documentation:
    - title: gii_api REST API
      content: |
        Welcome to the gii_api API.
baseUri: http://{host}:{port}/{version}
version: v1
mediaType: application/json
protocols: [HTTP]

/gii_countries:
    displayName: Collection of GII countries
    get:
        description: Get all countries
    post:
        description: Create a new country
        body:
            application/json:
                schema: !include gii_schema.json

    /{country}:
        displayName: A GII country
        get:
            description: Get a particular country
        delete:
            description: Delete a particular country
        patch:
            description: Update a particular country
```

It's that simple!

Now we can drop the database, delete the Elasticsearch index, and restart the server.

```bash
(venv)$ dropdb gii_api
(venv)$ http DELETE :9200/gii_api
HTTP/1.1 200 OK
Content-Length: 21
Content-Type: application/json; charset=UTF-8

{
    "acknowledged": true
}
(venv)$ pserve local.ini
...
Starting server in PID 45998.
serving on http://0.0.0.0:6543
```

It's time to post all the data to the API so that we can start making queries. With the server already running, open a new terminal. I like to use our built-in script for this. Activate the virtual environment, and call the `post2api` script like so:

```bash
$ cd gii_project/
$ . venv/bin/activate
(venv)$ nefertari.post2api -f gii_api/gii_data.json -u http://localhost:6543/api/gii_countries
Posting: {"_2010_maternal_mortality_ratio_deaths_per_100_000_live_births": "7", "gender_inequality_index_rank_2013": "9", "gender_inequality_index_value_2013": "0.068", "country": "Norway", "population_with_some_secondary_ed_aged_25_and_up_fem_2005_2012": "97.4", "_2013_share_of_seats_in_parliament_held_by_women": "39.6", "_2010_2015_adolescent_birth_rate_births_per_1_000_women_aged_15_19": "7.8", "labour_force_participation_rate_aged_15_and_above_female_2012": "61.5", "labour_force_participation_rate_aged_15_and_above_male_2012": "69.5", "hdi_rank": "1", "population_with_some_secondary_ed_aged_25_and_up_male_2005_2012": "96.7"}
201
...
```

## Querying the data

If you want to get a complete picture of what you can do, go directly to [the reference documentation](https://nefertari.readthedocs.org/en/stable/making_requests.html). This covers all of the following techniques (and more). 

Now we can start poking around in the data to see what kinds of interesting facts we are able to extract with Ramses.

Here's the most basic request, to show data for a specific country:

```bash
$ http :6543/api/gii_countries/Norway
HTTP/1.1 200 OK
Cache-Control: max-age=0, must-revalidate, no-cache, no-store
Content-Length: 723
Content-Type: application/json; charset=UTF-8
Date: Tue, 01 Sep 2015 17:49:42 GMT
Expires: Tue, 01 Sep 2015 17:49:42 GMT
Last-Modified: Tue, 01 Sep 2015 17:49:42 GMT
Pragma: no-cache
Server: waitress

{
    "_2010_2015_adolescent_birth_rate_births_per_1_000_women_aged_15_19": 7.8,
    "_2010_maternal_mortality_ratio_deaths_per_100_000_live_births": 7,
    "_2013_share_of_seats_in_parliament_held_by_women": 39.6,
    "_pk": "Norway",
    "_self": "http://localhost:6543/api/gii_countries/Norway",
    "_type": "GiiCountry",
    "_version": 0,
    "country": "Norway",
    "gender_inequality_index_rank_2013": 9.0,
    "gender_inequality_index_value_2013": 0.068,
    "hdi_rank": 1,
    "labour_force_participation_rate_aged_15_and_above_female_2012": 61.5,
    "labour_force_participation_rate_aged_15_and_above_male_2012": 69.5,
    "population_with_some_secondary_ed_aged_25_and_up_fem_2005_2012": 97.4,
    "population_with_some_secondary_ed_aged_25_and_up_male_2005_2012": 96.7
}
```

### Limit and sort

If you make a GET request to the `/api/gii_countries` collection, you'll get the default limit of 20 records.

How about something more interesting, like the top 50 countries sorted by their HDI ranking?

```bash
$ http :6543/api/gii_countries _limit==50 _sort==hdi_rank
```

If you want to reverse the sort order you can put a minus sign before the field name to be sorted by, e.g. `_sort==-hdi_rank`.

### More pagination

To customize where in the records the pagination begins or which page of the sequence to return, we use the `_start` and `_page` parameters.

#### Imaginary leaderboard app

<img src="https://upload.wikimedia.org/wikipedia/commons/2/20/Pinball_Dot_Matrix_Display_-_Demolition_Man.JPG" alt="Scoring" width=240 align=right hspace=30 />

For example, let's say we have a leaderboard app that classifies the top 5 countries as "gold medallists", the next 5 as "silver" and the 5 after that as "bronze". Maybe we only care about particular metrics, and want to filter out the noise from the other fields. Here are some examples of how to do that.

##### "Gold medal" countries for women's participation in the labour market:
```bash
$ http :6543/api/gii_countries _limit==5 _sort==-labour_force_participation_rate_aged_15_and_above_female_2012 _fields==country,labour_force_participation_rate_aged_15_and_above_female_2012
HTTP/1.1 200 OK
Cache-Control: max-age=0, must-revalidate, no-cache, no-store
Content-Length: 755
Content-Type: application/json; charset=UTF-8
Date: Tue, 01 Sep 2015 18:38:57 GMT
Expires: Tue, 01 Sep 2015 18:38:57 GMT
Last-Modified: Tue, 01 Sep 2015 18:38:57 GMT
Pragma: no-cache
Server: waitress

{
    "count": 5,
    "data": [
        {
            "_type": "GiiCountry",
            "country": "Tanzania (United Republic of)",
            "labour_force_participation_rate_aged_15_and_above_female_2012": 88.1
        },
        {
            "_type": "GiiCountry",
            "country": "Madagascar",
            "labour_force_participation_rate_aged_15_and_above_female_2012": 86.8
        },
        {
            "_type": "GiiCountry",
            "country": "Rwanda",
            "labour_force_participation_rate_aged_15_and_above_female_2012": 86.5
        },
        {
            "_type": "GiiCountry",
            "country": "Myanmar",
            "labour_force_participation_rate_aged_15_and_above_female_2012": 85.7
        },
        {
            "_type": "GiiCountry",
            "country": "Malawi",
            "labour_force_participation_rate_aged_15_and_above_female_2012": 84.7
        }
    ],
    "fields": "country,labour_force_participation_rate_aged_15_and_above_female_2012",
    "start": 0,
    "took": 4,
    "total": 206
}
```

<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/3/38/Flag_of_Tanzania.svg/1280px-Flag_of_Tanzania.svg.png" alt="Tanzania" width=160 align=right hspace=30 /> Way to go, Tanzania! (It would be interesting to learn more about the nature and quality of these jobs as well, but that is beyond our scope here.)

##### "Silver medallists" in women's labour market participation:

Let's add the `_start` argument to get the 6th-10th records, inclusive.

```bash
$ http :6543/api/gii_countries _limit==5 _start==5 _sort==-labour_force_participation_rate_aged_15_and_above_female_2012 _fields==country,labour_force_participation_rate_aged_15_and_above_female_2012
HTTP/1.1 200 OK
Cache-Control: max-age=0, must-revalidate, no-cache, no-store
Content-Length: 744
Content-Type: application/json; charset=UTF-8
Date: Wed, 02 Sep 2015 19:59:44 GMT
Expires: Wed, 02 Sep 2015 19:59:44 GMT
Last-Modified: Wed, 02 Sep 2015 19:59:44 GMT
Pragma: no-cache
Server: waitress

{
    "count": 5,
    "data": [
        {
            "_type": "GiiCountry",
            "country": "Burundi",
            "labour_force_participation_rate_aged_15_and_above_female_2012": 83.2
        },
        {
            "_type": "GiiCountry",
            "country": "Zimbabwe",
            "labour_force_participation_rate_aged_15_and_above_female_2012": 83.2
        },
        {
            "_type": "GiiCountry",
            "country": "Togo",
            "labour_force_participation_rate_aged_15_and_above_female_2012": 80.7
        },
        {
            "_type": "GiiCountry",
            "country": "Equatorial Guinea",
            "labour_force_participation_rate_aged_15_and_above_female_2012": 80.6
        },
        {
            "_type": "GiiCountry",
            "country": "Netherlands",
            "labour_force_participation_rate_aged_15_and_above_female_2012": 79.9
        }
    ],
    "fields": "country,labour_force_participation_rate_aged_15_and_above_female_2012",
    "start": 5,
    "took": 7,
    "total": 206
}
```

**Note**: Because the index starts at zero, we use `_start==5` to start at the 6th record.*

Give it up for Burundi and Zimbabwe!

##### "Bronze medallists":

We can use the `_start` parameter here like we did for the silver medallists, but let's try using `_page` to get the bronze countries instead. This should give us the 11th-15th records, inclusive. The `_page` parameter, like `_start`, is also zero-indexed.

```bash
$ http :6543/api/gii_countries _limit==5 _page==2 _sort==-labour_force_participation_rate_aged_15_and_above_female_2012 _fields==country,labour_force_participation_rate_aged_15_and_above_female_2012
HTTP/1.1 200 OK
Cache-Control: max-age=0, must-revalidate, no-cache, no-store
Content-Length: 765
Content-Type: application/json; charset=UTF-8
Date: Wed, 02 Sep 2015 20:37:11 GMT
Expires: Wed, 02 Sep 2015 20:37:11 GMT
Last-Modified: Wed, 02 Sep 2015 20:37:11 GMT
Pragma: no-cache
Server: waitress

{
    "count": 5,
    "data": [
        {
            "_type": "GiiCountry",
            "country": "Eritrea",
            "labour_force_participation_rate_aged_15_and_above_female_2012": 79.9
        },
        {
            "_type": "GiiCountry",
            "country": "Cambodia",
            "labour_force_participation_rate_aged_15_and_above_female_2012": 78.9
        },
        {
            "_type": "GiiCountry",
            "country": "Ethiopia",
            "labour_force_participation_rate_aged_15_and_above_female_2012": 78.2
        },
        {
            "_type": "GiiCountry",
            "country": "Burkina Faso",
            "labour_force_participation_rate_aged_15_and_above_female_2012": 77.1
        },
        {
            "_type": "GiiCountry",
            "country": "Lao People's Democratic Republic",
            "labour_force_participation_rate_aged_15_and_above_female_2012": 76.3
        }
    ],
    "fields": "country,labour_force_participation_rate_aged_15_and_above_female_2012",
    "start": 10,
    "took": 3,
    "total": 206
}
```

Nice showing, Eritrea!

### Elasticsearch DSL powers

This is where the real magic happens.

#### Full-text search

We already know that we can access individual country records by endpoints using the country's name e.g. `/api/gii_countries/Canada`, but I noticed that certain countries have more official/legal names according to the UN, and might not be listed under their more common names. For example, I'm pretty sure Venezuela is a country, but if I try to request its endpoint:

```bash
$ http :6543/api/gii_countries/Venezuela
HTTP/1.1 404 Not Found
Cache-Control: max-age=0, must-revalidate, no-cache, no-store
Content-Length: 312
Content-Type: application/json; charset=UTF-8
Date: Tue, 01 Sep 2015 19:08:17 GMT
Expires: Tue, 01 Sep 2015 19:08:17 GMT
Last-Modified: Tue, 01 Sep 2015 19:08:17 GMT
Pragma: no-cache
Server: waitress

{
    "explanation": "The resource could not be found.",
    "message": "'GiiCountry({'doc_type': 'GiiCountry', 'id': u'Venezuela', 'index': 'gii_api'})' resource not found",
    "request_url": "http://localhost:6543/api/gii_countries/Venezuela",
    "status_code": 404,
    "timestamp": "2015-09-01T19:08:17Z",
    "title": "Not Found"
}
```

No dice. Enter full-text search!

```bash
$ http :6543/api/gii_countries country==Venezuela
HTTP/1.1 200 OK
Cache-Control: max-age=0, must-revalidate, no-cache, no-store
Content-Length: 914
Content-Type: application/json; charset=UTF-8
Date: Tue, 01 Sep 2015 19:28:41 GMT
Etag: "b31365bc077839872f474e6bd8fe559c"
Expires: Tue, 01 Sep 2015 19:28:41 GMT
Last-Modified: Tue, 01 Sep 2015 19:28:41 GMT
Pragma: no-cache
Server: waitress

{
    "count": 1,
    "data": [
        {
            "_2010_2015_adolescent_birth_rate_births_per_1_000_women_aged_15_19": 83.2,
            "_2010_maternal_mortality_ratio_deaths_per_100_000_live_births": 92,
            "_2013_share_of_seats_in_parliament_held_by_women": 17.0,
            "_pk": "Venezuela (Bolivarian Republic of)",
            "_score": 2.109438,
            "_self": "http://localhost:6543/api/gii_countries/Venezuela%20%28Bolivarian%20Republic%20of%29",
            "_type": "GiiCountry",
            "_version": 0,
            "country": "Venezuela (Bolivarian Republic of)",
            "gender_inequality_index_rank_2013": 96.0,
            "gender_inequality_index_value_2013": 0.464,
            "hdi_rank": 67,
            "labour_force_participation_rate_aged_15_and_above_female_2012": 50.9,
            "labour_force_participation_rate_aged_15_and_above_male_2012": 79.2,
            "population_with_some_secondary_ed_aged_25_and_up_fem_2005_2012": 56.5,
            "population_with_some_secondary_ed_aged_25_and_up_male_2005_2012": 50.8
        }
    ],
    "fields": "",
    "start": 0,
    "took": 3,
    "total": 1
}
```

Aha! Turns out the full name for Venezuela is "Venezuela (Bolivarian Republic of)".

#### Ranges

Maybe we'd like to know which countries have women holding at least 50% of the seats in parliament. Voilà:

```bash
$ http :6543/api/gii_countries _2013_share_of_seats_in_parliament_held_by_women=="[50 TO *]"
HTTP/1.1 200 OK
Cache-Control: max-age=0, must-revalidate, no-cache, no-store
Content-Length: 1562
Content-Type: application/json; charset=UTF-8
Date: Tue, 01 Sep 2015 19:48:02 GMT
Etag: "963212cb572b10eb4efaa38f0bfaf4c6"
Expires: Tue, 01 Sep 2015 19:48:02 GMT
Last-Modified: Tue, 01 Sep 2015 19:48:02 GMT
Pragma: no-cache
Server: waitress

{
    "count": 2,
    "data": [
        {
            "_2010_2015_adolescent_birth_rate_births_per_1_000_women_aged_15_19": null,
            "_2010_maternal_mortality_ratio_deaths_per_100_000_live_births": null,
            "_2013_share_of_seats_in_parliament_held_by_women": 50.0,
            "_pk": "Andorra",
            "_score": 1.0,
            "_self": "http://localhost:6543/api/gii_countries/Andorra",
            "_type": "GiiCountry",
            "_version": 0,
            "country": "Andorra",
            "gender_inequality_index_rank_2013": null,
            "gender_inequality_index_value_2013": null,
            "hdi_rank": 37,
            "labour_force_participation_rate_aged_15_and_above_female_2012": null,
            "labour_force_participation_rate_aged_15_and_above_male_2012": null,
            "population_with_some_secondary_ed_aged_25_and_up_fem_2005_2012": 49.5,
            "population_with_some_secondary_ed_aged_25_and_up_male_2005_2012": 49.3
        },
        {
            "_2010_2015_adolescent_birth_rate_births_per_1_000_women_aged_15_19": 33.6,
            "_2010_maternal_mortality_ratio_deaths_per_100_000_live_births": 340,
            "_2013_share_of_seats_in_parliament_held_by_women": 51.9,
            "_pk": "Rwanda",
            "_score": 1.0,
            "_self": "http://localhost:6543/api/gii_countries/Rwanda",
            "_type": "GiiCountry",
            "_version": 0,
            "country": "Rwanda",
            "gender_inequality_index_rank_2013": 79.0,
            "gender_inequality_index_value_2013": 0.41,
            "hdi_rank": 151,
            "labour_force_participation_rate_aged_15_and_above_female_2012": 86.5,
            "labour_force_participation_rate_aged_15_and_above_male_2012": 85.5,
            "population_with_some_secondary_ed_aged_25_and_up_fem_2005_2012": 7.4,
            "population_with_some_secondary_ed_aged_25_and_up_male_2005_2012": 8.0
        }
    ],
    "fields": "",
    "start": 0,
    "took": 6,
    "total": 2
}
```

Not bad Andorra and Rwanda, not bad at all.


#### Aggregations

Let's say we want to know something that requires a little computation, like the average level of gender inequality worldwide. This (and all kinds of other questions) can be answered using aggregations.

First, open `local.ini` and change `elasticsearch.enable_aggregations` to `true` because this feature is disabled by default. Then, restart the server.

```bash
$ http :6543/api/gii_countries _aggs.avg_gender_ineq.avg.field==gender_inequality_index_value_2013
HTTP/1.1 200 OK
Cache-Control: max-age=0, must-revalidate, no-cache, no-store
Content-Length: 50
Content-Type: application/json; charset=UTF-8
Date: Tue, 01 Sep 2015 20:13:57 GMT
Expires: Tue, 01 Sep 2015 20:13:57 GMT
Last-Modified: Tue, 01 Sep 2015 20:13:57 GMT
Pragma: no-cache
Server: waitress

{
    "avg_gender_ineq": {
        "value": 0.3809693251533742
    }
}
```

The average global gender inequality, expressed as a percentage of "lost" human development, is 38%. That is, if there was no gender inequality, the world would be considered 38% more developed than its current state.

## Bonus

To see what the Elasticsearch query really looks like behind the scenes, you can add a logger to `local.ini`. Add a section like so:

```ini
[logger_elasticsearch]
level = INFO
handlers =
qualname = elasticsearch.trace
```

Then make sure the new logger is added to the loggers list on line 47:

```ini
[loggers]
keys = root, gii_api, nefertari, ramses, elasticsearch
```

Shut down and restart the server. Now if you rerun the above aggregation and look in the server log, you'll see the following:

```bash
2015-09-01 16:13:57,201 INFO  [elasticsearch.trace][Dummy-3] base.log_request_success: curl -XGET 'http://localhost:9200/gii_api/GiiCountry/_search?pretty&search_type=count' -d '{
  "aggregations": {
    "avg_gender_ineq": {
      "avg": {
        "field": "gender_inequality_index_value_2013"
      }
    }
  },
  "query": {
    "match_all": {}
  }
}'
```

With that information you can compare the queries that the server generates directly with the Elasticsearch aggregations documentation. Now you can explore and figure out how to build more advanced aggregations! Check out the [Elastic aggregations docs to learn more](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html). 

## Help

The Ramses team is available to chat on our Gitter channel, for help digging into the data, or for any help using the stack on your own projects. [Join us here](https://gitter.im/brandicted/ramses?utm_source=share-link&utm_medium=link&utm_campaign=share-link). 
