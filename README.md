# PureScript Registry

[![Import Packages](https://github.com/purescript/registry-preview/actions/workflows/importer.yml/badge.svg)](https://github.com/purescript/registry-preview/actions/workflows/importer.yml)
[![Registry API](https://github.com/purescript/registry-preview/actions/workflows/api.yml/badge.svg)](https://github.com/purescript/registry-preview/actions/workflows/api.yml)

The PureScript Registry stores PureScript packages (and metadata about them) and provides an API for registering, updating, transferring, and unpublishing packages. Most PureScript users will only interact with the registry via package managers.

:warning: As of September 2022, no package manager officially supports the registry. To register your package, please open an issue using the [Add a Package](#add-a-package) workflow. Once registered, your package will be added to the package sets automatically (if it compiles with the current package set), its docs will be uploaded to Pursuit, and any SemVer tag you publish on the repository in the future will automatically update your package in the registry.

## Registry Overview

This repository provides three things:

1. Metadata for every package version in the registry, stored in the `metadata` directory
2. Package sets, which are updated daily and stored in the `package-sets` directory
3. An API for registering, updating, transferring, and unpublishing packages, as well as for making manual package set updates

The PureScript Registry as a whole consists of this repository plus a few more resources:

1. The [Registry Index](https://github.com/purescript/registry-index) stores the manifest files for every package version in the registry. Package managers use the registry index to rapidly determine relationships among packages at specific versions.
2. The [Registry Storage](https://packages.registry.purescript.org) stores tarballs of packages that have been uploaded to the registry. Package managers download dependencies from the registry storage.
3. The [Registry Dev](https://github.com/purescript/registry-dev) repository is used to develop the registry. The registry-dev repository is also a library, which package managers can use to easily integrate with the registry by reusing its types and functions.

## Using the Registry API

Most PureScript users will only use the Registry API via their package manager. However, it is also possible to interact with it directly via GitHub issues in this repository. The guide below separates _Operations_ (API calls anyone can make) from _Authenticated Operations_ (API calls that can only be made by package owners or registry trustees).

When you submit an issue with the correct payload, this repository's CI will execute the API. Any errors will be reported as a comment on the issue you opened; you can call the API again by commenting on the same issue you opened with another valid payload. If the operation completes successfully then that will be reported to you in a comment and the registry will close the issue.

### Operations

The Registry API allows anyone to add a package, update a package, or upgrade the package set. To add or update a package, your package must contain a `bower.json` file, `spago.dhall` file, or [`purs.json` manifest file](https://github.com/purescript/registry/blob/24db591b992b99730ea23211d746bc4c5fe295d7/src/Registry/Schema.purs#L21-L30). Eventually, the registry will no longer accept packages that do not have a registry manifest file, but for now legacy manifests are still accepted. For an example of the new manifests, see the [manifests for the `prelude` package](https://github.com/purescript/registry-index/blob/main/pr/el/prelude).

#### Add a Package

To register a new package, open an issue with the JSON representation of the [Addition](https://github.com/purescript/registry-dev/blob/5ae9781a5cbacaec71d46a7106db7579af0b707c/src/Registry/Schema.purs#L248-L253) type. For example, you can copy the below payload into a [new issue on this repository](https://github.com/purescript/registry/issues/new), change the fields to match your library, and submit it.

```jsonc
{
  "newPackageLocation": {
    "githubOwner": "purescript",
    "githubRepo": "purescript-prelude"
  },
  "newRef": "v1.0.0",
  "packageName": "prelude",
  "buildPlan": { "compiler": "0.15.4" }
}
```

The registry will fetch the repository at the provided ref and attempt to register the package, compile it with the compiler version in the `buildPlan` field, upload the documentation to Pursuit, and add the package to the day's package set batch.

> For the `buildPlan` you must provide a compiler version. You can also optionally provide `resolutions`, which is an object in which keys are dependency names and values are specific versions. If you do not provide resolutions then the registry will solve your dependencies for you. If you do, then the registry will compile your package using your provided resolutions.

#### Update a Package

To publish a new package version, open an issue with the JSON representation of the [Update](https://github.com/purescript/registry-dev/blob/5ae9781a5cbacaec71d46a7106db7579af0b707c/src/Registry/Schema.purs#L255-L259) type. The registry will reuse the location stored for the given package in metadata to fetch the package at the provided ref, compile it with the compiler version in the `buildPlan` field, upload the documentation to Pursuit, and add the package to the day's package set batch. As with additions, you can optionally provide `resolutions` as part of your build plan.

Example payload:

```jsonc
{
  "packageName": "prelude",
  "updateRef": "v2.0.0",
  "buildPlan": { "compiler": "0.15.4" }
}
```

#### Modify the Package Set

Package sets are released once per day automatically. However, they sometimes need manual intervention (for example, to update a set of packages all together). The registry provides a batch package set update API for this purpose. You can add new packages or update package versions using the JSON representation of the [PackageSetUpdate](https://github.com/purescript/registry-dev/blob/5ae9781a5cbacaec71d46a7106db7579af0b707c/src/Registry/Schema.purs#L261-L264) type. The registry will update the package set to the indicated versions and recompile it. If successful, it will release a new package set with your changes.

Only Registry Trustees are able to remove packages (by setting their value to `null` in the payload) or upgrade the compiler version (by setting the `compiler` field). If you are not a Trustee, then you should just set the `packages` field to the new desired versions for the packages you wish to update.

Example payload:

```jsonc
{
  "packages": {
    "aff": "7.1.0",
    "argonaut": "9.1.0"
  }
}
```

### Authenticated Operations

The Registry API allows package owners or Registry Trustees to transfer or unpublish packages. A "package owner" is an [Owner](https://github.com/purescript/registry/blob/24db591b992b99730ea23211d746bc4c5fe295d7/src/Registry/Schema.purs#L55-L59) listed in the package manifest; if you wish to take an authenticated action for a package then you must first publish a version in which your SSH public key is listed in the `owners` field. For example:

```jsonc
// Your public key will list these three fields in the format <keytype> <public> <email>
{
  "email": "owner@email.org", // this does not need to be a valid email address, but it must match the key
  "keytype": "ssh-ed25519",
  "public": "AAAAC3NzaC1lZDI1NTE5AAAAIK0wmN/Cr3JXqmLW7u+g9pTh+wyqDHpSAQEIg9p"
}
```

If your key is listed in the package owners, then you can transfer or unpublish your package. To do so, create the JSON payload for one of the operations below, and then sign the payload with your SSH key and submit the result in the [Authenticated](https://github.com/purescript/registry-dev/blob/5ae9781a5cbacaec71d46a7106db7579af0b707c/src/Registry/Schema.purs#L225-L246) type. Let's walk through a short example of transferring a package.

First, we produce the payload for the operation we wish to take. For a transfer, that might be:

```json
{
  "packageName": "my-package",
  "newPackageLocation": {
    "githubOwner": "new-owner",
    "githubRepo": "purescript-my-package"
  }
}
```

Next, place the JSON for your authenticated operation in a file. Then, you can sign the file with your SSH key using this command:

```console
ssh-keygen -Y sign -f ~/.ssh/id_ed25519 -n file payload_file
```

The signature is written to a new file called payload_file.sig, which looks like this:

```
-----BEGIN SSH SIGNATURE-----
U1NIU0lHAAAAAQAAADMAAAALc3NoLWVkMjU1MTkAAAAg2rirQQddpzEzOZwbtM0LUMmlLG
krl2EkDq4CVn/Hw7sAAAAEZmlsZQAAAAAAAAAGc2hhNTEyAAAAUwAAAAtzc2gtZWQyNTUx
OQAAAEDyjWPjmOdG8HJ8gh1CbM8WDDWoGfm+TTd8Qa8eua9Bt5Cc+43S24i/JqVWmk98qV
YXoQmOYL4bY8t/q7cSNeMH
-----END SSH SIGNATURE-----
```

Alternately, if you use `-` as the filename, then the payload will be read from stdin and the signature will be written to stdout. Now, we can produce our authenticated operation:

```jsonc
{
  "payload":
    {
      "packageName": "my-package",
      "newPackageLocation": {
        "githubOwner": "new-owner",
        "githubRepo": "purescript-my-package"
      }
    },
  "signature": [
    "-----BEGIN SSH SIGNATURE-----",
    "U1NIU0lHAAAAAQAAADMAAAALc3NoLWVkMjU1MTkAAAAg2rirQQddpzEzOZwbtM0LUMmlLG"
    "krl2EkDq4CVn/Hw7sAAAAEZmlsZQAAAAAAAAAGc2hhNTEyAAAAUwAAAAtzc2gtZWQyNTUx",
    "OQAAAEDyjWPjmOdG8HJ8gh1CbM8WDDWoGfm+TTd8Qa8eua9Bt5Cc+43S24i/JqVWmk98qV",
    "YXoQmOYL4bY8t/q7cSNeMH",
    "-----END SSH SIGNATURE-----"
  ],
  "email": "owner@email.com"
}
```

You can now submit this payload to the registry to process the authenticated operation.

#### Transfer a Package

To transfer a package, create an authenticated payload for the [Transfer](https://github.com/purescript/registry-dev/blob/5ae9781a5cbacaec71d46a7106db7579af0b707c/src/Registry/Schema.purs#L272-L275) type. Provide the package name to transfer and the new location the registry should fetch from.

Example payload:

```json
{
  "packageName": "math",
  "newPackageLocation": {
    "githubOwner": "purescript-deprecated",
    "githubRepo": "purescript-math"
  }
}
```

#### Unpublish a Package Version

Packages versions can be unpublished for a short time after being published. To unpublish a recently-published package version, create an authenticated payload for the [Unpublish](https://github.com/purescript/registry-dev/blob/5ae9781a5cbacaec71d46a7106db7579af0b707c/src/Registry/Schema.purs#L266-L270) type.

Example payload:

```json
{
  "packageName": "halogen-hooks",
  "unpublishVersion": "2.0.0",
  "unpublishReason": "Accidentally committed key."
}
```
