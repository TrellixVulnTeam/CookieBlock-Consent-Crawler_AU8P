# CookieBlock Consent Webcrawler

This repository contains two crawlers to extract category labels for cookies from popular consent 
management platforms that present well-defined cookie categories to the user.

They were used to gather a dataset to train the classifiers used for the __CookieBlock__ extension.

Currently supported CMPS are:
* Cookiebot
* OneTrust (including OptAnon, CookiePro and CookieLaw domains)
* Termly

To run the fast crawl to filter domains for CMP presence, execute `run_presence_crawl.py`.

To run the complete consent label crawl with cookie retrieval, execute `run_consent_crawl.py`.

This tool is based on OpenWPM v0.12.0 by _Steven Englehardt et al._, see: https://github.com/mozilla/OpenWPM

It uses the Consent-O-Matic extension by _Janus Bager Kristensen et al._ to automatically provide affirmative consent to the CMP plugins, see: https://github.com/cavi-au/Consent-O-Matic

Licensed under GPLv3, see included LICENSE file.

## Installation

### Pre-requisites

The main pre-requisite for OpenWPM is conda, a cross-platform package management tool.

Conda is open-source, and can be installed from https://docs.conda.io/en/latest/miniconda.html.

### Install

An installation script, `install.sh` is included to: install the conda environment,
install unbranded firefox, and build the instrumentation extension.

All installation is confined to your conda environment and should not affect your machine.
The installation script will, however, override any existing conda environment named openwpm.

To run the install script, run

    $ ./openwpm/install.sh

After running the install script, activate your conda environment by running:

    $ conda activate openwpm

See `openwpm/README.md` for more detailed install instructions.

## Usage (Presence Crawler)
    run_presence_crawl.py (--numthreads <NUM>) (--url <u> | --pkl <fpkl> | --file <fpath> | --csv <csvpath>)... [--batches <BCOUNT>]
    
    Options:
        -n --numthreads <NUM>       Number of processes to run in parallel.
        -b --batches <BCOUNT>       Number of batches to split the input into. More batches lessens memory impact. [Default: 1]
        -u --url <u>                Domain string to check for reachability.
        -p --pkl <fpkl>             Path to pickled domains.
        -f --file <fpath>           Path to file containing one domain per line.
        -c --csv <csvpath>          Path to csv containing domains in second column. Separator is ",".
## Usage (Consent Crawler)
    run_consent_crawl.py (cookiebot|onetrust|termly|all|none) (--num_browsers <NUM>) (--url <u> | --pkl <fpkl> | --file <fpath> | --csv <csvpath> )... [--use_db <DB_NAME>]
    
    Options:
        -n --num_browsers <NUM>   Number of browsers to use in parallel
        -d --use_db <DB_NAME>     Use specified database file to add rows to.
        -u --url <u>              URL string to crawl
        -p --pkl <fpkl>           File path to pickled list of urls to parse.
        -f --file <fpath>         Path to file containing one URL per line
        -c --csv <csvpath>        Path to csv containing domains in second column. Separator is ",".

    Available modes are:
        * all        : Try to detect which CMP is used on the website, then retrieve data for that CMP.
        * cookiebot  : Assume website uses Cookiebot.
        * onetrust   : Assume website uses OneTrust.
        * termly     : Assume website uses Termly.
        * none       : Only gather cookies, no consent labels.

## Repository Contents
* `collected_data/` : Target directory for the consent webcrawler output.
* `crawler_profile/`: Contains the Firefox browser profile used with OpenWPM. 
    
    The profile includes a pre-configured instance of Consent-O-Matic that references 
    a custom Termly ruleset found at: https://github.com/dibollinger/Consent-O-Matic/blob/termly_rule/termly_rules.json 
* `filtered_domains/`: Target directory for the presence webcrawler output.
* `logs/`: Target directory for OpenWPM log files.
* `openwpm/`: Contains the modified OpenWPM codebase, version 0.12.0 
* `run_consent_crawl.py`: This script forms the entry point for the crawler that makes use of the OpenWPM framework, 
    which retrieves cookies, cookie categories, and builds a SQLite3 database storing this data.
* `run_presence_crawl.py`: Efficient crawl that only utilises the python requests library. Its purpose is to filter out potential
    candidates for the more time-costly OpenWPM crawl, by verifying whether the provided domains contain
    a consent management platform from which we can extract category labels.
    
## Description

