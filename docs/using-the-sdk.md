# Using the Virtual Channel SDK

Before using the Virtual Channel SDK, install the WFAPI SDK. You must have administrator privileges.

1.  Set up the compiler environment for each of the target platforms you build. The MAKEFILEs use variables to find the compiler and its files. Normally, installing the compiler sets these variables as environment variables; for example, `vsvars32.bat`. If not, you can set the environment variables or set the defaults in the MAKEFILE `src\examples\build\user.mak`.

2.  From the Start menu, choose Run, and type `c:\path\vcsdk`,
    where `c:\path` is the path to the Virtual Channel SDK `.zip`
    extracted package.

3.  Log off and log on or set the `CLNTROOT` environment variable to the directory where the SDK resides (for example, `c:\path\vcsdk`).
