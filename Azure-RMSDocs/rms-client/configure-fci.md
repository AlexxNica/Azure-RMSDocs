---
# required metadata

title: Azure RMS protection with Windows Server FCI - AIP
description: Instructions to use the Rights Management (RMS) client with the RMS Protection tool to configure File Server Resource Manager and file classification infrastructure (FCI).
author: cabailey
ms.author: cabailey
manager: mbaldwin
ms.date: 03/22/2017
ms.topic: article
ms.prod:
ms.service: information-protection
ms.technology: techgroup-identity
ms.assetid: 9aa693db-9727-4284-9f64-867681e114c9

# optional metadata

#ROBOTS:
#audience:
#ms.devlang:
ms.reviewer: esaggese
ms.suite: ems
#ms.tgt_pltfrm:
#ms.custom:

---

# RMS protection with Windows Server File Classification Infrastructure (FCI)

>*Applies to: Azure Information Protection, Windows Server 2016, Windows Server 2012, Windows Server 2012 R2*

Use this article for instructions and a script to use the Azure Information Protection client and PowerShell to configure File Server Resource Manager and File Classification Infrastructure (FCI).

This solution lets you automatically protect all files in a folder on a file server running Windows Server, or automatically protect files that meet a specific criteria. For example, files that have been classified as containing confidential or sensitive information. This solution connects directly to the Azure Rights Management service from Azure Information Protection to protect the files, so you must have this service deployed for your organization.

> [!NOTE]
> Although Azure Information Protection includes a [connector](../deploy-use/deploy-rms-connector.md) that supports File Classification Infrastructure, that solution supports native protection only—for example, Office files.
> 
> To support multiple file types with Windows Server file classification infrastructure, you must use the PowerShell **AzureInformationProtection** module, as documented in this article. The Azure Information Protection cmdlets, like the Azure Information Protection client, support generic protection as well as native protection, which means that file types other than Office documents can be protected. For more information, see [File types supported by the Azure Information Protection client](client-admin-guide-file-types.md) from the Azure Information Protection client admin guide.

The instructions that follow are for Windows Server 2012 R2 or Windows Server 2012. If you run other supported versions of Windows, you might need to adapt some of the steps for differences between your operating system version and the one documented in this article.

## Prerequisites for Azure Rights Management protection with Windows Server FCI
Prerequisites for these instructions:

