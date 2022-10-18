# Stedi read EDI demo

This repo contains an end-to-end demo for reading X12 EDI documents. This implementation demonstrates one possible way to interact with Stedi's APIs to achieve a typical EDI read workload; your implementation may include some or all of these products depending on your particular systems and requirements. 

The main orchestration point of this demo is a Stedi function called `read-inbound-edi`, which is written in [TypeScript](src/functions/read/inbound-edi/handler.ts). For this workload, the function is invoked when new items are uploaded to the [SFTP](https://www.stedi.com/docs/sftp)-enabled [Bucket](https://www.stedi.com/docs/bucket). The function processes X12 EDI documents that are uploaded to any directory named `inbound` within the bucket, converting it to a JSON payload that is then sent to an external webhook.

As the illustration below shows, the `read-inbound-edi` function performs several steps:

1. Accepts a bucket notification event that is generated when files are written to the bucket

2. Retrieves the contents of the uploaded file from the bucket

3. Calls the EDI Translate API, along with the `guideId`

4. The EDI Translate API retrieves the guide, validates that the EDI contents conform to the guide's JSON Schema specification, and converts the data to a JSON that conforms to the JSON Schema associated with the guide

5. Passes the JSON output from the EDI Translate API to [Mappings](https://www.stedi.com/docs/mappings) using a predefined Mapping. The mapping converts the JSON that was generated by the EDI Translate API to a custom JSON shape used by an external API

6. Calls an external webhook with the JSON output from the mapping operation as the payload

7. The function finally deletes the uploaded file from the bucket after it has been processed

![read-inbound-edi function flow](./assets/read-edi.svg)

## Resource directories

The [resources](./src/resources) directory contains templates that can be used to read an EDI document for specific transaction sets. The resource templates are organized in a hierarchical structure of subdirectories in the following pattern: `src/releases/${ediStandard}/${release}/${transactionType}` (for example: `src/releases/X12/5010/850`). Each template directory includes:
* `guide.json`: a [Guide](https://www.stedi.com/docs/guides) used to read and validate the EDI document and translate it to the JSON Schema of the guide
* `map.json`: a [Mapping](https://www.stedi.com/docs/mappings) that converts the data from the JSON schema of the guide to a custom JSON shape
* `input.edi`: the sample EDI input to the workflow  

## Prerequisites

1. [Node.js](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm) _(`npm` version must be 7.x or greater)_

1. Clone this repo and install the necessary dependencies: 

   ```bash
   git clone https://github.com/Stedi-Demos/read-edi-demo.git
   cd read-edi-demo
   npm ci
   ```
   
1. Go to [webhook.site](https://webhook.site/) and copy the unique URL. The demo will send output to this webhook 

1. This project uses `dotenv` to manage the environmental variables required. You must create a `.env` file in the root directory of this repo and add the three required environment variables:
   * `STEDI_API_KEY`: Your Stedi API Key - used to deploy the function and internally to interact with product APIs. If you don't already have one, you can generate an [API Key here](https://www.stedi.com/app/settings/api-keys). 
   * `ENABLED_TRANSACTION_SETS`: a comma separated list of transaction sets for which you would like to be able to read inbound EDI documents. The values in the list MUST match available resource sets under the [resources](./src/resources) directory, and conform to the pattern `${ediStandard}-${release}-${transactionType}` (for example: `X12-5010-855`). Note: you can always come back and add or remove entries from this list. After doing so, you'll just need to re-run the `create-guides`, `create-mappings`, and `deploy` steps described below in order to apply the changes.
   * `DESTINATION_WEBHOOK_URL`: the unique URL copied from [webhook.site](https://webhook.site/) in the previous step
   * `PROGRESS_TRACKING_WEBHOOK_URL` [_optional but recommended_]: the unique URL copied from [webhook.site](https://webhook.site/) in the previous step -- include this environment variable to enable additional progress tracking and improve observability of the function executions. For more details on the progress tracking, [see below](#additional-progress-tracking).

   Example `.env` file:

    ```
    STEDI_API_KEY=<REPLACE_ME>
    ENABLED_TRANSACTION_SETS=X12-5010-850,X12-5010-855
    DESTINATION_WEBHOOK_URL=https://webhook.site/<YOUR_UNIQUE_ID>
    PROGRESS_TRACKING_WEBHOOK_URL=https://webhook.site/<YOUR_UNIQUE_ID>
    ```
   
  The subsequent setup scripts will use the `ENABLED_TRANSACTION_SETS` environment variable to determine which resources to deploy to your account.

1. Create the Stedi guides by running:

   ```bash
   npm run create-guides
   ```
   
   For each guide that is created, an environment variable entry will automatically be added to the `.env` file. The output of the script will include a list of the environment variables that have been added:

   ```bash
   Updated .env file with 2 guide entries:

   X12_5010_850_GUIDE_ID=01GEGDX8T2W6W2MTEAKGER7P3S
   X12_5010_855_GUIDE_ID=01GEFFW2G33BCYDDKZ7H62T5Z5
   ```

1. Create the mappings by running:

   ```bash
   npm run create-mappings
   ```

   For each mapping that is created, an environment variable entry will automatically be added to the `.env` file. The output of the script will include a list of the environment variables that have been added:
 
   ```bash
   Updated .env file with 2 mapping entries:

   X12_5010_850_MAPPING_ID=01GEGCDRB3F1YQSENAWG9978Y7
   X12_5010_855_MAPPING_ID=01GE0W06T3M0GS1V1AYYCK50HD
   ```

1. Configure the buckets (one for SFTP access and one for tracking function executions):

   ```bash
   npm run configure-buckets
   ```

   For each bucket, an environment variable entry will automatically be added to the `.env` file. The output of the script will include a list of the environment variables that have been added:

   ```bash
   Updated .env file with 2 bucket entries:

   SFTP_BUCKET_NAME=4c22f54a-9ecf-41c8-b404-6a1f20674953-sftp
   EXECUTIONS_BUCKET_NAME=4c22f54a-9ecf-41c8-b404-6a1f20674953-executions
   ```

## Setup & Deployment

This repo includes a basic deployment script to bundle and deploy the `read-inbound-edi` function to Stedi. To deploy you must complete the following steps:

1. Confirm that your `.env` file contains the necessary environment variables: 
   - `STEDI_API_KEY` 
   - `ENABLED_TRANSACTION_SETS` 
   - `DESTINATION_WEBHOOK_URL`
   - `SFTP_BUCKET_NAME`
   - `EXECUTIONS_BUCKET_NAME`
   - For each transaction set in the `ENABLED_TRANSACTION_SETS` list:
     - a `<TXN_SET>_GUIDE_ID` variable
     - a `<TXN_SET>_MAPPING_ID` variable

   It should look something like the following:

   ```
   STEDI_API_KEY=<YOUR_STEDI_API_KEY>
   ENABLED_TRANSACTION_SETS=X12-5010-850,X12-5010-855
   DESTINATION_WEBHOOK_URL=https://webhook.site/<YOUR_UNIQUE_ID>
   PROGRESS_TRACKING_WEBHOOK_URL=https://webhook.site/<YOUR_UNIQUE_ID>
   X12_5010_850_GUIDE_ID=01GEGDX8T2W6W2MTEAKGER7P3S
   X12_5010_855_GUIDE_ID=01GEFFW2G33BCYDDKZ7H62T5Z5
   X12_5010_850_MAPPING_ID=01GEGCDRB3F1YQSENAWG9978Y7
   X12_5010_855_MAPPING_ID=01GE0W06T3M0GS1V1AYYCK50HD
   SFTP_BUCKET_NAME=4c22f54a-9ecf-41c8-b404-6a1f20674953-sftp
   EXECUTIONS_BUCKET_NAME=4c22f54a-9ecf-41c8-b404-6a1f20674953-executions
   ```

1. To deploy the function:
   ```bash
   npm run deploy
   ```

   This should produce the following output:

   ```
   > stedi-read-edi-demo@1.0.0 deploy
   > ts-node ./src/setup/deploy.ts

   Deploying read-inbound-edi
   Done read-inbound-edi
   Deploy completed at: 9/14/2022, 08:48:43 PM
   ```
   
1. Enable bucket notifications for the SFTP Bucket to invoke the `read-inbound-edi` function when files are written to the bucket.

   ```bash
   npm run enable-notifications
   ```

## Invoking the function

Once deployed, the function will be invoked when files are written to the SFTP bucket.

1. Using the [Buckets UI](https://www.stedi.com/app/buckets) navigate to the `inbound` directory for your trading partner: `<SFTP_BUCKET_NAME>/trading_partners/ANOTHERMERCH/inbound`

2. Upload the [input X12 5010 855 EDI](src/resources/X12/5010/855/input.edi) document to this directory.  You can choose an `input.edi` for any one of the [transaction sets](src/resources) included in the `ENABLED_TRANSACTION_SETS` list, but we'll show X12-5010-855 in the examples below. (_note_: if you upload the document to the root directory `/`, it will be intentionally ignored by the `read-inbound-edi`).

1. Look for the output of the function wherever you created your test webhook! The function sends the JSON output of the mapping to the endpoint you have configured

    JSON Mapping output:
    ```json
    {
      "orderAcknowledgements": [
        {
          "orderAcknowledgementDetails": {
            "internalOrderNumber": "ACME-4567",
            "orderNumber": "365465413",
            "orderDate": "2022-09-14",
            "orderAckDate": "2022-09-13"
          },
          "seller": {
            "name": "Marvin Acme",
            "address": {
              "street1": "123 Main Street",
              "city": "Fairfield",
              "state": "NJ",
              "zip": "07004",
              "country": "US"
            }
          },
          "shipTo": {
            "customerId": "DROPSHIP CUSTOMER",
            "name": "Wile E Coyote",
            "address": {
              "street1": "111 Canyon Court",
              "city": "Phoenix",
              "state": "AZ",
              "zip": "85001",
              "country": "US"
            }
          },
          "items": [
            {
              "id": "item-1",
              "quantity": 8,
              "unitCode": "EA",
              "price": 400,
              "vendorPartNumber": "VND1234567",
              "sku": "ACM/8900-400"
            },
            {
              "id": "item-2",
              "quantity": 4,
              "unitCode": "EA",
              "price": 125,
              "vendorPartNumber": "VND000111222",
              "sku": "ACM/1100-001"
            }
          ]
        }
      ] 
    }
    ```

1. You can also view the other resources that were created during setup in the UIs for the associated products:
   - [Guides Web UI](https://www.stedi.com/app/guides): a guide for each transaction set that you enabled
   - [Mappings Web UI](https://www.stedi.com/app/mappings): a mapping for each transaction set that you enabled

1. [Optional -- Bonus / Extra Credit] Try invoking the workflow via SFTP!
   1. Provision an SFTP user, by visiting the [SFTP Web UI](https://www.stedi.com/app/sftp), be sure to set its `Home directory` to `/trading_partners/ANOTHERMERCH` and record the password (it will not be shown again)
   1. Using the SFTP client of your choice (the `sftp` command line client and [Cyberduck](https://cyberduck.io/) are popular options) connect to the SFTP service using the credentials for the SFTP user that you created.
   1. Navigate to the `/inbound` subdirectory
   1. Upload the [input X12 5010 855 EDI](src/resources/X12/5010/855/input.edi) document to the `/inbound` directory via SFTP
   1. view the results at your webhook destination!

## Function execution tracking

The `read-outbound-edi` function uses the bucket referenced by the `EXECUTIONS_BUCKET_NAME` environment variable to track details about each invocation of the function.

1. At the beginning of each invocation, an `executionId` is generated. The `executionId` is a hash of the function name and the input payload. This allows subsequent invocations of the function with the same payload to be processed as retries.
 
1. The Bucket Notification event input to the function is written to the executions bucket in the following location:

   ```bash
   functions/read-outbound-edi/${executionId}/input.json
   ```
1. If any failures are encountered during the execution of the function, the details of the failure are written to the following location:

   ```bash
   functions/read-outbound-edi/${executionId}/failure.json 
   ```
   
1. Upon successful invocation of the function (either on the initial invocation, or on a subsequent retry), the `input.json` as well as the `failure.json` from previous failed invocations (if present) are deleted. Therefore, any items present in the `executions` bucket represent in-progress executions, or previously failed invocations (with details about the failure).

## Additional progress tracking

Additional progress tracking can be enabled by including the `PROGRESS_TRACKING_WEBHOOK_URL` environment variable in your `.env` file. The additional progress tracking provides more visibility into the process of translating the input X12 EDI document into the custom JSON output shape. The `read-inbound-edi` function records additional details as it processes documents, and it sends this output to the destination webhook URL. You can change the destination for the additional progress tracking by changing the corresponding environment variable (or remove the environment variable completely to disable this additional tracking):

  ```
  PROGRESS_TRACKING_WEBHOOK_URL=https://webhook.site/<YOUR_UNIQUE_ID>
  ```

Note: after updating (or removing) this environment variable, you will need to also update (or remove) the environment variable from the deployed `read-inbound-edi` function. You can do this via the [Functions UI](https://www.stedi.com/terminal/functions/read-inbound-edi), or by re-running `npm run deploy`.