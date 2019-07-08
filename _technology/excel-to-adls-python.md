---
classes: "wide"
title: "Writing Excel file to Azure Data Lake Store Gen-1 using Python and Pandas"
excerpt: "Using Azure Data Lake Store Python SDK"
---
## Azure Python SDK
Whenever dealing with Azure resources in Python code, [Azure Python SDK](https://docs.microsoft.com/en-us/python/azure/?view=azure-python) comes handy. Having a native Python API maintained by Azure saves time and effort required to create custom wrappers over Azure REST API.

## SDK Components for Azure Datalake Store Gen-1 
The [Azure Data Lake Store Gen-1](https://docs.microsoft.com/en-us/python/api/overview/azure/data-lake-store?view=azure-python) (ADLS) section of API consists of [management](https://docs.microsoft.com/en-us/python/api/azure-mgmt-datalake-store/?view=azure-python) and [client](https://docs.microsoft.com/en-us/python/api/azure-datalake-store/azure.datalake.store?view=azure-python) parts.  

Management package consists of modules required for resource management activities relating to data lake store, such as creating or deleting data lake store itself or configurations such as firewall and virutal network settings. We don't need management package for our task. However, you can install it with:
> pip install azure-mgmt-datalake-store

Client package is what we will need for the task at hand (writing files). It consists of modules for authenticating as a client (consumer of data) and downloading/uploading (other filesystem operations as well) files. Install client package with:
> pip install azure-datalake-store

We need `lib` and `core` moodules from client package.

`lib` module contains an `auth` function that we use for authenticating to ADLS as a client using serivce principals and secret from an application registered in Azure AD.

`core` module contains `AzureDLFileSystem` class that contains many function providing [file system like operations](https://docs.microsoft.com/en-us/python/api/azure-datalake-store/azure.datalake.store.core.azuredlfilesystem?view=azure-python) (creating, deleting, listing, moving, copying files and directories, access control lists and many more) on ADLS directly from your Python code. This class brings great convenience for managing data on ADLS.

## Example Code
The following code makes use of `open` function in `AzureDLFileSystem` class of `core` module:

<script src="https://gist.github.com/arunchandrapant/55ae9cb487fc8a47d6b034c591f767fd.js"></script>

## Code Explaination
`open` function in `AzureDLFileSystem` is like the open function from base Python used for reading-writing files into local file system. It accepts stream of bytes as input through `write` function.

Therefore, to write anything into ADLS you just need `open` function and a stream of object being written. There also exists a `put` function that can pick an object directly from file system and write it to ADLS. However, many times it's convenient to just write the stream directly, rather than first writing it to local file system and then using `put`. In my view `open` is a more generalized approach to writing data to ADLS.

## The very useful BytesIO() from io module
However, in some cases it might not be straightforward to obtain stream that you want to write to ADLS. Our task in hand is one such case. `ExcelWriter` function in `pandas` package expects a file like object to which it writes the excel file. In other words it wants to write file directly into file system. Under such conditions `BytesIO` function from `io` package turns out to be very useful. It simulates in memory stream as if it were a file object. Now `ExcelWriter` can easily write to it. When passing it into `open` function from `AzureDLFileSyste`, use `getvalue()` method of `BytesIO` object to get the stream back.

In my experience `io` package is a very useful one. It helps simplify things a lot in code. I recommend getting familiar with it.

