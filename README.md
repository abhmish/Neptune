With Amazon Neptune, you can use relationships to process financial and purchase transactions in near real time to easily detect fraud patterns. Neptune provides a fully managed service to execute fast graph queries to detect that a potential purchaser is using the same email address and credit card as a known fraud case. If you are building a retail fraud detection application, Neptune can help you build graph queries to easily detect relationship patterns like multiple people associated with a personal email address, or multiple people sharing the same IP address but residing in different physical addresses.

Amazon Neptune Supports two widely used Open Standards for Graph Database; 
1. Property Graphs
2. RDF Graphs

For this demo we are going to use Property Graphs

# Scenario 1: Credit Card Fraud Detection Using Amazon Neptune

Credit card fraud is a wide-ranging term for theft and fraud committed using or involving a payment card, such as a credit card or debit card, as a fraudulent source of funds in a transaction.The purpose may be to obtain goods without paying, or to obtain unauthorized funds from an account. Credit card fraud is also an adjunct to identity theft. 

The sample dataset that we are referring here is consists of users, merchants and their credit card transactions.

![Credit Card Fraud Graph](https://github.com/abhmish/Neptune/blob/master/img/credicardfraud.jpg)


## Steps:

1. Setup Amazon Neptune Cluster
2. Bulk Load data into Neptune
3. Query data using Apache TinkerPop Gremlin


## 1. Setup Amazon Neptune Cluster

We will setup Amazon Neptune cluster in US EAST REGION. Open the Neptune Quickstart link here: [https://docs.aws.amazon.com/neptune/latest/userguide/quickstart.html](https://docs.aws.amazon.com/neptune/latest/userguide/quickstart.html)

Select the cloud formation template for US EAST Region and Launch

## 2. Bulk Load data into Neptune

Copy data from [https://github.com/abhmish/Neptune/tree/master/data](https://github.com/abhmish/Neptune/tree/master/data) into your s3 bucket

Neptune currently support connecting from EC2 instance inside VPC same as that of Neptune cluster. The cloudformation stack has created EC2 instance. 

Open terminal inside EC2 instance created and run following command in terminal

```
curl -X POST 
    -H 'Content-Type: application/json' 
    http://<your-neptune-endpoint>:8182/loader -d '{ 
      "source" : "s3://<s3-bucket>/creditcardfraud",
      "accessKey" : "",
      "secretKey" : "",
      "format" : "csv",
      "region" : "us-east-1",
      "failOnError" : "FALSE"
    }'
```

This loads three datasets into Neptune; 1. List of Persons 2. List of Merchants 3. Transactions done by Person at Merchants

You can view the sample datasets at: s3://neptune101

The load command returns a load identifier which can be used to query the status of load request

`curl -X GET 'http://<your-neptune-endpoint>:8182/loader/<load-id>?details=true&errors=true&page=1&errorsPerPage=3'
`

For more information about loading data into Amazon Neptune visit: [https://docs.aws.amazon.com/neptune/latest/userguide/bulk-load.html](https://docs.aws.amazon.com/neptune/latest/userguide/bulk-load.html)


## 3. Query data using Apache TinkerPop Gremlin

We are going to use Apache TinkerPop Gremlin to query data. 

refer this to setup Gremlin console and connect to your Neptune cluster

[https://docs.aws.amazon.com/neptune/latest/userguide/access-graph-gremlin-console.html](https://docs.aws.amazon.com/neptune/latest/userguide/access-graph-gremlin-console.html)

* List Persons

`g.V().hasLabel("Person").limit(5)`


```
==>v[John]
==>v[Zoey]
==>v[Ava]
==>v[Olivia]
==>v[Mia]
```

* List Merchants

`g.V().hasLabel("Merchant").limit(5)`

==>v[Justice]
==>v[Sears]
==>v[Sprint]
==>v[Abercrombie]
==>v[Wallmart]

* List Persons and merchants where they have done transactions

`g.V().hasLabel("Person").project("person","merchants").by("name").by(out().fold())`

```
==>[person:John,merchants:[v[Justice],v[Sprint],v[American_Apparel],v[Just_Brew_It],v[Soccer_for_the_City]]]
==>[person:Zoey,merchants:[v[Abercrombie],v[MacDonalds],v[Just_Brew_It],v[Subway],v[Amazon]]]
==>[person:Ava,merchants:[v[Sears],v[Wallmart],v[American_Apparel],v[American_Apparel],v[Just_Brew_It]]]
==>[person:Olivia,merchants:[v[Wallmart],v[Just_Brew_It],v[Soccer_for_the_City],v[Soccer_for_the_City],v[Apple_Store],v[Urban_Outfitters],v[RadioShack],v[Macys]]]
==>[person:Mia,merchants:[v[Sears],v[MacDonalds],v[Soccer_for_the_City],v[Soccer_for_the_City],v[Starbucks],v[Amazon]]]
==>[person:Madison,merchants:[v[Abercrombie],v[Wallmart],v[MacDonalds],v[Subway],v[Apple_Store],v[Urban_Outfitters],v[RadioShack],v[Amazon],v[Macys]]]
==>[person:Paul,merchants:[v[Sears],v[Wallmart],v[Just_Brew_It],v[Starbucks],v[Apple_Store],v[Urban_Outfitters],v[RadioShack],v[Macys]]]
==>[person:Jean,merchants:[v[Abercrombie],v[Wallmart],v[Soccer_for_the_City],v[Subway],v[Amazon]]]
==>[person:Dan,merchants:[v[MacDonalds],v[MacDonalds],v[Soccer_for_the_City],v[Amazon]]]
==>[person:Marc,merchants:[v[Sears],v[Wallmart],v[American_Apparel],v[Soccer_for_the_City],v[Apple_Store],v[Urban_Outfitters],v[RadioShack],v[Amazon],v[Macys]]]

```

* Find all Merchants where fraud Transactions were done, along with list of Victims

```
g.V().
hasLabel("Merchant").
filter(__.inE().has("status","Disputed")).
project("merchant","persons").by("name").by(inE().outV().fold())

```

```
==>[merchant:Apple_Store,persons:[v[Paul],v[Marc],v[Olivia],v[Madison]]]
==>[merchant:Urban_Outfitters,persons:[v[Paul],v[Marc],v[Olivia],v[Madison]]]
==>[merchant:RadioShack,persons:[v[Paul],v[Marc],v[Olivia],v[Madison]]]
==>[merchant:Macys,persons:[v[Paul],v[Marc],v[Olivia],v[Madison]]] 
```

* Find all Undisputed transactions done prior to Disputed transactions

```
g.E().
hasLabel('HAS_BOUGHT_AT').has('status', 'Disputed').as('disputed').
outV().as('victim').
outE('HAS_BOUGHT_AT').has('status','Undisputed').as('undisputed').
filter(__.select('undisputed').where('undisputed',lt('disputed')).by('time')).as("t").inV().as("merchants").select("merchants","t","victim").
project("Victim","Merchants", "Amount","Time").
  by(__.select('victim').by('name')).
  by(__.select('merchants').by('name')).
  by(__.select('undisputed').by('amount')).
  by(__.select('undisputed').by('time')).dedup()
```

```
==>[Victim:Paul,Merchants:Sears,Amount:475,Time:Fri Mar 28 00:00:00 UTC 2014]
==>[Victim:Paul,Merchants:Wallmart,Amount:654,Time:Thu Mar 20 00:00:00 UTC 2014]
==>[Victim:Paul,Merchants:Just_Brew_It,Amount:986,Time:Thu Apr 17 00:00:00 UTC 2014]
==>[Victim:Paul,Merchants:Starbucks,Amount:239,Time:Thu May 15 00:00:00 UTC 2014]
==>[Victim:Madison,Merchants:Abercrombie,Amount:19,Time:Tue Jul 29 00:00:00 UTC 2014]
==>[Victim:Madison,Merchants:Wallmart,Amount:91,Time:Sun Jun 29 00:00:00 UTC 2014]
==>[Victim:Madison,Merchants:MacDonalds,Amount:630,Time:Mon Oct 06 00:00:00 UTC 2014]
==>[Victim:Madison,Merchants:Subway,Amount:352,Time:Tue Dec 16 00:00:00 UTC 2014]
==>[Victim:Madison,Merchants:Amazon,Amount:147,Time:Sun Aug 03 00:00:00 UTC 2014]
==>[Victim:Olivia,Merchants:Wallmart,Amount:231,Time:Sat Jul 12 00:00:00 UTC 2014]
==>[Victim:Olivia,Merchants:Just_Brew_It,Amount:742,Time:Tue Aug 12 00:00:00 UTC 2014]
==>[Victim:Olivia,Merchants:Soccer_for_the_City,Amount:74,Time:Thu Sep 04 00:00:00 UTC 2014]
==>[Victim:Olivia,Merchants:Soccer_for_the_City,Amount:924,Time:Sat Oct 04 00:00:00 UTC 2014]
==>[Victim:Marc,Merchants:Sears,Amount:430,Time:Sun Aug 10 00:00:00 UTC 2014]
==>[Victim:Marc,Merchants:Wallmart,Amount:964,Time:Sat Mar 22 00:00:00 UTC 2014]
==>[Victim:Marc,Merchants:American_Apparel,Amount:336,Time:Thu Apr 03 00:00:00 UTC 2014]
==>[Victim:Marc,Merchants:Soccer_for_the_City,Amount:11,Time:Thu Sep 04 00:00:00 UTC 2014]
==>[Victim:Marc,Merchants:Amazon,Amount:134,Time:Mon Apr 14 00:00:00 UTC 2014]
```

* Identify point of origin of Fraud

```
g.E(). 
hasLabel('HAS_BOUGHT_AT').has('status', 'Disputed').as('disputed'). 
outV().as('victim'). 
outE('HAS_BOUGHT_AT').has('status', 'Undisputed').as('undisputed').
filter(__.select('undisputed').where('undisputed', lt('disputed')).
by('time')).as("t").inV().as("merchants").
select("merchants","t","victim").group().
    by(__.select('merchants').coalesce(__.values('name'))).
    by(__.fold().project('Suspicious_Store', 'Count', 'Victims').
        by(__.unfold().select('merchants').coalesce(__.values('name'))).
        by(__.unfold().select('t').dedup().count()).
        by(__.unfold().select('victim').coalesce(__.values('name')).dedup().fold())).
unfold().select(values).dedup()
```

```==>[Suspicious_Store:Sears,Count:2,Victims:[Paul,Marc]]
==>[Suspicious_Store:Wallmart,Count:4,Victims:[Paul,Madison,Olivia,Marc]]
==>[Suspicious_Store:Abercrombie,Count:1,Victims:[Madison]]
==>[Suspicious_Store:MacDonalds,Count:1,Victims:[Madison]]
==>[Suspicious_Store:Soccer_for_the_City,Count:3,Victims:[Olivia,Marc]]
==>[Suspicious_Store:Just_Brew_It,Count:2,Victims:[Paul,Olivia]]
==>[Suspicious_Store:Amazon,Count:2,Victims:[Madison,Marc]]
==>[Suspicious_Store:Starbucks,Count:1,Victims:[Paul]]
==>[Suspicious_Store:Subway,Count:1,Victims:[Madison]]
==>[Suspicious_Store:American_Apparel,Count:1,Victims:[Marc]]
```


#  Scenario 2: Bank Fraud Detection Using Amazon Neptune

Graph databases can help to solve problems like fraud ring detection and other sophisticated fraud problems. The inherent capability of Graph databases to do perform entity link analysis on existing data in realtime. 

The sample dataset that we are referring here is consists of account holders and their contact information

![Fraud Ring Graph](https://github.com/abhmish/Neptune/blob/master/img/fraudring.png)

## Steps:

1. Bulk Load data into Neptune
2. Query data using Apache TinkerPop Gremlin


## 1. Bulk Load data into Neptune

Copy data from [https://github.com/abhmish/Neptune/tree/master/data](https://github.com/abhmish/Neptune/tree/master/data) into your s3 bucket

Neptune currently support connecting from EC2 instance inside VPC same as that of Neptune cluster. The cloudformation stack has created EC2 instance. 

Open terminal inside EC2 instance created and run following command in terminal

```
curl -X POST 
    -H 'Content-Type: application/json' 
    http://<your-neptune-endpoint>:8182/loader -d '
    { 
      "source" : "s3://<s3-bucket>/fraudring",
      "accessKey" : "",
      "secretKey" : "",
      "format" : "csv",
      "region" : "us-east-1",
      "failOnError" : "FALSE"
    }'
```

This loads three datasets into Neptune; 1. AccountHolders, 2. Contact Information for Account Holders

You can view the sample datasets at: s3://neptune101

The load command returns a load identifier which can be used to query the status of load request

`curl -X GET 'http://<your-neptune-endpoint>:8182/loader/<load-id>?details=true&errors=true&page=1&errorsPerPage=3'
`

For more information about loading data into Amazon Neptune visit: [https://docs.aws.amazon.com/neptune/latest/userguide/bulk-load.html](https://docs.aws.amazon.com/neptune/latest/userguide/bulk-load.html)


## 2. Query data using Apache TinkerPop Gremlin

We are going to use Apache TinkerPop Gremlin to query data. 

refer this to setup Gremlin console and connect to your Neptune cluster

[https://docs.aws.amazon.com/neptune/latest/userguide/access-graph-gremlin-console.html](https://docs.aws.amazon.com/neptune/latest/userguide/access-graph-gremlin-console.html)


* Find account holders who share more than one piece of legitimate contact information 

```
g.V().hasLabel("AccountHolder").as("ah").outE().inV().as("ci").
group().
by(__.select("ci").by()).
by(__.select("ah").by().fold()).
unfold().as("mapped").
project("FraudRing","ContactType","RingSize").
    by(__.select("mapped").select(values).unfold().as("ah1").select("ah1").coalesce(__.values("UniqueId")).fold()).
    by(__.select("mapped").select(keys).as("temp").select("temp").by(label)).
    by(__.select("mapped").select(values).unfold().count()).order().
by(select("RingSize"),decr).
where(__.select("RingSize").is(gt(1)))
```
```
==>[FraudRing:[MattSmith,JohnDoe,JaneAppleseed],ContactType:Address,RingSize:3]
==>[FraudRing:[MattSmith,JaneAppleseed],ContactType:SSN,RingSize:2]
==>[FraudRing:[JohnDoe,JaneAppleseed],ContactType:PhoneNumber,RingSize:2]
```

* Determine the financial risk of a possible fraud ring

```
g.V().hasLabel("AccountHolder").as("ah").outE().as("r").inV().as("ci").
group().
by(__.select("ci").by()).
by(__.select("ah").by().fold()).
unfold().as("mapped").project("FraudRing","ContactType","RingSize","FinancialRisk").
    by(__.select("mapped").select(values).unfold().as("ah1").select("ah1").coalesce(__.values("UniqueId")).fold()).
    by(__.select("mapped").select(keys).as("temp").select("temp").by(label)).
    by(__.select("mapped").select(values).unfold().count()).
    by(__.select('mapped').select(values).unfold().outE().dedup().
        or(__.hasLabel("HAS_CREDITCARD"),__.hasLabel("HAS_UNSECUREDLOAN")).dedup().
        choose(__.label()).
        option("HAS_CREDITCARD",__.inV().values("Limit")).
        option("HAS_UNSECUREDLOAN",__.inV().values("Balance")).sum()).
order().by(select("RingSize"),decr).
where(__.select('RingSize').is(gt(1)))

```
```
==>[FraudRing:[JohnDoe,JaneAppleseed,MattSmith],ContactType:Address,RingSize:3,FinancialRisk:34386]
==>[FraudRing:[JaneAppleseed,MattSmith],ContactType:SSN,RingSize:2,FinancialRisk:29386]
==>[FraudRing:[JaneAppleseed,JohnDoe],ContactType:PhoneNumber,RingSize:2,FinancialRisk:18045]

```
