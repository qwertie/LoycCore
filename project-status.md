---
layout: page
title: "Project Status & Task List"
stripes: true
---
Help wanted!

## Loyc .NET core libraries

This project includes Loyc.Essentials, Loyc.Collections, Loyc.Syntax and Loyc.Utilities.

- Status: In fairly good shape
- TODO: Add more 3D point math
- TODO: Add N-dimensional points and integrate them nicely with 2- and 3-dimensional points
- TODO: Add matrix math. Affine and perspective transforms, matrix inverse, etc.
- TODO: any other important functionality you think .NET framework should have but doesn't

## Loyc trees: in-memory LISP-inspired syntax trees

- Done: LNode class (Src/Loyc.Syntax/Nodes/LNode.cs) and derived classes

## LES: a compact representation of Loyc trees as plain text

- Parser: supports the basics. Line breaks are always ignored, contrary to spec.
    - TODO: Python mode, recognize line breaks
    - TODO: consider supporting additional literal types, e.g. BigInteger
- Printer: 
    - VERY basic, pure prefix notation only
    - TODO: good-looking output
- Syntax highlighting:
    - Visual Studio: supports token highlighting only using actual LES lexer, does not highlight superexpression "keywords".
        - TODO: rewrite the highlighter using LES lexer and parser. I already wrote the `SparseAList<T>` class for the specific purpose of storing token boundaries in the syntax highlighting engine.
    - Notepad++ UDL: exists in GitHub repo at 
        Visual Studio Integration\notepad++userDefinedLang_les.xml 
      with a second version for black background: notepad++userDefinedLang_les_dark.xml
      The UDL file cannot fully support LES (e.g. no nested comments) but it works pretty well.

## MiniTestRunner: Unit test and benchmark runner

- Work stopped, may not restart (I burned out trying to figure out how to do sandboxing and appdomains and how to seamlessly update the result tree when reloading a DLL and rerunning tests).
