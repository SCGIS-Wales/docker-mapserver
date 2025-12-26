# Docker image for MapServer

The main Mapfile should be in `/etc/mapserver/`.

You can use the image as is and mount a volume in `/etc/mapserver/` to customize it.

Only tags for minor releases exist, not tag for bug fixes.

If the container is run as root, apache listens on port `80`. If it is run as
another user, it listens on port `8080`.

## Image contents and mounts

- MapServer 8.6.0 built with OGC API, WMS/WFS clients and servers, KML, SOS, XML Mapfile, PROJ, Cairo/SVG, and optional Oracle Spatial support (disabled unless the image is built with `WITH_ORACLE=ON`).
- Web servers: Apache httpd with `mod_fcgid` is the default; `lighttpd` and `spawn-fcgi` are also available via the `SERVER` environment variable. The entrypoint is `/usr/local/bin/start-server` and listens on port `8080` (or `80` when running as root).
- Configuration scaffolding: `/etc/mapserver.conf` ships as a template for MapServer config directives and environment variables; OGC API HTML templates live in `/usr/local/share/mapserver/ogcapi/templates/html-bootstrap/`.

### Mounting mapfiles, data, and config

- Mapfiles go under `/etc/mapserver`. You can mount a single file or a directory; the default `MS_MAP_PATTERN` only allows Mapfiles in that path.
  - Example (single Mapfile):
    ```bash
    docker run --rm \
      -p 8080:8080 \
      --volume /home/user/maps/project.map:/etc/mapserver/mapserver.map:ro \
      ssddgreg/mapserver
    ```
- Data directories must be mounted at the same container paths referenced in your Mapfiles. Keep mounts read-only when possible, e.g. `--volume /path/to/data:/data:ro` if your Mapfile points to `/data/...`.
  - Example (map + data):
    ```bash
    docker run --rm \
      -p 8080:8080 \
      --volume /home/user/maps/project.map:/etc/mapserver/mapserver.map:ro \
      --volume /home/user/data:/data:ro \  # Mapfile should reference /data/... inside the container
      ssddgreg/mapserver
    ```
- Optional config: mount a custom MapServer config to `/etc/mapserver.conf` (or point `MAPSERVER_CONFIG_FILE` to your path) when you need map aliases, env vars, or other global settings.

#### Using AWS S3 data (no mount)

