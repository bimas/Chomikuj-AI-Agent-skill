---
name: chomikuj
description: Use when the user wants to write a script/program to log in to chomikuj.pl, search, upload, download, and delete files, list folders and files, create and delete folders, and in any other case where the user wants to communicate with chomikuj.pl using the SOAP API or WEB HTTP scraping.
author: Maciej Bimek
---

# Chomikuj.pl / ChomikBox SOAP API WIKI

This wiki is a detailed, algorithmic guide to the unofficial SOAP API of the chomikuj.pl service (based on the communication protocol of the official desktop application **ChomikBox**). It enables the implementation of a client in any programming language (e.g., Python, C#, Java, Go, PHP, JavaScript).

## Table of Contents
* [1. General Information and SOAP Protocol](#1-general-information-and-soap-protocol)
  * [List of All API Methods (SOAP Actions)](#list-of-all-api-methods-soap-actions)
* [2. Login and Authorization (Auth)](#2-login-and-authorization-auth)
* [3. Listing Folders, Subfolders, and Files](#3-listing-folders-subfolders-and-files)
  * [A. Listing Folder Structure (Folders)](#a-listing-folder-structure-folders)
  * [B. Listing Files in a Folder (Download)](#b-listing-files-in-a-folder-download)
  * [C. Creating a Folder (AddFolder)](#c-creating-a-folder-addfolder)
  * [D. Removing a Folder (RemoveFolder)](#d-removing-a-folder-removefolder)
  * [E. Deleting a File (DeleteFileAction - Web Action)](#e-deleting-a-file-deletefileaction---web-action)
* [4. Generating a File Download URL](#4-generating-a-file-download-url)
* [5. Searching for a File](#5-searching-for-a-file)
  * [Method A: Local search (on user account)](#method-a-local-search-on-user-account)
  * [Method B: Web/HTTP Search (Scraping and search forms)](#method-b-webhttp-search-scraping-and-search-forms)
* [6. Uploading a File (Upload)](#6-uploading-a-file-upload)
  * [A. Retrieving an Upload Token (UploadToken)](#a-retrieving-an-upload-token-uploadtoken)
  * [B. Transferring File Data (HTTP POST Multipart/Mixed)](#b-transferring-file-data-http-post-multipartmixed)
  * [C. Resuming an Interrupted Upload (Resume)](#c-resuming-an-interrupted-upload-resume)
* [7. Monitoring Application State (CheckEvents)](#7-monitoring-application-state-checkevents)

---

## 1. General Information and SOAP Protocol

Communication with the Chomikuj.pl API is based on the SOAP 1.1 protocol, transmitted via HTTP POST requests.

* **Endpoint Address:** `http://box.chomikuj.pl/services/ChomikBoxService.svc`
* **Port:** `80`
* **Content-Type Header:** `text/xml;charset=utf-8`
* **SOAP Namespace (Method Namespace):** `http://chomikuj.pl/`
* **Common HTTP headers for each SOAP request:**
  ```http
  POST /services/ChomikBoxService.svc HTTP/1.1
  Host: box.chomikuj.pl
  Content-Type: text/xml;charset=utf-8
  SOAPAction: http://chomikuj.pl/IChomikBoxService/[MethodName]
  Content-Length: [Body_Length]
  Connection: Keep-Alive
  ```
* **Structure of the SOAP Envelope (Sent):**
  Each request must be wrapped in a standard SOAP envelope:
  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/" s:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/">
    <s:Body>
      <[MethodName] xmlns="http://chomikuj.pl/">
        <!-- Method parameters -->
      </[MethodName]>
    </s:Body>
  </s:Envelope>
  ```

---

## List of All API Methods (SOAP Actions)

All SOAP operations of the ChomikBox client are handled by the `IChomikBoxService` service contract. Below is a complete list and a brief summary of the purpose of all available methods in this API:

1. **`Auth`** (SOAPAction: `http://chomikuj.pl/IChomikBoxService/Auth`)
   - **Description:** Authenticates the user based on the account name and hashed password (MD5). Returns the session identifier (`token`) and the account identifier (`hamsterId`).
2. **`CheckEvents`** (SOAPAction: `http://chomikuj.pl/IChomikBoxService/CheckEvents`)
   - **Description:** Queries the server to report the current status of transfers (isUploading, isDownloading) and keep the session active. Without calling this method, the server will refuse to process the web session synchronization (log_www) successfully.
3. **`Folders`** (SOAPAction: `http://chomikuj.pl/IChomikBoxService/Folders`)
   - **Description:** Retrieves the user's directory tree structure (starting from folder ID `0`).
4. **`Download`** (SOAPAction: `http://chomikuj.pl/IChomikBoxService/Download`)
   - **Description:** A multi-functional method used to:
     - Retrieve the list of files in a given folder (when the identifier is `[chomikbox_url_id]/[folder_id]`, and disposition is `download`).
     - Generate a direct ticket/URL to download a file from one's own account or another user's account (when the identifier is `[file_id]`).
5. **`UploadToken`** (SOAPAction: `http://chomikuj.pl/IChomikBoxService/UploadToken`)
   - **Description:** Registers a request to send a new file. Returns a unique upload key (`key`), a timestamp (`stamp`), and the address of the physical server to which the binary data should be uploaded directly.
6. **`AddFolder`** (SOAPAction: `http://chomikuj.pl/IChomikBoxService/AddFolder`)
   - **Description:** Creates a new subfolder in the selected parent directory.
7. **`RemoveFolder`** (SOAPAction: `http://chomikuj.pl/IChomikBoxService/RemoveFolder`)
   - **Description:** Deletes a selected folder along with its contents from the user's account.

---

## 2. Login and Authorization (Auth)

Login consists of two steps: first, authorization is performed via the SOAP interface, which returns a session identifier (token), and then synchronization with the web session is performed (optional, required to generate file lists and certain permissions).

> [!NOTE]
> **Session Validity and Keep-Alive:**
> Re-calling the `Auth` method to refresh the session is not recommended because it generates a new session token, which invalidates any previous synchronization with the web session (log_www). Instead, to keep the session active in the background, you should periodically call the `CheckEvents` method, which acts as a "sliding expiration" mechanism without changing the token.

### Algorithm:
1. Retrieve the user's password in plain text.
2. Calculate the MD5 hash of the password and convert it to lowercase hexadecimal format.
3. Prepare the XML structure for the `Auth` method with the header `SOAPAction: http://chomikuj.pl/IChomikBoxService/Auth`.
4. Send a POST request to the service endpoint.
5. Read the response and check the `<status>` (or `<a:status>`) field. If it is `Ok`, extract the session token (`<a:token>`) and hamster ID (`<a:hamsterId>`).

### Structure of the SOAP Request (Auth):
```xml
<Auth xmlns="http://chomikuj.pl/">
  <name>[Username]</name>
  <passHash>[MD5_Password_Hash]</passHash>
  <ver>4</ver><!-- Client protocol version -->
  <client>
    <name>chomikbox</name>
    <version>2.0.8.2</version><!-- ChomikBox app version -->
  </client>
</Auth>
```

### Structure of the SOAP Response (AuthResponse):
```xml
<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/">
  <s:Body>
    <AuthResponse xmlns="http://chomikuj.pl/">
      <AuthResult xmlns:a="http://chomikuj.pl" xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
        <a:status>Ok</a:status>
        <a:errorMessage i:nil="true"/>
        <a:hamsterId>[Hamster_ID]</a:hamsterId>
        <a:publisherId i:nil="true"/>
        <a:name>[Username]</a:name>
        <a:token>[Session_Token_UUID]</a:token>
      </AuthResult>
    </AuthResponse>
  </s:Body>
</s:Envelope>
```

### Synchronization with the Web Session (log_www) - Required for listing files:
To list files in folders using the `Download` SOAP method, it is necessary to obtain a `chomikbox_url_id`. This is achieved by synchronizing the SOAP session with the browser session (HTTP Cookie Jar):

> [!IMPORTANT]
> **CheckEvents Dependency:**
> Before performing the synchronization with the web session (Step 1 below), you must call the SOAP `CheckEvents` method (see Section 7) at least once. This call registers the SOAP client as an active ChomikBox process. If you skip this step, the request to `/action/chomikbox/DownloadFolderChomikBox` (Step 4) will return an HTML modal informing that the ChomikBox application is not running: `"To download a folder in one click, you must have ChomikBox running"`, instead of the actual `chomik://` link.

1. Send a GET request to: `http://chomikuj.pl/chomik/chomikbox/LoginFromBox?t=[Session_Token]&returnUrl=/ChomikBox` and save the received cookies.
2. Extract the CSRF verification token from the response body: `<input name="__RequestVerificationToken" ... value="([Verification_Token])" />`.
3. Send a POST request to: `http://chomikuj.pl/action/Login/TopBarLogin` with the body encoded as `application/x-www-form-urlencoded`:
   `ReturnUrl=&Login=[Username]&rememberLogin=true&Password=[Password]&__RequestVerificationToken=[Verification_Token]`
4. Send a POST request to: `http://chomikuj.pl/action/chomikbox/DownloadFolderChomikBox` with the parameters:
   `chomikName=[Username]&folderId=0&__RequestVerificationToken=[Verification_Token]`
5. Extract the URL identifier from the response using a regular expression: `chomik://files/(?::)?(\d+)/?` (depending on the website version, the link might look like `chomik://files/:[id]/` or `chomik://files/[id]/0`). We refer to this extracted number as `chomikbox_url_id` (used later for file listing).

---

## 3. Listing Folders, Subfolders, and Files

### A. Listing Folder Structure (Folders)
Retrieving the full directory tree of a user is done using the `Folders` method.

#### Algorithm:
1. Prepare the query for the `Folders` method with the header `SOAPAction: http://chomikuj.pl/IChomikBoxService/Folders`.
2. Set the `folderId` parameter to `0` to retrieve the structure from the root directory.
3. Send the POST request.
4. The response contains a structure of nested `<FolderInfo>` elements. Each folder has:
   - `<id>`: numerical folder identifier.
   - `<name>`: directory name.
   - `<folders>`: list of subfolders (nested `<FolderInfo>` objects).

#### Structure of the SOAP Request (Folders):
```xml
<Folders xmlns="http://chomikuj.pl/">
  <token>[Session_Token]</token>
  <hamsterId>[Hamster_ID]</hamsterId>
  <folderId>0</folderId><!-- 0 indicates root directory -->
  <depth>0</depth><!-- Listing depth (0 indicates retrieving the entire structure) -->
</Folders>
```

---

### B. Listing Files in a Folder (Download)
Listing files is done using the `Download` method, with the `disposition` parameter set to `download`.

#### Algorithm:
1. Prepare the request for the `Download` method with the header `SOAPAction: http://chomikuj.pl/IChomikBoxService/Download`.
2. Generate the request identifier `id` in the format: `[chomikbox_url_id]/[folder_id]`, where `chomikbox_url_id` was obtained during the web session synchronization (log_www), and `folder_id` is the identifier of the folder of interest.
3. Set the `<disposition>` parameter to `download`.
4. Set the `<agreementInfo>` -> `<AgreementInfo>` -> `<name>` parameter to `own`.
5. Send the POST request.
6. Receive the XML response. In the `<files>` section, extract all `<FileEntry>` elements. Each file has fields:
   - `<name>`: file name.
   - `<url>`: direct download identifier/url.

> [!NOTE]
> **Empty Folder Behavior:**
> If the requested folder is empty (contains no files), the SOAP server will return an error response with the field `<a:status>Error</a:status>` and the message `<a:errorMessage>failed : requested file(s) not available</a:errorMessage>`. The client application should handle this scenario and treat it as a successful retrieval of an empty file list (e.g., returning an empty list) rather than a critical communication failure.

#### Structure of the SOAP Request (Download for file listing):
```xml
<Download xmlns="http://chomikuj.pl/">
  <token>[Session_Token]</token>
  <sequence>
    <stamp>0</stamp><!-- Request sequence/timestamp -->
    <part>0</part><!-- Request part number -->
    <count>1</count><!-- Number of items in the request list -->
  </sequence>
  <disposition>download</disposition>
  <list>
    <DownloadReqEntry>
      <id>[chomikbox_url_id]/[folder_id]</id>
      <agreementInfo>
        <AgreementInfo>
          <name>own</name>
        </AgreementInfo>
      </agreementInfo>
    </DownloadReqEntry>
  </list>
</Download>
```

---

### C. Creating a Folder (AddFolder)
Creating new directories (folders) on the user's account is done using the `AddFolder` method.

#### Algorithm:
1. Prepare the SOAP request for the `AddFolder` method (`SOAPAction: http://chomikuj.pl/IChomikBoxService/AddFolder`).
2. Provide the `token` (active session identifier).
3. Provide `newFolderId` as the parent directory identifier (e.g., `0` for the root directory).
4. Provide `name` as the name of the newly created directory.
5. Send the POST request.
6. Read the XML response:
   - If the operation was successful, the `<status>` element will return `Ok`, and the `<a:folderId>` element (namespace `http://schemas.datacontract.org/2004/07/TeamSolutions.Chomik.DAO.ChomikBox`) will contain the numerical identifier of the newly created folder.
   - If a folder with the same name already exists in the selected location, the `<status>` element will return `Error`, and you will receive the error message `NameExistsAtDestination` in the `<errorMessage>` field.

#### Structure of the SOAP Request (AddFolder):
```xml
<AddFolder xmlns="http://chomikuj.pl/">
  <token>[Session_Token]</token>
  <newFolderId>[Parent_Folder_ID]</newFolderId>
  <name>[New_Folder_Name]</name>
</AddFolder>
```

#### Structure of the SOAP Response (AddFolderResponse):
```xml
<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/">
  <s:Body>
    <AddFolderResponse xmlns="http://chomikuj.pl/">
      <AddFolderResult xmlns:a="http://schemas.datacontract.org/2004/07/TeamSolutions.Chomik.DAO.ChomikBox" xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
        <status xmlns="http://chomikuj.pl">Ok</status>
        <errorMessage i:nil="true" xmlns="http://chomikuj.pl"/>
        <a:folderId>[Created_Folder_ID]</a:folderId>
      </AddFolderResult>
    </AddFolderResponse>
  </s:Body>
</s:Envelope>
```

---

### D. Removing a Folder (RemoveFolder)
Deleting existing directories from the user's account is done using the `RemoveFolder` method.

#### Algorithm:
1. Prepare the SOAP request for the `RemoveFolder` method (`SOAPAction: http://chomikuj.pl/IChomikBoxService/RemoveFolder`).
2. Provide the `folderId` of the folder to be deleted.
3. Set the `force` parameter to `1` (to force the deletion of the folder along with all its contents).
4. Send the POST request.
5. Read the response and check the `<status>` (or `<a:status>`) field. A successful deletion returns a status of `Ok`.

#### Structure of the SOAP Request (RemoveFolder):
```xml
<RemoveFolder xmlns="http://chomikuj.pl/">
  <token>[Session_Token]</token>
  <folderId>[Folder_ID_To_Delete]</folderId>
  <force>1</force><!-- Force delete with contents (1 = yes, 0 = no) -->
</RemoveFolder>
```

---

### E. Deleting a File (DeleteFileAction - Web Action)
Since the `IChomikBoxService` SOAP contract **does not** contain a dedicated method for deleting individual files (e.g., the SOAP method `RemoveFile` does not exist), file deletion in the official ChomikBox application is performed via a standard HTTP POST request to a web endpoint.

#### Requirements:
- Valid web session authorization (cookies obtained using the `LoginFromBox` method described in Section 2).

#### Endpoint Address:
`http://chomikuj.pl/action/FileDetails/DeleteFileAction` (also supports HTTPS)

#### HTTP Method:
`POST`

#### HTTP Headers:
```http
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest
```

#### POST Parameters (Form Data):
- `ChomikName`: The username of the account (owner of the file).
- `FolderId`: The numerical identifier of the parent directory (e.g., `0` for the root directory).
- `FileId`: The numerical identifier of the file to be deleted.
- `FolderTo`: Always set to `0` (represents the trash bin / delete action).

#### Example Request Body Format:
`ChomikName=[Hamster_Name]&FolderId=0&FileId=1234567890&FolderTo=0`

#### JSON Response (Success):
A successful operation returns a JSON response (Status `200 OK`):
```json
{
  "Type": "Growl",
  "Title": "",
  "Content": "File has been deleted",
  "refreshTopBar": false,
  "IsSuccess": true,
  "Data": {
    "Status": "OK"
  },
  "ContainsCaptcha": false,
  "trackingCodeJS": null
}
```

#### Response (Error):
If the specified `FileId` does not exist in the specified location, the server returns an error:
`HTTP Error 500: Internal Server Error`

---

## 4. Generating a File Download URL

To download a file, you must generate a single-use download token/URL using the `Download` method with a different parameter configuration.

### Algorithm:
1. Prepare the SOAP request for the `Download` method (`SOAPAction: http://chomikuj.pl/IChomikBoxService/Download`).
2. Set the `stamp` parameter in the `sequence` section to a value such as `2` (or any integer indicating the sequence number).
3. Provide the `file_id` (obtained from file listing or directly from the website) as the `<id>` element.
4. If you are downloading a file from your own account, set `<name>` in `agreementInfo` to `own`. If the file is from another user's account, set the value to `access`.
5. Send the POST request.
6. Read the XML response. In the `<url>` field (or nested in the response structure), you will receive the direct HTTP link where the file is available for download.

### Structure of the SOAP Request (Generating URL):
```xml
<Download xmlns="http://chomikuj.pl/">
  <token>[Session_Token]</token>
  <sequence>
    <stamp>2</stamp><!-- Request sequence number -->
    <part>0</part><!-- Request part number -->
    <count>1</count><!-- Number of items in the request list -->
  </sequence>
  <disposition>download</disposition>
  <list>
    <DownloadReqEntry>
      <id>[File_ID]</id>
      <agreementInfo>
        <AgreementInfo>
          <name>own</name>
        </AgreementInfo>
      </agreementInfo>
    </DownloadReqEntry>
  </list>
</Download>
```
> [!NOTE]
> **Download Stream Stability:**
> The direct HTTP download link obtained from the `<url>` field is completely independent of the SOAP session state once the connection is established. The download server (`s*.chomikuj.pl`) maintains a stable and uninterrupted connection until the full data stream is downloaded.

---

## 5. Searching for a File

### SOAP Protocol Limitations:
A major drawback and specificity of the ChomikBox SOAP service is the **lack of a direct file search method (both globally and locally)**. Searching must be implemented at the client application level using one of the following two strategies:

### Method A: Local search (on user account)
Implemented by the client application by recursively searching the directory tree structure retrieved via SOAP.

#### Algorithm:
1. Retrieve the user's full directory tree by calling the `Folders` method (root folder `0`).
2. Recursively traverse (e.g., using DFS or BFS) all folders.
3. For each folder, retrieve the file list using the `Download` method (as in Section 3B).
4. Perform local matching of the target filename against the names of files in the lists.

### Method B: Web/HTTP Search (Scraping and search forms)
Since SOAP does not offer direct methods for searching files (neither globally nor locally), searching on the service is performed using HTTP GET/POST requests to traditional web endpoints. Analysis of the forms on a logged-in user profile (`chomikuj.pl/[username]`) reveals three distinct search mechanisms:

#### 1. Global account / hamster search (HTTP GET)
Used to search for user accounts (hamsters) on the service.
* **URL:** `http://chomikuj.pl/action/Search`
* **Method:** `GET`
* **Parameters (Query String):**
  - `Query`: Search phrase (e.g., `Ziemia`).
* **Note:** The address `Search.aspx` is deprecated and does not return search results (it redirects to the portal home page for both logged-in and guest users). The address `/action/Search/SearchFiles` also does not exist (404).

#### 2. Global file search by type (HTTP POST)
Enables searching the service for specific file formats.
* **URL:** `http://chomikuj.pl/action/SearchFiles`
* **Method:** `POST`
* **Parameters (Form Data / URL-encoded):**
  - `FileName`: Search phrase (e.g., `Ziemia`).
  - `FileType`: File type filter. Permissible values:
    * `all` (all)
    * `video` (video files)
    * `image` (images)
    * `music` (music files)
    * `document` (documents)
    * `archive` (archives)
    * `application` (programs)

#### 3. Local search on a selected account (HTTP POST)
Allows narrowing the file search to a single, specific user account.
* **URL:** `http://chomikuj.pl/action/SearchFiles`
* **Method:** `POST`
* **Parameters (Form Data / URL-encoded):**
  - `FileName`: Search phrase.
  - `SearchOnAccount`: Value `true` (forces searching on the specified account).
  - `TargetAccountName`: The name of the hamster account to search.

---

## 6. Uploading a File (Upload)

File upload in Chomikuj.pl is a two-step process: first, an upload authorization token is obtained for the given file (via SOAP), and then the file is streamed directly to a dedicated upload server (via an HTTP POST Multipart request).

### A. Retrieving an Upload Token (UploadToken)
Before sending a file, the client must ask the main server which physical server it should be uploaded to, and obtain an upload session key (`key`) and timestamp (`stamp`).

#### Algorithm:
1. Prepare the request for the `UploadToken` method (`SOAPAction: http://chomikuj.pl/IChomikBoxService/UploadToken`).
2. Provide `folderId` (where to save the file) and `fileName` (the name of the file being created).
3. Send the POST request.
4. Read the response and extract:
   - `key`: upload authorization token.
   - `stamp`: timestamp used as a boundary element.
   - `server`: upload server address (e.g., `upload15.chomikuj.pl:80`). Extract the host string and port from this.

#### Structure of the SOAP Request (UploadToken):
```xml
<UploadToken xmlns="http://chomikuj.pl/">
  <token>[Session_Token]</token>
  <folderId>[Target_Folder_ID]</folderId>
  <fileName>[File_Name]</fileName>
</UploadToken>
```

#### Structure of the SOAP Response (UploadTokenResponse):
```xml
<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/">
  <s:Body>
    <UploadTokenResponse xmlns="http://chomikuj.pl/">
      <UploadTokenResult xmlns:a="http://chomikuj.pl" xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
        <a:status>Ok</a:status>
        <a:errorMessage i:nil="true"/>
        <a:key>[Upload_Key]</a:key>
        <a:locale>PL</a:locale>
        <a:server>[Upload_Server_Host:Port]</a:server>
        <a:stamp>[Upload_Stamp]</a:stamp>
      </UploadTokenResult>
    </UploadTokenResponse>
  </s:Body>
</s:Envelope>
```

---

### B. Transferring File Data (HTTP POST Multipart/Mixed)
The actual file is sent directly to the server received in the `UploadToken` response (e.g., `upload15.chomikuj.pl:80`). This request is a specific combination of HTTP POST and the Multipart format.

#### Algorithm:
1. Establish a direct TCP connection to the specified server and port.
2. Define a unique boundary separator in the format: `--!CHB[stamp]`, where `[stamp]` is the marker received in Step A.
3. Construct the POST request header:
   ```http
   POST /file/ HTTP/1.0
   Content-Type: multipart/mixed; boundary=!CHB[stamp]
   Host: [upload_server]
   Content-Length: [total_multipart_body_length]
   ```
4. The request body consists of Multipart sections describing the session parameters and the binary file stream. The sections are:
   - `chomik_id` (User identifier)
   - `folder_id` (Target folder)
   - `key` (Upload key from Step A)
   - `time` (Stamp marker from Step A)
   - `client` (`ChomikBox-2.0.8.2`)
   - `locale` (`PL`)
   - `file` (Binary file section)
5. Send the header, then transmit the file in chunks (e.g., a 1024-byte buffer) while monitoring progress.
6. Finally, send the closing boundary sequence: `\r\n--!CHB[stamp]--\r\n\r\n`.
7. Receive the server response. A successful upload returns an XML structure containing: `<resp res="1" fileid="[new_file_id]"/>`.

#### Example Request Body Format (Multipart with binary data):
```text
--!CHB[stamp]
name="chomik_id"
Content-Type: text/plain

[Hamster_ID]
--!CHB[stamp]
name="folder_id"
Content-Type: text/plain

[Folder_ID]
--!CHB[stamp]
name="key"
Content-Type: text/plain

[Upload_Key]
--!CHB[stamp]
name="time"
Content-Type: text/plain

[Upload_Stamp]
--!CHB[stamp]
name="client"
Content-Type: text/plain

ChomikBox-2.0.8.2
--!CHB[stamp]
name="locale"
Content-Type: text/plain

PL
--!CHB[stamp]
name="file"; filename="[File_Name]"

[FILE_BINARY_DATA]
--!CHB[stamp]--
```

---

### C. Resuming an Interrupted Upload (Resume)
The Chomikuj.pl upload protocol allows resuming a file upload if the connection was interrupted in progress.

#### Algorithm:
1. Before sending again, send a GET request checking the state on the upload server:
   ```http
   GET /resume/check/?key=[Upload_Key]& HTTP/1.1
   Host: [upload_server]
   User-Agent: ChomikBox
   ```
2. Receive the server response. If the file was already partially sent, the server will return XML with the size of the received data:
   ```xml
   <!-- skipThumbnails: 0 = do not skip, res: 1 = success status -->
   <resp file_size="[size_in_bytes_on_server]" skipThumbnails="0" res="1"/>
   ```
3. If the server returns a size greater than zero (e.g., `filesize_sent`):
   - Open the local file and move the reading pointer (seek) to the `filesize_sent` position.
   - Prepare an HTTP POST multipart/mixed request to `/file/` identical to the original, but **add a new section** in the Multipart before the file section:
     ```text
     --!CHB[stamp]
     name="resume_from"
     Content-Type: text/plain

     [filesize_sent]
     ```
   - Send the remaining part of the file as the file size in the header and multipart: `(total_size - filesize_sent)`.
   - Stream the remaining bytes of the file, send the closing boundary, and receive the final response.

---

## 7. Monitoring Application State (CheckEvents)

The `CheckEvents` method serves as the official SOAP session keep-alive mechanism. It allows refreshing the session on the server without needing to re-authenticate (via the `Auth` method), preventing the token from expiring and preserving the validity of the web session synchronization.
The ChomikBox client periodically queries the main server to verify session validity and report statistical information on current transfers using the `CheckEvents` method.

### Algorithm:
1. Prepare the query for the `CheckEvents` method (`SOAPAction: http://chomikuj.pl/IChomikBoxService/CheckEvents`).
2. Pass the statuses in the `stats` parameters:
   - `isUploading` (e.g., `0` or `1` if uploading is in progress).
   - `isDownloading` (e.g., `0` or `1` if downloading is in progress).
   - `panelSelectedTab` (identifier of the open panel tab).
   - `animation` (interface animation parameter).
3. Send the POST request.
4. The server response in the `<status>` (or `<status><#text>`) field should return `Ok`.

### Structure of the SOAP Request (CheckEvents):
```xml
<CheckEvents xmlns="http://chomikuj.pl/">
  <token>[Session_Token]</token>
  <stats>
    <isUploading>0</isUploading><!-- Upload status: 0 = none, 1 = in progress -->
    <isDownloading>0</isDownloading><!-- Download status: 0 = none, 1 = in progress -->
    <panelSelectedTab>0</panelSelectedTab><!-- Index of the open tab in ChomikBox (default 0) -->
    <animation>2</animation><!-- Hamster animation type in the app (default 2) -->
  </stats>
</CheckEvents>
```
