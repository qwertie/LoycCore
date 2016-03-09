---
layout: page
title: "Project Status & Task List"
stripes: true
---
Help wanted!

## Loyc .NET core libraries

This project includes Loyc.Essentials, Loyc.Collections, Loyc.Syntax and Loyc.Utilities.

- Status: In fairly good shape
- TODO: more unit tests. Some of the simplest methods may be broken if they've never been called
- TODO: Add more 3D point math
- TODO: Add N-dimensional points and integrate them nicely with 2- and 3-dimensional points
- TODO: Add matrix math. Affine and perspective transforms, matrix inverse, etc.
- TODO: any other important functionality you think .NET framework should have but doesn't

## All projects:

- TODO: Make PCL (Portable Class Library) solution for Loyc Core (and the main Loyc projects like EC# & LeMP)

## Loyc trees: in-memory LISP-inspired syntax trees

- Done: LNode class (Src/Loyc.Syntax/Nodes/LNode.cs) and derived classes

## LES: a C-like representation of Loyc trees as plain text

- Parser: done.
    - TODO: consider supporting additional literal types, e.g. BigInteger
- Printer: 
    - Basic: supports operator notation, braced blocks, and comment trivia
    - TODO: superexpressions, `indexed[expressions]`, `generic!T` notation, tuples
- Syntax highlighting:
    - Visual Studio: COMPLETE.
    - Notepad++ UDL: exists in GitHub repo at 
        Visual Studio Integration\notepad++userDefinedLang_les.xml 
      with a second version for black background: notepad++userDefinedLang_les_dark.xml
      The UDL file cannot fully support LES (e.g. no nested comments) but it works pretty well.

## MiniTestRunner: Unit test and benchmark runner

- Work stopped, may not restart (I burned out trying to figure out how to do sandboxing and appdomains and how to seamlessly update the result tree when reloading a DLL and rerunning tests).
