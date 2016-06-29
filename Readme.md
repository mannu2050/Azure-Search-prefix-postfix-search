---
services: active-directory
platforms: java
author: Manish Sharma
---

#Prefix & postfix searching with Azure Search
If you have data like:
(1) Tomcat
(2) Catalina
(3) StuffedCat
and you want to show all three if cat is specified in the search criteria. It is as simple as you want.

In traditional infix search you will get the resultset when it is starting with cat e.g. catalina will be returned once you search for cat in the above dataset.
The answer to this problem is custom analyzer refer [here]().

##Create Index:

``` REST
POST https://[servicename].search.windows.net/indexes?api-version=[api-version]
Content-Type: application/json 
api-key: [admin key]
Body:
{   
    "name":"people",
    "fields": [
        { "name":"id", "type":"Edm.String", "key":true, "searchable":false },
        { "name":"name", "type":"Edm.String", "searchable":true, "analyzer":"my_standard" },
        { "name":"partialName", "type":"Edm.String", "searchable":true, "searchAnalyzer":"standard", "indexAnalyzer":"prefixAnalyzer" },
        { "name":"suffixName", "type":"Edm.String", "searchable":true, "searchAnalyzer":"reverseText", "indexAnalyzer":"suffixAnalyzer" }
    ],
    "analyzers": [
        {"name":"my_standard","@odata.type":"#Microsoft.Azure.Search.CustomAnalyzer","tokenizer":"standard","tokenFilters": [ "lowercase", "asciifolding", "phonetic" ]},
        {"name":"prefixAnalyzer","@odata.type":"#Microsoft.Azure.Search.CustomAnalyzer","tokenizer":"standard","tokenFilters":[ "lowercase", "my_edgeNGram" ]},
        {"name":"suffixAnalyzer","@odata.type":"#Microsoft.Azure.Search.CustomAnalyzer","tokenizer":"standard","tokenFilters":[ "lowercase", "asciifolding", "reverse", "my_edgeNGram" ]},
        {"name":"reverseText","@odata.type":"#Microsoft.Azure.Search.CustomAnalyzer","tokenizer":"standard","tokenFilters":[ "lowercase", "reverse" ]}
    ],
    "tokenFilters": [
        {"name":"my_edgeNGram","@odata.type":"#Microsoft.Azure.Search.EdgeNGramTokenFilter","minGram":2,"maxGram":20}
    ]
}
``` 

##Push Data:

``` REST
POST https://[service name].search.windows.net/indexes/[index name]/docs/index?api-version=[api-version] 
Content-Type: application/json 
api-key: [admin key]
Body:
{
    "value":[
        {"id":"1","name":"Tomcat","partialName":" Tomcat ", "suffixName":" Tomcat "},
        {"id":"2","name":"Catalina","partialName":" Catalina ", "suffixName":" Catalina "},
        {"id":"3","name":"StuffedCat","partialName":" StuffedCat", "suffixName":"StuffedCat"}
    ]
}
```

So platform is set, time to rock it:
##Search

``` REST
GET https://testl300.search.windows.net/indexes/people/docs?search=cat&api-version=2015-02-28-Preview
Content-Type: application/json 
api-key: [admin key]

Result:
{"@odata.context":"https://testl300.search.windows.net/indexes('people')/$metadata#docs(id,name,partialName,suffixName)","value":[{"@search.score":0.065066196,"id":"3","name":"StuffedCat","partialName":" StuffedCat","suffixName":"StuffedCat"},{"@search.score":0.065066196,"id":"1","name":"Tomcat","partialName":" Tomcat ","suffixName":" Tomcat "},{"@search.score":0.065066196,"id":"2","name":"Catalina","partialName":" Catalina ","suffixName":" Catalina "}]}
```

##Happy Searching to you!