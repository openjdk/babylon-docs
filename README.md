
Welcome to babylon-docs!
========================

This repository is for design notes, presentations, guides, FAQs, and
other collateral surrounding [OpenJDK Project
Babylon](https://openjdk.org/projects/babylon).

Most documents here are written in (pandoc) Markdown.  Changes made to
Markdown sources are automatically formatted and pushed to the [Babylon
project area](https://openjdk.org/projects/babylon) on the OpenJDK web
site.

See https://openjdk.org/ for more information about the OpenJDK
Community and the JDK.


## How to preview babylon-docs in your browser?

To preview your changes, you can use the 
[OpenJDK Web Page Generator](https://github.com/mbreinhold/ojweb-generate.git).

Instructions:

```bash
cd babylon-docs/site/
git clone https://github.com/mbreinhold/ojweb-generate.git
echo 'include ojweb-generate/Makefile' > Makefile

make 
make preview
```

Then, visit `localhost:8081` in your browser. 

For more information about configuration and installation, visit the 
[project's repo](https://github.com/mbreinhold/ojweb-generate).
