# Open Release Definition
Open Release Definition Format describes discrete software releases in terms of
their life cycle, notating when bugs and vulnerabilities were introduced or
removed.

# What is ORDF?

ORDF, in a nutshell, specifies a release hierarchy and its implications.  It
provides this via a JSON schema describing each release, what it supersedes,
and what it adds or removes.

ORDF documents can readily store into a document store such as MongoDB.  This
allows for effecient querying and manipulation, generating ORDF on-demand.

# What does ORDF look like?

ORDF is in development.  The mock-ups here are subject to change.

An example ORDF document showing ISC Bind version 9.9.5 might look as below:

```json
{
  "name": "bind",
  "release": "9.7.0.",
  "supersedes": "9.6.9",
  "defects":
  [
    {
      "identifiers":
      {
        "cve": "2014-8500"
      },
      "nature":
      [
        "security"
      ]
    }
  ]
}

```

The above is, of course, incomplete.  There are other bugs, non-CVE bugs, and
so forth.  The identifier could also be a bug ID.

ISC Bind 9.9.6 repairs the CVE-2014-8500 defect:

```json
{
  "name": "bind",
  "release": "9.9.6.",
  "supersedes": "9.9.5",
  "repairs":
  [
    {
      "identifiers":
      {
        "cve": "2014-8500"
      },
    }
  ]
}

```

For Ubuntu Trusty LTS (14.04), the Bind9 package fixing this might follow the
below chain:

```json
{
  "name": "bind9",
  "release": "1:9.9.5.dfsg-1",
  "upstream":
  {
    "vendor": "isc",
    "product": "bind",
    "release": "9.9.5"
  }
}
{
  "name": "bind9",
  "release": "1:9.9.5.dfsg-2",
  "supersedes": "1:9.9.5.dfsg-1"
}
{
  "name": "bind9",
  "release": "1:9.9.5.dfsg-3",
  "supersedes": "1:9.9.5.dfsg-2"
}
{
  "name": "bind9",
  "release": "1:9.9.5.dfsg-3ubuntu0.1",
  "supersedes": "1:9.9.5.dfsg-3",
  "repairs":
  [
    {
      "identifiers":
      {
        "cve": "2014-8500"
      },
      "advisories":
      {
        "usn": "2437-1"
      }
    }
  ]
}

```

This might involve three different feeds.  The Canonical Trusty feed might
provide the information for `1:9.9.5.dfsg-3ubuntu0.1`, while a separate
Debian-provided feed gives information for `1:9.9.5.dfsg-3`, and an ISC feed
might provide information for `9.9.5`.  The Debian feed also identifies ISC
`bind` as the basis for Debian `bind9`—or, rather, as the basis for a
specific release.

Given all of these, we can see a release chain:

`9.9.5`->`1:9.9.5.dfsg-1`->`1:9.9.5.dfsg-2`->`1:9.9.5.dfsg-3`->`1:9.9.5.dfsg-3ubuntu0.1`

This means we can see the `defect` in 9.9.5 by following the chain backwards,
and the repair at `1:9.9.5.dfsg-3ubuntu0.1`.

# Why ORDF?

Modern vulnerability scanners use many complex metrics to estimate what may
be vulnerable.  They use individually-built databases, OVAL XMLs, and NASL
scripts.  Sometimes, they have to interpret version numbers of packages and
determine if they're newer than other packages.

OVAL feeds, for exaple, may specify that bind9 `>=1:9.9.5.dfsg-3ubuntu0.1` is
not vulnerable to CVE-2014-8500.  The vulnerability scanner then needs to read
into this.  The question is:  what if bind9 `>=1:9.9.3.dfsg-1ubuntu0.11` is
not vulnerable?  Is `1:9.9.5.dfsg-1` newer?  Of course not.  The OVAL scanner
uses a feed specifically for Ubuntu 14.04, which must re-declare all
vulnerabilities.

ORDF attempts to create an unbroken chain of custody for defects and their
repairs.  This means Debian only needs to declare that their package is based
on bind9 `9.9.5`; Ubuntu only needs to declare their package based on bind9
`9.9.5.dfsg-3`; and each release version only needs to declare what
vulnerabilities were added or removed _in that release_.

# How do we use ORDF?

Simply put, ORDF feed maintainers update the document for any release which
encounters a defect.  If Bind `9.8.0` is based on Bind `9.7.6`, then a
defect added to `9.7.6` affects `9.8.0` and so forth.

Likewise, if the developers then add code to `9.7.8` and `9.8.1` and cause a
vulnerability in both versions, then the same defect must be added separately
B
to `9.7.8` and `9.8.1`—Bind `9.8.0` won't inherit from `9.7.8` because it's not
based on that code base ("release", here).

One might think the developers may merge `9.7.7` into the `9.8` branch when
they release `9.8.1`, meaning we somehow need to denote that `9.8.1` is based
on `9.7.7`.  On the other hand, parts of the code may have been removed in
`9.8.0`, and defects in `9.7.7` may not then be merged.  This might be best
handled by a rebase section:

```json
{
  "name": "bind",
  "release": "9.7.7",
  "supersedes": "9.7.6",
  "defects":
  [
    {
      "identifiers":
      {
        "cve": "2014-8550"
      },
      "nature":
      [
        "security"
      ]
    }
  ]
}
{
  "name": "bind",
  "release": "9.7.8",
  "supersedes": "9.7.7",
  "repairs":
  [
    {
      "identifiers":
      {
        "cve": "2014-8500"
      }
    }
  ]
}
{
  "name": "bind",
  "release": "9.8.0",
  "supersedes": "9.7.6"
}
{
  "name": "bind",
  "release": "9.8.1",
  "supersedes": "9.8.0",
  "rebase":
  {
    "from": "9.7.6",
    "to": "9.7.7",
    "exclude":
    [
      {
        "cve": "2014-8550"
      }
    ]
  }
}
```

In the above, `9.7.9` fixes a defect introduced in `9.7.8`; however, that code
isn't in the `9.8` line, so the defect isn't brought in by merging `9.7.7`.
That defect is excluded.

Ideally, ORDF would stretch back through an application's entire history; in
practice, you might not have a full history.  ORDF is useful only as an
unbroken chain.  Missing early history is okay if your application knows about
a bug, because it can identify that a certain software is vulnerable to
CVE-xxxx-yyyy and then trace back to the earliest known release.  If anything
along the line says it repaired CVE-xxxx-yyyy, then you know the version you're
looking at is not vulnerable to that defect.  Thus ORDF can supplement other
vulnerability scanner metadata.

