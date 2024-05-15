---
title: "Build a datapipeline on AWS with Pulumi"
date: 2024-05-13T00:00:00+00:00
weight: 3
# aliases: ["/first"]
tags: [
  "pulumi","aws","apigateway","cognito","firehose","redshift"
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
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
images:
  - "images/aws_ptemplate.png"
cover:
    image: "images/aws_ptemplate.png" # image path/url
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

To build an end to end infrastructure that will write the `event` sent via `http` from any application (game, e-commerce etc. can be considered) to `AWS Redshift`



Before we start you have to complete the following steps: 

- [https://www.pulumi.com/docs/clouds/aws/get-started/begin/](https://www.pulumi.com/docs/clouds/aws/get-started/begin/)
- [https://www.pulumi.com/docs/clouds/aws/get-started/create-project/](https://www.pulumi.com/docs/clouds/aws/get-started/create-project/)

So you will have pulumi project and stack.

## Development


> :memo: **Note:** Although I will be explaining this article on AWS, I have created a project to create a similar architecture
> with GCP to get the settings via yaml and I will explain it through this project.


- For project, you can reach [here](https://github.com/cemayan/pulumi-template)
- For config,  you can reach  [here](https://github.com/cemayan/pulumi-template/blob/main/configs/datapipeline/firehose/redshift/apigateway/config.yaml)

---

To build a data pipeline we need to endpoint for http requests.
We can create a Lambda Function or API Gateway for handling to HTTP requests.

> 💡 **Tip:**  In this article we can continue with `Amazon API Gateway` because you can connect with `Amazon Kinesis Firehose` directly


First we need to set up Roles via IAM:


### IAM

Before creating AWS Services, we need to create roles. We can define inline and assume policy on AWS, you can find these definitions in config.yaml.

```go
// ConfigureIAM configures the IAM role according to given values.
// You can create the multiple role.
func (a *Aws) ConfigureIAM() error {

	a.roles = map[string]*iam.Role{}

	for _, role := range a.config.Iam.Roles {

		args := &iam.RoleArgs{
			Name:                pulumi.String(role.Name),
			ForceDetachPolicies: pulumi.Bool(role.ForceDetachPolicies),
		}

		if role.AssumePolicy != "" {
			args.AssumeRolePolicy = pulumi.String(role.AssumePolicy)
		}

		if role.InlinePolicy != "" {
			args.InlinePolicies = iam.RoleInlinePolicyArray{
				&iam.RoleInlinePolicyArgs{
					Name:   pulumi.String(fmt.Sprintf("%s-inline-role", role.Name)),
					Policy: pulumi.String(role.InlinePolicy),
				},
			}
		}

		iamRole, err := iam.NewRole(a.ctx, role.Name, args)

		if strings.HasPrefix(strings.ToLower(role.Name), "api_gateway") {
			a.roles["apigateway"] = iamRole
		} else if strings.HasPrefix(strings.ToLower(role.Name), "kinesis_firehose") {
			a.roles["firehose"] = iamRole
		} else if strings.HasPrefix(strings.ToLower(role.Name), "redshift_service") {
			a.roles["redshift"] = iamRole
		} else if strings.HasPrefix(strings.ToLower(role.Name), "lambda_firehose") {
			a.roles["lambdafirehose"] = iamRole
		}

		if err != nil {
			a.ctx.Log.Error(err.Error(), nil)
			continue
		}
	}
	return nil
}
```

### Storage

When using Redshift, we can use S3 to put backups.

```go
// CreateStorage created S3 according to given values
// ForceDestroy may set the false if files are important.
func (a *Aws) CreateStorage() error {

	s3Bucket, err := s3.NewBucket(a.ctx, a.config.Storage.Name, &s3.BucketArgs{
		Bucket:       pulumi.String(a.config.Storage.Name), // "ptemplate-datapipeline-storage-redshift"
		ForceDestroy: pulumi.Bool(a.config.Storage.ForceDestroy), // true
	})

	a.s3Bucket = s3Bucket

	return err
}

```


### DWH

In order to create a `Redshift` cluster, we need to assign parameters like below. You can find sample values via config.yaml. 
After this process is over, we need a table where we will store the events, we need to run a `SQL` in it, we can do this job with `Statement.

```go
// CreateDWH creates Redshift cluster according to given values
// Initial SQL will be executed
func (a *Aws) CreateDWH() error {

	cluster, err := redshift.NewCluster(a.ctx, a.config.Dwh.Redshift.Identifier, &redshift.ClusterArgs{
		ClusterIdentifier:  pulumi.String(a.config.Dwh.Redshift.Identifier), //  "ptemplate-datapipeline-cluster"
		DatabaseName:       pulumi.String(a.config.Dwh.Redshift.DbName), // "ptemplatedb"
		MasterUsername:     pulumi.String(a.config.Dwh.Redshift.MasterUser), // "master"
		MasterPassword:     pulumi.String(a.config.Dwh.Redshift.MasterPass), // "Verysecretpass1!!"
		NodeType:           pulumi.String(a.config.Dwh.Redshift.NodeType), // "dc2.large"
		NumberOfNodes:      pulumi.Int(a.config.Dwh.Redshift.NumberOfNodes), // 1
		ClusterType:        pulumi.String(a.config.Dwh.Redshift.ClusterType), // "single-node"
		SkipFinalSnapshot:  pulumi.Bool(a.config.Dwh.Redshift.SkipSnapshot), // true
		PubliclyAccessible: pulumi.Bool(a.config.Dwh.Redshift.PublicAccess), // true
		IamRoles: pulumi.StringArray{
			a.roles["redshift"].Arn,
		},
	}, pulumi.DependsOn([]pulumi.Resource{a.roles["redshift"]}))

	a.redshift = cluster

	statement := &redshiftdata.StatementArgs{
		ClusterIdentifier: pulumi.String(a.config.Dwh.Redshift.Identifier), //  "ptemplate-datapipeline-cluster"
		Database:          pulumi.String(a.config.Dwh.Redshift.DbName), // "ptemplatedb"
		DbUser:            pulumi.String(a.config.Dwh.Redshift.MasterUser), //  "master"
		/*
		      drop table if exists events;
		      CREATE TABLE public.events(game_name varchar(100),event_name varchar(100),event_data SUPER);
		 */
		Sql:               pulumi.String(a.config.Dwh.Redshift.Sql),
	}

	newStatement, err := redshiftdata.NewStatement(a.ctx, "statement", statement, pulumi.DependsOn([]pulumi.Resource{cluster}))
	a.redshiftStatement = newStatement
	return err
}
```



### Stream

`Amazon Data Firehose` provides the easiest way to acquire, transform, and deliver data streams within seconds to data lakes, data warehouses, and analytics services.

The main reason why I chose `Amazon Kinesis Firehose` is that, as mentioned in the above sentence, we can transfer and process data very quickly.

`Amazon Kinesis Firehose` supports various destinations, below you can see the code block to write data to S3 and even a partition enabled version (:dark_sunglasses: Bonus), but in this article we will go through Redshift

```go
// CreateStream creates Kinesis Firehose according to given values
// You can set the destination such as "s3,redshift"
// Also you can set partition enabled config for S3.(You can set the prefix)
func (a *Aws) CreateStream() error {

	args := &kinesis.FirehoseDeliveryStreamArgs{
		Name: pulumi.String(a.config.Stream.Name),
	}

	resources := []pulumi.Resource{}

	if a.config.Stream.Destination == "s3" {
		s3ConfArgs := &kinesis.FirehoseDeliveryStreamExtendedS3ConfigurationArgs{
			RoleArn:           a.roles["firehose"].Arn,
			BucketArn:         a.s3Bucket.Arn,
			BufferingSize:     pulumi.IntPtr(a.config.Stream.S3Conf.BufferingSize),
			BufferingInterval: pulumi.IntPtr(a.config.Stream.S3Conf.BufferingInterval),
			CloudwatchLoggingOptions: kinesis.FirehoseDeliveryStreamExtendedS3ConfigurationCloudwatchLoggingOptionsArgs{
				Enabled:       pulumi.BoolPtr(true),
				LogGroupName:  pulumi.String(a.config.Stream.Name),
				LogStreamName: pulumi.String(fmt.Sprintf("%v-stream", a.config.Stream.Name)),
			},

			ErrorOutputPrefix: pulumi.String("errors/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/!{firehose:error-output-type}/"),
		}

		if a.config.Stream.S3Conf.PartitionEnabled {
			s3ConfArgs.Prefix = pulumi.String(a.config.Stream.S3Conf.S3Prefix)
			s3ConfArgs.DynamicPartitioningConfiguration = &kinesis.FirehoseDeliveryStreamExtendedS3ConfigurationDynamicPartitioningConfigurationArgs{
				Enabled: pulumi.Bool(true),
			}
			s3ConfArgs.ProcessingConfiguration = &kinesis.FirehoseDeliveryStreamExtendedS3ConfigurationProcessingConfigurationArgs{
				Enabled: pulumi.Bool(true),
				Processors: kinesis.FirehoseDeliveryStreamExtendedS3ConfigurationProcessingConfigurationProcessorArray{
					&kinesis.FirehoseDeliveryStreamExtendedS3ConfigurationProcessingConfigurationProcessorArgs{
						Type: pulumi.String("RecordDeAggregation"),
						Parameters: kinesis.FirehoseDeliveryStreamExtendedS3ConfigurationProcessingConfigurationProcessorParameterArray{
							&kinesis.FirehoseDeliveryStreamExtendedS3ConfigurationProcessingConfigurationProcessorParameterArgs{
								ParameterName:  pulumi.String("SubRecordType"),
								ParameterValue: pulumi.String("JSON"),
							},
						},
					},
					&kinesis.FirehoseDeliveryStreamExtendedS3ConfigurationProcessingConfigurationProcessorArgs{
						Type: pulumi.String("AppendDelimiterToRecord"),
					},
					&kinesis.FirehoseDeliveryStreamExtendedS3ConfigurationProcessingConfigurationProcessorArgs{
						Type: pulumi.String("MetadataExtraction"),
						Parameters: kinesis.FirehoseDeliveryStreamExtendedS3ConfigurationProcessingConfigurationProcessorParameterArray{
							&kinesis.FirehoseDeliveryStreamExtendedS3ConfigurationProcessingConfigurationProcessorParameterArgs{
								ParameterName:  pulumi.String("JsonParsingEngine"),
								ParameterValue: pulumi.String("JQ-1.6"),
							},
							&kinesis.FirehoseDeliveryStreamExtendedS3ConfigurationProcessingConfigurationProcessorParameterArgs{
								ParameterName:  pulumi.String("MetadataExtractionQuery"),
								ParameterValue: pulumi.String("{game_name:.game_name}"),
							},
						},
					},
				},
			}
		}

		args.Destination = pulumi.String("extended_s3")
		args.ExtendedS3Configuration = s3ConfArgs
		resources = append(resources, a.s3Bucket)
	} else if a.config.Stream.Destination == "redshift" {

		redshiftConf := &kinesis.FirehoseDeliveryStreamRedshiftConfigurationArgs{
			RoleArn: a.roles["firehose"].Arn,
			ClusterJdbcurl: pulumi.All(a.redshift.Endpoint, a.redshift.DatabaseName).ApplyT(func(_args []interface{}) (string, error) {
				endpoint := _args[0].(string)
				databaseName := _args[1].(string)
				return fmt.Sprintf("jdbc:redshift://%v/%v", endpoint, databaseName), nil
			}).(pulumi.StringOutput),
			Username: pulumi.String(a.config.Stream.RedshiftConf.Username), //  "master"
			CloudwatchLoggingOptions: &kinesis.FirehoseDeliveryStreamRedshiftConfigurationCloudwatchLoggingOptionsArgs{
				Enabled:       pulumi.Bool(true),
				LogStreamName: pulumi.String(fmt.Sprintf("%v-kinesis-stream", a.config.Stream.Name)), // "ptemplate-datapipeline-stream-redshift-kinesis-stream"
				LogGroupName:  pulumi.String(fmt.Sprintf("%v-kinesis-loggroup", a.config.Stream.Name)),  // "ptemplate-datapipeline-stream-redshift-kinesis-loggroup"
			},
			Password:      pulumi.String(a.config.Stream.RedshiftConf.Password), //  "Verysecretpass1!!"
			DataTableName: pulumi.String(a.config.Stream.RedshiftConf.DataTableName), //  "events"
			CopyOptions:   pulumi.String(a.config.Stream.RedshiftConf.CopyOptions), // "FORMAT JSON 'auto'"
			S3Configuration: &kinesis.FirehoseDeliveryStreamRedshiftConfigurationS3ConfigurationArgs{
				RoleArn:           a.roles["firehose"].Arn,
				BucketArn:         a.s3Bucket.Arn,
				BufferingSize:     pulumi.Int(10),
				BufferingInterval: pulumi.Int(0),
			},
		}

		args.RedshiftConfiguration = redshiftConf
		args.Destination = pulumi.String("redshift")
		resources = append(resources, a.redshift)
	}

	firehose_, err := kinesis.NewFirehoseDeliveryStream(a.ctx, a.config.Stream.Name, args, pulumi.DependsOn(resources))
	a.firehose = firehose_

	return err
}
```

### Identity Management

For security, we need to add an authentication method to the endpoint where we will send the events.

If we built this architecture in a private space instead of a public environment, we could easily handle this at the user scale with `AWS_IAM`.

But as I mentioned at the beginning of the article, we want to send events from within a game or a similar structure.

We can do this by creating a user pool with `Amazon Cognito` because we can select Cognito User Pool as Authorizer in `API Gateway`.

Since the password we will create for the user is a secret, we must set it as secret on pulumi

```shell
pulumi config set --secret 'config:userpass' ${SECRET} -s ${STACK_NAME}
```

:warning: According to config.yaml _username:_ <mark>cemayan</mark> _password:_ <mark>Secretpass1!</mark>



```go
// CreateIdentityManagement creates a identity platform on AWS with Cognito
// UserPoolClient and ManagedUserPoolClient will be created that given values
// In order to take id_token from Cognito you need to sign in first with defined user below (Click View hosted UI button on AWS or
// https://<your domain>/oauth2/authorize?response_type=code&client_id=<your app client id>&redirect_uri=<your callback url>)
// After that you are able to be request to endpoint that created by API Gateway.
func (a *Aws) CreateIdentityManagement() error {

	userPool, err := cognito.NewUserPool(a.ctx, a.config.Authorizer.UserPool.Name, &cognito.UserPoolArgs{
		Name: pulumi.String(a.config.Authorizer.UserPool.Name), // "user-pool-redshift"
	})

	conf := config.New(a.ctx, "config")
	userPass := conf.RequireSecret("userpass")

	_, err = cognito.NewUser(a.ctx, "user", &cognito.UserArgs{
		Enabled:    pulumi.Bool(true),
		Password:   userPass,
		UserPoolId: userPool.ID(),
		Username:   pulumi.String(a.config.Authorizer.UserPool.User.Username),  // "cemayan"
	}, pulumi.DependsOn([]pulumi.Resource{userPool}))

	userPoolDomain, err := cognito.NewUserPoolDomain(a.ctx, a.config.Authorizer.UserPool.UserDomain.Name, &cognito.UserPoolDomainArgs{
		Domain:     pulumi.String(a.config.Authorizer.UserPool.UserDomain.Name), //       name: "ptemplateauthredshift"
		UserPoolId: userPool.ID(),
	}, pulumi.DependsOn([]pulumi.Resource{userPool}))

	allowedScopes := pulumi.StringArray{}

	/*
	   allowed_scopes:
	    - email
	    - openid
	    - phone
	    - profile
	    - aws.cognito.signin.user.admin
	 */
	for _, v := range a.config.Authorizer.UserPool.UserClient.AllowedScopes {
		allowedScopes = append(allowedScopes, pulumi.String(v))
	}

	allowedFlows := pulumi.StringArray{}

	/*
	      allowed_flows:
	        - implicit
	 */
	for _, v := range a.config.Authorizer.UserPool.UserClient.AllowedFlows {
		allowedFlows = append(allowedFlows, pulumi.String(v))
	}

	callbackUrls := pulumi.StringArray{}

	/*
	      callback_urls:
	        - https://localhost:3000/
	        - https://localhost:3000
	 */
	for _, v := range a.config.Authorizer.UserPool.UserClient.CallbackUrls {
		callbackUrls = append(callbackUrls, pulumi.String(v))
	}

	exAuthFlows := pulumi.StringArray{}

	/*
	      ex_auth_flows:
	        - ALLOW_USER_SRP_AUTH
	        - ALLOW_USER_PASSWORD_AUTH
	        - ALLOW_REFRESH_TOKEN_AUTH
	 */
	for _, v := range a.config.Authorizer.UserPool.UserClient.ExplicitAuthFlows {
		exAuthFlows = append(exAuthFlows, pulumi.String(v))
	}

	userPoolClient, err := cognito.NewUserPoolClient(a.ctx, a.config.Authorizer.UserPool.UserClient.Name, &cognito.UserPoolClientArgs{
		Name:               pulumi.String(a.config.Authorizer.UserPool.UserClient.Name), //  "user-client-redshift"
		UserPoolId:         userPool.ID(),
		ExplicitAuthFlows:  exAuthFlows,
		CallbackUrls:       callbackUrls,
		AllowedOauthScopes: allowedScopes,
		AllowedOauthFlows:  allowedFlows,
		GenerateSecret:     pulumi.Bool(true),
	}, pulumi.DependsOn([]pulumi.Resource{userPool}))

	a.userPool = userPool

	_, err = cognito.NewManagedUserPoolClient(a.ctx, "managed", &cognito.ManagedUserPoolClientArgs{
		NamePattern:                pulumi.String(a.config.Authorizer.UserPool.UserClient.Name), //  "user-client-redshift"
		AllowedOauthFlows:          allowedFlows,
		AllowedOauthScopes:         allowedScopes,
		CallbackUrls:               callbackUrls,
		ExplicitAuthFlows:          exAuthFlows,
		SupportedIdentityProviders: pulumi.StringArray{pulumi.String("COGNITO")},
		UserPoolId:                 userPool.ID(),
	}, pulumi.DependsOn([]pulumi.Resource{userPool}))

	a.ctx.Export("CognitoUserPoolClientId", userPoolClient.ID())
	a.ctx.Export("CognitoUserPoolDomain", userPoolDomain.Domain)
	return err
}
```

### Api Gateway

Now that we have come to our last service, we need an endpoint for everything we have created to make sense. Amazon API Gateway allows us to integrate with most AWS services.

Here we will integrate with `Amazon Kinesis Firehose`.

We need to add the previously created `Cognito User Pool` as an _authorizer_.

For this we create a _POST_ method and to handle the _CORS_ event we create an _OPTIONS_ method.

After creating these, we map the payload of the incoming POST request to the request to be sent to `Amazon Kinesis Firehose`.


Thus, the request we will send to the “/events” endpoint will return a 401 status code if it does not have an Authentication header.


Example Payload:

```json
{
    "game_name": "myfirstgame",
    "event_name": "weapon.fired",
    "event_data": {"weapon_name":"m4a1"}
}
```


```go
// CreateApiGateway create API Gateway according to given values
// You can create multiple Routes
// Example can be found in configs/datapipeline/redshift/apigateway/config.yaml
func (a *Aws) CreateApiGateway() error {

	restApi, err := apigateway.NewRestApi(a.ctx, a.config.APIGateway.Name, &apigateway.RestApiArgs{
		Name: pulumi.String(a.config.APIGateway.Name),
	})

	a.restApi = restApi

	if a.userPool != nil {
		authorizer, _ := apigateway.NewAuthorizer(a.ctx, a.config.Authorizer.Name, &apigateway.AuthorizerArgs{
			RestApi: restApi,
			Name:    pulumi.String(a.config.Authorizer.Name), // ptemplate-authorizer-redshift"
			ProviderArns: pulumi.StringArray{
				a.userPool.Arn,
			},
			Type: pulumi.String(a.config.Authorizer.Type), // "COGNITO_USER_POOLS"
		}, pulumi.DependsOn([]pulumi.Resource{restApi, a.userPool}))

		a.authorizer = authorizer
	}

	resources := []pulumi.Resource{}
	resources = append(resources, restApi)

	for _, route := range a.config.APIGateway.Routes {

		resource, _ := apigateway.NewResource(a.ctx, route.Name, &apigateway.ResourceArgs{
			RestApi:  restApi.ID(),
			ParentId: restApi.RootResourceId,
			PathPart: pulumi.String(route.Name),
		}, pulumi.DependsOn([]pulumi.Resource{restApi}))

		for _, integration := range route.Integrations {
			var err error

			methodArgs := &apigateway.MethodArgs{
				RestApi:    restApi.ID(),
				ResourceId: resource.ID(),
				HttpMethod: pulumi.String(integration.Method.Type),
			}

			if integration.Method.Auth == "COGNITO_USER_POOLS" {
				methodArgs.AuthorizerId = a.authorizer.ID()
			}

			methodArgs.Authorization = pulumi.String(integration.Method.Auth)

			method, err := apigateway.NewMethod(a.ctx, integration.Method.Name, methodArgs, pulumi.DependsOn([]pulumi.Resource{restApi, resource, a.authorizer}))

			resources = append(resources, method)

			respParamMap := pulumi.BoolMap{}

			for _, respPar := range integration.Method.Response.ResponseParams {
				respParamMap[respPar.Key] = pulumi.Bool(respPar.Val)
			}

			methodResp, err := apigateway.NewMethodResponse(a.ctx, fmt.Sprintf("response_%v", integration.Method.Name), &apigateway.MethodResponseArgs{
				RestApi:            restApi.ID(),
				ResourceId:         resource.ID(),
				HttpMethod:         method.HttpMethod,
				StatusCode:         pulumi.String(integration.Method.Response.StatusCode),
				ResponseParameters: respParamMap,
			}, pulumi.DependsOn([]pulumi.Resource{restApi, resource}))

			integrationArgs := &apigateway.IntegrationArgs{
				RestApi:               restApi.ID(),
				ResourceId:            resource.ID(),
				HttpMethod:            method.HttpMethod,
				Type:                  pulumi.String(integration.Type),
				IntegrationHttpMethod: pulumi.String(integration.HTTPMethod),
				Credentials:           a.roles["apigateway"].Arn,
				Uri:                   pulumi.String(integration.URI),
			}

			if len(integration.ReqParams) > 0 {
				reqParamMap := pulumi.StringMap{}

				for _, respPar := range integration.ReqParams {
					reqParamMap[respPar.Key] = pulumi.String(respPar.Val)
				}
				integrationArgs.RequestParameters = reqParamMap
			}

			if len(integration.ReqTemplate) > 0 {
				reqTemplateMap := pulumi.StringMap{}

				for _, reqTemp := range integration.ReqTemplate {
					reqTemplateMap[reqTemp.Key] = pulumi.String(reqTemp.Val)
				}
				integrationArgs.RequestTemplates = reqTemplateMap
			}

			integrationDependsOn := []pulumi.Resource{}

			if strings.Contains(integration.URI, "firehose:action") {
				integrationDependsOn = append(integrationDependsOn, a.firehose)
			}

			_integration, err := apigateway.NewIntegration(a.ctx, integration.Name, integrationArgs,
				pulumi.DependsOn(integrationDependsOn))

			resources = append(resources, _integration)

			integrationResponseArgs := &apigateway.IntegrationResponseArgs{
				RestApi:    restApi.ID(),
				ResourceId: resource.ID(),
				HttpMethod: methodResp.HttpMethod,
				StatusCode: methodResp.StatusCode,
			}

			if len(integration.ResTemplate) > 0 {
				resTemplateMap := pulumi.StringMap{}

				for _, respTemp := range integration.ResTemplate {
					resTemplateMap[respTemp.Key] = pulumi.String(respTemp.Val)
				}

				integrationResponseArgs.ResponseTemplates = resTemplateMap
			}

			if len(integration.ResParams) > 0 {
				resParamMap := pulumi.StringMap{}

				for _, respParam := range integration.ResParams {
					resParamMap[respParam.Key] = pulumi.String(respParam.Val)
				}

				integrationResponseArgs.ResponseParameters = resParamMap
			}

			integrationResp, err := apigateway.NewIntegrationResponse(a.ctx,
				fmt.Sprintf("integration_%v_response", integration.Method.Name),
				integrationResponseArgs, pulumi.DependsOn([]pulumi.Resource{_integration}))

			resources = append(resources, integrationResp)

			if err != nil {
				a.ctx.Log.Error(err.Error(), nil)
			}
		}
	}

	deployment, err := apigateway.NewDeployment(a.ctx, fmt.Sprintf("deploymentResource%v", a.config.APIGateway.DeploymentId), &apigateway.DeploymentArgs{
		RestApi:   restApi.ID(),
		StageName: pulumi.String(a.config.APIGateway.Stage), // "dev"
	}, pulumi.DependsOn(resources))

	a.ctx.Export("apiGatewayUrl", deployment.InvokeUrl)

	return err
}
```

## Testing

---

If you want to test it, you can use the following command after pulling the above project to your local machine.


```shell
make select-stack stack=datapipeline-firehose-redshift-apigateway
make up stack=datapipeline-firehose-redshift-apigateway
```

---

