==========
Signatures
==========

With Cuckoo you're able to create some customized signatures that you can run against
the analysis results in order to identify some predefined pattern that might
represent a particular malicious behavior or an indicator you're interested in.

These signatures are very useful to give a context to the analyses: both because they
simplify the interpretation of the results as well as for automatically identifying
malwares of interest.

Some examples you can use Cuckoo's signatures for:
    * Identify a particular malware family you're interested in by isolating some unique behaviors (like file names or mutexes).
    * Spot interesting modifications the malware performs on the system, such as installation of device drivers.
    * Identify particular malware categories, such as Banking Trojans or Ransomware by isolating typical actions commonly performed by those.

You can find signatures created by us and by other Cuckoo users on our `Community`_ repository.

.. _`Community`: https://github.com/cuckoobox/community

Getting started
===============

Creation of signatures is a very simple process and requires just a decent
understanding of Python programming.

First thing first, all signatures are and should be located inside *modules/signatures/*.

A basic example signature is the following:

    .. code-block:: python
        :linenos:

        from lib.cuckoo.common.abstracts import Signature

        class CreatesExe(Signature):
            name = "creates_exe"
            description = "Creates a Windows executable on the filesystem"
            severity = 2
            categories = ["generic"]
            authors = ["Cuckoo Developers"]
            minimum = "0.5"

            def run(self):
                return self.check_file(pattern=".*\\.exe$",
                                       regex=True)

As you can see the structure is really simple and consistent with the other
modules. We're going to get into details later, but as you can see at line **12**
from version 0.5 Cuckoo provides some helper functions that make the process of
creating signatures much easier.

In this example we just walk through all the accessed files in the summary and check
if there is anything ending with "*.exe*": in that case it will return ``True``, meaning that
the signature matched, otherwise return ``False``.

In case the signature gets matched, a new entry in the "signatures" section will be added to
the global container like following::

    "signatures": [
        {
            "severity": 2, 
            "description": "Creates a Windows executable on the filesystem", 
            "alert": false, 
            "references": [], 
            "data": [
                {
                    "file_name": "C:\\d.exe"
                }
            ], 
            "name": "creates_exe"
        }
    ]

We could rewrite the exact same signature by accessing the **global container**
directly:

    .. code-block:: python
        :linenos:

        from lib.cuckoo.common.abstracts import Signature

        class CreatesExe(Signature):
            name = "creates_exe"
            description = "Creates a Windows executable on the filesystem"
            severity = 2
            categories = ["generic"]
            authors = ["Cuckoo Developers"]
            minimum = "0.5"

            def run(self):
                for file_path in self.results["behavior"]["summary"]["files"]:
                    if file_path.endswith(".exe"):
                        return True

                return False

This obviously requires you to know the structure of the **global container**,
which you can observe represented in the JSON report of your analyses.

Creating your new signature
===========================

In order to make you better understand the process of creating a signature, we
are going to create a very simple one together and walk through the steps and
the available options. For this purpose, we're going to simply create a signature that checks whether
the malware analyzed opened a mutex named "i_am_a_malware".

The first thing to do is import the dependencies, create a skeleton and define
some initial attributes. These are the ones you can currently set:

    * ``name``: an identifier for the signature.
    * ``description``: a brief description of what the signature represents.
    * ``severity``: a number identifying the severity of the events matched (generally between 1 and 3).
    * ``categories``: a list of categories that describe the type of event being matched (for example "*banker*", "*injection*" or "*anti-vm*").
    * ``families``: a list of malware family names, in case the signature specifically matches a known one.
    * ``authors``: a list of people who authored the signature.
    * ``references``: a list of references (URLs) to give context to the signature.
    * ``enable``: if set to False the signature will be skipped.
    * ``alert``: if set to True can be used to specify that the signature should be reported (perhaps by a dedicated reporting module).
    * ``minimum``: the minimum required version of Cuckoo to successfully run this signature.
    * ``maximum``: the maximum required version of Cuckoo to successfully run this signature.

In our example, we would create the following skeleton:

    .. code-block:: python
        :linenos:

        from lib.cuckoo.common.abstracts import Signature

        class BadBadMalware(Signature): # We initialize the class inheriting Signature.
            name = "badbadmalware" # We define the name of the signature
            description = "Creates a mutex known to be associated with Win32.BadBadMalware" # We provide a description
            severity = 3 # We set the severity to maximum
            categories = ["trojan"] # We add a category
            families = ["badbadmalware"] # We add the name of our fictional malware family
            authors = ["Me"] # We specify the author
            minimum = "0.5" # We specify that in order to run the signature, the user will need at least Cuckoo 0.5

        def run(self):
            return

This is a perfectly valid signature. It doesn't really do anything as of yet,
now we need to define the conditions for the signature to be matched.

As we said, we want to match a pecurial mutex name, so we proceed as follows:

    .. code-block:: python
        :linenos:

        from lib.cuckoo.common.abstracts import Signature

        class BadBadMalware(Signature):
            name = "badbadmalware"
            description = "Creates a mutex known to be associated with Win32.BadBadMalware"
            severity = 3
            categories = ["trojan"]
            families = ["badbadmalware"]
            authors = ["Me"]
            minimum = "0.5"

        def run(self):
            return self.check_mutex("i_am_a_malware")

Simple as that, now our signature will return ``True`` whether the analyzed
malware was observed opening the specified mutex.

If you want to be more explicit and directly access the global container,
you could translate the previous signature in the following:

    .. code-block:: python
        :linenos:

        from lib.cuckoo.common.abstracts import Signature

        class BadBadMalware(Signature):
            name = "badbadmalware"
            description = "Creates a mutex known to be associated with Win32.BadBadMalware"
            severity = 3
            categories = ["trojan"]
            families = ["badbadmalware"]
            authors = ["Me"]
            minimum = "0.5"

        def run(self):
            for mutex in self.results["behavior"]["summary"]["mutexes"]:
                if mutex == "i_am_a_malware":
                    return True

            return False

Helpers
=======

As anticipated, from version 0.5 the ``Signature`` base class also provides
some helper methods that simplify the creation of signatures and avoid you
from directly accessing the global container (at least most of the times).

Following is a list of available methods.

.. function:: Signature.check_file(pattern[, regex=False])

    Checks whether the malware opened or created a file matching the specified pattern. Returns ``True`` in case it did, otherwise returns ``False``.

    :param pattern: file name or file path pattern to be matched
    :type pattern: string
    :param regex: enable to compile the pattern as a regular expression
    :type regex: boolean
    :rtype: boolean

    Example Usage:

    .. code-block:: python
        :linenos:

        self.check_file(pattern=".*\.exe$", regex=True)

.. function:: Signature.check_key(pattern[, regex=False])

    Checks whether the malware opened or created a registry key matching the specified pattern. Returns ``True`` in case it did, otherwise returns ``False``.

    :param pattern: registry key pattern to be matched
    :type pattern: string
    :param regex: enable to compile the pattern as a regular expression
    :type regex: boolean
    :rtype: boolean

    Example Usage:

    .. code-block:: python
        :linenos:

        self.check_key(pattern=".*CurrentVersion\\Run$", regex=True)

.. function:: Signature.check_mutex(pattern[, regex=False])

    Checks whether the malware opened or created a mutex matching the specified pattern. Returns ``True`` in case it did, otherwise returns ``False``.

    :param pattern: mutex pattern to be matched
    :type pattern: string
    :param regex: enable to compile the pattern as a regular expression
    :type regex: boolean
    :rtype: boolean

    Example Usage:

    .. code-block:: python
        :linenos:

        self.check_mutex("mutex_name")

.. function:: Signature.check_api(pattern[, process=None[, regex=False]])

    Checks whether Windows function was invoked. Returns ``True`` in case it was, otherwise returns ``False``.

    :param pattern: function name pattern to be matched
    :type pattern: string
    :param process: name of the process performing the call
    :type process: string
    :param regex: enable to compile the pattern as a regular expression
    :type regex: boolean
    :rtype: boolean

    Example Usage:

    .. code-block:: python
        :linenos:

        self.check_api(pattern="URLDownloadToFileW", process="AcroRd32.exe")

.. function:: Signature.check_argument(pattern[, name=Name[, api=None[, category=None[, process=None[, regex=False]]]])

    Checks whether the malware invoked a function with a specific argument value. Returns ``True`` in case it did, otherwise returns ``False``.

    :param pattern: argument value pattern to be matched
    :type pattern: string
    :param name: name of the argument to be matched
    :type name: string
    :param api: name of the Windows function associated with the argument value
    :type api: string
    :param category: name of the category of the function to be matched
    :type category: string
    :param process: name of the process performing the associated call
    :type process: string
    :param regex: enable to compile the pattern as a regular expression
    :type regex: boolean
    :rtype: boolean

    Example Usage:

    .. code-block:: python
        :linenos:

        self.check_argument(pattern=".*cuckoo.*", category="filesystem", regex=True)

.. function:: Signature.check_ip(pattern[, regex=False])

    Checks whether the malware contacted the specified IP address. Returns ``True`` in case it did, otherwise returns ``False``.

    :param pattern: IP address to be matched
    :type pattern: string
    :param regex: enable to compile the pattern as a regular expression
    :type regex: boolean
    :rtype: boolean

    Example Usage:

    .. code-block:: python
        :linenos:

        self.check_ip("123.123.123.123")

.. function:: Signature.check_domain(pattern[, regex=False])

    Checks whether the malware contacted the specified domain. Returns ``True`` in case it did, otherwise returns ``False``.

    :param pattern: domain name to be matched
    :type pattern: string
    :param regex: enable to compile the pattern as a regular expression
    :type regex: boolean
    :rtype: boolean

    Example Usage:

    .. code-block:: python
        :linenos:

        self.check_domain(pattern=".*cuckoosandbox.org$", regex=True)

.. function:: Signature.check_url(pattern[, regex=False])

    Checks whether the malware performed an HTTP request to the specified URL. Returns ``True`` in case it did, otherwise returns ``False``.

    :param pattern: URL pattern to be matched
    :type pattern: string
    :param regex: enable to compile the pattern as a regular expression
    :type regex: boolean
    :rtype: boolean

    Example Usage:

    .. code-block:: python
        :linenos:

        self.check_url(pattern="^.+\/load\.php\?file=[0-9a-zA-Z]+$", regex=True)