-   On each file server where you will run File Resource Manager with file classification infrastructure:

    -   You have installed File Server Resource Manager as one of the role services for the File Services role.

    -   You have identified a local folder that contains files to protect with Rights Management. For example, C:\FileShare.

    -   You have installed the AzureInformationProtection module and configured the prerequisites for Azure Rights Management. For more information, see [Using PowerShell with the Azure Information Protection client](client-admin-guide-powershell.md). Specifically, you have the following values to connect to the Azure Rights Management service by using a service principal: **BposTenantId**, **AppPrincipalId**, and **Symmetric key**. 

    -   If you want to change the default level of protection (native or generic) for specific file name extensions, you have edited the registry as described in the [Changing the default protection level of files](client-admin-guide-file-types.md#changing-the-default-protection-level-of-files) section from the admin guide.

    -   You have an Internet connection, with configured computer settings if required for a proxy server. For example: `netsh winhttp import proxy source=ie`

-   You have synchronized your on-premises Active Directory user accounts with Azure Active Directory or Office 365, including their email address. This is required for all users that might need to access files after they are protected by FCI and the Azure Rights Management service. If you do not do this step (for example, in a test environment), users might be blocked from accessing these files. If you need more information about this account configuration, see [Preparing for the Azure Rights Management service](../plan-design/prepare.md).

-   You have downloaded to the file server the Rights Management templates and identified the template ID that will protect the files. To do this, use the [Get-RMSTemplate](/powershell/azureinformationprotection/vlatest/get-rmstemplate) cmdlet. This scenario does not support departmental templates so you must either use a template that is not configured for a scope, or the scope configuration must include the application compatibility option such that the **Show this template to all users when the applications do not support user identity** check box is selected.

## Instructions to configure File Server Resource Manager FCI for Azure Rights Management protection
Follow these instructions to automatically protect all files in a folder, by using a PowerShell script as a custom task. Do these procedures in this order:

1.  Save the PowerShell script

2.  Create a classification property for Rights Management (RMS)

3.  Create a classification rule (Classify for RMS)

4.  Configure the classification schedule

5.  Create a custom file management task (Protect files with RMS)

6.  Test the configuration by manually running the rule and task

At the end of these instructions, all files in your selected folder will be classified with the custom property of RMS, and these files will then be protected by Rights Management. For a more complex configuration that selectively protects some files and not others, you can then create or use a different classification property and rule, with a file management task that protects just those files.

Note that if you make changes to the Rights Management template that you use for FCI, you must run `Get-RMSTemplate -Force` on the file server computer to get the updated template. The updated template will then be used to protect new files. If the changes to the template are important enough to re-protect the files on the file server, you can do this by running the Protect-RMSFile cmdlet interactively with an account that has the Export or Full Control usage rights for the files. You must also run `Get-RMSTemplate -Force` on this file server computer if you publish a new template that you want to use for FCI.

### Save the Windows PowerShell script

1.  Copy the contents of the [Windows PowerShell script](fci-script.md) for Azure RMS protection by using File Server Resource Manager. Paste the contents of the script and  name the file **RMS-Protect-FCI.ps1** on your own computer.

2.  Review the script and make the following changes:

    -   Search for the following string and replace it with your own AppPrincipalId that you use with the [Set-RMSServerAuthentication](/powershell/azureinformationprotection/vlatest/set-rmsserverauthentication) cmdlet to connect to the Azure Rights Management service:

        ```
        <enter your AppPrincipalId here>
        ```
        For example, the script might look like this:

        `[Parameter(Mandatory = $false)]`

        `[Parameter(Mandatory = $false)]             [string]$AppPrincipalId = "b5e3f76a-b5c2-4c96-a594-a0807f65bba4",`

    -   Search for the following string and replace it with your own symmetric key that you use with the [Set-RMSServerAuthentication](/powershell/azureinformationprotection/vlatest/set-rmsserverauthentication) cmdlet to connect to the Azure Rights Management service:

        ```
        <enter your key here>
        ```
        For example, the script might look like this:

        `[Parameter(Mandatory = $false)]`

        `[string]$SymmetricKey = "zIeMu8zNJ6U377CLtppkhkbl4gjodmYSXUVwAO5ycgA="`

    -   Search for the following string and replace it with your own BposTenantId (tenant ID) that you use with the [Set-RMSServerAuthentication](/powershell/azureinformationprotection/vlatest/set-rmsserverauthentication) cmdlet to connect to the Azure Rights Management service:

        ```
        <enter your BposTenantId here>
        ```
        For example, the script might look like this:

        `[Parameter(Mandatory = $false)]`

        `[string]$BposTenantId = "23976bc6-dcd4-4173-9d96-dad1f48efd42",`

3.  Sign the script. If you do not sign the script (more secure), you must configure Windows PowerShell on the servers that run it. For example, run a Windows PowerShell session with the **Run as Administrator** option, and type: **Set-ExecutionPolicy RemoteSigned**. However, this configuration lets all unsigned scripts run when they are stored on this server (less secure).

    For more information about signing Windows PowerShell scripts, see [about_Signing](https://technet.microsoft.com/library/hh847874.aspx) in the PowerShell documentation library.

4.  Save the file locally on each file server that will run File Resource Manager with file classification infrastructure. For example, save the file in **C:\RMS-Protection**. If you use a different path or folder name, choose a path and folder that does not include spaces. Secure this file by using NTFS permissions so that unauthorized users cannot modify it.

You're now ready to start configuring File Server Resource Manager.

### Create a classification property for Rights Management (RMS)

-   In File Server Resource Manager, Classification Management, create a new local property:

    -   **Name**: Type **RMS**

    -   **Description**:   Type **Rights Management protection**

    -   **Property Type**: Select **Yes/No**

    -   **Value**: Select **Yes**

We can now create a classification rule that uses this property.

### Create a classification rule (Classify for RMS)

-   Create a new classification rule:

    -   On the **General** tab:

        -   **Name**: Type **Classify for RMS**

        -   **Enabled**: Keep the default, which is that this checkbox is selected.

        -   **Description**: Type **Classify all files in the &lt;folder name&gt; folder for Rights Management**.

            Replace *&lt;folder name&gt;* with your chosen folder name. For example, **Classify all files in the C:\FileShare folder for Rights Management**

        -   **Scope**: Add your chosen folder. For example, **C:\FileShare**.

            Do not select the checkboxes.

    -   On the **Classification** tab:

    -   **Classification method**: Select **Folder Classifier**

    -   **Property** name: Select **RMS**

    -   Property **value**: Select **Yes**

Although you can run the classification rules manually, for ongoing operations, you will want this rule to run on a schedule so that new files will be classified with the RMS property.

### Configure the classification schedule

-   On the **Automatic Classification** tab:

    -   **Enable fixed schedule**: Select this checkbox.

    -   Configure the schedule for all classification rules to run, which includes our new rule to classify files with the RMS property.

    -   **Allow continuous classification for new files**: Select this check box so that new files will be classified.

    -   Optional: Make any other changes that you want, such as configuring options for reports and notifications.

Now you've completed the classification configuration, you're ready to configure a management task to apply the RMS protection to the files.

### Create a custom file management task (Protect files with RMS)

-   In **File Management Tasks**, create a new file management task:

    -   On the **General** tab:

        -   **Task name**: Type **Protect files with RMS**

        -   Keep the **Enable** checkbox selected.

        -   **Description**: Type **Protect files in &lt;folder name&gt; with Rights Management and a template by using a Windows PowerShell script.**

            Replace *&lt;folder name&gt;* with your chosen folder name. For example, **Protect files in C:\FileShare with Rights Management and a template by using a Windows PowerShell script**

        -   **Scope**: Select your chosen folder. For example, **C:\FileShare**.

            Do not select the checkboxes.

    -   On the **Action** tab:

        -   **Type**: Select **Custom**

        -   **Executable**: Specify the following:

            ```
            C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
            ```
            If Windows is not on your C: drive, modify this path or browse to this file.

        -   **Argument**: Specify the following, supplying your own values for &lt;path&gt; and &lt;template ID&gt;:

            ```
            -Noprofile -Command "<path>\RMS-Protect-FCI.ps1 -File '[Source File Path]' -TemplateID <template GUID> -OwnerMail [Source File Owner Email]"
            ```
            For example, if you copied the script to C:\RMS-Protection and the template ID you identified from the prerequisites is e6ee2481-26b9-45e5-b34a-f744eacd53b0, specify the following:

            `-Noprofile -Command "C:\RMS-Protection\RMS-Protect-FCI.ps1 -File '[Source File Path]' -TemplateID e6ee2481-26b9-45e5-b34a-f744eacd53b0 -OwnerMail [Source File Owner Email]"`

            In this command, **[Source File Path]** and **[Source File Owner Email]** are both FCI-specific variables, so type these exactly as they appear in the command above. The first one is used by FCI to automatically specify the identified file in the folder, and the second is for FCI to automatically retrieve the email address of the named Owner of the identified file. This command is repeated for each file in the folder, which in our example, is each file in the C:\FileShare folder that additionally, has RMS as a file classification property.

            > [!NOTE]
            > The **-OwnerMail [Source File Owner Email]** parameter and value ensures that the original owner of the file is granted the Rights Management owner of the file after it is protected. This ensures that the original file owner has all Rights Management rights to their own files. When files are created by a domain user, the email address is  automatically retrieved from Active Directory by using the user account name in the file's Owner property. To do this, the file server must be in the same domain or trusted domain as the user.
            > 
            > Whenever possible, assign the original owners to protected documents, to ensure that these users continue to have full control over the files that they created. However, if you use the [Source File Owner Email] variable as above, and a file does not have a domain user defined as the owner (for example, a local account was used to create the file, so the owner displays SYSTEM), the script will fail.
            > 
            > For files that do not have a domain user as owner, you can either copy and save these files yourself as a domain user, so that you become the owner for just these files. Or, if you have permissions, you can manually change the owner.  Or alternatively, you can supply a specific email address (such as your own or a group address for the IT department) instead of the [Source File Owner Email] variable, which means that all files you protect by using this script will use this email address to define the new owner.

    -   **Run the command as**: Select **Local System**

    -   On the **Condition** tab:

        -   **Property**: Select **RMS**

        -   **Operator**: Select **Equal**

        -   **Value**: Select **Yes**

    -   On the **Schedule** tab:

        -   **Run at**: Configure your preferred schedule.

            Allow plenty of time for the script to complete. Although this solution  protects all files in the folder, the script runs once for each file, each time. Although this takes longer than protecting all the files at the same time, which the RMS Protection tool supports, this file-by-file configuration for FCI is more powerful. For example, the protected files can have different owners (retain the original owner) when you use the   [Source File Owner Email] variable, and this file-by-file action will be required if you later change the configuration to selectively protect files rather than all files in a folder.

        -   **Run continuously on new files**: Select this checkbox.

### Test the configuration by manually running the rule and task

1.  Run the classification rule:

    1.  Click **Classification Rules** &gt; **Run Classification With All Rules Now**

    2.  Click **Wait for classification to complete**, and then click **OK**.

2.  Wait for the **Running Classification** dialog box to close and then view the results in the automatically displayed report. You should see **1** for the **Properties** field and the number of files in your folder. Confirm by using File Explorer and checking the properties of files in your chosen folder. On the **Classification** tab, you should see **RMS** as a property name and **Yes** for its **Value**.

3.  Run the file management task:

    1.  Click **File Management Tasks** &gt; **Protect files with RMS** &gt; **Run File Management Task Now**

    2.  Click **Wait for the task to complete**, and then click **OK**.

4.  Wait for the **Running File Management Task** dialog box to close and then view the results in the automatically displayed report. You should see the number of files that are in your chosen folder in the **Files** field. Confirm that the files in your chosen folder are now protected by RMS. For example, if your chosen folder is C:\FileShare, type the following in a Windows PowerShell session and confirm that no files have a status of **UnProtected**:

    ```
    foreach ($file in (Get-ChildItem -Path C:\FileShare -Force | where {!$_.PSIsContainer})) {Get-RMSFileStatus -f $file.PSPath}
    ```
    > [!TIP]
    > Some troubleshooting tips:
    > 
    > -   If you see **0** in the report, instead of the number of files in your folder, this indicates that the script did not run. First, check the script itself by loading it in Windows PowerShell ISE to validate the script contents and try running it to see if any errors are displayed. With no arguments specified, the script will try to connect and authenticate to Azure RMS.
    > 
    >     -   If the script reports that it couldn't connect to Azure RMS, check the values it displays for the service principal account, which you specified in the script. For more information about how to create this service principal account, see [Prerequisite 3: To protect or unprotect files without interaction](client-admin-guide-powershell.md#prerequisite-3-to-protect-or-unprotect-files-without-user-interaction) from the Azure Information Protection client admin guide.
    >     -   If the script reports that it could connect to Azure RMS, next check that it can find the specified template by running [Get-RMSTemplate](/powershell/azureinformationprotection/vlatest/get-rmstemplate) directly from Windows PowerShell on the server. You should see the template you specified returned in the results.
    > -   If the script by itself runs in Windows PowerShell ISE without errors, try running it as follows from a  PowerShell session, specifying a file name to protect and without the -OwnerEmail parameter:
    > 
    >     ```
    >     powershell.exe -Noprofile -Command "<path>\RMS-Protect-FCI.ps1 -File '<full path and name of a file>' -TemplateID <template GUID>"
    >     ```
    >     -   If the script runs successfully in this Windows PowerShell session, check  your entries for **Executive** and **Argument** in the file management task action.  If you have specified **-OwnerEmail [Source File Owner Email]**, try removing this parameter.
    > 
    >         If the file management task works successfully without  **-OwnerEmail [Source File Owner Email]**, check that the unprotected files have a domain user listed as the file owner, rather than **SYSTEM**.  To do this, use the **Security** tab for the file's properties, and then click **Advanced**. The **Owner** value is displayed immediately after the file **Name**. Also, verify that the file server is in the same domain or a trusted domain to lookup the user's email address from Active Directory Domain Services.
    > -   If you see the correct number of files in the report but the files are not protected, try protecting the files manually by using the [Protect-RMSFile](/powershell/azureinformationprotection/vlatest/protect-rmsfile) cmdlet, to see if any errors are displayed.

When you have confirmed that these tasks run successfully, you can close File Resource Manager. New files will be automatically protected and all files will be protected again when the schedules run. Re-protecting files ensures that any changes to the template are applied to the files.


## Modifying the instructions to selectively protect files
When you have the preceding instructions working, it's then very easy to modify them for a more sophisticated configuration. For example, protect files by using the same script but only for files that contain personal identifiable information, and perhaps select a template that has more restrictive rights.

To do this, use one of the built-in classification properties (for example, **Personally Identifiable Information**) or create your own new property. Then create a new rule that uses this property. For example, you might select the **Content Classifier**, choose the **Personally Identifiable Information** property with a value of **High**, and configure the string or expression pattern that identifies the file to be configured for this property (such as the  string "**Date of Birth**").

Now all you need to do is create a new file management task that uses the same script but perhaps with a different template, and configure the condition for the classification property that you have just configured. For example, instead of the condition that we configured previously (**RMS** property, **Equal**, **Yes**), select the **Personally Identifiable Information** property with the **Operator** value set to **Equal** and the **Value** of **High**.

[!INCLUDE[Commenting house rules](../includes/houserules.md)]