This tool allows the user to scrape websites for cookie consent purposes if 
the target website makes use of one of the supported Consent Management Platforms.
It automatically crawls each website, retrieves consent label if possible, and then
browses the subpages of the site to record cookie data.

Currently supported by the scripts are Cookiebot, OneTrust and Termly CMPs. 

Due to the GDPR, websites that offer their services to countries in the EU 
are required to request consent from visitors when the website attempts to 
store cookies on the visitor's browser. This is commonly accomplished by
websites using plugins offered by Consent Management Platforms (CMPs).

These plugins usually offer consent toggles for the visitor, and sometimes 
display detailed information of the purpose of each cookie present on the website. 
This crawler specifically targets CMP implementations that display such information,
for the purpose of gathering a dataset of cookie labels and purposes.

Each cookie is assigned to one of the following purpose classes:

* __Strictly Necessary Cookies__: Cookies that are required for the website to function 
    properly. These require no consent from the visitor and usually cannot be rejected, 
    but are still declared inside privacy policies and consent management popups.
* __Functional Cookies__: Cookies that provide additional services or improve the user 
    experience, but are not strictly necessarily for the website to function. This 
    includes cookies such as website style settings, user preferences, etc. 
* __Performance/Analytical Cookies__: These are cookies that gather anonymized data 
    from the user in order to report statistics of the website usage or website 
    performance to the host. This data should be used to improve the site and the 
    browsing experience for the visitors, but are not to be used for advertising 
    or data sale purposes.
* __Advertising/Tracking__: This category encompasses all cookies that are used 
    for advertising and tracking purposes. Often this also involves the collection
    of sensitive personal data, which may be sold to other interested parties. 
    This is generally the category of cookies where the loss of privacy is the largest
    concern. Depending on what data is being gathered, these cookies can identify a 
    visitor's habits, interests, interests both leisurly and political, as well as 
    name and identity, geographical location and social standing.
* __Uncategorized__: Some CMPs leave cookies uncategorized. This category catches
    all such declarations.
* __Unknown__: Some categories cannot be easily be assigned to any of the above categories. 
    This includes category labels such as "Information Storage and Access" or "Content Delivery" 
    as these labels state little about how the cookie is intended to be used. In addition,
    some CMP use language-specific declarations. This crawler only supports English language
    categories.

If a cookie has multiple purposes assigned, the tool will generally assign the less 
privacy-preserving class.

### Presence Crawler
Efficient crawl that only utilises the python requests library. Its purpose is to filter out potential
candidates for the more time-costly OpenWPM crawl, by verifying whether the provided domains contain
a consent management platform from which we can extract category labels.

Parameter `-n` specifies the total number of parallel processes to use to perform the crawl. Generally, 
the crawl has a low processing cost, so one can specify a large number of processes here to increase 
the speed of the crawl. This is particularly helpful in case a website has a long loading or read delay, 
because the other processes can still make progress.

Parameter `-b` is used to split the input into batches. After a batch is done, the result is flushed.
Useful to reduce the memory impact that the script has.

The presence crawl automatically attempts to find the correct URL for the given domain, and the 
results are dumped into the subfolder  `./filtered_urls/`. Results are split into:
* `bot_responses`: Crawls that failed likely because a bot detection script prevented access to the website.
* `cookiebot_responses`: URLs that apparently contain a valid Cookiebot CDN domain in its page source.
* `failed_urls`: URLs where the connection failed. (SSL errors are included)
* `http_responses`: URLs that reported a HTTP error which is not commonly associated with bot detection.
* `nocmp_responses`: URLs where no supported Consent Management Platform was found.
* `onetrust_responses`: URLs where the OneTrust CMP was found.
* `termly_responses`: URLs where the Termly CMP was found.

Note that this script does not guarantee that the resulting filtered URLs actually contain the CMP they
appear to use. It is still possible for the website to not have set up the CMP properly, or at all. It
also is not a guarantee that the websites which are reported having no consent management platform, 
actually do not have any of the examined CMPs. It is possible that the websites does not show this content
when not loaded through an actual browser.

Best used with an active VPN connection to a country currently in the EU. Due to GDPR, this maximizes the
chance for the consent management platform to be shown to the user.

### Cookie Consent Label Crawler
Efficient crawl that only utilises the python requests library. Its purpose is to filter out potential
candidates for the more time-costly OpenWPM crawl, by verifying whether the provided domains contain
a consent management platform from which we can extract category labels.

This script is noticeably slower than `run_presence_crawl.py`, as the OpenWPM framework 
utilizes the Selenium browser (with Firefox geckodriver) in order to perform the crawl, 
which takes up a significant portion of system resources.

