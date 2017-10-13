- [Frends.Cobalt](#frendscobalt)
    - [Installing](#installing)
    - [Building](#building)
    - [Contributing](#contributing)
    - [Documentation](#documentation)
        - [Technical overview](#technical-overview)
            - [Transfer overview](#transfer-overview)
            - [Error handling](#error-handling)
            - [Service Bus logging](#service-bus-logging)
        - [Transfer parameter reference](#transfer-parameter-reference)
            - [Cobalt task transfer parameters](#cobalt-task-transfer-parameters)
        - [Source](#source)
        - [Destination](#destination)
        - [Message processing steps](#message-processing-steps)
        - [Parameters](#parameters)
            - [Macro reference](#macro-reference)
        - [Transfer result](#transfer-result)
        - [Known issues](#known-issues)
            - [Source file operation may overwrite existing files on FTP](#source-file-operation-may-overwrite-existing-files-on-ftp)
            - [Failure in source file operation logs the transfer failed even if the files were transferred](#failure-in-source-file-operation-logs-the-transfer-failed-even-if-the-files-were-transferred)


# Frends.Cobalt
FRENDS Cobalt is a Task library built for transferring files between file, FTP, FTPS and SFTP endpoints with the possibility of processing the files during the transfer.

With FRENDS Cobalt you can easily automate recurring file transfers within the enterprise. The transfers are configured as parts of a FRENDS4 Process in the FRENDS4 management site. The same site can be used to run and monitor the transfers as well.

FRENDS Cobalt can read files from and deliver files to file system, FTP, FTPS and SFTP servers. Text file encoding can be changed and XML files can be converted using XSLT in message processing steps. Files can naturally be transferred without conversions as well.

Possible errors in FRENDS Cobalt executions are reported to the Windows EventLog on the Agent and in FRENDS4 management site as well, so operators can take action as required. In addition, FRENDS Cobalt uses Log4Net for logging purposes that can be configured to e.g. send reports as needed.

## Installing
You can install the task via FRENDS UI Task view or you can find the nuget package from the following nuget feed
`https://www.myget.org/F/frends/api/v2`


## Building
Source code is not yet available, but we hope to add it soon

## Contributing
Contributing is not possible at this time

## Documentation
### Technical overview
FRENDS Cobalt is implemented as a .NET 4.5.2 class that is run in a FRENDS4 process.
- [Transfer overview](#transfer-overview)
- [Error handling](#error-handling)
- [Service Bus logging](#service-bus-logging)
#### Transfer overview
The file transfer progress has the following steps:

Initialize

Initialize the transfer based on the configuration and open source connection(s).

    Note: If you have set the MaxConcurrentConnections to more than 1, the given number of source connections are opened.

- ListFiles  
Get a list of files from the source endpoint according to the filename/mask. If there are no files to transfer, the source connections are closed and the transfer ends with the ActionSkipped state
- Transfer files  
If there are files to transfer, they are then transferred individually.
If there are more than one source connection (MaxConcurrentConnections is set to > 1), the equal number of destination connections will be opened, and the files are transferred in parallel using these source-destination connection pairs.
For every file in the list returned from the source endpoint, the following process is repeated:

- GetFile  
Get a file from the source endpoint to the local work directory.
If the parameter "RenameSourceFileBeforeTransfer" is set to true (default), the file is first renamed with a temporary filename before transfer.

- ExecuteMessageProcessingSteps  
Execute every MessageProcessingStep defined in the configuration. The steps are executed in sequence, in the same order that they are defined in the configuration.  
The steps process local temporary files as follows: the path to the file originally fetched from the source endpoint is given to the first step. The first step returns the path to the file it created as a result of its processing (this may be the same path as given originally to the step if it did not need to change the file or create a new file). This path is then passed to the next step, which again returns a path to its result file, which is given to the next step, and so on.
Execute SourceFileOperation

- Rename or move the source file.  
This is done before transferring the file to the destination, this means that possible errors in the renaming or moving that would cause the transfer to fail will happen as early as possible - before we actually try to transfer files onward.

- Put the file to destination endpoint  
If destination file already exists, depending on the parameter "DestinationFileExistsAction" either an exception is thrown, the destination file is overwritten or the source file is appended to the destination file.  
If the parameter "RenameDestinationFileDuringTransfer" is true, the file is first transferred with a temporary file name and afterwards renamed to intended filename, otherwise the file is transferred with the intended filename. The intended filename has its possible file masks expanded. 

    Note: When "DestinationFileExistsAction" is set to Error, there is a small chance that the file with the same name will be overwritten. This will only happen if a file with the same name is written by another process after the Cobalt workflow checks for the file's existence.

- Delete source file  
Delete the source file if the parameter "SourceOperation" is set to Delete.

- Finish  
Close the source and destination endpoint connections.  
If the transfer is cancelled (e.g. by calling Terminate on the process instance), the files that are currently being transferred will be processed until finished, but no new files will be transferred. The task instance will finish once all file transfers that were processing at the time of cancellation have finished. The cancelled transfer end result will be Failed.  

When the task has finished, it returns a CobaltResult object which contains the following properties:

| Property | Type | Description |
|-|-|-|
| Success | bool | Did all of the file transfers succeed |
| UserResultMessage | string | A listing of all the files transferred and description of any errors encountered |
| ActionSkipped | bool | True if there were no files to transfer |



#### Error handling

If the opening of the connection to the source endpoint fails, or the listing of the source files fails, the transfer is immediately aborted and an error is logged to the event log.

    Note: This error will not be logged to the database.

If the source connection is opened correctly, but the transfer of a single file fails, the workflow will try to continue from the next file until all the files have been processed. 

    Note: This means there may be many redundant tries to move files if e.g. the user does not have rights to write to the destination. There will be a separate error for each attempt.

On connection errors FTP(S) and SFTP endpoints will try to reopen the connection and retry the failed operation once per operation.

#### Service Bus logging

To enable logging to a service bus message queue the agent running the task needs to have the following application key set:
Frends.Cobalt.FileTransferLogQueueName
This setting can be set in the FRENDSAccService.exe.config file. The configuration will be enabled after a service restart.

### Transfer parameter reference
There are two places where transfer parameters can be set:

- Technical transfer parameters, like workdir location, given as process parameters
- The transfer specific parameters, like source and destination details, given as connection point data

#### Cobalt task transfer parameters

The Cobalt task takes four different kinds of parameters: [Source endpoint](#source) settings, [Destination endpoint](#destination) settings, [Message processing](#message-processing-steps) settings and generic [Parameters](#parameters)

### Source
---
**Type**  
[File | Ftp | Ftps | Sftp | ServiceBus]

The protocol type to use. The type specific configuration is defined in the sections below

**Directory**

Directory on server where the transferred files are located. You can also use relative paths, in which case the path will be resolved relative to the initial working directory. For FTP/SFTP, this directory is the user's home directory, and relative paths should work OK.
    
    Always use absolute paths for File endpoints. Relative paths for File endpoints would resolve under \Windows\System32\, which is almost certainly never correct.

    When using FTP transfer the FTP user must have read, write and modify permissions to source/destination folder. If local directory is used, read, write and modify permissions to the source/destination directory must be assigned to FrendsAccService user.

Macros can be also used for defining the directory. See Macro reference for details on how to use macros.

**FileName**

Name of the file(s) to be transferred. File masks can also be used here for transferring several files. The file mask uses regular expressions, but for convenience, it has special handling for `*` and `?` wildcards. See the table below for examples.

| Mask | Matches | Does not match |
| - | - | - |
| `*t.txt`                      |   text.txt, t.txt                         |   foo.txt, text.foo
| `?t.txt`                      |   2t.txt                                  |   t.txt, tt.foo
| `test.(txt\|xml)`             |   test.txt, test.xml                      |   test.foo, foo.txt
| `test.[^t][^x][^t]`           |   test.xml, test.foo                      |   test.txt, test.tii
| `test\d{4,4}.txt`             |   test1234.txt, test6789.txt              |   test123.txt
| `<regex>^(?!prof).*_test.txt` |   pro_test.txt, pref_test.txt, _test.txt  |   prof_test.txt, pro_tet.txt
    Note: Because regular expressions are used, you need to escape special characters such as parentheses in your filenames with a backslash(\). 
    E.g. Filename: "customer\(123\)_Q1.xml" to only match the file: customer(123)_Q1.xml.
By default, the regular expressions are compiled as case-insensitive, which means that a regular expression: file\d{6,6}env.dta would match both filenames file123456env.dta and FILE445566ENV.DTA. If you want to use pure regular expressions for matching, like if you want your regular expression to be compiled as case sensitive or you want to avoid the special handlings for '*' and '?' characters you have to use `<regex>?` keyword at the beginning of your regular expression. If, for example, you want to make the previous example to match only the uppercase filenames, you would use a regular expression like: `<regex>(?-i)(FILE\d{6,6}ENV.DTA)?`.

**Server**  
Common server parameters for the transfer types that fetch files from a remote server.

**Address**  
Address of the server. E.g. "HOSTNAME" or "192.168.1.23"

**Port**  
Port number for server. E.g. 23

**Username**  
Username to logon to server. 

    Note: File endpoint only supports username for remote shares and needs that the username must be in the format DOMAIN\Username.

**Password**  
Password for user on the server.

**File**  
File specific settings

**FilePaths**  
An array of paths to files to transfer. This value is supposed to come from a file watch trigger reference #trigger.data.filePaths
If a value is given for the FilePaths, you cannot give a FileName in the main configuration.

**Ftp**  
FTP specific settings

**TransportType**  
[Ascii | Binary]  
Ascii mode sends files via FTP as 'text' and should *only* be used for transferring ASCII text files.
Binary modes send files via FTP as raw data and should be used for transferring non-text files. (default)

**Mode**  
[Passive | Active]  
FTP connection mode

**Keep connection alive interval**  
Sends NOOP command to keep connection alive at specified time-interval in seconds. For underlying library the minimum value is 30. If set to 0 the connection is not kept alive. Default value is 0.

**Connection timeout**  
The length of time, in seconds, until the connection times out. You can use value 0 to indicate that the connection does not time out. Default value is 60 seconds.

**Use large buffers**  
If set, will explicitly use larger TCP socket buffers (4 MB for receive, 256 kB for send) instead of the defaults. This may increase especially download speeds.

**Sftp**  
SFTP specific settings

**LoginType**  
[Username-Password | Username-PrivateKey | Username-Password-PrivateKey]  
- Username-Password = the user name and password given in the Server section is used for authentication
- Username-PrivateKey = the user name (given in the Server section) and the private key file (given in this section) are used for authentication.
- Username-Password-PrivateKey = the user name and password as well as the private key is used for authentication. This may not be very commonly supported. GlobalScape SFTP server for example does support it, but use it at your own risk.

**PrivateKeyFileName**  
Location of private key file to be used for authentication if public key authentication is used on the server

**PrivateKeyFilePassword**  
Password for private key file

**ServerFingerPrint**  
Fingerprint for SFTP server. When using Username-Password authentication it is recommended to use server fingerprint in order to be sure of the server you are connecting.

**Connection timeout**  
The length of time, in seconds, until the connection times out. You can use value 0 to indicate that the connection does not time out. Default value is 60 seconds.

**Use large buffers**  
If set, will explicitly use larger TCP socket buffers (4 MB for TCP receive, 256 kB for TCP send, 2 MB for SSH buffer) instead of the defaults. This may increase especially download speeds.

**Preferred host key algorithm**  
[None | RSA | DSS | Certificate | ED25519 | ECDsaNistP256 | ECDsaNistP384 | ECDsaNistP521 | Any]
The default algorithm for calculating the host key signature. DSS by default
- None = No algorithm
- RSA = RSA
- DSS = DSS
- Certificate = X509 certificate
- ED25519 = ED25519, Twisted Edwards Curve EdDSA algorithm
- ECDsaNistP256 = Elliptic Curve Digital Signature Algorithm based on NIST P-256 curve
- ECDsaNistP384 = Elliptic Curve Digital Signature Algorithm based on NIST P-384curve
- ECDsaNistP521 = Elliptic Curve Digital Signature Algorithm based on NIST P-521curve
- Any = Any algorithm

**Ftps**  
FTPS specific settings

**TransportType**  
[Ascii | Binary]  
Ascii mode sends files via FTP as 'text' and should *only* be used for transferring ASCII text files.
Binary modes send files via FTP as raw data and should be used for transferring non-text files. (default)

**Mode**  
[Passive | Active]  
FTP connection mode.

**SecureDataChannel**  
[true | false]  
Is protection of transferred data required.

**SslMode**  
[Explicit | Implicit]  
- Explicit = a separate request is made for establishing secure connection. Default FTP port is used for secure connection.
- Implicit = secure connection is established as soon as the connection to a server is made, but a different port (990) is used for implicit security.

**EnableClientAuth**  
[true | false]  
If client authentication is enabled both client and server are authenticated. The client certificate is searched from user's certificate store.

**ServerName**  
Required if client authentication is enabled. The ServerName must be exactly the same as in server certificate.

**Keep connection alive interval**  
Sends NOOP command to keep connection alive at specified time-interval in seconds. For underlying library the minimum value is 30. If set to 0 the connection is not kept alive. Default value is 0.

**Connection timeout**  
The length of time, in seconds, until the connection times out. You can use value 0 to indicate that the connection does not time out. Default value is 60 seconds.

**Use large buffers**  
If set, will explicitly use larger TCP socket buffers (4 MB for receive, 256 kB for send) instead of the defaults. This may increase especially download speeds.

**ServiceBus**  
Service Bus specific settings. Service bus endpoint can be used for sending and retrieving messages. Due to the limitations of service bus the maximum supported file size for sending is limited to 250KB (Service bus supports message sizes up to 256KB). Endpoint can be used to push many and to retrieve one message per execution. MaxConcurrentConnections should be set to 1 as the same connection is shared by all threads (increasing the value will just cause overhead). The endpoint sends and receives the message body serialized as a Stream.

**ConnectionString**  
Connection string for Azure Service Bus or Service Bus for Windows Server instance to use for messaging.

**Path**  
Path of the entity (queue, topic or subscription) to send or receive messages to or from. When sending messages, give the name of the queue or topic. When receiving messages, give the queue name, or if the entity is a subscription, the entire path to it, in the format [topic name]/subscriptions/[subscription name]
    Things to note:
    The topics, queues or subscriptions will not be created automatically. If they do not exist, an error will be thrown and the transfer will fail.
    For source endpoints, only a single message will be received for a transfer, i.e. batch receive is not supported. The message will also be completed immediately after receive, even before it is written to disk to a temporary file.

### Destination  
---
Destination parameter section contains same parameters as in Source section, so same descriptions generally apply to them too with exception in FileName parameter.

**FileName**  
File name or file mask for the remote file. '?' characters are not allowed in file mask. If no file name is given the original file name is used for destination file name.
Examples of usage:
- No value = use the source file name  
- *.xml = rename files with xml extension  
- *temp = append 'temp' to the source file name  
- new_* renamed file will start with string new_

Macros can also be used for destination file names. See Macro reference for details on how to use macros.

### Message processing steps  
You can define steps for processing messages after they have been fetched from the source endpoint but before they are sent to the destination endpoint.

**Type**  
[PassThrough | LocalBackup | FileSystemBackup | Xslt | Encoding]  
The message processing step type. By default PassThrough.
- PassThrough = No conversion is done.
- LocalBackup = Save a local copy of the transferred file and optionally clean up old backups.
- FileSystemBackup = Back up transferred file on a local file system and clean up old backups.
- Xslt = Convert the files with an XSL transformation e.g. to a text file, HTML file or another XML format.
- Encoding = Changes the encoding of the file.

**LocalBackup**  
Parameters for the backup step that takes a local backup of the transferred files.

**BackupDirectory**  
Root directory for the backups. Must be an absolute local path or network share, like "E:\Backup\" or "\\store\backup". 
For each transfer, a new backup directory named after the file transfer start time and the batch ID is created underneath the root directory. Copies of all files transferred in the single transfer will be put to this transfer specific directory. 

**Cleanup/DaysOlder**  
Number of days to keep the backups. Backup subdirectories older than this are deleted. Dates are compared by the last write time of the directory.

**Cleanup/Enabled**  
[true|false]  
Whether or not the automatic cleanup is enabled. If false (default), the files will not be cleaned up by the step. Note that if the automatic cleanup is disabled, you must clean up the files in some other way.

**FileSystemBackup**  
Parameters for the new backup step that takes a local file system backup of the transferred files. Basically performs the same functionality as LocalBackup step, but with more control over the backup directory and backup file naming.

**Destination**  
Destination must be an absolute local or network path ("E:\Backup\" or "\\store\backup\). Macros can be used to generate the backup directory or/and filename, e.g E:\Backup\%TransferId%\. This would generate a subfolder with transfer id (e.g. '8E9C5EC5-E838-4404-85BF-BC352BDA4462') under 'E:\Backup\' with all the transfered files inside. If a file already exists the existing file will be overwritten.

**Cleanup/DaysOlder**  
Number of days to keep the backups. Backup files and subdirectories are deleted from the base path of the Destination (path without macros). The base path for cleanup for 'E:\Backup\%TransferId%\' would be 'E:\Backup\'. Dates are compared by the last write time for subdirectories. For files the dates are compared by the file creation time.

**Cleanup/Enabled**  
[true|false]  
Whether or not the automatic cleanup is enabled. If false (default), the files will not be cleaned up by the step. Note that if the automatic cleanup is disabled, you must clean up the files in some other way.

**Xslt**  
XSLT processing step parameters

**XslFileLocation**  
Absolute path to the XSL transformation file

**Encoding**  
Encoding step parameters

**SourceEncoding**  
Source file character encoding from which to convert. Possible values are listed in the Name column of the table that appears in MSDN's Encoding class' documentation.

**DestinationEncoding**  
Destination file character encoding to which to convert. Possible values are listed in the Name column of the table that appears in MSDN's Encoding class'  documentation.

### Parameters
---
General parameters for the transfer

**MaxConcurrentTransfers**  
The maximum number of concurrent transfers, by default 1.
If set to more than 1, the given number of source and destination connections are opened, and the files transferred using the opened source-destination connection pairs.

**RenameSourceFileBeforeTransfer**  
[true | false]  
Should the source file be renamed with a temporary file name before transferring the file. This is a way to prevent other users to access the file with its original name while transfer is in progress. Set this to false if the used source endpoint implementation doesn't support source file renaming. Default value is true.
The temporary file name is created using the following scheme: cobalt_[current time in ticks][random 8 character file name].8CO
For example: cobalt_634509139954167644lomy3uzg.8CO

**RenameDestinationFileDuringTransfer**  
[true | false]  
Should the file be transferred to destination endpoint with a temporary file name and rename it after the transfer is completed. This is a way to prevent other users to access the file at the destination endpoint with its actual name before the transfer is completed. Set this to false if the used destination endpoint implementation doesn't support file renaming. Default value is true.
The temporary file name is created using the same scheme as for the source file: cobalt_[current time in ticks][random 8 character file name].8CO

**CreateDestinationDirectories**  
[true | false]  
Should directories be created in transfer destination if they do not exist already. Default value is false.

**DestinationFileExistsAction**  
[Append | Overwrite | Error]  
What to do if destination file exists. By default throws an error.
- Error = Throw an error and fail the transfer.(default)
- Append = Append to an existing file.
- Overwrite = Overwrite existing file.

**NoSourceAction**  
[Error | Info | Ignore]  
What to do if the source file is not found. By default throws an error.

    Note: This error is not logged by log4net transfer logger, neither is it written in the file transfer log database
- Error = Throw an error and fail the transfer. (default)
- Info = Alarm info and quit with success status.
- Ignore = Quit execution with success status.

**SourceOperation**  
What to do to the source file after a successful transfer

**Operation**  
[Delete | Rename | Move | Nothing]  
Defines the operation to perform to source file(s) after successful transfer. By default deletes the file from the source directory.
- Delete = Delete the source file(s).(default)
- Rename = Renames the source file(s) to use the given name. The name (given in the To parameter) can use file macros (and file masks, although they are not recommended). From 1.6 onwards, you can also move the renamed files to another directory, by giving a directory path. 

    Note: This may overwrite files.

- Move = Moves the source file(s) to different directory (given in the To parameter) using the original file names. 

    Note: This may overwrite files.

- Nothing = Do nothing, leaves the file to the source directory.

**To**  
The path to move or to rename the files to.  
For Rename, you can use file macros and also specify a directory where to move the files to, e.g. /subdir/%Date%file.txt. If you don't define a directory path, the source directory is used. When using rename, the To-parameter must always contain a file name.  

    Note: The possible directory must exist before the transfer is executed, because it is not created automatically.

For Move, you must give a directory name. You cannot use file macros, or define file names in the parameter. 

Note: The possible directory must exist before the transfer is executed, because it is not created automatically.

**EnableOperationLog**  
If set, the result object will contain the logs of all file, FTP etc. operations. See the transfer result reference for example.

**TransferName**  
Can be used to give a descriptive name to a single transfer.

**TransferLoggerName** (optional)  
Name for the log4net transfer logger, if one is used.

**WorkDir** (optional)  
Folder that FRENDS Cobalt uses to store local working copies of files. If empty, the FRENDS4 service user account's temporary directory is used. 

    Note: If you define the WorkDir, make sure the FRENDS4 Service user account has read, write and modify access to the directory.

#### Macro reference  
Macros can be used to dynamically configure source directory, destination directory or destination file name for a file transfer.  
Generally the following rules apply for macros:  
- Macros are case insensitive.
- You can use any number of macros in all of the cases.
- Dates and times are formatted with leading zeros.

The following macros can be used with all of dynamically configurable locations for file transfer:
- %Ticks% = will be replace with the current time as Ticks.
- %DateTime% = will be replaced with date and time in format: "yyyy-MM-dd-HH-mm-ss".
- %DateTimeMs% = will be replace with date and time in format: "yyyy-MM-dd-HH-mm-ss-fff".
- %Date% = will be replaced with date in format: "yyyy-MM-dd".
- %Time% = will be replaced with time in format: "HH-mm-ss".
- %Year% = will be replaced with current year.
- %Month% = will be replaced with current month.
- %Day% = will be replaced with current day.
- %Hour% = will be replaced with current hour.
- %Minute% = will be replaced with current minute.
- %Second% = will be replaced with current second.
- %Millisecond% = will be replaced with current millisecond.
- %WeekDay% = will be replaced with a number of weekday, ranging from 1 (monday) to 7 (sunday).
- %Guid% = will be replaced with a new unique identifier.
- %TransferId% = will be replaced with the transfer id.
- %TransferName% = will be replaced with TransferName parameter specified in Connection point schema.
-  %TransferGroupName% = will be replaced with TransferGroupName parameter specified in routine's task arguments.

Destination file name has two additional macros that can be used for dynamically creating destination file name.
- %SourceFileName% = will be replaced with source file name without extension.
- %SourceFileExtension% = will be replaced with source file's extension, with the dot '.' included, i.e. if the source file is named 'foo.txt', the %SourceFileExtension% will be expanded as '.txt'. If the source file name does not have an extension, the macro result will be empty, i.e. for original file name "foo", "bar%SourceFileExtension%" will result in "bar"

    Note: With destination file name it is recommended to only use file macros, since in future the usage of file masks for such purpose will be deprecated. Especially, if you want to add a time stamp to a transferred file name, you can only do this with file macros.

Below are some examples with imaginary sample.txt file:  
Not recommended:  
- *%Date% -> sample.txt2008-08-26
- %Date%* -> 2008-08-26sample.txt

Recommended:
- %SourceFileName%%Date%%SourceFileExtension% -> sample2008-08-26.txt

### Transfer result
The Cobalt transfer will return a result object with the following fields:

| Field | Description | Example |
| - | - | - |
| UserResultMessage | The result message, containing details on all transferred files, possible errors etc. | 3 files transferred: cobaltTestFile1.txt, cobaltTestFile2.txt, cobaltTestFile3.txt |
| Success | If true, the transfer was successful | true |
| ActionSkipped | If true, no files were transferred | false |
| SuccesfulTransferCount | Number of successfully transferred files | 3 |
| FailedTransferCount | Number of files that failed to transfer | 0 |
| TransferredFileNames | Array of file names of transferred files | ["cobaltTestFile1.txt","cobaltTestFile2.txt","cobaltTestFile3.txt"] |
| TransferredFilePaths | Array of full source file paths of transferred files | ["F:\cobaltTestFile1.txt","F:\cobaltTestFile2.txt","F:\cobaltTestFile3.txt"]
| TransferErrors | List of any transfer errors | |
| OperationsLog | If EnableOperationLog setting was on, this field will contain the logs of file, FTP, etc. operations executed during transfer. The log lines will be grouped by time stamp. | { "2016-11-18 08:34:44.60Z": "FILE LIST F:\\cobaltTestFile.txt \nFILE LIST F:\\output_cobaltTestFile.txt...", "2016-11-18 08:34:44.70Z": "FILE EXISTS F:\\cobaltTestFile1.txt: True \nPutFile: Uploading temporary destination file cobalt_6361505488471437865h4itdni.8CO \n...","2016-11-18 08:34:44.80Z": "RestoreSourceFile: Restoring source file from F:\\cobalt_636150548846361630hzickoeu.8CO to the original name cobaltTestFile.txt \n...", ... } |


### Known issues
#### Source file operation may overwrite existing files on FTP
When renaming or moving source the source files with SourceOperation with FTP, you should note that existing files may get overwritten. At least Microsoft's IIS FTP Server overwrites existing files renaming/moving files, this may happen on other servers as well.

If you want to store the original source files somewhere, and do not want to use the backup operation, it is recommended to append e.g. the datetime to the moved or renamed file name by using a file name like %DateTime%*

#### Failure in source file operation logs the transfer failed even if the files were transferred
SourceFileOperations Delete or Nothing are executed only after the actual file transfer has been completed. There is a possibility that they will fail (e.g. another file with the original source file name has been created in the directory, so renaming the temporary file back to the original name (option Nothing) will fail). This error will be logged as a failed file transfer, even though the file has been transferred to the destination. In this case, the logged error description will mention that only the source operation has failed and the file has been transferred.

If you get these kinds of errors, you should fix the original error, e.g. not writing a new file to the source directory with always the same name.