- Prefer GDAL `/vsis3/` paths in your Mapfiles when source data is in AWS S3. Example `DATA "/vsis3/my-bucket/path/to/data.tif"` or `CONNECTION "/vsis3/my-bucket/path/to.gpkg"` for GDAL-backed layers.
- Provide AWS credentials via environment variables: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_SESSION_TOKEN` if using temporary creds. You can also set `AWS_REGION` or `AWS_PROFILE` as needed.
- MapServer passes GDAL config options set in the `MAP` section; use them if you need to override credential locations or signing behavior. No container volume mount is needed when using `/vsis3/`.

  - Example docker run (AWS S3–backed data, no data mount):
    ```bash
    docker run --rm \
      -p 8080:8080 \
      -e AWS_ACCESS_KEY_ID=AKIA... \
      -e AWS_SECRET_ACCESS_KEY=... \
      -e AWS_SESSION_TOKEN=... \  # if using temporary credentials
      -e AWS_REGION=eu-west-1 \
      --volume /home/user/maps/project.map:/etc/mapserver/mapserver.map:ro \
      ssddgreg/mapserver
    ```

  - Example Mapfile (raster from AWS S3):
    ```mapfile
    MAP
      NAME "s3-demo"
      WEB
        METADATA
          "ows_enable_request" "*"
        END
      END
      LAYER
        NAME "raster"
        TYPE RASTER
        DATA "/vsis3/my-bucket/path/to/data.tif"
      END
    END
    ```

  - Example Mapfile (vector from AWS S3 GeoPackage via OGR):
    ```mapfile
    MAP
      NAME "s3-vector-demo"
      WEB
        METADATA
          "ows_enable_request" "*"
        END
      END
      LAYER
        NAME "roads"
        TYPE LINE
        CONNECTIONTYPE OGR
        CONNECTION "/vsis3/my-bucket/path/to/data.gpkg"
        DATA "roads_layer_name"  # the layer name inside the GPKG
      END
    END
    ```

  - What is a GeoPackage (GPKG)?
    - An OGC standard based on SQLite: a single portable file that can store vector features, rasters, spatial indexes, and metadata. It’s widely supported (GDAL/MapServer/QGIS/etc.) and is ideal for packaging multi-layer datasets into one file for easy sharing and cloud storage (e.g., AWS S3).

  - Creating a GeoPackage for AWS S3 (if needed):
    - Create the GeoPackage locally with GDAL, then upload to AWS S3. Writing GPKG directly to `/vsis3/` is generally not recommended because GeoPackage (SQLite) needs random writes.
    - Create locally (example converts a Shapefile to GPKG):
      ```bash
      ogr2ogr -f GPKG /tmp/data.gpkg /path/to/source.shp
      ```
    - Upload to AWS S3 (requires AWS CLI configured):
      ```bash
      aws s3 cp /tmp/data.gpkg s3://my-bucket/path/to/data.gpkg
      ```
    - Discover layer names in the GeoPackage on AWS S3:
      ```bash
      ogrinfo /vsis3/my-bucket/path/to/data.gpkg -so -al
      ```
    - Use in Mapfile (set the `DATA` to the layer name reported by `ogrinfo`):
      ```mapfile
      LAYER
        NAME "roads"
        TYPE LINE
        CONNECTIONTYPE OGR
        CONNECTION "/vsis3/my-bucket/path/to/data.gpkg"
        DATA "roads_layer_name"
      END
      ```

  - IAM role-based credentials (no explicit keys):
    - When running on EC2/ECS/EKS with an attached role, GDAL's `/vsis3/` uses the standard AWS credential provider chain automatically. You typically do not need to pass `AWS_*` variables.
    - Supported sources include:
      - EC2 Instance Metadata Service v2 (IMDSv2).
      - ECS task role via `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI` or `AWS_CONTAINER_CREDENTIALS_FULL_URI`.
      - EKS IRSA: `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE` are injected by Kubernetes when the ServiceAccount is annotated for IAM roles.
    - Region selection: set `AWS_REGION` (or `AWS_DEFAULT_REGION`) if your bucket's region is not auto-detected.
    - Profiles/files (optional): `AWS_PROFILE`, `AWS_SHARED_CREDENTIALS_FILE`, and `AWS_CONFIG_FILE` are also honored by GDAL.

  - Minimum AWS IAM permissions (read-only):
    - If your data is in AWS S3 and your Mapfiles reference `/vsis3/my-bucket/...`, attach a least-privilege inline policy to the role used by the container.

    - Strict (no listing, only known object keys are read):
      ```json
      {
        "Version": "2012-10-17",
        "Statement": [
          {
            "Effect": "Allow",
            "Action": ["s3:GetObject"],
            "Resource": "arn:aws:s3:::my-bucket/path/to/*"
          }
        ]
      }
      ```

    - Read + list within a prefix (recommended if GDAL needs to enumerate files, e.g., tile indexes or wildcards):
      ```json
      {
        "Version": "2012-10-17",
        "Statement": [
          {
            "Effect": "Allow",
            "Action": ["s3:ListBucket"],
            "Resource": "arn:aws:s3:::my-bucket",
            "Condition": {
              "StringLike": { "s3:prefix": ["path/to/*"] }
            }
          },
          {
            "Effect": "Allow",
            "Action": ["s3:GetObject"],
            "Resource": "arn:aws:s3:::my-bucket/path/to/*"
          }
        ]
      }
      ```

    - Optional (SSE-KMS encrypted objects): add decrypt permissions for your KMS key in addition to the S3 actions above:
      ```json
      {
        "Version": "2012-10-17",
        "Statement": [
          {
            "Effect": "Allow",
            "Action": ["kms:Decrypt"],
            "Resource": "arn:aws:kms:REGION:ACCOUNT_ID:key/KEY_ID"
          }
        ]
      }
      ```

### Pre-run checklist

1. Prepare Mapfiles for MapServer 8.6.0 (set `ows_enable_request="*"`/`oga_enable_request="*"` and `oga_onlineresource` when using OGC API; update any migration-related syntax).
2. Ensure every path in the Mapfiles exists inside the container: mount Mapfiles to `/etc/mapserver` and mount datasets to matching container paths.
3. Decide the server mode: default `SERVER=apache`; use `SERVER=lighttpd` for a single-process setup or `SERVER=spawn-fcgi` when pairing with an external lighttpd/ingress.
4. Choose the port mapping: by default the container listens on `8080` (or `80` as root). Map to a host port with `-p <host>:8080` (or `-p <host>:80` when running as root).
5. Set optional env vars as needed: `MAPSERVER_BASE_PATH` for subpaths, `OGCAPI_HTML_TEMPLATE_DIRECTORY` if overriding templates, `MS_DEBUGLEVEL`/`MS_ERRORFILE` for logging, and the Apache/lighttpd tunables below for performance.

## Apache Tunings

You can use the following environment variables (when starting the container)
to tune it:

- `MS_DEBUGLEVEL`: The debug level `0`=off `5`=verbose
- `MS_ERRORFILE`: Location of the logging/debug output (defaults to `stderr`)
- `MAX_REQUESTS_PER_PROCESS`: To work around memory leaks (defaults to `1000`)
- `MIN_PROCESSES`: The minimum number of fcgi processes to keep (defaults to `1`)
- `MAX_PROCESSES`: The maximum number of fcgi processes to keep (defaults to `5`)
- `MAPSERVER_CATCH_SEGV`: Set to `1` to have the stacktraces in case of crash
- `MAPSERVER_BASE_PATH`: To setup which is the base path of mapserver (defaults to `/`)
- `BUSY_TIMEOUT`: The maximum time limit for request handling (defaults to `300`)
- `IDLE_TIMEOUT`: Application processes which have not handled a request for
  this period of time will be terminated (defaults to `300`)
- `IO_TIMEOUT`: The maximum period of time the module will wait while trying to
  read from or write to a FastCGI application (defaults to `40`)
- `APACHE_LIMIT_REQUEST_LINE`: The maximum size of the HTTP request line in
  bytes (defaults to `8190`)

## Lighttpd

You can also use lighttpd as the web server.

The main benefit of that is to have only one running process per container, that's useful especially on Kubernetes.

For that you need two containers: one for the MapServer and `spawn-fcgi`, and one for `lighttpd`.

The environment variable needed by mapserver should be on the `spawn-fcgi` container.

The MapServer logs will be available on the 'lighttpd' container.

Used environment variables:

- `LIGHTTPD_CONF`: The lighttpd configuration file (defaults to `/etc/lighttpd/lighttpd.conf`)
- `LIGHTTPD_PORT`: The port lighttpd will listen on (defaults to `8080`)
- `LIGHTTPD_FASTCGI_HOST`: The host of the FastCGI server (`spawn-fcgi`, defaults to `spawn-fcgi`)
- `LIGHTTPD_FASTCGI_PORT`: The port of the FastCGI server (`spawn-fcgi`, defaults to `3000`)
- `LIGHTTPD_FASTCGI_SOCKET`: The socket of the FastCGI server (defaults to `''`)
- `LIGHTTPD_ACCESSLOG_FORMAT`: The format of the access log (defaults to `"%h %V %u %t \"%r\" %>s %b"`)

## Running multiple Mapfiles

This section is for if you would like to use more than one Mapfile, or use a Mapfile
that isn't `/etc/mapserver/mapserver.map`.

In this example we have two Mapfiles we want to use that both reference data in
different directories. My Mapfiles are `wms.map` and `wfs.map` and are located
in `/mapfiles/` on the host, and the data for these Mapfiles is located in the
host in the directory `/mapdata/wms` and `/mapdata/wfs`.

```bash
docker run -d \
  --restart=unless-stopped \
  --volume=/mapfiles/wms.map/:/etc/mapserver/wms.map:ro \
  --volume=/mapfiles/wfs.map/:/etc/mapserver/wfs.map:ro \
  --volume=/mapdata/wms/:/mapdata/wms/:ro \
  --volume=/mapdata/wfs/:/mapdata/wfs/:ro \
  ssddgreg/mapserver
