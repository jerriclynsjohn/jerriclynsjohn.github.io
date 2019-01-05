---
title: Discussion and implementation of a plugin using Roslyn
date: 2014-08-13 00:00:00 +05:30
categories:
- Revolt
- Technology
- Microsoft
- Roslyn
tags:
- Roslyn
- Visual Studio 2013
- VSIX
- Plugin
layout: post
description: I had a discussion about the performance and implementation details of
  a plugin using Roslyn in CodePlex and StackOverflow. I wanted to share this same
  to everyone.
modified: 2014-08-13 05:30:00 +05:30
image:
  feature: Revolt/Default.png
  credit: JLJ
  creditlink: http://www.swiftjuggler.com
comments: true
share: true
---

I had a discussion about the performance and implementation details of a plugin using Roslyn in [CodePlex](https://roslyn.codeplex.com/discussions/549445) and [StackOverflow](http://stackoverflow.com/questions/25262618/can-multiple-vsix-with-vb-c-diagnostic-analyzer-codefix-autoupdate-cause-perfo). I wanted to share this to everyone.

#To summarize the discussion was as follows

###Requirements

**Coexistence**: VSPackage extension (*to extend shell command component of Visual Studio*) and Managed Extensibility Framework/MEF extensions (*to customize and extend the editor to include Roslyn DiagnosticAnalyzer/CodeFix*), should coexist either in

- Single VSIX
- Maximum 2 VSIXs

**Performance**: The coexistence shouldn't affect the performance and AutoUpdate taken care by the VSPackage extension should not create redundant service calls.


###Caveats:

*As [mentioned][1] by [Srivatsn][2]*

1. Extensions that refer to both `Microsoft.CodeAnalysis.CSharp` and `Microsoft.CodeAnalysis.VisualBasic` will

  - Load both the compilers even if we try to open a C# project, this is not ideal.

2. If we have to analyze the symbols ISymbolAnalyzer,where you are analyzing just the symbols and not the syntax nodes, then we should adopt a single language-agnostic analyzer. This means we don't have to refer any C#/VB dlls (*Even Microsoft is thinking about implementing more language-agnostic analyzer*). Include two export attributes - one for each language, these attributes tell VS to instantiate and call these analyzers when the respective language is contained in the solution.

3. Compilation as a process leaves the memory after the compilation is done, but since there is a compilation happening at almost every keystroke and if the analyzer refers to both c# and VB, it will bring both compilers into memory. Since there is a persistence characteristic, it could be a problem if there is a large project under the solution (*This is my typical production scenario*)

4. There is a confusion whether the compiler is loaded when the respective syntax method is invoked or on instantiation of the exported analyzer (*which is again being filter through the MEF export attribute by mentioning the respective language use case*) since he also mentioned that the if a method that refers to both kind of syntax node might make the JIT compile and load the dlls.

5. Any analyzers linked to menu command would be VS specific and if they are linked to the project then it will participate in the build as well, even outside of the VS through MSBuild

6. VSIX should be able to export multiple components for extending both of those extension points.

*As [mentioned][3] by [VSadov][4]*

1. Persistence of the syntax tree data-structure and the need to re-do analysis at every keystroke(*delta-compilation: this is what Srivatsn's compilation means*) made them design the red-green tree method which helps in the performance of the delta-compilation.

*As [mentioned][5] by [SLaks][6]*

1. MEF exports doesn't make any difference whether they are packaged in a single VSIX or not (*but it should be noted that there is a performance issue related to combining both type analyzers into a single assembly which is an MEF export*)

*As [mentioned][7] by [Kevin Pilch][7]*

1. Although it doesn't matter where these assemblies are packaged in unless they are separate in concern when it comes to language specific references.
Virtual memory will be reserved if the analyzer references both the C# and VB specific Roslyn assemblies and these compiler assemblies are large

2. The performance problems being Disk loading and JIT costs (*I'm not sure how there is a JIT cost if there is no compilation and only reference in it*), but since there is an address space reserved there could be an issue in VS (*I'm not sure how that will be an issue.*)

3. What Microsoft does, according to him, is to create three projects to deal with this (*According to Srivatsn Microsoft is still trying for language-agnostic analyzers*)

  - Shared (No language specific binaries in it)
  - C# specific (+ shared libraries)
  - VB Specific (+ shared libraries)

4. If no language specific binaries are referred and if the MEF exports are appropriately attributed with ContentType or LanguageName then the above issue can be solved

5. We can bundle additional assemblies into a single VSIX (by embedding the other project in it) and VS will load each independently

##Final Implementation:

So Finally I came to a conclusion after discussion with my team as follows

 - A single VSIX implementation by embedding the following projects in it
   - **Update plugin**
     - Checks if update was present in the past 7 days
     - Then checks for the version number of the Plugin from server side via a JSON request
     - Then downloads the plugin from the server, stores the download date in VS settings for initial check
     - Disables the previous plugin
     - Uninstalls the previous plugin
     - Installs the new plugin
     - This functionality is triggered when:
       - the VS loads
       - Manual menu command (which should override the download date check)
   - **C# plugin**
     - Implements and refers only rules for C#
   - **VB Plugin**
     - Implements and refers only rules for VB


[1]: https://roslyn.codeplex.com/discussions/549445
[2]: https://www.codeplex.com/site/users/view/srivatsn
[3]: http://blogs.msdn.com/b/ericlippert/archive/2012/06/08/persistence-facades-and-roslyn-s-red-green-trees.aspx
[4]: https://www.codeplex.com/site/users/view/VSadov
[5]: http://stackoverflow.com/questions/25262618/can-multiple-vsix-with-vb-c-diagnostic-analyzer-codefix-autoupdate-cause-perfo/25270233#comment39362585_25262618
[6]: http://stackoverflow.com/users/34397/slaks
[7]: http://stackoverflow.com/questions/25262618/can-multiple-vsix-with-vb-c-diagnostic-analyzer-codefix-autoupdate-cause-perfo#comment39383014_25270233
[8]: http://stackoverflow.com/users/678625/kevin-pilch-bisson
