# chomikuj.pl / ChomikBox SOAP API - AI Agent skill

This skill contains a comprehensive specification and reference library for communicating with Chomikuj.pl. The documentation outlines the service's unofficial SOAP API (modeled after the official ChomikBox desktop application protocol) as well as HTTP/web-scraping methods used when SOAP endpoints are unavailable (such as deleting files or performing web-based searches).

## SKILL Language Versions
[EN](blob/main/SKILL/EN/SKILL.md) | [PL](blob/main/SKILL/PL/SKILL.md)

## Table of Contents
```
1. General Information and SOAP Protocol
   - List of All API Methods (SOAP Actions)
2. Login and Authorization (Auth)
3. Listing Folders, Subfolders, and Files
   - Listing Folder Structure (Folders)
   - Listing Files in a Folder (Download)
   - Creating a Folder (AddFolder)
   - Removing a Folder (RemoveFolder)
   - Deleting a File (DeleteFileAction - Web Action)
4. Generating a File Download URL
5. Searching for a File
   - Method A: Local search (on user account)
   - Method B: Web/HTTP Search (Scraping and search forms)
6. Uploading a File (Upload)
   - Retrieving an Upload Token (UploadToken)
   - Transferring File Data (HTTP POST Multipart/Mixed)
   - Resuming an Interrupted Upload (Resume)
7. Monitoring Application State (CheckEvents)
```
