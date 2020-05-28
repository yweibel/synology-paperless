
```
docker-compose up -d
docker-compose run --rm webserver createsuperuser
```


```
> id
uid=501(yweibel) gid=20(staff) groups=20(staff) ...
```

-> in docker-cimpose.env
```
USERMAP_UID=501
USERMAP_GID=20
```