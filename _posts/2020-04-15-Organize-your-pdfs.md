---
layout: post
title: Organize your PDFs by naming them by the content
---

I happen to download a lot of PDF files. Mostly ACM papers, ebooks, research papers etc. Sometimes they are named properly. But
mostly, they are named by some random names with no correlation to the interal contents whatsoever. As such, this is a pain
after a few weeks or months when you see the file, and cant remember what it was,and you have to open the file to see the contents.

So I wrote this small tool in Java which reads the PDF metadata information, and also the first page of the PDF and extracts the
possible name of the PDF file. It works well with most of the IEEE/ACM papers or files which are published with proper metadata
information, for example when you do it through LaTeX.

[Here](https://github.com/gagan405/pdf_renamer) is the code and the README on how to use it.
 
 
