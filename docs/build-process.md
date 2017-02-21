# Build Process

The source supplied in this SDK includes MAKEFILEs for use with the
Microsoft NMAKE utility. You run the utility from a command prompt. The
client-side virtual driver is not designed to be built using the visual
interface of Microsoft Visual C/C++, or with the visual interface of
Visual Studio. See the Microsoft documentation for details about NMAKE.

You can build both server and client examples for Win32 at one time
using the supplied batch file `src\examples\vc\buildit.bat`.

Compiled object files are placed in the `obj\retail` subfolder of the
platform folder; for example, `win32\obj\retail`.

The components can also be built with debugging information turned on.
The output directory changes to `obj\debug` for debug objects.