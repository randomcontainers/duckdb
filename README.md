# duckdb

Container images with [DuckDB](https://duckdb.org/), the in-process analytical SQL database, and its command-line shell. DuckDB queries CSV, Parquet and JSON files where they are and writes results back to any of those formats or to a database file. The images ship the `duckdb` binaries from each DuckDB release, unchanged and checked against their SHA-256 digests. They are built on Ubuntu and Alpine for `linux/amd64` and `linux/arm64`, and rebuilt when DuckDB publishes a release and when the base image changes.

This is an unofficial build, not affiliated with or endorsed by the DuckDB Foundation or DuckDB Labs. Report problems with the image in this repository and DuckDB bugs at [duckdb/duckdb](https://github.com/duckdb/duckdb/issues).

## Quick start

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/duckdb \
  -c "SELECT * FROM 'sales.csv' LIMIT 10"
```

Convert a CSV file to Parquet:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/duckdb \
  -c "COPY (FROM 'sales.csv') TO 'sales.parquet' (FORMAT parquet)"
```

The entrypoint runs `duckdb` under `tini` in `/work`, so file names are relative to the directory you mount. Without a database file, DuckDB works in memory. A few more commands:

```sh
# Open the shell on a database file, which is created if it does not exist
docker run --rm -it --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/duckdb \
  analytics.duckdb

# Run a SQL script against a database file
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/duckdb \
  -f report.sql analytics.duckdb

# Read SQL from stdin and print the result as JSON
echo "SELECT count(*) AS events FROM 'events/*.parquet'" \
  | docker run --rm -i --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/duckdb -json
```

The image has no default arguments: with `-it` and nothing else it opens the shell on an in-memory database, and without a terminal it runs the SQL on stdin. `duckdb -help` lists the options.

## Extensions

Parquet, JSON and ICU (time zones and collations) are compiled into the binary, together with DuckDB's core functions and the shell's autocomplete. Other extensions, such as `httpfs` for files over HTTPS and on S3, `spatial` and `excel`, are downloaded from DuckDB's extension repository the first time a query needs them, or with `INSTALL`. DuckDB checks the signature of every extension before it loads it.

Extensions are stored in `/cache/.duckdb/extensions`. Mount a volume on `/cache` to keep them between runs:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" -v duckdb-cache:/cache \
  ghcr.io/randomcontainers/duckdb -c "INSTALL spatial; LOAD spatial; SELECT * FROM ST_Read('parcels.geojson')"
```

The Alpine images get the musl builds of the extensions, which DuckDB publishes next to the glibc builds. With `--network none`, the compiled-in extensions and those already in `/cache` still work, and a query that needs any other extension fails.

## What is in the image

- `duckdb` in `/usr/local/bin`, unpacked from the DuckDB release download and otherwise unchanged: the glibc build (`duckdb_cli-linux-<arch>.gz`) on Ubuntu and the musl build (`duckdb_cli-linux-<arch>-musl.gz`) on Alpine.
- The distro's C++ runtime (`libstdc++` and `libgcc`), the only libraries `duckdb` needs besides the C library.
- CA certificates, for extension downloads and for `httpfs`.

DuckDB's documentation notes that its musl builds can be more than five times slower than the glibc builds for some workloads. Use the Ubuntu tags, which include `latest`, for heavy queries. `/usr/local/share/randomcontainers/duckdb/buildinfo` has the name and SHA-256 of the download, the SHA-256 of the unpacked binary and the version and platform it reports.

## Default or slim

DuckDB's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is a DuckDB release such as `1.5.6`. `<minor>` and `<major>` are its shorter forms, `1.5` and `1`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<minor>`, `<minor>-slim`, `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<minor>-ubuntu`, `<minor>-slim-ubuntu`, `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<minor>-alpine`, `<minor>-slim-alpine`, `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current DuckDB version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

A new major version of DuckDB can change the storage format of database files. Pin `<major>` or an exact version for jobs that keep database files between runs.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Each image contains DuckDB's binary for its own architecture and is built natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is `/cache` for every user, and `/cache` can be written by anyone. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

DuckDB keeps a write-ahead log (`<database>.wal`) next to a database file while it writes to it, so mount the directory, not only the file. Queries that do not fit in memory spill to `<database>.tmp` next to the database file, or to `.tmp` in `/work` for an in-memory database.

Under `/cache`, DuckDB keeps installed extensions and persistent secrets in `.duckdb/`, the shell history in `.duckdb_history`, and runs `.duckdbrc` at startup if it exists. Secrets created with `CREATE PERSISTENT SECRET` are stored unencrypted, so only mount a volume on `/cache` where that is acceptable.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new DuckDB release and are rebuilt when the base image changes. This example installs two extensions at build time, so that containers can load them without network access:

```dockerfile
FROM ghcr.io/randomcontainers/duckdb:slim-ubuntu@sha256:...
RUN duckdb -c "INSTALL httpfs; INSTALL spatial" && chmod -R a+rwX /cache/.duckdb
```

The extensions end up in `/cache/.duckdb/extensions` in the image; a volume mounted on `/cache` hides them. The `RUN` step runs as UID 1000, so without the `chmod`, containers that run under another UID can load these extensions but cannot install more or store persistent secrets. To install distro packages, switch to `USER root` and back to `USER 1000:1000` afterwards. The entrypoint is `["tini", "--", "duckdb"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new DuckDB releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/duckdb/` holds the version, the source URLs, the build information, the license files and `runtime-deps`, the list of distro packages DuckDB needs at run time beyond those in the base image.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/duckdb:latest \
  --repo randomcontainers/duckdb --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/duckdb:latest --format '{{ json .SBOM }}'
```

DuckDB is not installed from a distro package, so neither DuckDB nor the libraries compiled into it are in the SBOM, and image scanners do not report their advisories. Follow DuckDB's [security advisories](https://github.com/duckdb/duckdb/security/advisories) instead.

DuckDB publishes no checksum files or signatures for its binaries. The build checks each download against the SHA-256 digest that GitHub records for the release asset, pinned in `package.yml`, and stops unless the binary reports the version being built and the commit of its release tag. The same commit is checked out for the license files.

## Updates

The project checks the releases of [duckdb/duckdb](https://github.com/duckdb/duckdb/releases) every 15 minutes. A release is picked up once it is 24 hours old. The new version, the commit of its tag and the SHA-256 digests GitHub records for the four CLI downloads are then committed to `package.yml` and the images are rebuilt. Only the newest release is built, so bugfix releases of an older series, such as 1.4 LTS, are skipped; tags of older versions stay as they were last built. DuckDB 2.0 changes the libraries compiled into the binary, so the images stay on 1.x until their license files and tests have been updated for it.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_COMMIT=<commit from package.yml> \
  --build-arg CLI_AMD64_SHA256=<cli-amd64 sha256 from package.yml> \
  --build-arg CLI_ARM64_SHA256=<cli-arm64 sha256 from package.yml> \
  -t duckdb:local .
```

The build downloads the binaries of both architectures and unpacks the one for the target platform. For the Alpine image, use `Dockerfile.alpine` with `CLI_AMD64_MUSL_SHA256` and `CLI_ARM64_MUSL_SHA256`.

## Licenses

DuckDB is licensed under the MIT license. The binary also contains libraries that DuckDB keeps in its source tree, under their own terms:

- Apache-2.0: the Parquet format definitions, Apache Thrift, FastPFor, t-digest and Mbed TLS.
- BSD-2-Clause: jemalloc, LZ4, linenoise and concurrentqueue.
- BSD-3-Clause: RE2, Snappy, Zstandard, libpg_query and the HyperLogLog code from Redis.
- BSL-1.0: ska_sort.
- Zlib: pdqsort and the semaphore in concurrentqueue.
- PostgreSQL: the SQL parser, which is derived from PostgreSQL's.
- Unicode-DFS-2016 and ICU: ICU.
- Unicode-DFS-2015: the Unicode data in utf8proc.
- MIT: most of the other libraries, among them fmt, yyjson, miniz, Brotli and cpp-httplib.

Mbed TLS and Zstandard are also offered under the GPL; the image uses their Apache-2.0 and BSD-3-Clause terms. DuckDB's `LICENSE` and the license files under `third_party/`, `extension/*/third_party/`, `src/` and `tools/shell/` of the source tree at the release tag are in `/usr/local/share/randomcontainers/duckdb/licenses/`, at their paths in the tree (for example `third_party/zstd/LICENSE`). Snowball's is left out, because the binary does not contain Snowball. The same directory has the notices from the Redis sources and the PostgreSQL license, which the source tree lacks. The image's license label is `MIT AND Apache-2.0 AND BSD-2-Clause AND BSD-3-Clause AND BSL-1.0 AND Zlib AND PostgreSQL AND Unicode-DFS-2016 AND ICU AND Unicode-DFS-2015`. The Ubuntu and Alpine packages in the image keep their own licenses.

The source for each image:

- DuckDB: the binary is built by DuckDB from the `v<version>` tag of [duckdb/duckdb](https://github.com/duckdb/duckdb). `/usr/local/share/randomcontainers/duckdb/source` names the commit of that tag and the download URL of the binary.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

The files in this repository are available under the MIT license, see [LICENSE](LICENSE), except the license texts in `licenses/`, which keep their own terms.

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
