# CloudFormation Not Required
Collection of custom resources for things missing from CloudFormation. Leverages AWS built in AWS Lambda and [cfnresponse](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-lambda-function-code.html#cfn-lambda-function-code-cfnresponsemodule) to create custom types that can be used to bridge the gap.

[![forthebadge](https://forthebadge.com/images/badges/uses-badges.svg)](https://forthebadge.com)
[![forthebadge](https://forthebadge.com/images/badges/thats-how-they-get-you.svg)](https://forthebadge.com)
[![forthebadge](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://forthebadge.com)

# How to Use These
The custom resources should ideally be included as sub-stacks as that's the nicest way to pass parameters/re-use templates.

 1. Upload the snippets
 2. Use a [nested stack for each custom resource you desire](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-stack.html)
 3. Carry on

# API Gateway

## VPC Link Method Integration
As of April 2018, CloudFormation does not yet support [VPC Link integrations in API Gateway methods](https://docs.aws.amazon.com/apigateway/latest/developerguide/getting-started-with-private-integration.html). This is useful if you want to integrate with a private load balancer, or another resource in your VPC.

Example:
```
Resources:
  MyAlb:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Scheme: internal
      Type: network
      Subnets: [!Ref MyPrivateSubnetA, !Ref MyPrivateSubnetA]

  VpcLink:
    Type: AWS::ApiGateway::VpcLink
    Properties:
      Name: my-vpc-link
      Description: Link to private VPC
      TargetArns:
        - !Ref MyAlb

  MyApi:
    Type: AWS::ApiGateway::RestApi
    Properties:
      Name: My API

  ProxyResource:
    Type: AWS::ApiGateway::Resource
    Properties:
      RestApiId: !Ref MyApi
      ParentId: !GetAtt MyApi.RootResourceId
      PathPart: '{proxy+}'

  GetProxyRequest:
    Type: AWS::ApiGateway::Method
    Properties:
      AuthorizationType: NONE
      ApiKeyRequired: true
      HttpMethod: GET
      ResourceId: !Ref ProxyResource
      RestApiId: !Ref MyApi

  GetProxyRequestIntegration:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: http://s3.amazonaws.com/cloudformation-not-required/apigateway.vpc-link-method-integration.yaml
      Parameters:
        RestApiId: !Ref MyApi
        ResourceId: !Ref ProxyResource
        VpcConnectionId: !Ref VpcLink
        HttpMethod: 'GET'
        IntegrationUri: !Sub 'http://${MyAlb.DNSName}/{proxy}'
```

## Find Custom Domain Regional Domain Endpoint
As of April 2018, `!GetAtt MyApiGwDomain.DistributionDomainName` isn't yet supported for regional endpoints domains per https://forums.aws.amazon.com/thread.jspa?messageID=795128, nor does CloudFormation `AWS::ApiGateway::DomainName` currently return a [RegionalDomainName attribute](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-domainname.html#aws-resource-apigateway-domainname-returnvalues)

Example:
```
Resources:
  MyApiGwCustomDomain:
    Type: AWS::ApiGateway::DomainName
    Properties:
      DomainName: mygw.example.com
      RegionalCertificateArn: !Ref MyAcmCertificateArn
      EndpointConfiguration:
        Types:
          - REGIONAL

  MyRoute53Record:
    Type: AWS::Route53::RecordSetGroup
    Properties:
      HostedZoneName: 'example.com.'
      RecordSets:
        - Name: mygw.example.com
          Type: A
          AliasTarget:
            HostedZoneId: !GetAtt CustomRegionalDomain.RegionalHostedZoneId
            DNSName: !GetAtt CustomRegionalDomain.RegionalDomainName

  MyApi:
    Type: AWS::ApiGateway::RestApi
    Properties:
      Name: My API
      Description: My Great API
      EndpointConfiguration:
        Types: 
          - REGIONAL

  CustomRegionalDomain:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: http://s3.amazonaws.com/cloudformation-not-required/apigateway.regional-domain-endpoint.yaml
      Parameters:
        CustomApiGwDomainName: mygw.example.com
```
