---
title: Ontwikkelen voor Azure File Storage met .NET | Microsoft Docs
description: Informatie over het ontwikkelen van .NET-toepassingen en -services die gebruikmaken van Azure File Storage voor het opslaan van gegevens uit een bestand.
services: storage
documentationcenter: .net
author: RenaShahMSFT
manager: aungoo
editor: tysonn
ms.assetid: 6a889ee1-1e60-46ec-a592-ae854f9fb8b6
ms.service: storage
ms.workload: storage
ms.tgt_pltfrm: na
ms.devlang: dotnet
ms.topic: hero-article
ms.date: 05/27/2017
ms.author: renash
ms.translationtype: HT
ms.sourcegitcommit: 83f19cfdff37ce4bb03eae4d8d69ba3cbcdc42f3
ms.openlocfilehash: 7b94e70619324bb8dc8e7f8306f00f06e7476c1f
ms.contentlocale: nl-nl
ms.lasthandoff: 08/21/2017

---

# <a name="develop-for-azure-file-storage-with-net"></a>Ontwikkelen voor Azure File Storage met .NET 
> [!NOTE]
> In dit artikel leest u hoe u Azure File Storage beheert met .NET-code. Raadpleeg voor meer informatie over Azure File Storage, de [Inleiding tot Azure File Storage](storage-files-introduction.md).
>

[!INCLUDE [storage-selector-file-include](../../../includes/storage-selector-file-include.md)]

[!INCLUDE [storage-check-out-samples-dotnet](../../../includes/storage-check-out-samples-dotnet.md)]

## <a name="about-this-tutorial"></a>Over deze zelfstudie
In deze zelfstudie worden de basisbeginselen uitgelegd van het gebruik van .NET voor het ontwikkelen van toepassingen of services die Azure File Storage gebruiken voor het opslaan van gegevens uit een bestand. In deze zelfstudie maakt u een eenvoudige consoletoepassing en laten we zien hoe u basisbewerkingen uitvoert met .NET en Azure File Storage:

* De inhoud van een bestand ophalen
* Een quotum (maximumgrootte) voor de bestandsshare instellen.
* Een Shared Access Signature (SAS-sleutel) maken voor een bestand dat gebruikmaakt van een gedeeld toegangsbeleid dat voor de share is gedefinieerd.
* Een bestand kopiëren naar een ander bestand in hetzelfde opslagaccount.
* Een bestand kopiëren naar een blob in hetzelfde opslagaccount.
* Metrische gegevens van Azure Storage gebruiken voor het oplossen van problemen

