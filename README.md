# Paperless on Synology

As I went through some troubles to setup paperless on my Synology, I wanted to document it for others. I'm using a `DS218+` with DSM 6.2.3 and docker installed using the `Package Center`.

## Create required directories

Paperless requires three mount points to be configured:

* a `consume` directory from which paperless consumes the new document
* a `data` directory in which paperless creates its database
* a `media` directory in which paperless stores documents

You can use any directory you want but you need to correctly set the permissions. Configuring paperless with a user and group Id that are allowed to read/write these directories is required. 

For the `consume`, I have personally created a new share that my scanner can write to. 

### Getting the user and group Ids

You can find all userId and groupId in the `/etc/pwd` file of you NAS - I didn't find a better way to get them. You can proceed as follows:

1. enable the SSH server from the `Control Panel`. ![Enable the SSH server](img/ssh.png)
2. connect to you NAS using SSH
3. run the following command on your NAS: `more /etc/pwd`
4. for security reasons, it is recommended to disable the SSH server after you have retrieved the user and group Ids.

## Download the paperless image

Using the docker UI app, search for `paperless` in the `Registery` tab. Double click on image called `thepaperlessproject/paperless:latest` to download it.

![Downlaod the paperless image](img/getImage.png)

## Import the configuration

Using the Docker UI, in the `Container` tab, click on `setting > import` and import the two following configurations:

* [Consumer configuration](mypaperless-consumer.json) 
* [Web Server configuration](mypaperless-webserver.json) 

These configurations are the ones I used on my setup. You have to edit them to match your configuration before the import.

Change the language (English is by default the first one) and timezone:

```
{
         "key" : "PAPERLESS_OCR_LANGUAGES",
         "value" : "fra deu"
},
{
         "key" : "TZ",
         "value" : "Europe/Zurich"
},
```

Change the userId and groupId with the ones you have retrieved earlier:

```
{
    "key" : "USERMAP_UID",
    "value" : "1034"
},
{
    "key" : "USERMAP_GID",
    "value" : "100"
},
```

Update the directories bindings. You have to change every `host_volume_file` with the path matching your configuration, the other parameters should stay as they are.

```
"volume_bindings" : [
    {
        "host_volume_file" : "/scanner/paperless_consume",
        "mount_point" : "/consume",
        "type" : "rw"
    },
    {
        "host_volume_file" : "/docker/paperless/media",
        "mount_point" : "/usr/src/paperless/media",
        "type" : "rw"
    },
    {
        "host_volume_file" : "/docker/paperless/data",
        "mount_point" : "/usr/src/paperless/data",
        "type" : "rw"
    }
]
```

![Downlaod the paperless image](img/importConfig.png)

Once your container has been created, make sure the disk mapping has been correctly imported. I had some issues with the mapping import in the beginning: in my case, it was due to permission issues. If the import does not work out of the box, you can try to edit the container in the UI and re-create the mapping.

![Verify volume mapping](img/verifyVolumeMapping.png)

## Start the consumer

Using the Docker UI, start the `consumer` from the `Container` tab. To start the container, click on the I/O switch. Using the same UI, check the logs for no error. 

![Start](img/startContainer.png)

At the first start, a `db.sqlite3` should get created in the `data` directory and a `document` folder in the media directory.

## Start the webserver

Start the webserver container using the same Docker UI and check the logs for no error.

Once it's up and running, create the user you will use to access paperless:

* From the terminal tab, `create` a new terminal. 
* In the terminal, type `./manage.py createsuperuser` to create a new user.
* Provided the `username` and `password` you want to use to access paperless.
* Then browse to `http://<your-nas-ip>:8000`

![Create super user](img/createSuperUser.png)

## Enable SSL

If you want to encrypt the data between paperless and your browser, follow these steps. This is a "low security" configuration that I use on my home network. If you need something more serious please document yourself.

## Create the certificate

To create the certificate, I used the instructions from this page: https://code.luasoftware.com/tutorials/nginx/self-signed-ssl-for-nginx-and-chrome/

```
$ openssl genrsa -des3 -out certCA.key 2048
$ openssl req -x509 -new -nodes -key certCA.key -sha256 -days 2048 -out certCA.pem
$ openssl req -new -sha256 -nodes -out cert.csr -newkey rsa:2048 -keyout cert.key
```
For the last command, you should not set a `challenge password`.

create the configuration: cert.conf

```
$ nano cert.conf

authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = PUT_YOUR_HOSTNAME.PUT_YOUR_DOMAIN
DNS.2 = PUT_YOUR_HOSTNAME
```

For `[alt_names]`, I used this 

```
[alt_names]
DNS.1 = nas.local
DNS.2 = nas
```

with `nas` being a hostname known from my local DNS.

Now you can generate the certificate and rename the files to match paperless requirements
```
openssl x509 -req -in cert.csr -CA certCA.pem -CAkey certCA.key -CAcreateserial -out cert.crt -days 2048 -sha256 -extfile cert.conf

mv cert.crt ssl.cert
mv cert.key ssl.key
```

## Configure paperless to use SSL

Copy `ssl.cert` and `ssl.key` to the paperless `data` directory. To do that, one solution is to user Synology `File Station`: navigate to the correct location and use the `Upload data` functionality available on a right-click.

Stop the `mypaperless-webserver` container using the docker UI and edit its configuration. Change the `PAPERLESS_USE_SSL` environment variable to `true`. Then you can restart the container and point your browser to `https://<your-nas-ip>:8000`.

You will get a security warning about your certificate not being trusted. Most of the browser allows you to accept this security exception. I would however recommend not to do that and to follow the instructions of the next section.

## Trust Certificate

Accepting certificate security exception in your browser is a practice you should avoid. The best course of action is to tell your browser/OS to trust the certificate you have just created. Once you have done that, you won't get warned again regarding your certificate.

This configuration is specific to the OS and browser you use.

For Safari on macOS, you can simply import `ssh.cert` to macOS Keystore and mark it as trusted for SSL.

For Firefox on macOS, I had to import the in firefox preferences the `certCA.pem` and mark it as `This certificate can identify websites`.

Just google it ;-)

## Useful links

* doc: https://paperless.readthedocs.io/en/latest/setup.html
* synology: https://github.com/the-paperless-project/paperless/issues/385
* ssl: https://code.luasoftware.com/tutorials/nginx/self-signed-ssl-for-nginx-and-chrome/

