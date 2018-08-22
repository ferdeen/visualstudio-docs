---
title: "Command Implementation | Microsoft Docs"
ms.custom: ""
ms.date: "2018-06-30"
ms.prod: "visual-studio-dev14"
ms.reviewer: ""
ms.suite: ""
ms.technology: 
  - "vs-ide-sdk"
ms.tgt_pltfrm: ""
ms.topic: "article"
helpviewer_keywords: 
  - "commands, implementation"
ms.assetid: c782175c-cce4-4bd0-8374-4a897ceb1b3d
caps.latest.revision: 25
ms.author: "gregvanl"
manager: "ghogen"
---
# Command Implementation
[!INCLUDE[vs2017banner](../../includes/vs2017banner.md)]

The latest version of this topic can be found at [Command Implementation](https://docs.microsoft.com/visualstudio/extensibility/internals/command-implementation).  
  
To implement a command in a VSPackage, you must perform the following tasks:  
  
1.  In the .vsct file, set up a command group and then add the command to it. For more information, see [Visual Studio Command Table (.Vsct) Files](../../extensibility/internals/visual-studio-command-table-dot-vsct-files.md)'  
  
2.  Register the command with Visual Studio.  
  
3.  Implement the command.  
  
 The following sections explain how to register and implement commands.  
  
## Registering Commands with Visual Studio  
 If your command is to appear on a menu, you must add the <xref:Microsoft.VisualStudio.Shell.ProvideMenuResourceAttribute> to your VSPackage, and use as a value either the name of the menu or its resource ID.  
  
```  
[ProvideMenuResource("Menus.ctmenu", 1)]  
...  
    public sealed class MyPackage : Package  
    {.. ..}  
  
```  
  
 In addition, you must register the command with the <xref:Microsoft.VisualStudio.Shell.OleMenuCommandService>. You can get this service by using the <xref:Microsoft.VisualStudio.Shell.Package.GetService%2A> method if your VSPackage is derived from <xref:Microsoft.VisualStudio.Shell.Package>.  
  
```  
OleMenuCommandService mcs = GetService(typeof(IMenuCommandService)) as OleMenuCommandService;  
if ( null != mcs )  
{  
    // Create the command for the menu item.  
    CommandID menuCommandID = new CommandID(guidCommandGroup, myCommandID);  
    MenuCommand menuItem = new MenuCommand(MenuItemCallback, menuCommandID );  
    mcs.AddCommand( menuItem );  
}  
  
```  
  
## Implementing Commands  
 There are a number of ways to implement commands. If you want a static menu command, which is a command that always appears the same way and on the same menu, create the command by using <xref:System.ComponentModel.Design.MenuCommand> as shown in the examples in the previous section. To create a static command, you must provide an event handler that is responsible for executing the command. Because the command is always enabled and visible, you do not have to provide its status to Visual Studio. If you want to change the status of a command depending on certain conditions, you can create the command as an instance of the <xref:Microsoft.VisualStudio.Shell.OleMenuCommand> class and, in its constructor, provide an event handler to execute the command and a query-status handler to notify Visual Studio when the status of the command changes. You can also implement <xref:Microsoft.VisualStudio.OLE.Interop.IOleCommandTarget> as part of a command class or, you can implement <xref:Microsoft.VisualStudio.Shell.Interop.IVsHierarchy> if you are providing a command as part of a project. The two interfaces and the <xref:Microsoft.VisualStudio.Shell.OleMenuCommand> class all have methods that notify Visual Studio of a change in the status of a command, and other methods that provide the execution of the command.  
  
 When a command is added to the command service, it becomes one of a chain of commands. When you implement the status notification and execution methods for the command, take care to provide only for that particular command and to pass all other cases on to the other commands in the chain. If you fail to pass the command on (usually by returning <xref:Microsoft.VisualStudio.OLE.Interop.Constants>), Visual Studio may stop working properly.  
  
## Query Status Methods  
 If you are implementing either the <xref:Microsoft.VisualStudio.OLE.Interop.IOleCommandTarget.QueryStatus%2A> method or the <xref:Microsoft.VisualStudio.Shell.Interop.IVsUIHierarchy.QueryStatusCommand%2A> method, check for the GUID of the command set to which the command belongs and the ID of the command. Follow these guidelines:  
  
-   If the GUID is not recognized, your implementation of either method must return <xref:Microsoft.VisualStudio.OLE.Interop.Constants>.  
  
-   If your implementation of either method recognizes the GUID but has not actually implemented the command, then the method should return <xref:Microsoft.VisualStudio.OLE.Interop.Constants>.  
  
-   If your implementation of either method recognizes both the GUID and the command, then the method should set the command-flags field of every command (in the `prgCmds` parameter) by using the following flags:  
  
    -   <xref:Microsoft.VisualStudio.OLE.Interop.OLECMDF> if the command is supported.  
  
    -   <xref:Microsoft.VisualStudio.OLE.Interop.OLECMDF> if the command should not be visible.  
  
    -   <xref:Microsoft.VisualStudio.OLE.Interop.OLECMDF> if the command is toggled on and appears to have been checked.  
  
    -   <xref:Microsoft.VisualStudio.OLE.Interop.OLECMDF> if the command is enabled.  
  
    -   <xref:Microsoft.VisualStudio.OLE.Interop.OLECMDF> if the command should be hidden if it appears on a shortcut menu.  
  
    -   <xref:Microsoft.VisualStudio.OLE.Interop.OLECMDF> if the command is a menu controller and is not enabled, but its drop-down menu list is not empty and is still available. (This flag is rarely used.)  
  
-   If the command was defined in the .vsct file with the `TextChanges` flag, set the following parameters:  
  
    -   Set the `rgwz` element of the `pCmdText` parameter to the new text of the command.  
  
    -   Set the `cwActual` element of the `pCmdText` parameter to the size of the command string.  
  
 Also make sure that the current context is not an automation function, unless your command is specifically intended to handle automation functions.  
  
 To indicate that you support a particular command, return <xref:Microsoft.VisualStudio.VSConstants.S_OK>. For all other commands, return <xref:Microsoft.VisualStudio.OLE.Interop.Constants>.  
  
 In the following example, the query-status method first makes sure that the context is not an automation function, then finds the correct command-set GUID and command ID. The command itself is set to be enabled and supported. No other commands are supported.  
  
```  
public int QueryStatus(ref Guid pguidCmdGroup, uint cCmds, OLECMD[] prgCmds, IntPtr pCmdText)  
{  
    if (!VsShellUtilities.IsInAutomationFunction(m_provider.ServiceProvider))  
    {  
        if (pguidCmdGroup == VSConstants.VSStd2K && cCmds > 0)  
        {  
            // make the Right command visible   
            if ((uint)prgCmds[0].cmdID == (uint)VSConstants.VSStd2KCmdID.RIGHT)  
            {  
                prgCmds[0].cmdf = (int)Microsoft.VisualStudio.OLE.Interop.Constants.MSOCMDF_ENABLED | (int)Microsoft.VisualStudio.OLE.Interop.Constants.MSOCMDF_SUPPORTED;  
                return VSConstants.S_OK;  
            }  
        }  
        return Constants.OLECMDERR_E_NOTSUPPORTED;  
    }  
}  
  
```  
  
## Execution Methods  
 Implementation of the execute method resembles implementation of the query-status method. First, make sure that the context is not an automation function. Then test for both the GUID and the command ID. If the GUID or command ID is not recognized, return <xref:Microsoft.VisualStudio.OLE.Interop.Constants>.  
  
 To handle the command, execute it and return <xref:Microsoft.VisualStudio.VSConstants.S_OK> if the execution succeeds. Your command is responsible for error detection and notification; therefore, return an error code if the execution fails. The following example demonstrates how the execution method should be implemented.  
  
```  
public int Exec(ref Guid pguidCmdGroup, uint nCmdID, uint nCmdexecopt, IntPtr pvaIn, IntPtr pvaOut)  
{  
    if (!VsShellUtilities.IsInAutomationFunction(m_provider.ServiceProvider))  
    {  
        if (pguidCmdGroup == VSConstants.GUID_VSStandardCommandSet97)  
        {  
             if (nCmdID ==(uint) uint)VSConstants.VSStd2KCmdID.RIGHT)  
            {  
                //execute the command  
                return VSConstants.S_OK;  
            }  
        }  
    }  
    return Constants.OLECMDERR_E_NOTSUPPORTED;  
}  
  
```  
  
## See Also  
 [How VSPackages Add User Interface Elements](../../extensibility/internals/how-vspackages-add-user-interface-elements.md)
