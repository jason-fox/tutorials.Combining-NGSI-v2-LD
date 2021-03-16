# tutorials.Combining-NGSI-v2-LD


-  Want to augment our own NGSI-LD application using data from an existing third-party NGSI-v2  system on the edge
-  We are using common smart data models and have an existing public @context file
-  The third-party system is using NGSI-v2 (and potentially a different data model)
-  Cannot refactor the existing edge system 

Need to sync some data into our NGSI-LD system, and potentially maintain metadata links as well (e.g. a `luminosity` reading with Properties-of-Properties 
such as `unitCode` and `providedBy`)

# Architecture

![](https://jason-fox.github.io/tutorials.Combining-NGSI-v2-LD/img/architecture.png)


#### Request

```console
curl -L -X POST 'http://localhost:1027/v2/subscriptions/' \
-H 'Content-Type: application/json' \
-H 'fiware-service: openiot' \
--data-raw '{
  "description": "One-off Subscription to copy Device Entities",
  "subject": {
    "entities": [
      {
        "idPattern": ".*"
      }
    ]
  },
  "notification": {
    "http": {
      "url": "http://tutorial:3000/device/subscription/initialize"
    }
  },
  "expires": "2017-01-01T14:00:00.00Z"
}'
```


#### Request

```console
curl -L -X POST 'http://localhost:1027/v2/subscriptions/' \
-H 'Content-Type: application/json' \
-H 'fiware-service: openiot' \
--data-raw '{
  "description": "Notify Subscription Listener of Lamp context changes",
  "subject": {
    "entities": [
      {
        "idPattern": "Lamp.*"
      }
    ],
    "condition": {
      "attrs": ["luminosity"]
    }
  },
  "notification": {
    "http": {
      "url": "http://tutorial:3000/device/subscription/luminosity"
    },
    "attrs": ["luminosity", "controlledAsset", "supportedUnits"]
  },
  "throttling": 5
}'
```

#### Request

```console
curl -L -X GET 'http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:Lamp:001' \
-H 'Link: <https://fiware.github.io/tutorials.Step-by-Step/tutorials-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"'
```

#### Response

```json
{
    "@context": "https://fiware.github.io/tutorials.Step-by-Step/tutorials-context.jsonld",
    "id": "urn:ngsi-ld:Lamp:001",
    "type": "Lamp",
    "TimeInstant": {
        "type": "Property",
        "value": {
            "@type": "DateTime",
            "@value": "2021-03-16T12:24:48.918Z"
        }
    },
    "category": {
        "type": "Property",
        "value": [
            "actuator",
            "sensor"
        ]
    },
    "controlledAsset": {
        "object": "urn:ngsi-ld:Building:store001",
        "type": "Relationship",
        "observedAt": "2021-03-16T12:26:02.247Z"
    },
    "controlledProperty": {
        "type": "Property",
        "value": "light"
    },
    "function": {
        "type": "Property",
        "value": [
            "on",
            "off"
        ]
    },
    "luminosity": {
        "value": 1029,
        "type": "Property",
        "unitCode": "CDL",
        "observedAt": "2021-03-16T12:26:02.247Z"
    },
    "off_info": {
        "type": "Property",
        "value": {
            "@type": "commandResult",
            "@value": " "
        }
    },
    "off_status": {
        "type": "Property",
        "value": {
            "@type": "commandStatus",
            "@value": "UNKNOWN"
        }
    },
    "on_info": {
        "type": "Property",
        "value": {
            "@type": "commandResult",
            "@value": " "
        }
    },
    "on_status": {
        "type": "Property",
        "value": {
            "@type": "commandStatus",
            "@value": "UNKNOWN"
        }
    },
    "state": {
        "type": "Property",
        "value": {
            "@type": "Intangible",
            "@value": null
        }
    },
    "supportedProtocol": {
        "type": "Property",
        "value": "ul20"
    },
    "supportedUnits": {
        "value": "CDL",
        "type": "Property",
        "observedAt": "2021-03-16T12:26:02.247Z"
    }
}
```

#### Request

```console
curl -L -X GET 'http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:Building:store001' \
-H 'Link: <https://fiware.github.io/tutorials.Step-by-Step/tutorials-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"'
```

#### Response

```json
{
    "@context": "https://fiware.github.io/tutorials.Step-by-Step/tutorials-context.jsonld",
    "id": "urn:ngsi-ld:Building:store001",
    "type": "Building",
    "category": {
        "type": "Property",
        "value": "commercial"
    },
    "address": {
        "type": "Property",
        "value": {
            "streetAddress": "Bornholmer Straße 65",
            "addressRegion": "Berlin",
            "addressLocality": "Prenzlauer Berg",
            "postalCode": "10439"
        },
        "verified": {
            "type": "Property",
            "value": true
        }
    },
    "location": {
        "type": "GeoProperty",
        "value": {
            "type": "Point",
            "coordinates": [
                13.3986,
                52.5547
            ]
        }
    },
    "name": {
        "type": "Property",
        "value": "Bösebrücke Einkauf"
    },
    "furniture": {
        "type": "Relationship",
        "object": [
            "urn:ngsi-ld:Shelf:unit001",
            "urn:ngsi-ld:Shelf:unit002",
            "urn:ngsi-ld:Shelf:unit003"
        ]
    },
    "luminosity": {
        "value": 962,
        "type": "Property",
        "providedBy": {
            "type": "Relationship",
            "object": "urn:ngsi-ld:Lamp:001"
        },
        "unitCode": "CDL",
        "observedAt": "2021-03-16T13:35:04.690Z"
    }
}
```
