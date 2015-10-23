
*This documentation is embedded from Depsy's GitHub Readme, and is still in progress. Feel free to submit a pull request with updates and changes*


##Depsy is a better way to learn the impact of research software.##
It's being built at [Impactstory](https://impactstory.org/about) by Jason Priem and Heather Piwowar, and is funded by an [EAGER grant](http://blog.impactstory.org/impactstory-awarded-300k-nsf-grant/) from the National Science Foundation.

## What does Depsy include?

Depsy finds the impact of research software **packages** written in Python and R, and also impact of the **people** who write it.

[PyPI](http://pypi.python.org) for Python, and [CRAN](https://cran.r-project.org/web/packages/) for R.

R is a language for doing statistics.  As such, almost all of its packages are written by academics, for academics, to do reasearch--so we consider all R software on Depsy to be research software. We cover the *7,057* R software packages available on [CRAN](https://cran.r-project.org/web/packages/), the main library repository for R.

Python is a more general purpose programming language.  We try to establish whether a given Python package is research software by searching its metadata (package name, description, tags) for researchy-words (see [code on GitHub](https://github.com/Impactstory/depsy/blob/870c85ee4598643f496bca76e5a7dff994e53837/models/academic.py)). We cover all *57,243* active Python packages on [PyPI](http://pypi.python.org), Python's main package repository; of those, we count *4,166* as research software.

## Package impact

We currently calculate impact along three dimensions (and a fourth, coming later, that will capture social signals like GitHub watchers and Stack Overflow discussions). The overall impact score of a package is calculated as the average of these three: **downloads**, **software reuse**, and **literature mentions**. 

### Package impact: downloads

Gathering download statistics is easy.  We gathered monthly download numbers for python packages from Pypi (ie [numpy](https://pypi.python.org/pypi/numpy) ), and from the [CRAN logs](http://cranlogs.r-pkg.org/) api for R (ie [knitr](http://cranlogs.r-pkg.org/downloads/daily/1900-01-01:2020-01-01/knitr)).  We took a snaphost of these numbers in September 2015.


### Package impact: software Reuse

Quantifying software reuse isn't quite as easy, but it's really cool and important :)

The goal is a number that measures how much a library is depended on by other code -- a number that goes up when a library is depended on by more projects, when it is a dominant dependency the projects, and when it is depended on by projects that are itselves a big deal.  

Luckily a metric that captures these measures of importance in a network has already been invented: PageRank.  PageRank was Google's first metric for assessing the importance of a web page.  It's a great fit for measuring the importance of a software library in the software-dependency network.

How do we apply it to software?  There are three steps, which are covered in detail below:

1. Identify dependencies
2. Calculate PageRank
3. Display PageRank in a way that is interpretable

#### Identify dependencies

We'd like to find absolutely all of the code that uses a research software library.  Alas this is impossible -- most software lives only on private harddrives and servers, so we don't know that it exists, let alone that it depends on a given software library.

That said, there's great news: more and more code is becoming Open Source, available for anyone to browse and search.  GitHub is by far the largest such repository, and growing quickly (TODO: XXX cite).  Finding dependencies on GitHub serves as a proxy for all dependencies. 

Luckily the dependency information for software libraries is also available and searchable, so we are able to add to our sample all of the times that libraries in the PyPI and CRAN networks depend on other libraries.

Our approach, then, is to find library dependencies within GitHub, PyPI, and CRAN.  The specifics of how we do this differ by language:


##### Finding dependencies on R libraries 
(TODO:  way too many levels of heading, fix)

CRAN makes it super easy to know which libraries depend on which other libraries.  Yay :)  The CRAN website lists reverse depencencies for each library (see (the CRAN page for knitr)[https://cran.r-project.org/web/packages/knitr/] for what this looks like). We include the "Reverse depends" and "Reverse imports" fields, but not the "Reverse enhances" or "Reverse suggests" fields.

We also wanted to find all the times that software in GitHub depends on an R library.  We started by getting a list of all R GitHub repositories, using Google BigQuery (TODO: XXXX link).  For each GitHub repository we downloaded its source code, extracted lines that included the words "library" or "requires" from files that end in R and R/Sweave/knitr filename extensions, discarded comments, and then used regular expressions to extract the library name.

##### Finding dependencies on Python libraries
We want to know which Python libraries require which other Python libraries. This is more complicated to learn from PyPI than it was from CRAN, largely because the way to specify dependencies in a Python library distribution has changed over time, and we needed to find all the dependencies, no matter how they were specified.  So, to start with, we downloaded the source distribution of each PyPI library using its most recent release url listed on PyPI.  For the recently introduced "wheel" format", we extracted the list of required libraries from the package's metadata.json file; for other source distributions we extracted the list from the older requires.txt file.  When none of these files existed we parsed the package's setup.py file (TODO: jason or heather confirm we did use setup.py into the final version).  

In all of these cases we recorded all libraries that were required with the base install, excluding ones that were used only by certain environments (Windows pywin for example) or auxilliary install options (like development or test installations).

Finally, we wanted to find all the times that sofware in GitHub depends on a Python library.  This starts off simple:  we got a list of all Python GitHub repositories, using Google BigQuery (TODO: XXXX link).  For each GitHub repository we downloaded its source code.  From there, we looked for dependencies in two places to make sure we didn't miss anything.  First, if the software repository included a setup.py or requirements.txt file, we recorded the libraries listed therein.  Second, we looked within the python source code itself, extracting lines that included the words "import" from files that end in .py, discarded comments, and then used regular expressions to extract the import name.  We compared the import name to the source code directory tree.  If the import name was a subdirectory or a filename we assumed they were importing a local module, so we discarded the import.   

The final task was mapping the import name to a PyPI library.  This was more complicated than you may expect -- certainly more complicated than we expected!  The approach we finalized on was to first run through all Python package on PyPI and extract the package or module name that each package specifies in its setup.py.  Sometimes the setup.py runs code to generate the value, in which case we ccouldn't extract it from the static setup.py file -- in these cases we scraped the highest level directory from http://pydoc.net.  See for example (TODO: insert example).  We built a lookup from these import names back to the PyPI library names.  

Unfortunately, in some cases more than one PyPI library that uses the same import name (TODO: link to the ref that discusses this).  We assign all dependencies to the PyPI library with the most downloads.  A limitation of the current approach is that we therefore do not identify GitHub-based imports for PyPI libraries with ambiguous import names that do not have the most downloads (TODO: how many libraries is this).  We do not calculate PageRank for libraries, but rather estimate their PageRank percentile from their download percentile  (TODO: mention if this is displayed in the UI when it is).

#### Calculate PageRank

We load the dependencies into Python's (igraph)[http://depsy.org/package/python/python-igraph] library as a directed network, then calculate PageRank using the personal_pagerank method.  (You can play with it too -- just export the XXXX tables from our follower database, and run the code LLLLL here to import it into igraph.)

#### Display PageRank in a way that is interpretable

The last step is to give the PageRank measure some context -- PageRank numbers tend to be 1.3*10^-5, and 1.7*10^-6.  How can we interpret those.  Are those both high?  Low?  How do they compare to each other and other research software libraries?

Depsy makes the PageRank value grokkable by transforming it into a number between 0 and 1000, and then also providing its percentile compared to other libraries.  We do this by dividing the PageRank score by the maximum PageRank score in its network (so Python libraries are compared to Python libraries, and R to R), taking the log10 transform, adding an offset so all numbers are positive, and then multiplying it by the scaling factor that zooms all pagerank values into the range 0 to 1000.  A library with a PageRank Score of 0 isn't depended on by any code, whereas a PageRank Score of 1000 is depended on heavily by a lot of projects, including a lot of important projects.


### Package impact: citations

We get software literature mentions by searching the full text of the literature hosted in two places: Europe PMC and ADS (which includes arXiv and other things).  Because these sources have different abilities, we search differently in both places.

#### Europe PMC

We search for the name of the library, and the name of the library ANDed with "python" or "r statistical".  If the search has at least 20% of hits with the programming-language specific filter, we consider it searchable and count all of the mentions as valid citations.  If the search does not have an adequate proportion of hits with the language filter we consider it too generic a word for sufficient recall on its own, and instead just look up its PyPI URL and GitHub url (if known).  

#### ADS

We found ADS has poor precision when looking up software names, because it ignores puncutation.  This results in many incorrect matches, where it finds software names inside equations, for example.  To handle this we've sacrificed recall for precision, making the queries much more specific -- so for the r package landlab, basically  ("landlab library" OR "landlab open-source") etc.  This still results in many incorrect hits for common-word package names, so we use the ratio approach described above for PMC to identify and exclude these.  The fallback for common word package names for ADS is simply to return 0 hits because the URL searches we use for PMC are not precise enough.

#### The future
We've got big plans to improve the way we find literature mentions. We're going to include papers that often serve as proxies for software name mentions, particularly those identified within CRAN as the author's prefered attribution. We're also going to incorporate more network information, using not just how many people have cited something, but *who*. Eventually we'll incorporate citation information into the software reuse network, building a comprehensive, heterogeneous impact map. Finally, we'll be working with publishers to text-mine larger subsets of the research literature.



## Person impact

Depsy calculates transitive credit for the authors of research software.  We do this by accumulating the impact of the research software they've contributed to, in proportion to the size of the contribution they've made to each project. 

### Person impact: authorship
+ The author names are parsed from the author bylines in the PyPI and CRAN metadata
+ Links between packages and GitHub repositories were identified using metadata in the PyPI and CRAN as well as manual searching (todo: check if we ended up using any other methods)
+ For each Package with an associated GitHub repository, we augment the author list with the GitHub owner and GitHub commiters.  We get these using the GitHub api.  We consider an author identicial when they have the same email address, github credentials, or, barring that, identical name. 
+ Within a given project, authors are deduplicated using identical first and last names, even if they have different email addresses

### Person impact: contribution-level quantification
+ GitHub committers assigned a contribution-level proportional to their proportion of the software commits
+ Authors who aren't GitHub committers assigned contribution-level equal to that of the most-contributing GitHub committer
+ GitHub owners are assigned a token 1% contribution-level
+ Credits are weighted such that the contribution-level assigned across all contributors on a Package sums to 1.


### Person impact: component scores

We calculate research software impact subscore for downloads, software reuse, and literature citations for each author.  

We do this by summing a fraction of the percentile scores across each of the research software packages they've contributed to, using a fraction equal to the contribution-level they've made to the package, as we calculate above -- if you contribute more to a project, you inherit more of its impact.  

Next, more percentiles: we calculate the percentile of each of these aggregated fractional percentile scores against those of all the other people in the network who primarily code in the same language as the given author.  

The result of the calculations to this point is that each author has three percentiles that represent the impact they've made in research software: one for download, another for software reuse, and a third for citatons.

### Person impact: overall transitive credit

Finally, we calculate an overall person-level score.  We get this by averaging the three componenet subscore percentiles, then, yes, you guessed it, then taking the percentile of *that* across all people.

It is a lot of percentiles, we agree.  :)  The data is very skewed, with different distributions across the subscores.  Stay tuned for blog posts exploring this in more detail.


## Scope details and limitations

- we get contribution/commit information for packages hosted on GitHub.  We don't yet get contribution information from BitBucket or any other place where package contributor information might be stored.  We gather information on the first 100 committers. 
- we're missing GitHub links for many packages, which unfortunately means we are missing their detailed contributor/commit information.  We are currently finding links to GitHub that are included in Pypi and Cran metadata when the link is to a GitHub repo, but missing it when it is to a GitHub.io page.  We've also added some manually.  We attempted to link more though the contents of setup.py files on github, but this unfortuantely provided many false matches so we aren't including those links until we can improve the recall.
- we look for dependency information on all python and R GitHub repositories created before Jan 1 2015 (when the GitHub API changed, and Google Big Query changed their GitHub data structure).  We don't yet search BitBucket or any other code repositories.
- we include active packages, which means we've deleted those without source code available on CRAN/pypi, or in the case of pypi libraries with fewer than 50 downloads (affects about 100 packages).
- self citation happens. This can affect download numbers, but also imports and especially literature mentions.
- we do our best to guess whether a package is is_academic.  It's hard to know what this is for R; right now we consider all R packages to be academic.  
- download numbers are inflated by use of projects on CI servers; a download isn't necessarily by a person.

