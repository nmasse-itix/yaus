# Design Principles

## Context

*URL shortening is a technique in which a URL may be made substantially shorter and still direct to the required page*,
[tells us Wikipedia](https://en.wikipedia.org/wiki/URL_shortening). URL Shorteners are ubiquitous but nevertheless have
some drawbacks.

- They are run by big companies that can collect usage data and thus impair privacy.
- Public URL shortening services usually bind their users with an EULA.
- There is no guarantee that short URLs from a public service will be kept active.
  In fact, they can disappear at anytime if the backing company goes bankrupt.

And last but not least, all URL shorteners, be it a public service or a
self-hosted one will complicate the tasks of the web archeologist that will
try to re-construct the history of the Web from the available evidences in ten,
fifty or hundred years.

All those URL shortener keep their mapping table (short URL -> long URL) private
and this means the user (the one accessing the short URL, not the one creating it)
is completely dependent from the URL shortening service.

## Project goals

This project strives to provide a URL shortening service, that:

- is **public and transparent**: the mapping table (short URL -> long URL) is
  public and versioned in a GIT repository. Everyone can fork it, keep it in
  a safe place or update it with his own URLs.
- is **self-hostable** and respects the user **privacy**: instead of a couple
  of big instances of the **URL shortening service**, we would like to
  encourage smaller instances that would drive significantly less traffic
  and less temptation to drive income from usage data.

## Technical Design

### A GIT Repository to store the mapping table

The mapping table is stored in a GIT repository as a file or collection of files.
The file(s) contains the mapping table in a format that is easy to write for a
human, easily parseable by the machine and "merge-friendly".

The user wanting to add a short URL to the mapping table could:

- Fork the repository containing the mapping table
- Add his mappings to the table
- Create a branch with his modifications
- Commit his changes
- Push them to his own fork
- Submit a Pull Request to ask for inclusion in the main mapping table

The Pull Request could be subject to approval, review, etc.

## Auto-generated vs custom code

Usual URL Shorteners can generate a random short code or take a custom code
from the user.

In order to be as stateless as possible, the generated short code cannot be
random but needs to be generated deterministically. It can be a hash from the
input URL for instance.

## Hashing algorithm

URL Shorteners such as bit.ly use a combination of lower case, upper case
letters plus digits to generate a random code. The code is seven characters
long.

This translates to about 41 bits of entropy:

```raw
$ echo 'l(62^7)/l(2)' |bc -l
41.67937417270812646207
```

To implement a similar mechanism but fully deterministic, the SHA256 algorithm
is used to hash the target URL and the first six bytes are encoded in base64.

The short code for `https://framasoft.org/` is computed as such:

```raw
$ echo -n https://framasoft.org/ | openssl dgst -sha256 -binary |head -c6 |openssl base64 -e
t0P0JMya
```

## File Format of the mapping table

The mapping table uses YAML as file format it fits all the requirements.

A sample mapping table looks like:

```yaml
---
base_url: https://short.code/
mapping:
- url: https://framasoft.org/
- url: https://www.gnu.org/
  short-code: gnu-home
```

This sample table defines two entries:

- `https://short.code/t0P0JMya` that maps to `https://framasoft.org/`
- `https://short.code/gnu-home` that maps to `https://www.gnu.org/`

## Deployment

The app is packaged and deployed as a container. In this case, we need
to take into account that the filesystem might be read-only. Updates of the
mapping table comes with a new deployment of an updated image of the container.

In order to achieve rolling updates without service interruption, a health probe
needs to be implemented.

When the app is deployed outside of a container, an update of the mapping table
can be triggered by a `git pull` (from a crontab for instance). The app needs
to monitor the file containing the mapping table and hot reload the file once
modifications are detected.

## Coding principles

This app follows the [12 factors](https://12factor.net/).

## Minimal Viable Product

The MVP of this project has the following features:

- reads only one mapping table
- serves the requests for only one domain
- supports auto-generated and custom codes
- packaged as a container
