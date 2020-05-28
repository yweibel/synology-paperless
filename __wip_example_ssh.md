## Javier's

docker-compose.env

```
# Environment variables to set for Paperless
# Commented out variables will be replaced with a default within Paperless.
#
# In addition to what you see here, you can also define any values you find in
# paperless.conf.example here.  Values like:
#
# * PAPERLESS_PASSPHRASE
# * PAPERLESS_CONSUMPTION_DIR
# * PAPERLESS_CONSUME_MAIL_HOST
#
# ...are all explained in that file but can be defined here, since the Docker
# installation doesn't make use of paperless.conf.
PAPERLESS_OCR_THREADS=8

# Use this variable to set a timezone for the Paperless Docker containers. If not specified, defaults to UTC.
# TZ=America/Los_Angeles

# Additional languages to install for text recognition.  Note that this is
# different from PAPERLESS_OCR_LANGUAGE (default=eng), which defines the
# default language used when guessing the language from the OCR output.
PAPERLESS_OCR_LANGUAGES=fra spa cat deu ita 

# Set Paperless to use SSL for the web interface.
# Enabling this will require ssl.key and ssl.cert files in paperless' data directory.
PAPERLESS_USE_SSL=true

# You can change the default user and group id to a custom one
USERMAP_UID=1003
USERMAP_GID=1003
```

docker-compose.yml

```
version: '2.1'

services:
    webserver:
        build: ./
        # uncomment the following line to start automatically on system boot
        restart: always
        ports:
            # You can adapt the port you want Paperless to listen on by
            # modifying the part before the `:`.
            - "192.168.30.170:8000:8000"
        healthcheck:
            test: ["CMD", "curl" , "-kf", "https://localhost:8000"]
            interval: 30s
            timeout: 10s
            retries: 5
        volumes:
            - /mnt/local-hdd/paperless/data:/usr/src/paperless/data
            - /mnt/local-hdd/paperless/media:/usr/src/paperless/media
            # You have to adapt the local path you want the consumption
            # directory to mount to by modifying the part before the ':'.
            - /mnt/local-hdd/paperless/consume:/consume
        env_file: docker-compose.env
        # The reason the line is here is so that the webserver that doesn't do
        # any text recognition and doesn't have to install unnecessary
        # languages the user might have set in the env-file by overwriting the
        # value with nothing.
        environment:
            - PAPERLESS_OCR_LANGUAGES=
        command: ["gunicorn", "-b", "0.0.0.0:8000"]

    consumer:
        build: ./
        # uncomment the following line to start automatically on system boot
        restart: always
        depends_on:
            webserver:
                condition: service_healthy
        volumes:
            - /mnt/local-hdd/paperless/data:/usr/src/paperless/data
            - /mnt/local-hdd/paperless/media:/usr/src/paperless/media
            # This should be set to the same value as the consume directory
            # in the webserver service above.
            - /mnt/local-hdd/paperless/consume:/consume
            # Likewise, you can add a local path to mount a directory for
            # exporting. This is not strictly needed for paperless to
            # function, only if you're exporting your files: uncomment
            # it and fill in a local path if you know you're going to
            # want to export your documents.
            # - /path/to/another/arbitrary/place:/export
        env_file: docker-compose.env
        command: ["document_consumer"]
```