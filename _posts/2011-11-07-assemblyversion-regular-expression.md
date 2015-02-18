---
title: AssemblyVersion regular expression
categories : .Net
tags : TFS
date: 2011-11-07 11:40:36 +10:00
---

It seems that I keep having to come up with a regular expression that validates and parses a version string. I often use this for customising TFS build workflows for extracting and using application version information.

The AssemblyVersionAttribute appears to have the following rules:

* Major version is required
* Minor version is optional and will default to 0 if not specified
* Minor version cannot be *
* Build and Revision versions are optional and may be *
* Revision number cannot be specified if the build number is *

The following is the regular expression that covers all these rules.

    ^
    # Major version must exist
    (?<VersionPart&gt;\d+)
    (
      # Minor only supports numbers
      \.(?<VersionPart&gt;\d+)
    
      (\.
        (
          # Build can be *
          (?<VersionPart&gt;\*)
          |
          # or a number
          (
            (?<VersionPart&gt;\d+)
    
            # with optional Revision that may be * or numeric
            (\.(?<VersionPart&gt;\*|\d+))?
          )
        )
      )? # Build.Revision is optional
    )?$ # Minor.Build.Revision is optional{% endhighlight %}

The following is the same expression without the comments and whitespace.

    ^(?<VersionPart&gt;\d+)(\.(?<VersionPart&gt;\d+)(\.((?<VersionPart&gt;\*)|((?<VersionPart&gt;\d+)(\.(?<VersionPart&gt;\*|\d+))?)))?)?${% endhighlight %}

Enjoy.

