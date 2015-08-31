# Make an Elasticsearch-powered REST API for any data with Ramses

Many startup products seem to take the form of "collect data from a third party and mash it up with something else".

In this short guide I'm going to show you how download a dataset and create a REST API out of it. This new API uses Elasticsearch to power the endpoints, so you can build a product around your data without having to expose Elasticsearch directly in production. This also allows for proper authentication and security, custom logic, other databases, and even auto-generated client libraries.

Ramses is a bit like Parse for open source in that it provides the convenience of "backend as a service" except that you run the server yourself and have full access to the internals.

One interesting dataset I came across recently is the Gender Inequality Index that is published by the UN Development Programme. This dataset is a twist on the classic Human Development Index. The HDI ranks countries based on their levels of lifespan, education and income. The GII, on the other hand, ranks countries based on how they stack up in terms of gender (in)equality. The metrics in the GII are a combination of women's reproductive health, social empowerment, and labour force participation. This dataset is missing non-binary gender identities, so hopefully the UNDP will be able to add that soon.

Let's make a REST API out of the GII and then query it in fun ways using the Elasticsearch query DSL via the endpoint's URL.

You can see the completed code for this example here: https://github.com/chrstphrhrt/ramses-elasticsearch

## Set up the project

I'm assuming a you have some things already before we dive in:

* Recent versions of Elasticsearch and PostgreSQL installed and running in the background with default configurations.
* Python 2.7, 3.3 or 3.4 and virtualenv
* I'm using [httpie](https://github.com/jkbrzt/httpie) but you can use curl if you want.

Open a terminal, install Ramses, and create your new project:

```bash
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

There are two main files in the boilerplate project right now: `api.raml`, and `items.json`. `api.raml` is a [RAML](http://raml.org/) file, which is a DSL for describing REST APIs in YAML. It configures your endpoints. `items.json` is the schema that describes the fake boilerplate "Item" model.

We are going to replace these files with real ones based on the GII data.

First, download the data from the UNDP site to the root of the project (`gii_api/`), and rename the old `items.json` to `gii_schema.json`.

```bash
$ wget $GITHUB_LINK_TO_DATA
...
2015-08-31 15:58:34 (198 KB/s) - 'gii_data.json' saved [75659]
$ mv items.json gii_schema.json
```

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

We want to replace the "properties" section with the field names and types from our dataset. Update the `title` and `required` fields, and make the `country` field be the primary key, like so:

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

Now that we have data and a schema to describe it, let's hook up the endpoints. Open `api.raml` change the boilerplate Item endpoint to use the GII countries schema instead.

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

...after:

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

Simple!

Now we can drop the database, delete the Elasticsearch index and restart the server. Make sure to keep the

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
```

