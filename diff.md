```
Stack Main
IAM Statement Changes
┌───┬───────────────────────────────┬────────┬───────────────────────┬──────────────────────────────────┬──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│   │ Resource                      │ Effect │ Action                │ Principal                        │ Condition                                                                                                                                    │
├───┼───────────────────────────────┼────────┼───────────────────────┼──────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ + │ ${TestLambda.Arn}             │ Allow  │ lambda:InvokeFunction │ Service:apigateway.amazonaws.com │ "ArnLike": {                                                                                                                                 │
│   │                               │        │                       │                                  │   "AWS:SourceArn": "arn:aws:execute-api:ap-northeast-1:242201303782:${TestPrivateApi}/${TestPrivateApiDeploymentStageprod43B4EC9B}/GET/test" │
│   │                               │        │                       │                                  │ }                                                                                                                                            │
│ + │ ${TestLambda.Arn}             │ Allow  │ lambda:InvokeFunction │ Service:apigateway.amazonaws.com │ "ArnLike": {                                                                                                                                 │
│   │                               │        │                       │                                  │   "AWS:SourceArn": "arn:aws:execute-api:ap-northeast-1:242201303782:${TestPrivateApi}/test-invoke-stage/GET/test"                            │
│   │                               │        │                       │                                  │ }                                                                                                                                            │
├───┼───────────────────────────────┼────────┼───────────────────────┼──────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ + │ ${TestLambda/ServiceRole.Arn} │ Allow  │ sts:AssumeRole        │ Service:lambda.amazonaws.com     │                                                                                                                                              │
├───┼───────────────────────────────┼────────┼───────────────────────┼──────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ + │ execute-api:/*                │ Allow  │ execute-api:Invoke    │ AWS:*                            │                                                                                                                                              │
└───┴───────────────────────────────┴────────┴───────────────────────┴──────────────────────────────────┴──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
IAM Policy Changes
┌───┬───────────────────────────┬────────────────────────────────────────────────────────────────────────────────────┐
│   │ Resource                  │ Managed Policy ARN                                                                 │
├───┼───────────────────────────┼────────────────────────────────────────────────────────────────────────────────────┤
│ + │ ${TestLambda/ServiceRole} │ arn:${AWS::Partition}:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole     │
│ + │ ${TestLambda/ServiceRole} │ arn:${AWS::Partition}:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole │
└───┴───────────────────────────┴────────────────────────────────────────────────────────────────────────────────────┘
Security Group Changes
┌───┬────────────────────────────────────┬─────┬────────────┬──────────────────────────────────────┐
│   │ Group                              │ Dir │ Protocol   │ Peer                                 │
├───┼────────────────────────────────────┼─────┼────────────┼──────────────────────────────────────┤
│ + │ ${TestLambdaSecurityGroup.GroupId} │ In  │ TCP 443    │ ${VpcEndpoint/VpcEndpointSG.GroupId} │
│ + │ ${TestLambdaSecurityGroup.GroupId} │ Out │ Everything │ Everyone (IPv4)                      │
└───┴────────────────────────────────────┴─────┴────────────┴──────────────────────────────────────┘
(NOTE: There may be security-related changes not in this list. See https://github.com/aws/aws-cdk/issues/1299)

Resources
[+] AWS::EC2::VPCEndpoint VPC/DefaultVpc/ApiGatewayEndpoint VPCDefaultVpcApiGatewayEndpoint0B6C22E8
[+] AWS::EC2::SecurityGroup TestLambdaSecurityGroup TestLambdaSecurityGroupB2333BAC
[+] AWS::EC2::SecurityGroupIngress TestLambdaSecurityGroup/from MainVpcEndpointVpcEndpointSGD8B01787:443 TestLambdaSecurityGroupfromMainVpcEndpointVpcEndpointSGD8B017874431BC68A44
[+] AWS::IAM::Role TestLambda/ServiceRole TestLambdaServiceRoleC28C2D9C
[+] AWS::Lambda::Function TestLambda TestLambda2F70C45E
[+] AWS::Logs::LogGroup TestLambda/LogGroup TestLambdaLogGroupB66D2B71
[+] AWS::ApiGateway::RestApi TestPrivateApi TestPrivateApi2C4B5C27
[+] AWS::ApiGateway::Deployment TestPrivateApi/Deployment TestPrivateApiDeployment1CAD6163bafcb8d19d90f000ebde4cdfa85b7a83
[+] AWS::ApiGateway::Stage TestPrivateApi/DeploymentStage.prod TestPrivateApiDeploymentStageprod43B4EC9B
[+] AWS::ApiGateway::Resource TestPrivateApi/Default/test TestPrivateApitestA4669767
[+] AWS::Lambda::Permission TestPrivateApi/Default/test/GET/ApiPermission.MainTestPrivateApiF7422075.GET..test TestPrivateApitestGETApiPermissionMainTestPrivateApiF7422075GETtestAEF32D93
[+] AWS::Lambda::Permission TestPrivateApi/Default/test/GET/ApiPermission.Test.MainTestPrivateApiF7422075.GET..test TestPrivateApitestGETApiPermissionTestMainTestPrivateApiF7422075GETtest66E17D85
[+] AWS::ApiGateway::Method TestPrivateApi/Default/test/GET TestPrivateApitestGET43D641DE
[~] AWS::Lambda::Function ShortMemoryScan/ShortMemoryScanFunction ShortMemoryScanShortMemoryScanFunction29B169E6
 └─ [~] Code
     └─ [~] .S3Key:
         ├─ [-] 152c69a0aa964f23a6995372161a97edf702d4ec5f8c1275c252d80867b6d20d.zip
         └─ [+] c8295fff2ee483856c918c05c878ab7162ccc058eb170dd97418e75c000a4ef0.zip

Outputs
[+] Output TestPrivateApi/Endpoint TestPrivateApiEndpointE0E03A97: {"Value":{"Fn::Join":["",["https://",{"Ref":"TestPrivateApi2C4B5C27"},".execute-api.ap-northeast-1.",{"Ref":"AWS::URLSuffix"},"/",{"Ref":"TestPrivateApiDeploymentStageprod43B4EC9B"},"/"]]}}
```