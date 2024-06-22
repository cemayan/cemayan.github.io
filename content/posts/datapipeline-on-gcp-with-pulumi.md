---
title: "Build a datapipeline on GCP with Pulumi"
date: 2024-05-13T00:00:00+00:00
weight: 5
# aliases: ["/first"]
tags: [
  "pulumi","gcp","apigateway","bigquery","cloudfunction","cloudrun","pubsub"
]
author: "cemayan"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
disableShare: false
disableHLJS: false
hideSummary: false
UseHugoToc: true
images:
  - "images/gcp_ptemplate.png"
cover:
  image: "images/gcp_ptemplate.png" # image path/url
  alt: "architecture" # alt text
  caption: "architecture" # display caption under cover
editPost:
    URL: "https://github.com/cemayan/cemayan.github.io/tree/main/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---



## Introduction

#### About Pulumi: 

*Build and ship infrastructure faster using languages you know and love. Use Pulumi’s open source SDK to provision 
infrastructure on any cloud, and securely and collaboratively build and manage infrastructure using Pulumi Cloud.*

#### Why Pulumi?

To be able to develop in a programming language we love using the features of that language.

#### Aim of Article

To build an end to end infrastructure that will write the `event` sent via `http` from any application (game, e-commerce etc. can be considered) to `BigQuery`

---

Before we start you have to complete the following steps: 

