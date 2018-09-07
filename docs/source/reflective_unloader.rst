.. _Reflective Unloader:

Reflective Unloader
===================

This is code that can be used within a PE file to allow it to reflectively
reconstruct itself in memory at runtime. The result is a byte for byte copy of
the original PE file. This can be combined with `Reflective DLL Injection`_ to
allow code to reconstruct itself after being loaded through an arbitrary means.

The original PE file will not be modified in memory, this code makes a new copy
of the unloaded target module.

Usage
-----

1. The build environment is Visual Studio 2017.

2. Add the following files to the project:

   - ReflectivePolymorphism.c
   - ReflectivePolymorphism.h
   - ReflectiveUnloader.c
   - ReflectiveUnloader.h

3. Once the necessary files have been added, call ``ReflectiveUnloader()`` with
   a handle to the module to unload and reconstruct.

   -  For an executable this could be ``GetModuleHandle(NULL)``\ :sup:`1`
   -  For a DLL this could be ``hinstDLL`` from ``DllMain``

4. After compiling the project, run ``pe_patch.py`` to patch in the necessary
   shadow section data to the PE file. Without this step, the writable sections
   of the PE file will be corrupted in the unloaded copy. (See
   `below <#visual-studio-build-event>`__ for how to automate this.)

.. _Shadow Section:

Shadow Section
^^^^^^^^^^^^^^

It’s necessary to patch a reflectively unloadable PE file to get a perfect
byte-for-byte copy when it is reconstructed. The patching process creates a
shadow copy of all writable sections in a new section named ``.restore``. This
shadow section is then used when the ``ReflectiveUnloader`` function is called
to restore the original contents of the writable sections. Reflectively
unloadable PE files should be patched once after being compiled.

If the shadow section is not present, the unloader will simply skip this step.
This allows the unloader to perform the same task for arbitrary unpatched PE
files, however **any modifications to segments made at runtime will be present
in the unloaded PE file**.

The shadow section is composed of a null-terminated list of
``IMAGE_SECTION_HEADER`` strcutures. The final, null-termination entry has no
name and is ``sizeof(IMAGE_SECTION_HEADER)`` null bytes. Each section header's
``PointerToRawData`` field is updated to point to the associated data. Following
the list of section headers is the data for each section.

Shadow Section Contents
~~~~~~~~~~~~~~~~~~~~~~~

The following is a diagram illustrating an example layout of the shadow
section's contents which includes a backup of a single section (the ``.data``
section).

+-----------------------------------------+
| IMAGE_SECTION_HEADER (Name: .data\\x00) |
+-----------------------------------------+
| IMAGE_SECTION_HEADER (Name: \\x00)      |
+-----------------------------------------+
| [ .data section contents ]              |
+-----------------------------------------+

Visual Studio Build Event
~~~~~~~~~~~~~~~~~~~~~~~~~

The ``pe_patch.py`` script can be executed automatically for every build using a
build event. Right click the project in Solution Explorer, then navigate to
``Configuration Properties > Build Events > Post Build Event`` and adjust the
settings as follows:

+--------------+---------------------------------------------------------------+
| Setting Name | Setting Value                                                 |
+==============+===============================================================+
| Command Line | ``python $(SolutionDir)pe_patch.py "$(TargetPath)"            |
|              | "$(TargetPath)"``                                             |
+--------------+---------------------------------------------------------------+
| Description  | Patch in the .restore section                                 |
+--------------+---------------------------------------------------------------+
| Use In Build | Yes                                                           |
+--------------+---------------------------------------------------------------+

API Reference
-------------

.. c:function:: PVOID ReflectiveUnloader(HINSTANCE hInstance, PSIZE_T pdwSize)

    Unload the module indicated by hInstance and return a pointer to it's
    location in memory. If this function fails, NULL is returned.

    :param HINSTANCE hInstance: Handle to the module instance to unload from memory.
    :param PSIZE_T pdwSize: The size of the returned PE image.
    :return: A pointer to a blob of the unloaded PE image.
    :rtype: PVOID

.. c:function:: VOID ReflectiveUnloaderFree(PVOID pAddress, SIZE_T dwSize)

    Free memory that was previously allocated by ReflectiveUnloader().

    :param PVOID pAddress: Pointer to the blob returned by ReflectiveUnloader.
    :param SIZE_T dwSize: Size of the blob returned by ReflectiveUnloader.

.. _Reflective DLL Injection: https://github.com/stephenfewer/ReflectiveDLLInjection
