<!--
  ~ Licensed to the Apache Software Foundation (ASF) under one or more
  ~ contributor license agreements.  See the NOTICE file distributed with
  ~ this work for additional information regarding copyright ownership.
  ~ The ASF licenses this file to You under the Apache License, Version 2.0
  ~ (the "License"); you may not use this file except in compliance with
  ~ the License.  You may obtain a copy of the License at
  ~
  ~      http://www.apache.org/licenses/LICENSE-2.0
  ~
  ~ Unless required by applicable law or agreed to in writing, software
  ~ distributed under the License is distributed on an "AS IS" BASIS,
  ~ WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  ~ See the License for the specific language governing permissions and
  ~ limitations under the License.
  -->
Consent API
===========

Starting with Apache Unomi 1.3 (still in development), a new API for consent management is now available. This API
is designed to be able to store/retrieve/update visitor consents in order to comply with new 
privacy regulations such as the [GDPR](https://en.wikipedia.org/wiki/General_Data_Protection_Regulation).

Profiles with consents
----------------------

Visitor profiles now contain a new Consent object that contains the following information:

- a scope
- a type identifier for the consent. This can be any key to reference a consent. Note that Unomi does not manage consent 
definitions, it only stores/retrieves consents for each profile based on this type
- a status : GRANT, DENY or REVOKED
- a status date (the date at which the status was updated)
- a revocation date, in order to comply with GDPR this is usually set at two years

Here is an example of a Profile with a consent attached to it:

    {
        "profileId": "8cbe380f-57bb-419d-97bf-24bf30178550",
        "sessionId": "0d755f4e-154a-45c8-9169-e852e1d706d9",
        "consents": {
            "example/newsletter": {
                "scope": "example",
                "typeIdentifier": "newsletter",
                "status": "GRANTED",
                "statusDate": "2018-05-22T09:44:33Z",
                "revokeDate": "2020-05-21T09:44:33Z"
            }
        }
    }

It is of course possible to have multiple consents defined for a single visitor profile.

Consent type definitions
------------------------

Apache Unomi does not manage consent definitions, it leaves that to an external system (for example a CMS) so that it 
can handle user-facing UIs to create, update, internationalize and present consent definitions to end users. 

The only thing that is import to Apache Unomi to manage visitor consents is a globally unique key, that is called the
consent type.

Creating / update a visitor consent
-----------------------------------

A new built-in event type called "modifyConsent" can be sent to Apache Unomi to update a consent for the current
profile.

Here is an example of such an event:

    {
      "events": [
        {
          "scope": "example",
          "eventType": "modifyConsent",
          "source": {
            "itemType": "page",
            "scope": "example",
            "itemId": "anItemId"
          },
          "target": {
            "itemType": "anyType",
            "scope": "example",
            "itemId": "anyItemId"
          },
          "properties": {
            "consent": {
              "typeIdentifier": "newsletter",
              "scope": "example",
              "status": "GRANTED",
              "statusDate": "2018-05-22T09:27:09.473Z",
              "revokeDate": "2020-05-21T09:27:09.473Z"
            }
          }
        }
      ]
    }

You could send it using the following curl request:

    curl -H "Content-Type: application/json" -X POST -d '{"source":{"itemId":"homepage","itemType":"page","scope":"example"},"events":[{"scope":"example","eventType":"modifyConsent","source":{"itemType":"page","scope":"example","itemId":"anItemId"},"target":{"itemType":"anyType","scope":"example","itemId":"anyItemId"},"properties":{"consent":{"typeIdentifier":"newsletter","scope":"example","status":"GRANTED","statusDate":"2018-05-22T09:27:09.473Z","revokeDate":"2020-05-21T09:27:09.473Z"}}}]}' http://localhost:8181/context.json?sessionId=1234

How it works (internally)
-------------------------

Upon receiving this event, Apache Unomi will trigger the modifyAnyConsent rule that has the following definition:

    {
      "metadata" : {
        "id": "modifyAnyConsent",
        "name": "Modify any consent",
        "description" : "Modify any consent and sets the consent in the profile",
        "readOnly":true
      },
    
      "condition" : {
        "type": "modifyAnyConsentEventCondition",
        "parameterValues": {
        }
      },
    
      "actions" : [
        {
          "type": "modifyConsentAction",
          "parameterValues": {
          }
        }
      ]
    
    }
    
As we can see this rule is pretty simple it will simply execute the modifyConsentAction that is implemented by the 
[ModifyConsentAction Java class](https://github.com/apache/incubator-unomi/blob/9f1bab437fd93826dc54d318ed00d3b2e3161437/plugins/baseplugin/src/main/java/org/apache/unomi/plugins/baseplugin/actions/ModifyConsentAction.java)

This class will update the current visitor profile to add/update/revoke any consents that are included in the event.