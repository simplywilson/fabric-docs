---
title: NotebookUtils (former MSSparkUtils) for Fabric
description: Use NotebookUtils, a built-in package for Fabric Notebook, to work with file systems, modularize and chain notebooks together, manage data engineering items, and work with credentials.
ms.reviewer: snehagunda
ms.author: jingzh
author: JeneZhang
ms.topic: how-to
ms.custom:
  - build-2023
  - build-2023-dataai
  - build-2023-fabric
  - ignite-2023
ms.search.form: Microsoft Spark utilities, Microsoft NotebookUtils
ms.date: 07/25/2024
---

# NotebookUtils (former MSSparkUtils) for Fabric

Notebook Utilities (NotebookUtils) is a built-in package to help you easily perform common tasks in Fabric Notebook. You can use NotebookUtils to work with file systems, to get environment variables, to chain notebooks together, and to work with secrets. The NotebookUtils package is available in PySpark (Python) Scala, SparkR notebooks, and Fabric pipelines.

> [!NOTE]
>
> - MsSparkUtils has been officially renamed to **NotebookUtils**. The existing code will remain **backward compatible** and won't cause any breaking changes. It is **strongly recommend** upgrading to notebookutils to ensure continued support and access to new features. The mssparkutils namespace will be retired in the future.
> - NotebookUtils is designed to work with **Spark 3.4(Runtime v1.2) and above**. All new features and updates will be exclusively supported with notebookutils namespace going forward.

## File system utilities

*notebookutils.fs* provides utilities for working with various file systems, including Azure Data Lake Storage (ADLS) Gen2 and Azure Blob Storage. Make sure you configure access to [Azure Data Lake Storage Gen2](/azure/storage/blobs/data-lake-storage-introduction) and [Azure Blob Storage](/azure/storage/blobs/storage-blobs-introduction) appropriately.

Run the following commands for an overview of the available methods:

```python
notebookutils.fs.help()
```

**Output**

```console
notebookutils.fs provides utilities for working with various FileSystems.

Below is overview about the available methods:

cp(from: String, to: String, recurse: Boolean = false): Boolean -> Copies a file or directory, possibly across FileSystems
fastcp(from: String, to: String, recurse: Boolean = true): Boolean -> [Preview] Copies a file or directory via azcopy, possibly across FileSystems
mv(from: String, to: String, createPath: Boolean = false, overwrite: Boolean = false): Boolean -> Moves a file or directory, possibly across FileSystems
ls(dir: String): Array -> Lists the contents of a directory
mkdirs(dir: String): Boolean -> Creates the given directory if it does not exist, also creating any necessary parent directories
put(file: String, contents: String, overwrite: Boolean = false): Boolean -> Writes the given String out to a file, encoded in UTF-8
head(file: String, maxBytes: int = 1024 * 100): String -> Returns up to the first 'maxBytes' bytes of the given file as a String encoded in UTF-8
append(file: String, content: String, createFileIfNotExists: Boolean): Boolean -> Append the content to a file
rm(dir: String, recurse: Boolean = false): Boolean -> Removes a file or directory
exists(file: String): Boolean -> Check if a file or directory exists
mount(source: String, mountPoint: String, extraConfigs: Map[String, Any]): Boolean -> Mounts the given remote storage directory at the given mount point
unmount(mountPoint: String): Boolean -> Deletes a mount point
mounts(): Array[MountPointInfo] -> Show information about what is mounted
getMountPath(mountPoint: String, scope: String = ""): String -> Gets the local path of the mount point

Use notebookutils.fs.help("methodName") for more info about a method.
```

NotebookUtils works with the file system in the same way as Spark APIs. Take *notebookutils.fs.mkdirs()* and Fabric lakehouse usage for example:

| **Usage** | **Relative path from HDFS root** | **Absolute path for ABFS file system** |**Absolute path for local file system in driver node** |
|---|---|---|---|
| Non-default lakehouse | Not supported | *notebookutils.fs.mkdirs("abfss://<container_name>@<storage_account_name>.dfs.core.windows.net/<new_dir>")* | *notebookutils.fs.mkdirs("file:/<new_dir>")* |
| Default lakehouse | Directory under “Files” or “Tables”: *notebookutils.fs.mkdirs("Files/<new_dir>")* | *notebookutils.fs.mkdirs("abfss://<container_name>@<storage_account_name>.dfs.core.windows.net/<new_dir>")* |*notebookutils.fs.mkdirs("file:/<new_dir>")*|

