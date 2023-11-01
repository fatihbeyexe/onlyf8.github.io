---
title: Malicious Document Analysis Part-1 (Microsoft Word Documents)
language: EN
published: true
category: malware
---

<h1 style="text-align:center"> What are Malicious Documents? </h1>

In cyber attacks, there are multiple stages to an attack, and these stages have specific objectives. In the concept of the Cyber Kill Chain, it is known that attackers use different tools and technologies at different stages to carry out a successful attack. In this article, we will mainly focus on how malicious documents are detected and analyzed, especially at the beginning of the **"Exploit"** stage of attacks.

To determine our focus and goals, we need to be familiar with the techniques and languages used in malicious documents. When looking at **Microsoft Office** documents they may contain, shellcode, VBA macros and embedded files. **PDF** files, on the other hand, can include JavaScript, FlashScript, ActionScript, and embedded files. **XPS** file types often involve downloading files from malicious links through URI redirection, although the use of XPS files is quite rare. In addition to these techniques, new methods can emerge over time. In any malware analysis on a file type, the possibility of a 0-Day situation should be taken into account. For example, a previously unseen malicious code block may be present within Microsoft Office documents. The examples mentioned above are not always definitive, see Follina Zero Day.

The basic steps of document analysis are as follows:

```
1. Identify the embedded code in the file (shellcode, script, executable).

2. Extract the suspicious code from the file.

3. Perform deobfuscation/decoding (if necessary) to make the code interpretable.

4. If it's not an open-source language, analyze the code by disassembling and debugging.

5. Determine the next stage of the attack.

```
Knowing the techniques mentioned above guides us in detecting malicious documents. For instance, when we have a suspicious file at hand, the points we will examine include whether these techniques exist within the file. In this process, we will use various tools to assist us.

<h1 style="text-align:center">Microsoft Office Documents </h1>

Between 97-2003, Microsoft Office documents were stored in the OLE2 format. Starting from the 2003 version and onwards, Microsoft Office transitioned to the XML format. This structural difference is visibly apparent. Although the analysis stages are not significantly different, the tools used may vary in some aspects.

<img title="Difference" src="difference.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

In documents stored in the OLE2 format, macro files are found under the **Macros** directory, while in documents in the XML format, they are located under the **word** directory. However, macro files in both formats have the **.bin** extension and share the same structure. Therefore, you can directly extract the document file using the **7zip** tool, extract the macro file, and make it readable using tools like **OfficeMalScanner** or **oletools**. In terms of the analysis process, the subsequent steps remain the same.

1. Identify the embedded code in the file (shellcode, script, executable).

<img title="Doc File Content" src="doc-01.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

When examining the document, it typically employs a classic technique of making non-secret content appear secret and requesting the user to enable macros. Based on this, we can assume that there are macros within the file. By extracting the file using tools like 7-zip and examining its contents, we can often find the macro code inside a file named **vbaProject.bin** (typically) or with a different name but a **.bin** extension.

2. Extract the suspicious code from the file.

<img title="Extracting" src="doc-02.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

<img title="Folders" src="doc-03.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

3. Perform deobfuscation/decode (if necessary) to make the code interpretable.

<img title="vbaProject.bin" src="macro-00.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

The **vbaProject.bin** file is not in a readable format. To make this file readable, you can use the **olevba** or **OfficeMalScanner** tool. When you use the tool, you will obtain results like the following:

<img title="OfficeMalScanner" src="doc-04.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

<img title="Obfuscated Macro" src="macro-01.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

Now that we have a readable VBA macro code, we need to analyze the heavily obfuscated macro code step by step.

4. We will skip the "Disassemble and debug the code if it's not an open-source language" stage because we have a script in our hands.

5. Determine the next stage of the attack.

First, we make the names of created variables and functions readable. These randomly generated names can make it challenging to read. When examining the functions defined in the macro, we will focus on the **Document_Open** function. Since this function runs when the document is opened, we will perform step-by-step static analysis to understand what the code is attempting to do.

<img title="Obfuscated Macro" src="macro-02.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

The program, when following the flowchart, leads us to the function named **"zwxcnxajcshyp"**. In this function, there are 5 hardcoded values of unknown purpose, followed by a **Create** operation. Within the function, it is noticeable that **function6** is called multiple times, and the hardcoded values are passed as parameters to this function. It appears to be a **decode/decrypt** function, and we will analyze it accordingly.

<img title="Obfuscated Macro" src="macro-03.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

When looking at the **asciiTable** function (changed to that name) called from within **function6**, it is observed that it creates a dictionary-type list and within this list, a new table similar to an ASCII table is constructed.

<img title="Created ASCII Table" src="ascii-table.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

The reason for reconstructing the ASCII table in this manner is as follows: sandbox-like products automatically convert hardcoded hex/decimal values found in files into different types and search for suspicious strings within them. If the attacker were to use the default ASCII table here, some sandboxes might trigger suspicious/malicious alarms for the file. This is why a new ASCII table is created in this way.

Subsequently, using the numerical values from this newly created ASCII table, the hardcoded values of **variable1, variable2, variable3, variable4, and variable5** are broken down into 3-digit segments and decoded into their string form. Here, by writing a Python script, you can obtain the clear-text version of this data.

<img title="Python Script" src="dict_to_string.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

When decoded, the following results have been obtained:
<img title="Obfuscated Macro" src="decoded_strings.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

<img title="Obfuscated Macro" src="macro-05.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

When examining **Function5**, it is understood that it creates a file in the specified file name and directory provided in the **param1** variable.

<img title="Obfuscated Macro" src="macro-06.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

When looking at **Function9**, it is noticed that a random file name is generated to create a DLL file to make it difficult to extract IOC values easily.

In the image where we obtained the decoded values, when looking at the row enclosed in green, the value of the variable with the name **http://localhost/** inside the **XML** files in the **customXML** directory is retrieved using the **CustomXMLParts** method, and this value is taken as Text. When examining this value, it is determined that it contains an executable file with its byte order reversed.

<img title="XML Content" src="macro-04.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

The relevant data can be converted into a file using the **CyberChef** tool. The recipe used for this purpose is as follows:

<img title="CyberChef" src="cyberChef.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

Upon downloading the obtained file, it is determined to be a 64-bit structured DLL.

<img title="SecondStage" src="secondStage.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

A VirusTotal query reveals that it is a **Downloader** belonging to the **IcedID** malware family.

<img title="Virus Total" src="virusTotal.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

Thanks to Mehmet DEMIR for his contribution to the preparation of this blog post.

---

For your criticism, corrections, suggestions, and questions, please reach out to me through my contact addresses. Your feedback is valuable to me :)