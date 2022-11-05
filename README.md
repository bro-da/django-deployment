# django-deployment
a simple step by step process for hosting django on any server with out hassle 

## introduction
I made this repo because some of my friends had difficulties deploying django web apps on aws because They had little to no experiance working on cli based system .Most get through (even me) by refering multiple pages and using various hacks but after at the end the deployment They get so overwhemed They forget the entire process so i am creating this repo so that those with difficulty can get the all the commands for deployment right here and as a source for further reference 

## steps

* create a server (ec2 instance works best and is pretty painless to create) 
    * make sure to open the http and https and ssh ports when creating the instance otherwise you have to go tinker on the security to get them to open
    * login to the ec2 instance through the .pem file you received during the instance creation process
* after login when we look to the prompt we could see we are logged in as ubuntu its always better to create a new user and unlock the ssh login through password     so we could login anywhere and login even if we lose our pem keys
* steps to create a user in ubuntu
```shell
sudo adduser user
```
