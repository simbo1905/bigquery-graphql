
# A Demo Of GraphQL-Java Over BigQuery deployed onto KNative on GKE

This codebase is based on the tutorial [getting-started-with-spring-boot](https://www.graphql-java.com/tutorials/getting-started-with-spring-boot/).

Note: You probably want to use the Quarkus version of this codebase which is at 
[simbo1905/quarkus-graphql-bigquery](https://github.com/simbo1905/quarkus-graphql-bigquery). That 
version of the code is none blocking whereas this version of the code blocks on BigQuery 
which will harm scalability. 

This codebase uses a TTL Guava cache to cache the entities returned from BigTable. 

Exactly like the original tutorial if we query with:

```graphql
{
  bookById(id:"book-1"){
    id
    name
    pageCount
    author {
      firstName
      lastName
    }
  }
}
```

Then we get back: 

```json
{
  "data": {
    "book1": {
      "id": "book-1",
      "name": "Harry Potter and the Philosopher's Stone",
      "pageCount": 223,
      "author": {
        "firstName": "Joanne",
        "lastName": "Rowling"
      }
    }
  }
}
```

You can run GraphQL Playbround to try this out: 

![GraphQL Playground](https://raw.githubusercontent.com/simbo1905/bigquery-graphql/master/graphql-bigquery.png)

The GraphQL scheme is:

```graphql
directive @cache(ms : Int!) on FIELD_DEFINITION

type Query {
    bookById(id: ID): Book @cache(ms: 15000)
}

type Book {
    id: ID
    name: String
    pageCount: Int
    author: Author @cache(ms: 15000)
}

type Author {
    id: ID
    firstName: String
    lastName: String
}
```

The `@cache` directive causes the BigQuery query-by-id results to be cached in a per field in-memory cache 
with a TTL of the specified number of milliseconds. 

The matching BigQuery schema is:

```sql
create table demo_graphql_java.book ( id string, name string, pageCount string, authorId string ); 
create table demo_graphql_java.author ( id string, firstName string, lastName string );  
```

This codebase uses a generic file to `wirings.json` to map GraphQL onto BigQuery SQL. If we look in that file we have:

```json
[
  {
    "typeName": "Query",
    "fieldName": "bookById",
    "sql":"select id,name,pageCount,authorId from demo_graphql_java.book where id=@id",
    "mapperCsv":"id,name,pageCount,authorId",
    "gqlAttr": "id",
    "sqlParam": "id"
  },
  {
    "typeName": "Book",
    "fieldName": "author",
    "sql":"select id,firstName,lastName from demo_graphql_java.author where id=@id",
    "mapperCsv":"id,firstName,lastName",
    "gqlAttr": "authorId",
    "sqlParam": "id"
  }
]
```

That contains two wiring: 

 1. There is a field on `Query` called `bookById` which fines our top level query:
    * The graphql source parameter/attribute is `id` as we can query by e.g., `bookById(id:"book-1")`
    * The sql query named parameter is also `id` as that is the identity column on the book table. 
    * The sql query is a simple select-by-id that uses the sql param i.e., `where id=@id`
    * The list of fields returned by the query is named in `mapperCsv`. We have to pass this as BigQuery won't tell us this fact unlike a standard JDBC ResultSet :cry:.
 2. There is a field on `Book` called `author` which requires querying the author table based on the `authorId` of the book:
    * The graphql source parameter/attribute is `authorId` as this is the name of the attribute on the `Book` entity as loaded from BigQuery.
    * The sql query named parameter is `id` as that is also the column name of the identity column on the author table.
    * The sql query is also a simple select-by-id using `where id=@id`
    * Once again the list of the fields returned by the query is supplied as BigQuery doesn't provide that :cry:.

## TODO Development

At the moment the code assumes that all SQL query parameters are strings. 
At the moment the code assumes that all SQL queries are by an attribute named `id`. 
It is left as an exercise to the reader to upgrade the code to deal with other types and parameter names. 

## BigQuery Setup

On the Google Cloud console: 

 1. Create a dataset named `demo_graphql_java` see [here](https://cloud.google.com/bigquery/docs/datasets).
 2. Run `create_tables.sql` using the BigQuery console. 

First create the tables in a dataset `demo_graphql_java` as described above.

Create a service account `bigquery-graphql` then grant it the bigquery user role:

```sh
gcloud projects add-iam-policy-binding ${YOUR_PROJECT} \
  --member="serviceAccount:bigquery-graphql@${YOUR_PROJECT}.iam.gserviceaccount.com" \
  --role="roles/bigquery.user"
```

Grant the service account read to the two tables using these instructions [bigquery/docs/table-access-controls](https://cloud.google.com/bigquery/docs/table-access-controls#bq).

In *my* case I did something like this. YMMV you need to change the identifiers to match your project/sa:

```sh
$ cat policy.json
{
"bindings": [
 {
   "members": [
     "serviceAccount:bigquery-graphql@capable-conduit-300818.iam.gserviceaccount.com"
   ],
   "role": "roles/bigquery.dataViewer"
 }
]
}
$ bq set-iam-policy capable-conduit-300818:demo_graphql_java.book policy.json
$ bq set-iam-policy capable-conduit-300818:demo_graphql_java.author policy.json
```

On the console create a JSON keyfile for the service account and save it in the current directory.
Save the file name as "bigquery-sa.json".

## Running IntelliJ

You need to run the BigQueryGraphQLApplication then edit the config to:

  1. Set and Environment Variable `GOOGLE_APPLICATION_CREDENTIALS=bigquery-sa.json` where that is a json keyfile for a 
     valid service acccount with the valid permissions. 

## Running In Docker

Use maven to build the docker image (you will have to change the pom.xml to user your hub.docker.com username):

```shell
mvn package
```

Then run Docker passing in access to that keyfile: 

```sh
docker run -it \
  --volume $(pwd):/home/project \
  -e GOOGLE_APPLICATION_CREDENTIALS=/home/project/bigquery-sa.json \
  -p 8080:8080 simonmassey/bigquery-graphql:latest
```

## Docker Release Via GitHub Actions

Is done as per https://gist.github.com/simbo1905/495400fb708fdc91b75ce9b1bc4666cc

## Run On KNative

Do the first helloworld deployment where an early version is shown in the video [Serverless with Knative - Mete Atamel](https://www.youtube.com/watch?v=HiIJqMqFbC0).
**Note** Use the latest actual set up docs and scripts at [https://github.com/meteatamel/knative-tutorial/tree/master/setup](https://github.com/meteatamel/knative-tutorial/tree/master/setup) 
and you *only* need to setup Knative Serving and not anything else. 

We need to create a secret containing your service account token. Using [these instructions](https://knative.dev/docs/serving/samples/secrets-go/) 
I found that this worked:

```sh
kubectl create secret generic graphql-bigquery --from-file=bigquery-sa.json
```

The secret is referenced in service-v1.yaml which is the knative service you can install with: 

```sh
kubectl apply -f service-v1.yaml
```

You can `watch` all the config being deployed using:

```sh
watch -n 1 kubectl get pods,ksvc,configuration,revision,route
```

You can list and delete with:

```sh
kn service list
kn service delete $SVC
```

## Manage KNative via Helm and Helmfile

Grab the latest helm and put it on your path. Then install the KNative service with: 

```sh
helm install bigquery-graphql ./bigquery-graphql
```

You can uninstall it with: 

```sh
helm uninstall bigquery-graphql
```

Better yet use [github pages](https://docs.github.com/en/free-pro-team@latest/github/working-with-github-pages/creating-a-github-pages-site) and create a chart repo: 

```sh
helm package bigquery-graphql
mv bigquery-graphql-1.0.4.tgz charts/
helm repo index charts --url https://${YOUR_ORG}.github.io/bigquery-graphql/charts
git add charts/*
git commit -am 'charts update'
git pull && git push
```

Now you can use the declarative `helmfile.yaml` to update all the services with: 

```sh
helmfile sync
```

Note that my helmfile.yaml sets the env var `spring_profiles_active: gcp` to enable 
stackdrive logging. If you are not running on Google Cloud Platform with stackdriver
logging enabled that will give errors. You can disable it by setting the profile to be 
`default`. 
