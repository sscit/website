---
layout: post
title:  Trace LSP Communication to Visual Studio Code's Output Console
date:   2021-04-15 11:05:31 +0100
---

It is possible to trace all language server protocol communication to Visual Studio Code's output console, as described in the [developer's guide](https://code.visualstudio.com/api/language-extensions/language-server-extension-guide#logging-support-for-language-server). The explanation in the guide is quite hard to understand, though. In this post, I explain more details about how to configure VSCode's LSP implementation, so that the traces are printed out.

> If you are using vscode-languageclient to implement the client, you can specify a setting [langId].trace.server that instructs the Client to log communications between Language Client / Server to a channel of the Language Client's name.

It took me quite some time to figure out _where_ to specify the setting (aka _configuration option_) mentioned in the user guide. It is possible to define a file `settings.json` within the `.vscode` folder of your **project**. In this context, **project** means the files that are processed by your extension. In case of REL, a **project** is a set of requirements specification and requirements data files. The configuration option is not specified within your extension development folder. `settings.json` then contains configuration options that are applied to the running instance of VSCode (and its extensions), that opens the folder. 

If your LSP client extension is based on `vscode-languageclient`, you can define the setting `"ClientId.trace.server": "verbose"` within this file. The `ClientId` is defined within the `extension.js` file, as part of the properties to create a `LanguageClient` object (For an example, have a look at [REL's VSCode Extension](https://github.com/sscit/rel/blob/main/vscode-ext/extension.js#L28) )

The resulting `settings.json` then looks like this (assuming no other options are set)

```
{
    "RELLanguageClient.trace.server": "verbose",
}
```

With the configuration option set to `verbose`, as soon as the folder is opened in Visual Studio Code, all JSON messages to and from the language server are printed into the output console of Visual Studio Code, in a separate channel called `RELLanguageClient`.