# Laravel - Scalable apps in AWS

For laravel 5.3+.

This can be used with Elastic Beanstalk Multi Docker Container.

## By using this setup to deploy your apps your app will:

- Have one or more app instances depending on the load of your app
- Have a separate cache cluster server with one or multiple cache nodes (memcached).
- Have a separate database server  (default is mysql)
- CDN server configured and ready to use for your `app.css` and `app.js`.


## What is special about this setup?

- Database is automatically configured. No configuration needed what so ever.
- Cache is setup on a shared memcache (elasticache) server - AUTOMATICALLY! The server is even created automatically - see `.ebextensions/elasticache.config`
- S3 bucket is automatically created for your app, so you can store uploaded files! This means - scaling is ready out of the BOX!
- Cron that runs the scheduled laravel tasks.
- Migrations are run on each deploy.


## AWS Resources in use

- AWS Elastic Beanstalk will handle deployments.
- Load balancer with Auto scaling . Meaning you can have multiple EC2 instances running your app.
- ElastiCache configured as the cache (memcached) for laravel
- RDS for the database (mysql is default)
- S3 for storing uploaded assets (e.g. images).
- Cloudfront created for you to use. Env variable `CDN` refers to the CDN address, just use it right away for your assets.




## There is some setup you need to do

A fair warning. This is not a one-click install, you will need to know a bit about AWS services to get this working.
A fair amount of configuration is needed, but I have tried to explain the steps below.



# Getting Started



## Create your shiny laravel app in web folder.

```
git clone https://github.com/peec/laravel-aws ~/my-deploy-root
cd ~/my-deploy-root
laravel new myapp

# Note: change the app folder to "web".
mv myapp web
```



Directory structure should now be:

- .ebextensions/
- web/  `This is your app location`
- .gitignore
- .ebignore
- Dockerrun.aws.json
- README.md


When working with your laravel app, you should be inside the web folder.


### How to work with laravel and version control

You will work on your laravel app like you normally do inside the `web` folder. The web folder can be versioncontrolled in git like normal laravel projects is.

When you push changes with `eb deploy`, elastic beanstalk will push up the web folder as-is, so don't forget to push nessecary changes and branch correctly inside the web folder before pushing `eb deploy`.


## Configure .ebextensions/options.config.dist

```
cp  .ebextensions/options.config.dist .ebextensions/options.config
```


Configure these:

- AWS_ACCESS_KEY_ID <-- Get this via AWS IAM
- AWS_SECRET_ACCESS_KEY <-- Get this via AWS IAM
- GITHUB_OAUTH_TOKEN  <-- Token to get composer deps. You can get this in the github profile settings.
- APP_KEY <-- Set it to the laravel app key found in web/.env dir. (laravel dev copy).

#### Custom policies

**A note on the AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY .**

These are needed to fetch the associated resources information (elasticache). And should have access to:

```
"ec2:Describe*",
"cloudformation:Describe*",
"elasticbeanstalk:Describe*"
```



For this to work you need to modify the beanstalk role in AWS IAM

Add this ( IAM -> Roles -> aws-elasticbeanstalk-ec2-role -> Inline Policies -> Create Role Policy -> Custom Policy ) and here is what you add:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "ec2:Describe*",
        "cloudformation:Describe*",
        "elasticbeanstalk:Describe*"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```



## Configure Laravel


### Store uploaded files using s3

Lets first allow s3 uploads. This is also explained in the laravel docs.

```
cd web
composer require league/flysystem-aws-s3-v3 ~1.0
```

And also we want some new env variables.


`app/config/filesystems.php`

```
 ....
    
        'default' => env('IS_ON_AWS', false) ? 's3' : 'local',
 ....

        's3' => [
            'driver' => 's3',
            'key' => env('MEDIA_S3_ACCESS_KEY'),
            'secret' => env('MEDIA_S3_SECRET_KEY'),
            'region' => env('MEDIA_S3_REGION'),
            'bucket' => env('MEDIA_S3_BUCKET'),
        ],
 ....
```


## Configure AWS


### Beanstalk Multi Container Docker policy

The Multi-container Docker platform requires additional ECS permissions.

For more information see: https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/create_deploy_docker_ecstutorial.html#create_deploy_docker_ecstutorial_role



## Install elastic beanstalk tools on your local machine

http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/eb-cli3-install.html


## Create a beanstalk app

Lets run `eb init` to start the new beanstalk app. Follow the guide below.

```

