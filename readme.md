# <center>Sharing resources between AWS and GCP using Istio</center>

## Table of Contents
* [Introduction](#introduction)
* [1 - Create the lambda function](#lab-1---create-lambda-function-)
* [2 - Deploy Istio in an EKS cluster](#lab-2---deploy-istio-eks-)

## Introduction <a name="introduction"></a>

In the modernization journey to cloud-native architectures, you will eventually find that a multicloud architecture is a natural step.

Cost saving, vendor lock-in, versatility, HA, name your reason, this is a reality that we see more and more from the experience helping customers in production.

In this blog post we will learn how to make an AWS lambda function available to our applications running in GCP, using different Istio meshes and securing the traffic over the wire.


## 1 - Create the lambda function<a name="lab-1---create-lambda-function-"></a>

We are going to create a lambda function and expose it with a [private ALB](https://docs.aws.amazon.com/lambda/latest/dg/services-alb.html), only available inside the AWS VPC.

![img1.png](images/img1.png)

The code for the lambda is really simple, you can even deploy it directly from [this repo](https://serverlessrepo.aws.amazon.com/applications/arn:aws:serverlessrepo:us-east-1:072567720793:applications~ALB-Lambda-Target-HelloWorld)
```python
def lambda_handler(event, context):
	response = {
		"statusCode": 200,
		"statusDescription": "200 OK",
		"isBase64Encoded": False,
		"headers": {
			"Content-Type": "text/html; charset=utf-8"
	    }
	}

	response['body'] = """<html>
	<head>
		<title>Hello World!</title>
		<style>
			html, body {
			margin: 0; padding: 0;
			font-family: arial; font-weight: 700; font-size: 3em;
			text-align: center;
		}
		</style>
	</head>
	<body>
		<p>Hello World from Lambda</p>
	</body>
</html>"""
	return response
```

After the function is created, we need to create an internal ALB and route all traffic to the lambda. There are many [tutorials](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/lambda-functions.html) about this, but basically the steps are:

1. Create an internal ALB
2. On the navigation pane, under LOAD BALANCING, choose Target Groups.
3. Choose Create target group.
4. For Target group name, type a name for the target group.
5. For Target type, select Lambda function.
6. Register the Lambda function that is deployed earlier after you create the target group
7. Add a listener (or two) to the Load Balancer, forwarding all traffic to the target group

In the picture you can see a http listener and a https listener, using a certificate in ACM

![img2.png](images/img2.png)

As we said, it is not possible to call this function outside the AWS VPC
```shell
$ curl https://alb.jesus2.solo.io -v
*   Trying 10.0.1.186:443...
* connect to 10.0.1.186 port 443 failed: Operation timed out
*   Trying 10.0.2.117:443...
* connect to 10.0.2.117 port 443 failed: Operation timed out
*   Trying 10.0.3.223:443...
* After 74907ms connect time, move on!
* connect to 10.0.3.223 port 443 failed: Operation timed out
* Failed to connect to alb.jesus2.solo.io port 443 after 225096 ms: Couldn't connect to server
* Closing connection 0
curl: (28) Failed to connect to alb.jesus2.solo.io port 443 after 225096 ms: Couldn't connect to server
```

## 2 - Deploy Istio in an EKS cluster<a name="lab-2---deploy-istio-eks-"></a>