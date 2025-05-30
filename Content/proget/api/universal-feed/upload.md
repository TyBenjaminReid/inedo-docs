---
title: "Create/Upload Universal Package"
order: 2
---

*Create/Upload Universal Package* is an endpoint in [ProGet's Universal Feed API](/docs/proget/api/universal-feed) that will add or replace a package in a feed.

There are quite a few ways to access this endpoint, but the end result is the same. Because there are so many permutations of how you can use this, it's easiest to specify the various options and behaviors instead.

First and foremost, consider that a complete package consists of required metadata and content (arbitrary files and directories). This endpoint is designed to allow you to upload a complete, pre-built package, or upload a partial package with content and metadata you specified using path, query, form-encoded, and/or JSON parameters.

:::(Info) (Recommended: Using pgutil to Create and Upload Universal Packages)

While this endpoint will allow you create and upload a package to a Universal Feed, we recommend using [pgutil](/docs/proget/api/pgutil) commands to do this. this can easily be achieved using a combination of the `upack create` and `packages upload` commands:

```bash
$ pgutil upack create --name=myPackage --version=1.0.1 --source-directory=.\package-files\myPackage --target-directory=.\upacks

$ pgutil packages upload --feed=myUniversalFeed --input-file=.\upacks\MyPackage.1.0.1.upack

```
:::

### Creating a Universal Package from `.zip` or `.tgz`

This endpoint can also be used to take an archive in .zip or .tgz format, convert it to a Universal Package and upload it to ProGet. The metadata for this package will be created based on the parameters set.

:::(Info) (🚀 Quick Example: Creating and Uploading a Universal Package with Curl)
This example will convert a .zip of artifacts artifacts.zip to a Universal Package `myUniversalPackage 1.2.3` and then upload it to the feed `myUniversalFeed`, authenticating with the API key `abc12345`:

```
curl -X POST -H "Content-Type: application/zip" -H "X-ApiKey: abc12345" --data-binary "@C:\Proget\artifacts.zip" "https://proget.corp.local/upack/universal-feed/upload?name=myUniversalPackag&version=1.2.3"
```
:::

### Content Type
The Content-Type header can be any of the following:

* `application/json` - properties on the JSON object will be used for content and metadata parameters
* `application/x-www-form-urlencoded` - the key/value pairs will be used for content and metadata parameters
* `application/zip` - the request body will treated either as content or a partial package

:::(info) (📄 Note: Using application/zip Content-Type)
You must send the raw bytes of a ZIP file as the body of your request. If the archive doesn't conform to the [universal package format](/docs/proget/feeds/universal/universal-packages#manifest), ProGet will convert it for you, if you supply the required metadata via query string parameters.

If the archive is already in the .upack format, you can specify additional metadata parameters via the querystring.
:::

### Parameter	Description
Any of the following parameters fields may be specified through querystring or content; the format must follow a valid [metadata format](/docs/proget/feeds/universal/universal-packages#manifest) specification.

| Parameter | Details |
| --- | --- |
| `content-b64` | A string representing the contents as a base64-encoded ZIP archive; this is not valid with `application/zip` Content-Type, and will be considered duplicative if `content-url` is specified. |
| `content-url` | A url where content can be downloaded from as a ZIP archive; this is not valid with `application/zip` Content-Type, and will be considered duplicative if `content-b64` is specified |
| `group` | This may also be specified as the first path following the endpoint. |
| `name` | This may also be specified as either the last or second-to-last path. |
| `version` | This may also be specified as either the last path. |
| `dependencies` | When specified in JSON, it should be an array; otherwise (querystring or form format), it should be a comma-separated string of package identifiers. |
| _anything else_ | If any other parameter is specified (including the well-defined title, icon, description), it will be added as a package metadata property. |

## Request Specification
To upload a package, simply `POST` to the URL with a feed name and a JSON object in the body.

```
POST /upack/«feed-name»/upload
```

:::(warning) (⚠ No Duplicate Parameters)
If you specify a different package name in both the query and path, you'll get an error (400).
:::

**Uploading a Universal Package** requires the feed name (e.g. `myUniversalFeed`):

```
POST /upack/myUniversalFeed/upload

«contents of myUniversalPackage-1.2.3.upack»
```

## Response Specification

| Response | Details |
| --- | --- |
| **201 (Success)** | will successfully upload the package
| **400 (Package Error)** | returned if you specify a different package name in both the query and path |
|  **403 (Unauthorized API Key)** | indicates a [missing, unknown, or unauthorized API Key](/docs/proget/api/universal-feed#authentication); the body will be empty |


## Sample Usage Scripts

### Bulk upload of all packages (PowerShell)
This script will upload all `.upack` packages from the local folder `C:\ProGet\Package Uploads`, to the universal feed `universal-uploads`:

```powershell
$apiKey = "a1b2c3d4e5"
$directoryPath = "C:\ProGet\Package Uploads"
$uploadUrl = "https://proget.corp.local/upack/universal-uploads/upload"

$headers = @{
    "Content-Type" = "application/zip"
    "X-ApiKey" = $apiKey
}

$filePaths = Get-ChildItem -Path $directoryPath | Where-Object { $_.Extension -match '\.upack$' } | Select-Object -ExpandProperty FullName

if ($filePaths.Count -eq 0) {
    Write-Host "No files found in $directoryPath"
} else {
    foreach ($filePath in $filePaths) {
      Write-Host "Uploading file: $filePath"
      
      $response = Invoke-RestMethod -Uri $uploadUrl -Method Post -Headers $headers -InFile $filePath -ContentType "application/zip"
      
      Write-Host ("Response for $($filePath):")
      $response
    }
}
```

### Bulk Upload of all versions of a package (PowerShell)
This script will upload all versions of the `BaseModules` package from the local folder `C:\ProGet\Package Uploads`, to the universal feed `universal-uploads`:

```powershell
$apiKey = "a1b2c3d4e5"
$directoryPath = "C:\ProGet\Package Uploads"
$uploadUrl = "https://proget.corp.local/upack/universal-uploads/upload"
$specifiedPackageName = "BaseModules"

$headers = @{
    "Content-Type" = "application/zip"
    "X-ApiKey" = $apiKey
}

$filePaths = Get-ChildItem -Path $directoryPath | Where-Object { 
    $_.Extension -eq '.upack' -and $_.BaseName -like "$specifiedPackageName*" 
} | Select-Object -ExpandProperty FullName

if ($filePaths.Count -eq 0) {
    Write-Host "No files found in $directoryPath with the specified package name."
} else {
    foreach ($filePath in $filePaths) {
        Write-Host "Uploading file: $filePath"
        
        $response = Invoke-RestMethod -Uri $uploadUrl -Method Post -Headers $headers -InFile $filePath -ContentType "application/zip"
        
        Write-Host ("Response for $($filePath):")
        $response
    }
}
```