---
path: '20200612'
date: 2020-06-12T16:35:22.049Z
title: AWS Lambdas and Layers in Python
description: >-
  The simplest possible way I know of to show how to do AWS Lambdas with layers
  in Python.
---
AWS Lambda Layers solve a few problems: 

1. **I can't see my lambda code on the AWS Lambda console** because there's too much code from all the packages copied into the image. This is a UX issue with the dev console so IMHO this should just be fixed by AWS. But hey…
2. Also, **my package needs access to native binaries** that do native things like say `ffmpeg` for sound stuff.

(What it DOES NOT solve is:

1. My build is greater than 250MB

…because the Lambda limit of 250MB includes all layers.)

Solution: **move all the library dependencies to a separate Lambda Laye**r, aka a plain zip file of stuff you want that's copied into your image, usually locally-installed packages and binaries.

Here are the bits you need for AWS Lambda functions with Layers. The good news is that there's no change from normal Lambda stuff, so it's a free recap if you're not up to speed with Lambdas.

1. A CloudFormation Stack created from a `template.yaml` file that's used for managing the lifespan of all the bits and pieces. 
2. S3 bucket for the code. One bucket, named of your choice, holds separate zip files for the main lambda function and others for each layer. You can upload them directly to Lambda directly, but they can't be > 50MB (LOL like when does that happen)
3. A special directory in your own project where all the dependencies are installed (via pip install etc). It can be any name at all. 
4. A version of your template.yaml file (by convention often "`out.yaml`" but can be anything) that simply changes any URIs (code, etc) to point to the deployed S3 versions rather than a local reference.

This Python example shows how to do it. This is substantially based on [this NodeJS tutorial](https://aws.amazon.com/blogs/compute/working-with-aws-lambda-and-lambda-layers-in-aws-sam/#:~:text=To%20support%20Lambda%20layers%2C%20SAM,or%20sam%20local%20start%2Dapi.). 

```
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: sam app
Globals:
   Function:
       Timeout: 3
       Runtime: python3.7
 
Resources:
   TempConversionFunction:
       Type: AWS::Serverless::Function
       Properties:
           CodeUri: hello_world/
           Handler: app.lambda_handler
           Layers:
             - !Ref TempConversionDepLayer
           Events:
               HelloWorld:
                   Type: Api
                   Properties:
                       Path: /{conversion}/{value}
                       Method: get
 
   TempConversionDepLayer:
       Type: AWS::Serverless::LayerVersion
       Properties:
           LayerName: temp-converter-dependencies
           Description: Dependencies for sam app [temp-units-conv]
           ContentUri: dependencies/
           CompatibleRuntimes:
             - python3.7
           LicenseInfo: 'MIT'
           RetentionPolicy: Retain
```

## Python steps to add layers to an existing lambda function

This assumes there's already an s3 bucket with the existing lambda code in it. If not, then you need to create one, using 

`aws s3api create-bucket --bucket temp-converter`

…(in my case).

The core lambda function root is usually just inside the app folder, e.g. sam-app/hello-world

1. `cd hello-world`

**Create a special place to keep a project-specific installation of all the python packages needed.** The name  doesn't matter. Some AWS docs call it "packages", others use "dependencies". What matters is that the layer points to it in the template.yml "ContentUri" property (see purple highlight)

2. `mkdir ../dependencies`

Now install the pip files locally. If you're using pipenv, then you have to make a mirror of the Pipfile in the requirements.txt or use another way of installing them. 

3. `pip install --target ../dependencies/python -r requirements.txt`

Start up local version of the lambda function working and test your functions locally in Docker. Note that this requires that you've installed Docker, in case that wasn't obvious. The command below will show you what the URL is. 

`cd ../` 

then

`sam build` or `sam build --use-container` if you have native dependencies (e.g. `ffmpeg`)

then: 

4. `sam local start-api`

You can then test your api at `127.0.0.1:3000` without needing to deploy. 

Now prepare for deployment. The bucket name must be globally-unique. Do this once. Once the bucket's created, you don't need to create another. 

5. `$ aws s3api create-bucket --bucket temp-converter`

Deployment: currently two steps: `sam package`, and `sam deploy`. Note `template.yaml` must be included even though [the docs (June 2020) say it's optional](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-package.html). Otherwise it includes the template from the build directory or something. 

6. `$ sam package --template-file template.yaml --s3-bucket temp-converter --output-template-file out.yaml`

`sam package` turns the relative URLs into public Uris pointing to s3:

```
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: sam app
Globals:
 Function:
   Timeout: 3
   Runtime: python3.7
Resources:
 TempConversionFunction:
   Type: AWS::Serverless::Function
   Properties:
     CodeUri: *s3://temp-converter/1a202a4b79ea459037b364b4c7d2826e*
     Handler: app.lambda_handler
     Layers:
     - Ref: TempConversionDepLayer
     Events:
       HelloWorld:
         Type: Api
         Properties:
           Path: /{conversion}/{value}
           Method: get
 TempConversionDepLayer:
   Type: AWS::Serverless::LayerVersion
   Properties:
     LayerName: temp-converter-dependencies
     Description: Dependencies for sam app \[temp-units-conv]
     ContentUri: *s3://temp-converter/1ec4cbd8c61a6ff58ed03c8c26968ab5*
     CompatibleRuntimes:
     - python3.7
     LicenseInfo: MIT
     RetentionPolicy: Retain
```

Now deploy using this new output file. Stacks are a pile of AWS resources that it tracks the lifespan of. stack-name is whatever you want to call the resulting stack of this deployment. 

7. `$ sam deploy --template-file ./out.yaml --stack-name temp-converter-stack --capabilities CAPABILITY_IAM`