```

For accessing maps for the WFS service add `map=/etc/mapserver/wfs.map` to
your query string. Here is the URL for a `GetCapabilities` request:

`http://your.mapserver.host/?map=/etc/mapserver/wfs.map&service=WFS&request=GetCapabilities`

Similarly, for accessing maps for the WMS service add `map=/etc/mapserver/wms.map` to
your query string.

## OGC API

This image can be used to serve OGC API Features and OGC API Tiles.

Some details about the configuration.

You should define the `OGCAPI_HTML_TEMPLATE_DIRECTORY` generally to `/usr/local/share/mapserver/ogcapi/templates/html-bootstrap/`.

If you want to serve MapServer on a subpath, you can use the `MAPSERVER_BASE_PATH` with the subpath.

If you want to serve multiple MapFiles you should have a [Config file](https://mapserver.org/mapfile/config.html) with the `MAPS` directive.

In the Mapfile metadata you should have the following metadata (`MAP`.`WEB`.`METADATA`):

- `"ows_enable_request"` or `"oga_enable_request"` set to `"*"`.
- `"oga_onlineresource"` set to the URL (or path) of the OGC API.

The landing page is served by on `http://<host>:<port>/<base_path>/<map_name>/ogcapi`.

## Changelog

### 8.2

- The default url path is now at "/" (not "/mapserver" or others anymore).
- The default "APACHE_RUN_DIR" is now "/tmp/apache2" be sure this folder exists or/and can be created.

### 8.0

- `confd` and `entrypoints.d` are removed, you should replace it by a `volume_from` a configuration image
  or an init container.
- The `MS_MAPFILE` has no more default value, was `/etc/mapserver/mapserver.map`.

## Contributing

Install the pre-commit hooks:

```bash
pip3 install pre-commit
pre-commit install --allow-missing-config
```
