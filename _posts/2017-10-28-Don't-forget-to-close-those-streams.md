I'm working on a small side project in .NET Core that creates an EPUB e-book out of some HTML source material. In addition to the source files, you need to write [some metadata files](https://www.thoughtco.com/create-epub-file-from-html-and-xml-3467282) to complete the book, and this is where I was running into some problems.

Since an EPUB file is really just a ZIP archive of the folder with the source material, I figured I would keep things simple and use

```c#
ZipFile.CreateFromDirectory(sourceDirectory, output);
```

but I kept getting "File still in use" exceptions about one of the metadata files I had just written, even though at first glance it looked like the writer had been closed.

What led me down the wrong path for a little while is that apparently `ZipFile.CreateFromDirectory()` is quite buggy, and will often complain if you try to make a ZIP archive in the same parent directory as the target directory you're trying to compress (among other issues). But that turned out to be a red herring in this case, which is an important lesson: seeing that my own code was fine at first glance, when I saw that this library function is considered by others to be unreliable, I latched on to that assuming it had to be the cause.

But no, it was entirely my own fault. I was writing my metadata files like this:

```c#
string calibreTocFile = Path.Combine(outputDirectory, "toc.ncx");
using (XmlWriter writer = new XmlWriter(File.OpenWrite(calibreTocFile), writerSettings)) {
    WriteCalibreTocFile(chapters, writer);
}
```

assuming that wrapping the XmlWriter in a `using{}` block was enough to be sure the file handle was being released properly. Not so! The stream returned by `File.OpenWrite()` also needs to be cleaned up, so the correct code is

```c#
string calibreTocFile = Path.Combine(outputDirectory, "toc.ncx");
using (FileStream fileStream = File.OpenWrite(calibreTocFile));
using (XmlWriter writer = new XmlWriter(fileStream, writerSettings)) {
    WriteCalibreTocFile(chapters, writer);
}
```

and the problem vanished. This amounted to only about twenty minutes of puzzlement, but it's an important reminder that managed languages don't free you from every concern about resource cleanup. What's especially bad here is that the original code runs just fine, and without the call immediately afterward to create the ZIP archive, this could have been a resource leak that went undetected. 