- [https://www.pulumi.com/docs/clouds/gcp/get-started/begin/](https://www.pulumi.com/docs/clouds/gcp/get-started/begin/)
- [https://www.pulumi.com/docs/clouds/gcp/get-started/create-project/](https://www.pulumi.com/docs/clouds/gcp/get-started/create-project/)

So you will have pulumi project and stack.

## Development


> :memo: **Note:** Although I will be explaining this article on GCP, I have created a project to create a similar architecture
> with AWS to get the settings via yaml and I will explain it through this project.


- For project, you can reach [here](https://github.com/cemayan/pulumi-template)
- For config,  you can reach  [here](https://github.com/cemayan/pulumi-template/blob/main/configs/datapipeline/pubsub/bigquery/apigateway/config.yaml)

---

To build a data pipeline we need to endpoint for http requests.
We can create a Cloud Function or API Gateway for handling to HTTP requests.

> 💡 **Tip:**  In this article we can continue with `Google Cloud API Gateway` 


First we need to set up Storage:

### Storage

We need a function to write data to a topic in _Google Cloud Pubsub_, we can get zip file from the _Google Cloud Storage Bucket_.

it can be reach out from this [link](https://github.com/cemayan/pulumi-template/tree/main/functions/gcp/pubsubproducer) to Cloud function source code.
With this code it can be sent data to `Google Cloud Pubsub`

In order to create a zip file from source code you can run following code:

```shell
make function-zip
```

```go
// createBucketForFunction creates bucket for function
// it is used to get zipped function
func (g *Gcp) createBucketForFunction() (*storage.Bucket, *storage.BucketObject, error) {
    
    bucket, err := storage.NewBucket(g.ctx, g.config.Function.Build.Source.Storage.Bucket.Name, &storage.BucketArgs{
    Name:                     pulumi.String(g.config.Function.Build.Source.Storage.Bucket.Name), // "ptemplate-gcf-source"
    Location:                 pulumi.String(g.region), // "europe-west3"
    UniformBucketLevelAccess: pulumi.Bool(true),
    ForceDestroy:             pulumi.Bool(g.config.Function.Build.Source.Storage.ForceDestroy), // true
    })
    
    g.functionSourceBucket = bucket
    
    object, err := storage.NewBucketObject(g.ctx, g.config.Function.Build.Source.Storage.Bucket.Object.Name, &storage.BucketObjectArgs{
    Name:   pulumi.String(g.config.Function.Build.Source.Storage.Bucket.Object.Name), // "function.zip"
    Bucket: bucket.Name,
    Source: pulumi.NewFileAsset(g.config.Function.Build.Source.Storage.Bucket.Object.Path), // "assets/cloudfunction/function.zip"
    }, pulumi.DependsOn([]pulumi.Resource{bucket}))
    
    g.functionSourceBucketObj = object
    
    return bucket, object, err
}
```


### DWH

In order to create a `Bigquery` cluster, we need to assign parameters like below.You can find sample values via config.yaml.
After this process is over, we need a table where we will store the events, we need to create a schema.


```go
// CreateDWH creates BigQuery Dataset and Table according to given values
// Initial schema will be created
func (g *Gcp) CreateDWH() error {

	dataset, err := bigquery.NewDataset(g.ctx, g.config.Dwh.BigQuery.Dataset, &bigquery.DatasetArgs{
		DatasetId: pulumi.String(g.config.Dwh.BigQuery.Dataset), // "ptemplate_dataset"
		Location:  pulumi.String(g.region), // "europe-west2"
	})

	table, err := bigquery.NewTable(g.ctx, g.config.Dwh.BigQuery.TableId, &bigquery.TableArgs{
		DeletionProtection: pulumi.Bool(g.config.Dwh.BigQuery.DeletionProtection), // false
		TableId:            pulumi.String(g.config.Dwh.BigQuery.TableId), //  "ptemplate_events"
		DatasetId:          dataset.DatasetId,
        /*
           [
             {
               "name": "game_name",
               "type": "STRING",
               "mode": "NULLABLE"
             },
             {
               "name": "event_name",
               "type": "STRING",
               "mode": "NULLABLE"
             },
             {
               "name": "event_data",
               "type": "JSON",
               "mode": "NULLABLE"
             }
           ]
        */
		Schema:             pulumi.String(g.config.Dwh.BigQuery.Schema),
	})

	g.table = table

	return err
}
```


### Stream

With `Google Cloud Pubsub`, we can create a topic and write data in it and then create a subscription to transfer this data to a Google Cloud service.
Below you can find how to create subscriptions for `Google Cloud Storage` and `Bigquery`

```go
// CreateStream create Pubsub topic and subscription according to given values
// You can set the destination such as "cloudstorage,bigquery"
func (g *Gcp) CreateStream() error {

	var err error

	topic, err := pubsub.NewTopic(g.ctx, g.config.Stream.PubSubConf.Topic.Name, &pubsub.TopicArgs{
		Name: pulumi.String(g.config.Stream.PubSubConf.Topic.Name), //  "events_topic"
	})

	g.topic = topic

	subsArgs := &pubsub.SubscriptionArgs{
		Name:               pulumi.String(g.config.Stream.PubSubConf.Subscription.Name), //  "subscription"
		Topic:              topic.ID(),
		AckDeadlineSeconds: pulumi.Int(20),
	}

	resources := []pulumi.Resource{}

	if g.config.Stream.Destination == "cloudstorage" {
		subsArgs.CloudStorageConfig = &pubsub.SubscriptionCloudStorageConfigArgs{
			Bucket:      g.bucket.ID(),
			MaxDuration: pulumi.String(g.config.Stream.PubSubConf.Subscription.CloudStorageConf.Duration),
		}
	} else if g.config.Stream.Destination == "bigquery" {
		subsArgs.BigqueryConfig = &pubsub.SubscriptionBigqueryConfigArgs{
			Table: pulumi.All(g.table.Project, g.table.DatasetId, g.table.TableId).ApplyT(func(_args []interface{}) (string, error) {
				project := _args[0].(string)
				datasetId := _args[1].(string)
				tableId := _args[2].(string)
				return fmt.Sprintf("%v.%v.%v", project, datasetId, tableId), nil
			}).(pulumi.StringOutput),
			UseTableSchema: pulumi.Bool(true),
		}

		resources = append(resources, g.table)
	}

	_, err = pubsub.NewSubscription(g.ctx, g.config.Stream.PubSubConf.Subscription.Name, subsArgs, pulumi.DependsOn(resources))

	return err
}
```


### Function

Cloudfunctionsv2 service needs a service account, so we create it first.
It shows our zip file in the bucket as the source.


Requests coming through the `API Gateway` are sent directly to the `Cloud Run` endpoint.

> 💡 cloudfunctionsv2 is new version for cloudfunctions You can check the [link]( https://cloud.google.com/functions/docs/concepts/version-comparison)



```go

// createServiceAccount creates a service account for cloud function
func (g *Gcp) createServiceAccount() (*serviceaccount.Account, error) {

    account, err := serviceaccount.NewAccount(g.ctx, g.config.Iam.ServiceAcc.AccountID, &serviceaccount.AccountArgs{
    AccountId:                 pulumi.String(g.config.Iam.ServiceAcc.AccountID), //  "ptemplate-svc-acc"
    DisplayName:               pulumi.String(g.config.Iam.ServiceAcc.DisplayName), //  "Ptemplate Service Account"
    Project:                   pulumi.String(g.config.Iam.ServiceAcc.Project), // "pulumitemplate"
    CreateIgnoreAlreadyExists: pulumi.Bool(true),
    })
    
    g.serviceAcc = account
    
    return account, err
}



// CreateFunction creates Cloud function/Cloud Run according to given values
// You can upload zip to lambda
// If lambda function creation operation is successful URL will be exported.
// cloudfunctionsv2 is new version for cloudfunctions You can check the link below
// https://cloud.google.com/functions/docs/concepts/version-comparison
func (g *Gcp) CreateFunction() error {

	if g.config.Iam.ServiceAcc != nil {
		_, _ = g.createServiceAccount()
	}

	buildArgs := &cloudfunctionsv2.FunctionBuildConfigArgs{
		Runtime:    pulumi.String(g.config.Function.Build.Runtime), //  "go122"
		EntryPoint: pulumi.String(g.config.Function.Build.EntryPoint), // "PubsubProducer"
	}

	funcArgs := &cloudfunctionsv2.FunctionArgs{
		Name:        pulumi.String(g.config.Function.Name), //  ptemplate-proxy
		Location:    pulumi.String(g.region), // "europe-west2"
		Project:     pulumi.String(g.project), // "pulumitemplate"
		BuildConfig: buildArgs,

		ServiceConfig: &cloudfunctionsv2.FunctionServiceConfigArgs{
			MaxInstanceCount:    pulumi.Int(g.config.Function.ServiceConf.MaxInstance), // 1
			AvailableMemory:     pulumi.String(g.config.Function.ServiceConf.AvailableMem), // "256M"
			TimeoutSeconds:      pulumi.Int(g.config.Function.ServiceConf.Timeout), // 60
			ServiceAccountEmail: g.serviceAcc.Email, 
			EnvironmentVariables: pulumi.StringMap{
				"PROJECT_ID": pulumi.String(g.project), // "pulumitemplate"
				"TOPIC_ID":   pulumi.String(g.config.Stream.PubSubConf.Topic.Name)}, // "events_topic"
		},
	}

	if g.config.Function.Trigger != nil {

		triggerArgs := &cloudfunctionsv2.FunctionEventTriggerArgs{
			EventType:     pulumi.String(g.config.Function.Trigger.EventType),
			RetryPolicy:   pulumi.String("RETRY_POLICY_RETRY"),
			TriggerRegion: pulumi.String(g.config.Function.Trigger.Region),
		}

		if g.config.Template.Name == "data-pipeline" {
			triggerArgs.PubsubTopic = pulumi.String(g.config.Function.Trigger.EventType)
		}

		funcArgs.EventTrigger = triggerArgs
	}

	if g.config.Function.Build.Source != nil {
		bucket, object, _ := g.createBucketForFunction()

		buildArgs.Source = &cloudfunctionsv2.FunctionBuildConfigSourceArgs{
			StorageSource: &cloudfunctionsv2.FunctionBuildConfigSourceStorageSourceArgs{
				Bucket: bucket.Name,
				Object: object.Name,
			},
		}
	}

	function, err := cloudfunctionsv2.NewFunction(g.ctx, g.config.Function.Name, funcArgs, pulumi.DependsOn([]pulumi.Resource{g.functionSourceBucket, g.functionSourceBucketObj, g.serviceAcc}))

	g.function = function

	g.configureRolesForFunction()

	g.ctx.Export("function_url", function.Url)

	return err
}

```



### IAM

Creating “bindings” and “members” needed by services on GCP

```go
// configureRolesForFunction creates/binds a member according to given values.
func (g *Gcp) configureRolesForFunction() error {

	var err error

	for _, role := range g.config.Iam.Roles {
		if role.Type == "cloudfuncv2member" {
			_, err = cloudfunctionsv2.NewFunctionIamMember(g.ctx, role.Name, &cloudfunctionsv2.FunctionIamMemberArgs{
				Project:       pulumi.String(g.project), // "pulumitemplate"
				Location:      pulumi.String(g.region), // "europe-west2"
				CloudFunction: pulumi.String(g.config.Function.Name), // ptemplate-proxy
				Role:          pulumi.String(role.Role),
				Member: g.serviceAcc.Email.ApplyT(func(email string) (string, error) {
					return fmt.Sprintf("serviceAccount:%v", email), nil
				}).(pulumi.StringOutput),
			}, pulumi.DependsOn([]pulumi.Resource{g.function}))
		} else if role.Type == "pubsubmember" {
			_, err = pubsub.NewTopicIAMMember(g.ctx, role.Name, &pubsub.TopicIAMMemberArgs{
				Project: pulumi.String(g.project), // "pulumitemplate"
				Topic:   pulumi.String(g.config.Stream.PubSubConf.Topic.Name), // "events_topic"
				Role:    pulumi.String(role.Role),
				Member: g.serviceAcc.Email.ApplyT(func(email string) (string, error) {
					return fmt.Sprintf("serviceAccount:%v", email), nil
				}).(pulumi.StringOutput),
			}, pulumi.DependsOn([]pulumi.Resource{g.topic}))
		} else if role.Type == "cloudrunbinding" {
			_, err = cloudrun.NewIamBinding(g.ctx, role.Name, &cloudrun.IamBindingArgs{
				Project:  pulumi.String(g.project),
				Service:  pulumi.String(g.config.Function.Name), // ptemplate-proxy
				Location: pulumi.String(g.region),  // "europe-west2"
				Role:     pulumi.String(role.Role),
				Members: pulumi.StringArray{
					g.serviceAcc.Email.ApplyT(func(email string) (string, error) {
						return fmt.Sprintf("serviceAccount:%v", email), nil
					}).(pulumi.StringOutput),
				},
			}, pulumi.DependsOn([]pulumi.Resource{g.function}))
		}
	}

	return err
}

```


```go

// ConfigureIAM configures the IAM member according to given values.
// You can create the multiple role.
func (g *Gcp) ConfigureIAM() error {
	project, _ := organizations.LookupProject(g.ctx, &organizations.LookupProjectArgs{ProjectId: pulumi.StringRef(g.config.Iam.ServiceAcc.Project)}, nil)

	var err error

	for _, role := range g.config.Iam.Roles {

		if role.Type == "bucketmember" {
			_, _ = storage.NewBucketIAMMember(g.ctx, role.Name, &storage.BucketIAMMemberArgs{
				Bucket: g.bucket.Name,
				Role:   pulumi.String(role.Role),
				Member: pulumi.String(fmt.Sprintf(role.Member, project.Number)),
			}, pulumi.DependsOn([]pulumi.Resource{g.bucket}))
		} else if role.Type == "projectmember" {
			_, _ = projects.NewIAMMember(g.ctx, role.Name, &projects.IAMMemberArgs{
				Project: pulumi.String(*project.ProjectId),
				Role:    pulumi.String(role.Role),
				Member:  pulumi.String(fmt.Sprintf(role.Member, project.Number)),
			})
		}
	}

	return err
}

```

### Identity Platform

You can create a Oauth client on this [link](https://console.cloud.google.com/apis/credentials)
After that you need create a  Google provider for authentication. You can add provider on this [link](https://console.cloud.google.com/customer-identity/providers)


### Api Gateway

To create a gateway in Google Cloud Api Gateway service, we need an Openapi schema, we generate this schema according to 
the data in config.yaml and then show it as a spec file.

`spec.Paths.Event.Post.XGoogleBackend.Address` represents the Cloud run/function address
`spec.SecurityDefinitions.GoogleIDToken.XGoogleIssuer` represents Google provider for authentication
`spec.SecurityDefinitions.GoogleIDToken.XGoogleAudiences`represents Google Cloud Oauth client

After this step it will be created new endpoint what we defined.

And then we log in with Google and add the incoming id to the Authentication header when sending a request and ensure security.

In order to get a token in the development environment, you can get it by logging in with your Google account by serving the following html somewhere.


```html
<html>
<body>
<script src="https://accounts.google.com/gsi/client" async defer></script>
<script>
  function handleCredentialResponse(response) {
    console.log(response.credential);
  }
</script>
<div id="g_id_onload"
     data-client_id="yourid.apps.googleusercontent.com"
     data-callback="handleCredentialResponse">
</div>
<div class="g_id_signin" data-type="standard"></div>
</body>

</html>


```


```go
// generateSpecFile generates a yaml file according to given values
// Since oauth2 client cannot be created by Gcloud CLI we need to create Oauth2 Client first
// With this yaml you are able to use Google Authentication while using the function URL
func (g *Gcp) generateSpecFile() error {

	spec := types.OpenApiSpec{}
	spec.Swagger = "2.0"
	spec.Info.Title = g.config.APIGateway.Name
	spec.Info.Version = "1.0.0"

	spec.SecurityDefinitions.GoogleIDToken.AuthorizationURL = ""
	spec.SecurityDefinitions.GoogleIDToken.Flow = "implicit"
	spec.SecurityDefinitions.GoogleIDToken.Type = "oauth2"
	spec.SecurityDefinitions.GoogleIDToken.XGoogleIssuer = "https://accounts.google.com"
	spec.SecurityDefinitions.GoogleIDToken.XGoogleJwksURI = "https://www.googleapis.com/oauth2/v3/certs"
	spec.SecurityDefinitions.GoogleIDToken.XGoogleAudiences = g.config.Idp.ClientId

	spec.Schemes = []string{"https"}
	spec.Produces = []string{"application/json"}

	spec.Paths.Event.Post.OperationID = fmt.Sprintf("%v-op", g.config.APIGateway.Name) // "ptemplate-api-op"

	spec.Paths.Event.Post.XGoogleBackend.Address = fmt.Sprintf("https://%v-%v.cloudfunctions.net/%v", g.config.Function.Region, g.config.Iam.ServiceAcc.Project, g.config.Function.Name)

	securities := []types.Security{}

	securities = append(securities, types.Security{GoogleIDToken: make([]interface{}, 0)})

	spec.Paths.Event.Post.Security = securities
	spec.Paths.Event.Post.Responses.Num200.Description = "OK"

	yamlFile, _ := yaml.Marshal(&spec)

	err := os.MkdirAll("api/gcp", 0700)
	err = os.WriteFile("api/gcp/api.yaml", yamlFile, 0644)
	return err
}

// CreateApiGateway creates Api and Gateway according to given values
// Before create a Gateway you need to generate spec file
func (g *Gcp) CreateApiGateway() error {

	err := g.generateSpecFile()

	apiGw, err := apigateway.NewApi(g.ctx, g.config.APIGateway.Name, &apigateway.ApiArgs{
		ApiId: pulumi.String(g.config.APIGateway.Name), // "ptemplate-api"
	}, pulumi.DependsOn([]pulumi.Resource{g.function}))
	if err != nil {
		return err
	}
	invokeFilebase64, err := std.Filebase64(g.ctx, &std.Filebase64Args{
		Input: g.config.APIGateway.OpenApiSpec, // "api/gcp/api.yaml"
	}, nil)

	apiGwApiConfig, err := apigateway.NewApiConfig(g.ctx, fmt.Sprintf("%v-config", g.config.APIGateway.Name), &apigateway.ApiConfigArgs{
		Api:         apiGw.ApiId,
		ApiConfigId: pulumi.String(fmt.Sprintf("%v-config", g.config.APIGateway.Name)),
		OpenapiDocuments: apigateway.ApiConfigOpenapiDocumentArray{
			&apigateway.ApiConfigOpenapiDocumentArgs{
				Document: &apigateway.ApiConfigOpenapiDocumentDocumentArgs{
					Path:     pulumi.String(g.config.APIGateway.OpenApiSpec), // "api/gcp/api.yaml"
					Contents: pulumi.String(invokeFilebase64.Result),
				},
			},
		},
	}, pulumi.DependsOn([]pulumi.Resource{apiGw}))

	_, err = apigateway.NewGateway(g.ctx, fmt.Sprintf("%v-gw", g.config.APIGateway.Name), &apigateway.GatewayArgs{
		ApiConfig: apiGwApiConfig.ID(),
		GatewayId: pulumi.String(fmt.Sprintf("%v-gw", g.config.APIGateway.Name)), // "ptemplate-api-gw"
		Region:    pulumi.String(g.config.APIGateway.Region), // "europe-west2"
	}, pulumi.DependsOn([]pulumi.Resource{apiGwApiConfig}))
""
	return err
}
```


### Testing

If you want to test it, you can use the following command after pulling the above project to your local machine.

```shell
make select-stack stack=datapipeline-pubsub-bigquery-apigateway
make up stack=datapipeline-pubsub-bigquery-apigateway
```