The first positional argument defines which CMP to look for and extract category labels from.
If one specifies `all`, the crawl will look for each CMP in sequence until it finds a valid match.

Parameter `-n` specifies the number of parallel browsers to use. Since each Firefox browser eats a large amount
of memory and processing power, this should be chosen conservatively.

Parameter `--use_db <DB_NAME>` specifies the given SQLite database as output path. If not given, a 
new database will be created inside the subfolder `./collected_data`. 
Useful to continue crawls if one was interrupted prematurely.


#### Database Tables

Consent data table, stores data collected from the consent notice, including purpose label.

    TABLE consent_data  
      id INTEGER PRIMARY KEY,           -- Unique identifier
      browser_id INTEGER NOT NULL,      -- Browser that collected the data
      visit_id INTEGER NOT NULL,        -- Corresponds to the website that was targetted.
      name TEXT NOT NULL,               -- Name as specified in the CMP
      domain TEXT NOT NULL,             -- Domain as specified in the CMP

      cat_id INTEGER NOT NULL,          -- Category id to map to internal categories
      cat_name VARCHAR(256) NOT NULL,   -- Actual name of the purpose category.
      purpose TEXT,                     -- Description that specifies the purpose.
      expiry TEXT,                      -- Expiration time, given in some metric. Can also be Session.
      type_name VARCHAR(256),           -- Tracking technology type.
      type_id INTEGER                   -- Internal ID of the above type


Cookie data table, stores the data of the actual cookies that were encountered during the crawl.

    TABLE javascript_cookies
      id INTEGER PRIMARY KEY ASC,       -- Unique identifier
      browser_id INTEGER NOT NULL,      -- Browser that collected the data
      visit_id INTEGER NOT NULL,        -- Corresponds to the website that was targetted.
      record_type TEXT,                 -- Either "added-or-changed" or "deleted".
      time_stamp DATETIME               -- Timestamp when the cookie was set.
      
      name TEXT,                        -- Name (key) of the cookie.
      host TEXT,                        -- Origin of the cookie.
      path TEXT,                        -- Path under which the cookie is valid.
      value TEXT,                       -- Content of the cookie.
      expiry DATETIME,                  -- Expiration date of the cookie.
      is_http_only INTEGER,             -- Boolean HTTP_ONLY flag.
      is_host_only INTEGER,             -- Boolean HOST_ONLY flag
      is_session INTEGER,               -- Boolean that indicates whether this is a session cookie.
      is_secure INTEGER,                -- Boolean SECURE flag.
      same_site TEXT,                   -- Either "no_restriction", "lax" or "strict.
   

Debug stats relating to the consent crawl, including details on success or failure and for what reason.

    TABLE consent_crawl_results 
      id INTEGER PRIMARY KEY,                  -- Unique identifier.
      browser_id INTEGER NOT NULL,             -- Browser that collected the data.
      visit_id INTEGER NOT NULL,               -- Corresponds to the website that was targetted.
      cmp_type INTEGER NOT NULL,               -- Which type of CMP? (0: Cookiebot, 1: OneTrust, 2: Termly, -1: None)
      crawl_state INTEGER NOT NULL,            -- Resulting error state. 0 is success.
      report TEXT                              -- Report describing the error, or number of cookies extracted.


# Credits and License

* Command Line Scripts and Consent Label Scraper Code, Copyright © 2021 Dino Bollinger
    - Licensed under the GPLv3

* OpenWPM v0.12.0, Copyright © 2015 Steven Englehardt
    - https://github.com/mozilla/OpenWPM
    - Licensed under the GPLv3

* Consent-O-Matic r154, Copyright (c) 2020 Janus Bager Kristensen, Rolf Bagge, CAVI
    - https://github.com/cavi-au/Consent-O-Matic
    - MIT License

Note: The included browser profile references the Consent-O-Matic fork at:  https://github.com/dibollinger/Consent-O-Matic/tree/termly_rule

This project is released under the GPLv3, see the included LICENSE file.

---

The scripts in this repository were created as part of a master thesis on GDPR Compliance, 
and is part of a series of repositories for the __CookieBlock__ browser extension:

https://github.com/dibollinger/CookieBlock

__Thesis Supervision and Assistance:__
* Karel Kubicek
* Dr. Carlos Cotrini
* Prof. Dr. David Basin
* The Institute of Information Security at ETH Zürich

__Additional Thanks go to:__
* Mozilla
* Consent-O-Matic Authors


