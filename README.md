# Welcome to Zappa / DJango!

Thanks to:

 - https://www.agiliq.com/blog/2019/01/complete-serverless-django/
 - https://romandc.com/zappa-django-guide/

## Zappa/DJango AWS deployment
Deploying DJango project on AWS serverless architecture. 
## Step 1.
 - [ ] Configure AWS credential
> ~/.aws/credentials

    [default]
    aws_access_key_id = "your aws_iam_access_key"
    aws_secret_access_key = "your aws_iam_user_secret_key" 
       
## Step 2.
- [ ] Create VIRTUAL ENV
### 1. Create Virtual Environment and Activate

    $ virtualenv -p python3 env

If **windows** run ...

    $ env\Scripts\activate

If **ubuntu** or **mac** run ...

    $ source env/bin/activate

### 2. Install requirements by running the following command.

    $ sudo pip3 install -r requirements.txt

> Make sure following libraries are included in requirements.txt
> - Django==2.0.0
> - Pillow
> - mysqlclient
> - boto3
> - boto
> - django-storages==1.6.6
> - zappa

We will be using DJango v2.0.0 because of the compatibility of MySQL.

## Step 3.
 - [ ]  ZAPPA 

 ### 1. We will first initiate Zappa

    $ cd mysite/
    $ zappa init

Follow on screen instructions.
Once done, open `zappa_settings.json` and make sure following format is followed.

    {
	    "dev": {
	        "django_settings": "mysite.settings",
	        "profile_name": "default",
	        "project_name": "mysite",
	        "runtime": "python3.7",
	        "s3_bucket": "mysite-django",
	        "aws_region": "us-east-1"
	    }
    }

### 2. We deploy Zappa

    $ zappa deploy dev

Once done a link will be returned which we will add into `mysite/settings.py`.

    ALLOWED_HOSTS = ['url_returned']

We push updates to AWS.

    $ zappa update dev

## Step 4.
 - [ ] S3 Bucket
### 1. Configure Bucket Policy

  Add following lines into your S3 bucket CORS policy. 

    <?xml version="1.0" encoding="UTF-8"?>
    <CORSConfiguration xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
    <CORSRule>
        <AllowedOrigin>*</AllowedOrigin>
        <AllowedMethod>HEAD</AllowedMethod>
        <AllowedMethod>GET</AllowedMethod>
        <AllowedMethod>PUT</AllowedMethod>
        <AllowedMethod>POST</AllowedMethod>
        <AllowedMethod>DELETE</AllowedMethod>
        <AllowedHeader>*</AllowedHeader>
    </CORSRule>
    </CORSConfiguration>

### 2. Configure Static & Media Files

Add following lines into `mysite/settings.py`.
Make sure to change 

    INSTALLED_APPS = [ 
	    ...,
	    'storages',
	    ....
	]
	
	AWS_STORAGE_BUCKET_NAME = "mysite-django"
    AWS_S3_CUSTOM_DOMAIN = '%s.s3.amazonaws.com' % AWS_STORAGE_BUCKET_NAME
    AWS_ACCESS_KEY_ID = "iam_user_access_key"
    AWS_SECRET_ACCESS_KEY = "iam_user_secret_key"
    AWS_S3_FILE_OVERWRITE = False
    AWS_S3_OBJECT_PARAMETERS = {
        'CacheControl': 'max-age=86400',
    }
    AWS_LOCATION = 'static'
    MEDIAFILES_LOCATION = 'media'
    STATICFILES_DIRS = [
        os.path.join(BASE_DIR, 'mysite/static'),
    ]
    STATIC_URL = 'https://%s/%s/' % (AWS_S3_CUSTOM_DOMAIN, AWS_LOCATION)
    STATICFILES_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
    DEFAULT_FILE_STORAGE = 'mysite.storage_backends.MediaStorage'
    AWS_SECURITY_TOKEN_IGNORE_ENVIRONMENT = True
    
    STATIC_URL = '/static/'
    MEDIA_URL = '/media/'

