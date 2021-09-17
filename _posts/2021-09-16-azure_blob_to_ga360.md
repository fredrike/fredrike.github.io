---
title: Writing a file from Azure Blob to Google Analytics 360 with file upload
last_modified_at: 2021-09-17 10:00:00 +0200

---

The following describes how an Azure Function written in Python3 can read a file from Azure Blob Storage and upload it Google Analytics 360 for be used as a custom data source. The function uses the uploadData API described in [Google Analytics Management API](https://developers.google.com/analytics/devguides/config/mgmt/v3/mgmtReference/management/uploads/uploadData).

This writeup assumes that there already exists an Azure Function configured with the Python runtime (tested with Python3.8).

## Create a Service Accunt on Google Console

Follow the instructions [here](https://developers.google.com/analytics/devguides/config/mgmt/v3/authorization#service_accounts) and download the *public/private key* as `json`. Also, make sure that the newly created user (identified by an email address) have edit access in Google Analytics 360.

## Create code for the function app

The following is an example Python code that can be used to read an file from Azure Blob storage and upload it to Google Analytics 360.

File `${FUNCTION_APP}/requrements.txt`

```pip
azure-functions==1.6.0
azure-storage-blob==12.5.0

oauth2client==4.1.3
chardet==3.0.4
aiohttp==3.7.3
```

File `${FUNCTION_APP}/read_secrets/function.json`

```json
{
  "scriptFile": "__init__.py",
  "bindings": [
    {
      "authLevel": "function",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "methods": ["get"]
    },
    {
      "type": "http",
      "direction": "out",
      "name": "$return"
    }
  ]
}
```

File `${FUNCTION_APP}/read_secrets/__init__.py`

```python
"""Method to read file from blob and write to Google Analytisc."""
import json
import logging
import traceback

import azure.functions as func
from aiohttp import ClientSession, TCPConnector
from azure.storage.blob.aio import ContainerClient
from oauth2client.service_account import ServiceAccountCredentials


GA_SCOPE = "https://www.googleapis.com/auth/analytics.edit"
GA_ACCOUNT_ID = ""
GA_WEBPROPERTY_ID = ""
GA_CUSTOM_DATA_SOURCE_ID = ""
GA_UPLOAD_URL = (
    f"https://www.googleapis.com/upload/analytics/v3/management/accounts/"
    f"{GA_ACCOUNT_ID}/webproperties/{GA_WEBPROPERTY_ID}/customDataSources/"
    f"{GA_CUSTOM_DATA_SOURCE_ID}/uploads"
)

CONTAINER_NAME = "azure blob container name"
FILE_NAME = "filename including path in blob"

secrets = (
  GBQ_SECRET = "json key for the service account"
  BLOB_CONNECTION_STRING = "connection string to the blob"
)

async def read_blob(connection_string, blob_name):
    """Function that returns chunks of a file."""
    async with ContainerClient.from_connection_string(
        conn_str=connection_string, container_name=CONTAINER_NAME
    ) as con:
        blob = con.get_blob_client(blob_name)
        data = await blob.download_blob()
        return await data.readall()


async def main(req: func.HttpRequest) -> func.HttpResponse:
    """Main function."""
    logging.info("Python HTTP trigger function processed a request.")

    try:
        # Create token to be used when accessing Google Analtics.
        token = (
            ServiceAccountCredentials.from_json_keyfile_dict(
                json.loads(secrets[GBQ_SECRET]),
                scopes=GA_SCOPE,
            )
            .get_access_token()
            .access_token
        )
        status = ""
        async with ClientSession(connector=TCPConnector(limit=5)) as session:
            file_data = await blob_sender(
                    connection_string=secrets[BLOB_CONNECTION_STRING],
                    blob_name=FILE_NAME,
            )
            # Post file to GA_UPLOAD_URL
            async with session.post(
                GA_UPLOAD_URL,
                data=file_data,
                params={
                    "access_token": token,
                    "uploadType": "media",
                    "content_type": "application/octet-stream",
                }
            ) as resp:
                status = await resp.text()
                logging.info("ga_upload %s", status)

        return func.HttpResponse(
            status,
            mimetype="application/json",
            status_code=200,
        )

    except Exception:
        return func.HttpResponse(
            traceback.format_exc(),
            status_code=500,
        )
```

Don't forget to publish the updated function with:

```sh
func azure functionapp publish ${FUNCTION_APP}
```

Last modified at: 2021-09-17.
