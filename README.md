# tutorials.Combining-NGSI-v2-LD


-  Want to augment our own NGSI-LD application using data from an existing third-party NGSI-v2  system on the edge
-  We are using common smart data models and have an existing public `@context` file
-  The third-party system is using NGSI-v2 (and potentially a different data model)
-  Cannot refactor the existing edge system 

Need to sync some data into our NGSI-LD system, and potentially maintain metadata links as well (e.g. a `luminosity` reading with Properties-of-Properties 
such as `unitCode` and `providedBy`)

# Architecture

![](https://jason-fox.github.io/tutorials.Combining-NGSI-v2-LD/img/architecture.png)




### Copy NGSI v2 Data as NGSI-LD

Raise an expired NGSI-v2 subscription to raise a one-off subscription. Each entity is processed in turn by the 
`http://tutorial:3000/device/subscription/initialize` endpoint. The code for processing entities can be found
[here](https://github.com/FIWARE/tutorials.Step-by-Step/blob/master/context-provider/controllers/ngsi-ld/device-convert.js)

```javascript
function duplicateDevices(req, res) {
    async function copyEntityData(device, index) {
        await upsertDeviceEntityAsLD(device);
    }
    req.body.data.forEach(copyEntityData);
    res.status(204).send();
}
```

Each entry is converted an upserted using the function below:

```javascript
function upsertDeviceEntityAsLD(device) {
    return new Promise((resolve, reject) => {
        const json = convertEntityToLD(device);
        delete json['@context']; // Not required for Linked Data Upsert.

        const options = {
            url: basePath + '/entityOperations/upsert/?options=update',
            method: 'POST',
            json: [json],
            headers: {
                'Content-Type': 'application/json',
                Link:
                    '<' + dataModelContext + '>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"'
            }
        };

        request(options, (error, response, body) => {
            return error ? reject(error) : resolve();
        });
    });
}
```



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

### Raise as device shadowing subscription

Raise an ongoging NGSI-v2 subscription. Each entity is processed in turn by the `device/subscription/luminosity`
endpoint which updates the shadowed device and also upserts the `controllingAsset` (in this case the building
the lamp is located in.)


```javascript
function shadowDeviceMeasures(req, res) {
    const attrib = req.params.attrib;

    async function copyAttributeData(device, index) {
        await upsertDeviceEntityAsLD(device);
        if (device[attrib]) {
            await upsertLinkedAttributeDataAsLD(device, 'controlledAsset', attrib);
        }
    }

    req.body.data.forEach(copyAttributeData);
    res.status(204).send();
}
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

### Retrieve the NGSI-LD version of the device

Lamp1 is mapped to `http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:Lamp:001` within the NGSI-LD context broker.
It can be retrieved by a simple entity request. `unitCode` and `observedAt` have been extracted as _Property-of-Property_
string elements in the response.

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

### Retrieve the NGSI-LD version of the store

The subscription code also updates the `luminosity` attribute of the Building `urn:ngsi-ld:Building:store001`
this is a linked data element which, in addition to the `unitCode` holds a _Relationship-of-Property_ attribute called
`providedBy` enabling the navigation of the knowledge graph.

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
