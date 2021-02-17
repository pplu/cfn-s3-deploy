# Deploying web applications to S3 via CloudFormation

Web applications these days tend to have Frontend components that are prepared to live in a different place (URL) than 
the APIs they get their data from. This architecture leads us to being able to host frontend and backend in different 
servers / services, depending on their needs.

One option is to host your application frontend on S3: I love this option, because with a little setup effort, we can 
forget about some operational tasks:

 - Storage: I know your frontend will not be too big. Hosting your frontend on an instance means paying for storage that 
   you don't use (since the minimum storage for an instance is around 8GB). On S3 you will only pay for what you will pay
   for the few MBs you use.
 - Scale: Hosting static files (HTML, JS, CSS, etc) on an instance with a web server is not that difficult, and with a
   very small instance you can get away with some interesting load (no need to scale). But when you have to scale or maintain
   that instance (make it bigger, restart, etc): you are going to have to take time to do it without affecting your service.
   This is completely avoidable.
 - Security: If you configure S3 correctly: no patching duties.
 - Performance: You can get excellent performance by fronting your bucket with a CloudFront distribution, which serves 
   the contents of the bucket from closer locations to whoever is visiting it.

There are lots of tutorials and articles on how to host a static website on S3. Most of them take you manually through the steps
to do so. I create and maintain all production infrastructures via CloudFormation. Creating a stack that deploys an S3 Bucket, a 
CloudFront Distribution, a CloudFront AOI, a Bucket policy that only serves content to CloudFront and a DNS pointing to the 
CloudFront Distribution is fairly standard stuff. CloudFormation supports it without problem.

This brings you to a stage where you have the infrastructure, but there is nothing in the Bucket to serve! The tricky stuff starts 
when you want your primitives to be `deploy version 3 of the frontend`, `update to version 4 of the frontend`. CloudFormation 
doesn't have native primitives resources for creating objects on S3! But thanks to CloudFormation custom resources we can tailor
something to our needs.

## UnzipToS3 - Deploying the frontend

The UnzipToS3 Custom Resource is used to take a ZIP from one bucket (with the contents of your fronted) and deploys it to the `DestinationBucket`.
It also calls CloudFront to invalidate the cache, and thus serve the new contents.

Normally in the `OriginBucket` you will have various versions of your app, which UnzipToS3 can deploy

```
  DeployWebapp:
    Type: Custom::DeployWebApp
    Version: '1.0'
    Properties:
      ServiceToken: !GetAtt UnzipToS3.Arn
      DistributionId: !Ref CDN
      DestinationBucket: !Ref Bucket
      OriginBucket: 'bucket-name-with-frontend-artifacts'
      OriginKey: !Join [ '', [ 'frontend/frontend_', { Ref: Version }, '.zip' ] ]
```

The UnzipToS3 Custom Resource does the following:

 - Unzips **OriginBucket/OriginKey** to the root of **DestinationBucket**
 - Assigns Content-Type attributes so the unzipped files are served correctly
 - Sends an invalidation to CloudFront **DistributionId**

The Resource is conscious of CloudFormation updates, so on an update, it will:

 - Deploy the new zip
 - Delete all the files from the old zip (note that common files will be left as the ones on the new zip)

On deletion it will delete only the files that were deployed by the zip file (other files are left intact).

Caveat: which file belongs to which zip is implemented by marking each objects origin in S3 metadata. Uploading
untracked files, or reuploading files without the appropiate metadata will make the Custom Resource ignore the
file on updates and deletions.

The code for UnzipToS3 can be found here: [UnzipToS3.yaml](UnzipToS3.yaml)

## UploadToS3 - Configuring the frontend

Your frontend artifact should be shipped unconfigured (so the same artifact can be used in multiple environments,
like, development, preproduction and production). If this is so, you may probably need to generate a configuration
file.

```
  ConfigureWebapp:
    Type: Custom::UploadToS3
    Version: '1.0'
    Properties:
      ServiceToken: !GetAtt UploadToS3.Arn
      DestinationBucket: !Ref Bucket
      DestinationKey: 'config.xml'
      Content: 
        Fn::Sub:
        - |
          <?xml version="1.0" encoding="UTF-8"?>
          <version>
            <uri>${API_ENDPOINT}</uri>
            <client-id>${CLIENT_ID}</client-id>
            <user-pool-id>${USER_POOL}</user-pool-id>
          </version>
        - API_ENDPOINT: 'https://myapi.example.com'
          CLIENT_ID: !Ref UserPoolClient
          USER_POOL: !Ref UserPool
```

The UploadToS3 Custom Resource generates a file named **DestinationBucket/DestinationKey** with the contents in
the **Content** property.

This plants the information that the application needs for working correctly. In this example, it configures an API endpoint, 
as well as information for authenticating the user against a Cognito User Pool.

This Custom Resource also updates the file on S3 when the content changes, and deletes the file when the resource is deleted.

The code to UploadToS3 can be found here: [UploadToS3.yaml](UploadToS3.yaml)

## Bringing it together

In [Frontend.yaml](Frontend.yaml) there is a full example of a complete frontend stack that uses the two custom resources in
combination to deploy an application. Please note that this is an example intended for you to see how everything fits together.
If you pretend to use it, I recommend you adapt it to your needs.

# Author, Copyright and License

This article was authored by Jose Luis Martinez Torres.

This article is (c) 2021 Jose Luis Martinez Torres, Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).

The canonical, up-to-date source is [GitHub](https://github.com/pplu/cfn-s3-deploy). Feel free to contribute back.

