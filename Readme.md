---
services: azure-search
platforms: REST
author: Manish Sharma
---

#Prefix & postfix searching with Azure Search
In Azure Search there is a possibility to address custom search analyzers which will address custom specifications. 
In this post I am describing one of the custom analyzer scenario. 
Let us take an example in which a search index will have the following data:
(1) Tomcat
(2) Catalina
(3) StuffedCat

Now when user specify the search criteria as cat then all of the abovementioned will appears in search result. 
It is as simple as you want :).

In traditional infix search you will get the resultset when it is starting with cat e.g. catalina will be returned once user specifies search criteria as cat.
The answer to this problem is custom analyzer refer [here]().
The solution is having three parts: one is creating Index in which we specify the custom analyzer & define token filter.

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
If you concentrate on the analyzers part we have defined analyzer as standard, prefix, suffix & reverse analyzer. This is the crux of this solution, the edge_ngram tokenizer first breaks text down into words whenever 
it encounters one of a list of specified characters, then it emits N-grams of each word where the start of the N-gram is anchored to the beginning of the word. However in this case we took reverse text too, now this will address
our requirement to match both the suffix & prefix matching.

This will require us to replicate the same data into two additional fields i.e. partial name & suffix name. Now the index is created, let us try to push some data into it.
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

So platform is set, its time to rock it:
##Search

``` REST
GET https://testl300.search.windows.net/indexes/people/docs?search=cat&api-version=2015-02-28-Preview
Content-Type: application/json 
api-key: [admin key]

Result:
{"@odata.context":"https://testl300.search.windows.net/indexes('people')/$metadata#docs(id,name,partialName,suffixName)","value":[{"@search.score":0.065066196,"id":"3","name":"StuffedCat","partialName":" StuffedCat","suffixName":"StuffedCat"},{"@search.score":0.065066196,"id":"1","name":"Tomcat","partialName":" Tomcat ","suffixName":" Tomcat "},{"@search.score":0.065066196,"id":"2","name":"Catalina","partialName":" Catalina ","suffixName":" Catalina "}]}
```

##Happy Searching to you!