Create a new file `mysite/storage_backends.py`.

    from storages.backends.s3boto3 import S3Boto3Storage

    class MediaStorage(S3Boto3Storage):
        location = 'media'
        file_overwrite = False

Upload static files to S3 Bucket

    $ python3 manage.py collectstatic

> Make sure static folder exists in mysite/ folder.

Once done update, AWS.

    $ zappa update dev

### 3. Give S3 Read and Write Permissions to Lamda 

> **NOTE:**
> Setting Up Routing to allow file upload on AWS
> - Go to VPS service and create endpoint.
> - Make sure to link with existing subnets.

.

## Step 5.

 - [ ] Database MySQL.

### 1. Setup Database
We will setup RDS, Aurora MySQL database.
Go to RDS service and create Database.

 1. Standard Create
 2. Amazon Aurora
 3. Amazon Aurora with MySQL compatibility
 4. Aurora (MySQL)-5.6.10a
 5. Regional
 6. Serverless
 7. Database name and username must be **lowercase**
 8. Create Password
 9. Select Capacity

Click on Create.

### 2. Give database access to Lamda

 1. Go to Lamda Service
 2. Select Lamda function
 3. Scroll Down to network and add subnets and security group
 4. Click save

### 3. Allow port through firewall

 1. Go to security group located on the bottom right side of the database page you just created.
 2. At the bottom add inboud rule with Aurora/MYSQL, TCP, port 3306, and CUSTOM Security Group.

### Connecting with DJango

Add following lines of code in `settings.py`.

    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.mysql',
            'NAME': 'mysitedb', # dbname
            'USER': 'mysiteadmin', # master username
            'PASSWORD': 'password', # master password
            'PORT': '3306',
            'HOST': 'mysitedb.cluster-cmrgi4dsutgn.us-east-1.rds.amazonaws.com', # Endpoint
        }
    }

 Create a folder in mysite + all other apps with name `management`.
 Filestructure should be as follows.
 
 ```bash
.
├── commands
│   ├── create_db.py
│   └── __init__.py
└── __init__.py

```

Add following code in `create_db.py`.

	# mysite/management/commands/create_db.py
    import sys
    import logging
    import MySQLdb
    
    from django.core.management.base import BaseCommand, CommandError
    from django.conf import settings
    
    rds_host = 'mysitedb.cluster-cmrgi4dsutgn.us-east-1.rds.amazonaws.com'
    db_name = 'mysitedb'
    user_name = 'mysiteadmin'
    password = 'password'
    port = 3306
    
    logger = logging.getLogger()
    logger.setLevel(logging.INFO)
    
    
    class Command(BaseCommand):
        help = 'Creates the initial database'
    
        def handle(self, *args, **options):
            print('Starting db creation')
            try:
                db = MySQLdb.connect(host=rds_host, user=user_name,
                                     password=password, db="mysql", connect_timeout=5)
                c = db.cursor()
                print("connected to db server")
                c.execute("""CREATE DATABASE mysitedb;""")
                c.execute(
                    """GRANT ALL PRIVILEGES ON db_name.* TO 'mysite_admin' IDENTIFIED BY 'mysiteadmin';""")
                c.close()
                print("closed db connection")
            except:
                logger.error(
                    "ERROR: Unexpected error: Could not connect to MySql instance.")
                sys.exit()

> Make sure to change according to your requirement.

Once done update code on AWS.

    $ zappa update dev

We create tables.

    $ zappa manage dev create_db

If you face any mysql database problems add following lines into `mysite/__init__.py`.

    import pymysql
    
    pymysql.install_as_MySQLdb()

And install `pymysql` library.

    $ pip3 install pymysql

Run migrations.

    $ zappa manage dev migrate

Create Super User
    
    $ zappa invoke --raw dev "from django.contrib.auth.models import User; User.objects.create_superuser('admin', 'admin@admin.com', 'admin')"
