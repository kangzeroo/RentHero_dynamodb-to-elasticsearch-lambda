# DynamoDB to ElasticSearch

This package allow you to easily ZIP a Lambda function and start processing DynamoDB Streams in order to index your DynamoDB objects in ElasticSearch.

It processes the following events:

   - INSERT
   - REMOVE
   - MODIFY

We force an index refresh for each event, for close to realtime indexing.

The DynamoDB JSON objects are unmarshaled and types are correctly converted. (Binary types have never been tested though)

List of numbers and strings are converted to list of Strings. ES doesn't allow arrays of different data types.

Blog reference explaining what we're doing here: https://aws.amazon.com/blogs/compute/indexing-amazon-dynamodb-content-with-amazon-elasticsearch-service-using-aws-lambda/

Unfortunatly, AWS removed the Lambda Blueprint doing what we are doing here. So we've done it ourself again.

# Get started

The deployment process is done through the Makefile. You will also need the following:

   - An S3 Bucket to put this lambda function
   - IAM role for the lambda function

You need to declare the following environment variables at the top of the Makefile. To get started:

   - AWS_BUCKET_CODE: Where your Lambda function will go in AWS S3
   - IAM_ROLE: The IAM role you created for the Lambda function to run with
   - ENV: The environment you're running: DEV, QA, PROD, STAGE, ... This is used to pull the correct config file from S3
   - PROFILE (optional): The AWS profile to use to deploy. Will use default profile by default ...

Obvisouly you need your AWS environment setup correctly.

## Kangzes Readme Portion
In order to use this lambda, you must first put the `dev_es_creds` in your S3 bucket and it has to be pointing to your Elasticsearch endpoint. Make sure the same ES endpoint is in `/lib/env.py`. Then run the `make create/...` command. You will find your S3 bucket populated with a copy of this lambda. You will also find a Lambda function that needs to be hooked up manually from your DynamoDB tables. Be sure to also set your IAM routes to allow dynamodb stream read access and list access to this lambda function. Finally, to check that your lambda is working look at its logs.

## Code Gotchas
Inside `src/DynamoToES/index.py` line 109 is the below code. Use this to modify your ES indices on creation (AWS ElasticSearch has many limitations, one of which is the inability to use the Indices REST api. Thus when we autogenerate indices via this lambda, we may define mappings here). This is useful for defining `geo_point` mapping types.

```
es.indices.create(table,
                      body='''{
                                "settings": {
                                    "index.mapping.coerce": true
                                },
                                "mappings": {
                                    "renthero_intents_hit": {
                                        "properties": {
                                            "GEO_POINT": { "type": "geo_point" }
                                        }
                                    }
                                }
                            }''')
```

## Create a config file

Create a simple file named `{ENV}_es_creds` and put the following in it:

```
ES_ENDPOINT='https://search-esclust-esclus-xxxxx-xxxxxx.{region}.es.amazonaws.com'
```

This file just contains the endpoint to your cluster. You should have one file per environment.

Upload it to your bucket where the Lambda function will also be uploaded.

Before Zipping the function, we will download that file locally and will inject it in the Lambda function as the file `lib/env.py`. The function depends on it and `import` this file.

That allows you to NOT hardcode your endpoint.
This way you can deploy several functions for different clusters and environments without touching the code.

Just set your environment variables `ENV` correctly, name your file right, upload it and that's it.

## Create the function

The first time you need to create the function in AWS Lambda.

```
make create/DynamoToES DESC="Process DynamoDB stream to ES"
```

This will download your config file from S3, install all the Python packages in the `build` folder, ZIP the whole thing, upload the ZIP file to S3 and create your Lambda function. The default is 128MB. You can change the memory of your function with the Makefile or in the console.

## Update the function

Let's say you make some changes to the code.

```
make deploy/DynamoToES DESC="Process DynamoDB stream to ES"
```

That will update the ZIP and refresh your Lambda function.

## Next

Now your Lambda function is created, head to AWS Lambda, find the function we just created and click `Triggers`.
Now add triggers for all the DynamoDB tables you want to process.

Your dynamoDB tables must have Dynamo Stream activated.

Check your CloudWatch logs to make sure your function processes things correctly!

enjoy

## Debugging

- If you are having trouble with `setup.cpg`, make sure you have an empty `/build` folder.<br/>
- If you want to add a VPC with private subnets, uncomment `vpc-config` in the `create` function of the Makefile.