### List files

To list the content of a directory, use *notebookutils.fs.ls('Your directory path')*. For example:

```python
notebookutils.fs.ls("Files/tmp") # works with the default lakehouse files using relative path 
notebookutils.fs.ls("abfss://<container_name>@<storage_account_name>.dfs.core.windows.net/<path>")  # based on ABFS file system 
notebookutils.fs.ls("file:/tmp")  # based on local file system of driver node 
```

### View file properties

This method returns file properties including file name, file path, file size, and whether it's a directory and a file.

```python
files = notebookutils.fs.ls('Your directory path')
for file in files:
    print(file.name, file.isDir, file.isFile, file.path, file.size)
```

### Create new directory

This method creates the given directory if it doesn't exist, and creates any necessary parent directories.

```python
notebookutils.fs.mkdirs('new directory name')  
notebookutils.fs.mkdirs("Files/<new_dir>")  # works with the default lakehouse files using relative path 
notebookutils.fs.ls("abfss://<container_name>@<storage_account_name>.dfs.core.windows.net/<new_dir>")  # based on ABFS file system 
notebookutils.fs.ls("file:/<new_dir>")  # based on local file system of driver node 
```

### Copy file

This method copies a file or directory, and supports copy activity across file systems.

```python
notebookutils.fs.cp('source file or directory', 'destination file or directory', True)# Set the third parameter as True to copy all files and directories recursively
```

### Performant copy file

This method offers a more efficient approach to copying or moving files, particularly when dealing with large data volumes. For enhanced performance on Fabric, it is advisable to utilize `fastcp` as a substitute for the traditional `cp` method.

```python
notebookutils.fs.fastcp('source file or directory', 'destination file or directory', True)# Set the third parameter as True to copy all files and directories recursively
```

### Preview file content

This method returns up to the first 'maxBytes' bytes of the given file as a String encoded in UTF-8.

```python
notebookutils.fs.head('file path', maxBytes to read)
```

### Move file

This method moves a file or directory, and supports moves across file systems.

```python
notebookutils.fs.mv('source file or directory', 'destination directory', True) # Set the last parameter as True to firstly create the parent directory if it does not exist
notebookutils.fs.mv('source file or directory', 'destination directory', True, True) # Set the third parameter to True to firstly create the parent directory if it does not exist. Set the last parameter to True to overwrite the updates.
```

### Write file

This method writes the given string out to a file, encoded in UTF-8.

```python
notebookutils.fs.put("file path", "content to write", True) # Set the last parameter as True to overwrite the file if it existed already
```

### Append content to a file

This method appends the given string to a file, encoded in UTF-8.

```python
notebookutils.fs.append("file path", "content to append", True) # Set the last parameter as True to create the file if it does not exist
```

### Delete file or directory

This method removes a file or directory.

```python
notebookutils.fs.rm('file path', True) # Set the last parameter as True to remove all files and directories recursively
```

### Mount/unmount directory

