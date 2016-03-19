--- 
layout: post
title: "Using Office 365 Migration Api using C#"
author: "Kees Schollaart" 
backgroundUrl: /img/migration.jpg
comments: true 
---  

This blogpost describes how to use the Office 365 Migration Api using custom code. 
The Office365 Migration Api enables you to upload lots of documents into SharePoint Online withoud being throttled.

## What is the Migration Api?

This blogpost assumes that you know what the Migration Api is and when to use it. 

Steven Pogrebivsky has written an [blogpost](http://www.cmswire.com/cms/information-management/demystifying-the-new-migration-api-for-sharepoint-029245.php) on this subject which I recomment to read first.

Also Benjamin Niaulin of Share Gate has written an [blogpost](http://en.share-gate.com/blog/how-to-use-office-365-migration-api), off course promoting their software which can help you with your migration.

## The Microsoft PowerShell Scripts
Microsoft provides some PowerShell cmdlets to help you migrate data using the Migration Api. Using the [SharePoint Online Management Shell Windows PowerShell environment](https://technet.microsoft.com/library/fp161372.aspx) you can use [a set of Migration Api related operations](https://technet.microsoft.com/library/mt203955.aspx). 
  
What this cmdlets basically do, is:

- They create a manifest package of your files 

- Upload your files to Azure Storage

- Use [Site.CreateMigrationJob()](https://msdn.microsoft.com/EN-US/library/office/microsoft.sharepoint.client.site.createmigrationjob.aspx) to start the MigrationJon

- Read the reporting-queue to provide current job-status


This PowerShell scripts are easy to use but do not bring a lot of flexibility. For example:

- Impossible to migrate custom metadata

- Can not (or hardly) be automated 

- Only on source or destination (folder)

- Reporting on running job is very simple

## Undocumented features
Some technical parts of this migration process are documented, like [Site.CreateMigrationJob()](https://msdn.microsoft.com/EN-US/library/office/microsoft.sharepoint.client.site.createmigrationjob.aspx).

Other parts are not (well) documented, for example the structure of the manifest-package and the reporting-queue messageformat.

There's also no example project on how to do advanced migration scenario's and how to use the Api's yourself.

## Migration using C#
All the things the PowerShell scripts do, we can do ourselfs using .NET. I created a [Proof Of Concept C# Console Application](https://github.com/keesschollaart81/MigrationApiDemo) doing:

1 Create and upload some test-files to Azure Blob Storage

2 Create a Manifest Package based on this test-files

3 Upload this Manifest Package to Azure Blob Storage

4 Start the Migration Job using CSOM

5 Monitor the Reporting Queue and wait for the job to complete

6 Persist errors from the queue and download the log-files from the Migration Job

## How to use this code
You first need:

- A SharePoint Online tenant with credentials having write access to the destination Site/Document Library/Folder 

- An existing Site with an existing Document Library to migrate the files to

- An acountname and accountkey of at least one Azure Storage account, for one migration job, we need two Blob Containers and one Queue. They can be in the same Storage Account but that is not required.

After setting this variables in the appsettings-section of the ```App.config``` file you're good to go, just run the application. 
After running this application succesfully two test-files will be uploaded in the configured SharePoint environment. Log files will be stored in the application folder using log4net's FileAppender.

This application is just to show how to migrate files using .NET with a minimal set of code. **It cannot be used as-is in real world scenario's**! 

When you want to use this code, the first thing to change will be the origin of the files from in-memory-generation to (for example) a folder.

## More information
Feel free to use this code for own/commercial use. For questions or information, contact me! 

- [Migration Api using C#, GitHub](https://github.com/keesschollaart81/MigrationApiDemo)

- [SharePoint Online and OneDrive Migration Content Roadmap](https://technet.microsoft.com/library/mt203955.aspx)

- [Channel9: Migration to SharePoint Online Best Practices and New API Investments](https://channel9.msdn.com/Events/Ignite/2015/BRK3153)