cd ~/my-deploy-root
eb init

Select a default region
.....
4) eu-west-1 : EU (Ireland)
....
(default is 3): 4

Select an application to use
1) xxxx
2) yyyy
3) [ Create new Application ]
(default is 3): 3

Enter Application Name
(default is "myapp"): 
Application myapp has been created.

It appears you are using Multi-container Docker. Is this correct?
(y/n): y

Select a platform version.
1) Multi-container Docker 1.11.2 (Generic)
2) Multi-container Docker 1.11.1 (Generic)
3) Multi-container Docker 1.9.1 (Generic)
4) Multi-container Docker 1.6.2 (Generic)
(default is 1): 1
Do you want to set up SSH for your instances?
(y/n): y

Select a keypair.
1) xxxx
2) yyyy
3) zzzz
4) [ Create new KeyPair ]
(default is 4): 1

```





And to upload files, use laravel's "cloud" disk.


## Create a new enviroment

Lets create a new environment for the production env.

Adjust things according to needs.

```
cd ~/my-deploy-root
eb create --database --database.engine mysql --database.instance db.t2.micro --database.size 10 -i t2.micro --database.username ebroot --database.password mydbpassword321 myapp-prod
```

**CHANGES**

- Change database.password to your needs
- Change myapp-prod to e.g. yourapp-prod or e.g. yourapp-staging, a naming convention is great here.


Now wait for a while (30 minutes or so), all resources like ElastiCache, RDS, EC2 etc must be created. This can take some minutes.


Open up the EB console to check status.


```
eb console
```



## Test your app.

Lets open the app in the browser.

```
eb open
```


You should see the laravel welcome page.


## Check (or manage) your laravel app via SSH in the EC2

```
# log into the EC2
eb ssh

# List all the docker containers.
sudo docker ps 


# Get the "CONTAINER ID" of the "peec/laravel-docker-aws" and run this (replace XXXXXXXXX with the CONTAINER ID ):

# Lets run bash for one of the containers.....
sudo docker exec -i -t XXXXXXXXX /bin/bash


# List the laravel files
ls -al 


# See the automatically generated environment variables:
cat .env


# all is fine, go out of the container

exit

# And go out of the EC2

exit


```






# Q / A


### I don't want to pay for Cloudfront, how do i rem ove it ?

Just remove the file `.ebextensions/cdn.config`. 

```
git add --all
git commit -m "remove cdn"
eb deploy
```


### I don't want to pay for elasticache,  how do I remove it?

Just remove the file `.ebextensions/elasticache.config`. Beanstalk will skip creating the elasticache resource. Standard cache will be used instead.

```
git add --all
git commit -m "remove elasticache"
eb deploy
```

### How do i add environment variables to the laravel app?

- Go to the AWS web console
- Find the Elastic Beanstalk service and click into your environment.
- Add laravel specific environment variables under "Configuration -> Software Configuration".


### How do I configure laravel to send e-mails via AWS SES.

Go to the AWS web console and add these environment variables under "Configuration -> Software Configuration":

- MAIL_DRIVER: `ses`
- SES_KEY: `YOUR_AWS_ACCESS_KEY`
- SES_SECRET: `YOUR_AWS_SECRET_KEY`

By default the region is `region` inside the laravel/config/services.php ("ses" section) so you need to edit this in your app to your needs and redeploy.

- RDS


### What resources do i pay for when using this?

- Load balancer *required*
- EC2 *required*
- Elasticache
- Cloudfront
- S3

As you can see, the monthly costs can be 100$ easily. 

The pricing depends on how you configure .ebextensions folder.


### How do i upload files to s3?


If you have followed the configuration steps explained, it's just as you would do normally:

```
Route::post('/avatars', function (){
    request()->file('avatar')->store('avatars');
    return back();
});
```



# Troubleshooting 


In the aws web console (beanstalk environment) you can download log files to troubleshoot (via Logs -> Request logs -> Last 100 lines)

#### Error: You have not correctly added AWS ACCSESS / SECRET keys to Dockerrun.aws.json

**The Error:**

```
An error occurred (AuthFailure) when calling the DescribeInstances operation: Authorization header or parameters are not formatted correctly.
configure-instance-resources:ERROR: Could not get json from aws ec2 describe-instances 
```

**Solution:**

Go to Dockerrun.aws.json and configure AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY . Also the keys associated to a specific key set must have access to describe resources.. See `Custom policies` above.
Redeploy `eb deploy`.