Find  more information about detailed usage in [File mount and unmount](#file-mount-and-unmount).

## Notebook utilities

Use the Notebook Utilities to run a notebook or exit a notebook with a value. Run the following command to get an overview of the available methods:

```python
notebookutils.notebook.help()
```

**Output:**

```console

The notebook module.

exit(value: String): void -> This method lets you exit a notebook with a value.
run(path: String, timeoutSeconds: int, arguments: Map, workspace: String): String -> This method runs a notebook and returns its exit value.
runMultiple(DAG: Any): Map[String, MsNotebookRunResult] -> [Preview] Runs multiple notebooks concurrently with support for dependency relationships.
validateDAG(DAG: Any): Boolean -> [Preview] This method check if the DAG is correctly defined.

[Preview] Below methods are only support Fabric Notebook.
create(name: String, description: String = "", content: String = "", defaultLakehouse: String = "", defaultLakehouseWorkspace: String = "", workspaceId: String = ""): Artifact -> Create a new Notebook.
get(name: String, workspaceId: String = ""): Artifact -> Get a Notebook by name or id.
update(name: String, newName: String, description: String = "", workspaceId: String = ""): Artifact -> Update a Artifact by name.
delete(name: String, workspaceId: String = ""): Boolean -> Delete a Notebook by name.
list(workspaceId: String = "", maxResults: Int = 1000): Array[Artifact] -> List all Notebooks in the workspace.
updateDefinition(name: String, content: String = "", defaultLakehouse: String = "", defaultLakehouseWorkspace: String = "", workspaceId: String = "") -> Update the definition of a Notebook.

Use notebookutils.notebook.help("methodName") for more info about a method.
```

> [!NOTE]
> Notebook utilities aren't applicable for Apache Spark job definitions (SJD).

### Reference a notebook

This method references a notebook and returns its exit value. You can run nesting function calls in a notebook interactively or in a pipeline. The notebook being referenced runs on the Spark pool of the notebook that calls this function.

```python
notebookutils.notebook.run("notebook name", <timeoutSeconds>, <parameterMap>, <workspaceId>)
```

For example:

```python
notebookutils.notebook.run("Sample1", 90, {"input": 20 })
```

Fabric notebook also supports referencing notebooks across multiple workspaces by specifying the *workspace ID*.

```python
notebookutils.notebook.run("Sample1", 90, {"input": 20 }, "fe0a6e2a-a909-4aa3-a698-0a651de790aa")
```

You can open the snapshot link of the reference run in the cell output. The snapshot captures the code run results and allows you to easily debug a reference run.

:::image type="content" source="media\notebook-utilities\reference-run.png" alt-text="Screenshot of reference run result." lightbox="media\notebook-utilities\reference-run.png":::

:::image type="content" source="media\notebook-utilities\run-snapshot.png" alt-text="Screenshot of a snapshot example." lightbox="media\notebook-utilities\run-snapshot.png":::

> [!NOTE]
>
> - The cross-workspace reference notebook is supported by **runtime version 1.2 and above**.
> - If you use the files under [Notebook Resource](how-to-use-notebook.md#notebook-resources), use `notebookutils.nbResPath` in the referenced notebook to make sure it points to the same folder as the interactive run.

### Reference run multiple notebooks in parallel

> [!IMPORTANT]
> This feature is in [preview](../get-started/preview.md).

The method `notebookutils.notebook.runMultiple()` allows you to run multiple notebooks in parallel or with a predefined topological structure. The API is using a multi-thread implementation mechanism within a spark session, which means the compute resources are shared by the reference notebook runs.

With `notebookutils.notebook.runMultiple()`, you can:

- Execute multiple notebooks simultaneously, without waiting for each one to finish.

- Specify the dependencies and order of execution for your notebooks, using a simple JSON format.

- Optimize the use of Spark compute resources and reduce the cost of your Fabric projects.

- View the Snapshots of each notebook run record in the output, and debug/monitor your notebook tasks conveniently.

- Get the exit value of each executive activity and use them in downstream tasks.

You can also try to run the notebookutils.notebook.help("runMultiple") to find the example and detailed usage.

Here's a simple example of running a list of notebooks in parallel using this method:

```python

notebookutils.notebook.runMultiple(["NotebookSimple", "NotebookSimple2"])

```

The execution result from the root notebook is as follows:

:::image type="content" source="media\notebook-utilities\reference-notebook-list.png" alt-text="Screenshot of reference a list of notebooks." lightbox="media\notebook-utilities\reference-notebook-list.png":::

The following is an example of running notebooks with topological structure using `notebookutils.notebook.runMultiple()`. Use this method to easily orchestrate notebooks through a code experience.

```python
# run multiple notebooks with parameters
DAG = {
    "activities": [
        {
            "name": "NotebookSimple", # activity name, must be unique
            "path": "NotebookSimple", # notebook path
            "timeoutPerCellInSeconds": 90, # max timeout for each cell, default to 90 seconds
            "args": {"p1": "changed value", "p2": 100}, # notebook parameters
        },
        {
            "name": "NotebookSimple2",
            "path": "NotebookSimple2",
            "timeoutPerCellInSeconds": 120,
            "args": {"p1": "changed value 2", "p2": 200}
        },
        {
            "name": "NotebookSimple2.2",
            "path": "NotebookSimple2",
            "timeoutPerCellInSeconds": 120,
            "args": {"p1": "changed value 3", "p2": 300},
            "retry": 1,
            "retryIntervalInSeconds": 10,
            "dependencies": ["NotebookSimple"] # list of activity names that this activity depends on
        }
    ],
    "timeoutInSeconds": 43200, # max timeout for the entire DAG, default to 12 hours
    "concurrency": 50 # max number of notebooks to run concurrently, default to 50
}
notebookutils.notebook.runMultiple(DAG, {"displayDAGViaGraphviz": False})
```

The execution result from the root notebook is as follows:

:::image type="content" source="media\notebook-utilities\reference-notebook-list-with-parameters.png" alt-text="Screenshot of reference a list of notebooks with parameters." lightbox="media\notebook-utilities\reference-notebook-list-with-parameters.png":::

We also provide a method to check if the DAG is correctly defined.
```python
notebookutils.notebook.validateDAG(DAG)
```

> [!NOTE]
> - The parallelism degree of the multiple notebook run is restricted to the total available compute resource of a Spark session.
> - The upper limit for notebook activities or concurrent notebooks is **50**. Exceeding this limit may lead to stability and performance issues due to high compute resource usage. If issues arise, consider separating notebooks into multiple ```runMultiple``` calls or reducing the concurrency by adjusting the **concurrency** field in the DAG parameter.
> - The default timeout for entire DAG is 12 hours, and the default timeout for each cell in child notebook is 90 seconds. You can change the timeout by setting the **timeoutInSeconds** and **timeoutPerCellInSeconds** fields in the DAG parameter.

### Exit a notebook

This method exits a notebook with a value. You can run nesting function calls in a notebook interactively or in a pipeline.

- When you call an *exit()* function from a notebook interactively, the Fabric notebook throws an exception, skips running subsequent cells, and keeps the Spark session alive.

- When you orchestrate a notebook in a pipeline that calls an *exit()* function, the notebook activity returns with an exit value, completes the pipeline run, and stops the Spark session.

- When you call an *exit()* function in a notebook that is being referenced, Fabric Spark will stop the further execution of the referenced notebook, and continue to run the next cells in the main notebook that calls the *run()* function. For example: Notebook1 has three cells and calls an *exit()* function in the second cell. Notebook2 has five cells and calls *run(notebook1)* in the third cell. When you run Notebook2, Notebook1 stops at the second cell when hitting the *exit()* function. Notebook2 continues to run its fourth cell and fifth cell.

```python
notebookutils.notebook.exit("value string")
```

For example:

**Sample1** notebook with following two cells:

- Cell 1 defines an **input** parameter with default value set to 10.

- Cell 2 exits the notebook with **input** as exit value.

:::image type="content" source="media\notebook-utilities\input-exit-value.png" alt-text="Screenshot showing a sample notebook of exit function." lightbox="media\notebook-utilities\input-exit-value.png":::

You can run the **Sample1** in another notebook with default values:

```python
exitVal = notebookutils.notebook.run("Sample1")
print (exitVal)
```

**Output:**

```console
Notebook is executed successfully with exit value 10
```

You can run the **Sample1** in another notebook and set the **input** value as 20:

```python
exitVal = notebookutils.notebook.run("Sample1", 90, {"input": 20 })
print (exitVal)
```

**Output:**

```console
Notebook is executed successfully with exit value 20
```
<!---
## Session management

### Stop an interactive session

Instead of manually selecting stop, sometimes it's more convenient to stop an interactive session by calling an API in the code. For such cases, we provide an API *mssparkutils.session.stop()* to support stopping the interactive session via code. It's available for Scala and Python.

```python
mssparkutils.session.stop()
```

The *mssparkutils.session.stop()* API stops the current interactive session asynchronously in the background. It stops the Spark session and release resources occupied by the session so they're available to other sessions in the same pool.

> [!NOTE]
> We don't recommend calling language built-in APIs like *sys.exit* in Scala or *sys.exit()* in Python in your code, because such APIs kill the interpreter process, leaving the Spark session alive and the resources not released.
--->

### Manage notebook artifacts

`notebookutils.notebook` provides specialized utilities for managing Notebook items programmatically. These APIs can help you create, get, update, and delete Notebook items easily.

To utilize these methods effectively, consider the following usage examples:

#### Creating a Notebook

```python
with open("/path/to/notebook.ipynb", "r") as f:
    content = f.read()

artifact = notebookutils.notebook.create("artifact_name", "description", "content", "default_lakehouse_name", "default_lakehouse_workspace_id", "optional_workspace_id")
```

#### Getting content of a Notebook

```python
artifact = notebookutils.notebook.get("artifact_name", "optional_workspace_id")
```

#### Updating a Notebook

```python
updated_artifact = notebookutils.notebook.update("old_name", "new_name", "optional_description", "optional_workspace_id")
```

```python
updated_artifact_definition = notebookutils.notebook.updateDefinition("artifact_name",  "content", "default_lakehouse_name", "default_Lakehouse_Workspace_name", "optional_workspace_id")
```

#### Deleting a Notebook

```python
is_deleted = notebookutils.notebook.delete("artifact_name", "optional_workspace_id")
```

#### Listing Notebooks in a workspace

```python
artifacts_list = notebookutils.notebook.list("optional_workspace_id")
```

## Credentials utilities

You can use the Credentials Utilities to get access tokens and manage secrets in an Azure Key Vault.

Run the following command to get an overview of the available methods:

```python
notebookutils.credentials.help()
```

**Output:**

```console
Help on module notebookutils.credentials in notebookutils:

NAME
    notebookutils.credentials - Utility for credentials operations in Fabric

FUNCTIONS
    getSecret(akvName, secret) -> str
        Gets a secret from the given Azure Key Vault.
        :param akvName: The name of the Azure Key Vault.
        :param secret: The name of the secret.
        :return: The secret value.
    
    getToken(audience) -> str
        Gets a token for the given audience.
        :param audience: The audience for the token.
        :return: The token.
    
    help(method_name=None)
        Provides help for the notebookutils.credentials module or the specified method.
        
        Examples:
        notebookutils.credentials.help()
        notebookutils.credentials.help("getToken")
        :param method_name: The name of the method to get help with.

DATA
    creds = <notebookutils.notebookutils.handlers.CredsHandler.CredsHandler...

FILE
    /home/trusted-service-user/cluster-env/trident_env/lib/python3.10/site-packages/notebookutils/credentials.py
```

### Get token

getToken returns a Microsoft Entra token for a given audience and name (optional). The following list shows the currently available audience keys:

- **Storage Audience Resource**: "storage"
- **Power BI Resource**: "pbi"
- **Azure Key Vault Resource**: "keyvault"
- **Synapse RTA KQL DB Resource**: "kusto"

Run the following command to get the token:

```python
notebookutils.credentials.getToken('audience Key')
```

### Get secret using user credentials

getSecret returns an Azure Key Vault secret for a given Azure Key Vault endpoint and secret name using user credentials.

```python
notebookutils.credentials.getSecret('https://<name>.vault.azure.net/', 'secret name')
```

## File mount and unmount

Fabric supports the following mount scenarios in the Microsoft Spark Utilities package. You can use the *mount*, *unmount*, *getMountPath()*, and *mounts()* APIs to attach remote storage (ADLS Gen2) to all working nodes (driver node and worker nodes). After the storage mount point is in place, use the local file API to access data as if it's stored in the local file system.

### How to mount an ADLS Gen2 account

The following example illustrates how to mount Azure Data Lake Storage Gen2. Mounting Blob Storage works similarly.

This example assumes that you have one Data Lake Storage Gen2 account named *storegen2*, and the account has one container named *mycontainer* that you want to mount to */test* into your notebook Spark session.

:::image type="content" source="media\notebook-utilities\mount-container-example.png" alt-text="Screenshot showing where to select a container to mount." lightbox="media\notebook-utilities\mount-container-example.png":::

To mount the container called *mycontainer*, *notebookutils* first needs to check whether you have the permission to access the container. Currently, Fabric supports two authentication methods for the trigger mount operation: *accountKey* and *sastoken*.

### Mount via shared access signature token or account key

NotebookUtils supports explicitly passing an account key or [Shared access signature (SAS)](/azure/storage/common/storage-sas-overview) token as a parameter to mount the target.

For security reasons, we recommend that you store account keys or SAS tokens in Azure Key Vault (as the following screenshot shows). You can then retrieve them by using the *notebookutils.credentials.getSecret* API. For more information about Azure Key Vault, see [About Azure Key Vault managed storage account keys](/azure/key-vault/secrets/about-managed-storage-account-keys).

:::image type="content" source="media\notebook-utilities\use-azure-key-vault.png" alt-text="Screenshot showing where secrets are stored in an Azure Key Vault." lightbox="media\notebook-utilities\use-azure-key-vault.png":::

Sample code for the *accountKey* method:

```python
# get access token for keyvault resource
# you can also use full audience here like https://vault.azure.net
accountKey = notebookutils.credentials.getSecret("<vaultURI>", "<secretName>")
notebookutils.fs.mount(  
    "abfss://mycontainer@<accountname>.dfs.core.windows.net",  
    "/test",  
    {"accountKey":accountKey}
)
```

Sample code for *sastoken*:

```python
# get access token for keyvault resource
# you can also use full audience here like https://vault.azure.net
sasToken = notebookutils.credentials.getSecret("<vaultURI>", "<secretName>")
notebookutils.fs.mount(  
    "abfss://mycontainer@<accountname>.dfs.core.windows.net",  
    "/test",  
    {"sasToken":sasToken}
)
```

Mount parameters:
- fileCacheTimeout: Blobs will be cached in the local temp folder for 120 seconds by default. During this time, blobfuse will not check whether the file is up to date or not. The parameter could be set to change the default timeout time. When multiple clients modify files at the same time, in order to avoid inconsistencies between local and remote files, we recommend shortening the cache time, or even changing it to 0, and always getting the latest files from the server.
- timeout: The mount operation timeout is 120 seconds by default. The parameter could be set to change the default timeout time. When there are too many executors or when mount times out, we recommend increasing the value.

You can use these parameters like this:

```python
notebookutils.fs.mount(
   "abfss://mycontainer@<accountname>.dfs.core.windows.net",
   "/test",
   {"fileCacheTimeout": 120, "timeout": 120}
)
```

> [!NOTE]
> For security purposes, it is advised to avoid embedding credentials directly in code. To further safeguard your credentials, any secrets displayed in notebook outputs will be redacted. For more information, see [Secret redaction](author-execute-notebook.md#secret-redaction).

### How to mount a lakehouse

Sample code for mounting a lakehouse to */test*:

```python
notebookutils.fs.mount( 
 "abfss://<workspace_id>@msit-onelake.dfs.fabric.microsoft.com/<lakehouse_id>", 
 "/test"
)
```

### Access files under the mount point by using the *notebookutils fs* API

The main purpose of the mount operation is to let customers access the data stored in a remote storage account with a local file system API. You can also access the data by using the *notebookutils fs* API with a mounted path as a parameter. This path format is a little different.

Assume that you mounted the Data Lake Storage Gen2 container *mycontainer* to */test* by using the mount API. When you access the data with a local file system API, the path format is like this:

```python
/synfs/notebook/{sessionId}/test/{filename}
```

When you want to access the data by using the notebookutils fs API, we recommend using *getMountPath()* to get the accurate path:

```python
path = notebookutils.fs.getMountPath("/test")
```

- List directories:

   ```python
   notebookutils.fs.ls(f"file://{notebookutils.fs.getMountPath('/test')}")
   ```

- Read file content:

   ```python
   notebookutils.fs.head(f"file://{notebookutils.fs.getMountPath('/test')}/myFile.txt")
   ```

- Create a directory:

   ```python
   notebookutils.fs.mkdirs(f"file://{notebookutils.fs.getMountPath('/test')}/newdir")
   ```

### Access files under the mount point via local path

You can easily read and write the files in mount point using the standard file system. Here's a Python example:

```python
#File read
with open(notebookutils.fs.getMountPath('/test2') + "/myFile.txt", "r") as f:
    print(f.read())
#File write
with open(notebookutils.fs.getMountPath('/test2') + "/myFile.txt", "w") as f:
    print(f.write("dummy data"))
```

### How to check existing mount points

You can use *notebookutils.fs.mounts()* API to check all existing mount point info:

```python
notebookutils.fs.mounts()
```

### How to unmount the mount point

Use the following code to unmount your mount point *(/test* in this example):

```python
notebookutils.fs.unmount("/test")
```

### Known limitations

- The current mount is a job level configuration; we recommend you use the *mounts* API to check if a mount point exists or not available.

- The unmount mechanism is not automatically applied. When the application run finishes, to unmount the mount point and release the disk space, you need to explicitly call an unmount API in your code. Otherwise, the mount point will still exist in the node after the application run finishes.

- Mounting an ADLS Gen1 storage account is not supported.


## Lakehouse utilities

`notebookutils.lakehouse` provides utilities specifically tailored for managing Lakehouse items. These utilities empower you to create, get, update, and delete Lakehouse artifacts effortlessly.

### Overview of methods

Below is an overview of the available methods provided by `notebookutils.lakehouse`:

```python
# Create a new Lakehouse artifact
create(name: String, description: String = "", definition: ItemDefinition = null, workspaceId: String = ""): Artifact

# Retrieve a Lakehouse artifact
get(name: String, workspaceId: String = ""): Artifact

# Get a Lakehouse artifact with properties
getWithProperties(name: String, workspaceId: String = ""): Artifact

# Update an existing Lakehouse artifact
update(name: String, newName: String, description: String = "", workspaceId: String = ""): Artifact

# Delete a Lakehouse artifact
delete(name: String, workspaceId: String = ""): Boolean 

# List all Lakehouse artifacts
list(workspaceId: String = "", maxResults: Int = 1000): Array[Artifact]

# List all tables in a Lakehouse artifact
listTables(lakehouse: String, workspaceId: String = "", maxResults: Int = 1000): Array[Table] 

# Starts a load table operation in a Lakehouse artifact
loadTable(loadOption: collection.Map[String, Any], table: String, lakehouse: String, workspaceId: String = ""): Array[Table] 
```

### Usage examples

To utilize these methods effectively, consider the following usage examples:

#### Creating a Lakehouse
```python
artifact = notebookutils.lakehouse.create("artifact_name", "Description of the artifact", "optional_workspace_id")
```

#### Getting a Lakehouse
```python
artifact = notebookutils.lakehouse.get("artifact_name", "optional_workspace_id")
```

```python
artifact = notebookutils.lakehouse.getWithProperties("artifact_name", "optional_workspace_id")
```

#### Updating a Lakehouse
```python
updated_artifact = notebookutils.lakehouse.update("old_name", "new_name", "Updated description", "optional_workspace_id")
```

#### Deleting a Lakehouse
```python
is_deleted = notebookutils.lakehouse.delete("artifact_name", "optional_workspace_id")
```

#### Listing Lakehouses in a workspace
```python
artifacts_list = notebookutils.lakehouse.list("optional_workspace_id")
```

#### Listing all tables in a Lakehouse
```python
artifacts_tables_list = notebookutils.lakehouse.listTables("artifact_name", "optional_workspace_id")
```

#### Starting a load table operation in a Lakehouse
```python
notebookutils.lakehouse.loadTable(
    {
        "relativePath": "Files/myFile.csv",
        "pathType": "File",
        "mode": "Overwrite",
        "recursive": False,
        "formatOptions": {
            "format": "Csv",
            "header": True,
            "delimiter": ","
        }
    }, "table_name", "artifact_name", "optional_workspace_id")
```

### Additional information

For more detailed information about each method and its parameters, utilize the `notebookutils.lakehouse.help("methodName")` function.

## Runtime utilities

### Show the session context info

With ``` notebookutils.runtime.context ``` you can get the context information of the current live session, including the notebook name, default lakehouse, workspace info, if it's a pipeline run, etc.

```python
notebookutils.runtime.context
```

## Known issue 

When using runtime version above 1.2 and run ``` notebookutils.help() ```, the listed **fabricClient**, **PBIClient** APIs are not supported for now, will be available in the further. Additionally, the **Credentials** API isn't supported in Scala notebooks for now.

## Related content

- [Microsoft Spark Utilities (MSSparkUtils) for Fabric](microsoft-spark-utilities.md)
- [Develop, execute, and manage Microsoft Fabric notebooks](author-execute-notebook.md)
- [Manage and execute notebooks in Fabric with APIs](notebook-public-api.md)