> [!Note]  
> Omdat Azure File Storage toegankelijk is via SMB, is het mogelijk om eenvoudige toepassingen te schrijven die toegang hebben tot de Azure-bestandsshare met behulp van de standaard System.IO-klassen voor i/o-bestanden. In dit artikel wordt beschreven hoe u toepassingen schrijft die gebruikmaken van de Azure Storage .NET SDK, die gebruikmaakt van de [Azure File storage REST-API](https://docs.microsoft.com/rest/api/storageservices/fileservices/file-service-rest-api) voor communicatie met Azure File Storage. 


## <a name="create-the-console-application-and-obtain-the-assembly"></a>De consoletoepassing maken en de assembly verkrijgen
Maak in Visual Studio een nieuwe Windows-consoletoepassing. De volgende stappen laten zien hoe u een consoletoepassing maakt in Visual Studio 2017, maar de stappen zijn hetzelfde in andere versies van Visual Studio.

1. Selecteer **Bestand** > **Nieuw** > **Project**
2. Selecteer **Geïnstalleerd** > **Sjablonen** > **Visual C#** > **Klassiek Windows-bureaublad**
3. Selecteer **Consoletoepassing (.NET Framework)**
4. Typ een naam voor de toepassing in het veld **Naam:**
5. Selecteer **OK**

Alle codevoorbeelden in deze zelfstudie kunnen worden toegevoegd aan de methode `Main()` van het bestand `Program.cs` van de consoletoepassing.

U kunt de Azure Storage-clientbibliotheek gebruiken in elk type .NET-toepassing, waaronder een Azure-cloudservice of -web-app en bureaublad- en mobiele toepassingen. In deze gids gebruiken we een consoletoepassing voor de eenvoud.

## <a name="use-nuget-to-install-the-required-packages"></a>NuGet gebruiken om de vereiste pakketten te installeren
Er zijn twee pakketten waarnaar u in uw project moet verwijzen om deze zelfstudie te kunnen voltooien:

* [Microsoft Azure Storage-clientbibliotheek voor .NET](https://www.nuget.org/packages/WindowsAzure.Storage/): dit pakket biedt programmatisch toegang tot gegevensbronnen in uw opslagaccount.
* [Configuration Manager-bibliotheek van Microsoft Azure voor .NET](https://www.nuget.org/packages/Microsoft.WindowsAzure.ConfigurationManager/): dit pakket biedt een klasse voor het parseren van een verbindingsreeks in een configuratiebestand, ongeacht waar de toepassing wordt uitgevoerd.

Met NuGet kunt u beide pakketten verkrijgen. Volg deze stappen:

1. Klik met de rechtermuisknop op het project in **Solution Explorer** en kies **NuGet-pakketten beheren**.
2. Zoek online naar 'WindowsAzure.Storage' en klik op **Installeren** om de Storage-clientbibliotheek en de afhankelijkheden ervan te installeren.
3. Zoek online naar 'WindowsAzure.ConfigurationManager' en klik op **Installeren** om Azure Configuration Manager te installeren.

## <a name="save-your-storage-account-credentials-to-the-appconfig-file"></a>Uw opslagaccountreferenties opslaan in het bestand app.config
Vervolgens slaat u uw referenties op in het bestand app.config van het project. Bewerk het bestand app.config zo dat het op het volgende voorbeeld lijkt, waarbij u `myaccount` vervangt door de naam van het opslagaccount, en `mykey` door de sleutel van het opslagaccount.

```xml
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
    <startup>
        <supportedRuntime version="v4.0" sku=".NETFramework,Version=v4.5" />
    </startup>
    <appSettings>
        <add key="StorageConnectionString" value="DefaultEndpointsProtocol=https;AccountName=myaccount;AccountKey=StorageAccountKeyEndingIn==" />
    </appSettings>
</configuration>
```

> [!NOTE]
> De nieuwste versie van de Azure-opslagemulator biedt geen ondersteuning voor Azure File Storage. De verbindingsreeks werkt alleen met Azure File Storage als de reeks verwijst naar een Azure-opslagaccount in de cloud.

## <a name="add-using-directives"></a>Using-instructies toevoegen
Open het bestand `Program.cs` in Solution Explorer en plaats de volgende using-instructies aan het begin van het bestand.

```csharp
using Microsoft.Azure; // Namespace for Azure Configuration Manager
using Microsoft.WindowsAzure.Storage; // Namespace for Storage Client Library
using Microsoft.WindowsAzure.Storage.Blob; // Namespace for Azure Blobs
using Microsoft.WindowsAzure.Storage.File; // Namespace for Azure File storage
```

[!INCLUDE [storage-cloud-configuration-manager-include](../../../includes/storage-cloud-configuration-manager-include.md)]

## <a name="access-the-file-share-programmatically"></a>Via een programma toegang krijgen tot de bestandsshare
Vervolgens voegt u de volgende code aan de methode `Main()` (na de hierboven weergegeven code) toe om de verbindingsreeks op te halen. Met deze code wordt een verwijzing opgehaald naar het bestand dat we eerder hebben gemaakt en wordt de inhoud ervan weergegeven in het consolevenster.

```csharp
// Create a CloudFileClient object for credentialed access to Azure File storage.
CloudFileClient fileClient = storageAccount.CreateCloudFileClient();

// Get a reference to the file share we created previously.
CloudFileShare share = fileClient.GetShareReference("logs");

// Ensure that the share exists.
if (share.Exists())
{
    // Get a reference to the root directory for the share.
    CloudFileDirectory rootDir = share.GetRootDirectoryReference();

    // Get a reference to the directory we created previously.
    CloudFileDirectory sampleDir = rootDir.GetDirectoryReference("CustomLogs");

    // Ensure that the directory exists.
    if (sampleDir.Exists())
    {
        // Get a reference to the file we created previously.
        CloudFile file = sampleDir.GetFileReference("Log1.txt");

        // Ensure that the file exists.
        if (file.Exists())
        {
            // Write the contents of the file to the console window.
            Console.WriteLine(file.DownloadTextAsync().Result);
        }
    }
}
```

Voer de consoletoepassing uit om de uitvoer te zien.

## <a name="set-the-maximum-size-for-a-file-share"></a>De maximale grootte voor een bestandsshare instellen
Vanaf versie 5.x van de Azure Storage-clientbibliotheek kunt u het quotum (de maximumgrootte) voor een bestandsshare instellen, in gigabytes. U kunt ook controleren hoeveel gegevens er momenteel zijn opgeslagen in de share.

Door het quotum voor een share in te stellen, kunt u de totale grootte instellen van de bestanden die zijn opgeslagen in de share. Als de totale grootte van de bestanden in de share het quotum voor de share overschrijdt, kunnen clients bestaande bestanden niet meer vergroten en geen nieuwe bestanden meer maken, tenzij deze bestanden leeg zijn.

In onderstaand voorbeeld ziet u hoe u het huidige gebruik voor een share controleert en een quotum voor de share instelt.

```csharp
// Parse the connection string for the storage account.
CloudStorageAccount storageAccount = CloudStorageAccount.Parse(
    Microsoft.Azure.CloudConfigurationManager.GetSetting("StorageConnectionString"));

// Create a CloudFileClient object for credentialed access to Azure File storage.
CloudFileClient fileClient = storageAccount.CreateCloudFileClient();

// Get a reference to the file share we created previously.
CloudFileShare share = fileClient.GetShareReference("logs");

// Ensure that the share exists.
if (share.Exists())
{
    // Check current usage stats for the share.
    // Note that the ShareStats object is part of the protocol layer for the File service.
    Microsoft.WindowsAzure.Storage.File.Protocol.ShareStats stats = share.GetStats();
    Console.WriteLine("Current share usage: {0} GB", stats.Usage.ToString());

    // Specify the maximum size of the share, in GB.
    // This line sets the quota to be 10 GB greater than the current usage of the share.
    share.Properties.Quota = 10 + stats.Usage;
    share.SetProperties();

    // Now check the quota for the share. Call FetchAttributes() to populate the share's properties.
    share.FetchAttributes();
    Console.WriteLine("Current share quota: {0} GB", share.Properties.Quota);
}
```

### <a name="generate-a-shared-access-signature-for-a-file-or-file-share"></a>Een Shared Access Signature genereren voor een bestand of bestandsshare
Vanaf versie 5.x van de Azure Storage-clientbibliotheek kunt u een Shared Access Signature (SAS) genereren voor een bestandsshare of voor een afzonderlijk bestand. U kunt op een bestandsshare ook een beleid voor gedeelde toegang maken om handtekeningen voor gedeelde toegang te beheren. Het maken van een beleid voor gedeelde toegang wordt aanbevolen omdat dit een manier biedt om de SAS in te trekken als deze verdacht is.

In het volgende voorbeeld wordt een beleid voor gedeelde toegang gemaakt op een share en wordt dat beleid vervolgens gebruikt om de beperkingen te verstrekken voor een SAS voor een bestand in de share.

```csharp
// Parse the connection string for the storage account.
CloudStorageAccount storageAccount = CloudStorageAccount.Parse(
    Microsoft.Azure.CloudConfigurationManager.GetSetting("StorageConnectionString"));

// Create a CloudFileClient object for credentialed access to Azure File storage.
CloudFileClient fileClient = storageAccount.CreateCloudFileClient();

// Get a reference to the file share we created previously.
CloudFileShare share = fileClient.GetShareReference("logs");

// Ensure that the share exists.
if (share.Exists())
{
    string policyName = "sampleSharePolicy" + DateTime.UtcNow.Ticks;

    // Create a new shared access policy and define its constraints.
    SharedAccessFilePolicy sharedPolicy = new SharedAccessFilePolicy()
        {
            SharedAccessExpiryTime = DateTime.UtcNow.AddHours(24),
            Permissions = SharedAccessFilePermissions.Read | SharedAccessFilePermissions.Write
        };

    // Get existing permissions for the share.
    FileSharePermissions permissions = share.GetPermissions();

    // Add the shared access policy to the share's policies. Note that each policy must have a unique name.
    permissions.SharedAccessPolicies.Add(policyName, sharedPolicy);
    share.SetPermissions(permissions);

    // Generate a SAS for a file in the share and associate this access policy with it.
    CloudFileDirectory rootDir = share.GetRootDirectoryReference();
    CloudFileDirectory sampleDir = rootDir.GetDirectoryReference("CustomLogs");
    CloudFile file = sampleDir.GetFileReference("Log1.txt");
    string sasToken = file.GetSharedAccessSignature(null, policyName);
    Uri fileSasUri = new Uri(file.StorageUri.PrimaryUri.ToString() + sasToken);

    // Create a new CloudFile object from the SAS, and write some text to the file.
    CloudFile fileSas = new CloudFile(fileSasUri);
    fileSas.UploadText("This write operation is authenticated via SAS.");
    Console.WriteLine(fileSas.DownloadText());
}
```

Voor meer informatie over het maken en gebruiken van handtekeningen voor gedeelde toegang, raadpleegt u [Using Shared Access Signatures (SAS)](../common/storage-dotnet-shared-access-signature-part-1.md?toc=%2fazure%2fstorage%2ffiles%2ftoc.json) (Shared Access Signatures (SAS) gebruiken) en [Create and use a SAS with Azure Blobs](../blobs/storage-dotnet-shared-access-signature-part-2.md) (Een SAS maken en gebruiken met Azure Blobs).

## <a name="copy-files"></a>Bestanden kopiëren
Vanaf versie 5.x van de Azure Storage-clientbibliotheek kunt u een bestand kopiëren naar een ander bestand, een bestand naar een blob of een blob naar een bestand. In de volgende secties ziet u hoe u deze kopieerbewerkingen uitvoert via een programma.

U kunt ook AzCopy gebruiken om een bestand naar een ander bestand te kopiëren of om een blob naar een bestand te kopiëren of omgekeerd. Zie [Transfer data with the AzCopy Command-Line Utility](../common/storage-use-azcopy.md?toc=%2fazure%2fstorage%2ffiles%2ftoc.json) (Gegevensoverdracht met het AzCopy-opdrachtregelprogramma).

> [!NOTE]
> Als u een blob kopieert naar een bestand of een bestand naar een blob, moet u een Shared Access Signature (SAS) gebruiken om het bronobject te verifiëren, zelfs als u binnen hetzelfde opslagaccount kopieert.
> 
> 

**Een bestand kopiëren naar een ander bestand** In het volgende voorbeeld wordt een bestand gekopieerd naar een ander bestand in dezelfde share. Omdat met deze kopieerbewerking bestanden in hetzelfde opslagaccount worden gekopieerd, kunt u verificatie met gedeelde sleutel gebruiken om de kopie te maken.

```csharp
// Parse the connection string for the storage account.
CloudStorageAccount storageAccount = CloudStorageAccount.Parse(
    Microsoft.Azure.CloudConfigurationManager.GetSetting("StorageConnectionString"));

// Create a CloudFileClient object for credentialed access to Azure File storage.
CloudFileClient fileClient = storageAccount.CreateCloudFileClient();

// Get a reference to the file share we created previously.
CloudFileShare share = fileClient.GetShareReference("logs");

// Ensure that the share exists.
if (share.Exists())
{
    // Get a reference to the root directory for the share.
    CloudFileDirectory rootDir = share.GetRootDirectoryReference();

    // Get a reference to the directory we created previously.
    CloudFileDirectory sampleDir = rootDir.GetDirectoryReference("CustomLogs");

    // Ensure that the directory exists.
    if (sampleDir.Exists())
    {
        // Get a reference to the file we created previously.
        CloudFile sourceFile = sampleDir.GetFileReference("Log1.txt");

        // Ensure that the source file exists.
        if (sourceFile.Exists())
        {
            // Get a reference to the destination file.
            CloudFile destFile = sampleDir.GetFileReference("Log1Copy.txt");

            // Start the copy operation.
            destFile.StartCopy(sourceFile);

            // Write the contents of the destination file to the console window.
            Console.WriteLine(destFile.DownloadText());
        }
    }
}
```

**Een bestand kopiëren naar een blob** In het volgende voorbeeld wordt een bestand gemaakt en vervolgens gekopieerd naar een blob in hetzelfde opslagaccount. In het voorbeeld wordt een SAS voor het bronbestand gemaakt, die tijdens de kopieerbewerking door de service wordt gebruikt om toegang tot het bronbestand te verifiëren.

```csharp
// Parse the connection string for the storage account.
CloudStorageAccount storageAccount = CloudStorageAccount.Parse(
    Microsoft.Azure.CloudConfigurationManager.GetSetting("StorageConnectionString"));

// Create a CloudFileClient object for credentialed access to Azure File storage.
CloudFileClient fileClient = storageAccount.CreateCloudFileClient();

// Create a new file share, if it does not already exist.
CloudFileShare share = fileClient.GetShareReference("sample-share");
share.CreateIfNotExists();

// Create a new file in the root directory.
CloudFile sourceFile = share.GetRootDirectoryReference().GetFileReference("sample-file.txt");
sourceFile.UploadText("A sample file in the root directory.");

// Get a reference to the blob to which the file will be copied.
CloudBlobClient blobClient = storageAccount.CreateCloudBlobClient();
CloudBlobContainer container = blobClient.GetContainerReference("sample-container");
container.CreateIfNotExists();
CloudBlockBlob destBlob = container.GetBlockBlobReference("sample-blob.txt");

// Create a SAS for the file that's valid for 24 hours.
// Note that when you are copying a file to a blob, or a blob to a file, you must use a SAS
// to authenticate access to the source object, even if you are copying within the same
// storage account.
string fileSas = sourceFile.GetSharedAccessSignature(new SharedAccessFilePolicy()
{
    // Only read permissions are required for the source file.
    Permissions = SharedAccessFilePermissions.Read,
    SharedAccessExpiryTime = DateTime.UtcNow.AddHours(24)
});

// Construct the URI to the source file, including the SAS token.
Uri fileSasUri = new Uri(sourceFile.StorageUri.PrimaryUri.ToString() + fileSas);

// Copy the file to the blob.
destBlob.StartCopy(fileSasUri);

// Write the contents of the file to the console window.
Console.WriteLine("Source file contents: {0}", sourceFile.DownloadText());
Console.WriteLine("Destination blob contents: {0}", destBlob.DownloadText());
```

Op dezelfde manier kunt u een blob naar een bestand kopiëren. Als het bronobject een blob is, maakt u een SAS om tijdens de kopieerbewerking de toegang tot de blob te verifiëren.

## <a name="troubleshooting-azure-file-storage-using-metrics"></a>Problemen met Azure File Storage oplossen met metrische gegevens
Azure Storage Analytics ondersteunt nu metrische gegevens voor Azure File Storage. Met metrische gegevens kunt u aanvragen volgen en problemen diagnosticeren.


U kunt metrische gegevens voor Azure File Storage inschakelen vanuit de [Azure Portal](https://portal.azure.com). U kunt metrische gegevens ook inschakelen via een programma. Daarvoor roept u de bewerking Set File Service Properties aan via de REST API of een van de analogen daarvan in de Storage-clientbibliotheek.


In het volgende codevoorbeeld ziet u hoe u de Storage-clientbibliotheek voor .NET gebruikt om metrische gegevens in te schakelen voor Azure File Storage.

Voeg behalve de instructies die u hierboven hebt toegevoegd, ook de volgende `using`-instructies toe aan het bestand `Program.cs`:

```csharp
using Microsoft.WindowsAzure.Storage.File.Protocol;
using Microsoft.WindowsAzure.Storage.Shared.Protocol;
```

Azure Blobs, Azure Table en Azure Queues gebruiken het gedeelde type `ServiceProperties` in de `Microsoft.WindowsAzure.Storage.Shared.Protocol`-naamruimte. Azure File Storage heeft echter een eigen type, het type `FileServiceProperties`, in de `Microsoft.WindowsAzure.Storage.File.Protocol`-naamruimte. De code kan echter alleen worden gecompileerd als er vanuit uw code naar beide naamruimten wordt verwezen.

```csharp
// Parse your storage connection string from your application's configuration file.
CloudStorageAccount storageAccount = CloudStorageAccount.Parse(
        Microsoft.Azure.CloudConfigurationManager.GetSetting("StorageConnectionString"));
// Create the File service client.
CloudFileClient fileClient = storageAccount.CreateCloudFileClient();

// Set metrics properties for File service.
// Note that the File service currently uses its own service properties type,
// available in the Microsoft.WindowsAzure.Storage.File.Protocol namespace.
fileClient.SetServiceProperties(new FileServiceProperties()
{
    // Set hour metrics
    HourMetrics = new MetricsProperties()
    {
        MetricsLevel = MetricsLevel.ServiceAndApi,
        RetentionDays = 14,
        Version = "1.0"
    },
    // Set minute metrics
    MinuteMetrics = new MetricsProperties()
    {
        MetricsLevel = MetricsLevel.ServiceAndApi,
        RetentionDays = 7,
        Version = "1.0"
    }
});

// Read the metrics properties we just set.
FileServiceProperties serviceProperties = fileClient.GetServiceProperties();
Console.WriteLine("Hour metrics:");
Console.WriteLine(serviceProperties.HourMetrics.MetricsLevel);
Console.WriteLine(serviceProperties.HourMetrics.RetentionDays);
Console.WriteLine(serviceProperties.HourMetrics.Version);
Console.WriteLine();
Console.WriteLine("Minute metrics:");
Console.WriteLine(serviceProperties.MinuteMetrics.MetricsLevel);
Console.WriteLine(serviceProperties.MinuteMetrics.RetentionDays);
Console.WriteLine(serviceProperties.MinuteMetrics.Version);
```

U kunt ook het [artikel voor het oplossen van problemen met Azure File Storage](storage-troubleshoot-windows-file-connection-problems.md) raadplegen voor richtlijnen voor end-to-end-probleemoplossing.

## <a name="next-steps"></a>Volgende stappen
Raadpleeg de volgende koppelingen voor meer informatie over Azure File Storage.

### <a name="conceptual-articles-and-videos"></a>Conceptuele artikelen en video's
* [Azure File Storage: een naadloos SMB-bestandssysteem voor Windows en Linux](https://azure.microsoft.com/documentation/videos/azurecon-2015-azure-files-storage-a-frictionless-cloud-smb-file-system-for-windows-and-linux/)
* [Azure File Storage gebruiken met Linux](storage-how-to-use-files-linux.md)

### <a name="tooling-support-for-file-storage"></a>Hulpprogramma-ondersteuning voor File Storage
* [AzCopy gebruiken met Microsoft Azure Storage](../common/storage-use-azcopy.md?toc=%2fazure%2fstorage%2ffiles%2ftoc.json)
* [De Azure CLI gebruiken met Azure Storage](../common/storage-azure-cli.md?toc=%2fazure%2fstorage%2ffiles%2ftoc.json#create-and-manage-file-shares)
* [Problemen met betrekking tot Azure File Storage oplossen](https://docs.microsoft.com/azure/storage/storage-troubleshoot-file-connection-problems)

### <a name="reference"></a>Naslaginformatie
* [Naslaginformatie over de Storage-clientbibliotheek voor .NET](https://msdn.microsoft.com/library/azure/dn261237.aspx)
* [Naslaginformatie over REST API voor bestandsservices](http://msdn.microsoft.com/library/azure/dn167006.aspx)

### <a name="blog-posts"></a>Blogberichten
* [Azure File storage is now generally available (Azure File Storage is nu algemeen beschikbaar)](https://azure.microsoft.com/blog/azure-file-storage-now-generally-available/)
* [Inside Azure File Storage (Een kijkje achter de schermen van Azure File Storage)](https://azure.microsoft.com/blog/inside-azure-file-storage/)
* [Introducing Microsoft Azure File Service (Introductie van Microsoft Azure File-service)](http://blogs.msdn.com/b/windowsazurestorage/archive/2014/05/12/introducing-microsoft-azure-file-service.aspx)
* [Persisting connections to Microsoft Azure File Storage (Permanente verbindingen met Microsoft Azure File Storage)](http://blogs.msdn.com/b/windowsazurestorage/archive/2014/05/27/persisting-connections-to-microsoft-azure-files.aspx)
